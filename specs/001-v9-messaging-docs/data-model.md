# Data Model: v9 Messaging Documentation Update

**Phase**: 1 — Design
**Branch**: `001-v9-messaging-docs`
**Date**: 2026-02-18

This is a documentation project. "Data model" describes the Hugo content page schema —
the front matter fields, section structure, and navigation weights required for each
new and updated page.

---

## Hugo Front Matter Schema

All pages in the Juice docs site use this front matter schema (TOML or YAML):

### New Setup Guide Pages (under `content/core/messaging/`)

```toml
---
title: "<Page title>"
date: "<ISO date>"
weight: <integer>          # controls sidebar order within section
draft: false
---
```

### Section Index Pages (`_index.md` with chapter = true)

```toml
+++
title = "<Section title>"
date = "<ISO date>"
weight = <integer>
chapter = true
pre = "<b>N. </b>"         # optional numbering prefix
+++
```

### Archive Pages (under `v8.5.0/` subdirectories)

```toml
---
title: "<Original title> (v8.5.0)"
date: "<original date>"
weight: 999                # high weight = appears last / de-prioritised
draft: false
---
```

Archive pages MUST NOT have `chapter = true`. They should include a banner at the top:

```markdown
> **⚠ You are viewing archived v8.5.0 documentation.**
> This content describes the pre-v9 API.
> See [EventBus v9]({{< ref "core/messaging/_index.md" >}}) for current documentation.
```

---

## Content Page Inventory

### New Pages

| File path | Title | Weight | Section |
|---|---|---|---|
| `content/core/messaging/_index.md` | Messaging | 7 | core (new section) |
| `content/core/messaging/internal/_index.md` | Internal Messaging Setup | 1 | messaging |
| `content/core/messaging/outbox/_index.md` | Outbox Setup | 2 | messaging |
| `content/core/messaging/delivery/_index.md` | Delivery Setup | 3 | messaging |
| `content/core/messaging/consumption/_index.md` | Consumption Setup | 4 | messaging |
| `content/core/messaging/full-setup/_index.md` | Full Pipeline Setup | 5 | messaging |

### Archive Pages (new)

| File path | Title | Weight | Archived from |
|---|---|---|---|
| `content/core/eventbus/v8.5.0/_index.md` | Event bus (v8.5.0) | 999 | `content/core/eventbus/_index.md` |
| `content/core/mediator/v8.5.0/_index.md` | MediatR (v8.5.0) | 999 | `content/core/mediator/_index.md` |
| `content/core/integration/v8.5.0/_index.md` | Integration service (v8.5.0) | 999 | `content/core/integration/_index.md` |

### Updated Pages (existing, modified)

| File path | Change |
|---|---|
| `content/core/eventbus/_index.md` | Add v9 banner + link to new messaging section + archive link |
| `content/core/mediator/_index.md` | Add v9 banner + link to new messaging section + archive link |
| `content/core/integration/_index.md` | Add v9 banner + link to new messaging section + archive link |
| `content/core/_index.md` | Add Messaging entry to section links |

---

## Required Sections Per Page Type

### Setup Guide Pages (US1–US5)

Each setup guide MUST contain these sections in order:

1. **Overview** (paragraph) — one-sentence summary of what this setup enables
2. **When to use this setup** (bullet list) — concrete conditions for choosing this guide
3. **Prerequisites** (bullet list or "None") — which other setup guides must be completed first
4. **NuGet packages** — table of required packages with links
5. **DI Registration** (code block) — complete, compilable `services.AddMessaging(...)` example
   with `hl_lines` highlighting the key line(s)
6. **Key concepts** (sub-sections per concept) — explanation of each type/interface used
7. **Code example** (code block) — a realistic usage example (handler, consumer, etc.)
8. **Health checks** (if applicable) — registration + monitoring guidance
9. **See also** (links) — cross-references to related setup guides

### Messaging Section Index (`content/core/messaging/_index.md`)

Must contain:
1. **IMessage hierarchy diagram** (ASCII or image reference)
2. **Publish-vs-Consume role table** (`IMessage` → publish, `IIntegrationEvent` → consume)
3. **Setup guide index** (links to all five guides with one-line descriptions)
4. **Internal Messaging Setup content** (merged here as the first/simplest guide)

### Archive Pages

Archive pages are verbatim copies of the source page with one addition at the top:
the v8.5.0 archive banner (see schema above). No content modifications allowed.

---

## Navigation Weight Plan for `content/core/`

Existing weights (inferred from current nav order) and new slot:

| Weight | Section |
|---|---|
| 1 | Overview |
| 2 | Logging |
| 3 | Options |
| 4 | Configuration |
| 5 | Swagger |
| **6** | **Messaging *(new section)*** |
| 7 | MediatR *(updated page)* |
| 8 | Event bus *(updated page)* |
| 9 | Integration service *(updated page)* |
| 10+ | Entity Framework, Multi-tenant, Modular, etc. |

---

## State Transitions (OutboxDelivery — for reference in Delivery guide)

The `OutboxDelivery` state machine that the Delivery guide must explain:

```
NotPublished
    │
    ▼ (DeliveryHostedService picks up, send-pending intent)
InProgress
    │
    ├──► Published  (broker ACK received)
    │
    ├──► Failed     (publish error; NextAttemptOn = backoff time)
    │       │
    │       └──► InProgress  (retry-failed intent, when NextAttemptOn ≤ Now)
    │
    └──► Failed     (InProgress for > 15 min; recover-timeout intent resets)

DeliveryState values: NotPublished=0, InProgress=1, Published=2, Failed=3, Skipped=4
```

Default retry backoff schedule (configurable via `AddDeliveryPolicies()`):

| Attempt | Delay |
|---------|-------|
| 1 | 10 seconds |
| 2 | 5 minutes |
| 3 | 10 minutes |
| 4 | 30 minutes |
| 5+ | 1 hour |
