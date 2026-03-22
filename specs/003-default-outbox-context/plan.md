# Implementation Plan: DefaultOutboxContext — Full-Route IMessageService Outside Transactions

**Branch**: `003-default-outbox-context` | **Date**: 2026-03-19 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/003-default-outbox-context/spec.md`

## Summary

Update the Local Transport documentation to reflect the `DefaultOutboxContext` feature: `IMessageService` is no longer limited to `"local-channel"` — when registered via `AddDefaultMessageService()`, it is backed by `MessageService<DefaultOutboxContext>` and supports all three routes (`"local-channel"`, `"local"`, broker). The primary page to update is `content/core/messaging/local-transport/_index.md`. A new subsection documents `DefaultOutboxContext` (schema, `IsManaged = false`, no separate migrations) and a coexistence table clarifies when to inject `IMessageService` vs `IMessageService<TContext>`.

## Technical Context

**Language/Version**: Hugo (Markdown + front matter) with Mermaid diagrams; C# .NET 9 code examples
**Primary Dependencies**: Hugo static site; existing Juice.Messaging.Outbox.EF v9 package
**Storage**: N/A (documentation only)
**Testing**: Manual: `hugo server` to verify rendering; link checker for cross-references
**Target Platform**: juice-docs Hugo site (`content/core/messaging/local-transport/`)
**Project Type**: Single-section documentation update (one primary file, no new pages required)
**Performance Goals**: N/A
**Constraints**: Hugo front matter must be valid; code examples must be complete and compilable; no broken `{{< ref >}}` links
**Scale/Scope**: 1 primary file updated (`local-transport/_index.md`); 0 new pages required (content fits within existing page)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Simplicity | ✅ PASS | Single file update; no new pages |
| II. DDD | ✅ PASS | Docs reflect DDD outbox pattern correctly |
| III. Deployment Flexibility | ✅ PASS | Documents both local-channel (no DB) and DefaultOutboxContext (with DB) paths |
| IV. Integration-Ready | ✅ PASS | NuGet package table will include `Juice.Messaging.Outbox.EF` |
| V. Modular Architecture | ✅ PASS | Coexistence section explicitly documents `IMessageService` and `IMessageService<TContext>` independence |
| VI. Multi-Tenancy | ✅ PASS | No tenant-specific changes; existing tenant propagation docs unchanged |
| VII. Observability | ✅ PASS | Delivery section notes manual delivery configuration requirement |
| Documentation Standards | ✅ PASS | Hugo front matter preserved; complete code examples; cross-references via `{{< ref >}}` |

No violations. **PROCEED**.

## Project Structure

### Documentation (this feature)

```text
specs/003-default-outbox-context/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── contracts/           # Phase 1 output — content contracts per section
│   ├── imessageservice-table.md
│   ├── di-registration.md
│   ├── default-outbox-context-section.md
│   └── coexistence-section.md
└── tasks.md             # Phase 2 output (/speckit.tasks command)
```

### Source Files (to be modified)

```text
content/core/messaging/local-transport/
└── _index.md            # Primary target: update 3 sections, add 2 new sections
```

## Complexity Tracking

*No Constitution Check violations — table not required.*

## Implementation Strategy

**MVP (US1 + US2)**: Update `IMessageService` table + add `DefaultOutboxContext` section. Delivers the two highest-risk misconceptions: (1) "IMessageService only does local-channel" and (2) "I need separate migrations".

**Increment 2 (US3)**: Add coexistence section. Low risk, stands alone.

**Increment 3 (US4)**: Update delivery section with DefaultOutboxContext note. Small targeted addition.

### Edits to `local-transport/_index.md`

| Section | Change type | US |
|---------|-------------|-----|
| `IMessageService` table | Replace rows — add `AddDefaultMessageService()` path | US1 |
| NuGet packages table | Add `Juice.Messaging.Outbox.EF` row | US1 |
| DI Registration | Add subsection "Full-route outside transactions" | US1 |
| New: `DefaultOutboxContext` | New `##` section after DI Registration | US2 |
| New: Coexistence | New `##` section — when-to-use table | US3 |
| Route Behavior / Delivery note | Add callout: delivery is manual for `DefaultOutboxContext` | US4 |
