---
title: "What's New in BgService Logging 7.0.3"
menuTitle: "What's new?"
date: 2023-10-20T08:21:52+07:00
draft: false
weight: 1
---

#### Quick access
- [Fixed unexpected error on stopping Windows service](#fixed-unexpected-error-on-stopping-windows-service)
- [Change scopes behavior](#change-scopes-behavior)
- [Write log scopes to file](#write-log-scopes-to-file)
- [Naming convention](#naming-convention)
- [Configurable write interval](#configurable-write-interval)
- [SignalR logger provider](#signalr-logger-provider)

### File logger provider
#### Fixed unexpected error on stopping Windows service

Sometimes you may receive an error message when trying to stop a service on Windows because it cannot be stopped properly.

#### Change scopes behavior

You can now use *JobState* scope inside *JobId*, so there is two ways to use *JobState*

- Original usage (yes, it still works):

```cs
string? state;
// {Logging:File:Directory}/{ServiceDescription}/{JobId} - {JobDescription}.log
using (_logger.BeginScope(new List<KeyValuePair<string, object>>
{
    new KeyValuePair<string, object>("JobId", JobId),
    new KeyValuePair<string, object>("JobDescription", JobDescription)
})){
    // job processing
    _logger.LogInformation("Begin invoke");
    state = await InvokeAsync();
}

// {Logging:File:Directory}/{ServiceDescription}/{JobId} - {JobDescription}_{JobState}.log
// {Logging:File:Directory}/{ServiceDescription}/{JobId} - {JobDescription}_{JobState} ({increased number}).log
using (_logger.BeginScope(new List<KeyValuePair<string, object>>
{
    new KeyValuePair<string, object>("JobId", JobId),
    new KeyValuePair<string, object>("JobDescription", JobDescription),
    new KeyValuePair<string, object>("JobState", state)
})){
    // job is completed with state
    _logger.LogInformation("Invoke result: {0}", state);
}

```
- Now it's more natural:

```cs
// {Logging:File:Directory}/{ServiceDescription}/{JobId} - {JobDescription}.log
using (_logger.BeginScope(new List<KeyValuePair<string, object>>
{
    new KeyValuePair<string, object>("JobId", JobId),
    new KeyValuePair<string, object>("JobDescription", JobDescription)
})){
    // job processing
    _logger.LogInformation("Begin invoke");
    var state = await InvokeAsync();
    // {Logging:File:Directory}/{ServiceDescription}/{JobId} - {JobDescription}_{JobState}.log
    // {Logging:File:Directory}/{ServiceDescription}/{JobId} - {JobDescription}_{JobState} ({increased number}).log
    using (_logger.BeginScope(new List<KeyValuePair<string, object>>
    {
        new KeyValuePair<string, object>("JobState", state)
    })){
        // job is completed with state
        _logger.LogInformation("Invoke result: {0}", state);
    }
}

```
#### Write log scopes to file

Any log scopes specified by the string will be written to the log file. See [Log scopes]({{<ref "./#write-log-scopes-to-file" >}})

#### Naming convention

Correcting directory patterns:
- *Unnamed service*: {Logging:File:Directory}/General/
- *Named service*: {Logging:File:Directory}/{ServiceDescription}/

Correcting file name patterns:
- *Default*: {yyyy_MM_dd-HHmm}.log
- *Job*: {JobId} - {JobDescription}.log
- *Job has state*: {JobId} - {JobDescription}_{JobState}.log
- *Job has re-run*: {JobId} - {JobDescription}_{JobState} ({increased number}).log

#### Configurable write interval

You can now configure the [BufferTime]({{<ref "./#configure-log-builder" >}}) to decide how long to buffer before log entries are written to the file

### SignalR logger provider

Supported logging to SignalR hub [here]({{<ref "./#signalr-logger-provider" >}}).