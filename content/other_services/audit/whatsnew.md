---
title: "What's New in Audit 10.3.0"
menuTitle: "What's new?"
date: 2026-10-26T09:00:00+07:00
draft: false
weight: 1
---

### Audit 10.3.0

#### Multi-tenant audit records
*AccessLog* and *DataAudit* now carry a *TenantId*, and *AuditDbContext* inherits Juice's *MultiTenantDbContext*. Both entities are configured as `IsMultiTenant(SharingType.Global)`:
- A **tenant** reads and writes only its own audit records.
- The **root** context (no tenant resolved: the gRPC audit server, an admin view, or a non multi-tenant app) reads and writes the records of all tenants.
- Records saved without a tenant are stamped with the current tenant (`TenantNotSetMode.Overwrite`); changing another tenant's records throws (`TenantMismatchMode.Throw`).

How the tenant is captured:
- *AuditMiddleware* reads it from *ITenantAccessor* when the request starts, and again after the pipeline if it was resolved later.
- *DataAudit* entries take it from *DataEvent.TenantId*, falling back to the request's tenant.
- The gRPC contract has new optional `tenantId` fields on *AccessLogMessage* and *DataAuditMessage*, so a central audit server stores the tenant reported by each client. Older clients keep working; their records have no tenant.

Upgrade notes:
- Apply the new **AddTenantId** migration (SqlServer and PostgreSQL). It adds a nullable *TenantId* column and a *(TenantId, DateTime)* index to both tables.
- *Juice.Audit.EF* now depends on *Juice.EF.MultiTenant* (Finbuckle).
- Code that queries *AuditDbContext* while a tenant is resolved now gets only that tenant's records.
- The *DataAudit* constructor has a new optional *tenantId* parameter: rebuild code compiled against an earlier version.

### Audit 7.0.3

#### Quick access
- [Fixed file upload issue](#fixed-file-upload-issue)
- [Fixed middleware registration](#fixed-middleware-registration)
- [Added more AuditFilterOptions methods](#added-more-auditfilteroptions-methods)
- [Handle aborted request](#handle-aborted-request)

#### Fixed file upload issue

In the case when you are using multipart form data to upload files, the file section stream must be processed for upload first. Otherwise, it can cause the error [Unexpected end of Stream...](https://github.com/dotnet/aspnetcore/issues/18087). So we have to move the collection of request form data after other middleware is processed to avoid the issue.

#### Fixed middleware registration
Fixed an error that occurred when you registered AuditMiddleware with the *IApplicationBuilder.UseAudit* extension.

#### Added more AuditFilterOptions methods
- *Merge*: merge new filter entries if they do not already exist
- *IsExists*: check if the filter entry already exists

#### Handle aborted request
- Added *AuditFilterOptions.RequestAbortedStatusCode* option to specify the status code when the request is aborted. Default value is **408 Request Timeout**
- Handle aborted request to add *AccessLog*