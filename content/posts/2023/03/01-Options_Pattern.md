---
title: Using the Options Pattern in .NET
date: 2023-03-01
tags:
  - C#
  - Patterns
summary: The Options pattern is THE way to configure your .NET applications.
draft: false
---

## Intro

The [Options Pattern](https://learn.microsoft.com/en-us/dotnet/core/extensions/options)
is a great way to group related settings for your application that you are probably
already using, but there might be some capabilities and best practices that you might
not be taking advantage of or aware of.

For those that might not be aware of the Options pattern in .NET, you can have a group
of related settings defined in your `AppSettings.json` file like this:

```json
{
  "Storage": {
    "StorageAccountName": "imgstor123",
    "BlobContainer": "thumbnails",
    "Retries": 1
  }
}
```

You can then define a class to represent these settings:

```csharp
public class StorageOptions
{
    public static string SectionName = "Storage";

    public string StorageAccountName { get; init; }
    public string BlobContainer { get; init; }
    public int Retries { get; init; }
}
```

You can then load these settings into this class during application startup while
registering it for dependency injection like this:

```csharp
var config = builder.Configuration();

builder.Services
    .AddOptions<StoprageOptions>()
    .Bind(config.GetSection(StorageOptions.SectionName));
```

And then in your classes, you can simply do this to get access to those settings:

```csharp
public class ImageController
{
    private readonly StorageOptions _storageOptions;

    public ImageController(IOptions<StorageOptions> storageOptions)
    {
        _storageOptions = storageOptions?.Value ?? ArgumentNullException.ThrowIfNull(storageOptions)
    }
}
```

## Validating settings

One thing people might not be aware of is that you can actually validate your
application settings to make sure that the values provided are what you expect them
to be. For instance, you can make sure that settings have values (required), are in a
valid range, an enum value, match a RegEx expression, etc.

You do this by adding validation attributes from the
`System.ComponentModel.DataAnnotations` namespace to the properties in your options
class.

```csharp {hl_lines=[1,7,8,9,12,15]}
using System.ComponentModel.DataAnnotations;

public class StorageOptions
{
	public static string SectionName = "Storage";

	[Required]
	[MinLength(3)]
	[MaxLength(32)]
	public string StorageAccountName { get; init; }

	[Required]
	public string BlobContainer { get; init; }

	[Range(1, 5)]
	public int Retries { get; init; } = 3;
}
```

With these in place, we just need to tell the runtime to validate our settings when
registering the option at startup:

```csharp {hl_lines=[6]}
builder.Services
    .AddOptions<MyOptionsClass>()
    .Bind(config.GetSection(StorageOptions.SectionName))
    .ValidateDataAnnotations();
```

If there is a validation issue with any of your settings, an `OptionsValidationException`
exception will be raised that you can handle. There is a small gotcha here though...
_**validation will only occur when you resolve the Option into it's underlying type**_
using the `.Value` property.

What if you want to validate your settings earlier than that?

## Validating on startup

Have no fear, there is a simple way to trigger validation during application
startup so that you can be sure your application is configured correctly before getting
to far along.

```csharp {hl_lines=[7]}
builder.Services
    .AddOptions<MyOptionsClass>()
    .Bind(config.GetSection(StorageOptions.SectionName))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

## Just the beginning

There is a lot more than meets the eye around the Options pattern in .NET. In addition
to `IOptions<T>`, there is also `IOptionsSnapshot<T>` (for Scoped or Transient
scenarios) and `IOptionsMonitor<T>` that allows for dynamic updates to settings and
selective option invalidation.

Hope that helps!