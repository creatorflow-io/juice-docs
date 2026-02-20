---
title: "Integration service (v8.5.0)"
date: 2023-04-03T20:24:09+07:00
draft: false
weight: 999
---
> **⚠ Archived — v8.5.0**
> This page describes the pre-v9 Juice API.
> For current documentation see [Messaging]({{< ref "core/messaging/_index.md" >}}).

### Integration event service
In the [IntegrationEventLog]({{< ref "core/eventbus/#integrationeventlog">}}) section, we talked about atomicity and resiliency when publishing to the event bus, followed by each step of the process.
In this section, we will package the steps of publishing events and updating event states into **IIntegrationEventService** service, and then combine them with **MediatR** behavior for transaction processing.

```csharp
    public interface IIntegrationEventService
    {
        Task PublishEventsThroughEventBusAsync(Guid transactionId);

        Task AddAndSaveEventAsync(IntegrationEvent evt);
    }

    public interface IIntegrationEventService<out TContext> : IIntegrationEventService
        where TContext : DbContext, IUnitOfWork
    {
        TContext DomainContext { get; }

        /// <summary>
        /// Use specified transaction when working with transient DbContext
        /// </summary>
        /// <param name="evt"></param>
        /// <param name="transaction"></param>
        /// <returns></returns>
        Task AddAndSaveEventAsync(IntegrationEvent evt, IDbContextTransaction transaction);
    }

```
Now the process steps will look like this:

```csharp {linenos=false,hl_lines=[2,6,19,21,24],linenostart=1}
    ...
    using Juice.Integrations.EventBus;
    ...

    var context = scope.ServiceProvider.GetRequiredService<DomainDbContext>();
    var integrationEventService = scope.ServiceProvider.GetRequiredService<IIntegrationEventService<DomainDbContext>>();

    // do something with your DomainDbContext
    context.Add(new Content());

    // create an integration event
    var evt = new ContentPublishedIntegrationEvent($"Content {content.Code} was published.");

    // save DomainDbContext changes and add an integration event at the same time, in the same transaction
    await ResilientTransaction.New(context).ExecuteAsync(async (transaction) =>
    {
        // Achieving atomicity between original catalog database operation and the IntegrationEventLog thanks to a local transaction
        await context.SaveChangesAsync();
        await integrationEventService.AddAndSaveEventAsync(evt);
        // commit transaction
        await context.CommitTransactionAsync(transaction);

        // if everything is ok then you can publish created integration event throw service bus
        await integrationEventService.PublishEventsThroughEventBusAsync(transaction.TransactionId);
    });

```
NOTE: to use this pattern, your **DomainDbContext** must implement [IUnitOfWork]({{< ref "core/entityframework#IUnitOfWork" >}}) interface.

The library can be accessed via Nuget:
- [Juice.Integrations](https://www.nuget.org/packages/Juice.Integrations)

### Transaction behavior

One more step to streamline the process in the section above when integrating with **MediatR**.
We have implemented an abstraction class to handle the transaction step by step:
- Begin a resilient transaction
- Process **IRequest** command
- Commit transaction (auto add and save integration events by *UnitOfWork* pattern)
- Publish pending events through event bus
- Return processed command result

You need to implement **TransactionBehavior<TRequest, TResponse, TContext>** for your request and DbContext like this:

```csharp {linenos=false,hl_lines=[1,2],linenostart=1}
    using Juice.Integrations.EventBus;
    using Juice.Integrations.MediatR.Behaviors;
    ...

    internal class TimerTransactionBehavior<T, R> : TransactionBehavior<T, R, TimerDbContext>
        where T : IRequest<R>
    {
        public TimerTransactionBehavior(TimerDbContext dbContext, IIntegrationEventService<TimerDbContext> integrationEventService, ILogger<TimerTransactionBehavior<T, R>> logger) : base(dbContext, integrationEventService, logger)
        {
        }
    }
```

Then register as a MediatR behavior

```csharp
    services.AddScoped(typeof(IPipelineBehavior<CreateTimerCommand, TimerRequest>),
                    typeof(TimerTransactionBehavior<CreateTimerCommand, TimerRequest>));
```

To understand this section, please read more about using UnitOfWork and MediatR.

The library can be accessed via Nuget:
- [Juice.Integrations](https://www.nuget.org/packages/Juice.Integrations)
