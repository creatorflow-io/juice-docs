---
title: "Breaking changes"
menuTitle: "Breaking changes"
date: 2023-04-03T20:23:30+07:00
draft: false
weight: 2
---

#### Quick access
- [ExternalScopeLoggerProvider](#use-externalscopeloggerprovider-to-share-scopes)

#### Use ExternalScopeLoggerProvider to share scopes (since 7.0.3)
The abstract class *LoggerProvider* should no longer inherit the interface *ISupportExternalScope* but use the *ExternalScopeLoggerProvider* instead.
