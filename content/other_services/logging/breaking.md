---
title: "Breaking changes in Logging 7.0.3"
menuTitle: "Breaking changes"
date: 2023-04-03T20:23:30+07:00
draft: false
weight: 2
---


#### Quick access
- [Moved file logging provider](#moved-file-logging-provider)
- [Default service name](#default-service-name)

#### Moved file logging provider
The FileLoggerProvider is not included in *Juice.BgService.ServiceBase* but moved to new *Juice.Extensions.Logging.File* package.

#### Default service name
The default service name has been changed from *Default* to *General*, but it can be configured using the *GeneralName* option.

#### Change the scope property name
- *JobId* changed to *TraceId*
- *JobDescription* changed to *Operation*
- *JobState* changed to *OprationState*