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

You surely know git and it might be the best example of event-sourcing project that every developer know of. It stores every modifications you made in your code into events that are called commits, then the developer has access to the whole history of changes. If you want to implement a project that needs to access the history of what happened throughout the life of the user of even the whole application, then you are in the right place ! My 2 years learnings and the serverless paradigm will help you produce the most stable and the less painful solution you can think of.

CQRS (Command and Query Responsibility Segregation) is a pattern that separates read and update operations for a data store. It's a good thing to grasp a bit what it is to understand better what will be approached in this article, but don't worry it is not required as it is quite straightforward. Still multiple resources already deal with this topic that you can easily find. Event sourcing is a way to implement the data store in a CQRS pattern.

Present a schema of archi with blocks without specifying technologies. Say that we will detail on each part of the architecture and specify the technologies involved.

Plan : I don't have to split in two the plan because people don't want the path that lead to it but the final result, the answer.

# Plan

- Let's build together this architecture
  - **COMMAND & EVENT DB** : A command equals one lambda that stores an event
    - store event needs sometimes to have information about the user and as the source of truth id the event DB, you need a method that computes the current data from all events (aggregate) => big reducer
  - **FANOUT** : Fanout that pushes the event to EventBridge (avoid infite retries and only 2 lambdas pluged to DynamoDB streams) => event notification
  - **PROJECTION** : Lambda to project
    - one lambda per event and that project on every projection (vertical projection) => not a good idea because one lambda does multiple things and you don't take advantage of the event driven architecture because every event is listened by only one lambda
    - one lambda per projection that listens to all event that contains data used in that projection (horizontal projection) : need to use a big switch to differ the cases. As every event is different your lambda will do only one thing but the complexity of the code of your lambda will be proportional to the number of events that trigger it.
    - another solution is to have a lambda for every couple couple event/projection. But again there is a pb because the number of lambda allowed in one stack is limited.
  - **WHY REPLAY** : Architecture looks like _this_ and it works fine.
    - but you find yourself resetting all data every time you modify or add a projection.
    - That's one thing you can't do in production
    - Replay : multiple solutions were tried to tackle this issue
      - pb of order in event bridge: need last known version
      - **do i need to present them quickly ?**
- What about an architecture with stable production, modifications resilient, easier DevX => the dream
  - **DISPATCHER** : the solution found to solve all previous issues about simplicity and order in replay and about code complexity in projection lambda. Explanations about how it works. => state carried events (vs. notification)
  - **REPLAY SOLVED** : now two solutions
    - lambda / step functions => send only last event
    - event bridge archive
  - **EASIER DEVX** : projection easy => only one put operation extracting the need information from the aggregate
    - one lambda per projection
    - one put identical for every event because based on aggregate spread inside event
  - **AGGREGATE** : Aggregate concentrate all job logic _(logique métier ?)_
    - needs to be strongly tested
- **LIMITS** : Limits of that architecture
  - A lot of setup
    - not necessary for small applications
  - CRUD is a viable solution
    - if your applications doesn't require giving access to history of events to the users
    - have an history of what have been done just to monitor is possible :
      - logs
      - versioning with dynamodb
  - other limits ??
