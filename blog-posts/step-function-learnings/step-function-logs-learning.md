---
published: false
title: "🤔 Where are the execution logs of my AWS Express workflow? 📖"
cover_image: https://raw.githubusercontent.com/MaximeVivier/Articles/master/blog-posts/step-function-learnings/assets/banner.png
description: 'How to have logs for an Express Workflows on AWS Step Functions with the CDK'
tags: aws, stepfunctions, serverless, cdk
series:
canonical_url:
---

## TL;DR

In this article you will learn how to add **logs in an Express workflow in Step Functions** with the CDK because it is not enabled by default. Since they are enabled by default on a Standard State Machine, it can be misleading going from a Standard to an Express workflow.

Here is the snippet of code you need.

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
          bar: 'foo',
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

## :bulb: Standard workflows have logs enabled by default inside AWS Step Functions
> :orange_book: Read this to better grasp what is happening with these log groups in Step Function or skip to the next section for the tutorial you came for

AWS explains it really clearly in its [documentation](https://docs.aws.amazon.com/step-functions/latest/dg/cw-logs.html), here is a quote describing the default logs behaviors of both Standard and Express types of State Machine.

>Standard Workflows record execution history in AWS Step Functions, although you can optionally configure logging to Amazon CloudWatch Logs.
>
>Unlike Standard Workflows, Express Workflows don't record execution history in AWS Step Functions. To see execution history and results for an Express Workflow, you must configure logging to Amazon CloudWatch Logs. Publishing logs doesn't block or slow down executions.

The following images show three different State Machines: one Express configured with logs, one Express configured without logs and one Standard. The only one that has a log group associated is the Express one configured with a log group.

![Three types of state machine](./assets/three-types-of-state-machine.png 'Three types of state machine')

![One log group associated to one of the Express workflow](./assets/one-log-group.png 'One log group associated to one of the Express workflow')

And the logs available for the Standard type of State Machine are inside the AWS Step Functions service but there is no log group associated.

![Logs of Standard State Machine is in the AWS Step Function service](./assets/logs-for-standard-step-func.png 'Logs of Standard State Machine is in the AWS Step Function service')

## :computer: CDK is your best friend

![CDK is da real MVP](./assets/youDaRealMVP.jpeg 'CDK is da real MVP')

All you need is to create a log group, configure it and then attach it to the State Machine via the logs prop. If you want to jump directly to the snippet [here](#white_check_mark-how-to-configure-logs-in-an-express-state-machine-in-cdk) is a shortcut.

The CDK handles all the work of creating the role for the State Machine to write inside CloudWatch.

The LogLevel allows you to select what kind of information you want to have. The four levels are: `OFF`, `ALL`, `ERROR` and `FATAL`. It is recommended by AWS to choose `ALL` because to have logs of all executions and not only the failed ones in this [documentation](https://docs.aws.amazon.com/step-functions/latest/dg/diff-standard-express-exec-details-ui.html#exp-wf-exec-limitation-details-log-dependent-test).

But the `includeExecutionData` property of the CDK construct makes all the difference. Thanks to this property set to `true`, you have all step transition data displayed in the Step Functions console.

Express with NO *logs execution data*

![Express with NO logs execution data](./assets/express-with-logs-but-not-includeExecutionData-exe-logs.png 'Express with NO logs execution data')

Express with *logs execution data* because `includeExecutionData` is set to `true`

![Express with logs execution data because includeExecutionData is set to true](./assets/express-with-logs-exe-logs.png 'Express with logs execution data because includeExecutionData is set to true')

### :white_check_mark: How to configure logs in an Express workflow in CDK

Here is the snippet of code that you need to make it happen. 🧑‍💻

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
          bar: 'foo',
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

## Conclusion

You are all set to have logs on all your State Machines in CDK.
