# Feature Specification: DefaultOutboxContext — Full-Route IMessageService Outside Transactions

**Feature Branch**: `003-default-outbox-context`
**Created**: 2026-03-19
**Status**: Draft
**Input**: Add documentation for `DefaultOutboxContext` and `AddDefaultMessageService()` — the mechanism that gives `IMessageService` full-route publishing capability (local-channel, local, broker) for code that runs outside domain transactions.

## User Scenarios & Testing *(mandatory)*

### User Story 1 — `IMessageService` Full-Route Registration via `AddDefaultMessageService()` (Priority: P1)

A developer building application services, background jobs, or API controllers needs to publish messages from code that runs **outside** a `TransactionBehavior` pipeline. They previously had to inject `IMessageService<TContext>` and wire up a domain `DbContext` just to get local/broker route support. They want a simpler path: call `AddDefaultMessageService()` and inject `IMessageService` as normal.

**Why this priority**: This is the primary registration API introduced by this feature. Every other topic (DefaultOutboxContext, delivery, coexistence) depends on understanding this entry point first.

**Independent Test**: Navigate to the updated Local Transport page or the new DefaultOutboxContext section and verify: (a) the `AddDefaultMessageService(opts => opts.UseSqlServer(...))` snippet compiles in context, (b) the route capability table shows `IMessageService` now supporting all three routes, (c) the "no DB" local-channel path is still documented for the case where `DefaultOutboxContext` is not needed.

**Acceptance Scenarios**:

1. **Given** a developer has only `"local-channel"` events, **When** they read the registration section, **Then** the doc makes clear they do not need `AddDefaultMessageService()` — `AddLocalChannel()` alone is sufficient.
2. **Given** a developer needs `"local"` or broker routes outside a transaction, **When** they read the registration section, **Then** they find a complete `AddDefaultMessageService(opts => opts.UseSqlServer(connStr))` snippet that replaces any prior `AddMessageService()` call.
3. **Given** a developer injects `IMessageService`, **When** they read the DI registration table, **Then** they understand that `AddDefaultMessageService()` registers `IMessageService` backed by `MessageService<DefaultOutboxContext>` — giving it the same full-route capability as `IMessageService<TContext>`.
4. **Given** a developer calls both `AddMessageService()` (old) and `AddDefaultMessageService()` (new), **When** they read the warnings box, **Then** they see a clear misconfiguration warning: `IMessageService` is registered with `TryAddScoped`; the first call wins.

---

### User Story 2 — `DefaultOutboxContext` Concept and Schema (Priority: P1)

A developer wants to understand what `DefaultOutboxContext` is, why it exists, and what database schema it requires. They need to know whether they need separate migrations or if they can reuse an existing outbox table.

**Why this priority**: Without this, developers cannot answer "do I need EF migrations?" or "will this collide with my existing `OutboxContext`?". Misconceptions here cause broken deployments.

**Independent Test**: The page must answer three questions without ambiguity: (1) What tables does `DefaultOutboxContext` use? (2) What migration project creates them? (3) Can `DefaultOutboxContext` and `OutboxContext` share the same database?

**Acceptance Scenarios**:

1. **Given** a developer asks "do I need new EF migrations?", **When** they read the schema section, **Then** they learn: no new migration project — `DefaultOutboxContext` uses the same schema as `OutboxContext`; run existing `OutboxContext` migrations against the target database.
2. **Given** a developer has `OutboxContext` registered for broker delivery and now adds `DefaultOutboxContext`, **When** they read the isolation note, **Then** they understand the two contexts are separate EF registrations with separate connection strings; no DI collision occurs.
3. **Given** a developer wonders about `IsManaged`, **When** they read the transaction behavior section, **Then** they understand: `DefaultOutboxContext.IsManaged` is always `false`, so `SaveEventsAsync` is called immediately as a standalone operation — never deferred by `TransactionBehavior`.

---

### User Story 3 — Coexistence with Domain-Aware `IMessageService<TContext>` (Priority: P2)

A developer has both a domain `DbContext` (for `TransactionBehavior`-aware publishing via `IMessageService<OrderDbContext>`) and application-level publishing via `IMessageService`. They want to confirm both can be registered simultaneously without conflict and understand which service to inject in each scenario.

**Why this priority**: Most production apps will have both. Without this, developers will either over-engineer (using `IMessageService<TContext>` everywhere) or under-engineer (missing the durable path for application services).

**Independent Test**: The coexistence section must include a DI registration snippet registering both, and a usage table explaining injection context for each.

**Acceptance Scenarios**:

1. **Given** a developer registers both `AddDefaultMessageService()` and `AddMessageService<OrderDbContext>()`, **When** they read the coexistence section, **Then** they find a working DI snippet and a note that `IMessageService` (default) and `IMessageService<OrderDbContext>` (domain-aware) resolve to different instances.
2. **Given** a developer is inside a command handler (inside `TransactionBehavior`), **When** they read the when-to-use table, **Then** they understand to inject `IMessageService<TContext>` (defers `SaveEventsAsync` until commit).
3. **Given** a developer is in an API controller or background service, **When** they read the when-to-use table, **Then** they understand to inject `IMessageService` backed by `DefaultOutboxContext` (saves immediately, no transaction deferral needed).

---

### User Story 4 — Delivery Configuration for `DefaultOutboxContext` (Priority: P2)

A developer who registers `AddDefaultMessageService()` also needs durable `"local"` or broker delivery retry. They need to know how to configure `DeliveryHostedService<DefaultOutboxContext>` and understand that it is NOT auto-configured by `AddDefaultMessageService()`.

**Why this priority**: Delivery is intentionally manual. Without documentation, developers will assume delivery is auto-wired and ship without a working retry loop for durable routes.

**Independent Test**: The delivery section must include a complete snippet using `AddDelivery()` + `AddDeliveryProcessor<DefaultOutboxContext>()` and must state clearly that `AddDefaultMessageService()` does not auto-register delivery.

**Acceptance Scenarios**:

1. **Given** a developer registers `AddDefaultMessageService()` and uses the `"local"` route, **When** they read the delivery section, **Then** they find a note: delivery is NOT auto-configured — call `AddDelivery()` + `AddDeliveryProcessor<DefaultOutboxContext>()` explicitly.
2. **Given** a developer wants broker delivery via `DefaultOutboxContext`, **When** they read the delivery snippet, **Then** they find `builder.AddDeliveryProcessor<DefaultOutboxContext>("rabbitmq")` as the registration call.
3. **Given** a developer only uses the `"local-channel"` route, **When** they read the delivery section, **Then** they see a note confirming delivery configuration is not needed for `"local-channel"` (in-memory, no retry).

---

## Functional Requirements *(mandatory)*

### FR-1: Update `IMessageService` Interface Table
The existing Local Transport page's "IMessageService — Unified Publishing Interface" table must be updated:
- Remove the row showing `IMessageService` supports only `"local-channel"`
- Replace with: `IMessageService` (via `AddDefaultMessageService()`) supports all three routes

### FR-2: Update DI Registration Section
The existing "DI Registration" section must be extended with a new subsection:
- **"Full-route outside transactions (DefaultOutboxContext)"** showing `AddDefaultMessageService(opts => opts.UseSqlServer(connStr))`
- Include a warning callout: first `IMessageService` registration wins (`TryAddScoped`)
- Keep the existing "local-channel only (zero DB)" subsection unchanged

### FR-3: New `DefaultOutboxContext` Section or Page
Document `DefaultOutboxContext`:
- What it is: standalone `DbContext, IOutboxContext` with its own connection string
- Schema: same tables (`OutboxEvents`, `OutboxDeliveries`) as `OutboxContext`; no separate migration project needed
- `IsManaged = false`: always executes `SaveEventsAsync` immediately
- Isolation from `OutboxContext`: separate EF registration, no DI collision, can target same or different DB

### FR-4: Coexistence Section
Add a "Coexistence with `IMessageService<TContext>`" subsection:
- DI snippet registering both services
- When-to-use table: command handler (domain-aware, `IMessageService<TContext>`) vs. controller/background service (default outbox, `IMessageService`)

### FR-5: Delivery Section Update
Update the delivery documentation to clarify:
- `AddDefaultMessageService()` does NOT auto-register `DeliveryHostedService<DefaultOutboxContext>`
- Explicit configuration: `AddDelivery(b => b.AddDeliveryProcessor<DefaultOutboxContext>(publisherKey))`
- Note that `"local-channel"` route requires no delivery configuration

### FR-6: Remove Outdated Constraint
Remove or correct any documentation that states `IMessageService` supports only `"local-channel"`. This was true before this feature; it is no longer accurate.

---

## Success Criteria *(mandatory)*

1. A developer can follow the docs to register `AddDefaultMessageService()`, inject `IMessageService`, and publish to `"local"` or broker routes — without referencing source code.
2. A developer can determine whether to use `IMessageService` or `IMessageService<TContext>` based solely on the when-to-use table.
3. A developer with an existing `OutboxContext` setup can add `DefaultOutboxContext` without confusion about migration requirements or DI collisions.
4. The delivery section leaves no ambiguity: delivery is always a manual configuration step after `AddDefaultMessageService()`.
5. No page in the docs retains the outdated claim that `IMessageService` supports only `"local-channel"`.

---

## Scope *(mandatory)*

**In scope**:
- Update `content/core/messaging/local-transport/_index.md` — `IMessageService` table, DI registration section, new DefaultOutboxContext subsection, coexistence subsection, delivery clarification
- Optionally: new standalone page `content/core/messaging/local-transport/default-outbox-context.md` if content is too large for the existing page

**Out of scope**:
- EF migration guide (migrations belong to `OutboxContext` — no new migration project)
- Broker-specific delivery docs (covered by existing Delivery page)
- Changes to MediatR, MessageContext, or Outbox pages

---

## Key Entities *(optional)*

| Term | Definition |
|---|---|
| `DefaultOutboxContext` | Standalone `DbContext, IOutboxContext` registered by `AddDefaultMessageService()`; `IsManaged` always false |
| `AddDefaultMessageService(configure)` | `MessagingBuilder` extension in `Juice.Messaging.Outbox.EF`; backs `IMessageService` with `MessageService<DefaultOutboxContext>` |
| `IMessageService` | Non-generic publishing interface; previously local-channel only; now full-route when backed by `DefaultOutboxContext` |
| `IMessageService<TContext>` | Domain-aware generic; defers `SaveEventsAsync` inside `TransactionBehavior` |
| `IsManaged` | Property on `IOutboxContext`; when `false`, `SaveEventsAsync` runs immediately; `DefaultOutboxContext` always returns false |

---

## Dependencies & Assumptions *(optional)*

- `Juice.Messaging.Outbox.EF` NuGet package provides both `DefaultOutboxContext` and `AddDefaultMessageService()`
- `OutboxContext` migrations must already exist and target the same database; no new migration project is required
- Delivery is manual by design — developers must explicitly call `AddDelivery()` + `AddDeliveryProcessor<DefaultOutboxContext>()`
- The docs site uses Hugo with Mermaid support for diagrams

---

## Clarifications

### Session 2026-03-19

- Q: Are EF migrations needed for `DefaultOutboxContext`? → A: No — `DefaultOutboxContext` reuses the `OutboxContext` schema. Run existing `OutboxContext` migrations; no separate migration project.
- Q: Is delivery auto-configured by `AddDefaultMessageService()`? → A: No — delivery is always configured manually via `AddDelivery()` + `AddDeliveryProcessor<DefaultOutboxContext>()`.
