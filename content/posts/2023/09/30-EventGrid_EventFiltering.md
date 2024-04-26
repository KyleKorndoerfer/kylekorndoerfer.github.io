---
title: Using Azure Event Grid Event Filtering Effectively
date: 2023-09-30
tags:
  - Azure
  - Azure Event Grid
summary: Carefully planning your events can bring large benefits later
draft: false
---

Azure Event Grid is a globally distributed, highly scalable Pub/Sub messaging service
that makes building a distributed, event-based system quick and easy. It is integrated
into almost every Azure service allowing you to respond to almost anything that happpens
in Azure. In order to properly leverage this capability and evolve your applications
effectively over time, you need to plan how to filter your messages appropriately.

## Event Filtering

At it's most basic Event Grid functions as a messaging clearinghouse... a mediator 
between two or more systems in order to exchange messages without being directly
linked to each other. When your application is small and the number of publishers
is small this works fine, but as your application grows and the number of publishers
grow you need an effective means of only responding to messages that you care about.

This is where **Event Filtering** becomes crucial and there are two main ways this can
be accomplished. But first, we need to understand the anatomy of an Event Grid message.

### Event Grid Events

The format for an Event Grid Event is as follows (represented as JSON with required properties highlighted):

```json {hl_lines=[4,5,6,7]}
[
  {
    "topic": string,
    "subject": string,
    "id": string,
    "eventType": string,
    "eventTime": string,
    "data": {
      // json data unique to each message type
    },
    "dataVersion": string,
    "metadataVersion": string
  }
]
```

The `subject` & `eventType` properties are what allows you to properly filter
messages down to what your subscriber cares about when you are creating your 
Event Grid subscription.

### Subject filtering

As you could already guess, **subject filters** allows you to filter incoming messages
by looking at the subject field. This is purely a string feild, so you can use any
value that you would like. However, in order to avoid different components in your
system from publishing events with the same subject, you should establish a convention
that prevents this from happening.

As mentioned above, almost every Azure service publishes events that you can subscribe
to so you can take action when those events occur (like a blob being uploaded or modified).
If we were to simply use `Modified` as the subject, how would we know that this event
came from a blob storage container or an entity being updated in CosmosDB?

The convention used by Microsoft for their events is to use a series of tokens 
separated by `/` characters that get more specific the farther right you move. For
example:

> `/blobServices/default/containers/{containername}/blobs/{blobname}`

This subject starts with a broad token that indicates this event came from `blobServices`
and then drills down through the containers to a specific blob the farther right we move.

When filtering on the subject property, you can filter by a starting value, ending
value, or both utilizing `subjectStartsWith` and `subjectEndsWith`.

Continuing with the storage account example above, we could choose to only receive
messages for events where the subject ends in `.jpg` or messages where the subject
begins with `/blobServices/default/containers/profilePictures` for events that 
happened in the __profilePictures__ container (regardless of file type).

When looking to determine a convention for your own subjects, try to build a path
from least speficic to most specific to allow varying levels of specificity.

### Event Types

Using the `eventType` property, we can choose to filter on certain _types_ of messages.
This is usually very helpful since different event tyoes usually have a different 
payload in the `data` property. Again, having a well established convention helps
making filtering more precise.

A simple but very effective strategy is to use the following format, similar to 
namespaces/packages in programming languages:

> `{Company}.{Product}.{Service}.{Action}`

where:
- `Company` is the name of your company to prevent collisions with messages from 
other companies or products you might use.
- `Product` is the top level name of your product. Even if you only have a single 
product currently, your might not in the future.
- `Service` is the individual service within the product that is publishing the message.
This is especially helpful when your application is composed of many different 
microservices. If it isn't, think about functional areas in your product that making
grouping easier.
- `Action` describes the event that is happening, like **InventoryRestock**, or 
**PaymentDeclined**.

Putting these together could result in and eventType similar to:

> Acme.OnlineOrdering.Checkout.OrderPlaced

or

> Acme.OnlineOrdering.Inventory.StockPulled

## Conclusion

By determining a naming convention for both the `subject` and the `eventType`
properties early in the development of your application, you can ensure that as
your application grows so does the ability to control what messages are received
and where they are delivered.


### More resources

[Event-type filtering](https://learn.microsoft.com/en-us/azure/event-grid/event-filtering#event-type-filtering)