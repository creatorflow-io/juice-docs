# Tasks: v9 Messaging Documentation Update

**Input**: Design documents from `/specs/001-v9-messaging-docs/`
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/ ✅, quickstart.md ✅

**Tests**: No tests requested in spec.md — no test tasks included.

**Organization**: Tasks are grouped by user story to enable independent implementation and
testing of each story. Archive tasks are in Phase 2 (Foundational) because they must
precede any modification to existing pages (per `quickstart.md` Step 1 rationale).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no ordering dependency)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2)
- Include exact file paths in all descriptions

---

## Phase 1: Setup (Directory Structure)

**Purpose**: Create the Hugo content directory structure before writing any content files.

- [x] T001 Create Hugo subdirectories `content/core/messaging/outbox/`, `content/core/messaging/delivery/`, `content/core/messaging/consumption/`, `content/core/messaging/full-setup/`, `content/core/eventbus/v8.5.0/`, `content/core/mediator/v8.5.0/`, `content/core/integration/v8.5.0/`

---

## Phase 2: Foundational (Archive + Navigation)

**Purpose**: Archive the three existing pages verbatim before modifying them, then add
v9 banners to the live pages and update core navigation. These MUST complete before any
new content pages are written.

**⚠️ CRITICAL**: T002–T004 (archive) must complete before T005–T007 (banners). Banners
link to archive pages which must already exist.

- [x] T002 Archive EventBus — copy `content/core/eventbus/_index.md` to `content/core/eventbus/v8.5.0/_index.md`; change front matter title to `"Event bus (v8.5.0)"` and weight to `999`; add archive banner immediately after front matter per `contracts/page-schema.md` § Archive Banner Contract
- [x] T003 [P] Archive MediatR — copy `content/core/mediator/_index.md` to `content/core/mediator/v8.5.0/_index.md`; change title to `"MediatR (v8.5.0)"` and weight to `999`; add archive banner per `contracts/page-schema.md` § Archive Banner Contract
- [x] T004 [P] Archive Integration service — copy `content/core/integration/_index.md` to `content/core/integration/v8.5.0/_index.md`; change title to `"Integration service (v8.5.0)"` and weight to `999`; add archive banner per `contracts/page-schema.md` § Archive Banner Contract
- [x] T005 Add v9 update banner to `content/core/eventbus/_index.md` immediately after front matter, linking to `core/messaging/_index.md` (new API) and `core/eventbus/v8.5.0/_index.md` (archive) using Hugo `ref` shortcode — see `quickstart.md` § Step 2 for exact banner text
- [x] T006 [P] Add v9 update banner to `content/core/mediator/_index.md` immediately after front matter, linking to `core/messaging/_index.md` and `core/mediator/v8.5.0/_index.md`
- [x] T007 [P] Add v9 update banner to `content/core/integration/_index.md` immediately after front matter, linking to `core/messaging/_index.md` and `core/integration/v8.5.0/_index.md`
- [x] T008 Add Messaging section link at weight=6 to `content/core/_index.md` alongside the existing EventBus, MediatR, and Integration entries (per `data-model.md` § Navigation Weight Plan)

**Checkpoint**: All archive pages exist, all three live pages show v9 banners, core navigation includes Messaging. User story content pages can now be created.

---

## Phase 3: User Story 1 — Internal Messaging Setup (Priority: P1) 🎯 MVP

**Goal**: Create the Messaging section index at `content/core/messaging/_index.md`
with the IMessage hierarchy overview, setup guide index, and the complete Internal
Messaging Setup guide (MediatR + IdempotencyBehavior + idempotency backend selection).

**Independent Test**: Navigate to `/core/messaging/` and confirm: (1) IMessage ASCII
tree and publish-vs-consume role table are visible; (2) links to all four setup sub-guides
are present; (3) a complete DI registration block with `services.AddMessaging()` +
`AddIdempotencyRedis()`/`AddIdempotencyInMemory()` is shown; (4) no outbox or broker
code appears anywhere on the page. A developer can follow this page alone to produce
a working internal-only DI registration with `IIdempotentRequest` deduplication.

### Implementation

- [x] T009 [US1] Create `content/core/messaging/_index.md` with front matter (`chapter=true`, `weight=6`, `title="Messaging"`, `date="2026-02-19"`, `draft=false`); write `## IMessage Hierarchy` section containing the ASCII type tree (`IMessage → IEvent → IIntegrationEvent`; `MessageBase → IntegrationEvent`) and publish-vs-consume role table per `research.md` R-007; write `## Setup Guides` section with links and one-line descriptions for all four sub-guides (Outbox, Delivery, Consumption, Full Setup)
- [x] T010 [US1] Add `## Internal Messaging Setup` section to `content/core/messaging/_index.md` — "When to use" bullet list (internal MediatR commands only, no broker needed), "Prerequisites" (none), NuGet packages table (`Juice.Messaging`, `Juice.MediatR.Behaviors`, `Juice.MediatR.Contracts`, `Juice.Messaging.Idempotency.[Backend]`) with `https://www.nuget.org/packages/` links; add `### DI Registration` code block with `hl_lines` per `contracts/page-schema.md` § US1 showing `services.AddMessaging()` + `AddIdempotencyRedis()` + `services.AddMediatR()` with `IdempotencyRequestBehavior<,>`
- [x] T011 [US1] Add `### IIdempotentRequest` section to `content/core/messaging/_index.md` — interface definition (from `research.md` R-005: `string IdempotencyKey` property), `CreateOrderCommand` usage example marking command with `IIdempotentRequest`, v8.5.0 comparison table (`IIdentifiedCommand<T>` wrapper → direct `IIdempotentRequest` on command; `IRequestManager.ExistAsync()` → automatic via `IdempotencyBehavior`; no injection needed)
- [x] T012 [US1] Add `### IdempotencyBehavior Pipeline` and `### Idempotency Backend Selection` sections to `content/core/messaging/_index.md` — pipeline order note (`int.MinValue` priority, runs before all other behaviors), backend comparison table (InMemory/DistributedCache/Redis/EF with registration method, package, use case per `research.md` R-006), note that one `IIdempotencyService` registration serves both `IdempotencyBehavior` (MediatR side) and `IntegrationEventDispatcher` (consume side); add `## See also` links (Outbox Setup, v8.5.0 EventBus archive, v8.5.0 MediatR archive)

**Checkpoint**: `/core/messaging/` fully renders with hierarchy overview and internal setup complete. US1 independently testable.

---

## Phase 4: User Story 2 — Outbox Setup (Priority: P1)

**Goal**: Create `content/core/messaging/outbox/_index.md` covering `TransactionBehavior`,
`IOutboxService`, publishing policies, and staging any `IMessage` in a transaction —
including plain domain events without converting to `IntegrationEvent`.

**Independent Test**: Navigate to `/core/messaging/outbox/` and confirm: (1) complete
`services.AddMessaging()` block with `AddPublishingPolicies()` + `AddOutbox()` is shown;
(2) a code example stages a plain domain event via `IOutboxService.AddEventAsync()`;
(3) a `PublishingPolicies` JSON config example is present; (4) no broker or delivery
worker code appears. A developer can follow this page alone to produce a working
transactional outbox DI registration.

### Implementation

- [x] T013 [US2] Create `content/core/messaging/outbox/_index.md` with front matter (`weight=2`, `title="Outbox Setup"`, `date="2026-02-19"`, `draft=false`); write Overview paragraph (one-sentence summary of transactional outbox staging), "When to use" bullet list (commands that modify state + must publish messages atomically, replacing direct `IEventBus.PublishAsync()` calls), Prerequisites section (Internal Messaging Setup for `TransactionBehavior` DbContext), NuGet packages table (`Juice.Messaging`, `Juice.Messaging.Outbox.EF`, `Juice.MediatR.Behaviors` with links)
- [x] T014 [US2] Add `## DI Registration` section to `content/core/messaging/outbox/_index.md` — complete `services.AddMessaging()` block containing `AddPublishingPolicies(config.GetSection("PublishingPolicies"))`, `AddOutbox(outbox => { outbox.AddOutboxRepository(); outbox.AddDeliveryIntents(); })`, and `services.AddMediatR()` with `TransactionBehavior<,,>` open behavior; use `hl_lines` per `contracts/page-schema.md` § US2
- [x] T015 [US2] Add `## IOutboxService — Staging Events` and `## Publishing Any IMessage` sections to `content/core/messaging/outbox/_index.md` — explain `AddEventAsync(message)` + `SaveEventsAsync(transactionId)` flow; code example staging a plain domain event (`IMessage` subtype, not `IntegrationEvent`) via `IOutboxService`; callout box: "`IIntegrationEvent` is only required on the **consuming** side — the outbox accepts any `IMessage` including domain events"
- [x] T016 [US2] Add `## IMessagePublishingPolicy`, `## TransactionBehavior — Subclassing`, `## OutboxEvent + OutboxDelivery Tables`, and `## See also` sections to `content/core/messaging/outbox/_index.md` — `appsettings.json` `PublishingPolicies` config example; `AppTransactionBehavior<T,R>` concrete subclass code per `research.md` R-004; brief two-table model note (migrations required); "See also" links (Delivery Setup, Full Setup, v8.5.0 EventBus archive, v8.5.0 Integration archive)

**Checkpoint**: `/core/messaging/outbox/` fully renders. US2 independently testable.

---

## Phase 5: User Story 3 — Delivery Setup (Priority: P2)

**Goal**: Create `content/core/messaging/delivery/_index.md` covering `DeliveryProcessor`,
the three intent strategies, retry backoff schedule, `OutboxDelivery` state machine, and
`AddOutboxDeliveryHealthCheck`.

**Independent Test**: Navigate to `/core/messaging/delivery/` and confirm: (1) complete
`AddDelivery()` registration is shown; (2) intent strategies table lists send-pending,
retry-failed, recover-timeout with triggers; (3) default retry backoff table (10s→5m→10m→
30m→1h) is present with override example; (4) health check registration is shown. A
developer can follow this page alone to produce a working delivery DI block.

### Implementation

- [x] T017 [US3] Create `content/core/messaging/delivery/_index.md` with front matter (`weight=3`, `title="Delivery Setup"`, `date="2026-02-19"`, `draft=false`); write Overview paragraph (background delivery worker forwards staged outbox messages to the broker), "When to use" bullet list (service that has outbox configured and needs a delivery worker, or running delivery in a separate process/service), Prerequisites section (Outbox Setup), NuGet packages table (`Juice.Messaging.Outbox.Delivery`, `Juice.EventBus.RabbitMQ` with links)
- [x] T018 [US3] Add `## DI Registration` section to `content/core/messaging/delivery/_index.md` — complete `services.AddMessaging()` block with `AddDelivery(delivery => { delivery.AddDeliveryProcessor<AppDbContext>("rabbitmq", "send-pending", "retry-failed", "recover-timeout"); delivery.AddDeliveryPolicies(config.GetSection("DeliveryPolicies")); delivery.EventBus.AddRabbitMQ(...) })` with `AddConnection` + `AddProducer`; use `hl_lines` per `contracts/page-schema.md` § US3; include note that `AddDelivery()` automatically creates `EventBusBuilder` internally — do not call `AddEventBus()` separately
- [x] T019 [US3] Add `## Intent Strategies`, `## Retry Backoff Schedule`, and `## OutboxDelivery State Machine` sections to `content/core/messaging/delivery/_index.md` — intent strategies table (send-pending: picks up `NotPublished` records; retry-failed: retries `Failed` records where `NextAttemptOn ≤ Now`; recover-timeout: resets `InProgress` records stuck >15 min back to `Failed`); default backoff table (Attempt 1→10s, 2→5m, 3→10m, 4→30m, 5+→1h per `data-model.md`); state machine ASCII diagram (`NotPublished → InProgress → Published/Failed`) per `data-model.md` § State Transitions
- [x] T020 [US3] Add `## Health Check` and `## See also` sections to `content/core/messaging/delivery/_index.md` — `services.AddHealthChecks().AddOutboxDeliveryHealthCheck<AppDbContext>(opts => { opts.StuckMessageThresholdMinutes = 15; opts.MaxStuckMessages = 50; }, tags: new[] { "live" })` code block with `hl_lines` per `contracts/page-schema.md` § US3; guidance on monitoring stuck-message count; "See also" links (Consumption Setup, Full Setup)

**Checkpoint**: `/core/messaging/delivery/` fully renders. US3 independently testable.

---

## Phase 6: User Story 4 — Consumption Setup (Priority: P2)

**Goal**: Create `content/core/messaging/consumption/_index.md` covering RabbitMQ
consumer registration, `IIntegrationEventHandler<T>` implementation, automatic
idempotency, `MessageContext` header propagation, dead-letter exchange, and health check.

**Independent Test**: Navigate to `/core/messaging/consumption/` and confirm: (1)
complete RabbitMQ consumer DI registration is shown; (2) a full `IIntegrationEventHandler<T>`
implementation is present; (3) `MessageContext` header mapping table is present;
(4) "Why IIntegrationEvent?" is explained; (5) `AddRabbitMQHealthCheck` is shown.
A developer can follow this page alone to produce a working consumer DI block.

### Implementation

- [x] T021 [US4] Create `content/core/messaging/consumption/_index.md` with front matter (`weight=4`, `title="Consumption Setup"`, `date="2026-02-19"`, `draft=false`); write Overview paragraph (consume integration events from a broker queue with automatic deduplication), "When to use" bullet list (service subscribing to events published by other services, needs `IIntegrationEventHandler<T>` dispatch), Prerequisites section (Internal Messaging Setup for `IIdempotencyService`), NuGet packages table (`Juice.EventBus`, `Juice.EventBus.RabbitMQ`, `Juice.Messaging.Idempotency.[Backend]` with links)
- [x] T022 [US4] Add `## DI Registration` section to `content/core/messaging/consumption/_index.md` — complete `services.AddMessaging()` block with `AddIdempotencyRedis()` + `AddDelivery(delivery => { delivery.EventBus.AddConsumerServices(consumers => { consumers.Subscribe<OrderCreatedEvent, OrderCreatedHandler>(); }).AddConsumerRetryPolicies(config.GetSection("RetryPolicies")).AddRabbitMQ(rabbitMQ => { rabbitMQ.AddConnection("rabbitmq", ...).AddConsumer("orders", "orders-queue", "rabbitmq", consumer => { consumer.WithDeadLetterExchange("dlx.orders", routingPattern: "{0}.parking"); }); }); })`; use `hl_lines` per `contracts/page-schema.md` § US4
- [x] T023 [US4] Add `## Why IIntegrationEvent Is Required` and `## IIntegrationEventHandler<T> — Writing a Handler` sections to `content/core/messaging/consumption/_index.md` — explanation: `IntegrationEventDispatcher` uses `IIntegrationEvent.EventName` + type registry to deserialize incoming bytes and route to the correct handler; plain `IMessage` does not carry `EventName` so infrastructure cannot identify its type from bytes (from `research.md` R-007); complete handler example (`OrderCreatedHandler : IIntegrationEventHandler<OrderCreatedEvent>`) with `HandleAsync()` implementation and subscription registration
- [x] T024 [US4] Add `## MessageContext — Header Propagation`, `## Automatic Idempotency in IntegrationEventDispatcher`, `## Dead-Letter Exchange Configuration`, `## Health Check`, and `## See also` sections to `content/core/messaging/consumption/_index.md` — header mapping table (`x-correlation-id`→`CorrelationId`, `x-causation-id`→`CausationId`, `x-tenant-id`→`TenantId`; `MessageContext.Initialize()` called automatically before handler); auto-idempotency note (no handler code changes needed; dispatcher calls `IIdempotencyService.TryCreateRequestAsync` before invoke); DLX routing explanation + `x-death` headers; `services.AddHealthChecks().AddRabbitMQHealthCheck("rabbitmq", tags: new[] { "ready" })` code block with `hl_lines`; "See also" links (Full Setup, v8.5.0 EventBus archive, v8.5.0 Integration archive)

**Checkpoint**: `/core/messaging/consumption/` fully renders. US4 independently testable.

---

## Phase 7: User Story 5 — Full Pipeline Setup (Priority: P3)

**Goal**: Create `content/core/messaging/full-setup/_index.md` composing all four
sub-builders in a single `services.AddMessaging()` call, with sub-builder ordering
rationale and the complete v8.5.0 → v9 migration table.

**Independent Test**: Navigate to `/core/messaging/full-setup/` and confirm: (1) a
single `services.AddMessaging()` block contains all four sub-builder chains in the
documented order; (2) numbered rationale for sub-builder order is present; (3) both
`AddRabbitMQHealthCheck` and `AddOutboxDeliveryHealthCheck` appear in a single health
check chain; (4) the 16-row migration table covers every v8.5.0 public interface.

### Implementation

- [x] T025 [US5] Create `content/core/messaging/full-setup/_index.md` with front matter (`weight=5`, `title="Full Pipeline Setup"`, `date="2026-02-19"`, `draft=false`); write Overview paragraph (single `AddMessaging()` call wiring internal idempotency, outbox, delivery, and RabbitMQ consumption together), Prerequisites section (complete all four setup guides OR follow this guide from scratch), NuGet packages table (all packages from US1–US4 combined: `Juice.Messaging`, `Juice.Messaging.Outbox.EF`, `Juice.Messaging.Outbox.Delivery`, `Juice.MediatR.Behaviors`, `Juice.MediatR.Contracts`, `Juice.EventBus`, `Juice.EventBus.RabbitMQ`, `Juice.Messaging.Idempotency.*` with links)
- [x] T026 [US5] Add `## Sub-Builder Registration Order` and `## Complete DI Registration` sections to `content/core/messaging/full-setup/_index.md` — numbered list explaining WHY order matters (1. `AddPublishingPolicies` before outbox: outbox reads policy at staging time; 2. `AddOutbox` before delivery: delivery builder reads outbox config; 3. `AddIdempotency*` before EventBus dispatcher: dispatcher resolves it at startup; 4. `AddDelivery` last: its constructor auto-calls `services.AddEventBus()`; 5. RabbitMQ inside `delivery.EventBus.AddRabbitMQ()` per `research.md` R-003); complete `services.AddMediatR()` + `services.AddMessaging()` code block with all four sub-builders, `AddRabbitMQHealthCheck`, `AddOutboxDeliveryHealthCheck`, and `hl_lines` per `contracts/page-schema.md` § US5
- [x] T027 [US5] Add `## Migration from v8.5.0` section to `content/core/messaging/full-setup/_index.md` — copy the complete 16-row concept mapping table and "What Is Unchanged" section verbatim from `contracts/migration-table.md`; add `## See also` links (Internal Messaging Setup, Outbox Setup, Delivery Setup, Consumption Setup, v8.5.0 EventBus archive, v8.5.0 MediatR archive, v8.5.0 Integration archive)

**Checkpoint**: `/core/messaging/full-setup/` fully renders with complete DI example and 16-row migration table. US5 independently testable. All six documentation deliverables are now authored.

---

## Phase 8: Polish & Validation

**Purpose**: Hugo build validation, URL link checking, and content checklist per
`quickstart.md` § Step 6. Verifies US6 (archive accessibility) and all new pages.

- [x] T028 Run `hugo build --minify` from repo root — confirm 0 errors and 0 warnings about broken `ref` cross-references
- [x] T029 [P] Verify all 5 new messaging URLs return HTTP 200: `/core/messaging/`, `/core/messaging/outbox/`, `/core/messaging/delivery/`, `/core/messaging/consumption/`, `/core/messaging/full-setup/`
- [x] T030 [P] Verify all 3 archive URLs return HTTP 200: `/core/eventbus/v8.5.0/`, `/core/mediator/v8.5.0/`, `/core/integration/v8.5.0/`
- [x] T031 [P] Content checklist for each new page (`_index.md` × 5): front matter complete (title, date, weight, draft=false), NuGet package table present with `nuget.org` links, at least one code block with `hl_lines`, no placeholder code (`[...]`, `TODO`, ellipsis-only blocks), "See also" links at bottom per `quickstart.md` § Content checklist per new page
- [x] T032 [P] Verify archive pages: each `v8.5.0/_index.md` shows the `⚠ Archived — v8.5.0` banner at the top, does NOT appear in the Hugo primary sidebar navigation, but is reachable by direct URL and via the banner link on the corresponding live page

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 — BLOCKS all user story content work
- **US1 (Phase 3, P1)**: Depends on Phase 2
- **US2 (Phase 4, P1)**: Depends on Phase 2 — parallelizable with US1
- **US3 (Phase 5, P2)**: Depends on Phase 2 — parallelizable with US1 and US2
- **US4 (Phase 6, P2)**: Depends on Phase 2 — parallelizable with US1, US2, US3
- **US5 (Phase 7, P3)**: Depends on Phase 2 — logically authored last (cross-references US1–US4 content)
- **Polish (Phase 8)**: Depends on all content phases (T009–T027)

### User Story Dependencies

- **US1**: No content dependencies — only Phase 2 completion required
- **US2**: No content dependencies on US1 — only Phase 2 completion required
- **US3**: No content dependencies on US2 — only Phase 2 completion (outbox prerequisite is page-level guidance, not implementation dependency)
- **US4**: No content dependencies on US1/US3 — only Phase 2 completion required
- **US5**: Should be authored after US1–US4 so all `## See also` links can be verified accurate
- **US6**: Archive created in Phase 2 (Foundational); accessibility verified in Phase 8 (T030, T032)

### Within Each Phase

- **Phase 2**: T002 ‖ T003 ‖ T004 (parallel archive); then T005 ‖ T006 ‖ T007 (parallel banners, after archive); then T008
- **Phase 3 (US1)**: T009 → T010 → T011 → T012 — sequential (all write sections to the same file)
- **Phase 4 (US2)**: T013 → T014 → T015 → T016 — sequential (same file)
- **Phase 5 (US3)**: T017 → T018 → T019 → T020 — sequential (same file)
- **Phase 6 (US4)**: T021 → T022 → T023 → T024 — sequential (same file)
- **Phase 7 (US5)**: T025 → T026 → T027 — sequential (same file)
- **Phase 8**: T028 first; then T029 ‖ T030 ‖ T031 ‖ T032

### Parallel Opportunities

```
Phase 2 (archive):  T002 ‖ T003 ‖ T004
Phase 2 (banners):  T005 ‖ T006 ‖ T007  (after T002–T004)
After Phase 2:      US1(T009–T012) ‖ US2(T013–T016) ‖ US3(T017–T020) ‖ US4(T021–T024)
Phase 8:            T029 ‖ T030 ‖ T031 ‖ T032  (after T028)
```

---

## Parallel Example: After Foundation

```bash
# Once Phase 2 (Foundational) is complete, all story pages can be created in parallel:
Task A: "Create content/core/messaging/_index.md (US1 — Internal Messaging Setup)"
Task B: "Create content/core/messaging/outbox/_index.md (US2 — Outbox Setup)"
Task C: "Create content/core/messaging/delivery/_index.md (US3 — Delivery Setup)"
Task D: "Create content/core/messaging/consumption/_index.md (US4 — Consumption Setup)"

# US5 is authored last (after US1–US4 complete) to verify cross-links:
Task E: "Create content/core/messaging/full-setup/_index.md (US5 — Full Pipeline Setup)"
```

---

## Implementation Strategy

### MVP First (User Stories 1 and 2 Only — P1)

1. Complete Phase 1: Setup (directory structure)
2. Complete Phase 2: Foundational (archive + banners + navigation)
3. Complete Phase 3: US1 — Internal Messaging Setup (`messaging/_index.md`)
4. Complete Phase 4: US2 — Outbox Setup (`messaging/outbox/_index.md`)
5. **STOP and VALIDATE**: Both P1 stories render cleanly, `hugo build` passes
6. Publish/deploy if needed

### Incremental Delivery

1. Phases 1+2 → Foundation ready (archives live, banners visible, Messaging in nav)
2. Phase 3 → US1 live → validate independently at `/core/messaging/`
3. Phase 4 → US2 live → validate independently at `/core/messaging/outbox/`
4. Phases 5+6 → US3+US4 live (can parallel) → validate independently
5. Phase 7 → US5 live → validate, migration table present
6. Phase 8 → Full build + link check → publish

### Parallel Team Strategy

With multiple contributors after Phase 2 is complete:

- **Contributor A**: US1 (T009–T012) then US5 draft (T025–T027)
- **Contributor B**: US2 (T013–T016)
- **Contributor C**: US3 (T017–T020) then US4 (T021–T024)

Each contributor validates their page(s) independently before US5 is finalized.

---

## Notes

- [P] tasks = different files, no ordering dependency within the same phase
- [Story] label maps each task to its spec.md user story for traceability
- Archive tasks (T002–T004) carry no story label (Foundational phase per template rules); they collectively satisfy US6 (archive accessibility), verified in Phase 8 via T030 and T032
- Each content `_index.md` is built across multiple sequential tasks (e.g., T009+T010+T011+T012 all target `messaging/_index.md`) to enable incremental section-by-section authoring without overwhelming a single task
- All code blocks MUST use `hl_lines` per `quickstart.md` § Code Example Standards — never omit even for short snippets
- All NuGet links MUST follow `[Juice.Messaging](https://www.nuget.org/packages/Juice.Messaging)` format per `quickstart.md` § NuGet Link Format
- Consult `contracts/page-schema.md` for the required section order and exact code block content for each page before authoring
- Consult `research.md` for all confirmed API signatures and method names — do not invent method names from memory
- Commit after each phase checkpoint to preserve incremental progress
