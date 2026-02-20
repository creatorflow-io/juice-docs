<!--
Sync Impact Report
===================
Version change: 1.0.0 → 1.1.0

Rationale: MINOR bump — material guidance updates to Principles III and VII
  reflecting the v9 messaging architecture (Outbox pattern, policy-based
  routing, idempotency service). Principle names and intent are unchanged;
  only concrete implementation guidance updated.

Modified principles:
  - III. Deployment Flexibility → III. Deployment Flexibility
      Old: IEventBus abstraction with RabbitMQ default
      New: IOutboxService + IMessagePublishingPolicy + ITransportPublisher
           with full outbox/delivery lifecycle
  - VII. Observability & Auditability → VII. Observability & Auditability
      Old: IIntegrationEventLogService tracks event states
      New: OutboxEvent/OutboxDelivery two-table model, IIdempotencyService,
           health checks for outbox delivery and broker connection

Added sections: None

Removed sections: None

Templates requiring updates:
  - .specify/templates/plan-template.md ✅ compatible (no hard-coded
    messaging API references)
  - .specify/templates/spec-template.md ✅ compatible
  - .specify/templates/tasks-template.md ✅ compatible

Follow-up TODOs: None
-->

# Juice Constitution

## Core Principles

### I. Simplicity & Lightweight

Every component in Juice MUST remain simple, easy to use, and
lightweight. The codebase provides extensions and small tools—not a
full framework—so developers start with only what they need.

- New features MUST NOT introduce unnecessary abstractions or
  heavy runtime overhead.
- Components MUST start quickly and work in low-memory environments.
- YAGNI (You Aren't Gonna Need It) applies: do not add functionality
  until it is actually required.
- Complexity MUST be justified; three similar lines of code are
  preferred over a premature abstraction.

### II. Domain-Driven Design

All feature development MUST follow Domain-Driven Design patterns
as the foundational architecture.

- Aggregate roots MUST implement `IAggregrateRoot<TNotification>` and
  encapsulate domain logic within their boundaries.
- Commands and command handlers MUST be separated from the domain model.
- Domain events MUST be used for intra-service side effects; integration
  events (extending `IntegrationEvent` record) MUST be used for
  cross-service communication.
- The UnitOfWork pattern (`IUnitOfWork`) MUST be used for transactional
  consistency within a bounded context.

### III. Deployment Flexibility

The same domain code MUST be deployable as either a monolithic
application or as independent microservices without modification.

- Domain and infrastructure layers MUST NOT contain deployment-model
  assumptions; no direct broker calls in domain code.
- Modular registration (`ModuleStartup`) MUST be used for monolithic
  composition; standalone startup for microservice deployment.
- Cross-service messaging MUST use the Outbox pattern:
  - Stage events atomically within the domain transaction via
    `IOutboxService<TContext>.AddEventAsync()` and `SaveEventsAsync()`.
  - A background `DeliveryHostedService` drives delivery asynchronously
    using intent strategies (send-pending, retry-failed,
    recover-timeout).
  - Transport publishers (`ITransportPublisher`) MUST be registered as
    keyed services, enabling multiple brokers (RabbitMQ, future
    providers) to coexist.
- Message routing MUST be governed by `IMessagePublishingPolicy`,
  which resolves `PublishRoute(PublisherKey, Destination)` based on
  domain, tenant, and event type — keeping routing rules out of
  application code.

### IV. Integration-Ready

All libraries and services MUST integrate with third-party libraries
and external systems without conflict.

- Public APIs MUST be defined in separate `*.Api.Contracts` projects
  containing DTOs, Protos, and IntegrationEvents.
- Integration event contracts MUST extend `IntegrationEvent` (abstract
  record implementing `IMessage` and `IIntegrationEvent`) and reside
  in `Juice.EventBus.Contracts` or a feature-specific contracts project.
- gRPC, REST, and SignalR endpoints MUST coexist; contract projects
  MUST NOT depend on implementation details.

### V. Modular Architecture

Features MUST be self-contained modules that can be independently
developed, tested, and composed.

- Each feature MUST be organized as `{namespace}.{feature}` with clear
  Domain, Commands, Events, and CommandHandlers folders.
- Modules MUST declare dependencies and incompatibilities via the
  `[Feature]` attribute.
- Cross-module communication MUST go through MediatR (in-process) or
  the messaging outbox (cross-process); direct class references between
  unrelated modules are prohibited.

### VI. Multi-Tenancy by Design

Multi-tenancy MUST be a first-class concern, not an afterthought.

- Tenant identification and isolation MUST use the established
  `MultiTenantDbContext` and Finbuckle integration.
- Per-tenant configuration, options, and data isolation MUST be
  supported in every data-bearing service.
- Cross-tenant entity access MUST be explicitly declared using
  `IsCrossTenant()` and protected by per-tenant authentication
  conventions.
- The `TenantId` field on `IMessage` and the `x-tenant-id` message
  header MUST be propagated through `MessageContext.Current` so that
  every published and consumed event carries full tenant context.

### VII. Observability & Auditability

All services MUST provide structured logging, audit capabilities, and
messaging health checks to ensure debuggability in distributed
environments.

- Database contexts MUST implement `IAuditableDbContext` for automatic
  change tracking (CreatedUser, ModifiedUser, timestamps).
- Data change events MUST be dispatched through MediatR for downstream
  audit processing.
- Logging MUST use the .NET `ILogger` abstraction with support for
  custom `LoggerProvider` implementations.
- Outbox delivery state MUST be tracked using the two-table model:
  - `OutboxEvent` — the persisted, serialized message.
  - `OutboxDelivery` — per-publisher delivery attempt with states:
    NotPublished → InProgress → Published / Failed / Skipped.
  - `IIdempotencyService` MUST be used on the consuming side to
    prevent duplicate event processing (EF, caching, or Redis backends
    are available).
- Health checks MUST be registered for:
  - Broker connectivity (`AddRabbitMQHealthCheck`).
  - Outbox delivery backlog (`AddOutboxDeliveryHealthCheck`).
- `MessageContext` (CorrelationId, CausationId, ExecutionId, Source)
  MUST be initialised at every service entry point and propagated
  through all message headers (`x-correlation-id`, `x-causation-id`).

## Technology Stack & Constraints

- **Language**: C# on .NET (latest LTS or current supported version).
- **ORM**: Entity Framework Core with support for SQL Server and
  PostgreSQL; dynamic schema via `ISchemaDbContext`.
- **Messaging (v9+)**: Outbox-first publishing pipeline:
  - `Juice.Messaging` — core abstractions (`IMessage`, `IEvent`,
    `IOutboxService`, `IMessagePublishingPolicy`, `ITransportPublisher`,
    `IMessageSerializer`).
  - `Juice.Messaging.Outbox.EF` — EF-backed outbox repository
    (`OutboxEvent` + `OutboxDelivery` tables).
  - `Juice.Messaging.Outbox.Delivery` — background delivery service
    with intent strategies and configurable retry backoff.
  - `Juice.Messaging.Idempotency.*` — duplicate-detection service
    (EF, caching, or Redis backends).
  - `Juice.EventBus` — `IntegrationEventDispatcher`, subscription
    manager, and `IEventBus` publishing façade.
  - `Juice.EventBus.RabbitMQ` — RabbitMQ transport (`RabbitMQProducer`
    implements `ITransportPublisher`; `RabbitMQConsumerEngine` handles
    dead-letter exchange and retry headers).
  - External dependencies: `RabbitMQ.Client` v7+, `Polly` v8+ (retry),
    `Newtonsoft.Json` (serialization).
- **Multi-Tenancy**: Finbuckle.MultiTenant with file, EFCore, or gRPC
  tenant stores.
- **Mediation**: MediatR for command/query dispatching and pipeline
  behaviors (e.g., `TransactionBehavior`).
- **Distribution**: NuGet packages for .NET libraries; npm packages
  for JavaScript/TypeScript clients (e.g., `@juice-js/tenants`).
- **Documentation**: Hugo-based static site (this repository) hosted
  at juice-docs.creatorflow.io.

## Documentation Standards

- All documentation in this repository MUST be written as Hugo content
  files with proper front matter (title, date, weight).
- Code examples MUST be complete and compilable; use `hl_lines` for
  emphasis on key lines.
- Each core component page MUST include: overview, code examples,
  NuGet package links, and references to related components.
- Breaking changes and version history MUST be tracked in dedicated
  `breaking.md` and `whatsnew.md` files per component.
- The quickstart section MUST guide developers through the full path:
  Domain, Infrastructure, Contracts, API, Microservice, Monolithic.

## Governance

- This constitution supersedes all other development practices for
  the Juice documentation project.
- Amendments MUST be documented with a version bump, rationale, and
  migration plan if principles are added or removed.
- Version numbering follows semantic versioning:
  - MAJOR: Backward-incompatible principle removals or redefinitions.
  - MINOR: New principle or section added, or material guidance update.
  - PATCH: Clarifications, wording fixes, non-semantic refinements.
- All pull requests MUST verify compliance with these principles.
- The CLAUDE.md file (if present) provides runtime development
  guidance and MUST align with this constitution.

**Version**: 1.1.0 | **Ratified**: 2026-02-17 | **Last Amended**: 2026-02-18
