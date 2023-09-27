---
title: "Manageable services"
date: 2023-09-20T17:22:07+07:00
draft: false
weight: 0
---

#### Purposes
- Operate background services remotely (*Start, Stop, Invoke Job...*)
- Load background service as as plugin dynamically
- Monitor service status
- Custom service logging
#### Usage
We provide *ServiceManager* service to manage and operate *IManagedService* instances and Web API to access it.

{{< figure src="/images/services/BackgroundService.svg" title="Background service sequence diagram" >}}

```cs
using Juice.BgService.Management;
using Juice.BgService.Extensions.Logging;
using Juice.Extensions.Options.DependencyInjection;
...

var builder = WebApplication.CreateBuilder(args);

// see Logging section
builder.Logging.AddBgServiceFileLogger(builder.Configuration.GetSection("Logging:File"));

builder.Services.AddBgService(builder.Configuration.GetSection("BackgroundService"))
    .UseFileStore(builder.Configuration.GetSection("File"));

// optional use separated appsettings file for background service file store.
builder.SeparateStoreFile("Store");

var pluginPaths = new string[]
{
    // plugin absolute paths
};

// Optional plugins
builder.Services.AddPlugins(options =>
{
    options.AbsolutePaths = pluginPaths;
    options.ConfigureSharedServices = (services, sp) =>
    {
    };
});

// required for web api
builder.Services.AddControllers();
...
var app = builder.Build();
...
app.MapControllers();

```

#### Implementation
We provide [Juice.BgService.ServiceBase](https://www.nuget.org/packages/Juice.BgService.ServiceBase) that contains a number of abstract services that you can inherit.

1. **BackgroundService**

This abstract service contains only basic methods that can be managed by *ServiceManager*, so you can implement anything in your service by overriding the *ExecuteAsync()* method.

2. **ScheduledService**

This abstract service is processed according to the schedule options, so you can schedule something to be done by overriding the *InvokeAsync()* method.

The *ScheduledServiceOptions* has *Frequencies* options to scheduling a task (we can add one or more *Frequency*):
- *RunOnStartup*: if **false**, you must start service manually.
- *Occurs*: (**Daily, Weekly, Monthly, Once**)
- *Daily*: describes daily schedule if the *Occurs* is set to **Daily**
    + *RecursEvery*: service will recurs after number of **days**
    + *OccursOnceAt*: if specified, the *InvokeAsync* method will be called once at specified time after every *RecursEvery* day(s).
    + *OccursEvery*: if no *OccursOnceAt* is specified, the *InvokeAsync* method will be called after every specified interval (30s by default), in the working time and recurs after every *RecursEvery* day(s).
    + *StartingAt*: describes the start time of work during the day
    + *Duration*: describes working time of the day
- *Weekly*: describes weekly schedule if the *Occurs* is set to **Weekly**. It is almost the same as the *Daily* frequency above but has differences:
    + *RecursEvery*: service will recurs after number of **weeks**
    + *OnDays*: describes the days of the week on which the service will execute
    + *StartOfWeek*: describes the first day of the week
    Ex: If the schedule is { RecursEvery: 2, OnDays: [Mon, Wed, Fri], StartOfWeek: Tue } and today is Saturday of the week of execution, the next execution will be next Monday. After that, the service will await for 2 weeks.
- *Monthly*: describes monthly schedule if the *Occurs* is set to **Monthly**. It is almost the same as the *Daily* frequency above but has more options:
    + *RecursEvery*: service will recurs after number of **months**
    + *OnDay*: service will only be performed on this day of the month (optional)
    + *DayOfWeek*: service will only be performed on this day of the week, combined with *On* to specify the day in month (Ex: the third Friday of month)
    + *On*: (*First, Second, Third, Fourth, Last*) describes the week in month
    + *SpecialDay*: (Day, Weekday, Weekend), combined with *On* to specify the special day in month (Ex: the last weekend of month)

3. **FileWatcherService**

This abstract service will monitor the file changes and invoke these methods:
- OnFileRenamedAsync
- OnFileDeletedAsync
- OnFileReadyAsync

You will override these method to handle yourself.

#### Service repository
We are implemented a default service repository based on configuration. You can call *UseFileStore(configurationSection)* to use it, or implement a repository yourself.

The sample source code is available on [github](https://github.com/creatorflow-io/Juice/tree/master/services/bgservice/test/).

The library can be accessed via Nuget:
- [Juice.BgService](https://www.nuget.org/packages/Juice.BgService)
- [Juice.BgService.ServiceBase](https://www.nuget.org/packages/Juice.BgService.ServiceBase)
- [Juice.BgService.Api](https://www.nuget.org/packages/Juice.BgService.Api)