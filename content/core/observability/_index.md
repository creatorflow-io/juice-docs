---
title: "Observability (OpenTelemetry)"
date: "2026-10-29"
weight: 10
draft: false
---

## Overview

Juice records **distributed traces** and **framework metrics** with the standard .NET APIs
(`System.Diagnostics.ActivitySource` and `Meter`). The core packages do not reference OpenTelemetry:
the application host decides whether to collect them, which exporters to use (OTLP, Jaeger, Azure Monitor...)
and how to sample.

One request produces one connected trace, across processes:

```
TraceId = 4bf92f3577b34da6a3ce929d0e0e4736

POST /orders                         (ASP.NET Core server span, tagged with the Juice context)
 └─ mediator PlaceOrder              Juice.OpenTelemetry
     └─ create OrderPlaced           outbox save: the context is stored with the message
         ├─ deliver rabbitmq         one per delivery attempt, possibly hours later
         │   └─ send orders          RabbitMQ / NATS producer
         └─ process orders-queue     consumer, in another service
             └─ handle OrderPlacedHandler
```

The trace context travels in the standard W3C `traceparent` / `tracestate` message headers.
It is separate from the [MessageContext]({{< ref "core/messaging/message-context/_index.md" >}}):
`x-correlation-id` and `x-causation-id` keep their meaning and are copied onto the spans as tags.

| | Purpose |
|---|---|
| Trace (`Activity`) | Distributed execution flow: what ran where, and how long |
| Metrics (`Meter`) | Aggregated framework measurements |
| `MessageContext` | Juice business correlation (correlation, causation, execution, source) |
| `ITimeTracker` | Detailed local timing inside one execution |
| Audit | Business and compliance history |

---

## Setup

Install the integration package (net8.0 and later) next to the OpenTelemetry SDK and the exporter you use:

```
dotnet add package Juice.OpenTelemetry
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
```

```cs
// Program.cs
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource.AddService("orders-api"))
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()   // the HTTP server span
        .AddJuiceInstrumentation()        // mediator, outbox, messaging spans
        .AddOtlpExporter())
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddJuiceInstrumentation()        // juice.* metrics
        .AddJuiceDeliveryMetrics()        // optional: delivery.metrics, infra.rabbitmq.metrics, infra.nats.metrics
        .AddOtlpExporter())
    .WithLogging(logging => logging
        .AddJuiceInstrumentation()        // Juice context, payload redaction (10.8); before the exporter
        .AddOtlpExporter());

// mediator spans and metrics
builder.Services.AddJuiceOpenTelemetry();

builder.Services.AddMediatR(m => m.RegisterServicesFromAssemblyContaining<Program>());
builder.Services.AddMessaging(/* transports, outbox... */);

var app = builder.Build();

// initializes MessageContext and tags the HTTP server span with it
app.UseMessageContext("orders-api");

app.MapPost("/orders", async (PlaceOrder command, IMediator mediator) =>
{
    await mediator.Send(command);
    return Results.Accepted();
});

app.Run();
```

Every service in the chain configures the same way. The consumer side needs no extra code: the RabbitMQ and NATS
consumers, the `local` and `local-channel` transports read `traceparent` from the message and continue the trace.

`AddJuiceInstrumentation()` only adds the `"Juice"` source and meter (`JuiceActivitySource.Name`, `JuiceMeter.Name`),
so a host can also call `AddSource("Juice")` / `AddMeter("Juice")` directly without the package. It does **not**
collect the [delivery and broker meters](#delivery-and-broker-meters); add `AddJuiceDeliveryMetrics()` (10.8) for those,
or `AddMeter("delivery.metrics", "infra.rabbitmq.metrics", "infra.nats.metrics")` without the package.

### Options

```cs
builder.Services.AddJuiceOpenTelemetry(options =>
{
    options.EnableMediatorInstrumentation = true;      // default
    options.EnableNotificationInstrumentation = true;  // IMediator.Publish, default
    options.Filter = type => type != typeof(GetDashboardQuery); // skip a chatty request
});
```

Exporter endpoints and sampling are **not** Juice options: configure them on the OpenTelemetry SDK
(`AlwaysOn`, `ParentBased`, `TraceIdRatioBased`, tail sampling in a collector...).

---

## Spans

All spans are on the `Juice` source. Payloads, request values and secrets are never recorded.

| Span | Kind | Where |
|---|---|---|
| `mediator {RequestType}` | Internal | every mediator request or notification (`Juice.OpenTelemetry`); outermost behavior, covers idempotency and the transaction |
| `create {MessageType}` | Producer | the outbox saves a message; its context is stored in the outbox row |
| `deliver {publisherKey}` | Internal | one outbox delivery attempt (retry count, outcome) |
| `send {exchange \| subject \| MessageType}` | Producer | RabbitMQ / NATS producer, `local-channel` enqueue |
| `process {queue \| destination \| MessageType}` | Consumer | RabbitMQ / NATS consumer, `local`, `local-channel` dispatch (dispatch result) |
| `handle {Handler}` | Internal | one integration event handler |

Juice does not create HTTP or database spans: use the ASP.NET Core and EF Core / SqlClient / Npgsql instrumentations.
`UseMessageContext` (and `[MessageContext]`) tag the ASP.NET Core server span instead.

Common tags: `juice.correlation_id`, `juice.causation_id`, `juice.execution_id`, `juice.source`, `juice.message_type`,
`messaging.system`, `messaging.destination.name`, `messaging.message.id`, `juice.dispatch.result`. A failure sets the
span status to `Error` with an `exception` event.

---

## Metrics

All instruments are on the `Juice` meter. Dimensions are bounded: message, tenant and correlation ids are never used.

| Instrument | Type | Unit | Dimensions |
|---|---|---|---|
| `juice.mediator.requests` | Counter | `{request}` | `juice.operation`, `juice.mediator.kind`, `error.type` |
| `juice.mediator.errors` | Counter | `{error}` | `juice.operation`, `juice.mediator.kind`, `error.type` |
| `juice.mediator.request.duration` | Histogram | `s` | `juice.operation`, `juice.mediator.kind`, `error.type` |
| `juice.messaging.published` | Counter | `{message}` | `messaging.system`, `messaging.destination.name`, `juice.message_type` |
| `juice.messaging.consumed` | Counter | `{message}` | `messaging.system`, `messaging.consumer.group.name`, `juice.message_type`, `juice.dispatch.result` |
| `juice.messaging.process.duration` | Histogram | `s` | as `consumed` (or `error.type`) |
| `juice.messaging.errors` | Counter | `{error}` | `messaging.system`, `messaging.operation.name`, `juice.message_type`, `error.type` |
| `juice.outbox.published` | Counter | `{message}` | `juice.publisher`, `juice.message_type` |
| `juice.outbox.errors` | Counter | `{error}` | `juice.publisher`, `juice.message_type`, `error.type` |

### Delivery and broker meters

The earlier meters are still recorded for existing dashboards. They are collected only when the host adds them:
`AddJuiceDeliveryMetrics()` (10.8) adds all three. A meter whose package is not installed records nothing.

| Meter | Instrument | Type | Unit | Dimensions | Recorded by |
|---|---|---|---|---|---|
| `delivery.metrics` | `delivery_attempt_total` | Counter | | `publisher` | outbox delivery, `local`, `local-channel` |
| `delivery.metrics` | `delivery_success_total` | Counter | | `publisher` | outbox delivery, `local`, `local-channel` |
| `delivery.metrics` | `delivery_failure_total` | Counter | | `publisher`, `error_type` | outbox delivery, `local`, `local-channel` |
| `delivery.metrics` | `delivery_latency_ms` | Histogram | `ms` | `publisher` | outbox delivery, `local`, `local-channel` |
| `delivery.metrics` | `channel_dropped_total` | Counter | | `publisher`, `message_type` | `local-channel` (channel full) |
| `delivery.metrics` | `local_delivery_phase1_fallback_total` | Counter | | `publisher` | `local` |
| `delivery.metrics` | `channel_queue_depth` | Gauge | | | `local-channel` |
| `delivery.metrics` | `identified_commands_received` / `_duplicated` / `_processed` / `_failed` | Counter | | `command` | idempotency behavior |
| `delivery.metrics` | `identified_commands_latency_ms` | Histogram | `ms` | `command` | idempotency behavior |
| `infra.rabbitmq.metrics` | `rabbitmq_channel_error_total` | Counter | | `channel` | RabbitMQ |
| `infra.nats.metrics` | `jetstream_publish_error_total` | Counter | | `destination` (before 10.8: `subject`) | NATS JetStream producer |
| `infra.nats.metrics` | `jetstream_consume_retry_total` | Counter | | `destination` | NATS JetStream consumer |
| `infra.nats.metrics` | `jetstream_consume_parked_total` | Counter | | `destination` | NATS JetStream consumer |

Outbox and local delivery record the same four `delivery_*` instruments (`Juice.Messaging.Diagnostics.DeliveryMetrics`),
so they export as one series per instrument, told apart by `publisher`.

---

## Logs

Juice writes logs with `ILogger`; OpenTelemetry logging exports them (10.8). `AddJuiceInstrumentation()` on the logging
builder adds a processor that runs before the exporters, so add it first:

```cs
builder.Services.AddOpenTelemetry()
    .WithLogging(logging => logging
        .AddJuiceInstrumentation(options =>
        {
            options.IncludePayloads = false;   // default
        })
        .AddOtlpExporter());
```

| Exported with each log | Source |
|---|---|
| Trace id, span id | the OpenTelemetry SDK, from the current span |
| `juice.correlation_id`, `juice.causation_id`, `juice.execution_id`, `juice.source` | `MessageContext`, same names as the span tags |
| Message template and its scalar values | the log call |

**Payloads are not exported.** Some Juice logs carry a message payload or a whole command (the outbox debug log,
the transaction error log). For categories starting with `Juice.`, the processor removes destructured (`{@Command}`)
and `Payload` attributes, and the formatted message when it held one; the body keeps the message template. Set
`IncludePayloads = true` to export them, or add names to `PayloadAttributes`. Your application's own logs are left as
written: keep payloads, secrets and personal data out of them, and remember that exception messages are exported as
thrown.

**Exporter outages.** The OTLP exporter batches logs on a background thread: logging never waits for it and never
fails because of it. When the backend is down or slow, records beyond the batch queue (`MaxQueueSize`, 2048 by
default) are dropped, so memory stays bounded. Tune the queue with
`AddOtlpExporter((exporter, processor) => processor.BatchExportProcessorOptions.MaxQueueSize = 4096)`.

`IncludeScopes` and `IncludeFormattedMessage` are OpenTelemetry options:
`WithLogging(logging => ..., options => options.IncludeScopes = true)`.

---

## Testing

`Juice.XUnit.ActivityCollector` records Juice spans with a plain `ActivityListener`, without the OpenTelemetry SDK.
Listeners are process wide, so read the spans of your own trace:

```cs
using var collector = new ActivityCollector();
using var root = collector.StartRoot();

await messageService.PublishAsync(new OrderPlaced());

var process = await collector.WaitForAsync(root.TraceId, "process OrderPlaced");
process.GetTagItem("juice.dispatch.result").Should().Be("success");
```

---

## Notes

- Without a listener no activity is created; instruments are no-ops while nothing collects them.
- Telemetry never fails a business operation.
- `Juice.OpenTelemetry` targets net8.0+ like OpenTelemetry .NET. The core tracing and metrics also work on net6.0
  with `AddSource("Juice")` / `AddMeter("Juice")`.
- Messages saved by an older version have no `traceparent`: their deliveries start a new trace.

The library can be accessed via Nuget:
- [Juice.OpenTelemetry](https://www.nuget.org/packages/Juice.OpenTelemetry)
