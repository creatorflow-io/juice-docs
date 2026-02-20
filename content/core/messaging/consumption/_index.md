---
title: "Consumption Setup"
date: "2026-02-19"
weight: 4
draft: false
---

## Overview

The Consumption setup registers a RabbitMQ consumer engine that deserializes incoming
messages from a queue, routes them to the correct `IIntegrationEventHandler<T>`, and
automatically deduplicates re-delivered messages using `IIdempotencyService` — with no
handler code changes required.

---

## When to use

- Your service needs to receive integration events published by other services
- You want automatic idempotent event handling without writing deduplication logic in handlers
- You need dead-letter exchange (DLX) support for unhandled or failed messages

---

## Prerequisites

[Internal Messaging Setup]({{< ref "core/messaging/_index.md" >}}) — required for
`IIdempotencyService` registration, which `IntegrationEventDispatcher` resolves at startup.

---

## NuGet packages

| Package | Purpose |
|---|---|
| [Juice.EventBus](https://www.nuget.org/packages/Juice.EventBus) | `IIntegrationEventHandler<T>`, `IntegrationEventDispatcher`, `ISubscriptionsManager` |
| [Juice.EventBus.RabbitMQ](https://www.nuget.org/packages/Juice.EventBus.RabbitMQ) | `RabbitMQConsumerHostedService`, `AddRabbitMQ()` sub-builder |
| [Juice.Messaging.Idempotency.Redis](https://www.nuget.org/packages/Juice.Messaging.Idempotency.Redis) | Redis idempotency backend (recommended) |
| [Juice.Messaging.Idempotency.Caching](https://www.nuget.org/packages/Juice.Messaging.Idempotency.Caching) | In-memory or distributed cache backend |

---

## DI Registration

```csharp {linenos=false,hl_lines=[4,7,8,9,10,11,12,15,16,17],linenostart=1}
services.AddMessaging()
    .AddIdempotencyRedis(opts =>
    {
        opts.ConnectionString = configuration.GetConnectionString("Redis");
    })
    .AddEventBus()
        .AddConsumerServices(consumers =>
        {
            consumers.Subscribe<OrderCreatedEvent, OrderCreatedEventHandler>(); // cross-consumer
        })
        .AddConsumerRetryPolicies(configuration.GetSection("RetryPolicies"))
        .AddRabbitMQ(rabbitMQ =>
        {
            rabbitMQ.AddConnection("rabbitmq", configuration.GetSection("RabbitMQ"))
                    .AddConsumer("orders", "orders-queue", "rabbitmq", consumer =>
                    {
                        consumer.Subscribe<OrderCreatedEvent, OrderCreatedEventHandler>(); // per-consumer
                    });
        });

```

---

## Why IIntegrationEvent Is Required on the Consuming Side

The outbox accepts any `IMessage` for publishing, but the broker consumer infrastructure
requires `IIntegrationEvent` for routing. Here is why:

When a message arrives from the broker, `IntegrationEventDispatcher` receives raw bytes.
It must determine **which .NET type** to deserialize to, and **which handler** to invoke.
It does this using `IIntegrationEvent.EventName` combined with a type registry populated
at startup by `Subscribe<TEvent, THandler>()`.

```
Incoming bytes
    │
    ▼
IntegrationEventDispatcher reads EventName header
    │
    ▼
Type registry lookup: EventName → typeof(OrderCreatedEvent)
    │
    ▼
Deserialize bytes → OrderCreatedEvent instance
    │
    ▼
Invoke OrderCreatedEventHandler.HandleAsync(event)
```

A plain `IMessage` does not carry `EventName`, so the dispatcher cannot identify its type
from raw bytes. Therefore:

- **Publishing**: any `IMessage` works — the outbox stores the full type information
- **Consuming**: the event record **must implement `IIntegrationEvent`** so `EventName` is available

> **Practical implication**: If service B needs to consume an event that service A publishes
> as a plain domain event, service B must create an `IntegrationEvent` subtype, OR
> the event implements `IIntegrationEvent` to register with the subscriptions manager 
> with the subscription key is the service A domain event name

---

## IIntegrationEventHandler\<T\> — Writing a Handler

Implement `IIntegrationEventHandler<TEvent>` for each event type your service consumes.
Handlers are plain classes — inject any services you need via constructor.

```csharp {linenos=false,hl_lines=[1,7],linenostart=1}
public class OrderCreatedEventHandler : IIntegrationEventHandler<OrderCreatedEvent>
{
    private readonly ILogger<OrderCreatedEventHandler> _logger;
    private readonly IOrderRepository _orders;

    public OrderCreatedEventHandler(
        ILogger<OrderCreatedEventHandler> logger,
        IOrderRepository orders)
    {
        _logger = logger;
        _orders = orders;
    }

    public async Task HandleAsync(OrderCreatedEvent @event)
    {
        _logger.LogInformation("Processing order {OrderId}", @event.OrderId);
        await _orders.RecordReceivedAsync(@event.OrderId, @event.CreatedAt);
    }
}
```

**Register the handler** in DI and subscribe it in `AddConsumerServices()`:

```csharp {linenos=false,hl_lines=[1,4],linenostart=1}
services.AddTransient<OrderCreatedEventHandler>();

// Inside AddConsumerServices (shown in DI Registration above):
consumers.Subscribe<OrderCreatedEvent, OrderCreatedEventHandler>();
```

---

## MessageContext — Header Propagation

`MessageContext` carries correlation, causation, and tenant identity across service
boundaries. It must be initialized at an entry point. `IntegrationEventDispatcher` calls `MessageContext.Initialize()` automatically
from incoming RabbitMQ headers **before** your handler is invoked.

| Incoming header | MessageContext field | Description |
|---|---|---|
| `x-correlation-id` | `CorrelationId` | Trace ID that spans the entire request chain |
| `x-causation-id` | `CausationId` | ID of the message that caused this event |
| `x-tenant-id` | `TenantId` | Tenant identifier for multi-tenant routing |

---

## Automatic Idempotency in IntegrationEventDispatcher

`IntegrationEventDispatcher` calls `IIdempotencyService.TryCreateRequestAsync(messageId, eventName)`
**before** invoking any handler. If the `MessageId` already exists, the message is silently
acknowledged and the handler is **not** invoked. No handler code changes are needed.

> **Key point**: The same `IIdempotencyService` registration (Redis, EF, or caching) that
> guards `IIdempotentRequest` commands on the MediatR side also guards integration event
> handlers on the consume side. One backend registration covers both.

---


## Health Check

```csharp {linenos=false,hl_lines=[2],linenostart=1}
services.AddHealthChecks()
    .AddRabbitMQHealthCheck("rabbitmq", tags: new[] { "ready" });
```

Include the `ready` tag in your readiness probe. The check verifies that the named
RabbitMQ connection (`"rabbitmq"`) can be established. If it returns `Unhealthy`, the
service will not receive new messages until connectivity is restored.

**Partial-setup health check** (consumption without delivery):
If your service only consumes (no outbox or delivery), register only
`AddRabbitMQHealthCheck` — skip `AddOutboxDeliveryHealthCheck` (no delivery backlog to monitor).

---

## See also

- [Full Setup]({{< ref "core/messaging/full-setup/_index.md" >}}) — combine consumption + outbox + delivery in one registration
- [Internal Messaging Setup]({{< ref "core/messaging/_index.md" >}}) — idempotency backend selection
- [Event bus v8.5.0 archive]({{< ref "core/eventbus/v8.5.0/_index.md" >}}) — original `IIntegrationEventHandler` registration docs
- [Integration service v8.5.0 archive]({{< ref "core/integration/v8.5.0/_index.md" >}}) — original `IIntegrationEventService` docs
