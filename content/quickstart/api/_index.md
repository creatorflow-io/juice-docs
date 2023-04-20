---
title: "API"
date: 2023-04-13T10:50:32+07:00
draft: false
weight: 6
---

This project will be named `{namespace}.{feature}.Api`.

In this project, we will implement API that is defined in contracts and other integration services.
- Swagger
- SignalR
- gRPC implementation
- Integration event subscribing and handling
- Handle domain events to publish integration events

If you are building a monolithic application, then you can use the internal event bus instead of external.
You can also handle domain events from other domains directly instead of implementing integration events.