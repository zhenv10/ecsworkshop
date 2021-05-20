---
title: "Embedded tab content"
disableToc: true
hidden: true
---

### Deploy our application, service, and environment

First, let's test the existing code for any errors.

```bash
cd ~/environment/windows-ecs-cdk-example
cdk synth
```

This creates the cloudformation templates which output to a local directory `cdk.out`.   Successful output will contain (ignore any warnings generated):

```bash
Successfully synthesized to /Users/zhenv/Desktop/ecs-windows-workloads/cdk.out
Supply a stack id (VPCStack, RDSStack, ECSWindowsStack) to display its template.
```

(Note this is not a required step as `cdk deploy` will generate the templates again - this is an intermediary step to ensure there are no errors in the stack before proceeding.  If you encounter errors here stop and address them before deployment.)

Then, to deploy this application and all of its stacks, run:

```bash
cdk deploy --all --require-approval never --outputs-file result.json
```

The process takes approximately 10 minutes.  The results of all the actions will be stored in `result.json` for later reference.

{{%expand "Expand to view deployment screenshots" %}}
<!-- ![CDK Output 1](/images/cdk-output-1.png)
![CDK Output 2](/images/cdk-output-2.png) -->
{{% /expand%}}

### Code Review

Let's review whats happening behind the scenes.

The repository contains a sample application that deploys an ***ECS EC2 Service***.  The service runs this ASP.NET application that connects to a ***AWS RDS Aurora MySQL Database***.  The credentials for this application are stored in ***AWS Secrets Manager***.

First, let's look at the application context variables:

{{%expand "Review cdk.json" %}}

```json
{
  "app": "npx ts-node --prefer-ts-exts bin/windows-workloads.ts",
  "context": {
    "@aws-cdk/aws-apigateway:usagePlanKeyOrderInsensitiveId": true,
    "@aws-cdk/core:enableStackNameDuplicates": "true",
    "aws-cdk:enableDiffNoFail": "true",
    "@aws-cdk/core:stackRelativeExports": "true",
    "@aws-cdk/aws-ecr-assets:dockerIgnoreSupport": true,
    "@aws-cdk/aws-secretsmanager:parseOwnedSecretName": true,
    "@aws-cdk/aws-kms:defaultKeyPolicies": true,
    "@aws-cdk/aws-s3:grantWriteWithoutAcl": true,
    "@aws-cdk/aws-ecs-patterns:removeDefaultDesiredCount": true,
    "@aws-cdk/aws-rds:lowercaseDbIdentifier": true,
    "@aws-cdk/aws-efs:defaultEncryptionAtRest": true,
    "containerPort": 5000,
    "containerImage": "public.ecr.aws/o0u3i9v5/dnapiserver",
    "dbUser": "admin",
    "dbPort": 1433
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
import { Stack, StackProps, Construct } from '@aws-cdk/core';
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
import { App, StackProps, Stack, Duration, RemovalPolicy } from "@aws-cdk/core";
import {
    DatabaseSecret, Credentials, ParameterGroup, SqlServerEngineVersion, DatabaseInstance, DatabaseInstanceEngine, OptionGroup, StorageType
} from '@aws-cdk/aws-rds';
import { Vpc, Port, SubnetType, InstanceType, InstanceClass, InstanceSize, SecurityGroup, Peer } from '@aws-cdk/aws-ec2';
import { Secret } from '@aws-cdk/aws-secretsmanager';

export interface RDSStackProps extends StackProps {
    vpc: Vpc
}

export class RDSStack extends Stack {
    readonly vpc: Vpc;

    readonly dbSecret: DatabaseSecret;
    readonly sqlServerInstance: DatabaseInstance;
    readonly secgroup: SecurityGroup;

    constructor(scope: App, id: string, props: RDSStackProps) {
        super(scope, id, props);

        const dbUser = this.node.tryGetContext("dbUser");
        const dbPort = this.node.tryGetContext("dbPort") || 1433;

        this.dbSecret = new Secret(this, 'dbSecret', {
            secretName: "ecsworkshop/test/todo-app/sql-server",
            generateSecretString: {
                secretStringTemplate: JSON.stringify({
                    username: dbUser,
                }),
                excludePunctuation: true,
                includeSpace: false,
                generateStringKey: 'password'
            }
        });

        this.secgroup = new SecurityGroup(this, "apiSQLsg", {
            vpc: props.vpc,
            securityGroupName: "api-sqlsvr-sg",
            allowAllOutbound: false
        });

        this.secgroup.addIngressRule(Peer.anyIpv4(), Port.tcp(1433));

        this.sqlServerInstance = new DatabaseInstance(this, "MSSQLServer", {
            vpc: props.vpc,
            instanceIdentifier: "api-mssqlserver",
            engine: DatabaseInstanceEngine.sqlServerEx({ version: SqlServerEngineVersion.VER_14 }),
            credentials: Credentials.fromSecret(this.dbSecret, dbUser),
            instanceType: InstanceType.of(InstanceClass.BURSTABLE3, InstanceSize.SMALL),
            allocatedStorage: 100,
            storageType: StorageType.GP2,
            multiAz: false,
            vpcSubnets: { subnetType: SubnetType.PRIVATE },
            deletionProtection: false,
            backupRetention: Duration.days(0),
            removalPolicy: RemovalPolicy.DESTROY,
            parameterGroup: ParameterGroup.fromParameterGroupName(this, 'ParameterGroup', 'default.sqlserver-ex-14.0'),
            optionGroup: OptionGroup.fromOptionGroupName(this, 'OptionGroup', 'default:sqlserver-ex-14-00'),
            securityGroups: [this.secgroup],
            enablePerformanceInsights: false,
            autoMinorVersionUpgrade: false
        });

        this.sqlServerInstance.connections.allowFromAnyIpv4(Port.tcp(dbPort));

    }
}
```

Here, another Cloudformation Stack is setup containing the template to build an RDS Aurora MySQL Cluster.

The credentials to use with RDS are created with the following code:

```ts
        this.dbSecret = new Secret(this, 'dbSecret', {
            secretName: "ecsworkshop/test/todo-app/sql-server",
            generateSecretString: {
                secretStringTemplate: JSON.stringify({
                    username: dbUser,
                }),
                excludePunctuation: true,
                includeSpace: false,
                generateStringKey: 'password'
            }
        });
```

In this example, a new randomized secret password for the RDS database is created and stored along with all the other parameters needed to connect to the database.   This is all done automatically through Secrets Manager integration with RDS.   When run, the stored credentials within Secrets Manager will look like this:

<!-- ![Secrets Manager Detail](/images/secrets-manager-detail.png) -->

The stored credentials are passed to the DB along with the other context parameters.
<!-- 
A key feature in AWS Secrets Manager is the ability to rotate credentials automatically as a security best practice. In order to setup a new credentials rotation, a block is added in the constructor of `lib/rds-stack.ts`.

```ts
        new SecretRotation(
            this,
            secretName: `ecsworkshop/test/todo-app/aurora-pg`,
            {
                secret: this.dbSecret,
                application: SecretRotationApplication.POSTGRES_ROTATION_SINGLE_USER,
                vpc: props.vpc,
                vpcSubnets: { subnetType: SubnetType.PRIVATE },
                target: this.postgresRDSserverless,
                automaticallyAfter: Duration.days(30),
            }
        );
``` -->

Every 30 days, the secret will be rotated and will automatically configure a Lambda function to trigger the rotation using the `single user` method.  More information on the lambdas and methods for credential rotation can be found [here](https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_available-rotation-templates.html)

{{% /expand%}}

{{%expand "Review lib/windows-workloads.ts" %}}

Finally, the ECS service stack is defined in `lib/windows-workloads.ts`

The ECS EC2 cluster application is created here using the `ecs-patterns` library of the CDK.   This automatically creates the service from a given `containerImage` and sets up the code for a load balancer that is connected to the cluster and is public-facing.   The key benefit here is not having to manually add all the boilerplate code to make the application accessible to the world.   CDK simplifies infrastructure creation by abstraction.

The stored credentials created in the RDS Stack are read from Secrets Manager and passed to our container task definition via the `secrets` property.  The secrets unique ARN is passed into this stack as a parameter `dbSecretArn`.

```ts
import { App, Stack, StackProps, Duration } from '@aws-cdk/core';
import { Vpc, Peer, Port, SecurityGroup, InstanceType, InstanceClass, InstanceSize } from "@aws-cdk/aws-ec2";
import {
  Cluster, EcsOptimizedImage, WindowsOptimizedVersion, TaskDefinition,
  Compatibility, NetworkMode, ContainerImage, LogDrivers, Secret as ECSSecret
} from "@aws-cdk/aws-ecs";
import { ApplicationLoadBalancedEc2Service } from '@aws-cdk/aws-ecs-patterns';
import { LogGroup, RetentionDays } from "@aws-cdk/aws-logs";
import { Secret } from '@aws-cdk/aws-secretsmanager';
import { Role, ServicePrincipal, Effect, PolicyStatement } from '@aws-cdk/aws-iam';

export interface ECSWindowsStackProps extends StackProps {
  vpc: Vpc
  dbSecretArn: string
}

export class ECSWindowsStack extends Stack {

  constructor(scope: App, id: string, props: ECSWindowsStackProps) {

    super(scope, id, props);

    const containerPort = this.node.tryGetContext("containerPort");
    const containerImage = this.node.tryGetContext("containerImage");
    const creds = Secret.fromSecretCompleteArn(this, 'mssqlcreds', props.dbSecretArn);

    const cluster = new Cluster(this, 'Cluster', {
      clusterName: "ecs-windows-demo",
      vpc: props.vpc
    });

    const asg = cluster.addCapacity('WinEcsNodeGroup', {
      instanceType: InstanceType.of(InstanceClass.T2, InstanceSize.LARGE),
      machineImage: EcsOptimizedImage.windows(WindowsOptimizedVersion.SERVER_2019),
      minCapacity: 1,
      maxCapacity: 3,
      canContainersAccessInstanceRole: false,
    });

    const taskSecGroup = new SecurityGroup(this, "winEcs-security-group", {
      vpc: props.vpc
    });

    const taskRole = new Role(this, "EcsTaskRole", {
      assumedBy: new ServicePrincipal("ecs-tasks.amazonaws.com")
    });

    taskRole.addToPolicy(new PolicyStatement({
      actions: [
        "secretsmanager:GetSecretValue",
        "kms:Decrypt"
      ],
      resources: [
        `arn:aws:secretsmanager:${Stack.of(this).region}:${Stack.of(this).account}:secret/*`,
        `arn:aws:kms:${Stack.of(this).region}:${Stack.of(this).account}:key/*`,
      ],
      effect: Effect.ALLOW
    }))

    taskSecGroup.addIngressRule(Peer.ipv4("0.0.0.0/0"), Port.tcp(5000));
    taskSecGroup.addIngressRule(Peer.ipv4("0.0.0.0/0"), Port.tcp(80));

    asg.addSecurityGroup(taskSecGroup);

    const userData = [
      '[Environment]::SetEnvironmentVariable("ECS_ENABLE_AWSLOGS_EXECUTIONROLE_OVERRIDE", $TRUE, "Machine")',
      `Initialize-ECSAgent -Cluster ${cluster.clusterName} -EnableTaskIAMRole -LoggingDrivers '["json-file","awslogs"]'`
    ];

    asg.addUserData(...userData);

    const task = new TaskDefinition(this, "apiTask", {
      compatibility: Compatibility.EC2,
      cpu: "1024",
      memoryMiB: "2048",
      networkMode: NetworkMode.NAT,
      taskRole: taskRole
    });

    const logGroup = new LogGroup(this, "TodoAPILogging", {
      retention: RetentionDays.ONE_DAY,
    })

    const container = task.addContainer("TodoAPI", {
      image: ContainerImage.fromRegistry(containerImage),
      memoryLimitMiB: 2048,
      cpu: 1024,
      essential: true,
      logging: LogDrivers.awsLogs({
        streamPrefix: "TodoAPI",
        logGroup: logGroup,
      }),
      secrets: {
        DBHOST: ECSSecret.fromSecretsManager(creds!, 'host'),
        DBUSER: ECSSecret.fromSecretsManager(creds!, 'username'),
        DBPASS: ECSSecret.fromSecretsManager(creds!, 'password')
      }
    });

    container.addPortMappings({
      containerPort: containerPort
    });

    const ecsEc2Service = new ApplicationLoadBalancedEc2Service(this, 'demoapp-service-demo', {
      cluster,
      cpu: 1024,
      desiredCount: 2,
      minHealthyPercent: 50,
      maxHealthyPercent: 300,
      serviceName: 'winapp-service-demo',
      taskDefinition: task,
      publicLoadBalancer: true,
    });

    ecsEc2Service.targetGroup.configureHealthCheck({
      path: "/health",
      healthyThresholdCount: 2,
      unhealthyThresholdCount: 5,
      interval: Duration.seconds(60),
      timeout: Duration.seconds(15)
    });

  }
}

```

{{% /expand%}}

{{%expand "Review bin/secret-ecs-app.ts" %}}
Finally, the stacks and the CDK infrastructure application itself are created in `bin/windows-workloads.ts`, the entry point for the cdk defined in the `cdk.json` mentioned earlier.

```ts
#!/usr/bin/env node
import { App } from '@aws-cdk/core';
import { ECSWindowsStack } from '../lib/ecs-windows-stack';
import { RDSStack } from '../lib/rds-stack';
import { VPCStack } from '../lib/vpc-stack';

const app = new App();

const vpcStack = new VPCStack(app, 'VPCStack', {
    maxAzs: 2
});

const rdsStack = new RDSStack(app, 'RDSStack', {
    vpc: vpcStack.vpc
});

const ecsStack = new ECSWindowsStack(app, "ECSWindowsStack", {
    vpc: vpcStack.vpc,
    dbSecretArn: rdsStack.dbSecret.secretArn
});

ecsStack.addDependency(rdsStack);
rdsStack.addDependency(vpcStack);
```

A new CDK app is created `const App = new App()`, and the aforementioned stacks from `lib` are instantiated.  After creating the VPC, the VPC object is passed into the RDS and ECS stacks.  Dependencies are added to ensure the VPC is created before the RDS stack.

When creating the ECS stack, the same VPC object is passed along with a reference to the RDS stack generated `dbSecretArn` so that the ECS stack can look up the appropriate secret.  A dependency is created so that the ECS stack is created after the RDS Stack.
{{% /expand%}}

After deployment finishes, the last step for this tutorial is to get the LoadBalancer URL and run the migration which populates the database.

```bash
url=$(jq -r '.ECSWindowsStack' result.json | grep 'LoadBalancer*' | cut -f2 -d: | tr -d ' ",')
curl -s $url/migrate | jq
```

(Note that the migration may take a few seconds to connect and run.)

The custom method `migrate` creates the database schema and a single row of data for the sample application. It is part of the sample application in this tutorial.

To view the app, open a browser and go to the Load Balancer URL `ECSWi-demoa-xxxxxxxxxx.yyyyy.elb.amazonaws.com` (the URL is clickable in the Cloud9 interface):
![Secrets Todo](/images/secrets-todo.png)

Similar to the secrets section of this workshop, this Todo application is fully functional. Feel free to experiment with it by creating, editing, and deleting tasks. You can also connect to the MySQL Database using a database client or the `mysql` command line tool to browse the database.

As an added benefit of using RDS Aurora Postgres Serverless, you can also use the query editor in the AWS Management Console - find more information **[here](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/query-editor.html)**. All you need is the secret ARN created during stack creation.  Fetch this value at the Cloud9 terminal and copy/paste into the query editor dialog box.   Use the database name `tododb` as the target database to connect.

```bash
aws secretsmanager list-secrets | jq -r '.SecretList[].ARN'
```
