# Feature Specification: MediatR Patterns, Local Transport & MessageContext Documentation

**Feature Branch**: `002-v9-docs-mediator-local`
**Created**: 2026-03-14
**Status**: Draft
**Input**: User description: "Add MediatR patterns guide, local transport, and MessageContext documentation"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - MediatR Patterns Guide (Priority: P1)

A developer new to the Juice framework visits the Mediator documentation page to learn how to implement in-process messaging using MediatR. They need to understand the three MediatR patterns (Request/Response, Notification, Stream Request), see concrete implementation examples for each pattern and its handler, and learn how to register MediatR with the Juice DI extensions.

**Why this priority**: MediatR is the foundational in-process messaging mechanism used across all Juice services. Every developer needs this knowledge before they can implement commands, queries, or domain event handlers. Without this guide, developers must reverse-engineer the patterns from source code.

**Independent Test**: Can be fully tested by navigating to `/core/mediator/` and verifying the page contains concept explanations, code examples for all three patterns, and registration guidance. Delivers immediate value as a standalone reference.

**Acceptance Scenarios**:

1. **Given** a developer visits the Mediator documentation page, **When** they read the overview, **Then** they see a summary table of all three MediatR patterns with their interfaces, handler interfaces, and use cases.
2. **Given** a developer needs to implement a command, **When** they read the Request/Response section, **Then** they find a complete example showing message definition (with `MessageBase` + `IRequest<T>`), handler implementation with constructor injection, and dispatch via `mediator.Send()`.
3. **Given** a developer needs to broadcast a domain event in-process, **When** they read the Notification section, **Then** they find an example showing message definition, multiple handler implementations, dispatch via `mediator.Publish()`, and a note on sequential execution behavior.
4. **Given** a developer needs async streaming results, **When** they read the Stream Request section, **Then** they find an example showing `IStreamRequest<T>` definition, `IAsyncEnumerable<T>` handler with `yield return`, and consumption via `mediator.CreateStream()`.
5. **Given** a developer is setting up MediatR, **When** they read the Registration section, **Then** they find `AddMediatR` configuration with `RegisterServicesFromAssembly`, optional behaviors (`AddIdempotencyRequestBehavior`, `AddOpenBehavior`), and a table confirming all handler types are auto-discovered as Transient.

---

### User Story 2 - Local Transport Documentation (Priority: P1)

A developer building a monolith or modular monolith needs to dispatch messages to handlers within the same process without requiring a broker. They visit the Local Transport documentation page to understand the two in-process routes (`"local-channel"` for fire-and-forget, `"local"` for durable outbox-backed dispatch), configure routing via `IMessagePublishingPolicy`, and implement `IMessageService` publishing.

**Why this priority**: Local transport is a new v9 capability that enables monolith and modular monolith architectures. It fills the gap between pure MediatR (no durability) and full broker messaging (infrastructure overhead). Without documentation, developers cannot discover or adopt this feature.

**Independent Test**: Can be fully tested by navigating to `/core/messaging/local-transport/` and verifying the page explains both routes, provides DI registration examples, shows publishing policy configuration, and documents idempotency behavior across dispatch paths.

**Acceptance Scenarios**:

1. **Given** a developer visits the Local Transport page, **When** they read the overview, **Then** they see a comparison table of the three routes (`"local-channel"`, `"local"`, `"rabbitmq"`) with durability, latency, and use case columns.
2. **Given** a developer wants fire-and-forget dispatch, **When** they read the DI Registration section, **Then** they find a minimal `AddLocalChannel` example that requires no database.
3. **Given** a developer wants durable in-process dispatch, **When** they read the full registration example, **Then** they find `AddMessageService<TContext>`, `AddLocalPublisher<TContext>`, and `AddDelivery` configured with both `"local"` and `"rabbitmq"` processor keys.
4. **Given** a developer wants config-driven routing, **When** they read the Publishing Policy section, **Then** they find a complete `appsettings.json` example showing domain-based routing rules with priority ordering.
5. **Given** a developer is concerned about duplicate handling, **When** they read the Idempotency section, **Then** they understand how the `"local"` route dispatches twice (immediate + delivery retry) and how the idempotency key `(EventName, "{Source}:{MessageId}")` prevents duplicate handler invocation.

---

### User Story 3 - MessageContext Documentation (Priority: P1)

A developer needs to understand how `MessageContext` provides distributed tracing (CorrelationId, CausationId, ExecutionId, Source) across async call chains and service boundaries. They visit the MessageContext documentation page to learn how to initialize it at every entry point (HTTP middleware, broker consumer, test, delivery retry), access it via `IMessageContextAccessor`, and understand how header propagation works through the outbox pipeline.

**Why this priority**: `MessageContext` is required at every entry point — without proper initialization, messaging operations throw `InvalidOperationException`. Every developer who writes an HTTP endpoint that publishes events, or a consumer that handles them, needs this guide.

**Independent Test**: Can be fully tested by navigating to `/core/messaging/message-context/` and verifying the page contains lifecycle diagrams, initialization examples for all entry point types, field explanations, and a cross-service tracing example.

**Acceptance Scenarios**:

1. **Given** a developer visits the MessageContext page, **When** they read the overview, **Then** they understand that `MessageContext` is `AsyncLocal`-based and must be initialized at every entry point.
2. **Given** a developer building an HTTP API, **When** they read the initialization section, **Then** they find both middleware (`UseMessageContext`) and attribute (`[InitializeMessageContext]`) options with code examples.
3. **Given** a developer debugging cross-service communication, **When** they read the tracing example, **Then** they can follow CorrelationId, CausationId, and ExecutionId across 4 services in a sequence diagram.
4. **Given** a developer writing tests, **When** they read the test initialization section, **Then** they find the `[InitializeMessageContext]` attribute and manual `MessageContext.Initialize()` patterns.

---

### User Story 4 - Cross-Reference Integration (Priority: P2)

A developer following any existing messaging guide (Outbox, Delivery, Consumption, Full Setup, or the Messaging index) needs to discover the new Mediator, Local Transport, and MessageContext pages through cross-references. Existing pages must link to the new content without breaking any existing navigation.

**Why this priority**: Documentation discoverability depends on cross-linking. New pages that are not referenced from related guides remain hidden. This is lower priority because the pages can still be found via the sidebar navigation.

**Independent Test**: Can be tested by visiting each existing messaging page and verifying "See also" sections and Setup Guides table include links to the new pages. Hugo build must produce zero errors.

**Acceptance Scenarios**:

1. **Given** a developer is on the Messaging index page, **When** they look at the Setup Guides table, **Then** they see rows for "Mediator", "Local Transport", and "MessageContext" linking to the correct pages.
2. **Given** a developer is on the Outbox, Delivery, or Full Setup page, **When** they scroll to "See also", **Then** they find a link to the Local Transport page.
3. **Given** a developer is on the Consumption page, **When** they read the MessageContext section, **Then** they find a cross-reference link to the full MessageContext Setup page.
4. **Given** the Full Setup page, **When** a developer reads the NuGet packages table, **Then** it includes `Juice.Messaging.Local` with its purpose.
5. **Given** the Full Setup page, **When** a developer reads the DI Registration example, **Then** it shows `AddMessageService`, `AddLocalPublisher`, and `AddDeliveryProcessor` with the `"local"` key alongside the existing `"rabbitmq"` key.
6. **Given** the Full Setup page, **When** a developer reads the migration table, **Then** it includes two new rows for `IMessageService` and `LocalTransportPublisher` (marked as "new in v9").
7. **Given** all documentation changes are applied, **When** `hugo --minify` is run, **Then** the build produces zero errors and generates all expected pages.

---

### Edge Cases

- What happens when a developer follows the Local Transport guide without having set up the Outbox first? The page must clearly state prerequisites and link to the Outbox Setup guide.
- What happens when a developer uses `IMessageService` (non-generic) with a `"local"` route that requires outbox writes? The documentation must explain that `IMessageService<TContext>` is required for outbox-backed routes.
- What happens when `"local"` route immediate dispatch fails? The documentation must explain that the outbox record remains `NotPublished` and `DeliveryHostedService` retries it.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The Mediator page MUST explain the concept of MediatR as an in-process mediator pattern with three message types.
- **FR-002**: The Mediator page MUST provide complete, compilable code examples for Request/Response (command + query), Notification (multiple handlers), and Stream Request (IAsyncEnumerable handler + consumer) patterns.
- **FR-003**: The Mediator page MUST include a Registration section showing `AddMediatR` with `RegisterServicesFromAssembly`, optional behaviors, and a handler discovery table.
- **FR-004**: The Mediator page MUST include NuGet package references with links to nuget.org.
- **FR-005**: The Mediator page MUST include a "See also" section linking to Messaging, Outbox Setup, and the v8.5.0 archive.
- **FR-006**: The Local Transport page MUST explain the `"local-channel"` (non-durable) and `"local"` (outbox-backed) routes with a comparison table.
- **FR-007**: The Local Transport page MUST provide DI registration examples for both local-channel-only and full (all routes) setups.
- **FR-008**: The Local Transport page MUST document `IMessagePublishingPolicy` configuration via `appsettings.json` with domain-based routing rules.
- **FR-009**: The Local Transport page MUST explain the dual-dispatch idempotency mechanism for the `"local"` route, including the key format and cross-scope deduplication requirements.
- **FR-010**: The Local Transport page MUST document the `IMessageService` and `IMessageService<TContext>` interfaces and when to use each.
- **FR-011**: The Local Transport page MUST include an ASCII flow diagram showing the `"local"` route's publish-outbox-channel-delivery-idempotency path.
- **FR-012**: The MessageContext page MUST explain the `AsyncLocal`-based lifecycle and the requirement to initialize at every entry point.
- **FR-013**: The MessageContext page MUST provide initialization examples for HTTP (middleware and attribute), broker consumer, test, and delivery retry entry points.
- **FR-014**: The MessageContext page MUST explain CorrelationId, CausationId, ExecutionId, and Source fields with a visual diagram.
- **FR-015**: The MessageContext page MUST include a cross-service tracing example (sequence diagram) showing header propagation across multiple services.
- **FR-016**: The MessageContext page MUST document `IMessageContextAccessor` for reading context values in handlers.
- **FR-017**: Existing pages (Messaging index, Outbox, Delivery, Consumption, Full Setup) MUST be updated with cross-reference links to the new pages.
- **FR-018**: The Full Setup page MUST be updated with Local Transport NuGet packages, DI registration steps, and migration table entries.
- **FR-019**: All new and updated pages MUST use `hl_lines` on code blocks, include nuget.org links, and have a "See also" section.
- **FR-020**: The Hugo site MUST build with zero errors after all changes.

### Key Entities

- **MediatR patterns**: Request/Response (`IRequest<T>`), Notification (`INotification`), Stream Request (`IStreamRequest<T>`) — three in-process messaging patterns with their handler counterparts
- **Local Transport routes**: `"local-channel"` (non-durable in-memory channel), `"local"` (outbox-backed with immediate dispatch) — two in-process delivery routes
- **IMessageService**: Unified publishing interface that routes messages based on `IMessagePublishingPolicy` configuration; generic variant `IMessageService<TContext>` supports outbox-backed routes
- **LocalTransportPublisher**: Outbox delivery publisher that dispatches in-process with idempotency deduplication, keyed as `"local"` in the delivery processor registry
- **MessageContext**: `AsyncLocal`-based context carrying CorrelationId, CausationId, ExecutionId, Source, and TenantId across async call chains; must be initialized at every entry point

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A developer can implement all three MediatR patterns (Request/Response, Notification, Stream Request) by following only the Mediator documentation page, without consulting external resources.
- **SC-002**: A developer can set up local-channel-only in-process messaging by following the Local Transport page in under 10 minutes.
- **SC-003**: A developer can set up durable local transport with config-driven routing by following the Local Transport page in under 20 minutes.
- **SC-004**: Every new documentation page is reachable from at least two other pages via cross-reference links (sidebar + "See also" or Setup Guides table).
- **SC-005**: A developer can initialize `MessageContext` at an HTTP entry point and trace a request across services by following only the MessageContext documentation page.
- **SC-006**: The Hugo documentation site builds with zero errors and generates all expected page URLs including `/core/mediator/`, `/core/messaging/local-transport/`, `/core/messaging/message-context/`.
- **SC-007**: All code examples on new pages include `hl_lines` for key lines, nuget.org package links, and a "See also" section with relevant cross-references.
