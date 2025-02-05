---
title: ASP.NET Core
sidebar_order: 2
---

Sentry has an integration with _ASP.NET Core_ through the [Sentry.AspNetCore NuGet package](https://www.nuget.org/packages/Sentry.AspNetCore).

## Installation

Using package manager:

```powershell
Install-Package Sentry.AspNetCore -Version {% sdk_version sentry.dotnet.aspnetcore %}
```

Or using the .NET Core CLI:

```sh
dotnet add package Sentry.AspNetCore -v {% sdk_version sentry.dotnet.aspnetcore %}
```

This package extends [Sentry.Extensions.Logging]({%- link _documentation/platforms/dotnet/microsoft-extensions-logging.md -%}). This means that besides the ASP.NET Core related features, through this package you'll also get access to all the framework's logging integration and also the features available in the main [Sentry]({%- link _documentation/platforms/dotnet/index.md -%}) SDK.

## Overview of the features

* Easy ASP.NET Core integration, single line: `UseSentry`
* Captures unhandled exceptions in the middleware pipeline
* Captures exceptions handled by the framework `UseExceptionHandler` and Error page display
* Captures process-wide unhandled exceptions (AppDomain)
* Captures `LogError` or `LogCritical`
* Any event sent will include relevant application log messages
* RequestId as tag
* URL as tag
* Environment is automatically set (IHostingEnvironment)
* Release automatically set (`AssemblyInformationalVersionAttribute`, `AssemblyVersion` or environment variable)
* Request payload can be captured if opt-in
* Support for EventProcessors registered with DI
* Support for ExceptionProcessors registered with DI
* Supports configuration system (bind to `SentryAspNetCoreOptions`)
* Server OS info sent
* Server Runtime info sent
* Request headers sent
* HTTP Proxy configuration
* Request body compressed
* Event flooding protection (429 retry-after and internal bound queue)
* Assembly Strong named
* Tested on Windows, Linux and macOS
* Tested on .NET Core, .NET Framework and Mono

### Configuration

The simplest way to configure the `Sentry.AspNetCore` package is not by calling `SentrySdk.Init` directly but rather using the extension method `UseSentry` to the `WebHostBuilder` in combination with the framework's configuration system.
The configuration values bind to `SentryAspNetCoreOptions` which extends `SentryLoggingOptions` from the `Sentry.Extensions.Logging` and further extends `SentryOptions` from the `Sentry` package.

This means all options for the inner packages are available to be configured through the configuration system.

When using `WebHost.CreateDefaultBuilder`, the framework automatically loads `appsettings.json` and environment variables, binding to `SentryAspNetCoreOptions`.

For example:

```json
  "Sentry": {
    "Dsn": "___PUBLIC_DSN___",
    "IncludeRequestPayload": true,
    "SendDefaultPii": true,
    "MinimumBreadcrumbLevel": "Debug",
    "MinimumEventLevel": "Warning",
    "AttachStackTrace": true,
    "Debug": true,
    "DiagnosticsLevel": "Error"
  },
```

> An example of some of the options that can be configured via `appsettings.json`.

It's also possible to bind properties to the SDK via environment variables, like:

Windows
```sh
set Sentry__Debug=true
```

Linux or macOS
```sh
export Sentry__Debug=true
```

ASP.NET Core will automatically read this environment variable and bind it to the SDK configuration object.

When configuring the SDK via the frameworks configuration system, it's possible to add the SDK by simply calling `UseSentry` without providing any further information:

ASP.NET Core 2.x:

```csharp
public static IWebHost BuildWebHost(string[] args) =>
    WebHost.CreateDefaultBuilder(args)
        // Add the following line:
        .UseSentry()
```

ASP.NET Core 3.0:

```csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
        {
            // Add the following line:
            webBuilder.UseSentry();
        });
```

Your DSN would be location if it was defined in `appsettings.json` or any other form of configuration.

Some of the settings require actual code. For those, like the `BeforeSend` callback you can simply:

```csharp
.UseSentry(options =>
{
    options.BeforeSend = @event =>
    {
        // Never report server names
        @event.ServerName = null;
        return @event;
    };
})
```

> Example modifying all events before they are sent to avoid server names being reported.

## Dependency Injection

Much of the behavior of the ASP.NET Core integration with Sentry can be customized by using the frameworks dependency injection system. That is done by registering your own implementation of some of the exposed abstraction.

### Lifetimes

The lifetime used will be respected by the SDK. For example, when registering a `Transient` dependency, a new instance of the processor is created for each event sent out to Sentry. This allows the use of non thread-safe event processors.

### Capturing the affected user

When opting-in to [SendDefaultPii](#senddefaultpii), the SDK will automatically read the user from the request by inspecting `HttpContext.User`. Default claim values like `NameIdentifier` for the _Id_ will be used. 

If you wish to change the behavior of how to read the user from the request, you can register a new `IUserFactory` into the container:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton<IUserFactory, MyUserFactory>();
}
```

### Adding event and exception processors

Event processors and exception processors can be added via DI:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddTransient<ISentryEventProcessor, MyEventProcessor>();
    services.AddScoped<ISentryEventExceptionProcessor, MyExceptionProcessor>();
}
```

## Options and Initialization

As previously mentioned, this package is a wrapper around [Sentry.Extensions.Logging]({%- link _documentation/platforms/dotnet/microsoft-extensions-logging.md -%}) and [Sentry]({%- link _documentation/platforms/dotnet/index.md -%}). Please refer to the documentation of these packages to get the options that are defined at those levels.

Below, the options that are specific to `Sentry.AspNetCore` will be described.

### SendDefaultPii

Although this setting is part of the [Sentry]({%- link _documentation/platforms/dotnet/index.md -%}) package, in the context of ASP.NET Core, it means reporting the user by reading the frameworks `HttpContext.User`. The strategy to create the `SentryUser` can be customized. Please read [retrieving user info](#capturing-the-affected-user) for more.

### Environment

The environment name is automatically populated by reading the frameworks `IHostingEnvironment` value. 

> Note: This option is part of the [Sentry]({%- link _documentation/platforms/dotnet/index.md -%}) package. The value of `IHostingEnvironment` will only be used if **no other method was used**. 

Methods that take precedence over `IHostingEnvironment` are:

* Programatically: `options.Environment`
* Environment variable _SENTRY_ENVIRONMENT_
* Configuration system like `appsettings.json`



### IncludeRequestPayload

Opt-in to send the request body to Sentry. By enabling this flag, all requests will have the `EnableRewind` method invoked. This is done so that the request data can be read at a later point in case an error happens while processing the request.

### IncludeActivityData

Opt-in to capture values from [System.Diagnostic.Activity](https://github.com/dotnet/corefx/blob/master/src/System.Diagnostics.DiagnosticSource/src/ActivityUserGuide.md) if one exists.

### Samples

* A [simple example](https://github.com/getsentry/sentry-dotnet/tree/master/samples/Sentry.Samples.AspNetCore.Basic) without MVC.
* An [example](https://github.com/getsentry/sentry-dotnet/tree/master/samples/Sentry.Samples.AspNetCore.Mvc) using MVC and most of the SDK features.
* For [more samples](https://github.com/getsentry/sentry-dotnet-samples) of the .NET SDKs
