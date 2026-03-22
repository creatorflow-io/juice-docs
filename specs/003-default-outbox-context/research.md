# Research: DefaultOutboxContext Documentation

**Branch**: `003-default-outbox-context` | **Date**: 2026-03-19

All decisions are derived from the implemented feature in `Juice` release/9.0 branch. No external research required — ground truth is the source code and existing tests.

---

## Decision 1: Single page vs new page for DefaultOutboxContext content

**Decision**: Update existing `content/core/messaging/local-transport/_index.md` — no new page.

**Rationale**: The existing page already covers `IMessageService` in full context. Splitting to a new page would require readers to navigate between two pages to understand the complete `IMessageService` surface. The total content addition (≈80–100 lines) fits comfortably within the existing page structure.

**Alternatives considered**:
- `content/core/messaging/local-transport/default-outbox-context.md` — rejected: creates navigation split for tightly coupled content
- New top-level `content/core/messaging/default-outbox-context/_index.md` — rejected: `DefaultOutboxContext` is a registration concern, not a messaging concept; it belongs under local-transport

---

## Decision 2: `IMessageService` table update strategy

**Decision**: Replace the existing two-row table with a three-row table:

| Interface | Registration | Routes | When to use |
|---|---|---|---|
| `IMessageService` | `AddLocalChannel()` only | `"local-channel"` only | Fire-and-forget, zero DB, no retry |
| `IMessageService` | `AddDefaultMessageService()` | All three | Application services, controllers, background jobs outside transactions |
| `IMessageService<TContext>` | `AddMessageService<TContext>()` | All three | Inside `TransactionBehavior` — defers outbox save until commit |

**Rationale**: The old two-row table is accurate for `AddLocalChannel()` only setups. With `AddDefaultMessageService()`, `IMessageService` is no longer limited to local-channel. Both registration patterns remain valid and should coexist in the table.

**Alternatives considered**:
- Add a note beneath the existing table — rejected: the misconception is in the table cell itself; a note is too easy to miss
- Two separate tables — rejected: overly complex for a two-row addition

---

## Decision 3: `DefaultOutboxContext` section placement

**Decision**: Add `## DefaultOutboxContext` immediately after the `## DI Registration` section.

**Rationale**: Developers reading top-to-bottom encounter "what is it?" right after "how to register it". This matches the existing page's teach-as-you-go structure (registration → concept → behavior).

---

## Decision 4: Delivery note placement

**Decision**: Add a callout block inside the existing `## DI Registration` subsection for `AddDefaultMessageService()`, not in the delivery page.

**Rationale**: The delivery page covers `AddDelivery()` generically. The specific note "DefaultOutboxContext delivery is NOT auto-configured" belongs next to the registration snippet where developers make the decision, not on a separate delivery page they may not read until later.

---

## Decision 5: NuGet package to add

**Decision**: Add `Juice.Messaging.Outbox.EF` to the NuGet packages table with purpose: "`DefaultOutboxContext`, `AddDefaultMessageService()`, EF-backed outbox repository".

**Rationale**: `AddDefaultMessageService()` is defined in `Juice.Messaging.Outbox.EF`. The existing table omits this package because the prior doc predates the DefaultOutboxContext feature.

---

## Confirmed Facts from Source Code

| Fact | Source |
|---|---|
| `DefaultOutboxContext : DbContext, IOutboxContext` | `core/src/Juice.Messaging.Outbox.EF/DefaultOutboxContext.cs` |
| `IsManaged` always false | `IOutboxContext` impl — `DefaultOutboxContext` never sets it true |
| Same schema as `OutboxContext` via `ConfigureOutbox(modelBuilder)` | `DefaultOutboxContext.OnModelCreating` |
| `AddDefaultMessageService()` in `Juice.Messaging.Outbox.EF` | `OutboxMessagingBuilderExtensions.cs` |
| `IMessageService` registered via `TryAddScoped` (first-wins) | `OutboxMessagingBuilderExtensions.cs` line `TryAddScoped<IMessageService>(...)` |
| Delivery not auto-registered | No `AddDelivery()` call in `AddDefaultMessageService()` |
| `MessageService<TContext>` no longer inherits base `MessageService` | `MessageServiceT.cs` (refactored) |
