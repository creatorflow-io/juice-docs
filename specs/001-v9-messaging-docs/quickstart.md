# Contributor Quickstart: v9 Messaging Documentation

**Phase**: 1 — Design
**Branch**: `001-v9-messaging-docs`
**Date**: 2026-02-18

This guide tells a documentation contributor exactly how to create and validate each
new page. Follow it in order; each step produces one concrete output.

---

## Prerequisites

- Hugo installed (`hugo version` — match the version in `go.mod` or site config)
- Checked out on branch `001-v9-messaging-docs`
- Read `research.md` and `contracts/page-schema.md` before writing any page

---

## Step 1 — Create the Archive Pages (lowest risk, do first)

Archive the three existing pages verbatim before modifying them.

### 1a. Archive EventBus

```bash
# From repo root
mkdir -p content/core/eventbus/v8.5.0
cp content/core/eventbus/_index.md content/core/eventbus/v8.5.0/_index.md
```

Edit `content/core/eventbus/v8.5.0/_index.md`:
- Change `title` front matter to `"Event bus (v8.5.0)"`
- Change `weight` to `999`
- Add the archive banner **immediately after the front matter closing delimiter**:

```markdown
> **⚠ Archived — v8.5.0**
> This page describes the pre-v9 Juice API.
> For current documentation see [Messaging]({{< ref "core/messaging/_index.md" >}}).
```

### 1b. Archive MediatR

```bash
mkdir -p content/core/mediator/v8.5.0
cp content/core/mediator/_index.md content/core/mediator/v8.5.0/_index.md
```

Edit `content/core/mediator/v8.5.0/_index.md`: same banner + title + weight=999.

### 1c. Archive Integration service

```bash
mkdir -p content/core/integration/v8.5.0
cp content/core/integration/_index.md content/core/integration/v8.5.0/_index.md
```

Edit `content/core/integration/v8.5.0/_index.md`: same banner + title + weight=999.

### Validate archive

```bash
hugo server
```

Navigate to:
- `/core/eventbus/v8.5.0/` — should show original content + banner
- `/core/mediator/v8.5.0/` — should show original content + banner
- `/core/integration/v8.5.0/` — should show original content + banner

---

## Step 2 — Update Existing Pages with v9 Banners

Add this notice to the **top** of each existing page (after front matter), pointing to
the new messaging section:

```markdown
> **ℹ v9 update**
> This feature has been updated in Juice v9.
> See [Messaging]({{< ref "core/messaging/_index.md" >}}) for the current API.
> The original v8.5.0 content is preserved [here]({{< ref "v8.5.0" >}}).
```

Files to update:
- `content/core/eventbus/_index.md`
- `content/core/mediator/_index.md`
- `content/core/integration/_index.md`

---

## Step 3 — Create the Messaging Section

### 3a. Create directory structure

```bash
mkdir -p content/core/messaging/internal
mkdir -p content/core/messaging/outbox
mkdir -p content/core/messaging/delivery
mkdir -p content/core/messaging/consumption
mkdir -p content/core/messaging/full-setup
```

### 3b. Create the section index (Overview + Internal Setup)

Create `content/core/messaging/_index.md` following the schema in
`contracts/page-schema.md` § US1.

Required sections (in order):
1. IMessage hierarchy — ASCII tree + publish-vs-consume table
2. Setup guide index — links + one-line descriptions
3. Internal Messaging Setup — full US1 content with code examples

### 3c. Create each setup guide

For each file below, follow the matching schema section in `contracts/page-schema.md`:

| File | Schema section |
|---|---|
| `content/core/messaging/outbox/_index.md` | US2 |
| `content/core/messaging/delivery/_index.md` | US3 |
| `content/core/messaging/consumption/_index.md` | US4 |
| `content/core/messaging/full-setup/_index.md` | US5 |

---

## Step 4 — Add Migration Table to Full Setup Page

Copy the migration table from `contracts/migration-table.md` into
`content/core/messaging/full-setup/_index.md` under a `## Migration from v8.5.0`
heading.

---

## Step 5 — Update Core Section Index

Edit `content/core/_index.md`: add a link to the new Messaging section alongside
the existing EventBus, MediatR, and Integration entries.

---

## Step 6 — Validate

### Build check

```bash
hugo build --minify
# Should complete with 0 errors, 0 warnings about broken refs
```

### Link check (each new URL must return 200)

- `/core/messaging/`
- `/core/messaging/outbox/`
- `/core/messaging/delivery/`
- `/core/messaging/consumption/`
- `/core/messaging/full-setup/`
- `/core/eventbus/v8.5.0/`
- `/core/mediator/v8.5.0/`
- `/core/integration/v8.5.0/`

### Content checklist per new page

- [ ] Front matter complete (title, date, weight, draft=false)
- [ ] NuGet package table present with links
- [ ] At least one complete code block with `hl_lines`
- [ ] No placeholder or stub code (`[...]`, `TODO`, ellipsis-only blocks)
- [ ] "See also" links at the bottom
- [ ] Archive pages have the v8.5.0 banner at the top

---

## Code Example Standards

All code blocks in new pages MUST follow this format:

````markdown
```csharp {linenos=false,hl_lines=[3,5],linenostart=1}
// Complete, compilable example
// No partial snippets with unexplained ...
services.AddMessaging(builder =>
{
    builder.AddIdempotencyRedis(opts => opts.Configuration = "localhost");
});
```
````

Use `hl_lines` to highlight the line(s) that are the key focus of the example.

---

## NuGet Link Format

Use this format for all NuGet package links (matches existing site convention):

```markdown
- [Juice.Messaging](https://www.nuget.org/packages/Juice.Messaging)
- [Juice.EventBus.RabbitMQ](https://www.nuget.org/packages/Juice.EventBus.RabbitMQ)
```
