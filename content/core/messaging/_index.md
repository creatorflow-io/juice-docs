+++
title = "Messaging"
date = "2026-02-19"
weight = 6
chapter = true
+++

Juice v9 unifies all message types under a single `IMessage` hierarchy and introduces a
`MessagingBuilder` entry point that composes the outbox, delivery, idempotency, and
broker layers with a single `services.AddMessaging()` call.

## IMessage Hierarchy

Every message type in Juice v9 — MediatR commands, domain events, and integration events —
implements `IMessage`, giving them a shared identity (`MessageId`, `CreatedAt`, `TenantId`).

```
IMessage
├── MessageId : Guid
├── CreatedAt : DateTimeOffset
└── TenantId  : string?

IEvent : IMessage
└── EventName : string

IIntegrationEvent : IEvent          ← required only on the CONSUMING side
└── (marker — enables dispatcher routing)

MessageBase (abstract record) : IMessage
└── MessageId = Guid.NewGuid(), CreatedAt = UtcNow (defaults)

IntegrationEvent (abstract record) : MessageBase, IIntegrationEvent
└── EventName → GetType().Name (default)    ← optional to customizing the routing key on the PRODUCER side
```

### Publish vs. Consume roles

| Role | Type required | Reason |
|---|---|---|
| **Publishing** (outbox staging) | Any `IMessage` | Outbox accepts any message including plain domain events |
| **Consuming** (broker dispatch) | Must implement `IIntegrationEvent` | `IntegrationEventDispatcher` uses `EventName` + type registry to deserialize and route from raw bytes |

> **Key insight**: A plain domain event (`IMessage`) can be published and delivered through
> the outbox without ever implementing `IIntegrationEvent` if you do not want to customizing the event name.
> Only services that need to *receive* it via broker infrastructure must use `IIntegrationEvent`.

---

## Setup Guides

Choose the guide that matches your service's needs. Each is independently followable.

| Guide | What it sets up |
|---|---|
| [MessageContext]({{< ref "core/messaging/message-context/_index.md" >}}) | `MessageContext` initialization via middleware, `[MessageContext]` attribute, or `[InitializeMessageContext]` for tests |
| [Mediator]({{< ref "core/mediator/_index.md" >}}) | MediatR Request/Response, Notification, and Stream Request patterns with handler implementation |
| [Outbox Setup]({{< ref "core/messaging/outbox/_index.md" >}}) | Transactional staging of any `IMessage` inside a MediatR `TransactionBehavior` |
| [Delivery Setup]({{< ref "core/messaging/delivery/_index.md" >}}) | Background `DeliveryHostedService` that forwards staged messages to the broker |
| [Local Transport]({{< ref "core/messaging/local-transport/_index.md" >}}) | In-process dispatch via `IMessageService` — `"local-channel"` (non-durable) and `"local"` (outbox-backed) routes with immediate dispatch and idempotency deduplication |
| [Consumption Setup]({{< ref "core/messaging/consumption/_index.md" >}}) | RabbitMQ consumer engine, `IIntegrationEventHandler<T>` dispatch, and automatic idempotency |
| [Full Setup]({{< ref "core/messaging/full-setup/_index.md" >}}) | All sub-builders combined in one `AddMessaging()` call, with migration table |

---

## How It All Fits Together

The diagram below shows how a message flows from an HTTP request through the messaging
pipeline to its final destination — in-process handler or broker.

```mermaid
flowchart TB
    subgraph Entry["Entry Points"]
        HTTP["HTTP Request"]
        Consumer["RabbitMQ Consumer"]
        Test["xUnit Test"]
    end

    subgraph MC["MessageContext (AsyncLocal)"]
        direction LR
        CID["CorrelationId"]
        CAID["CausationId"]
        EID["ExecutionId"]
        SRC["Source"]
    end

    subgraph Init["MessageContext Initialization"]
        MW["MessageContextMiddleware<br/><small>global — all requests</small>"]
        Attr["[MessageContext] attribute<br/><small>per controller/action</small>"]
        TestAttr["[InitializeMessageContext]<br/><small>xUnit before/after</small>"]
        AutoConsumer["RabbitMQConsumerEngine<br/><small>from message headers</small>"]
    end

    HTTP --> MW & Attr
    Consumer --> AutoConsumer
    Test --> TestAttr
    MW & Attr & TestAttr & AutoConsumer --> MC

    subgraph Publish["Publishing"]
        IMS["IMessageService.PublishAsync()"]
        Policy["IMessagePublishingPolicy<br/><small>resolves routes from config</small>"]
        IMS --> Policy
    end

    MC --> Publish

    subgraph Routes["Route Resolution"]
        LC["local-channel<br/><small>non-durable</small>"]
        LO["local<br/><small>durable, in-process</small>"]
        BR["rabbitmq<br/><small>durable, cross-service</small>"]
    end

    Policy --> LC & LO & BR

    subgraph Dispatch["Dispatch"]
        Channel["Channel&lt;IMessage&gt;<br/><small>in-memory queue</small>"]
        Outbox["OutboxEvent +<br/>OutboxDelivery<br/><small>DB tables</small>"]
        BgSvc["LocalChannel<br/>BackgroundService"]
        DeliverySvc["DeliveryHostedService"]
        LTP["LocalTransport<br/>Publisher"]
        RMQ["RabbitMQProducer"]
        Handler["IIntegrationEventHandler&lt;T&gt;<br/>INotificationHandler&lt;T&gt;"]
        Broker["RabbitMQ Broker"]
    end

    LC --> Channel
    LO --> Outbox
    LO -.->|"immediate<br/>best-effort"| Channel
    BR --> Outbox

    Channel --> BgSvc --> Handler
    Outbox --> DeliverySvc
    DeliverySvc -->|"key=local"| LTP --> Handler
    DeliverySvc -->|"key=rabbitmq"| RMQ --> Broker

    subgraph Headers["Outbox Headers (from MessageContext)"]
        direction LR
        H1["x-correlation-id"]
        H2["x-causation-id"]
        H3["x-source"]
        H4["x-tenant-id"]
        H5["x-message-id"]
    end

    MC -.->|"stamped into"| Headers
    Headers -.->|"stored in"| Outbox
```

> **Key concept**: `MessageContext` is initialized once at the entry point, flows through
> all async operations via `AsyncLocal`, and is stamped into outbox headers. When a message
> crosses a service boundary (via broker) or is retried (via `LocalTransportPublisher`),
> `MessageContext` is **restored from these headers** — preserving correlation across the
> entire chain.

---

## Internal Messaging Setup

Use this setup when your service only needs in-process MediatR command deduplication —
no external broker, no outbox table, no delivery worker.

### When to use

- Service handles `IIdempotentRequest` commands from an HTTP API or message queue handler
- No need to publish events to other services from this registration
- You want idempotent command handling without any infrastructure dependencies

### Prerequisites

None. This is the simplest, self-contained setup.

### NuGet packages

| Package | Purpose |
|---|---|
| [Juice.Messaging](https://www.nuget.org/packages/Juice.Messaging) | `MessagingBuilder` entry point, `IMessage` hierarchy |
| [Juice.MediatR.Behaviors](https://www.nuget.org/packages/Juice.MediatR.Behaviors) | `IdempotencyRequestBehavior<,>`, `TransactionBehavior<,,>` |
| [Juice.MediatR.Contracts](https://www.nuget.org/packages/Juice.MediatR.Contracts) | `IIdempotentRequest` interface |
| [Juice.Messaging.Idempotency.Redis](https://www.nuget.org/packages/Juice.Messaging.Idempotency.Redis) | Redis idempotency backend (recommended for production) |
| [Juice.Messaging.Idempotency.Caching](https://www.nuget.org/packages/Juice.Messaging.Idempotency.Caching) | In-memory or distributed cache backend |
| [Juice.Messaging.Idempotency.EF](https://www.nuget.org/packages/Juice.Messaging.Idempotency.EF) | EF idempotency backend (auditable / SQL-required) |

### DI Registration

```csharp {linenos=false,hl_lines=[3,8,9],linenostart=1}
services.AddMessaging(builder =>
{
    builder.AddIdempotencyRedis(opts =>
    {
        opts.Configuration = configuration.GetConnectionString("Redis");
    });
});

services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(MyHandler).Assembly);
    cfg.AddIdempotencyRequestBehavior();
});
```

---

### IIdempotentRequest (replaces IRequestManager)

`IIdempotentRequest` is the v9 replacement for the v8.5.0 `IRequestManager` + `IIdentifiedCommand<T>` pattern.
Mark your command with `IIdempotentRequest` and the `IdempotencyBehavior` handles deduplication automatically.

```csharp {linenos=false,hl_lines=[1,5],linenostart=1}
// In Juice.MediatR.Contracts (package: Juice.MediatR.Contracts)
public interface IIdempotentRequest : IBaseRequest
{
    // Caller-supplied unique key — generated by the sender, not the server
    string IdempotencyKey { get; }
}
```

**Usage example — mark your command directly:**

```csharp {linenos=false,hl_lines=[1,6],linenostart=1}
public record CreateOrderCommand(string OrderId, string CustomerId)
    : IRequest<IOperationResult>, IIdempotentRequest
{
    // IdempotencyKey is the caller-supplied unique identifier
    // The framework calls IIdempotencyService automatically — no handler changes needed
    public string IdempotencyKey => OrderId;
}
```

**v8.5.0 → v9 comparison:**

| v8.5.0 | v9 |
|---|---|
| Wrap command: `new IdentifiedCommand<T>(cmd, id)` | Mark command directly with `IIdempotentRequest` |
| Inject `IRequestManager` into handler | No injection needed |
| Call `requestManager.ExistAsync(id)` manually | `IdempotencyBehavior` calls `IIdempotencyService` automatically |
| Register `IIdentifiedCommand<T>` handler | Register your command handler directly |

---

### IdempotencyBehavior Pipeline

`IdempotencyRequestBehavior<TRequest, TResponse>` intercepts any `IRequest<T>` that also
implements `IIdempotentRequest`. It runs at pipeline order `int.MinValue` — before all
other behaviors — so duplicate commands are rejected before any handler logic executes.

**How it works:**

1. Behavior checks `IIdempotencyService.TryCreateRequestAsync(key, typeName)`
2. If the key already exists → returns the cached result immediately (no handler invoked)
3. If the key is new → proceeds through the pipeline, then stores the result

> **Shared registration**: One `IIdempotencyService` registration serves both
> `IdempotencyBehavior` (MediatR pipeline) and `IntegrationEventDispatcher` (consumer side).
> No separate registration is needed for each.

---

### Idempotency Backend Selection

| Backend | Registration | Package | Use case |
|---|---|---|---|
| In-memory | `builder.AddIdempotencyInMemory()` | `Juice.Messaging.Idempotency.Caching` | Tests and development only |
| Distributed cache | `builder.AddIdempotencyDistributedCache()` | `Juice.Messaging.Idempotency.Caching` | Simple, non-critical workloads |
| **Redis** | `builder.AddIdempotencyRedis(opts => { ... })` | `Juice.Messaging.Idempotency.Redis` | **Production (recommended)** |
| EF | `builder.AddIdempotencyEF(config, opts => { ... })` | `Juice.Messaging.Idempotency.EF` | Auditable / SQL Server required |

---

## See also

- [MessageContext]({{< ref "core/messaging/message-context/_index.md" >}}) — middleware, attribute, and test initialization
- [Mediator]({{< ref "core/mediator/_index.md" >}}) — Request/Response, Notification, and Stream Request patterns
- [Outbox Setup]({{< ref "core/messaging/outbox/_index.md" >}}) — add transactional message staging
- [Local Transport]({{< ref "core/messaging/local-transport/_index.md" >}}) — in-process dispatch with `IMessageService`
- [Full Setup]({{< ref "core/messaging/full-setup/_index.md" >}}) — combine all sub-builders + migration table
- [MediatR v8.5.0 archive]({{< ref "core/mediator/v8.5.0/_index.md" >}}) — original `IRequestManager` documentation
- [Event bus v8.5.0 archive]({{< ref "core/eventbus/v8.5.0/_index.md" >}}) — original `IEventBus` documentation
