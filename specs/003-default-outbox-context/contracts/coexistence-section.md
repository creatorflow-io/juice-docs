# Content Contract: Coexistence Section

**Location**: `content/core/messaging/local-transport/_index.md`
**Section**: `## Coexistence — IMessageService and IMessageService<TContext>` (new section)
**Change type**: Insert new `##` section after `## DefaultOutboxContext`

---

## Required Content

```markdown
## Coexistence — `IMessageService` and `IMessageService<TContext>`

Both registrations can coexist in the same application without conflict:

```csharp {linenos=false,linenostart=1}
services.AddMessaging(builder =>
{
    // Domain-aware: used inside TransactionBehavior
    builder.AddMessageService<AppDbContext>();

    // Default outbox: used outside transactions
    builder.AddDefaultMessageService(opts =>
        opts.UseSqlServer(configuration.GetConnectionString("Outbox")));
});

// DI coexistence — both resolve independently:
services.AddScoped<IOutboxService<AppDbContext>, AppOutboxService>();
```

### When to inject which

| Injection context | Interface to inject | Why |
|---|---|---|
| Command handler inside `TransactionBehavior` | `IMessageService<TContext>` | Defers `SaveEventsAsync` until after domain transaction commits |
| API controller | `IMessageService` | No transaction context; saves immediately via `DefaultOutboxContext` |
| Background service / `IHostedService` | `IMessageService` | No transaction context; saves immediately |
| Application service (non-transactional) | `IMessageService` | Simpler; no domain `DbContext` dependency |
| Domain event handler inside `TransactionBehavior` | `IMessageService<TContext>` | Must participate in the same transaction scope |

> **`IMessageService` is registered with `TryAddScoped`.** The first call wins. If both
> `AddMessageService<TContext>()` and `AddDefaultMessageService()` are called, only the
> first `IMessageService` registration is active. `IMessageService<TContext>` (the generic
> form) is **always registered independently** — it does not conflict with `IMessageService`.
```

---

## Acceptance criteria

- [ ] Section heading contains "Coexistence"
- [ ] DI snippet shows both `AddMessageService<AppDbContext>()` and `AddDefaultMessageService()`
- [ ] When-to-use table has at least 5 rows covering command handler, controller, background service, application service, domain event handler
- [ ] `TryAddScoped` first-wins note present
- [ ] Table clearly explains `IMessageService<TContext>` deferral vs `IMessageService` immediate save
