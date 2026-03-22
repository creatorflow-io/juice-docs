# Content Contract: DefaultOutboxContext Section

**Location**: `content/core/messaging/local-transport/_index.md`
**Section**: `## DefaultOutboxContext` (new section)
**Change type**: Insert new `##` section after `## DI Registration`

---

## Required Content

```markdown
## DefaultOutboxContext

`DefaultOutboxContext` is a standalone EF Core `DbContext` implementing `IOutboxContext`.
It owns its own database connection string and is completely independent of any domain
`DbContext` registered in the application.

### Schema

`DefaultOutboxContext` uses the **same tables** (`OutboxEvents` and `OutboxDeliveries`)
as `OutboxContext`. The schema is applied via the shared `ConfigureOutbox()` extension —
no separate migration project is needed.

> **To create the tables**: run the existing `OutboxContext` migrations against the target
> database. `DefaultOutboxContext` resolves the schema from those migrations automatically.

### Transaction behavior (`IsManaged = false`)

`DefaultOutboxContext.IsManaged` is always `false`. This means `SaveEventsAsync` is
**never deferred** — it executes immediately as a standalone operation, regardless of
whether a domain transaction is in progress.

This is intentional: `DefaultOutboxContext` is designed for code paths that are **outside**
a `TransactionBehavior` pipeline. For code inside `TransactionBehavior` — where you need
outbox saves to be deferred until the domain transaction commits — use
`IMessageService<TContext>` with your domain `DbContext` instead.

### Isolation from `OutboxContext`

If you also register `OutboxContext` (e.g., for broker delivery driven by `AppDbContext`),
the two contexts are registered as separate EF services with separate connection strings.
They do not collide in DI — one is `IOutboxContext` for `OutboxContext`, the other for
`DefaultOutboxContext`. Both can coexist and even target the same database if desired.
```

---

## Acceptance criteria

- [ ] Section heading is `## DefaultOutboxContext`
- [ ] Schema subsection explains same tables, no separate migrations, uses `ConfigureOutbox()`
- [ ] `IsManaged = false` explained with consequence (immediate `SaveEventsAsync`)
- [ ] Contrast with `IMessageService<TContext>` (deferred) is explicit
- [ ] Isolation note confirms no DI collision with `OutboxContext`
