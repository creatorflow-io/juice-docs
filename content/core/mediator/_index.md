---
title: "Mediator"
date: 2023-04-03T20:23:50+07:00
draft: false
weight: 7
---

MediatR provides three in-process messaging patterns. In Juice v9 all three can extend
`MessageBase` so every message carries `MessageId`, `CreatedAt`, and `TenantId` automatically.

| Pattern | Interface | Handler interface | Use when |
|---|---|---|---|
| **Request/Response** | `IRequest<TResponse>` | `IRequestHandler<T, TResponse>` | Command or query — exactly one handler, returns a result |
| **Notification** | `INotification` | `INotificationHandler<T>` | In-process fan-out — multiple handlers, no return value |
| **Stream Request** | `IStreamRequest<TResponse>` | `IStreamRequestHandler<T, TResponse>` | Async streaming — handler yields items one at a time |

---

## NuGet Package

| Package | Purpose |
|---|---|
| [MediatR](https://www.nuget.org/packages/MediatR) | Core MediatR library |
| [Juice.MediatR.Behaviors](https://www.nuget.org/packages/Juice.MediatR.Behaviors) | `IdempotencyRequestBehavior<,>`, `TransactionBehavior<,,>` |
| [Juice.MediatR.Contracts](https://www.nuget.org/packages/Juice.MediatR.Contracts) | `IIdempotentRequest` interface |

---

## Request / Response — Commands and Queries

A request routes to **exactly one handler** and always returns a result. Use it for commands
(state-changing operations) and queries (read-only operations).

**Define the message:**

```csharp {linenos=false,hl_lines=[2,7],linenostart=1}
// Command — modifies state, idempotent (safe to retry)
public record CreateOrderCommand(string OrderId, string CustomerId)
    : MessageBase, IRequest<IOperationResult>, IIdempotentRequest
{
    public string IdempotencyKey => OrderId;
}

// Query — reads state, no side effects
public record GetOrderQuery(string OrderId) : MessageBase, IRequest<OrderDto?>;
```

**Implement the handler:**

```csharp {linenos=false,hl_lines=[1,5,8],linenostart=1}
public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, IOperationResult>
{
    private readonly AppDbContext _db;

    public CreateOrderCommandHandler(AppDbContext db) => _db = db;

    public async Task<IOperationResult> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        _db.Orders.Add(new Order(cmd.OrderId, cmd.CustomerId));
        await _db.SaveChangesAsync(ct);
        return OperationResult.Success();
    }
}
```

**Dispatch:**

```csharp {linenos=false,hl_lines=[1],linenostart=1}
var result = await mediator.Send(new CreateOrderCommand(orderId, customerId));
```

---

## Notification — In-Process Events

A notification fans out to **zero or more handlers** and returns nothing. Use it to
broadcast a domain event to multiple in-process listeners without coupling them to the sender.

**Define the message:**

```csharp {linenos=false,hl_lines=[1],linenostart=1}
public record OrderStatusChangedNotification(string OrderId, string NewStatus)
    : MessageBase, INotification;
```

**Implement handlers (as many as needed):**

```csharp {linenos=false,hl_lines=[1,10],linenostart=1}
public class SendStatusEmailHandler : INotificationHandler<OrderStatusChangedNotification>
{
    public async Task Handle(OrderStatusChangedNotification n, CancellationToken ct)
    {
        // send email ...
    }
}

public class WriteAuditLogHandler : INotificationHandler<OrderStatusChangedNotification>
{
    public async Task Handle(OrderStatusChangedNotification n, CancellationToken ct)
    {
        // write audit entry ...
    }
}
```

**Dispatch:**

```csharp {linenos=false,hl_lines=[1],linenostart=1}
await mediator.Publish(new OrderStatusChangedNotification(orderId, "Shipped"));
```

> **Sequential execution**: Handlers run one after another in registration order by default.
> An exception in one handler stops the remaining handlers. For fire-and-forget or parallel
> dispatch, implement a custom `INotificationPublisher` and register it with MediatR.

---

## Stream Request — Async Streaming

A stream request returns `IAsyncEnumerable<TResponse>`. The handler yields items one at a
time and the caller consumes them as they arrive — ideal for large result sets or
server-sent event feeds.

**Define the message:**

```csharp {linenos=false,hl_lines=[1],linenostart=1}
public record GetOrderEventsQuery(string OrderId) : MessageBase, IStreamRequest<OrderEventDto>;
```

**Implement the handler:**

```csharp {linenos=false,hl_lines=[1,2,8,9],linenostart=1}
public class GetOrderEventsHandler
    : IStreamRequestHandler<GetOrderEventsQuery, OrderEventDto>
{
    private readonly AppDbContext _db;

    public GetOrderEventsHandler(AppDbContext db) => _db = db;

    public async IAsyncEnumerable<OrderEventDto> Handle(
        GetOrderEventsQuery query,
        [EnumeratorCancellation] CancellationToken ct)
    {
        await foreach (var evt in _db.OrderEvents
            .Where(e => e.OrderId == query.OrderId)
            .AsAsyncEnumerable()
            .WithCancellation(ct))
        {
            yield return new OrderEventDto(evt.Type, evt.OccurredAt);
        }
    }
}
```

**Consume the stream:**

```csharp {linenos=false,hl_lines=[1],linenostart=1}
await foreach (var evt in mediator.CreateStream(new GetOrderEventsQuery(orderId)))
{
    Console.WriteLine($"{evt.Type} at {evt.OccurredAt}");
}
```

---

## Registration

Register MediatR once, scanning all handler types from your assembly:

```csharp {linenos=false,hl_lines=[3,5,6],linenostart=1}
services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(MyHandler).Assembly);

    cfg.AddIdempotencyRequestBehavior();                    // deduplication — requires IIdempotentRequest
    cfg.AddOpenBehavior(typeof(AppTransactionBehavior<,>)); // transaction — requires Outbox setup
});
```

`RegisterServicesFromAssembly` discovers and registers all three handler types automatically:

| Handler interface | Discovered automatically | DI lifetime |
|---|---|---|
| `IRequestHandler<T, R>` | Yes | Transient |
| `INotificationHandler<T>` | Yes | Transient |
| `IStreamRequestHandler<T, R>` | Yes | Transient |

Handlers support full constructor injection. Scoped services such as `DbContext` and
repositories resolve correctly because MediatR resolves handlers within the active DI scope.

---

## See also

- [Messaging]({{< ref "core/messaging/_index.md" >}}) — `IMessage` hierarchy, idempotency backends, and `MessagingBuilder` setup
- [Outbox Setup]({{< ref "core/messaging/outbox/_index.md" >}}) — transactional staging with `TransactionBehavior`
- [MediatR v8.5.0 archive]({{< ref "core/mediator/v8.5.0/_index.md" >}}) — original `IRequestManager` / `IdentifiedCommand` pattern
