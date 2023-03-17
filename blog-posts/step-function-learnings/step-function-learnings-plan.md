---
published: true
title: 'I answer to your worst step functions pain points'
description: 'Step Function learnings'
tags: serverless, stepfunction, dynamodb, javascript
series:
canonical_url:
---

In this article you will learn how to add **logs in an express step function** with the CDK because it is not enabled by default.

TL;DR

## Standard has logs enabled by default inside AWS Step Function
### Read this to better grasp what is happening with those log groups in Step Function or skip to the next section for the tutorial you came for

AWS explains it really clearly in its documentation, here is a quote describing the default logs behaviors of both Standard and Express types of State Machine.

>Standard Workflows record execution history in AWS Step Functions, although you can optionally configure logging to Amazon CloudWatch Logs.
>
>Unlike Standard Workflows, Express Workflows don't record execution history in AWS Step Functions. To see execution history and results for an Express Workflow, you must configure logging to Amazon CloudWatch Logs. Publishing logs doesn't block or slow down executions.

In the following images, you can see that for three different State Machines one Express configured with logs, one Express configured without logs and one Standard, the only one that has a log group associated is the Express one configured with a log group.

![Three types of state machine](./assets/three-types-of-state-machine.png 'Three types of state machine')

![One log group associated to one of the Express State machine](./assets/one-log-group.png 'One log group associated to one of the Express State machine')

And the logs available for the Standard type of State Machine are inside the AWS Step Function service but there is no log group associated.

![Logs of Standard State Machine is in the AWS Step Function service](./assets/one-log-group.png 'Logs of Standard State Machine is in the AWS Step Function service')

## How to configure logs in an Express Step Function in CDK

This is pretty easy and straightforward but you just need to know it.

```ts
import { App, RemovalPolicy, Stack } from 'aws-cdk-lib';
import { LogGroup, RetentionDays } from 'aws-cdk-lib/aws-logs';
import { LogLevel, Pass, StateMachine, StateMachineType } from 'aws-cdk-lib/aws-stepfunctions';

export class ArticleStack extends Stack {
  constructor(scope: App, id: string) {
    super(scope, id);

    const expressLogGroup = new LogGroup(this, 'ExpressLogs', {
      retention: RetentionDays.ONE_DAY,
      removalPolicy: RemovalPolicy.DESTROY,
    });

    new StateMachine(this, 'ExpressWithLogs', {
      definition: new Pass(scope, 'ExampleStepForExpressWithLogs', {
        parameters: {
          tata: 'titi',
        },
      }),
      stateMachineType: StateMachineType.EXPRESS,
      logs: {
        destination: expressLogGroup,
        level: LogLevel.ALL,
        includeExecutionData: true,
      },
    });
  }
}
```

For this snippet of code, I used the V2 of the CDK (2.56 to be more precise)

All you need is to create a log group, configure it and then attach it to the State Machine via the logs prop.

## Conclusion

You are all set to have logs on all your Step Functions in CDK.
