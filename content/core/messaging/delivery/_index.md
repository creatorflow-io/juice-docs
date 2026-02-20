---
title: "Delivery Setup"
date: "2026-02-19"
weight: 3
draft: false
---

## Overview

The Delivery setup registers `DeliveryHostedService`, a background worker that reads
`OutboxDelivery` records from the database and forwards them to the broker transport
(e.g., RabbitMQ). It operates independently of the outbox staging layer and can run in
the same process or in a dedicated worker service.

---

## When to use

- You have [Outbox Setup]({{< ref "core/messaging/outbox/_index.md" >}}) configured and need a delivery worker
- You want to run the delivery worker in a separate process from the outbox-writing service
- You need retry backoff, stuck-message recovery, and a health check for the delivery backlog

---

## Prerequisites

[Outbox Setup]({{< ref "core/messaging/outbox/_index.md" >}}) — the `OutboxEvent` and
`OutboxDelivery` tables must exist and be populated by `IOutboxService<TContext>`.

---

## NuGet packages

| Package | Purpose |
|---|---|
| [Juice.Messaging.Outbox.Delivery](https://www.nuget.org/packages/Juice.Messaging.Outbox.Delivery) | `DeliveryHostedService`, `AddDelivery()` sub-builder, intent strategies |
| [Juice.EventBus.RabbitMQ](https://www.nuget.org/packages/Juice.EventBus.RabbitMQ) | RabbitMQ transport producer (`ITransportPublisher`) |

---

## DI Registration

```csharp {linenos=false,hl_lines=[4,5,6,7,8,9,10,11],linenostart=1}
services.AddMessaging(builder =>
        .AddDelivery(delivery =>
        {
            delivery.AddDeliveryProcessor<AppDbContext>("rabbitmq", // the producer key
                "send-pending", "retry-failed", "recover-timeout" // default intents
                );
            delivery.AddDeliveryPolicies(configuration.GetSection("DeliveryPolicies"));
            delivery.EventBus.AddRabbitMQ(rabbitMQ =>
            {
                rabbitMQ.AddConnection("rabbitmq", configuration.GetSection("RabbitMQ"))
                        .AddProducer("rabbitmq", "rabbitmq");
            });
        });
```

> **Important**: `AddDelivery()` automatically calls `services.AddEventBus()` in its
> constructor. Do **not need** to call `services.AddEventBus()` separately when using
> `AddDelivery()`.

---

## Intent Strategies

Intent strategies define what work `DeliveryHostedService` performs on each polling cycle.
Pass them to `AddDeliveryProcessor<TContext>()` in the order you want them evaluated.

| Strategy | Trigger condition | Purpose |
|---|---|---|
| `send-pending` | `OutboxDelivery.State == NotPublished` | Pick up newly staged messages and publish them |
| `retry-failed` | `State == Failed` AND `NextAttemptOn ≤ UtcNow` | Re-attempt delivery after the backoff delay has elapsed |
| `recover-timeout` | `State == InProgress` AND stuck for > 15 minutes | Reset timed-out in-progress records to `Failed` so they can be retried |

All three strategies are recommended for production. Each runs as a keyed, independently
configurable hosted service intent.

---

## Retry Backoff Schedule

The default policy applies exponential-style delays between retry attempts:

| Attempt | Delay before retry |
|---|---|
| 1st retry | 10 seconds |
| 2nd retry | 5 minutes |
| 3rd retry | 10 minutes |
| 4th retry | 30 minutes |
| 5th+ retry | 1 hour |

**Override with a custom policy** via `appsettings.json`:

```json {linenos=false,hl_lines=[3,4,5,6,7],linenostart=1}
{
  "DeliveryPolicies": {
    "RetryDelays": [
      "00:00:30",
      "00:05:00",
      "00:15:00",
      "01:00:00"
    ]
  }
}
```

---

## OutboxDelivery State Machine

```
NotPublished
    │
    ▼  (send-pending — picks up new records)
InProgress
    │
    ├──► Published   (broker ACK received — terminal state)
    │
    ├──► Failed      (publish error; NextAttemptOn = UtcNow + backoff delay)
    │       │
    │       └──► InProgress  (retry-failed — when NextAttemptOn ≤ UtcNow)
    │
    └──► Failed      (recover-timeout — InProgress for > 15 min is reset to Failed)

DeliveryState values: NotPublished=0, InProgress=1, Published=2, Failed=3, Skipped=4
```

Messages that fail permanently (e.g., max retries exhausted and no further `retry-failed`
triggers) remain in `Failed` state for audit. `Skipped` is used when a publishing policy
has no matching route.

---

## Health Check

Register `AddOutboxDeliveryHealthCheck` to expose a `live` health endpoint that alerts
when messages are stuck in `InProgress` longer than the configured threshold:

```csharp {linenos=false,hl_lines=[2,3,4,5],linenostart=1}
services.AddHealthChecks()
    .AddOutboxDeliveryHealthCheck<AppDbContext>(opts =>
    {
        opts.StuckMessageThresholdMinutes = 15;
        opts.MaxStuckMessages = 50;
    }, tags: new[] { "live" });
```

**Monitoring guidance**: Include the `live` tag in your liveness probe. If
`AddOutboxDeliveryHealthCheck` returns `Unhealthy`, investigate whether:
- `recover-timeout` intent is registered and running
- The broker connection is available
- The `DeliveryHostedService` itself is alive (check `IHostedService` registration)

**Partial-setup health check** (outbox without delivery worker):
If you run only the Outbox setup (no delivery worker in this process), skip the health
check — there are no `InProgress` records to monitor in a process that doesn't run
`DeliveryHostedService`.

---

## See also

- [Consumption Setup]({{< ref "core/messaging/consumption/_index.md" >}}) — add a RabbitMQ consumer to the same service
- [Full Setup]({{< ref "core/messaging/full-setup/_index.md" >}}) — combine outbox + delivery + consumption in one registration
