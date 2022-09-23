---
published: false
title: 'A prod-safe event sourcing architecture has never been so simple to implement in AWS serverless'
cover_image:
description: 'Description of the article'
tags: serverless, eventsourcing, AWS, CQRS
series:
canonical_url:
---

If you you're about to implement a **serverless** architecture for an event sourcing based application, you are in the right place. This article aims at sharing with you **2 years of learnings** :heart_eyes:, saving you the trouble to make the same mistakes I did, and providing you with a guidebook to have the most stable and the least painful solution you can go with. :rocket:

In this article I assume that you have at least a basic knowledge of what is CQRS. [Here](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs) is one excellent page to understand more deeply the ins and outs of CQRS pattern if you feel the need before diving into the interesting things.

## Quick overview of an event sourcing solution implementing a CQRS interface

:money_with_wings: Bank applications are one of the type of business requiring to **store a trace of all events** and **use the history** - just to name a few, accounting, medical records, transportation logistics...

Let's take the example of a banking application that would need to display both the balance of the account and the last transaction made.

![Logo kumo](./assets/ArchiMissingDataSynchro.png 'Logo Kumo')

This architecture is an implementation of **CQRS interface** for the use case of the bank which has to store all credit and debit events. There are two **commands** writing credit and debit events in an **event ledger**. There are two **queries** that fetch each its associated **projection**, the total balance of the account and the last transaction made.

As you can see, the missing part of the diagram is the synchronization between the write model and the read model. That's the block we will focus on.

## First and foremost, in event sourcing, the order of the events is essential

All credit and debit events for every user are stored in the event ledger. The access pattern I chose is the `userId` as the **partition key** and the number of the event also called the `version` as the **sort key**.

![Logo kumo](./assets/event-store-schema.png 'Logo Kumo')

> :question: Why using a `version` field when we can simply take the `timestamp` to keep track of the order of the events?

Two commands coming in concurrently will write data based on the same version of the aggregate because they fetched it simultaneously. Let's say a user has 50$ on his bank account and two purchases come in at the same time, one of 40$ and one of 25$. Both event would both succeed the business validation tests. The composite key `userId`/`timestamp` of both event would be unique because timestamps would be really close but not exactly the same. Hence, both events would be written in the event store and you would be overdrawn which is forbidden in our bank. On the other hand, if the sort key is the version of the event, one out of both would be invalid. Exact simultaneity doesn't exist in computer science, so the first purchase of 40$ would be written and then shortly after the second purchase would be rejected because the event to be written is based on the same version of the last event.

## How is the synchronization between read and write models ensured ?

> The pain point in CQRS is to have reliable reading data compared to the source of truth. We need to find a way to compute and write projections according to events written in the event store.

### A fanout listening to Dynamodb Streams emits the events

The objective here is to react in real time to events written inside the event ledger in order to compute the projections. For that a lambda as a fanout can be used by plugging it to the DynamoDB streams of the event ledger. DynamoDB batches its streams of events in arrays, `INSERT` actions have to be filtered with the filter patterns and for each of them an event is published in Amazon EventBridge. These EventBridge events are those triggering the lambda that project on the read models.

> :question: You may wonder why not directly plug the projector to the streams

For 2 reasons:

- Stream events that trigger a failing lambda will retry indefinitely until the lambda succeeds. That makes it impossible to have a custom retry policy for events triggering a projector.
- You can only plug 2 lambda to the streams, and generally application contain more than 2 sets of data to display therefore more than 2 projectors.

Mais en fait maintenant tu n'as plus ces deux limitations => je peux juste mettre un EDIT en mode maintenant vous pouvez racourcir la chaîne

The fanout delivers events to one or multiple recipients. By doing so, each projector only needs to listen to events altering the data it's associated to. For example a projector writing user individual information such as the name family name and address doesn't need to be triggered by a credit event.

![Archi without dispatch aggregate](./assets/archiWoDispatchAggregate.png 'Archi without dispatch aggregate')

## From a dev reliable architecture to a prod-safe architecture

This works fine in dev but not in production yet. This data synchronization architecture will face bugs later on. Let me explain :

- When traffic starts to grow, more and more events will flow and one thing EventBridge doesn't ensure is the order of the events sent in the bus event. In order not to overwrite an information with an older one, the last version of event that updated the projection must be kept track of.
- EventBridge ensures that an event is sent at least once, but it can still be received more than once. So projections need to be idempotent.
- In dev, when a projection needs to be updated or a new one needs to be created, wiping the databases and creating new data is the easy way to go to test this new feature. But in production that is not an option. Projections are made from the succession of all events that happened, that's why the solution is to find a way to replay all events to recreate the projections. Projectors are idempotent and disorder-proof, so they can handle smoothly a replay of all events.

## EventBridge Archive, the bogus good idea

> :bulb: EventBridge offers the possibility to replay all events that went through its service. This feature is called EventBridge Archive.

There are [2 main issues with that solution]():

- actually Archive is not 100% accurate and might be incomplete or have a bit more events than what's stored in the event store
- replaying all events implies a high traffic when having a high number of events

> :star2: If you think about it, none of the above problems would be one if only projectors had access to the aggregate when updating a projection.

## A trick to make data synchronization and replay easier

The trick I came up with is implementing state carried events. Instead of computing the same aggregate in every projector and giving them all access to the event store, the aggregate is computed once and it is attached to the dispatched events. It takes the form of a new Lambda function that takes only one event in argument, computes the aggregate and attaches it to the event before republishing it. I named it `dispatchAggregate`.

The fanout still transforms an array of streams to published EventBridge events triggering the dispatchAggregate Lambda. It also has still very low risk of failing by not adding the action of computing the aggregate out of all DynamoDB events.

![Archi with dispatch aggregate](./assets/archiWithDispatchAggregate.png 'Archi with dispatch aggregate')

This is the EventBridge event such as it is dispatched after the fanout and such as it was listened by projectors before the trick :magic_wand:

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

### The job of the projectors become trivial

The aggregate represents the current state of the data at the version of the latest event taken into account. Furthermore, there is no rule about the shape of the aggregate and it can be customized in the most exploitable way. For this use case, it can be built with keys being all the projections.

```json
{
  "lastKnownVersion": 3,
  "totalBalance": 500,
  "lastMonthBalance": -20
}
```

With an access to the aggregate, projectors have at their disposal the exact value of the projection they each handle and the last version of this data. So it's not necessary to update the projection based on the previous value anymore. The projector can **simply** use the **PutItem** command of DynamoDB to **overwrite the previous** value only if the version incoming is strictly superior to current one stored. The projections become idempotent and they allow more flexibility in the way they are triggered.

### The replay is then extremely simple

It's as simple as writing a `replay` type event in the ledger. All projectors need to listen to the corresponding EventBridge event. Then every time a `replay` type event is written in the ledger, all projectors update their projection with their latest version.

> Do not forget to ignore this `replay` type event in the reducer when computing the aggregate

## A possible drawback of the dispatchAggregate lambda

The computation time of the dispatchAggregate lambda adds run time to the process. But it is acceptable after checking the results I have on my lambda.

The **run time** of the dispatchAggregate lambda with a **cold start** is **250ms** on average and pretty much consistently.

![Dispatch aggregate cold start](./assets/dispatchAggregate-cold-start.png 'Dispatch aggregate cold start')

But the **run time** of the same lambda when it's already hot is between **30ms** and **80ms** for the slowest runs, depending on the `userId` in question.

![Dispatch aggregate 30ms](./assets/dispatchAggregate-30ms.png 'Dispatch aggregate 30ms')

![Dispatch aggregate 80ms](./assets/dispatchAggregate-80ms.png 'Dispatch aggregate 80ms')

The latency is very low in general because there is only one gateway through which the events go, therefore the lambda is hot most of the time.

> The run time topic may become an issue when the **number of events** becomes **enormous**.

For computing the aggregate, **all events** must be **fetched** and reduced within the lambda and it might be really long to handle that much events. But here again, there is a solution. In fact, **snapshots of the aggregate** can be stored in the event ledger. A snapshot representing the aggregate of version N allow **not to worry about events** of versions strictly inferior to N and start from this one when computing the aggregate.

## What's left for me to do to implement the most effective solution with the latest releases

Both the issues I've mentioned about the retry policy of the lambda plugged to the streams and the number of lambda you can plug to a stream is no longer true since this summer 2022.

In terms of efficiency, I still need a single lambda to compute the aggregate since it's time-consuming operation, so the rule of plugging multiple lambda to the streams is not a rule that will change this architecture.

The opportunity here is that I can have the lambda dispatching the events to the projectors failing without worrying about and infinite retry or loosing data. The error handling allows the configuration of destination for failed-events and for a retry policy of my choice. Having both a fanout that must not fail and a dispatchAggregate to compute and attach the aggregate to the event becomes useless. A step can now be skipped by having the dispatchAggregate receiving the DynamoDB streams, computing and attaching the aggregate to the events before publishing them in EventBridge. A retry policy and a destination for failed-events must be configured on this lambda.

![Final archi without fanout](./assets/finalArchiWoFanout.png 'Final archi without fanout')

This should improve the performances and make it the most efficient and up to date solution for synchronizing data between an event ledger and projections in an event sourcing application.
