---
title: "Local Transport"
date: "2026-03-14"
weight: 4
draft: false
---

## Overview

The Local Transport adds two in-process delivery routes and a unified `IMessageService`
publishing interface. Messages can be dispatched to handlers within the same process
without requiring a broker, while still supporting configuration-driven routing that can
switch between local and broker delivery with no code changes.

| Route | Publisher Key | Durable? | Latency | Use case |
|---|---|---|---|---|
| **In-memory channel** | `"local-channel"` | No | Immediate | Fast fire-and-forget: cache invalidation, lightweight side-effects |
| **Outbox + in-process** | `"local"` | Yes | Near-immediate | Reliable cross-aggregate events within the same service |
| **Outbox + broker** | `"rabbitmq"` | Yes | Polling interval | Cross-service messaging (existing) |

---

## When to use

- You need in-process event dispatch without broker overhead (`"local-channel"`)
- You need durable, retryable in-process dispatch backed by the outbox (`"local"`)
- You want a single `PublishAsync` call site where routing is config-driven
- You are building a monolith or modular monolith where some events stay in-process

---

## Prerequisites

- [Internal Messaging Setup]({{< ref "core/messaging/_index.md" >}}) for `IIdempotencyService` and `MessagingBuilder`
- [Outbox Setup]({{< ref "core/messaging/outbox/_index.md" >}}) if using the `"local"` durable route

---

## NuGet packages

| Package | Purpose |
|---|---|
| [Juice.Messaging.Local](https://www.nuget.org/packages/Juice.Messaging.Local) | `IMessageService`, `LocalChannelBackgroundService`, `LocalTransportPublisher` |
| [Juice.Messaging](https://www.nuget.org/packages/Juice.Messaging) | `MessagingBuilder`, `IMessagePublishingPolicy`, `IOutboxService<T>` |
| [Juice.Messaging.Idempotency.Caching](https://www.nuget.org/packages/Juice.Messaging.Idempotency.Caching) | In-memory idempotency (dev/test) |
| [Juice.Messaging.Idempotency.Redis](https://www.nuget.org/packages/Juice.Messaging.Idempotency.Redis) | Redis idempotency (production) |

---

## IMessageService — Unified Publishing Interface

Two interfaces are available depending on whether your routes need outbox writes:

| Interface | Routes supported | When to use |
|---|---|---|
| `IMessageService` | `"local-channel"` only | Fire-and-forget, no DB |
| `IMessageService<TContext>` | All three (`"local-channel"`, `"local"`, broker) | When any route may require outbox writes |

Both expose a single method:

```csharp
Task PublishAsync(IMessage message, CancellationToken cancellationToken = default);
```

The same `PublishAsync` call works for domain events (`INotification`) and integration
events (`IIntegrationEvent`). Routing is resolved by `IMessagePublishingPolicy` at
runtime — changing the route in configuration changes the transport with no code changes.

---

## DI Registration

### Local-channel only (zero DB)

```csharp {linenos=false,hl_lines=[3,4],linenostart=1}
services.AddMessaging(builder =>
{
    builder.AddLocalChannel(opts =>
    {
        opts.MaxConcurrency = 10;  // optional: limit concurrent handlers (default: unlimited)
    });
    builder.AddIdempotencyInMemory();  // or AddIdempotencyRedis for production
});

// Register publishing policy
services.AddSingleton<IMessagePublishingPolicy>(
    new FixedRoutePolicy("local-channel", string.Empty));
// or use config-driven policies:
// builder.AddPublishingPolicies(configuration.GetSection("PublishingPolicies"));
```

### All routes (local-channel + local + broker)

```csharp {linenos=false,hl_lines=[3,4,5],linenostart=1}
services.AddMessaging(builder =>
{
    builder.AddPublishingPolicies(configuration.GetSection("PublishingPolicies"));

    builder.AddOutbox(outbox =>
    {
        outbox.AddOutboxRepository();
        outbox.AddDeliveryIntents();
    });

    // Registers IMessageService<AppDbContext> (implies AddLocalChannel)
    builder.AddMessageService<AppDbContext>();

    // Registers LocalTransportPublisher as keyed ITransportPublisher "local"
    builder.AddLocalPublisher<AppDbContext>();

    builder.AddIdempotencyRedis(opts =>
    {
        opts.Configuration = configuration.GetConnectionString("Redis");
    });

    // Delivery worker for outbox-backed routes
    builder.AddDelivery(delivery =>
    {
        // "local" publisher key dispatches in-process via LocalTransportPublisher
        delivery.AddDeliveryProcessor<AppDbContext>("local",
            "send-pending", "retry-failed", "recover-timeout");

        // "rabbitmq" publisher key dispatches to broker (existing)
        delivery.AddDeliveryProcessor<AppDbContext>("rabbitmq",
            "send-pending", "retry-failed", "recover-timeout");

        delivery.AddDeliveryPolicies(configuration.GetSection("DeliveryPolicies"));
    });
});
```

---

## Publishing Policy Configuration

Route events to local or broker transports via `appsettings.json`:

```json {linenos=false,linenostart=1}
{
    "PublishingPolicies": {
        "Default": {
            "Publishers": [
                { "Key": "rabbitmq", "Destination": "default_exchange" }
            ]
        },
        "Rules": [
            {
                "Priority": 1000,
                "Match": { "Domain": "Cache" },
                "Publishers": [
                    { "Key": "local-channel", "Destination": "" }
                ]
            },
            {
                "Priority": 900,
                "Match": { "Domain": "Workflows" },
                "Publishers": [
                    { "Key": "local", "Destination": "" }
                ]
            },
            {
                "Priority": 800,
                "Match": { "Domain": "Orders" },
                "Publishers": [
                    { "Key": "local", "Destination": "" },
                    { "Key": "rabbitmq", "Destination": "orders_exchange" }
                ]
            }
        ]
    }
}
```

In this example:
- **Cache** domain events use `"local-channel"` (instant, non-durable)
- **Workflows** domain events use `"local"` (durable, in-process)
- **Orders** events go to **both** `"local"` and `"rabbitmq"` (local handler + cross-service)

---

## Usage Example

```csharp {linenos=false,hl_lines=[5,12],linenostart=1}
public class OrderCommandHandler : IRequestHandler<CreateOrderCommand, IOperationResult>
{
    private readonly AppDbContext _db;
    private readonly IMessageService<AppDbContext> _messaging;

    public OrderCommandHandler(AppDbContext db, IMessageService<AppDbContext> messaging)
    {
        _db = db;
        _messaging = messaging;
    }

    public async Task<IOperationResult> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        var order = new Order(cmd.OrderId, cmd.CustomerId);
        _db.Orders.Add(order);

        // Routing determined by IMessagePublishingPolicy config — no transport knowledge here
        await _messaging.PublishAsync(new OrderCreatedEvent(order.Id), ct);

        return OperationResult.Success();
    }
}
```

---

## Route Behavior Details

### `"local-channel"` — Non-Durable, Immediate

1. Message enqueued to `Channel<IMessage>` (unbounded, singleton)
2. `PublishAsync` returns **immediately** — handler runs asynchronously
3. `LocalChannelBackgroundService` drains the channel and dispatches:
   - `INotification` → full MediatR notification pipeline (`INotificationPublisher.Publish<T>`)
   - `IIntegrationEvent` → `IntegrationEventDispatcher` → all `IIntegrationEventHandler<T>` in DI
4. Handler exceptions are caught and logged — they do not crash the service
5. Concurrency bounded by `MaxConcurrency` option (default: unlimited)
6. **Not durable**: messages in the channel are lost on process shutdown

### `"local"` — Durable, Near-Immediate

1. Message written to `OutboxEvent` + `OutboxDelivery` tables (same transaction as domain data)
2. After outbox commit, message is **also enqueued to the channel** for immediate best-effort dispatch
3. `LocalChannelBackgroundService` dispatches the handler (same as `"local-channel"`)
4. `IntegrationEventDispatcher` records the idempotency key on successful dispatch
5. `DeliveryHostedService` later processes the same outbox record via `LocalTransportPublisher`
6. `LocalTransportPublisher` restores `MessageContext` from outbox headers (`x-source`, `x-correlation-id`)
7. Idempotency check detects the duplicate → handler is **not invoked again**
8. If immediate dispatch fails, the outbox record remains `NotPublished` — delivery retries as normal

```
Publish ──► Outbox write ──► Enqueue to channel (immediate, best-effort)
                │                      │
                │                      ▼
                │              Handler runs (idempotency records key)
                │
                ▼
         DeliveryHostedService (polling)
                │
                ▼
         LocalTransportPublisher
                │
                ▼
         Idempotency check → Duplicated (skip)
         OR (if immediate failed) → Handler runs
```

---

## Idempotency Across Dispatch Paths

When using the `"local"` route, the same message may be dispatched twice:
1. Immediately after outbox commit (via channel)
2. By `DeliveryHostedService` (via `LocalTransportPublisher`)

The `IntegrationEventDispatcher` prevents duplicate handler invocation using the
idempotency key `(EventName, "{Source}:{MessageId}")`:

| Dispatch path | How `Source` is determined |
|---|---|
| Immediate (channel) | `MessageContext.Current.Source` (set at entry point) |
| Delivery retry (`LocalTransportPublisher`) | Restored from outbox header `x-source` |

Both paths produce the **same key**, so the second dispatch is skipped.

> **Important**: The `IIdempotencyService` must be registered with a lifetime that shares
> state across DI scopes. `InMemoryIdempotencyService` is scoped by default — for
> cross-scope deduplication, use Redis or EF-backed idempotency in production.

---

## Transaction Behavior

| Context | `"local-channel"` | `"local"` / broker |
|---|---|---|
| Inside `TransactionBehavior` | Enqueued to channel immediately | Outbox write joins transaction; no immediate channel dispatch (not yet committed) |
| Outside active transaction | Enqueued to channel immediately | `SaveEventsAsync(null)` commits standalone; then enqueued to channel for immediate dispatch |

> **Note**: When called inside `TransactionBehavior`, the `"local"` route does not
> dispatch immediately because the transaction has not committed yet. The message is
> delivered by `DeliveryHostedService` after commit. Use `IOutboxService<T>` directly
> (via domain event handlers + `TransactionBehavior`) for transactional scenarios.

---

## Handler Registration

Local transport reuses existing handler interfaces — no new interfaces needed:

```csharp {linenos=false,linenostart=1}
// Integration event handler — same as broker consumers
public class OrderCreatedHandler : IIntegrationEventHandler<OrderCreatedEvent>
{
    public Task HandleAsync(OrderCreatedEvent @event)
    {
        // Handle the event
        return Task.CompletedTask;
    }
}

// Domain event handler — same as MediatR notifications
public class CacheInvalidationHandler : INotificationHandler<ProductUpdatedEvent>
{
    public ValueTask Handle(ProductUpdatedEvent notification, CancellationToken ct)
    {
        // Invalidate cache
        return ValueTask.CompletedTask;
    }
}

// DI registration — standard
services.AddTransient<IIntegrationEventHandler<OrderCreatedEvent>, OrderCreatedHandler>();
services.AddTransient<INotificationHandler<ProductUpdatedEvent>, CacheInvalidationHandler>();
```

---

## Metrics

Both local routes emit the same delivery metrics as broker routes:

| Metric | `"local-channel"` | `"local"` |
|---|---|---|
| `delivery.attempts` | Per channel dispatch | Per `LocalTransportPublisher` call |
| `delivery.success` | Handler completed | Handler completed |
| `delivery.failure` | Handler threw exception | Handler threw exception |
| `delivery.latency` | Dispatch duration | Dispatch duration |

Use the publisher key (`"local-channel"` or `"local"`) to filter metrics by transport.

---

## See also

- [MessageContext Setup]({{< ref "core/messaging/message-context/_index.md" >}}) — middleware, attribute, and test initialization
- [Internal Messaging Setup]({{< ref "core/messaging/_index.md" >}}) — MediatR + idempotency standalone
- [Outbox Setup]({{< ref "core/messaging/outbox/_index.md" >}}) — transactional outbox staging
- [Delivery Setup]({{< ref "core/messaging/delivery/_index.md" >}}) — background delivery worker
- [Full Setup]({{< ref "core/messaging/full-setup/_index.md" >}}) — all components combined
