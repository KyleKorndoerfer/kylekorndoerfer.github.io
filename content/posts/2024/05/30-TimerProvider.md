---
title: Testing around DateTime.Now
date: 2024-05-30
tags:
  - .NET
  - C#
  - Testing
summary: Testing around DateTime.Now is now native in .NET 8!
draft: false
---

Spend any time writing code and unit tests and you'll quickly learn to hate
using `DateTime.Now` and `DateTime.UtcNow` in your code because it makes testing
a bear, resulting in creating the same boiler plate code in every project. Well,
starting with .NET 8 you can kiss that boilerplate code goodbye!

## What's wrong with using `DateTime.Now`?

The `DateTime` type is a read-only struct and the `Now` property is only a getter.
Because of this, we cannot use traditional mocking approaches to set a fake value
for the return value of `Now`.

Up until now the most common approach, highlighted in many a book/article/course
on unit testing, was to create a wrapper class that you would use through dependency 
injection instead of directly calling `DateTime.Now`. For example:

```csharp
public interface IProvideTime
{
    public DateTime Now { get; }
    public DateTime GetUtcNow { get; }
}

public class CusomtTimeProvider : IProvideTime
{
    public DateTime Now => DateTime.Now;
    public DateTime UtcNow => DateTime.UtcNow;
}

// register during startup
builder.Services.AddSingleton<IProvideTime, CustomTimeProvider>();

// and inject into any classes that need the current DateTime
public class MessengerService(IProvideTime timeProvider)
{
    public void SendMessage(string message)
    {
        DateTime messageSent = timeProvider.UtcNow;
        // now use it...
    }
}
```

This allowed us to write unit tests for any code that needs to grab the current
moment in time in order to make decisions, like the price of an item before, 
during, and after a sale.

```csharp
// unit testing code
ITimeProvider _fakeTimeProvider = Substitute.For<IProvideTime>();
_fakeTimeProvider.Now.Returns(new DateTime(2024, 05, 01, 15, 0, 0));

MessengerService svc = new MessengerService(_fakeTimeProvider);
svc.SendMessage("Hello!");
```

Almost **_everyone_** needed to write this type of wrapper for almost **_every_**
project that they worked on. Even though it is very simple, it was so common that
it always felt like this should have been baked into the framework a long time ago.

## Enter the `TimeProvider` class in .NET 8!

Instead of injecting a custom interface into our classes, we can now inject a
`TimeProvider` instance and use either the `GetLocalNow()` or `GetUtcNow()` methods
to return either the local date/time or UTC date/time respectively.

Our example above now becomes:

```csharp
// register during startup
builder.Services.AddSingleton(TimeProvider.System);

// and inject into any classes that need the current DateTime
public class MessengerService(IProvideTime timeProvider)
{
    public void SendMessage(string message)
    {
        DateTime messageSent = timeProvider.UtcNow;
        // now use it...
    }
}
```

When it comes to the unit tests, this also becomes much simpler because Microsoft
has provided a `FakeTimeProvider` class that inherits from the `TimeProvider` class.
Now you just need to call the `SetUtcNow` method to set the DateTime value that
you need for your test

```csharp
// code
public class MessengerService(TimeProvider timeProvider)
{
    public void SendMessage(string message)
    {
        DateTime messageSent = timeProvider.GetUtcNow();  // .GetLocalNow()
        // now use it...
    }
}

// startup
builder.Services.AddSingleton(TimeProvider.System);

// unit test code
FakeTimeProvider _timeProvider = new FakeTimeProvider();
_timeProvider.SetUtcNow(new DateTime(2024, 05, 01, 15, 0, 0));

MessengerService svc = new MessengerService(_fakeTimeProvider);
svc.SendMessage("Hello!")
```

Finally, no more boilerplate code needed in every new project!