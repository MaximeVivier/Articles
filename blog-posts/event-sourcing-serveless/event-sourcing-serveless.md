---
published: false
title: 'A prod-safe event sourcing architecture has never been so simple to implement in AWS serverless'
cover_image:
description: 'Description of the article'
tags: serverless, eventsourcing, AWS, CQRS
series:
canonical_url:
---

:money_with_wings: Bank applications are one of the type of business requiring to **store a trace of all events** and **use the history** - just to name a few, accounting, medical records, transportation logistics... If you have that need and you're thinking about implementing it in **serverless**, you are in the right place. This article aims at sharing with you **2 years of learnings** :heart_eyes:, saving you the trouble to make the same mistakes I did, and providing you with a guidebook to have the most stable and the least painful solution you can go with. :rocket:

## Event sourcing for storing all actions made

You probably use a bank application in your every day life and you wonder how it works. Every credit and debit transaction made is stored as an **event** in a data store. This event data store is the **source of truth** and the current state of your account is computed from all events stored in the datastore. This is known as **event sourcing**.

<img src="./assets/compute-event-sourcing-state.png" style="display: block; margin: auto" width="200"/>

> - Ça te paraît trop loin d'avoir ici ce que je présente dans mon article ? --> Est-ce que ça doit être vraiment au tout début de l'article

## Event sourcing's best friend interface is CQRS

CQRS (**Command** and **Query** Responsibility Segregation) is a pattern that **separates read and write operations** applicable to a data store solution, which is an event sourcing solution in our case. The credit and debit events are written into a database different from the database which the displayed data is read from. This is the best interface to handle event sourcing data storing solution.

It's a good thing to know CQRS to better grasp what will be approached in this article, but don't worry it is not required. Numerous resources already deal with this topic and you can easily look it up on the web. [Here](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs) is one excellent page to understand more deeply the ins and outs of CQRS pattern.

## Let's have an overview of the global architecture

Let me show you the example of the banking application. It needs to display both the balance of your account and the last transaction you made.

<img src="./assets/event-sourcing-x-CQRS-simple-schema.png" style="display: block; margin: auto" width="600"/>

If you are user 2, you see that the application will fetch the total balance of your account from a read model that is called a **projection**. The entity that handles that action is called a **query** (cQrs). The projection of the balance is computed from the events stored in a database called the **event store**. A data synchronization system must be implemented to have accurate reading data compared to the source of truth. And when you make a transaction, a request is made to a **command** (Cqrs) that will write the event in the event store.

Now let's see what AWS serverless technologies to use to implement it.

> - Présentaion de plan
> - Spot les endroits où il manque du why

## The event store is the source of truth of the application

ADR : pourquoi on a choisi DynamoDB pourquoi la version (déjà un peu dit), préciser pourquoi pas timstamp (écriture concurente d'événements)

All credit and debit events for every user are stored in the event store. The access pattern I chose is the **user id** as the `partition key` and the number of the event also called the **version** as the `sort key`. That way we have a composite primary key with the PK and the SK.

> :question: Why using a version field when we can take the timestamp to keep track of the order of the events?
>
> :arrow_right: That's a good question, and it will be best answered when the aggregate notion will be introduced.

<img src="./assets/event-store-schema.png" style="display: block; margin: auto" width="600"/>

The event store represents the first block of the application.

<img src="./assets/archi-step-1-event-store.png" style="display: block; margin: auto" height="300"/>

## A command is the Cqrs tool to write data

It's a handler that writes an event into the event store when triggered.

API Gateway is the AWS managed service that enables serverless HTTP endpoints routing.

Whenever you pay 20$ with your card, a POST request is made to the endpoint <code>account/{accountId}/<b>debit</b></code> with a body `{ amount: 20 }`. Then this triggers a lambda function dedicated to writing **debit events** in the event store, and in this case it writes the event `{ 'accountId': 'abcd1234-user2', 'version': 12, 'eventType': 'Debit', 'amount': 20, 'timestamp': '2022-07-25T20:13Z'}`. According to the same principle, **credit events** are written into the event store.

<img src="./assets/archi-step-2-command.png" style="display: block; margin: auto" height="300"/>

But we face an issue here. What happens if you want to buy blue jeans for 100\$ but you only have 50\$ and considering that we don't accept overdraft? We need **a way to know the current balance of your account** to implement some **business logic** to avoid unauthorized transactions. The computation of the current state of the account in question is called the aggregate.

## Computing the aggregate is knowing the current state to make business decisions

Be sure to remember that the source of truth of the application is the event store. Therefore business verifications need to be made on the event store and not on the projection which is only the read model. For example if you want to buy blue jeans for 100\$ but you only have 50\$, considering that the bank doesn't accept overdraft it should be an unauthorized event. **The aggregate is the way to know the current balance of your account** to implement that kind of **business logic** by computing the current state of the account in question. In our example the balance of the current state is 50\$.

<img src="./assets/aggregate-schema.png" style="display: block; margin: auto" height="150"/>

Before creating an event in the command, the lambda queries DynamoDB on the partition key with your user id and gets a sorted list of your events based on the version of them. Then every debit and credit event are simply replayed in a reducer that outputs the total balance of the account. Unfortunately the computation of the balance reveals that you can't afford these blue. Consistent data is then stored in the source of truth.

> :question: So why the version field ?

Two commands coming in concurrently, means when they are about to be put in the database they both depend on the same aggregate they fetched to perform business tests because they arrived approximately at the same time. Let's say you have 50$ on your bank account and you make two purchases simultaneously, one of 40$ and one of 25$. If the sort key is the timestamp, both the events would be accepted because they would both succeed the business validation tests and you would be overdrawn. On the other hand, if the sort key is the version of the event, one out of two event would be invalid. Exact simultaneity doesn't exist in computer science, so the first purchase of 40$ would be written and then miliseconds after the second purchase would be accepted because the event to be written is based on the same version of aggregate and then have the same user id and version as the event written just before. The version fields is important to deal with this kind of situation.

<img src="./assets/archi-step-3-aggregate.png" style="display: block; margin: auto" height="300"/>

## The projections are the representation of the data in the application

First things first, where do we store our data? As banking application, the number of read actions is going to be substantial. Also as the number of clients increases we need to have automatic scalable resources to absorb the growing traffic without giving too much thoughts about it. For these reasons we are going to go also with DynamoDB to create the projections database. :warning: **Le why ici n'est toujours pas ouf** :warning:

How to shape the data? The main purpose of CQRS is to split writing and reading data, so the reading part is optimized by correctly generating the models. In order to be as fast as possible, the read models projections can store data as it is viewed in the application, each projection representing a business entity. As the amount of storage is not the issue but the running time is, this is the most efficient way to store data even if it means duplicating some parts of read models data.

In this use case, the application displays the total balance of your account and the last transaction you made. For that purpose we will simply store these data for every user. The partition key is configured as the id of the user, that way every projection regarding the same user is stored in the same partition. The access pattern must allow for filtering the type of projection when retrieving data, so the sort key is the type of the projection.

<img src="./assets/projection-schema.png" style="display: block; margin: auto" width="600"/>

Here comes the new block of the application: the **projection**.

<img src="./assets/archi-step-4-projection.png" style="display: block; margin: auto" width="700"/>

## The reading part of the CQRS

The application doesn't show anything for the moment because there is still no data fetched. Like the command, lambda functions behind HTTP endpoints are set up but here their goal is to retrieve data from DynamoDB. These lambda rightly named Query, aka the Q in CQRS.

You go to your application to see the last transaction occurred into your account to be sure you emitted it. By navigating to the page where you can see it, a GET request is sent to the url <code>account/{accountId}/<b>last-transaction</b></code>. A lambda is then triggered, it queries the projection DynamoDB with the required parameters `{ 'accountId': 'abcd1234-user2, 'projectionName': 'LastTransaction' }`. It sends back the data synchronously to the frontend for it to display to you your last transaction recorded, and fortunately you are the emitter.

We have the two sides of the CQRS architecture, with the writing in the command and the reading in the query. But there is a big part missing between those two.

<img src="./assets/archi-step-5-query.png" style="display: block; margin: auto" width="1000"/>

## How is the synchronization between read and write models ensured ?

The pain point in CQRS is to have reliable reading data compared to the source of truth. We need to find a way to compute and write projections according to events written in the event store.

### A projector is a handler

A **projector** is simply a lambda writing into DynamoDB.

But where does the data come from? It's legitimate to think about Event Bridge events to carry data to trigger lambda. For example, the total balance projector can be triggered by a 20$ debit event `{ 'eventType': 'Debit', 'amount': 20}`. The DynamoDB UpdateItem method gives the opportunity to manipulate the value of an existing record without querying it beforehand. In the case of a debit, the number 20 is subtracted from the existing value of the projection. The same projector can listen to credit events also and updates the projection the same way. We can think of updating the last transaction projection with an event carrying `{ 'eventType': 'Credit', 'amount': 40}` by overwriting the previous value of the projection.

<img src="./assets/archi-step-6-projector.png" style="display: block; margin: auto" width="1200"/>

### A fanout listening to Dynamodb Streams emits the events

The objective to react in real time to events written inside the event store. For that DynamoDb offers a powerful tool that is called DynamoDB Streams. We plug a lambda to the streams that will be triggered by every action made in the database. DynamoDB batches its streams of events in arrays, so we filter the `INSERT` actions and for each of them we publish an event in Amazon EventBridge.

> :question: You may wonder why not directly plug projector to the streams

For 2 reasons:

- Stream events that trigger a failing lambda will retry indefinitely until the lambda succeeds. That makes it impossible to have a custom retry policy for events coming from streams.
- You can only plug 2 lambda to the streams, and generally application contain more than 2 sets of data to display.

This type of handler is called a fanout. It delivers events to one or multiple recipients. By doing so, each projector only needs to listen to event altering the data it's associated to. For example a projector writing user individual information such as the name family name and address doesn't need to be triggered by a credit event.

The architecture seems now complete, from the emission of the event, to the writing of it, then the synchronization of the read models and eventually the reading of this new data.

<img src="./assets/archi-step-7-fanout.png" display="block" style="display: block; margin: auto" width="1200"/>

# And now that's how to have a prod-proof architecture

This works fine in dev but not in production yet. But with this data synchronization architecture you will face bugs later on. Let me explain :

- When traffic starts to grow, more and more events will flow and one thing EventBridge doesn't ensure is the order of the events sent in the bus event. In order not to overwrite an information with an older one, the last version of event that updated the projection must be kept track of.
- EventBridge ensures that an event is sent at least once, but it can still be received more than once. So projections need to be idempotent idempotent.
- In production, when a projection needs to updated or a new one needs to be created, wiping the databases and creating new data is the easy way to go to test this new feature. But in production that is not an option. Projections are made from the succession of all events that happened, that's why the solution is to find a way to replay all events to recreate the projections. Projectors are idempotent and disorder-proof, so they can handle smoothly a replay of all events.

> :bulb: EventBridge offers the possibility to replay all events that went through its service. This feature is called EventBridge Archive.

There are 2 main issues with that solution:

- actually Archive is not 100% accurate and might be incomplete or have a bit more events than what's stored in the event store
- replaying all events implies a high traffic when having a high number of events

> :star2: If you think about it, none of the above problems would be one if only projectors had access to the aggregate when updating a projection.

## Let's give access to the aggregate to all projectors

The aggregate represents the current state of the data at the version of the latest event taken into account. Furthermore, there is no rule about the shape of the aggregate and it can be used in the most exploitable way for this use case, such as direct computing of the projections. By doing so, a projection would only have to be updated with the corresponding part of the aggregate if its version is more recent than the version of the current stored projection. **Drawback = lambda en plus mais perf est => Aller voir dans les logs pour donner une estimation du temps pour l'execution du dispatch aggregate (sans cold start)**

The replay would only require to trigger an event listened by all projector in order to update all projections at once.

The trick I came up with is implementing state carried events which means attaching the aggregate to the dispatched events. This is instead of computing it every time a projector is triggered and giving them all access to the event store. It takes the form a new Lambda function that takes only one event in argument, computes the aggregate and attaches it to the event before republishing it. The fanout still transforms an array of streams to published EventBridge events. It also has still very low risk of failing by not adding the action of computing the aggregate out of all DynamoDB events.

Now projectors have access to the exact value of the projection they each handle and the version of the event it comes from. So it not necessary to update the projection based on the previous value anymore. The projector can simply use the PutItem command of DynamoDB to overwrite the previous value only if the version incoming is strictly superior to current one stored.

<img src="./assets/archi-step-8-dispatch-aggregate.png" display="block" style="display: block; margin: auto" width="1200"/>

> :fire: And there it is !

A prod-safe architecture implementing an event sourcing data storing pattern and managing it with a CQRS interface. All blocks are serverless managed services that imply a cost effective solution while being scalable and good for the environment.

## Limits

- It requires a lot of setup, it may not be adequate for all type of application.
- If you don't intend to use the history of events as an actual data for your application but only for legal obligations, you could still log every event made in a database. That way you keep track of every actions a user performed but you don't have to worry about all this setup by simply implementing CRUD solution.
- Other than creating another database to logs these actions, you can activate DynamoDB versioning to keep track of all changes made on the table.
