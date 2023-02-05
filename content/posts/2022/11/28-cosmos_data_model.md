---
title: CosmosDB Data Models & Partioning
date: 2022-11-28
tags:
  - Azure
  - CosmosDB
summary: How you structure your data models when working with CosmosDB can make future adjustments much easier.
draft: false
---

A little bit of thought and effort up front can save you a lot of hassel later.

## The importance of data models

Every document in CosmosDB is required to have two properties: an `id` and a partition
key of your choosing. Usually, defining the `id` property is a simple task... it
is the order number or a generated unqiue identifier (UUID/GUID). However, choosing
a partition key is often a lot harder.

Partition keys are how CosmosDB gains the ability to scale to massive amounts of
data while still being blazingly fast at retrieving specific documents (point
reads). A well choosen partition key balances between speed of access and your
ability to scale nearly infintely. If you choose too narrow of a partition key,
then you can have 'hot partition' issues where the individual logical partition
becomes your bottleneck, even if you have plenty of capacity. This is like having
a room with 10 doors, but almost everyone is trying to leave through one door at
the same time... you can't all fit so you must take turns which slows everything
down. Also, an individual partition key is limited to just 20GB of data.

Importantly, the `id` and partition key properties must be defined at that moment
you create your collection. If you choose the wrong property on your document when
getting started, then you will need to create an entirely new collection using a
new property for the partition key and then migrate your data to the new collection.

With so much riding on choosing the right value for your property key, you will
most certainly not have all of the information you need when getting started and
will make a mistake. However, if you plan for this possibility up-front you can
help mitigate potential issues down the road.

## The scenario

You are modeling the online orders for a pizza chain to be stored in CosmosDB.
After some initial discussion, you decided that your are going to use the name
of the city as the partition key so your data model looks something like this:

``` csharp
public class Order
{
	// Unique id assigned to the order
	public int Id { get; set; }

	// the city the store is located in
	public string City { get; set; }

	// other properties for the order like Total, DateOrdered, etc.
}
```

Initially, this model served you quite well. You did so well in fact, that after
several months you start to offer franchise opportunities. After expanding into
a few different states, you suddenly run into the situation where you have a
duplicate city name (which is actually quite common; see Rochester or Springfield).

Because of the data model defined above, you now have no choice but to create a
new collection that uses a different property for the partition key so that you
no longer have a collision on the `city` property. But what should we use?

## Future proofing the data model

A best practice for creating a CosmosDB data model is to use a property that
wraps one or more of the properties in your data model as the partition the
partition key. By using a wrapper peroperty, you give yourself the ability to
adjust the partition key at any time, in place, without having to migrate your
data to a new container (possibly resulting in downtime for the migration).

For eaxmple, if we had defined our intial data model like this:

``` csharp
public class Order
{
	// Unique id assigned to the order
	public int Id { get; set; }

	// partition key that wraps one or more properties
	public string PartitionKey { get { return City; } }

	// the city the store is located in
	public string City { get; set; }

	// other properties for the order like Total, DateOrdered, etc.
}
```

Then we could easily update the `PartitionKey` property to use a different value
without having to create a new collection and migrate our data. The *migration*
can happen in place with now downtime.

Fo example, we could update the partition key property as follows:

``` csharp
public string PartitionKey { get { return $"{City}_{State}"; } }
```

This update to the data model would **expand** the range of key values from just
cities to the combination of cities **and** states, allowing for way more partitions.
This also has the advantage of keeping the partition key easy to remember (you
will almost always know the city and state for a given store/order) which allows
you to limit the number of cross-partition queries that you run keeping your costs
lower.