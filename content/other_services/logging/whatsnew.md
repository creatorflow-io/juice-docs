---
title: "What's New in Logging 7.0.3"
menuTitle: "What's new?"
date: 2023-10-20T08:21:52+07:00
draft: false
weight: 1
---

#### We moved logging extensions from BackgroundService project to new Logging project to maintain it in general.

#### Quick access
- [Fixed unexpected error on stopping Windows service](#fixed-unexpected-error-on-stopping-windows-service)
- [Change scopes behavior](#change-scopes-behavior)
- [Write log scopes to file](#write-log-scopes-to-file)
- [Naming convention](#naming-convention)
- [Configurable write interval](#configurable-write-interval)
- [SignalR logger provider](#signalr-logger-provider)
- [DB logger provider](#db-logger-provider)

### File logger provider
#### Fixed unexpected error on stopping Windows service

Sometimes you may receive an error message when trying to stop a service on Windows because it cannot be stopped properly.

#### Change scopes behavior

You can now use *OperationState* scope inside *TraceId*, so there is two ways to use *OperationState*

- Original usage (yes, it still works):

```cs
string? state;
// {Logging:File:Directory}/{ServiceDescription}/{TraceId} - {Operation}.log
using (_logger.BeginScope(new List<KeyValuePair<string, object>>
{
    new KeyValuePair<string, object>("TraceId", TraceId),
    new KeyValuePair<string, object>("Operation", Operation)
})){
    // job processing
    _logger.LogInformation("Begin invoke");
    state = await InvokeAsync();
}

// {Logging:File:Directory}/{ServiceDescription}/{TraceId} - {Operation}_{OperationState}.log
// {Logging:File:Directory}/{ServiceDescription}/{TraceId} - {Operation}_{OperationState} ({increased number}).log
using (_logger.BeginScope(new List<KeyValuePair<string, object>>
{
    new KeyValuePair<string, object>("TraceId", TraceId),
    new KeyValuePair<string, object>("Operation", Operation),
    new KeyValuePair<string, object>("OperationState", state)
})){
    // job is completed with state
    _logger.LogInformation("Invoke result: {0}", state);
}

// scopes ["Task", "Success"]    
using (_logger.BeginScope("Task"))
using (_logger.BeginScope("Success"))
{
    _logger.LogInformation("Invoked");
}

```
- Now it's more natural:

```cs
// {Logging:File:Directory}/{ServiceDescription}/{TraceId} - {Operation}.log
using (_logger.BeginScope(new List<KeyValuePair<string, object>>
{
    new KeyValuePair<string, object>("TraceId", TraceId),
    new KeyValuePair<string, object>("Operation", Operation)
})){
    // job processing
    _logger.LogInformation("Begin invoke");
    var state = await InvokeAsync();
    // {Logging:File:Directory}/{ServiceDescription}/{TraceId} - {Operation}_{OperationState}.log
    // {Logging:File:Directory}/{ServiceDescription}/{TraceId} - {Operation}_{OperationState} ({increased number}).log
    using (_logger.BeginScope(new List<KeyValuePair<string, object>>
    {
        new KeyValuePair<string, object>("OperationState", state)
    })){
        // job is completed with state
        _logger.LogInformation("Invoke result: {0}", state);
    }
}

// scopes ["Task", "Success"]
using (_logger.BeginScope(new string[]{"Task", "Success"}))
{
    _logger.LogInformation("Invoked");
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
- *Job*: {TraceId} - {Operation}.log
- *Job has state*: {TraceId} - {Operation}_{OperationState}.log
- *Job has re-run*: {TraceId} - {Operation}_{OperationState} ({increased number}).log

#### Configurable write interval

You can now configure the [BufferTime]({{<ref "./#configure-log-builder" >}}) to decide how long to buffer before log entries are written to the file

### SignalR logger provider

Supported logging to SignalR hub [here]({{<ref "./#signalr-logger-provider" >}}).

### Db logger provider

Supported logging to SignalR hub [here]({{<ref "./#db-logger-provider" >}}).