# Feature Specification: v9 Messaging Documentation Update

**Feature Branch**: `001-v9-messaging-docs`
**Created**: 2026-02-18
**Status**: Draft
**Input**: User description: "I want to update documents to replace old doc about eventbus,
integration, mediator by new messaging principle where both mediator request, notification
and integrationevent now inherited from IMessage. They reuse the IIdempotencyService to
handle deduplicated incoming integration events and mediator IIdempotentRequest instead of
IRequestManager pattern. The current doc must be stored as v8.5.0 for archive."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Setup Messaging for Internal Messages Only (Priority: P1)

A developer building a service that only uses in-process messaging (MediatR commands,
queries, and notifications) wants to enable idempotent request handling without any
external broker or outbox. They follow the documentation to register MediatR with its
handlers and pipeline behaviors, add `IdempotencyBehavior` to the MediatR pipeline,
and register the Messaging Idempotency service with their chosen backend (EF, caching,
or Redis). When done, every `IIdempotentRequest` command in their service is
automatically deduplicated without any broker, outbox table, or delivery worker.

**Why this priority**: This is the simplest, most self-contained setup. It documents
the MediatR + idempotency foundation that all other setups build on top of. A developer
must understand this layer before adding outbox or broker capabilities.

**Independent Test**: A reader can follow only the "Internal Messaging Setup" guide and
produce a working DI registration that includes MediatR, at least one pipeline behavior
(`IdempotencyBehavior`), and one idempotency backend — with no outbox or broker code
present — and have their `IIdempotentRequest` handlers deduplicated.

**Acceptance Scenarios**:

1. **Given** a developer reads the Internal Messaging setup guide, **When** they follow
   the step-by-step DI registration example, **Then** they can register MediatR, its
   handlers, and `IdempotencyBehavior` as a pipeline behavior in a single, coherent code
   block with no extraneous broker or outbox dependencies.
2. **Given** a developer reads the idempotency backend section, **When** they review the
   backend comparison, **Then** they can choose among EF, caching, and Redis backends and
   add the correct registration call for exactly one backend.
3. **Given** a developer reads the `IIdempotentRequest` section, **When** they compare it
   to the v8.5.0 `IRequestManager` pattern, **Then** they understand that marking a
   command with `IIdempotentRequest` is the only change needed for the framework to call
   `IIdempotencyService` automatically via the behavior — no manual `IRequestManager`
   calls required.
4. **Given** a developer reads the message hierarchy callout, **When** they review the
   overview, **Then** they understand that MediatR requests, notifications, domain events,
   and `IntegrationEvent` all implement `IMessage`, giving them shared `MessageId`,
   `CreatedAt`, and `TenantId` fields, and that `IIntegrationEvent` is the consume-side
   contract used only by the broker infrastructure.

---

### User Story 2 - Setup Messaging with Outbox Only (Priority: P1)

A developer wants their service to publish messages transactionally using the Outbox
pattern — without yet setting up a delivery worker or broker (e.g., delivery may run
in a separate process). They follow the documentation to add `TransactionBehavior` to
the MediatR pipeline (so every command runs inside a managed transaction), register
`IOutboxService` against their DbContext, and configure publishing policies
(`IMessagePublishingPolicy`) that define where messages should be routed when a delivery
worker eventually picks them up. Any `IMessage` — including plain domain events — can
be staged in the outbox without converting to an `IntegrationEvent` first.

**Why this priority**: Transactional outbox staging is the mandatory replacement for
direct-publish calls from v8. Getting this setup correct is the highest-impact docs
change because mistakes here cause message loss or duplicate delivery.

**Independent Test**: A reader can follow only the "Outbox Setup" guide and produce a
working DI registration that stages a domain event via `IOutboxService` inside a
`TransactionBehavior`-managed transaction, with a publishing policy configured —
without any delivery worker or RabbitMQ dependency present.

**Acceptance Scenarios**:

1. **Given** a developer reads the Outbox setup guide, **When** they follow the DI
   registration example, **Then** they can register `TransactionBehavior<TRequest,
   TResponse, TContext>` as a MediatR pipeline behavior and `IOutboxService<TContext>`
   bound to their DbContext, using the `MessagingBuilder` outbox sub-builder.
2. **Given** a developer reads the "Publish any IMessage" section, **When** they review
   the code examples, **Then** they see both a domain event example (plain `IMessage`)
   and an integration event example staged via `AddEventAsync`, and understand both
   work identically for publishing — `IIntegrationEvent` is only needed on the
   consuming side.
3. **Given** a developer reads the publishing policy section, **When** they follow the
   configuration example, **Then** they can define `IMessagePublishingPolicy` rules that
   map a domain and event type to a `PublishRoute(PublisherKey, Destination)` without
   touching application code.
4. **Given** a developer reads the migration note, **When** they compare the new Outbox
   staging pattern to the v8.5.0 `IIntegrationEventService` pattern, **Then** they can
   identify the equivalent of each v8.5.0 step (`SaveEventAsync`, `MarkEventAsInProgress`,
   `MarkEventAsPublished`, `MarkEventAsFailed`) in the new outbox model.

---

### User Story 3 - Setup Messaging Delivery (Priority: P2)

A developer has an outbox configured and now wants to add the background delivery worker
that reads `OutboxDelivery` records and forwards them to the broker. They follow the
documentation to register the delivery processor against their DbContext and a named
transport publisher, configure the three intent strategies (send-pending, retry-failed,
recover-timeout), set retry backoff delays, and add the outbox delivery health check so
operations can monitor for stuck messages.

**Why this priority**: Delivery is a separate, independently deployable concern from
outbox staging. It can be added after the outbox is working and tested, so it is
lower priority than the outbox setup itself.

**Independent Test**: A reader can follow only the "Delivery Setup" guide and produce
a working DI registration that runs `DeliveryHostedService`, processes pending
`OutboxDelivery` records for a named publisher, retries failed ones, and recovers timed-
out in-progress records — independently of the outbox staging and consumption setups.

**Acceptance Scenarios**:

1. **Given** a developer reads the Delivery setup guide, **When** they follow the
   `MessagingBuilder` delivery sub-builder example, **Then** they can register
   `DeliveryProcessor<TContext>` bound to a specific publisher key (e.g., `"rabbitmq"`)
   and their DbContext.
2. **Given** a developer reads the intent strategies section, **When** they review the
   example, **Then** they can enable all three strategies (`send-pending`, `retry-failed`,
   `recover-timeout`) for a processor with a single fluent call and understand the
   purpose and trigger condition of each.
3. **Given** a developer reads the retry backoff section, **When** they look at the
   default policy table (10s → 5m → 10m → 30m → 1h), **Then** they understand the
   default schedule and can override it with a custom delivery policy configuration.
4. **Given** a developer reads the health check section, **When** they follow the
   example, **Then** they can register `AddOutboxDeliveryHealthCheck<TContext>` with a
   configurable stuck-message threshold (minutes and count) on the `"live"` tag.

---

### User Story 4 - Setup Messaging Consumption (Priority: P2)

A developer wants their service to consume integration events from a RabbitMQ queue.
They follow the documentation to register `IIntegrationEventHandler<T>` implementations,
configure the RabbitMQ consumer engine through the `MessagingBuilder` RabbitMQ
sub-builder (connection, queue binding, dead-letter exchange), and register the Messaging
Idempotency service so that `IntegrationEventDispatcher` automatically deduplicates
re-delivered messages. The docs explain why consuming requires `IIntegrationEvent` (not
just `IMessage`) and how `MessageContext` fields (CorrelationId, CausationId) are
automatically extracted from incoming headers.

**Why this priority**: Consumption is independently deployable but depends on a
publisher existing. It is parallel with delivery (both are P2) and can be developed
and tested independently using a mock or test broker.

**Independent Test**: A reader can follow only the "Consumption Setup" guide and produce
a working DI registration that starts a RabbitMQ consumer, routes incoming messages to
the correct `IIntegrationEventHandler<T>`, deduplicates via `IIdempotencyService`, and
sends unhandled messages to a dead-letter exchange — without any outbox or delivery code.

**Acceptance Scenarios**:

1. **Given** a developer reads the Consumption setup guide, **When** they follow the
   `MessagingBuilder` RabbitMQ sub-builder example, **Then** they can add a connection,
   bind a consumer queue, subscribe handlers with
   `.Subscribe<TEvent, THandler>()`, and configure a dead-letter exchange in a single
   fluent block.
2. **Given** a developer reads the idempotency integration section, **When** they review
   it, **Then** they understand that `IntegrationEventDispatcher` calls
   `IIdempotencyService.TryCreateRequestAsync` automatically before invoking a handler,
   and that they only need to register one idempotency backend — no handler code changes.
3. **Given** a developer reads the "Why IIntegrationEvent?" section, **When** they
   review the explanation, **Then** they understand that the consumer infrastructure
   uses `IIntegrationEvent.EventName` and type registry to deserialize incoming bytes
   and route to the correct handler, which is why plain `IMessage` is insufficient on
   the consume side.
4. **Given** a developer reads the `MessageContext` section, **When** they look at the
   header mapping table, **Then** they can identify which incoming header (`x-correlation-id`,
   `x-causation-id`, `x-tenant-id`) populates which `MessageContext` field, and
   understand that `MessageContext.Initialize()` is called automatically by the consumer
   engine before handler invocation.
5. **Given** a developer reads the health check section, **When** they follow the
   example, **Then** they can register `AddRabbitMQHealthCheck` with a named connection
   on the `"ready"` tag.

---

### User Story 5 - Setup Full Messaging Pipeline (Priority: P3)

A developer building a complete microservice needs all four capabilities — internal
MediatR idempotency, outbox staging, background delivery, and RabbitMQ consumption —
wired together in a single, coherent DI setup. They follow the "Full Setup" guide to
chain all sub-builders under one `services.AddMessaging()` call, understand the order
in which sub-builders must be registered, and arrive at a production-ready configuration
with health checks for both the broker and the outbox delivery backlog.

**Why this priority**: The full setup is the sum of the four independent setups above.
It should be documented as a composition guide, not a new concept — making it P3.

**Independent Test**: A reader can follow only the "Full Setup" guide and produce a
complete, working DI registration that combines MediatR + `IdempotencyBehavior`,
`TransactionBehavior` + `IOutboxService`, `DeliveryProcessor`, and `RabbitMQConsumer`
in a single `AddMessaging()` call — producing no compiler errors and no missing
service registrations.

**Acceptance Scenarios**:

1. **Given** a developer reads the Full Setup guide, **When** they copy the complete
   DI registration example, **Then** the snippet contains all four sub-builder chains
   (`AddMediatR`, outbox, delivery, and RabbitMQ) in the correct order inside a single
   `services.AddMessaging(builder => { ... })` call.
2. **Given** a developer reads the sub-builder ordering section, **When** they review
   the explanation, **Then** they know whether sub-builder order is significant (e.g.,
   whether the outbox sub-builder must precede the delivery sub-builder).
3. **Given** a developer reads the full health check section, **When** they follow the
   example, **Then** they register both `AddRabbitMQHealthCheck` (broker) and
   `AddOutboxDeliveryHealthCheck` (backlog) in a single health check builder chain.

---

### User Story 6 - Access Archived v8.5.0 Documentation (Priority: P3)

A developer maintaining a v8.5.0 application needs to reference the original EventBus,
Integration service, and MediatR docs that existed before the v9 update. They can
navigate to a clearly labeled archive section and read those pages without confusion
with the current docs.

**Why this priority**: Existing v8.5.0 users must not lose their reference material.
Archiving is a separate, independent deliverable with no risk to the new docs.

**Independent Test**: A reader can navigate to the archive section and find the complete
original text of the v8.5.0 EventBus, Integration service, and MediatR pages, clearly
labeled as "v8.5.0 archive", without those pages appearing in the main navigation.

**Acceptance Scenarios**:

1. **Given** a reader opens the docs site, **When** they navigate to the archive section,
   **Then** they find pages labeled "v8.5.0" for EventBus, Integration service, and
   MediatR that contain the original pre-v9 content unchanged.
2. **Given** a reader is on any new v9 messaging page, **When** they look for a link to
   the archived equivalent, **Then** they find a clearly visible "v8.5.0 archive"
   reference or banner pointing to the archived page.
3. **Given** a developer searches for "IRequestManager" on the docs site, **When** they
   view results, **Then** they find it in the v8.5.0 archive with a note pointing to
   `IIdempotentRequest` as the v9 replacement.

---

### Edge Cases

- When a developer publishes a plain domain event (`IMessage`) via the outbox, and
  another service wants to consume it, must the publisher also define an
  `IIntegrationEvent` version? The docs MUST answer this question explicitly.
- How are pages in the "v8.5.0 archive" section excluded from the default Hugo
  navigation menu while remaining discoverable via site search?
- How should `MessagingBuilder` sub-builder ordering be documented — does the order of
  chaining matter (e.g., must outbox sub-builder precede delivery sub-builder)?
- If a service uses only US1 (internal) and US2 (outbox) but not US3 (delivery), which
  health checks should it register? The docs MUST address partial-setup health checks.
- If a developer uses both `IdempotencyBehavior` (MediatR side) and the automatic
  idempotency in `IntegrationEventDispatcher` (consume side), do they share the same
  `IIdempotencyService` registration or do they need separate registrations?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Documentation MUST archive the existing EventBus, Integration service,
  and MediatR pages verbatim as a v8.5.0 version, clearly labeled and accessible
  from both the archive section and a visible banner on each corresponding new page.
- **FR-002**: Documentation MUST include a message hierarchy overview (inline or as a
  dedicated page) explaining that domain events, `IntegrationEvent`, `IIdempotentRequest`,
  and MediatR notifications all implement `IMessage`, carrying `MessageId`, `CreatedAt`,
  and `TenantId`, and distinguishing the publish-side role of `IMessage` from the
  consume-side role of `IIntegrationEvent`.
- **FR-003**: Documentation MUST provide a standalone "Internal Messaging Setup" guide
  covering: MediatR + handler registration, `IdempotencyBehavior` pipeline behavior, and
  idempotency backend selection (EF, caching, Redis) — with no outbox or broker steps.
- **FR-004**: Documentation MUST provide a standalone "Outbox Setup" guide covering:
  `TransactionBehavior` registration, `IOutboxService<TContext>` binding, publishing
  policy (`IMessagePublishingPolicy`) configuration, and a code example staging a plain
  domain event without converting it to an `IntegrationEvent`.
- **FR-005**: Documentation MUST provide a standalone "Delivery Setup" guide covering:
  `DeliveryProcessor<TContext>` registration, the three intent strategies
  (send-pending, retry-failed, recover-timeout), retry backoff defaults and override,
  and `AddOutboxDeliveryHealthCheck` registration.
- **FR-006**: Documentation MUST provide a standalone "Consumption Setup" guide
  covering: `IIntegrationEventHandler<T>` registration, RabbitMQ sub-builder
  configuration (connection, queue binding, dead-letter exchange), automatic idempotency
  via `IntegrationEventDispatcher`, `MessageContext` header-to-field mapping, and
  `AddRabbitMQHealthCheck` registration. It MUST explain why `IIntegrationEvent` is
  required on the consuming side when `IMessage` suffices for publishing.
- **FR-007**: Documentation MUST provide a "Full Setup" guide showing all four
  sub-builders chained in a single `services.AddMessaging()` call, with an explicit
  note on whether sub-builder ordering is significant.
- **FR-008**: Documentation MUST include a migration comparison table mapping every
  v8.5.0 public messaging concept (`IEventBus`, `IntegrationEventLog`,
  `IIntegrationEventService`, `IRequestManager`, direct-publish pattern) to its
  v9 equivalent.
- **FR-009**: Documentation MUST document `IIdempotentRequest` as the replacement for
  `IRequestManager` and clarify that a single `IIdempotencyService` registration serves
  both the `IdempotencyBehavior` (MediatR side) and `IntegrationEventDispatcher`
  (consume side).
- **FR-010**: All setup guides MUST use `MessagingBuilder` and its sub-builders as the
  DI entry point. No guide MUST show registrations that bypass `AddMessaging()`.
- **FR-011**: All new documentation pages MUST include at least one complete,
  compilable code example per major concept, using Hugo `hl_lines` for emphasis.
- **FR-012**: All new documentation pages MUST include the corresponding NuGet package
  names and links for every concept they describe.
- **FR-013**: New pages MUST be reachable from the Hugo site navigation under an
  appropriate section (e.g., "Core" or a new "Messaging" section).
- **FR-014**: Archived v8.5.0 pages MUST NOT appear in the primary navigation menu but
  MUST remain reachable via direct link and site search.

### Key Entities

- **Message hierarchy overview**: Inline callout or standalone page showing the
  `IMessage` type tree and the publish-vs-consume role table for `IMessage` /
  `IIntegrationEvent`.
- **Internal Messaging Setup guide**: Standalone setup page for US1 (MediatR +
  `IdempotencyBehavior` + idempotency backend, no broker).
- **Outbox Setup guide**: Standalone setup page for US2 (`TransactionBehavior` +
  `IOutboxService` + publishing policies).
- **Delivery Setup guide**: Standalone setup page for US3 (`DeliveryProcessor` +
  intent strategies + retry policy + health check).
- **Consumption Setup guide**: Standalone setup page for US4 (RabbitMQ consumer +
  `IIntegrationEventHandler<T>` + idempotency + `MessageContext` + health check).
- **Full Setup guide**: Composition page for US5 showing all four sub-builders combined.
- **v8.5.0 Archive pages**: Verbatim copies of the current EventBus, Integration
  service, and MediatR pages under a versioned archive path.
- **Migration comparison table**: Structured table mapping every v8.5.0 public
  messaging interface to its v9 equivalent, included in the Full Setup guide or as
  a standalone reference page.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A developer can follow any single setup guide (US1–US4) in isolation and
  produce a working DI registration for that scenario without reading any other guide.
- **SC-002**: All six documentation deliverables (Internal, Outbox, Delivery,
  Consumption, Full Setup, v8.5.0 Archive) are live and reachable from the Hugo site
  within one release cycle.
- **SC-003**: Every setup guide contains at least one complete, compilable code example;
  zero guides are published with placeholder or stub code.
- **SC-004**: The v8.5.0 archive pages for EventBus, Integration service, and MediatR
  are accessible via direct URL and return a valid response after the update.
- **SC-005**: The migration comparison table covers 100% of the v8.5.0 public
  messaging interfaces (`IEventBus`, `IntegrationEventLog`, `IRequestManager`,
  `IIntegrationEventService`) with a v9 equivalent for each.
- **SC-006**: A developer migrating from v8.5.0 can reach the archived equivalent of
  any current page within two navigation steps from the new page.
- **SC-007**: The Full Setup guide produces a DI registration that covers all four
  sub-builder chains in a single `AddMessaging()` call, verified by a complete code
  example with no missing registrations.
