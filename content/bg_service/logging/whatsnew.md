---
title: "What's New in BgService Logging 7.0.3"
menuTitle: "What's new?"
date: 2023-10-20T08:21:52+07:00
draft: false
weight: 1
---

#### Quick access
- 
- [Name correction](#name-correction)
- [Configurable write interval](#configurable-write-interval)

#### Fixed unexpected error on stopping Windows service

Sometimes you may receive an error message when trying to stop a service on Windows because it cannot be stopped properly.

#### Name correction

Correcting directory patterns:
- *Unnamed service*: {Logging:File:Directory}/General/
- *Named service*: {Logging:File:Directory}/{ServiceDescription}/

Correcting file name patterns:
- *Default*: {yyyy_MM_dd-HHmm}.log
- *Job*: {JobId} - {JobDescription}.log
- *Job has state*: {JobId} - {JobDescription}_{JobState}.log
- *Job has re-run*: {JobId} - {JobDescription}_{JobState} ({increased number}).log

#### Configurable write interval

You can now configure the [BufferTime]({{<ref "./#Configure-log-builder" >}}) to decide how long to buffer before log entries are written to the file