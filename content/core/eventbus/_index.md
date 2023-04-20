---
title: "Event bus"
date: 2023-04-03T20:23:59+07:00
draft: false
weight: 8
---

To communicate between services we defined an **IEventBus** interface with a built-in RabbitMQ broker.
Please follow up [Implementing event-based communication between microservices](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/multi-container-microservice-net-applications/integration-event-based-microservice-communications) for details.

```
namespace Juice.EventBus
{
    /// <summary>
    /// Event bus
    /// </summary>
    public interface IEventBus
    {
        /// <summary>
        /// Publish <see cref="IntegrationEvent"/> to implemented broker like RabbitMQ, ServiceBus...
        /// </summary>
        /// <param name="event"></param>
        Task PublishAsync(IntegrationEvent @event);

        /// <summary>
        /// Subscribe an <see cref="IntegrationEvent"/> with specified <see cref="IIntegrationEventHandler{T}"/>
        /// </summary>
        /// <typeparam name="T"></typeparam>
        /// <typeparam name="TH"></typeparam>
        void Subscribe<T, TH>(string? key = default)
            where T : IntegrationEvent
            where TH : IIntegrationEventHandler<T>;

        /// <summary>
        /// Subscribe an <see cref="IntegrationEvent"/> with specified <see cref="IIntegrationEventHandler{T}"/>
        /// </summary>
        /// <typeparam name="T"></typeparam>
        /// <typeparam name="TH"></typeparam>
        void Unsubscribe<T, TH>(string? key = default)
            where TH : IIntegrationEventHandler<T>
            where T : IntegrationEvent;
    }
}
```

### RabbitMQ broker

To use RabbitMQ broker as an event bus backend, we register RabbitMQEventBus services via *IServiceCollection extension*

```csharp {linenos=false,hl_lines=[2,4],linenostart=1}
    ...
    using Juice.EventBus.RabbitMQ.DependencyInjection
    ...
    services.RegisterRabbitMQEventBus(configuration.GetSection("RabbitMQ"));

    // OR
    // services.RegisterRabbitMQEventBus(configuration.GetSection("RabbitMQ"), options=> {
        // configure options here
    // });

    // RabbitMQ options
    // "RabbitMQ": {
    //      "RabbitMQEnabled": false,
    //      "SubscriptionClientName": "xunit_test", // client name must unique for a service (not service instance)
    //      "RetryCount": 5,
    //      "Connection": "your_rabbitmq_host",
    //      "Port": 5672,
    //      "VirtualHost": null,
    //      "UserName": null,
    //      "Password": null
    // }
```

The library can be accessed via Nuget:
- [Juice.EventBus.RabbitMQ](https://www.nuget.org/packages/Juice.EventBus.RabbitMQ)

### IntegrationEvent
You can define your integration event that inherit **IntegrationEvent** record then publish throw **IEventBus**
```
    using Juice.EventBus;
    public record ContentPublishedIntegrationEvent : IntegrationEvent
    {
        public ContentPublishedIntegrationEvent(string message)
        {
            Message = message;
        }
        public string Message { get; set; }
    }
```
```
    ...
    using Juice.EventBus;
    ...

    await eventBus.PublishAsync(new ContentPublishedIntegrationEvent($"Hello"));

``` 

The library can be accessed via Nuget:
- [Juice.EventBus](https://www.nuget.org/packages/Juice.EventBus)

### IntegrationEventHandler
Someone will implement an integration handler from **IIntegrationEventHandler<TEvent>** interface in other app to handle your integration event for their purposes.

For example: a handler that print received message and event time.
```
    ...
    using Juice.EventBus;
    ...

    public class ContentPublishedIntegrationEventHandler : IIntegrationEventHandler<ContentPublishedIntegrationEvent>
    {
        private ILogger _logger;
        public ContentPublishedIntegrationEventHandler(ILogger<ContentPublishedIntegrationEventHandler> logger)
        {
            _logger = logger;
        }
        public async Task HandleAsync(ContentPublishedIntegrationEvent @event)
        {
            await Task.Yield();
            _logger.LogInformation("[X] Received {0} at {1}", @event.Message, @event.CreationDate);
        }
    }
```

Then register into DI and subscribe event

```
    // configure services
    services.AddTransient<ContentPublishedIntegrationEventHandler>();

    ...
    // configure WebApplication 
    var eventBus = app.Services.GetRequiredService<IEventBus>();

    eventBus.Subscribe<ContentPublishedIntegrationEvent, ContentPublishedIntegrationEventHandler>();
```

The library can be accessed via Nuget:
- [Juice.EventBus](https://www.nuget.org/packages/Juice.EventBus)

### IntegrationEventLog

It is a part of balanced approach for atomicity and resiliency when publishing to the event bus.
You can follow [Designing atomicity and resiliency when publishing to the event bus](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/multi-container-microservice-net-applications/subscribe-events#designing-atomicity-and-resiliency-when-publishing-to-the-event-bus) topic for more information.
We provide **IIntegrationEventLogService** interface with an implementation for EF backend use SQLServer or PostgreSQL.

```
    public interface IIntegrationEventLogService : IDisposable
    {
        IntegrationEventLogContext LogContext { get; }
        
        // Use to process pending events
        Task<IEnumerable<IntegrationEventLogEntry>> RetrieveEventLogsPendingToPublishAsync(Guid transactionId);

        // Save an integration event within a same transaction with domain DBContext
        Task SaveEventAsync(IntegrationEvent @event, IDbContextTransaction transaction);

        // Change event state after publish it to the service bus
        Task MarkEventAsPublishedAsync(Guid eventId);

        // Change event state before publish it to the service bus
        Task MarkEventAsInProgressAsync(Guid eventId);

        // Change event state on failure publising
        Task MarkEventAsFailedAsync(Guid eventId);

    }

    public interface IIntegrationEventLogService<out TContext> : IIntegrationEventLogService
        where TContext : DbContext
    {
        /// <summary>
        /// Ensure event log context has an associated connection with input <see cref="T"/> context.
        /// <para>Throw <see cref="ArgumentException"/> if input context has not same type with <see cref="TContext"/></para>
        /// </summary>
        /// <typeparam name="T"></typeparam>
        /// <param name="context"></param>
        void EnsureAssociatedConnection<T>(T context) where T : DbContext;
    }
```

We need to register services and a **DomainDbContext** to use this feature.

```csharp {linenos=false,hl_lines=[2,3,6,10],linenostart=1}
    ...
    using Juice.EventBus.IntegrationEventLog.EF;
    using Juice.EventBus.IntegrationEventLog.EF.DependencyInjection;
    ...

    services.AddIntegrationEventLog() // add integration event log services
        .RegisterContext<DomainDbContext>(schema); 

    // migrate DB if need
    
```
Call `.RegisterContext<DomainDbContext>(schema)` will register an **IntegrationEventLogContext** factory that mapped to specified schema and work with DomainDbContext. So **IIntegrationEventLogService** service can get `Func<DomainDbContext, IntegrationEventLogContext>` as a service and create an **IntegrationEventLogContext** instance from a **DomainDbContext** instance.

They will have associated DbConnection and can access the same transaction.

Step by step, the process goes like this:

1. The application begins a local database transaction.

2. It then updates the state of your domain entities and inserts an event into the integration event table.

3. Finally, it commits the transaction, so you get the desired atomicity and then

4. You publish the event somehow (next).

When implementing the steps of publishing the events, you have these choices:

- Publish the integration event right after committing the transaction and use another local transaction to mark the events in the table as being published. Then, use the table just as an artifact to track the integration events in case of issues in the remote microservices, and perform compensatory actions based on the stored integration events.

- Use the table as a kind of queue. A separate application thread or process queries the integration event table, publishes the events to the event bus, and then uses a local transaction to mark the events as published.

```csharp {linenos=false,hl_lines=[2,3],linenostart=1}
    ...
    using Juice.EventBus;
    using Juice.EventBus.IntegrationEventLog.EF;
    ...

    var context = scope.ServiceProvider.GetRequiredService<DomainDbContext>();
    var eventLogService = scope.ServiceProvider.GetRequiredService<IIntegrationEventLogService<DomainDbContext>>();
    eventLogService.EnsureAssociatedConnection(context);

    // do something with your DomainDbContext
    context.Add(new Content());

    // create an integration event
    var evt = new ContentPublishedIntegrationEvent($"Content {content.Code} was published.");

    // save DomainDbContext changes and add an integration event at the same time, in the same transaction
    await ResilientTransaction.New(context).ExecuteAsync(async (transaction) =>
    {
        // Achieving atomicity between original catalog database operation and the IntegrationEventLog thanks to a local transaction
        await context.SaveChangesAsync();
        await eventLogService.SaveEventAsync(evt, transaction);
    });

    // if everything is ok then you can publish created integration event throw service bus

    var eventBus = scope.ServiceProvider.GetRequiredService<IEventBus>();
    try
    {
        logger.LogInformation("----- Publishing integration event: {IntegrationEventId_published} from {AppName} - ({@IntegrationEvent})", evt.Id, nameof(IntegrationEventLogTest), evt);

        await eventLogService.MarkEventAsInProgressAsync(evt.Id);
        await eventBus.PublishAsync(evt);
        await eventLogService.MarkEventAsPublishedAsync(evt.Id);
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "ERROR Publishing integration event: {IntegrationEventId} from {AppName} - ({@IntegrationEvent})", evt.Id, nameof(IntegrationEventLogTest), evt);
        await eventLogService.MarkEventAsFailedAsync(evt.Id);
    }
```

The library can be accessed via Nuget:
- [Juice.EventBus.IntegrationEventLog.EF](https://www.nuget.org/packages/Juice.EventBus.IntegrationEventLog.EF)
- [Juice.EventBus.IntegrationEventLog.EF.SqlServer](https://www.nuget.org/packages/Juice.EventBus.IntegrationEventLog.EF.SqlServer)
- [Juice.EventBus.IntegrationEventLog.EF.PostgreSQL](https://www.nuget.org/packages/Juice.EventBus.IntegrationEventLog.EF.PostgreSQL)