---
title: "Breaking changes"
linktitle: "Breaking changes"
date: 2025-06-24T20:23:30+07:00
draft: false
weight: 2
---

#### Change RabbitMQ default behavior on handling failure
From version 8.4.x, we added `AckOnProcessed` option in the `RabbitMQOptions` and change default behavior when `IIntegrationEventHandler` service found for incoming integration event but it throws an exception.
- **Before**: the `Nack` message will be sent back to the broker, so the integration event will be processed repeatedly util it's done.
- **From 8.4.x**: the `Ack` message will be sent back to the broker, so the integration event will be processed once, even if it fails.

#### Tenant intended when publish message
`IEventBus` now added `tenantId` parameter into `PublishAsync` method to supports multi-tenant.