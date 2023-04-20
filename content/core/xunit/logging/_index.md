---
title: "Logging"
date: 2023-04-04T16:08:17+07:00
weight: 2
---

Sometimes we want to show the app logs in unit tests for easier debugging.
So **TestOutputLoggerProvider** can help us.

```csharp {linenos=false,hl_lines=[1,14,19],linenostart=1}
    using Juice.Extensions.DependencyInjection;
    using Microsoft.Extensions.Logging;
    ...

    var builder = new DependencyResolver();
    builder.ConfigureServices(services =>
    {
        var configService = services.BuildServiceProvider().GetRequiredService<IConfigurationService>();
        var configuration = configService.GetConfiguration();

        services.AddLogging(builder =>
        {
            builder.ClearProviders()
            .AddTestOutputLogger()
            .AddConfiguration(configuration.GetSection("Logging"));
        });

        // must register an instance of ITestOutputHelper
        services.AddSingleton(_output);
        // configure other services
        ...
    });
```

The library can be accessed via Nuget:
- [Juice.XUnit](https://www.nuget.org/packages/Juice.XUnit)