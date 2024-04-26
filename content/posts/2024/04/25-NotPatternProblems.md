---
title: C# not pattern pitfall 
date: 2024-04-25
tags:
  - .NET
  - C#
  - Pattern
summary: Be careful when using C# Patterns with the not keyword
draft: true
---

Using the pattern capabilities in C# can make code more readable and easier
to understand but can come with some pitfalls that could catch you off guard
if you are not careful.

## Pattern power

Patterns can be especially useful when used with Enumerations. For example,
given the following enum:

```csharp
enum OrderStatus
{
    Placed,
    Processing,
    Shipped,
    Delivered
}

OrderStatus orderStatus = OrderStatus.Placed;
```

if you wanted to check if the order status was one of two potential values
without using patterns you would typically do this:

```csharp
if (orderStatus == OrderStatus.Placed || orderStatus == OrderStatus.Processing)
{
    // do something
}        
```

but with a pattern, you don't need to repeat the variable that you are checking:

```csharp
if (orderStatus is OrderStatus.Placed or OrderStatus.Processing)
{
    // do something
}
```

This expression is shorter in length and follows a more natural way of reading.
But what if you wanted to check if the order status was **not** one of two 
different values?

## With great power...

On the surface, you would right it like this, right?:

```csharp
if (orderStatus is not OrderStatus.Shipped or OrderStatus.Delivered)
{
    // do something
}
```

This compiles and appears to be correct. Wouldn't you naturally assume that you
would enter the body of the `if` block when the order status _is not_ Shipped or
Delivered? If you run this code with `orderStatus = OrderStatus.Shipped` you 
will skip the body of the if statement... as expected. However, if you set 
`orderStatus = OrderStatus.Shipped` you _**will**_ enter the body of the `if` 
block!! But why?

## ... comes great responsibility

The reason is that the C# compiler is actually interpreting the `if` expression as
if it had been written like this (which can be confirmed with unit tests):

```csharp
if (orderStatus is not OrderStatus.Shipped || orderStatus is OrderStatus.Delivered)
{
    // do something
}    
```

now that you understand that, your first few thoughts might be to try and mess
with  the combinations of the `is not` and `and`/`or` keywords but the only
syntax that is going to be valid is to group the conditions being checked inside
of parentheses as follows:

```csharp
if (orderStatus is not (OrderStatus.Shipped or OrderStatus.Delivered))
{
    // do something
}
```

Once it is written this way, you will only enter the body of the `if` block when
the order status _**is not**_ equal to `OrderStatus.Shipped` or 
`OrderStatus.Delivered`.

## Wrap up

This is something that Visual Studio will not currently alert you to unless you
have the [JetBrains ReSharper plugin][1] installed. Fortunately, I was writing
code like this using [JetBrains Rider][2] which flagged this issue with the
warning of

> _The pattern is redundant, it does not produce any runtime checks_

which led me to the description and solution for this problem in the 
[JetBrains Rider documentation][3].

<!-- links -->
[1]: <https://www.jetbrains.com/resharper> "ReSharper Visual Studio Plugin"
[2]: <https://www.jetbrains.com/rider> "Cross-platform .NET Editor"
[3]: <https://www.jetbrains.com/help/resharper/PatternIsRedundant.html> "Pattern is Redundant"