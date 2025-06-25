---
title: "Logging"
date: 2023-04-03T20:22:47+07:00
draft: false
weight: 3
version: 7.0.2
linktitle: "v7.0.2"
---
{{< versions doc=core_logging >}}
There are many great logging frameworks for .Net Core like Log4Net, SeriLog, NLog...
However, if you wish to customize logging system for special purpose, 
then you can inherit the *LoggerProvider* abstract class:

```csharp {linenos=false,linenostart=1}
    public abstract class LoggerProvider : IDisposable,
        ILoggerProvider, ISupportExternalScope
    {
        ...

        public IExternalScopeProvider ScopeProvider { get; }

        /// <summary>
        /// Writes the specified log information to a log file.
        /// </summary>
        public abstract void WriteLog<TState>(LogEntry<TState> entry, string formattedMessage);

        #region IDisposable Support

        /// <summary>
        /// Cleanup resources on disposing.
        /// </summary>
        protected abstract void Cleanup();

        ...

        #endregion
    }
```

Once you have an implementation, you can register it when you add logging:

```csharp {linenos=false,hl_lines=[10],linenostart=1}
    services.AddLogging(builder =>
        {
            builder.ClearProviders(); // optional
            builder.AddConfiguration();

            // configure your options if needed

            // register service
            builder.Services.TryAddEnumerable(
                ServiceDescriptor.Singleton<ILoggerProvider, YourLoggerProvider>());
        });

```

Or register to existing application logging builder:

```csharp {linenos=false,hl_lines=[6],linenostart=1}
    var builder = WebApplication.CreateBuilder(args);
    // configure your options if needed

    // register service
    builder.Logging.Services.TryAddEnumerable(
            ServiceDescriptor.Singleton<ILoggerProvider, YourLoggerProvider>());
```


See [Background service logging]({{<ref "bg_service/logging/_index.md" >}}) and [XUnit output logging]({{<ref "core/xunit/logging/_index.md" >}}) for example.

The library can be accessed via Nuget:
- [Juice.Extensions.Logging](https://www.nuget.org/packages/Juice.Extensions.Logging)
