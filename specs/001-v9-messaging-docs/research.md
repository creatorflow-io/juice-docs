# Research: v9 Messaging Documentation Update

**Phase**: 0 — Research
**Branch**: `001-v9-messaging-docs`
**Date**: 2026-02-18

---

## R-001: MessagingBuilder Entry Point

**Decision**: Use `services.AddMessaging(builder => { ... })` as the single DI entry
point for all messaging configuration.

**Rationale**: `AddMessaging()` is the extension method in
`Microsoft.Extensions.DependencyInjection` (from `Juice.Messaging`) that returns a
`MessagingBuilder`. All sub-builders are accessed from this builder. It registers
the default `IMessageSerializer` (Newtonsoft.Json).

**Concrete API**:
```csharp
// Namespace: Microsoft.Extensions.DependencyInjection
// Package: Juice.Messaging
public static MessagingBuilder AddMessaging(
    this IServiceCollection services,
    Action<MessagingBuilder>? buildAction = default)
```

**Alternatives considered**: Direct `services.AddXxx()` calls — rejected because
they bypass the builder chain and break sub-builder ordering guarantees.

---

## R-002: Sub-Builder API Reference

**Decision**: Document each sub-builder's entry method and key registration calls
as the canonical setup pattern.

### Outbox Sub-Builder
- **Entry**: `messagingBuilder.AddOutbox(outbox => { ... })`
- **Package**: `Juice.Messaging.Outbox.EF`
- **Key methods**:
  - `outbox.AddOutboxRepository()` — binds `IOutboxRepository<TContext>`
  - `outbox.AddDeliveryIntents()` — registers three keyed intent strategies
  - `outbox.Intent<TContext, TOutboxIntent>()` — custom intent registration
- **Side effect**: `AddOutbox()` automatically registers `IOutboxService<TContext>` →
  `OutboxEventService<TContext>` via `AddOutboxCore()`.

### Delivery Sub-Builder
- **Entry**: `messagingBuilder.AddDelivery(delivery => { ... })`
- **Package**: `Juice.Messaging.Outbox.Delivery`
- **Key methods**:
  - `delivery.AddDeliveryProcessor<TContext>(publisherKey, intents…)` — registers processor
  - `delivery.AddDeliveryPolicies(config)` — retry backoff schedule
  - `delivery.EventBus` — property to access the auto-created `EventBusBuilder`
- **Critical note**: `DeliveryBuilder` constructor calls `services.AddEventBus()`
  automatically, so `EventBusBuilder` is always created inside the delivery context.

### EventBus Sub-Builder
- **Entry**: `delivery.EventBus` (accessed from `DeliveryBuilder`) OR
  `services.AddEventBus()` (standalone, without delivery)
- **Package**: `Juice.EventBus`
- **Key methods**:
  - `eventBus.AddPublishingServices()` — registers `IEventBus` → `CompositeEventPublisher`
  - `eventBus.AddConsumerServices(key)` — registers `IntegrationEventDispatcher` +
    `ISubscriptionsManager`
  - `eventBus.AddConsumerRetryPolicies(config)` — retry policy for consumer engine
  - `eventBus.Messaging` — back-reference to parent `MessagingBuilder`

### RabbitMQ Sub-Builder
- **Entry**: `delivery.EventBus.AddRabbitMQ(rabbitMQ => { ... })` OR
  `eventBus.AddRabbitMQ(...)` standalone
- **Package**: `Juice.EventBus.RabbitMQ`
- **Key methods**:
  - `rabbitMQ.AddConnection(name, config)` — registers keyed `IRabbitMQPersistentConnection`
  - `rabbitMQ.AddProducer(key, connectionName)` — keyed `ITransportPublisher`
  - `rabbitMQ.AddConsumer(key, queueName, connectionName, consumer => { ... })` — starts
    `RabbitMQConsumerHostedService`
- **On consumer builder**:
  - `.Subscribe<TEvent, THandler>()` — registers handler subscription
  - `.WithDeadLetterExchange(exchange, routingPattern)` — DLX configuration

---

## R-003: Sub-Builder Registration Order

**Decision**: Order MUST be: `AddMessaging` → policies → `AddOutbox` →
`AddIdempotency*` → `AddDelivery` (which creates EventBus) → RabbitMQ inside delivery.

**Rationale**: Confirmed from test code in
`Juice.EventBus.Tests/DependencyInjection/EventBustTestServiceCollectionExtensions.cs`:
1. `AddMessaging()` first — establishes the builder
2. `AddPublishingPolicies()` — must precede outbox (outbox reads policy at staging time)
3. `AddOutbox()` — must precede `AddDelivery()` (delivery builder reads outbox config)
4. `AddIdempotency*()` — before EventBus/RabbitMQ so dispatcher can resolve it
5. `AddDelivery()` — creates `EventBusBuilder` internally via its constructor
6. RabbitMQ configured inside `delivery.EventBus.AddRabbitMQ()`

**Canonical order confirmed**:
```csharp
services
    .AddMessaging(builder =>
    {
        builder.AddPublishingPolicies(config.GetSection("PublishingPolicies"));
        builder.AddOutbox(outbox => { /* ... */ });
        builder.AddIdempotencyRedis(options => { /* ... */ });
        builder.AddDelivery(delivery =>
        {
            delivery.AddDeliveryProcessor<AppDbContext>("rabbitmq", "send-pending",
                "retry-failed", "recover-timeout");
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
```

**Alternatives considered**: Any other order — rejected; `DeliveryBuilder` constructor
auto-creates `EventBusBuilder`, so EventBus cannot be configured before delivery.

---

## R-004: MediatR Pipeline Behaviors

### IdempotencyBehavior
- **Class**: `IdempotencyRequestBehavior<TRequest, TResponse>` (two variants: with/without response)
- **Namespace**: `Juice.MediatR.Behaviors`
- **Package**: `Juice.MediatR.Behaviors`
- **Pipeline order**: `int.MinValue` (runs before all other behaviors)
- **Registration**:
  ```csharp
  services.AddMediatR(cfg =>
  {
      cfg.RegisterServicesFromAssembly(typeof(MyHandler).Assembly);
      cfg.AddOpenBehavior(typeof(IdempotencyRequestBehavior<,>));
  });
  ```
- **Trigger**: Any `IRequest<T>` that also implements `IIdempotentRequest`

### TransactionBehavior
- **Class**: `TransactionBehavior<TRequest, TResponse, TContext>` (abstract)
- **Namespace**: `Juice.MediatR.Behaviors`
- **Pipeline order**: `int.MaxValue - 20` (runs late — after validation, before actual handler)
- **Registration**: Must be subclassed per DbContext:
  ```csharp
  internal class AppTransactionBehavior<T, R>
      : TransactionBehavior<T, R, AppDbContext>
      where T : IRequest<R>
  {
      public AppTransactionBehavior(AppDbContext db,
          IIntegrationEventService<AppDbContext> svc, ILogger<...> log)
          : base(db, svc, log) { }
  }

  services.AddScoped(typeof(IPipelineBehavior<,>), typeof(AppTransactionBehavior<,>));
  ```

---

## R-005: IIdempotentRequest Interface

**Decision**: `IIdempotentRequest` replaces v8.5.0 `IRequestManager` for command deduplication.

**Concrete definition**:
```csharp
// Namespace: Juice.MediatR
// Package: Juice.MediatR.Contracts
public interface IIdempotentRequest : IBaseRequest
{
    /// <summary>
    /// The IdempotencyKey must be generated by the caller, not the server.
    /// Used to recognize and ignore duplicate requests.
    /// </summary>
    string IdempotencyKey { get; }
}
```

**Usage pattern**:
```csharp
public record CreateOrderCommand(string OrderId, ...) : IRequest<IOperationResult>,
    IIdempotentRequest
{
    public string IdempotencyKey => OrderId; // caller-supplied unique key
}
```

**v8.5.0 comparison**:
| v8.5.0 | v9 |
|--------|-----|
| Implement `IIdentifiedCommand<T>` | Implement `IIdempotentRequest` on your command |
| Inject `IRequestManager` into handler | No injection needed — `IdempotencyBehavior` handles it |
| Call `requestManager.ExistAsync(id)` | Automatic via pipeline behavior |

---

## R-006: Idempotency Backend Selection

**Decision**: Document all four backends; recommend Redis for production, EF for
persistence requirements, distributed cache for simple scenarios, in-memory for testing.

| Backend | Registration method | Package | Use case |
|---------|--------------------|---------|-|
| In-memory | `builder.AddIdempotencyInMemory()` | `Juice.Messaging.Idempotency.Caching` | Tests only |
| Distributed cache | `builder.AddIdempotencyDistributedCache()` | `Juice.Messaging.Idempotency.Caching` | Simple/non-critical |
| Redis | `builder.AddIdempotencyRedis(opts => { ... })` | `Juice.Messaging.Idempotency.Redis` | Production (recommended) |
| EF | `builder.AddIdempotencyEF(config, opts => { ... })` | `Juice.Messaging.Idempotency.EF` | Auditable / SQL-required |

**Shared registration**: One `IIdempotencyService` registration serves both
`IdempotencyRequestBehavior` (MediatR pipeline, blocks duplicate commands) and
`IntegrationEventDispatcher` (consumer side, blocks duplicate events). They share
the same service by scope. No separate registration needed.

---

## R-007: IMessage Hierarchy and Publish vs. Consume

**Decision**: Document the asymmetry explicitly: any `IMessage` can be published;
only `IIntegrationEvent` can be consumed by infrastructure handlers.

**Hierarchy confirmed from codebase**:
```
IMessage
├── MessageId: Guid
├── CreatedAt: DateTimeOffset
└── TenantId: string?

IEvent : IMessage
└── EventName: string

IIntegrationEvent : IEvent         ← required for infrastructure consumption
└── (marker interface for dispatcher routing)

MessageBase (abstract record) : IMessage
└── (default values: MessageId = Guid.NewGuid(), CreatedAt = UtcNow)

IntegrationEvent (abstract record) : MessageBase, IIntegrationEvent
└── EventName → GetType().Name by default
```

**Why the asymmetry**: `IntegrationEventDispatcher` uses `IIntegrationEvent.EventName`
and the type registry to deserialize incoming bytes and route to the correct
`IIntegrationEventHandler<T>`. A plain `IMessage` does not carry `EventName`,
so the infrastructure cannot identify its type from raw bytes on the consume side.

**Implication for domain events**: If a domain event needs to be consumed by another
service's handler, it MUST implement `IIntegrationEvent` (i.e., extend
`IntegrationEvent`). If it only needs to be published/delivered (e.g., for audit
or workflow trigger), plain `IMessage` suffices.

---

## R-008: Archive Strategy

**Decision**: Follow the existing `v7.x` versioned-subdirectory pattern.

**Rationale**: `content/core/eventbus/v7.x/_index.md` already exists, proving Hugo
renders versioned subdirs cleanly. The same pattern for `v8.5.0/` is consistent and
requires no theme changes.

**Archive path mapping**:
| Current page | Archive path |
|---|---|
| `content/core/eventbus/_index.md` | `content/core/eventbus/v8.5.0/_index.md` |
| `content/core/mediator/_index.md` | `content/core/mediator/v8.5.0/_index.md` |
| `content/core/integration/_index.md` | `content/core/integration/v8.5.0/_index.md` |

**Navigation exclusion**: Archive pages set `chapter = false` and high `weight` values
(e.g., 999) in front matter so they appear last/optionally in the nav. A Hugo partial
or `draft: false` + explicit navigation list (if the theme uses `menu`) keeps them
off the primary sidebar while remaining reachable by direct URL and search.

---

## R-009: NuGet Package Reference Table

| Component | NuGet Package |
|-----------|--------------|
| Core messaging + `MessagingBuilder` | `Juice.Messaging` |
| Outbox EF repository | `Juice.Messaging.Outbox.EF` |
| Delivery hosted service | `Juice.Messaging.Outbox.Delivery` |
| MediatR behaviors | `Juice.MediatR.Behaviors` |
| MediatR contracts (`IIdempotentRequest`) | `Juice.MediatR.Contracts` |
| EventBus dispatcher + subscriptions | `Juice.EventBus` |
| RabbitMQ transport | `Juice.EventBus.RabbitMQ` |
| Idempotency (in-memory + distributed cache) | `Juice.Messaging.Idempotency.Caching` |
| Idempotency (Redis) | `Juice.Messaging.Idempotency.Redis` |
| Idempotency (EF) | `Juice.Messaging.Idempotency.EF` |

---

## R-010: Existing Content to Archive

**Pages confirmed for verbatim archiving**:
- `content/core/eventbus/_index.md` — EventBus + IntegrationEventLog + RabbitMQ broker docs
- `content/core/mediator/_index.md` — MediatR IIdempotentRequest, IRequestManager docs
- `content/core/integration/_index.md` — IntegrationEventService + TransactionBehavior docs

**v8.5.0 interface-to-v9 mapping** (for migration table):
| v8.5.0 Concept | v9 Replacement |
|---|---|
| `IEventBus` | `IOutboxService<T>` + `IEventBus` (publish façade via `CompositeEventPublisher`) |
| `IntegrationEventLog` table | `OutboxEvent` + `OutboxDelivery` tables |
| `IIntegrationEventService` | `IOutboxService<T>` |
| `IIntegrationEventService.AddAndSaveEventAsync()` | `IOutboxService.AddEventAsync()` + `SaveEventsAsync()` |
| `IIntegrationEventService.PublishEventsThroughEventBusAsync()` | Handled automatically by `DeliveryHostedService` |
| `IRequestManager` (v8 MediatR dedup) | `IIdempotentRequest` + `IdempotencyBehavior` |
| `IIdentifiedCommand<T>` wrapper pattern | Direct `IIdempotentRequest` on command |
| `services.RegisterRabbitMQEventBus()` | `builder.AddMessaging().AddDelivery(d => d.EventBus.AddRabbitMQ(...))` |
| Direct `eventBus.PublishAsync()` from handler | `IOutboxService.AddEventAsync()` staged in transaction |
