---
title: "Tenant pattern (decision record)"
date: 2026-11-24T10:00:00+07:00
draft: false
weight: 20
---

**Status:** Accepted, 24 November 2026. **Applies to:** the Juice framework (`Juice.Extensions.MultiTenant`, `Juice.EF.MultiTenant`),
juice-tenants (`services/multitenant`), and every service built on Juice (Juice.Observability, Juice.Notifications, the services under
`services/`).

## Context

Juice services identify tenants in three different ways today.

| | How the request's tenant is found | Key used | Checks the tenant exists and is active |
|---|---|---|---|
| Juice framework and juice-tenants | Finbuckle strategies: base path, header, route, per-tenant OIDC login | The **Identifier** (the tenant's name in URLs). The gRPC `__tenant__` header carries the Identifier too | Yes, through the tenant store |
| Juice.Observability | Each deployment serves one fixed tenant (`JUICE_PLATFORM_TENANT_ID`); the token's `platform_tenant` claim must equal it | The Id | Its own `platform_tenants` table, provisioned at install |
| Juice.Notifications | The `/tenants/{tenantId}` route, which must match the token's `platform_tenant` claim (`TenantAccess`) | The Id (a GUID) | No; only "is this application registered for the tenant" |

The differences cost us:

- **Two keys.** The framework passes Identifiers between services, while Observability and Notifications store Ids. A service talking to
  another must know which one it is given. Renaming a tenant's Identifier breaks anything that stored it.
- **Ids aren't always GUIDs.** juice-tenants creates a tenant without an Id by copying its Identifier (`Id = request.Id ?? request.Identifier`,
  `CreateTenantCommandHandler`), so an "id" may be a name.
- **Tenant status is ignored by the services.** A suspended tenant keeps receiving notifications, because nothing in Notifications asks.
- **Each service writes its own check.** The claim name, the admin role and the "may this caller act in this tenant" rule are coded again
  in every service, each with its own tests and its own local-stack seeding.

## Decision

### 1. The Id is the tenant's identity; the Identifier is only for display

- The **Id** is a GUID issued by juice-tenants when the tenant is created. It is required and never changes.
- Everything that stores or carries a tenant uses the Id: database columns, integration events and message envelopes, tokens,
  service-to-service calls (REST and gRPC), and logs and telemetry.
- The **Identifier** (`acme` in `acme.example.com` or `/acme/...`) is for browser URLs and the UI only. It is turned into the Id once, where a
  browser request enters an app, and never sent further.
- **The Id never depends on a name** (owner, 24 November 2026): not a hash of the tenant's name, and not a name-based UUID. A tenant known
  only by a name from before this pattern (a Juice.Observability deployment's `JUICE_PLATFORM_TENANT_ID`, such as `local-dev`) gets a
  random Id like any other, and keeps its old name only where it was already stored.

### 2. The token says which tenant a caller belongs to

- The claim is **`platform_tenant`**, holding the tenant **Id**.
  - **People:** issued from the Keycloak user attribute `platform_tenant` by a user-attribute mapper. Identity directories read the same
    attribute to list a tenant's people.
  - **Service clients:** issued by a hard-coded claim mapper on the client.
- Roles come from the `roles` claim:
  - A **tenant-admin role** per service (for example `notifications-tenant-admin`) lets its holder administer the tenant its claim names.
  - A **platform-admin role** lets its holder act in any tenant, naming the tenant explicitly (section 3).
- One tenant per person for now. If a person later needs several, only the shared reader of this claim changes (section 4), not the
  services.

### 3. Each kind of caller supplies the tenant one way

| Caller | Where the tenant comes from | Rule |
|---|---|---|
| A browser app | The Identifier in its URL (base path or host), through the tenant store | The app uses the Id from then on, including in API calls |
| A person or tenant admin calling an API | The `platform_tenant` claim | A tenant in the route (`/tenants/{tenantId}`) is allowed but must equal the claim |
| A platform admin or service account | The `/tenants/{tenantId}` route, or the `X-Tenant-Id` header on gRPC | Always explicit; never inferred |
| A message or background job | The tenant Id in the message envelope | Run in `IScopedTenantResolver.Resolve(id)` |
| A single-tenant deployment (Observability) | The deployment's fixed tenant Id | The claim must equal it |

Every resolved tenant is then checked against the tenant store:

- An unknown tenant gets **404**. A caller who may not see it gets the same 403 as for another tenant, so responses don't reveal which
  tenants exist.
- A tenant still starting (`Initializing`, `Initialized`, `PendingToActive`) gets **503** with the problem code `tenant_starting`: try
  again later.
- Any other tenant that isn't `Active` (`Inactive`, `Suspended`, `Abandoned`, and the ones never approved) gets **403** with the problem
  code `tenant_inactive`.
- Background work for a tenant that isn't `Active` is held, not dropped. It waits until the tenant is `Active` again (expiring as it would
  anyway), and the wait is logged.

### 4. One shared package

The pattern lives in one package, which every service sets up with one call (for example `AddJuiceTenancy()`), instead of each service
writing it again. It is `Juice.MultiTenant.AspNetCore` in the Juice framework (owner, 24 November 2026), released with it;
juice-tenants' gRPC store plugs into it. It holds:

- **Names:** the claim, header and route parameter (`platform_tenant`, `X-Tenant-Id`, `tenantId`) as constants, each overridable by
  options.
- **Resolution:** the strategies in the order of section 3.
- **Authorization:** `ITenantAccess`, the one "may this caller act in this tenant" check (claim equals tenant, the tenant-admin role, the
  platform-admin role), with authorization policies built on it.
- **The tenant store:** juice-tenants over gRPC, cached for a short time (about a minute) and refreshed on the lifecycle events of
  section 6. It is keyed by Id, with a by-Identifier lookup for browser apps.
- **The message envelope:** a tenant Id header written by the outbox and read by the consumers.

### 5. Data

- One database per service, with a `TenantId` column (the Id) on every tenant-owned table. It is part of that table's keys and foreign keys,
  so a row can't point at another tenant's row.
- An EF query filter by tenant (`Juice.EF.MultiTenant`). A query across tenants turns the filter off explicitly and is reserved for platform
  operations.
- No database per tenant unless a customer needs that isolation. Even then, it is a setting in the tenant store (a per-tenant connection
  string), not a code change.

### 6. The tenant lifecycle is published as events

- juice-tenants publishes integration events, keyed by Id, when a tenant is created, activated, made inactive, suspended or deleted.
- Each service decides what an event means for it. For example, Notifications stops sending, and Observability stops alerting. A service
  keeps no list of tenants of its own beyond what its rows need.

## Consequences

**Testing**

- The test helper (`Juice.MultiTenant.TestHelper`) gives an in-memory store with two fixed tenants, A and B (fixed GUIDs, one active, one
  suspended), and a token maker that sets `platform_tenant` and the roles.
- Every service runs the same set of tenant tests, shipped with the package:
  - A's token cannot read or change B's data.
  - A route tenant that differs from the claim is refused.
  - A suspended tenant gets 403, and an unknown tenant gets 404.
  - A platform admin must name the tenant.
  - A message carrying B's Id runs in B.

**Deployment**

- A new tenant is created in juice-tenants (its Id is a GUID) and given to its people as the Keycloak attribute. No service is configured
  for it.
- The local stacks seed the same fixed tenant Ids everywhere (Notifications uses `0193f0aa-0000-7000-8000-0000000000a1` today), so a person
  set up once works in every service.

**Maintenance**

- One package and one set of names. Changing the claim, allowing several tenants per person or changing the store happens in one place.
- The cost is a dependency on that package and on juice-tenants being reachable. The store's cache, plus refresh on lifecycle events, keeps
  short outages of juice-tenants from failing requests.

## Changes, in order

1. **juice-tenants:** require a GUID Id when a tenant is created (or generate one); stop copying the Identifier. Every deployed tenant
   already has a GUID Id (owner, 24 November 2026), so no data migration is needed. The lifecycle integration events of section 6
   already exist (`TenantCreatedIntegrationEvent`, the status-changed and deleted events, sent through the outbox with the Id and the
   Identifier); document them as the contract services consume.
2. **Juice framework:**
   - Build the shared package of section 4.
   - Move the gRPC `__tenant__` header and the default resolution from the Identifier to the Id (`X-Tenant-Id`). Keep reading the old
     header for one major version.
   - Keep the Identifier only for browser routing (base path, host, per-tenant OIDC login).
   - Extend `Juice.MultiTenant.TestHelper` with the fixed tenants, the token maker and the shared tenant tests.
3. **Juice.Notifications:**
   - Replace `TenantAccess` with the shared `ITenantAccess`. Its rules already match section 3.
   - Add the tenant-status check (403 `tenant_inactive`), and skip deliveries for inactive tenants.
   - React to the lifecycle events.
4. **Juice.Observability:** keep the single-tenant deployment, using the shared package's fixed-tenant mode for the claim check and the
   tenant tests.
   - **Its tenant:** until now a free-form name (`JUICE_PLATFORM_TENANT_ID`, such as `local-dev`), stamped on all its telemetry. It is
     replaced by the tenant's Id (owner, 24 November 2026).
   - **The data:** the API renames the tenant at start, and a script rewrites the telemetry stamped with the old name.
   - **Enrolled agents:** they are told the new tenant, signed by a command key they trust.
5. **The other services under `services/`:** adopt the package as each is next released, Id-keyed from the start.
6. **`@juice-js` libraries:** each library takes the tenant Id from the token's claim by default (`@juice-js/notifications` already does).
   `@juice-js/tenant` keeps resolving the URL's Identifier for the app and exposes the matching Id.

## Not decided here

- **Several tenants per person:** a claim holding a list, and how the person picks one.
- **Per-tenant identity providers:** a realm per tenant, which the per-tenant OIDC conventions allow.
- **Per-tenant databases:** whether any customer needs them, and at what tier.
