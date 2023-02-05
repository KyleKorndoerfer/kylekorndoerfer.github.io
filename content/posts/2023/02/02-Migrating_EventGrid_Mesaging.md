---
title: Migrating to Azure.Messaging.EventGrid
date: 2023-02-02
tags:
  - C#
  - Azure
  - EventGrid
summary: There are a few gotchas when migrating from Microsoft.Azure.EventGrid to the Azure.Messaging.EventGrid package.
draft: false
---

Recently we started updating an Azure Functions project that processes dead letter
messages from EventGrid that couldn't be delivered to our Azure Storage Queues. As
part of the upgrade we migrated from the deprecated `Microsoft.Azure.EventGrid`
library to the `Azure.Messaging.EventGrid` library and came across a few differences
that required somce changes to our code.

## Getting the EventGridEvents from the blob trigger

Previously, we used the `Newtonsoft.JSON` library to deserialize the data passed
to the blob trigger as follows:

```csharp
[FunctionName("DeadLetterProcessing")]
public void Run(
		[BlobTrigger(_blobPath, Connection = _conn)] string myBlob,
		string name,
		ILogger log)
{
    EventGridEvent[] _events = JsonConvert.DeserializeObject<EventGridEvent>(myBlob);

	// etc.
}
```

However, using the new library the `EventGridEvent` type no longer has a default
constructor that allows you to deserialize the JSON data directly. Instead, you
need to use static methods on the `EventGridEvent` type to process the data as
follows:

- `EventGridEvent.Parse()` for a single event
- `EventGridEvent.ParseMany()` for multiple events

When processing dead letter events from EventGrid, it will _always_ be an array
of at least 1 event, so we will need to use the `ParseManu()` method.

We will also need to use the static methods on the `BinaryData` type to process
the data received by the blob trigger into a `BinaryData` type.:

- `BinaryData.FromString()` if you used the `string` binding option on the blob trigger
- `BinaryData.FromStream()` if you used the `Stream` binding option on the blo trigger

```csharp
[FunctionName("DeadLetterProcessing")]
public void Run(
        [BlobTrigger(_blobPath, Connection = _conn)]Stream myBlob,
        string name,
        ILogger log)
{
    EventGridEvent[] _events = EventGridEvent.ParseMany(BinaryData.FromStream(myBlob));
}
```

## Getting the custom data from the `EventGridEvent.Data` property

The next issue we encountered is trying to deserialize the custom data stored in
the `Data` property of the `EventGridEvent`. Again, we used to just deserialize
the `Data` property directly but this is no longer an object type, but rather a
`BinaryData` type.

This means that we need to use the `.ToObjectFromJson<T>()` method of the
`BinaryData` type to process the custom data property into our custom data object.

```csharp
EventGridEvent[] _events = EventGridEvent.ParseMany(BinaryData.FromString(myBlob));

foreach (EventGridEvent eventGridEvent in _events)
{
    CustomEventGridData data = eventGridEvent.Data.ToObjectFromJson<CustomEventGridData>();

    // use you data as needed
}
```

Of particular note, the new library makes use of `System.Text.Json` to serialize
the custom data. This means that you need to switch to using the `JsonPropertyName`
attribute on your data classes in order to deserialize the data properly.

Hence our custom data object need to be updated as follows:

```csharp
using System.Text.Json.Serialization;

public class CustomEventGridData
{
    [JsonPropertyName("orderNumber")]
    public int OrderNumber { get; set; }

    [JsonPropertyName("orderTotal")]
    public int OrderTotal { get; set; }
}
```

# Wrap up

None of this was hard, but we needed to pay closer attention to the changes
described in [Migration Guide]( https://aka.ms/azsdk/net/migrate/eg) that was
linked from the deprecation notice on the old `Microsoft.Azure.EventGrid` library.
However, the portion that applied to our situation was a couple of small paragraphs
near the end of the document while the rest seemed more focused on the support
for the `CloudEvent` type from `Azure.Core`... a standardized event type from the
[Cloud Native Computing Foundation](https://cncf.io/projects/cloudevents).
Details of the specification can be found at [cloudevents.io](https://cloudevents.io).

Hopefully this helps someone else switching to the new library find what they are
looking for quicker!