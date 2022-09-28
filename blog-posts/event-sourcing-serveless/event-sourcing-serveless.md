---
published: false
title: 'Serverless event-sourcing with AWS: State of the art data synchronization'
cover_image:
description: 'Serverless Event-sourcing event-ledger DynamoDB AWS'
tags: serverless, eventsourcing, AWS, CQRS,
series:
canonical_url:
---

If you're about to implement a **serverless** architecture for an event sourcing based application, you are in the right place. This article aims at sharing with you **2 years of learnings** :heart_eyes:, saving you the trouble to make the same mistakes we did at Kumo, and providing you with a guidebook to have the most stable and the least painful solution you can go with. :rocket:

> :books: In this article I assume that you have at least a basic knowledge of what is **CQRS**. [This article is an excellent page to understand more deeply the ins and outs of CQRS pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs) if you feel the need before diving into the interesting things.

## :eyes: Quick overview of an event sourcing solution implementing a CQRS interface

:money_with_wings: Bank applications are one of the type of business requiring to **store a trace of all business events**. These business events are the transactions made on a bank account such as credit and debit actions.

Let's take the example of a **banking application** that would need to display both the balance of the account and the last transaction made.

![CQRS interface with AWS serverless technologies](./assets/ArchiMissingDataSynchro.png 'CQRS interface with AWS serverless technologies')

This architecture is an implementation of **CQRS interface** for the use case of the bank which has to store all credit and debit events. There are two **commands** writing credit and debit events in an **event ledger**. There are two **queries** that each fetch its associated **projection**, the total balance of the account and the last transaction made.

As you can see, the missing part of the diagram is the synchronization between the write model and the read model. That's the block this article focus on.

## :bar_chart: First and foremost, in event sourcing, the order of events is essential

All credit and debit events for every user are stored in the event ledger. The access pattern chosen is the `userId` as the **partition key** and the number of the event also called the `version` as the **sort key**.

![Event sourcing : DynamoDB event ledger](./assets/event-store-schema.png 'Event sourcing : DynamoDB event ledger')

> :question: Why using a `version` field when we can simply take the `timestamp` to keep track of the order of the events?

Two commands coming in **concurrently** will write data based on the **same version of the aggregate** because they fetched it simultaneously. Let's say a user has 50$ on his bank account and two purchases come in at the same time, one of 40$ and one of 25$. Both event would **both succeed the business validation tests**. The composite key `userId`/`timestamp` of both event would be unique because timestamps would be really close but not exactly the same. Hence, both events would be written in the event ledger and you would be **overdrawn** which is **forbidden** in our bank.

![Choice of version as SK in event sourcing](./assets/concurrent-events-version-sk-choice-bad.png 'Choice of version as SK in event sourcing')

On the other hand, if the **sort key** is the `version` of the event, **one out of both would be invalid**. Exact simultaneity doesn't exist in computer science, so the first purchase of 40$ would be written and then shortly after the second purchase would be rejected because the event to be written is based on the same version of the last event.

![Choice of version as SK in event sourcing](./assets/concurrent-events-version-sk-choice-good.png 'Choice of version as SK in event sourcing')

## :point_right: :point_left: How is the synchronization between read and write models ensured ?

> The pain point in CQRS is to have **reliable reading data** compared to the source of truth. Projections need to be updated according to events written in the event ledger.

### :ear: Listening to written events

The objective here is to **react in real time to events** written inside the event ledger in order to compute the projections. For that a lambda as a **fanout** can be used by plugging it to the **DynamoDB streams** of the **event ledger**. `INSERT` actions have to be filtered with the filter patterns so that only theses actions trigger the fanout. DynamoDB batches its streams of events in arrays, so for each of them an event is published in Amazon EventBridge. These EventBridge events are those triggering the lambda that project on the read models.

> :question: You may wonder why not **directly** plug the **projector** to the streams

[For 3 reasons](#chart_with_upwards_trend-whats-left-for-me-to-do-to-implement-the-most-effective-solution-with-the-latest-releases):

- Stream events that trigger a failing lambda will retry indefinitely until the lambda succeeds. That makes it **impossible** to have a **custom retry policy** for events triggering a projector.
- You can only plug **2 lambda maximum** to the streams, and generally application contain more than 2 sets of data to display therefore more than 2 projectors.
- The correct tool to plug multiple listeners to messages is an Event Bus.

The fanout delivers events to one or multiple recipients. Each projector, which is a recipient of those events, only needs to listen to events altering the data it's associated to. For example a projector writing user individual information such as the name family name and address doesn't need to be triggered by a credit event.

![Data synchronization for CQRS with AWS serverless technologies](./assets/archiWoDispatchAggregate.png 'Data synchronization for CQRS with AWS serverless technologies, 1st version')

## :muscle: From a dev reliable architecture to a prod-safe architecture

This works fine in dev but **not in production yet**. This data synchronization architecture will face bugs later on. Let me explain :

- When traffic starts to grow, more and more events will flow and one thing **EventBridge doesn't ensure** is the **order of the events** sent in the bus event. In order not to overwrite an information with an older one, the last version of event that updated the projection must be kept track of.
- EventBridge ensures that an **event** is sent at least once, but it can still be **received more than once**. So projections need to be idempotent.
- In dev, when a **projection** needs to be **updated** or a new one needs to be created, wiping the databases and creating new data is the easy way to go to test this new feature. But in production that is not an option. Projections are made from the succession of all events that happened, that's why the solution is to find a way to **replay** all events to recreate the projections. Projectors must be idempotent and disorder-proof, thus they can handle smoothly a replay of all events.

## :unamused: EventBridge Archive, the bogus good idea

> :bulb: EventBridge offers the possibility to replay all events that went through its service. This feature is called **EventBridge Archive**.

There are 2 main issues with that solution:

- replaying all events implies a **high traffic** when having a high number of events
- **some events** that are replayed can be **useless**, so that's traffic useless traffic and thus useless run time
- this leaves **two sources of truth**. And Archive data can't be edited, it is immutable. It can be an issue.

## :magic_wand: A trick to make data synchronization and replay easier

The best solution would be that every projector has access to the current state of the data every time they need to update their projection. They would all have to fetch all events to compute the aggregate every time an update of the projection would be needed. The same aggregate would be computed across multiple projectors at the same time, which makes this solution not very efficient.

> :sunglasses: Unless... !

The trick we came up with at Kumo is implementing **state carried events**. Instead of computing the same aggregate in every projector and giving them all access to the event ledger, the **aggregate** is **computed once** and it is **attached to the dispatched events**. It takes the form of a new Lambda function that takes only one event in argument, computes the aggregate and attaches it to the event before republishing it. We named it `dispatchAggregate`.

The fanout still transforms an array of streams to published EventBridge events triggering the dispatchAggregate Lambda. It also has still very low risk of failing by not adding the action of computing the aggregate out of all DynamoDB events.

![Data synchronization for CQRS with AWS serverless technologies with aggregate dispatcher](./assets/archiWithDispatchAggregate.png 'Data synchronization for CQRS with AWS serverless technologies with aggregate dispatcher')

This is the EventBridge event such as it is dispatched after the fanout and such as it was listened by projectors before the trick.

```json
{
  "userId": "163082d2-b005-75d7-5eb6-aa1f83efd55d",
  "eventType": "Debit",
  "payload": {
    "amount": 30
  },
  "version": 3
}
```

And this is the shape of the event triggering the projectors that contain the aggregate.

```json
{
  "aggregate": {
    "lastKnownVersion": 3,
    "totalBalance": 500,
    "lastMonthBalance": -20
  },
  "lastEvent": {
    "userId": "163082d2-b005-75d7-5eb6-aa1f83efd55d",
    "eventType": "Debit",
    "payload": {
      "amount": 30
    },
    "version": 3
  }
}
```

### :white_check_mark: The job of the projectors become trivial

The **aggregate** represents the **current state** of the data at the **version** of the **latest event** taken into account. Furthermore, there is no rule about the shape of the aggregate and it can be customized in the most exploitable way. For this use case, it can be built with keys being all the projections.

```json
{
  "lastKnownVersion": 3,
  "totalBalance": 500,
  "lastMonthBalance": -20
}
```

With an access to the aggregate, projectors have at their disposal the **exact value of the projection** they each handle and the last version of this data. So it's not necessary to update the projection based on the previous value anymore. The projector can **simply** use the **PutItem** command of DynamoDB to **overwrite the previous** value only if the version incoming is strictly superior to current one stored. The projections become idempotent and they allow more flexibility in the way they are triggered.

### :repeat_one: The replay is then extremely simple :ok_hand:

It's as simple as writing a `replay` type event in the ledger. All projectors need to listen to the corresponding EventBridge event. Then every time a `replay` type event is written in the ledger, all projectors update their projection with their latest version.

> :bookmark: Do not forget to ignore this `replay` type event in the reducer when computing the aggregate

## :mega: A possible drawback of the dispatchAggregate lambda

The computation time of the dispatchAggregate lambda adds run time to the process. But it is acceptable after checking the results we have on our lambda.

The **run time** of the dispatchAggregate lambda with a **cold start** is **250ms** on average and pretty much consistently.

![Aggregate dispatcher cold start](./assets/dispatchAggregate-cold-start.png 'Aggregate dispatcher cold start')

But the **run time** of the same lambda when it's already hot is between **30ms** and **80ms** for the slowest runs, depending on the `userId` in question.

![Aggregate dispatcher run time](./assets/dispatchAggregate-30ms.png 'Aggregate dispatcher run time')

![Aggregate dispatcher run time](./assets/dispatchAggregate-80ms.png 'Aggregate dispatcher run time')

The latency is very low in general because there is only one gateway through which the events go, therefore the lambda is hot most of the time.

> The run time topic may become an issue when the **number of events** becomes **enormous**.

For computing the aggregate, **all events** must be **fetched** and reduced within the lambda and it might be really long to handle that much events. But here again, there is a solution. In fact, **snapshots of the aggregate** can be stored in the event ledger. A snapshot representing the aggregate of version N allow **not to worry about events** of versions strictly inferior to N and start from this one when computing the aggregate.

## :chart_with_upwards_trend: What's left for me to do to implement the most effective solution with the latest releases

Both the issues I've mentioned about the retry policy of the lambda plugged to the streams and the number of lambda you can plug to a stream is no longer true since this summer 2022.

In terms of efficiency, We **still need a single lambda** to compute the **aggregate** since it's time-consuming operation, so the rule of plugging multiple lambda to the streams is not a rule that will change this architecture.

The opportunity here is that we can have the lambda dispatching the events to the projectors failing without worrying about and infinite retry or loosing data. The **error handling** allows the configuration of **destination for failed-events** and for a **retry policy of my choice**. Having both a fanout that must not fail and a dispatchAggregate to compute and attach the aggregate to the event becomes useless. A step can now be skipped by having the dispatchAggregate receiving the DynamoDB streams, computing and attaching the aggregate to the events before publishing them in EventBridge. A retry policy and a destination for failed-events must be configured on this lambda.

![Data synchronization for CQRS with AWS serverless technologies with aggregate dispatcher](./assets/finalArchiWoFanout.png 'Data synchronization for CQRS with AWS serverless technologies with aggregate dispatcher')

This should make it the most efficient and up to date solution for synchronizing data between an event ledger and projections in an event sourcing application.

- parler de castore
