---
title: "Outbox Setup"
date: "2026-02-19"
weight: 2
draft: false
---

## Overview

The Outbox pattern ensures messages are staged within the same database transaction as your
domain changes, so no message is ever lost if the broker is unavailable at commit time.
The `IOutboxService<TContext>` stores events in an `OutboxEvent` table inside your
DbContext transaction; a separate `DeliveryHostedService` picks them up and forwards them
to the broker.

---

## When to use

- Your command handler modifies domain state **and** must publish a message atomically
- You are replacing direct `IEventBus.PublishAsync()` calls from v8.5.0
- You want to publish plain domain events without converting them to `IntegrationEvent`
- The delivery worker may run in a separate process or be added later

---

## Prerequisites

[Internal Messaging Setup]({{< ref "core/messaging/_index.md" >}}) — required for
`TransactionBehavior`'s DbContext dependency injection.

---

## NuGet packages

| Package | Purpose |
|---|---|
| [Juice.Messaging](https://www.nuget.org/packages/Juice.Messaging) | `MessagingBuilder`, `IOutboxService<T>`, `IMessage` |
| [Juice.Messaging.Outbox.EF](https://www.nuget.org/packages/Juice.Messaging.Outbox.EF) | `OutboxEvent` + `OutboxDelivery` EF tables, `AddOutbox()` sub-builder |
| [Juice.MediatR.Behaviors](https://www.nuget.org/packages/Juice.MediatR.Behaviors) | `TransactionBehavior<TRequest, TResponse, TContext>` |

---

## DI Registration

```csharp {linenos=false,hl_lines=[3,4,5,6,12],linenostart=1}
services.AddMessaging(builder =>
{
    builder.AddPublishingPolicies(configuration.GetSection("PublishingPolicies"));
    builder.AddOutbox(outbox =>
    {
        outbox.AddOutboxRepository();
        outbox.AddDeliveryIntents();
    });
});

services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(MyHandler).Assembly);
    cfg.AddOpenBehavior(typeof(TransactionBehavior<,,>));  // must subclass — see below
});
```

> **Note**: `AddOutbox()` automatically registers `IOutboxService<TContext>` →
> `OutboxEventService<TContext>` as a scoped service. You do not need to register it manually.

---

## IOutboxService — Staging Events

`IOutboxService<TContext>` replaces `IIntegrationEventService` from v8.5.0. Inject it
into your command handler or domain service and stage events before committing.

```csharp {linenos=false,hl_lines=[5,6],linenostart=1}
public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, IOperationResult>
{
    private readonly AppDbContext _db;
    private readonly IOutboxService<AppDbContext> _outbox;

    public CreateOrderCommandHandler(AppDbContext db, IOutboxService<AppDbContext> outbox)
    {
        _db = db;
        _outbox = outbox;
    }

    public async Task<IOperationResult> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        var order = new Order(cmd.OrderId, cmd.CustomerId);
        _db.Orders.Add(order);

        // Stage message — TransactionBehavior commits both together
        await _outbox.AddEventAsync(new OrderCreatedEvent(order.Id), ct);
        await _outbox.SaveEventsAsync(ct);

        return OperationResult.Success();
    }
}
```

**v8.5.0 → v9 mapping for staging:**

| v8.5.0 | v9 |
|---|---|
| `IIntegrationEventService.AddAndSaveEventAsync(evt)` | `IOutboxService.AddEventAsync(evt)` + `SaveEventsAsync()` |
| `IIntegrationEventLogService.SaveEventAsync(evt, transaction)` | `IOutboxService.AddEventAsync(evt)` (transaction managed by `TransactionBehavior`) |
| `MarkEventAsInProgressAsync` / `MarkEventAsPublishedAsync` / `MarkEventAsFailedAsync` | Handled automatically by `DeliveryHostedService` — no application code needed |

---

## Publishing Any IMessage (Including Domain Events)

The outbox accepts **any** `IMessage`, not just `IntegrationEvent`. You can stage a plain
domain event without wrapping it in an integration event type.

```csharp {linenos=false,hl_lines=[1,8],linenostart=1}
// Plain domain event — implements IMessage via MessageBase, NOT IIntegrationEvent
public record OrderShippedEvent(Guid OrderId, DateTimeOffset ShippedAt) : MessageBase;

// In handler — stage without any conversion
await _outbox.AddEventAsync(new OrderShippedEvent(order.Id, DateTimeOffset.UtcNow), ct);
await _outbox.SaveEventsAsync(ct);

// IIntegrationEvent is only needed if ANOTHER SERVICE wants to consume this event via broker
```

> **When do you need `IIntegrationEvent`?** Only when a consuming service needs to receive
> the event through the broker infrastructure (`IntegrationEventDispatcher` routes by
> `EventName`). If the event is only for local audit, workflow triggers, or internal
> delivery, plain `IMessage` is sufficient.

---

## IMessagePublishingPolicy — Routing Configuration

Publishing policies tell the delivery worker **where** to route each message type.
Configure them in `appsettings.json`:

```json {linenos=false,linenostart=1}
{
    "PublishingPolicies": {
        "Default": {
            "Publishers": [
                {
                    "Key": "rabbitmq",
                    "Destination": "default_exchange"
                }
            ]
        },
        "Rules": [
            {
                "Priority": 1000,
                "Match": {
                    "TenantTier": "enterprise",
                    "Event": "ContentPublishedIntegrationEvent"
                },
                "Publishers": [
                    {
                    "Key": "rabbitmq",
                    "Destination": "x.content.vip"
                    }
                ]
            },
            {
                "Priority": 800,
                "Match": {
                    "TenantTier": "free",
                    "Event": "ContentPublishedIntegrationEvent"
                },
                "Publishers": [
                    {
                    "Key": "rabbitmq",
                    "Destination": "x.content.free"
                    }
                ]
            },
            {
                "Priority": 600,
                "Match": {
                    "Domain": "Contents"
                },
                "Publishers": [
                    {
                    "Key": "rabbitmq",
                    "Destination": "x.content.integration"
                    },
                    {
                    "Key": "kafka",
                    "Destination": "x.content.integration"
                    }
                ]
            },
            {
            "Priority": 600,
                "Match": {
                    "Event": "LogEvent"
                },
                "Publishers": [
                    {
                    "Key": "rabbitmq",
                    "Destination": "x.logs"
                    }
                ]
            },
            {
                "Priority": 0,
                "Match": {},
                "Publishers": [
                    {
                    "Key": "rabbitmq",
                    "Destination": "default_exchange"
                    }
                ]
            }
        ]
    }
}
```

Policies are resolved by event type name. The `"default"` policy applies to any event
without a specific match.

---

## TransactionBehavior — Subclassing Guide

`TransactionBehavior<TRequest, TResponse, TContext>` is **abstract** and must be subclassed
for your DbContext. It runs at pipeline order `int.MaxValue - 20` (after validation behaviors,
before the actual handler) and:

1. Begins a resilient database transaction
2. Processes the MediatR `IRequest` command
3. Commits the transaction (domain changes + outbox events atomically)
4. Returns the command result

```csharp {linenos=false,hl_lines=[1,2,8],linenostart=1}
// In Juice.MediatR.Behaviors (package: Juice.MediatR.Behaviors)
internal class AppTransactionBehavior<T, R>
    : TransactionBehavior<T, R, AppDbContext>
    where T : IRequest<R>
{
    public AppTransactionBehavior(
        AppDbContext db,
        IOutboxService<AppDbContext> outboxService,
        ILogger<AppTransactionBehavior<T, R>> logger)
        : base(db, outboxService, logger) { }
}

// Registration — use open generic to cover all commands:
services.AddScoped(typeof(IPipelineBehavior<,>), typeof(AppTransactionBehavior<,>));
```

---

## OutboxEvent + OutboxDelivery Tables

The **AppDbContext** must be inherited from *IOutboxContext* to improve the transaction execution time.

The outbox uses a **two-table model**, replacing the single `IntegrationEventLog` table
from v8.5.0:

| Table | Purpose |
|---|---|
| `OutboxEvent` | Stores the staged message payload and metadata |
| `OutboxDelivery` | Tracks delivery state per publisher key (`NotPublished → InProgress → Published/Failed`) |

Run EF migrations after adding `Juice.Messaging.Outbox.EF` to generate these tables:

```bash
dotnet ef migrations add AddOutboxTables --context AppDbContext
dotnet ef database update --context AppDbContext
```

---

## See also

- [Delivery Setup]({{< ref "core/messaging/delivery/_index.md" >}}) — add the background delivery worker
- [Full Setup]({{< ref "core/messaging/full-setup/_index.md" >}}) — combine outbox + delivery + consumption
- [Event bus v8.5.0 archive]({{< ref "core/eventbus/v8.5.0/_index.md" >}}) — original `IntegrationEventLog` documentation
- [Integration service v8.5.0 archive]({{< ref "core/integration/v8.5.0/_index.md" >}}) — original `IIntegrationEventService` documentation
