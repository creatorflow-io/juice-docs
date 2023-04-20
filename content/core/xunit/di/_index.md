---
title: "Dependency Injection"
date: 2023-04-11T16:57:00+07:00
draft: false
weight: 1
---

To run complex unit tests with a full dependency injection pattern, we provide  a simple **DependencyResolver**.

```csharp {linenos=false,hl_lines=[1],linenostart=1}
    using Juice.Extensions.DependencyInjection;
    ...

    var resolver = new DependencyResolver
    {
        CurrentDirectory = AppContext.BaseDirectory
    };

    resolver.ConfigureServices(services =>
    {

        var configService = services.BuildServiceProvider().GetRequiredService<IConfigurationService>();

        // IConfigurationRoot
        var configuration = configService.GetConfiguration(args);

        // configure options
        // register services
    });

    // access IServiceProvider
    var serviceProvider = resolver.ServiceProvider;

```

The library can be accessed via Nuget:
- [Juice](https://www.nuget.org/packages/Juice)