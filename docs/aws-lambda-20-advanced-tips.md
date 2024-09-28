---
title: 20 Advanced Tips for AWS Lambda
author: ahmed khaled
pubDatetime: 2024-09-12T13:12:51Z
slug: aws-lambda-20-advanced-tips
featured: false
draft: false
tags:
  - lambda
  - aws
  - serverless
description: "From the basics of what is AWS Lambda, to 20 advanced tips, 4 tools and a workshop"
---

**AWS Lambda** is a serverless compute service that runs your code in response to events and automatically manages the underlying compute resources for you. It's the most basic serverless building block, especially for [event-driven architectures](https://ahmedkhaled4d.com/p/event-driven-architecture-aws-basic-concepts).

Here's how it works:

- You create a function and write the code that goes in it.
- You set up a trigger for that function, such as an HTTP request
- You configure CPU and memory, and give that function an execution role with [IAM permissions](https://ahmedkhaled4d.com/p/aws-iam-permissions-a-comprehensive-guide)
- When the trigger event occurs, an isolated execution starts, receives the event, runs the code, and returns
- You only pay for the time the code was actually running

Obviously, that code runs somewhere. The point is that **you don't manage or care where** ('cause it's serverless, you see). Every time a request comes in, Lambda will either use an available execution environment or start a new one. That means Lambda scales up and down automatically and nearly instantly.

Here's the fine details:

- Supported languages are: Node.js, TypeScript, Python, Ruby, Java, Go, C# and PowerShell. Use a custom runtime for other languages.
- Lambda functions can be invoked from HTTP requests, in response to events from other services, or at defined time intervals (cron jobs).
- Billing is actually calculated as execution time \* assigned memory (GB-seconds), plus a fixed charge per invocation. CPU is tied to memory.
- Lambdas aren't actually instantaneous, there's a [cold start](https://aws.amazon.com/blogs/compute/operating-lambda-performance-optimization-part-1/?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda) (time to start the execution environment). Check the tips below for how to mitigate it.
- Logs are automatically generated and sent to CloudWatch Logs.

## Actionable tips

The most important tip is that **you don't need to do everything in this list**, and you don't need to do everything **right now**. But take your time to read it, **I bet there's at least one thing in there that you should be doing but aren't**.

- Lambdas don't run in a VPC, unless you [configure them for that](https://docs.aws.amazon.com/lambda/latest/dg/configuration-vpc.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda). You need to do that if you want to access VPC resources, such as an [RDS or Aurora database](https://ahmedkhaled4d.com/p/managed-relational-databases-aws-rds-aurora).
- Use environment variables.
- If you need secrets, put them in [Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda) and put the secret name in an environment variable.
- As always, grant minimum permissions only.
- Use [versioning](https://docs.aws.amazon.com/lambda/latest/dg/configuration-versions.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda) and [aliases](https://docs.aws.amazon.com/lambda/latest/dg/configuration-aliases.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda), so you can do [canary deployments](https://aws.amazon.com/blogs/compute/implementing-canary-deployments-of-aws-lambda-functions-with-alias-traffic-shifting/?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda), [blue-green deployments](https://docs.aws.amazon.com/whitepapers/latest/overview-deployment-options/bluegreen-deployments.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda) and [rollbacks](https://docs.aws.amazon.com/lambda/latest/dg/lambda-rolling-deployments.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda).
- Use [Lambda layers](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda) to reuse code and libraries.
- For constant traffic, Lambda is more expensive than anything serverful (e.g. [ECS](https://ahmedkhaled4d.com/p/ecs-basics-and-tips)). The benefit of Lambda is in scaling out extremely fast, and scaling in to 0 (i.e. if there's no traffic you don't pay).
- Not everything needs to be serverless. Choose the best runtime for each service.
- Use [Lambda Power Tuning](https://docs.aws.amazon.com/lambda/latest/operatorguide/profile-functions.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda) to optimize memory and concurrency settings for better performance and cost efficiency.
- Set [provisioned concurrency](https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda) to guarantee a min number of hot execution environments. It's not free.
- There's an account limit for concurrent Lambda executions. To ensure a particular function always has available limit, use reserved concurrency. It also serves as a maximum concurrency for that function.
- Use [compute savings plans](https://aws.amazon.com/savingsplans/compute-pricing/?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda) to save money.
- Already using containers? [Run your containerized apps in Lambda](https://docs.aws.amazon.com/lambda/latest/dg/images-create.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda). You could also consider ECS Fargate.
- Don't use function URLs, if you want to trigger functions from HTTP(s) requests use API Gateway instead. [Here's a tutorial](https://docs.aws.amazon.com/lambda/latest/dg/services-apigateway-tutorial.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda).
- If you have Lambdas that call other Lambdas, monitoring and tracing is a pain, unless you [use AWS X-Ray](https://ahmedkhaled4d.com/p/observability-with-aws-x-ray).
- Use [SnapStart](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda) to improve cold starts by 10x (only in Java, for now...)
- Code outside the handler only runs on environment initialization, not on every invocation. Put there everything you can, such as initializing the SDK.
- Reduce request latency with [global accelerators](https://docs.aws.amazon.com/global-accelerator/latest/dg/what-is-global-accelerator.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda).
- If you're processing data streams from Kinesis or DynamoDB, [configure parallelization factor](https://docs.aws.amazon.com/lambda/latest/dg/with-kinesis.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda#kinesis-polling-and-batching) to process a shard with more than one simultaneous Lambda invocation.

## Recommended Tools and Resources

Lambda is supported by every **IaC tool** out there. But if you're working with serverless, you'll want to **check out these options** (and pick one, don't mix them):

- [AWS SAM](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda): It's like CloudFormation, but built for serverless. In fact, to deploy an app your SAM template is translated to CFN (or you can [use Accelerate](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/accelerate.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda)). And it lets you run Lambda functions locally.
- [Serverless Framework](https://www.serverless.com/framework/?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda): Cloud-agnostic IaC tool specifically built for serverless. Works great with Lambda and many other serverless AWS services such as SQS, SNS, API Gateway, DynamoDB and more. The bad news is that, if a service is not supported, you can't work around that. Also lets you run your apps locally (though I've encountered a few problems with SNS and SQS).
- [AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/home.html?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda): An IaC tool built for programmers. Most tools are declarative, you write a config file. CDK is imperative, you use a programming language and declare variables, control structures and loops. It's not specific for serverless, but it's a lot more dev-friendly than most. Also supports locally running your apps.

All of the above are great, but a bit limited for running things locally. [LocalStack](https://localstack.cloud/?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda) is better (though some features are paid). Try to use your IaC tool's capabilities, but if you hit a wall, definitely **give LocalStack a shot**.

[Serverless Land](https://serverlessland.com/?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda) is a place with a ton of serverless resources.

And check out the [Serverlespressso AWS Workshop](https://workshop.serverlesscoffee.com/?utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda). They built a serverless app to serve coffee, and [set up a coffee shop at the expo in AWS Re:Invent 2021](https://www.youtube.com/watch?v=M6lPZCRCsyA&utm_source=simpleaws&utm_medium=referral&utm_campaign=20-advanced-tips-for-aws-lambda).
