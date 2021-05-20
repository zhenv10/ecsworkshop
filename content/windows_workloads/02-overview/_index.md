---
title: "Windows Workloads Overview"
chapter: false
weight: 10
---

A significant amount of enterprise web applications use ASP.NET and being able to deploy these applications on containers is a great method for decoupling monolithic applications. You can run your Windows workloads on containers with Amazon ECS container instances that are launched using Amazon ECS-optimized Windows Server AMIs. For a complete list of ECS-optimized AMIs please refer to: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-optimized_AMI.html#ecs-optimized-ami-windows

Windows container instances run their very own version of the Amazon ECS container agent. Unlike the ECS container agent that runs inside a container for Linux instances, the ECS container agent for windows runs as a service on the host.

For additonal information, [check out the official AWS documentation.](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_Windows.html)

Here is a diagram that outlines what we will be building:

## need diagram of solution here

The application we will be working with is a simple Todo application with a React frontend that communicates with an ASP.NET API. The user can interact with the frontend to create new tasks, edit or delete existing tasks, and mark existing tasks as complete or incomplete which works in conjunction with the API to record changes to an Amazon RDS database. 

This tutorial will include step by step instructions on the deployment of a preconfigured environment on ECS through the use of AWS CDK.
