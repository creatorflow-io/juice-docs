# Page Content Schema: v9 Messaging Documentation

**Phase**: 1 — Design
**Branch**: `001-v9-messaging-docs`
**Date**: 2026-02-18

This contract defines the required content sections and code example format for each
new documentation page. Implementors MUST follow this structure to pass the FR-011
(complete code examples), FR-012 (NuGet links), and FR-013 (navigation) requirements.

---

## US1 — `content/core/messaging/_index.md` (Messaging Overview + Internal Setup)

```
Front matter: chapter=true, weight=7, title="Messaging"

## IMessage Hierarchy
[ASCII hierarchy + publish-vs-consume table]

## Setup Guides
[Links to outbox, delivery, consumption, full-setup pages]

## Internal Messaging Setup
### When to use
### Prerequisites
### NuGet packages
  - Juice.Messaging
  - Juice.MediatR.Behaviors
  - Juice.MediatR.Contracts
  - Juice.Messaging.Idempotency.[Backend]

### DI Registration
```csharp {hl_lines=[3,5,6]}
services.AddMessaging(builder =>
{
    builder.AddIdempotencyRedis(opts => { ... }); // or InMemory/EF
});
services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(MyHandler).Assembly);
    cfg.AddOpenBehavior(typeof(IdempotencyRequestBehavior<,>));
});
```

### IIdempotentRequest (replaces IRequestManager)
[Interface definition + usage example]

### IdempotencyBehavior pipeline
[Explanation of how behavior intercepts IIdempotentRequest commands]

### Idempotency backend selection
[Comparison table: In-memory / Distributed Cache / Redis / EF]

### See also
[Links to Outbox Setup, v8.5.0 archive]
```

---

## US2 — `content/core/messaging/outbox/_index.md` (Outbox Setup)

```
Front matter: weight=2, title="Outbox Setup"

## Overview
## When to use
## Prerequisites
  - Internal Messaging Setup (for TransactionBehavior DbContext)

## NuGet packages
  - Juice.Messaging
  - Juice.Messaging.Outbox.EF
  - Juice.MediatR.Behaviors

## DI Registration
```csharp {hl_lines=[3,4,5]}
services.AddMessaging(builder =>
{
    builder.AddPublishingPolicies(config.GetSection("PublishingPolicies"));
    builder.AddOutbox(outbox =>
    {
        outbox.AddOutboxRepository();
        outbox.AddDeliveryIntents();
    });
});
services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(MyHandler).Assembly);
    cfg.AddOpenBehavior(typeof(TransactionBehavior<,,>));  // subclassed
});
```

## IOutboxService — staging events
[AddEventAsync + SaveEventsAsync explanation]

## Publishing any IMessage (including domain events)
[Code example: stage a domain event WITHOUT converting to IntegrationEvent]

## IMessagePublishingPolicy — routing configuration
[appsettings.json PublishingPolicies section example]

## TransactionBehavior — subclassing guide
[Code example: concrete AppTransactionBehavior<T,R>]

## OutboxEvent + OutboxDelivery tables
[Brief explanation: two tables, migrations required]

## See also
[Links to Delivery Setup, Full Setup, v8.5.0 archive]
```

---

## US3 — `content/core/messaging/delivery/_index.md` (Delivery Setup)

```
Front matter: weight=3, title="Delivery Setup"

## Overview
## When to use
## Prerequisites
  - Outbox Setup

## NuGet packages
  - Juice.Messaging.Outbox.Delivery
  - Juice.EventBus.RabbitMQ (for broker transport)

## DI Registration
```csharp {hl_lines=[3,4,5,6]}
services.AddMessaging(builder =>
{
    builder.AddDelivery(delivery =>
    {
        delivery.AddDeliveryProcessor<AppDbContext>("rabbitmq",
            "send-pending", "retry-failed", "recover-timeout");
        delivery.AddDeliveryPolicies(config.GetSection("DeliveryPolicies"));
        delivery.EventBus.AddRabbitMQ(rabbitMQ =>
        {
            rabbitMQ.AddConnection("rabbitmq", config.GetSection("RabbitMQ"))
                    .AddProducer("rabbitmq", "rabbitmq");
        });
    });
});
```

## Intent strategies
[Table: send-pending / retry-failed / recover-timeout — trigger + purpose]

## Retry backoff schedule
[Default table: 10s → 5m → 10m → 30m → 1h + override example]

## OutboxDelivery state machine
[Diagram: NotPublished → InProgress → Published/Failed]

## Health check
```csharp {hl_lines=[2]}
services.AddHealthChecks()
    .AddOutboxDeliveryHealthCheck<AppDbContext>(opts =>
    {
        opts.StuckMessageThresholdMinutes = 15;
        opts.MaxStuckMessages = 50;
    }, tags: new[] { "live" });
```

## See also
[Links to Consumption Setup, Full Setup]
```

---

## US4 — `content/core/messaging/consumption/_index.md` (Consumption Setup)

```
Front matter: weight=4, title="Consumption Setup"

## Overview
## When to use
## Prerequisites
  - Internal Messaging Setup (for IIdempotencyService)

## NuGet packages
  - Juice.EventBus
  - Juice.EventBus.RabbitMQ
  - Juice.Messaging.Idempotency.[Backend]

## DI Registration
```csharp {hl_lines=[3,5,6,7,8,9,10]}
services.AddMessaging(builder =>
{
    builder.AddIdempotencyRedis(opts => { ... });
    builder.AddDelivery(delivery =>
    {
        delivery.EventBus
            .AddConsumerServices(consumers =>
            {
                consumers.Subscribe<OrderCreatedEvent, OrderCreatedHandler>();
            })
            .AddConsumerRetryPolicies(config.GetSection("RetryPolicies"))
            .AddRabbitMQ(rabbitMQ =>
            {
                rabbitMQ.AddConnection("rabbitmq", config.GetSection("RabbitMQ"))
                        .AddConsumer("orders", "orders-queue", "rabbitmq", consumer =>
                        {
                            consumer.WithDeadLetterExchange("dlx.orders",
                                routingPattern: "{0}.parking");
                        });
            });
    });
});
```

## Why IIntegrationEvent is required on the consuming side
[Explanation: EventName + type registry deserialization]

## IIntegrationEventHandler<T> — writing a handler
[Code example: complete handler + registration]

## MessageContext — header propagation
[Table: x-correlation-id / x-causation-id / x-tenant-id → MessageContext fields]

## Automatic idempotency in IntegrationEventDispatcher
[Explanation: no handler code changes needed]

## Dead-letter exchange configuration
[DLX routing + x-death headers]

## Health check
```csharp {hl_lines=[2]}
services.AddHealthChecks()
    .AddRabbitMQHealthCheck("rabbitmq", tags: new[] { "ready" });
```

## See also
[Links to Full Setup, v8.5.0 archive]
```

---

## US5 — `content/core/messaging/full-setup/_index.md` (Full Pipeline)

```
Front matter: weight=5, title="Full Pipeline Setup"

## Overview
## Prerequisites
  - Complete all four setup guides OR follow this guide from scratch

## NuGet packages
  - All packages from US1–US4 combined

## Sub-builder registration order
[Numbered list explaining WHY order matters]

## Complete DI Registration
```csharp {hl_lines=[2,5,8,11,15,16,21,22]}
services.AddMediatR(cfg => { /* handlers + behaviors */ });
services.AddMessaging(builder =>
{
    // 1. Publishing policies (needed by outbox at staging time)
    builder.AddPublishingPolicies(config.GetSection("PublishingPolicies"));

    // 2. Outbox (must precede delivery)
    builder.AddOutbox(outbox =>
    {
        outbox.AddOutboxRepository();
        outbox.AddDeliveryIntents();
    });

    // 3. Idempotency backend (before EventBus dispatcher)
    builder.AddIdempotencyRedis(opts => { ... });

    // 4. Delivery (creates EventBus internally)
    builder.AddDelivery(delivery =>
    {
        delivery.AddDeliveryProcessor<AppDbContext>("rabbitmq",
            "send-pending", "retry-failed", "recover-timeout");
        delivery.AddDeliveryPolicies(config.GetSection("DeliveryPolicies"));
        delivery.EventBus
            .AddConsumerServices(consumers =>
            {
                consumers.Subscribe<OrderCreatedEvent, OrderCreatedHandler>();
            })
            .AddRabbitMQ(rabbitMQ =>
            {
                rabbitMQ.AddConnection("rabbitmq", config.GetSection("RabbitMQ"))
                        .AddProducer("rabbitmq", "rabbitmq")
                        .AddConsumer("orders", "orders-queue", "rabbitmq");
            });
    });
});
services.AddHealthChecks()
    .AddRabbitMQHealthCheck("rabbitmq", tags: new[] { "ready" })
    .AddOutboxDeliveryHealthCheck<AppDbContext>(..., tags: new[] { "live" });
```

## Migration table (v8.5.0 → v9)
[Full table from research R-010]

## See also
[Links to all four individual guides + v8.5.0 archive]
```

---

## Archive Pages — Banner Contract

Every archive page MUST start with this banner (immediately after front matter):

```markdown
> **⚠ Archived — v8.5.0**
> This page describes the pre-v9 Juice API.
> For current documentation see [Messaging]({{< ref "core/messaging/_index.md" >}}).
```

The rest of the page content is the verbatim original, unchanged.
