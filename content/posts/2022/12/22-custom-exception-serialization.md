---
title: Custom Exceptions and Serialization
date: 2022-12-22
tags:
  - C#
  - .NET
  - Azure
summary: Creating your own custom exceptions helps convey more meaning and information relavant to your application.
draft: false
---

Custom exceptions types can be very helpful... if implemented correctly!

## Creating Custom Exceptions

While it is always good to use the many built in Exception types (such as
`ArgumentNullException`), it is always a good practice to create your own custom
exceptions in order to convey more meaning and information relavant to your application.

On the surface this is very simple but, if you aren't careful, there can be hidden
gotchas that can eventually trip you up as it did on my team recently. The
following is an example based on what we discovered.

## Basics

The basics for creating a custom exception class can easily be found in the
[Microsoft documentation here](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/exceptions/creating-and-throwing-exceptions#defining-exception-classes).
Basically, your custom exception should inherit from `Exception` and implement 3
different constructors:

```csharp
public class InvalidProductCodeException : Exception
{
	public InvalidProductCodeException() : base() {}
	public InvalidProductCodeException(string message) : base(message) {}
	public InvalidProductCodeException(string message, Exception inner) : base(message, inner) {}
}
```

Thats it! You now have a custom exception that works great if all you need is to
key off of the type of exception being thrown. However, in most cases you need
to include some additional information with your exception so you can make more
informed decisions about what to do when it occurs.

The solution is obvious... add a property and a constructor that initializes that property!

```csharp {hl_lines=[5,8]}
// code from above omitted for brevity

public InvalidProductCodeException(string message, Exceptions inner, string code) : base(message, inner)
{
	ProductCode = code;
}

public string ProductCode { get; private set; }
```

## Making the exception serializable

However, this is only part of the solution because exceptions need to be serializeable
in order to work across domain and remoting boundaries. So, you need to mark your
class as serializable and add a protected constructor that gets called by the serializer
as follows:

```C#
[Serializable]
public class InvalidProductCodeException : Exception
{
    // code from above omitted for brevity

    protected InvalidProductCodeException(SerializationInfo info, StreamingContext context)
    {
	    // Add implementation
    }
}
```

This is what we had for quite some time that was happily working with our API service.
When this exception was thrown we were able to provide the details of it back to
our clients letting them know exactly what the problem was and which product code
was invalid. All good... or so we thought! (I'm sure some of you have already spotted
the issue)

## Encountering the issue

After many years of this working, we decided to refactor some of our code into
an [Azure Durable Function](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview?tabs=csharp)
because it was geting more complex and we wanted the procesing to be more asynchronous.
So we had a Durable Orchestrator that called out to several Activities to perform
the work.

What we noticed was that if the code inside of the Activity threw our
`InvalidProductCodeException`, exception, when we tried to access the `ProductCode`
property inside of the catch block in the Orchestrator it was always `null`.

After using the [Durable Function Monitor](https://marketplace.visualstudio.com/items?itemName=DurableFunctionsMonitor.durablefunctionsmonitor)
plugin to VS Code to inspect the state of the Orchestrator instance, we saw that
the return value from the Activity didn't even have the `ProductCode` property
in it.

At this point we looked closer at our custom exception and realized that while we
 marked the custom exception as serializable and added the protected constructor,
 we never provided an implementation so the custom exception properties were not
 getting deserialized back into the excpetion!

What happens with Durable Funtions is that the Orchestrator and Activities are
essentially different processes. After an activity is done executing, it writes
its state into the instance hub and exits. The Durable Function Runtime is notified
that this has happened and restarts the Orchestrator that kicked off the Activity
and runs up to the point that it last executed (re-entrant), picking up the
serialized state/outcome from the Activity and deserializes it. Since we didn't
implement the serialization for the custom properties in our exception class,
it didn't know what to do with the `ProductCode` property so it ignored it.

## The solution

The solution was fairly straight forward... fully implement serialization properly
by providing a body to the protected constructor for serialization and then
override the `GetObjectData()` method to be aware of the custom properties.

THe completed, corrected, custom exception class is as follows:

```C#
using System.Runtime.Serialization;

[Serializable]
public class IvalidProductCodeException : Exception
{
	public InvalidProductCodeException() : base() {}
	public InvalidProductCodeException(string message) : base(message) {}
	public InvalidProductCodeException(string message, Exception inner) : base(message, inner) {}

	public InvalidProductCodeException(string message, Exceptions inner, string code)
			: base(message, inner)
	{
		ProductCode = code;
	}

	protected InvalidProductCodeException(SerializationInfo info, StreamingContext context)
	{
		ProductCode = info.GetString("ProductCode");
	}

	public override void GetObjectData(SerializationInfo info, StreamingContext context)
	{
		if (info == null)
		{
			throw new ArgumentNullException(nameof(info));
		}

		info.AddValue("ProductCode", ProductCode);

		// MUST call through to the base class to let it save its own state.
		base.GetObjectData(info, context);
	}

	public string ProductCode { get; private set; }
}
```

Hope this helps!