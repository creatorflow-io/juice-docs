# Migration Table Contract: v8.5.0 → v9 Messaging

**Phase**: 1 — Design
**Branch**: `001-v9-messaging-docs`
**Date**: 2026-02-18

This is the canonical migration reference table to be included in the Full Pipeline
Setup guide (`content/core/messaging/full-setup/_index.md`). It MUST cover 100% of
the v8.5.0 public messaging interfaces (FR-008, SC-005).

---

## v8.5.0 → v9 Concept Mapping

| v8.5.0 Concept | Package (v8.5.0) | v9 Replacement | Package (v9) |
|---|---|---|---|
| `IEventBus` interface | `Juice.EventBus` | `IOutboxService<T>` for staging; `IEventBus` (CompositeEventPublisher) remains as a publish façade | `Juice.Messaging` / `Juice.EventBus` |
| `IEventBus.PublishAsync()` direct call | `Juice.EventBus` | `IOutboxService.AddEventAsync()` + `SaveEventsAsync()` inside transaction | `Juice.Messaging.Outbox.EF` |
| `IntegrationEventLog` table (single-table) | `Juice.EventBus.IntegrationEventLog.EF` | `OutboxEvent` + `OutboxDelivery` two-table model | `Juice.Messaging.Outbox.EF` |
| `IIntegrationEventLogService` | `Juice.EventBus.IntegrationEventLog.EF` | `IOutboxService<T>` | `Juice.Messaging` |
| `IIntegrationEventLogService.SaveEventAsync()` | — | `IOutboxService.AddEventAsync()` + `SaveEventsAsync(transactionId)` | — |
| `IIntegrationEventLogService.MarkEventAsInProgressAsync()` | — | Handled automatically by `DeliveryHostedService` | — |
| `IIntegrationEventLogService.MarkEventAsPublishedAsync()` | — | Handled automatically by `DeliveryHostedService` | — |
| `IIntegrationEventLogService.MarkEventAsFailedAsync()` | — | Handled automatically by `DeliveryHostedService` | — |
| `IIntegrationEventService` | `Juice.Integrations` | `IOutboxService<T>` | `Juice.Messaging` |
| `IIntegrationEventService.AddAndSaveEventAsync()` | — | `IOutboxService.AddEventAsync()` | — |
| `IIntegrationEventService.PublishEventsThroughEventBusAsync()` | — | Removed — `DeliveryHostedService` publishes automatically | — |
| `TransactionBehavior` (v8, in `Juice.Integrations.MediatR`) | `Juice.Integrations` | `TransactionBehavior<T,R,TContext>` (abstract, in `Juice.MediatR.Behaviors`) | `Juice.MediatR.Behaviors` |
| `services.RegisterRabbitMQEventBus()` | `Juice.EventBus.RabbitMQ` | `builder.AddMessaging().AddDelivery(d => d.EventBus.AddRabbitMQ(...))` | `Juice.Messaging` + `Juice.EventBus.RabbitMQ` |
| `IRequestManager` (command deduplication) | `Juice.MediatR` | `IIdempotentRequest` + `IdempotencyBehavior` | `Juice.MediatR.Contracts` + `Juice.MediatR.Behaviors` |
| `IIdentifiedCommand<T>` wrapper | `Juice.MediatR` | Removed — mark command directly with `IIdempotentRequest` | `Juice.MediatR.Contracts` |
| `requestManager.ExistAsync(id)` in handler | — | Removed — `IdempotencyBehavior` handles it automatically | — |
| `IntegrationEvent` record (v8) | `Juice.EventBus` | `IntegrationEvent` record still exists; now extends `MessageBase` (carries `MessageId`, `CreatedAt`, `TenantId`) | `Juice.EventBus.Contracts` |
| `IIntegrationEventHandler<T>.HandleAsync()` | `Juice.EventBus` | Unchanged | `Juice.EventBus` |
| RabbitMQ `SubscriptionClientName` config | `Juice.EventBus.RabbitMQ` | `AddConsumer(key, queueName, connectionName)` | `Juice.EventBus.RabbitMQ` |

---

## What Is Unchanged

These v8.5.0 concepts carry forward to v9 without replacement:

- `IntegrationEvent` abstract record (still the base for integration events)
- `IIntegrationEventHandler<T>` interface (handlers unchanged)
- `IIntegrationEvent` interface (still required on the consuming side)
- `RabbitMQ.Client` as the broker transport
- `Finbuckle.MultiTenant` for tenant resolution
- MediatR `IRequest<T>`, `INotification` patterns (MediatR itself unchanged)
- `IAggregrateRoot<TNotification>` domain pattern
- `IUnitOfWork` interface
