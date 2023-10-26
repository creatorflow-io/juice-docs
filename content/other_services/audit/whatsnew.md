---
title: "What's New in Audit 7.0.3"
menuTitle: "What's new?"
date: 2023-10-20T08:21:52+07:00
draft: false
weight: 1
---

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