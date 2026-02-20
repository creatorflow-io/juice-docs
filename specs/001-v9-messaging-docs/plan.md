# Implementation Plan: v9 Messaging Documentation Update

**Branch**: `001-v9-messaging-docs` | **Date**: 2026-02-18 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-v9-messaging-docs/spec.md`

## Summary

Replace the existing EventBus, Integration service, and MediatR documentation with six
new setup-scenario guides reflecting the Juice v9 unified messaging architecture
(`IMessage` hierarchy, `MessagingBuilder` + sub-builders, Outbox pattern, idempotency).
Archive all three current pages verbatim as v8.5.0. Provide a migration comparison table
mapping every v8.5.0 public interface to its v9 equivalent.

## Technical Context

**Language/Version**: Hugo (static site) for documentation; C# .NET 9 as the target
codebase being documented.
**Primary Dependencies**: Hugo site generator (existing); `Juice.Messaging`,
`Juice.EventBus`, `Juice.EventBus.RabbitMQ`, `Juice.Messaging.Outbox.EF`,
`Juice.Messaging.Outbox.Delivery`, `Juice.Messaging.Idempotency.*` as the packages
documented.
**Storage**: N/A — documentation-only content changes; no database involved.
**Testing**: Manual Hugo build validation (`hugo server`) and link-checking; ensure
no broken cross-references and all new pages return HTTP 200.
**Target Platform**: Hugo static site hosted at juice-docs.creatorflow.io.
**Project Type**: Web content — Hugo Markdown files with front matter.
**Performance Goals**: N/A for documentation.
**Constraints**: All pages MUST use Hugo front matter (title, date, weight, draft).
Code examples MUST use `hl_lines`. NuGet package links MUST be included per page.
Each setup guide MUST be independently readable without requiring other guides.
**Scale/Scope**: 6 new `_index.md` content pages + 3 archive `_index.md` pages +
1 migration table. 3 existing pages updated with v9 banner + archive links.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Check | Status |
|-----------|-------|--------|
| I. Simplicity & Lightweight | Each setup guide covers exactly one scenario; no cross-dependencies between guides | ✅ Pass |
| II. Domain-Driven Design | Docs reflect DDD patterns: domain events publishable directly, commands via `IIdempotentRequest`, aggregate roots produce domain events | ✅ Pass |
| III. Deployment Flexibility | Four independent setup guides (internal-only → outbox → delivery → consumption) allow incremental adoption; US5 covers full pipeline | ✅ Pass |
| IV. Integration-Ready | Contracts in `*.Api.Contracts` section referenced; `IIntegrationEvent` consume-side contract documented; gRPC/REST coexistence noted | ✅ Pass |
| V. Modular Architecture | `MessagingBuilder` sub-builder pattern mirrors modular feature registration | ✅ Pass |
| VI. Multi-Tenancy | `TenantId` on `IMessage`, `x-tenant-id` header, and `MessageContext` propagation documented in Consumption guide | ✅ Pass |
| VII. Observability | Both health checks (`AddRabbitMQHealthCheck`, `AddOutboxDeliveryHealthCheck`) documented in Delivery and Consumption guides; `MessageContext` fields documented | ✅ Pass |
| Documentation Standards | Hugo front matter, `hl_lines`, NuGet links, `breaking.md`/`whatsnew.md` per component pattern | ✅ Pass |

**Gate result: PASS — no violations. Proceed to Phase 0.**

## Project Structure

### Documentation (this feature)

```text
specs/001-v9-messaging-docs/
├── plan.md              ← this file
├── research.md          ← Phase 0 output
├── data-model.md        ← Phase 1 output
├── quickstart.md        ← Phase 1 output (contributor guide)
├── contracts/           ← Phase 1 output
│   ├── page-schema.md
│   └── migration-table.md
└── tasks.md             ← Phase 2 output (/speckit.tasks)
```

### Source Code (Hugo content)

```text
content/core/
├── _index.md                          (existing — add "Messaging" link)
│
├── messaging/                         (NEW section)
│   ├── _index.md                      (US1+overview: IMessage hierarchy + internal setup)
│   ├── outbox/
│   │   └── _index.md                  (US2: Outbox setup guide)
│   ├── delivery/
│   │   └── _index.md                  (US3: Delivery setup guide)
│   ├── consumption/
│   │   └── _index.md                  (US4: Consumption setup guide)
│   └── full-setup/
│       └── _index.md                  (US5: Full pipeline setup guide)
│
├── eventbus/
│   ├── _index.md                      (UPDATED: add v9 banner + archive link)
│   ├── v8.5.0/
│   │   └── _index.md                  (ARCHIVE: verbatim copy of current eventbus page)
│   └── v7.x/
│       └── _index.md                  (existing — unchanged)
│
├── mediator/
│   ├── _index.md                      (UPDATED: add v9 banner + archive link)
│   └── v8.5.0/
│       └── _index.md                  (ARCHIVE: verbatim copy of current mediator page)
│
└── integration/
    ├── _index.md                      (UPDATED: add v9 banner + archive link)
    └── v8.5.0/
        └── _index.md                  (ARCHIVE: verbatim copy of current integration page)
```

**Structure Decision**: Hugo content-only project. No backend or frontend src directories.
All changes are under `content/core/`. A new `messaging/` subsection groups all six
scenario guides. Archive pages follow the existing `v7.x` versioned-subdirectory
pattern already established under `content/core/eventbus/`.

## Complexity Tracking

> No constitution violations requiring justification.
