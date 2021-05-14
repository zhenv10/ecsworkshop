---
title: "Embedded tab content"
disableToc: true
hidden: true
---

### Deploy our application, service, and environment

First, let's test the existing code for any errors.

```bash
cd ~/environment/ecs-windows-workloads
cdk synth
```

This creates the cloudformation templates which output to a local directory `cdk.out`.   Successful output will contain (ignore any warnings generated):

```bash
Successfully synthesized to /home/ec2-user/environment/ecs-windows-workloads/cdk.out
Supply a stack id (VPCStack, RDSStack, ECSStack) to display its template.
```

(Note this is not a required step as `cdk deploy` will generate the templates again - this is an intermediary step to ensure there are no errors in the stack before proceeding.  If you encounter errors here stop and address them before deployment.)

Then, to deploy this application and all of its stacks, run:

```bash
cdk deploy --all --require-approval never --outputs-file result.json
```

The process takes approximately 10 minutes.  The results of all the actions will be stored in `result.json` for later reference.

{{%expand "Expand to view deployment screenshots" %}}
NEED RESULT SCREENSHOTS
{{% /expand%}}

### Code Review

Let's review whats happening behind the scenes.

The repository contains a sample application that deploys a

First, let's look at the application context variables:

{{%expand "Review cdk.json" %}}

```json
{
  "app": "npx ts-node --prefer-ts-exts bin/windows-workloads.ts",
  "context": {
    "@aws-cdk/core:enableStackNameDuplicates": "true",
    "aws-cdk:enableDiffNoFail": "true",
    "@aws-cdk/core:stackRelativeExports": "true",
    "@aws-cdk/aws-ecr-assets:dockerIgnoreSupport": true,
    "@aws-cdk/aws-secretsmanager:parseOwnedSecretName": true,
    "@aws-cdk/aws-kms:defaultKeyPolicies": true,
    "@aws-cdk/aws-s3:grantWriteWithoutAcl": true,
    "dbUser": "postgres",
    "dbPort": 1433,
    "containerPort": 5000,
    "containerImage": "public.ecr.aws/o0u3i9v5/dnapiserver"
  }
}
```

Custom CDK context variables are added to the JSON for the application to consume:

* `dbUser` - database username
* `dbPort` - database port
* `containerPort` - port on which the container in the ECS cluster runs
* `containerImage` - image name that will be deployed from ECR

These values will be referenced by using the function `tryGetContext(<context-value>)` throughout the rest of the application.
{{% /expand%}}

Next, let's look at the Cloudformation stacks constructs.   The files in `lib` each represent a Cloudformation Stack containing the component parts of the application infrastructure.

{{%expand "Review lib/vpc-stack.ts" %}}

```ts
import { App, Stack, StackProps, Construct } from '@aws-cdk/core';
import { Vpc, SubnetType } from '@aws-cdk/aws-ec2'

export interface VpcProps extends StackProps {
    maxAzs: number;
}

export class VPCStack extends Stack {
    readonly vpc: Vpc;

    constructor(scope: Construct, id: string, props: VpcProps) {
        super(scope, id, props);

        if (props.maxAzs !== undefined && props.maxAzs <= 1) {
            throw new Error('maxAzs must be at least 2.');
        }

        this.vpc = new Vpc(this, 'ecsWorkshopVPC', {
            cidr: "10.0.0.0/16",
            subnetConfiguration: [
                {
                    cidrMask: 24,
                    name: 'public',
                    subnetType: SubnetType.PUBLIC,
                },
                {
                    cidrMask: 24,
                    name: 'private',
                    subnetType: SubnetType.PRIVATE,
                },
            ],
        });
    }
}
```

The VPC stack creates a new VPC within the AWS account.   The CIDR address space for this VPC is `10.0.0.0./16`.   It will set up 2 public subnets with NAT Gateways and 2 private subnets with all the appropriate routing information automatically.   An interface is setup to pass in the value for `maxAzs` which is set to 2 in the main application.
{{% /expand%}}

{{%expand "Review lib/rds-stack.ts" %}}

```ts
code for RDS Stack goes here

```

Here, another Cloudformation Stack is setup containing the template to build an RDS Microsoft SQL Server Express Edition Instance.

In this example, a new randomized secret password for the RDS database is created and stored along with all the other parameters needed to connect to the database.   This is all done automatically through Secrets Manager integration with RDS.   When run, the stored credentials within Secrets Manager will look like this:

![Secrets Manager Detail](/images/secrets-manager-detail.png)

The stored credentials are passed to the DB along with the other context parameters.

{{%expand "Review lib/ecs-windows-stack.ts" %}}

Finally, the ECS service stack is defined in `lib/ecs-windows-stack.ts`

The ECS Fargate cluster application is created here using the `ecs-patterns` library of the CDK.   This automatically creates the service from a given `containerImage` and sets up the code for a load balancer that is connected to the cluster and is public-facing.   The key benefit here is not having to manually add all the boilerplate code to make the application accessible to the world.   CDK simplifies infrastructure creation by abstraction.

The stored credentials created in the RDS Stack are read from Secrets Manager and passed to our container task definition via the `secrets` property.  The secrets unique ARN is passed into this stack as a parameter `dbSecretArn`.

```ts
Code for windows stack here
```

{{% /expand%}}

{{%expand "Review bin/secret-ecs-app.ts" %}}
Finally, the stacks and the CDK infrastructure application itself are created in `bin/windows-workloads.ts`, the entry point for the cdk defined in the `cdk.json` mentioned earlier.

```ts
code for bin goes here.
```

A new CDK app is created `const App = new App()`, and the aforementioned stacks from `lib` are instantiated.  After creating the VPC, the VPC object is passed into the RDS and ECS stacks.  Dependencies are added to ensure the VPC is created before the RDS stack.

When creating the ECS stack, the same VPC object is passed along with a reference to the RDS stack generated `dbSecretArn` so that the ECS stack can look up the appropriate secret.  A dependency is created so that the ECS stack is created after the RDS Stack.
{{% /expand%}}

After deployment finishes, the last step for this tutorial is to get the LoadBalancer URL and run the migration which populates the database.

```bash
url=$(jq -r '.ECSStack.LoadBalancerDNS' result.json)   //check if this is still valid - name may be different or different export needed
//add code to curt the api/todos endpoint
```


#TODO
To view the app, open a browser and go to the Load Balancer URL `ECSST-Farga-xxxxxxxxxx.yyyyy.elb.amazonaws.com` (the URL is clickable in the Cloud9 interface):
![Secrets Todo](/images/secrets-todo.png)

This is a fully functional todo app.  Try creating, editing, and deleting todo items.  Using the information output from deploy along with the secrets stored in Secrets Manager, connect to the Postgres Database using a database client or the `psql` command line tool to browse the database.
