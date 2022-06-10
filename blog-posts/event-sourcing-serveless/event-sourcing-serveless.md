---
published: false
title: 'The best way to do event sourcing in serverless'
cover_image:
description: 'Description of the article'
tags: serverless, event-sourcing, AWS, CQRS
series:
canonical_url:
---

# Introduction

You probably use a bank application in your every day life and you wonder how they work. They rely on event sourcing to store their data and they compute the balance of your account from all credit/debit events stored. Numerous applications need to store a trace of all events to keep and use the history. If you have that need and you're thinking about implementing it in serverless, you are in the right place. Thus I want to share with you, 2 years of learnings that will lead you to have the most stable and the less painful solution you can go with.

## CQRS and event sourcing

CQRS (Command and Query Responsibility Segregation) is a pattern that separates read and update operations for a data store. It's a good thing to grasp a bit what it is to understand better what will be approached in this article, but don't worry it is not required as it is quite straightforward. Still numerous resources already deal with this topic that you can easily look up on the web.

Event sourcing is a way to implement the data store in a CQRS pattern interface. There are also a lot of resources on the internet about it if you want to quickly take a look at it before deep diving into our banking application.

## What we will build together

Let's build the example of the banking application. We want to store data with the event sourcing pattern to be able to create any data view models from the list of credit/debit events of a user. For reading and writing data, we want to set up a CQRS interface which fits perfectly with the event sourcing data storing system. For now Let's take a look at what the architecture of such an application should look like.

_insert schema of archi agnostic of technologies_

We have two projections, one to get the total balance of your account and one to get the current month balance. The synchronization of the read models with the event table which is the source of truth relies on an Event Driven Architecture. We react to writing events in the event table to trigger updates of the read models.

Now let's see how to implement it with AWS serverless technologies.

# Plan

- **EVENT DB**
  - source of truth of your application data in which you store every event that occurs in your account.
  - Use DynamoDB to store credit/debit events
    - pk=idUser, sk='CreditEvent'|'DebitEvent', amount=number
  - Version is useful when updating projections that will be dealt later in the article.
- **COMMAND**
  - API Gateway HTTP (simple vs. API Gateway REST qui est plus compliquée et plus chère)
  - Lmabda that is triggered by http route to store an event in db
    - one route/lambda to store a credit event
    - one route/lambda to store a debit event
- **EDA**
  - The pain point to do event sourcing is to have reliable reading data, consistency compared to the source of truth. That's why we use an EDA with a fanout to do so.
  - **FANOUT**
    - Fanout that pushes the event to EventBridge (avoid infinite retries and only 2 lambdas plugged to DynamoDB streams). Dispatch a generic event that contains the payload and the type of it.
  - **DISPATCHER**
    - state carried event
      - compute the aggregate
        - list of events on 3 months
        - compute total
        - compute current month
        - aggregate: { totalBalance: x, currentMonthBalance: y }
      - extract the type of event to send from the generic event
        - credit or debit
      - create an event according to its type, we attach the aggregate and the payload of the stored event
        - creditEvent with the aggregate computed and the amount of N
      - dispatch it through event bridge.
    - projectors listen to events affecting data projected in the view model
- **PROJECTION**
  - **DB**
    - need to have read models
    - DynamoDB to store data projections of a job entity for a view in the application. Need total balance on a page and current month balance on another one of the user profile.
      - pk: id user / sk: TotalBalanceForProfile / amount: number
      - pk: id user / sk: CurrentMonthBalanceForProfile / amount: number
    - Use of lastKnownVersion explained later in the Replay section
  - **PROJECTOR**
    - one lambda function that listens to all events that will affect the projection
    - Both TotalBalanceForProfile & CurrentMonthBalanceForProfile depend on credit and debit events.
    - lambda reacts to credit and debit event
    - most recent data is in aggregate
      - extract it from the event coming because the event used are state carried events
    - puts new data
    - code example of the put to update projection with dynamodb toolbox (simplified without the version verification)
- **REPLAY**
  - Why replay event version and proj lastKnowVersion
    - events table needs version
    - db proj needs lastKnownVersion
  - and that's what makes it prod proof
- **LIMITS**

  - A lot of setup
    - not necessary for small applications
  - CRUD is a viable solution
    - if your applications doesn't require giving access to history of events to the users
    - have an history of what have been done just to monitor is possible :
      - logs
      - versioning with dynamodb
  - other limits ??
