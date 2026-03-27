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

### Minimal setup

```csharp {linenos=false,hl_lines=[4,5,6,7,8,9,10,11],linenostart=1}
services.AddMessaging()
    .AddDelivery(delivery =>
    {
        delivery.AddDeliveryProcessor<AppDbContext>("rabbitmq"); // default intents applied automatically
        delivery.AddDeliveryPolicies(configuration.GetSection("DeliveryPolicies"));
        delivery.EventBus.AddRabbitMQ(rabbitMQ =>
        {
            rabbitMQ.AddConnection("rabbitmq", configuration.GetSection("RabbitMQ"))
                    .AddProducer("rabbitmq", "rabbitmq");
        });
    });
```

### Per-processor policies

Use the delegate overload of `AddDeliveryProcessor<TContext>` to configure a policy
that applies only to that processor, independently of the global policy:

```csharp {linenos=false,linenostart=1}
delivery.AddDeliveryProcessor<AppDbContext>("rabbitmq", proc =>
{
    proc.AddDeliveryPolicies(opts =>
    {
        opts.DefaultPolicy = new PolicyConfiguration
        {
            BatchSize = 50,
            Interval = TimeSpan.FromSeconds(10)
        };
    });
});
```

Processor-scoped policies sit below global key-matched entries in the resolution chain
but above the global `DefaultPolicy`. See [Delivery Policies](#delivery-policies) for
the full resolution order.

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
| `recover-timeout` | `State == InProgress` AND stuck for > configured timeout | Reset timed-out in-progress records to `Failed` so they can be retried |

All three strategies are registered automatically when using `AddDeliveryProcessor<TContext>(publisher)`.

---

## Delivery Policies

A `DeliveryPolicy` controls how a processor polls and retries messages. Every field has
a built-in default; you only need to specify the fields you want to override.

### Built-in defaults

| Field | Default | Description |
|---|---|---|
| `Interval` | 5 seconds | Polling interval between batch processing cycles |
| `BatchSize` | 10 | Records processed per cycle |
| `Timeout` | 5 minutes | Age threshold before a stuck `InProgress` record is recovered |
| `InitialRetryDelay` | 5 seconds | Delay before the first retry attempt |
| `RetryDelayMultiplier` | 2.0 | Exponential backoff multiplier |
| `MaxRetryAttempts` | 3 | Number of retry attempts before the record stays in `Failed` |

### Policy resolution order

When `DeliveryHostedService` requests a policy for a given context, `IDeliveryPolicyResolver`
checks sources in this order and uses the first match:

| Priority | Source | Key / condition |
|---|---|---|
| 1 (highest) | Global `Policies` dictionary — exact match | `"{publisher}:{intent}:{contextName}"` |
| 2 | Global `Policies` dictionary — wildcard context | `"{publisher}:{intent}:*"` |
| 3 | Global `Policies` dictionary — wildcard intent + context | `"{publisher}:*:*"` |
| 4 | Global `Policies` dictionary — wildcard publisher + context | `"*:{intent}:*"` |
| 5a | Processor `Policies` dictionary — same four wildcard levels as above | keys scoped to this processor |
| 5b | Processor `DefaultPolicy` | fallback within the processor scope |
| 6 | Global `DefaultPolicy` | Set via `delivery.AddDeliveryPolicies(opts => ...)` |
| 7 (lowest) | Built-in defaults | `DeliveryPolicy.Default` |

Each matched `PolicyConfiguration` inherits any unset fields from the next fallback:
- A processor `Policies` entry inherits from the processor `DefaultPolicy`, which inherits from the global `DefaultPolicy`.
- A global `Policies` entry inherits directly from the global `DefaultPolicy`.

### Global policies via configuration

Configure shared policies in `appsettings.json`. In JSON, replace `:` in key names
with `__` (double underscore):

```json {linenos=false,linenostart=1}
{
  "DeliveryPolicies": {
    "DefaultPolicy": {
      "Interval": "00:00:05",
      "BatchSize": 10
    },
    "Policies": {
      "rabbitmq__send-pending__AppDbContext": {
        "BatchSize": 20,
        "Interval": "00:00:03"
      },
      "rabbitmq__send-pending__*": {
        "BatchSize": 15
      },
      "rabbitmq__*__*": {
        "Interval": "00:00:10"
      }
    }
  }
}
```

Register the section with `AddDeliveryPolicies`:

```csharp {linenos=false,linenostart=1}
delivery.AddDeliveryPolicies(configuration.GetSection("DeliveryPolicies"));
```

`AddDeliveryPolicies` is safe to call multiple times — each call merges its entries
into the shared options.

### Global policies via code

```csharp {linenos=false,linenostart=1}
delivery
    .AddDeliveryPolicies(opts =>
    {
        opts.DefaultPolicy = new PolicyConfiguration { BatchSize = 20 };
        opts.Policies["rabbitmq:retry:*"] = new PolicyConfiguration { BatchSize = 5 };
    })
    .AddDeliveryPolicies(opts =>
    {
        opts.Policies["rabbitmq:send-pending:*"] = new PolicyConfiguration { Interval = TimeSpan.FromSeconds(3) };
    });
```

### Per-processor code policy

A processor-scoped policy applies only to that processor type and sits below all global
key-matched entries (priorities 1–4). Use it when a specific `TContext` needs its own
policy without touching the global configuration.

The processor `Policies` dictionary supports the same four wildcard levels as the global
one, allowing intent-specific overrides within the processor scope:

```csharp {linenos=false,linenostart=1}
delivery.AddDeliveryProcessor<SlowDbContext>("rabbitmq", proc =>
{
    proc.AddDeliveryPolicies(opts =>
    {
        // Intent-specific overrides for this processor
        opts.Policies["rabbitmq:send-pending:*"] = new PolicyConfiguration { BatchSize = 20 };
        opts.Policies["rabbitmq:retry:*"]        = new PolicyConfiguration { BatchSize = 2 };

        // Fallback for any intent not listed above
        opts.DefaultPolicy = new PolicyConfiguration { Interval = TimeSpan.FromSeconds(30) };
    });
});
```

Unset fields in a processor `Policies` entry inherit from the processor `DefaultPolicy`,
which in turn inherits from the global `DefaultPolicy`. A global key-matched entry
(priorities 1–4) always overrides any processor-scoped policy.

---

## Retry Backoff Schedule

The default policy uses exponential backoff:
`delay = InitialRetryDelay × RetryDelayMultiplier^(attempt - 1)`

With built-in defaults (`InitialRetryDelay = 5s`, `RetryDelayMultiplier = 2.0`):

| Attempt | Delay before retry |
|---|---|
| 1st retry | 5 seconds |
| 2nd retry | 10 seconds |
| 3rd retry | 20 seconds |

Records that exhaust `MaxRetryAttempts` (default 3) remain in `Failed` state for audit.

**Override the retry behaviour** via a processor or global policy:

```csharp {linenos=false,linenostart=1}
delivery.AddDeliveryPolicies(opts =>
    opts.DefaultPolicy = new PolicyConfiguration
    {
        InitialRetryDelay = TimeSpan.FromSeconds(30),
        RetryDelayMultiplier = 3.0,
        MaxRetryAttempts = 5
    });
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
    └──► Failed      (recover-timeout — InProgress for > configured timeout is reset to Failed)

DeliveryState values: NotPublished=0, InProgress=1, Published=2, Failed=3, Skipped=4
```

Messages that fail permanently remain in `Failed` state for audit. `Skipped` is used
when a publishing policy has no matching route.

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

- [Local Transport]({{< ref "core/messaging/local-transport/_index.md" >}}) — in-process dispatch via `"local-channel"` and `"local"` routes
- [Consumption Setup]({{< ref "core/messaging/consumption/_index.md" >}}) — add a RabbitMQ consumer to the same service
- [Full Setup]({{< ref "core/messaging/full-setup/_index.md" >}}) — combine outbox + delivery + consumption in one registration
