# Content Contract: IMessageService Table

**Location**: `content/core/messaging/local-transport/_index.md`
**Section**: `## IMessageService — Unified Publishing Interface`
**Change type**: Replace existing table

## Current (to be replaced)

```markdown
| Interface | Routes supported | When to use |
|---|---|---|
| `IMessageService` | `"local-channel"` only | Fire-and-forget, no DB |
| `IMessageService<TContext>` | All three (`"local-channel"`, `"local"`, broker) | When any route may require outbox writes |
```

## Required (replacement)

```markdown
| Interface | Registration | Routes supported | When to use |
|---|---|---|---|
| `IMessageService` | `AddLocalChannel()` only | `"local-channel"` | Fire-and-forget, zero DB, no retry needed |
| `IMessageService` | `AddDefaultMessageService()` | All three | Application services, controllers, background jobs — outside domain transactions |
| `IMessageService<TContext>` | `AddMessageService<TContext>()` | All three | Inside `TransactionBehavior` — defers outbox save until after commit |
```

## Acceptance criteria

- [ ] Table has three data rows (not two)
- [ ] `IMessageService` appears twice — once for each registration path
- [ ] No cell retains the text "local-channel only"
- [ ] `IMessageService<TContext>` row accurately describes deferred save behavior
