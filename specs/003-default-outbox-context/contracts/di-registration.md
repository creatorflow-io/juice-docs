# Content Contract: DI Registration Section

**Location**: `content/core/messaging/local-transport/_index.md`
**Section**: `## DI Registration`
**Change type**: Add new subsection; update NuGet table

---

## NuGet Packages Table Update

Add one row to the existing packages table:

| Package | Purpose |
|---|---|
| `Juice.Messaging.Outbox.EF` | `DefaultOutboxContext`, `AddDefaultMessageService()`, EF-backed outbox repository |

---

## New Subsection: "Full-route outside transactions (DefaultOutboxContext)"

Insert after the existing "All routes (local-channel + local + broker)" subsection.

### Heading

```markdown
### Full-route outside transactions (`DefaultOutboxContext`)
```

### Content

```markdown
Use `AddDefaultMessageService()` when you need full-route publishing (`"local-channel"`,
`"local"`, and broker routes) from code that runs **outside** a `TransactionBehavior`
pipeline — for example, API controllers, background services, or application-layer services
that are not part of a domain transaction.

```csharp {linenos=false,hl_lines=[4],linenostart=1}
services.AddMessaging(builder =>
{
    // Registers DefaultOutboxContext + IMessageService (backed by MessageService<DefaultOutboxContext>)
    builder.AddDefaultMessageService(opts =>
        opts.UseSqlServer(configuration.GetConnectionString("Outbox")));

    builder.AddLocalChannel();          // required for "local-channel" route dispatch
    builder.AddIdempotencyRedis(opts =>
        opts.Configuration = configuration.GetConnectionString("Redis"));

    builder.AddPublishingPolicies(configuration.GetSection("PublishingPolicies"));
});
```

> **Delivery is not auto-configured.** If you use the `"local"` or broker routes, add the
> delivery worker explicitly:
>
> ```csharp
> builder.AddDelivery(delivery =>
> {
>     delivery.AddDeliveryProcessor<DefaultOutboxContext>("local",
>         "send-pending", "retry-failed", "recover-timeout");
> });
> ```

> **`IMessageService` registration uses `TryAddScoped` — first call wins.** Do not call
> both `AddMessageService<TContext>()` and `AddDefaultMessageService()` in the same
> container. Use `IMessageService<TContext>` if you need domain-transaction deferral inside
> `TransactionBehavior`; use `AddDefaultMessageService()` for everything else.
```

---

## Acceptance criteria

- [ ] New subsection exists with heading `### Full-route outside transactions (DefaultOutboxContext)`
- [ ] Code snippet shows `AddDefaultMessageService(opts => opts.UseSqlServer(...))`
- [ ] Delivery callout block present, showing `AddDeliveryProcessor<DefaultOutboxContext>`
- [ ] `TryAddScoped` first-wins warning present
- [ ] `Juice.Messaging.Outbox.EF` appears in NuGet table
