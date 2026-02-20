---
title: "Full Pipeline Setup"
date: "2026-02-19"
weight: 5
draft: false
---

## Overview

This guide combines all four messaging capabilities — internal MediatR idempotency,
transactional outbox staging, background delivery, and RabbitMQ consumption — into a
single, coherent `services.AddMessaging()` call. Follow this guide for new services or
when you want the complete picture before choosing an incremental approach.

---

## Prerequisites

Either:
- Complete the four individual setup guides ([Internal Messaging]({{< ref "core/messaging/_index.md" >}}), [Outbox]({{< ref "core/messaging/outbox/_index.md" >}}), [Delivery]({{< ref "core/messaging/delivery/_index.md" >}}), [Consumption]({{< ref "core/messaging/consumption/_index.md" >}})), OR
- Follow this guide from scratch

---

## NuGet packages

Install all packages from the four setup guides:

| Package | Purpose |
|---|---|
| [Juice.Messaging](https://www.nuget.org/packages/Juice.Messaging) | `MessagingBuilder`, `IOutboxService<T>`, `IMessage` hierarchy |
| [Juice.Messaging.Outbox.EF](https://www.nuget.org/packages/Juice.Messaging.Outbox.EF) | Outbox tables, `AddOutbox()` sub-builder |
| [Juice.Messaging.Outbox.Delivery](https://www.nuget.org/packages/Juice.Messaging.Outbox.Delivery) | `DeliveryHostedService`, `AddDelivery()` sub-builder |
| [Juice.MediatR.Behaviors](https://www.nuget.org/packages/Juice.MediatR.Behaviors) | `IdempotencyRequestBehavior<,>`, `TransactionBehavior<,,>` |
| [Juice.MediatR.Contracts](https://www.nuget.org/packages/Juice.MediatR.Contracts) | `IIdempotentRequest` interface |
| [Juice.EventBus](https://www.nuget.org/packages/Juice.EventBus) | `IIntegrationEventHandler<T>`, `IntegrationEventDispatcher` |
| [Juice.EventBus.RabbitMQ](https://www.nuget.org/packages/Juice.EventBus.RabbitMQ) | RabbitMQ transport (producer + consumer) |
| [Juice.Messaging.Idempotency.Redis](https://www.nuget.org/packages/Juice.Messaging.Idempotency.Redis) | Redis idempotency backend (recommended) |
| [Juice.Messaging.Idempotency.Caching](https://www.nuget.org/packages/Juice.Messaging.Idempotency.Caching) | In-memory / distributed cache backend |
| [Juice.Messaging.Idempotency.EF](https://www.nuget.org/packages/Juice.Messaging.Idempotency.EF) | EF idempotency backend (auditable) |

---

## Sub-Builder Registration Order

**The order of sub-builder calls inside `AddMessaging()` is significant.** Register them
in this exact sequence:

1. **`AddPublishingPolicies()`** — must come before `AddOutbox()` because the outbox reads
   routing policies at message-staging time
2. **`AddOutbox()`** — must come before `AddDelivery()` because the delivery builder reads
   outbox configuration during its construction
3. **`AddIdempotency*()`** — must come before `AddDelivery()` because `IntegrationEventDispatcher`
   (created inside delivery) resolves `IIdempotencyService` at startup
4. **`AddDelivery()`** — creates `EventBusBuilder` internally via its constructor; therefore
   `services.AddEventBus()` must **not** be called separately
5. **RabbitMQ** — configured inside `delivery.EventBus.AddRabbitMQ()` because the
   `EventBusBuilder` is owned by `DeliveryBuilder`

---

## Complete DI Registration

```csharp {linenos=false,hl_lines=[2,5,8,11,15,16,21,22],linenostart=1}
services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(MyHandler).Assembly);
    cfg.AddIdempotencyRequestBehavior();
    cfg.AddOpenBehavior(typeof(AppTransactionBehavior<,>));  // subclassed TransactionBehavior
});

services.AddMessaging(builder =>
{
    // 1. Publishing policies (needed by outbox at staging time)
    builder.AddPublishingPolicies(configuration.GetSection("PublishingPolicies"));

    // 2. Outbox (must precede delivery)
    builder.AddOutbox(outbox =>
    {
        outbox.AddOutboxRepository();
        outbox.AddDeliveryIntents();
    });

    // 3. Idempotency backend (before EventBus dispatcher is created)
    builder.AddIdempotencyRedis(opts =>
    {
        opts.Configuration = configuration.GetConnectionString("Redis");
    });

    // 4. Delivery (creates EventBusBuilder internally — do not call AddEventBus() separately)
    builder.AddDelivery(delivery =>
    {
        delivery.AddDeliveryProcessor<AppDbContext>("rabbitmq",
            "send-pending", "retry-failed", "recover-timeout");
        delivery.AddDeliveryPolicies(configuration.GetSection("DeliveryPolicies"));
    });

    // 5. EventBus with infrastructure
    builder.AddEventBus()
        .AddConsumerServices(consumers =>
        {
            consumers.Subscribe<OrderCreatedEvent, OrderCreatedEventHandler>();
        })
        .AddRabbitMQ(rabbitMQ =>
        {
            // 6. RabbitMQ inside EventBus
            rabbitMQ.AddConnection("rabbitmq", configuration.GetSection("RabbitMQ"))
                    .AddProducer("rabbitmq", "rabbitmq")
                    .AddConsumer("orders", "orders-queue", "rabbitmq");
        });
});

services.AddHealthChecks()
    .AddRabbitMQHealthCheck("rabbitmq", tags: new[] { "ready" })
    .AddOutboxDeliveryHealthCheck<AppDbContext>(opts =>
    {
        opts.StuckMessageThresholdMinutes = 15;
        opts.MaxStuckMessages = 50;
    }, tags: new[] { "live" });
```

---

## Migration from v8.5.0

### v8.5.0 → v9 Concept Mapping

| v8.5.0 Concept | Package (v8.5.0) | v9 Replacement | Package (v9) |
|---|---|---|---|
| `IEventBus` interface | `Juice.EventBus` | `IOutboxService<T>` for staging; `IEventBus` (CompositeEventPublisher) remains as a publish façade | `Juice.Messaging` / `Juice.EventBus` |
| `IEventBus.PublishAsync()` direct call | `Juice.EventBus` | `IOutboxService.AddEventAsync()` + `SaveEventsAsync()` inside transaction | `Juice.Messaging.Outbox.EF` |
| `IntegrationEventLog` table (single-table) | `Juice.EventBus.IntegrationEventLog.EF` | `OutboxEvent` + `OutboxDelivery` two-table model | `Juice.Messaging.Outbox.EF` |
| `IIntegrationEventLogService` | `Juice.EventBus.IntegrationEventLog.EF` | `IOutboxService<T>` | `Juice.Messaging` |
| `IIntegrationEventLogService.SaveEventAsync()` | — | `IOutboxService.AddEventAsync()` + `SaveEventsAsync(transactionId)` | — |
| `IIntegrationEventLogService.MarkEventAsInProgressAsync()` | — | Handled automatically by `DeliveryHostedService` | — |
| `IIntegrationEventLogService.MarkEventAsPublishedAsync()` | — | Handled automatically by `DeliveryHostedService` | — |
| `IIntegrationEventLogService.MarkEventAsFailedAsync()` | — | Handled automatically by `DeliveryHostedService` | — |
| `IIntegrationEventService` | `Juice.Integrations` | `IOutboxService<T>` | `Juice.Messaging` |
| `IIntegrationEventService.AddAndSaveEventAsync()` | — | `IOutboxService.AddEventAsync()` | — |
| `IIntegrationEventService.PublishEventsThroughEventBusAsync()` | — | Removed — `DeliveryHostedService` publishes automatically | — |
| `TransactionBehavior` (v8, in `Juice.Integrations.MediatR`) | `Juice.Integrations` | `TransactionBehavior<T,R,TContext>` (abstract, in `Juice.MediatR.Behaviors`) | `Juice.MediatR.Behaviors` |
| `services.RegisterRabbitMQEventBus()` | `Juice.EventBus.RabbitMQ` | `builder.AddMessaging().AddDelivery(d => d.EventBus.AddRabbitMQ(...))` | `Juice.Messaging` + `Juice.EventBus.RabbitMQ` |
| `IRequestManager` (command deduplication) | `Juice.MediatR` | `IIdempotentRequest` + `IdempotencyBehavior` | `Juice.MediatR.Contracts` + `Juice.MediatR.Behaviors` |
| `IIdentifiedCommand<T>` wrapper | `Juice.MediatR` | Removed — mark command directly with `IIdempotentRequest` | `Juice.MediatR.Contracts` |
| `requestManager.ExistAsync(id)` in handler | — | Removed — `IdempotencyBehavior` handles it automatically | — |
| `IntegrationEvent` record (v8) | `Juice.EventBus` | `IntegrationEvent` record still exists; now extends `MessageBase` (carries `MessageId`, `CreatedAt`, `TenantId`) | `Juice.EventBus.Contracts` |
| `IIntegrationEventHandler<T>.HandleAsync()` | `Juice.EventBus` | Unchanged | `Juice.EventBus` |
| RabbitMQ `SubscriptionClientName` config | `Juice.EventBus.RabbitMQ` | `AddConsumer(key, queueName, connectionName)` | `Juice.EventBus.RabbitMQ` |

### What Is Unchanged

These v8.5.0 concepts carry forward to v9 without replacement:

- `IntegrationEvent` abstract record (still the base for integration events)
- `IIntegrationEventHandler<T>` interface (handlers unchanged)
- `IIntegrationEvent` interface (still required on the consuming side)
- `RabbitMQ.Client` as the broker transport
- `Finbuckle.MultiTenant` for tenant resolution
- MediatR `IRequest<T>`, `INotification` patterns (MediatR itself unchanged)
- `IAggregrateRoot<TNotification>` domain pattern
- `IUnitOfWork` interface

---

## See also

- [Internal Messaging Setup]({{< ref "core/messaging/_index.md" >}}) — MediatR + `IdempotencyBehavior` standalone
- [Outbox Setup]({{< ref "core/messaging/outbox/_index.md" >}}) — transactional outbox standalone
- [Delivery Setup]({{< ref "core/messaging/delivery/_index.md" >}}) — delivery worker standalone
- [Consumption Setup]({{< ref "core/messaging/consumption/_index.md" >}}) — RabbitMQ consumer standalone
- [Event bus v8.5.0 archive]({{< ref "core/eventbus/v8.5.0/_index.md" >}}) — original `IEventBus` and `IntegrationEventLog` docs
- [MediatR v8.5.0 archive]({{< ref "core/mediator/v8.5.0/_index.md" >}}) — original `IRequestManager` docs
- [Integration service v8.5.0 archive]({{< ref "core/integration/v8.5.0/_index.md" >}}) — original `IIntegrationEventService` docs
