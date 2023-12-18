---
title: "Logging"
date: 2023-04-03T21:02:02+07:00
draft: false
weight: 1
version: latest
---
{{< versions doc=logging >}}

#### Quick access
- [File logger](#file-logger-provider)
- [SignalR logger](#signalr-logger-provider)
- [Db logger](#db-logger-provider)

### File logger provider
#### Purposes

- Separate log folder for services
- Limit log file size
- Limit the number of log files
- Separate log file for job

#### Usage
1. ##### Configure log builder
Call *AddFileLogger()* on the logging builder to add a customized file logger provider.
```cs
using Juice.Extensions.Logging;
...

var builder = WebApplication.CreateBuilder(args);

builder.Logging.AddFileLogger(builder.Configuration.GetSection("Logging:File"));

```

*Logging:File* configuration section may be present:
- *Directory*: where to store logging files
- *RetainPolicyFileCount*: default value is **50**
- *MaxFileSize*: default value is **5 * 1024 * 1024**;
- *ForkJobLog*: default value is **true**
- *BufferTime*: the log entries will be written to the file after the interval (5 seconds by default)
- *GeneralName*: the default name of sub-directory if there is no *ServiceDescription* provided. It's optional and the default value is *General*

```json
// appsettings.json
{
  "Logging": {    
    "File": {
      "Directory": "C:\\Workspace\\Services\\logs",
      "BufferTime": "00:00:03",
      //"ForkJobLog":  false
    }
  }
}
```
2. ##### Separate logs by scopes

To separate logs folder, we will init new log scope with specified properties:
- *ServiceId*: Guid value
- *ServiceDescription*: string value

You may want to store *_logScope* to dispose it later.
```cs
// {Logging:File:Directory}/{ServiceDescription}/{yyyy_MM_dd-HHmm}.log
_logScope = _logger.BeginScope(new List<KeyValuePair<string, object>>
{
    new KeyValuePair<string, object>("ServiceId", Id),
    new KeyValuePair<string, object>("ServiceDescription", Description)
});
```

To separate log file for job, we will init new log scope with specified properties:
- *TraceId*: string value
- *Operation*: string value

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

```

3. ##### Write log scopes to file

Any log scopes specified by the string will be written to the log file

```cs
using (logger.BeginScope("Scope 1")){
  using(logger.BeginScope("Scope 1.1")){
    logger.LogInformation("Log inside scope");
  }
}
logger.LogInformation("Log outside scope");

//---- Begin: Scope 1
//-------- Begin: Scope 1.1
//
//{Timestamp} {LogLevel}: Log inside scope
//
//--------   End: Scope 1.1
//----   End: Scope 1
//{Timestamp} {LogLevel}: Log outside scope

```

The library can be accessed via Nuget:
- [Juice.Extensions.Logging.File](https://www.nuget.org/packages/Juice.Extensions.Logging.File)

### SignalR logger provider

#### Purposes

- Send logs to signalR hub
- Log channel by *ServiceId*
- Support log *State*, *Contextual* and *Scopes*

#### Usage

1. ##### Configure log builder
Call *AddSignalRLogger()* on the logging builder to add a customized file logger provider.
```cs
using Juice.Extensions.Logging;
...

var builder = WebApplication.CreateBuilder(args);

builder.Logging.AddSignalRLogger(builder.Configuration.GetSection("Logging:SignalR"));

```

*Logging:SignalR* configuration section may be present:
- *Directory*: where to store logging files for the *SignalRLogger* itself and the *HubConnection*
- *RetainPolicyFileCount*: default value is **50**
- *MaxFileSize*: default value is **5 * 1024 * 1024**;
- *BufferTime*: the log entries will be written to the file after the interval (5 seconds by default)
- *GeneralName*: the sub-directory name to store logging files. It's optional and the default value is *SignalRLogger* and *SignalRLoggerHubConnection*
- *HubUrl*: the signalR hub's url
- *JoinGroupMethod*: optional method name to join group with *ServiceId*, default value is *JoinGroup*
- *LogMethod*: optional method name to send the log data, default value is *LoggingAsync*
- *StateMethod*: optional method name to send the state, default value is *StateAsync*
- *ScopesSupported*: hub is supported scopes logging or not
- *ExcludedScopes*: excluded scopes to send to the client. We will exclude *ServiceId*, *ServiceDescription*, *TraceId*, *OperationState*, *Contextual* by default, you can override this default by implementing [IScopesFilter](#implement-iscopesfilter) yourself
- *Disabled*: soft disable this logging channel

```json
// appsettings.json
{
  "Logging": {    
    "File": {
      "Directory": "C:\\Workspace\\Services\\logs",
      "HubUrl": "https://localhost:57754/loghub"
    }
  }
}
```
2. ##### Using logs scopes

Basically, the supported scopes are like the [File logger provider](#separate-logs-by-scopes) above, but SignalR logger provider is support more stronger named scopes:
- *Contextual*: make it possible for your logging client to specify log colors by contextual. Ex: success, warning, danger...
- *OperationState*: if specified, we will send to the *StateAsync* method on the hub

Any [log scopes specified by the string](#write-log-scopes-to-file) will be sent to the log hub if it is supported.

```cs
  using (_logger.BeginScope(new List<KeyValuePair<string, object>>
  {
      new KeyValuePair<string, object>("TraceId", jobId),
      new KeyValuePair<string, object>("Operation", "Invoke recurring tasks")
  }))
  {
      // will send scopes []
      _logger.LogInformation("Starting...");
      for (var i = 0; i < 10; i++)
      {
          if (_stopRequest!.IsCancellationRequested) { break; }
          using (_logger.BeginScope($"Task {i}"))
          {
            // will send scopes ["Task {i}"]
            for (var j = 0; j < 10; j++)
            {
                _logger.LogInformation("Processing {subtask}", j);
                  using (_logger.BeginScope(new List<KeyValuePair<string, object>>
                {
                    new KeyValuePair<string, object>("Contextual", "success")
                }))
                {
                    _logger.LogError("Subtask {task} completed", j);
                }
            }
            
            using (_logger.BeginScope("Succeeded"))
            {
                // will send scopes ["Task {i}", "Succeeded"]
                _logger.LogInformation("Task {task} succeeded", i);
            }
          }
      }
      using (_logger.BeginScope(new List<KeyValuePair<string, object>>
      {
          new KeyValuePair<string, object>("OperationState", "Succeeded")
      }))
      {   // will send to the state method, not logging method
          _logger.LogInformation("End");
      }
  }

```

3. ##### Hub implementation

The code block below is an example of signalR hub to handle logging

```cs
    public class LogHub : Hub<ILogClient>
    {
        public async Task LoggingAsync(Guid serviceId, string? traceId, string category, string message, LogLevel level, string? contextual, object[] scopes)
        {
            await Clients.Others.LoggingAsync(serviceId, traceId, category, message, level, contextual, scopes);
        }
        public async Task StateAsync(Guid serviceId, string? traceId, string state, string message)
        {
            await Clients.Others.StateAsync(serviceId, traceId, state, message);
        }
        public async Task BeginScopeAsync(Guid serviceId, string? traceId, string category, object scope)
        {
            await Clients.Others.BeginScopeAsync(serviceId, traceId, category, scope);
        }
        public async Task EndScopeAsync(Guid serviceId, string? traceId, string category, object scope)
        {
            await Clients.Others.EndScopeAsync(serviceId, traceId, category, scope);
        }
    }
```

4. ##### Implement *IScopesFilter*
To custom the log scopes filter to send to SingalR client, you can implement the *IScopesFilter* and register it with DI

```csharp
// CustomScopesFilter.cs
using Microsoft.Extensions.Options;
internal class CustomScopesFilter : IScopesFilter{
  public bool IsIncluded(string scope)
  {
      // your filter
  }
}
```

```c#
// Program.cs
using Juice.Extensions.Logging;
builder.Logging.AddSignalRLogger<CustomScopesFilter>(builder.Configuration.GetSection("Logging:SignalR"));
```

---
**NOTE**

- If you are not supported for *scopes* in the logging method, please set the *ScopesSupported* to *False*
---

The library can be accessed via Nuget:
- [Juice.Extensions.Logging.SignalR](https://www.nuget.org/packages/Juice.Extensions.Logging.SignalR)


### DB logger provider
#### Purposes

- Store log records to the DB, included *ServiceId*, *TraceId* and *Operation*
- Store log metrics by *Service*, *Operation* and *Category* to the DB

#### Usage

Call *AddDbLogger()* and/or *AddMetricsLogger()* on the logging builder to add a customized file logger provider.
```cs
using Juice.Extensions.Logging;
...

var builder = WebApplication.CreateBuilder(args);

// require RootConfiguration to access the ConnectionStrings
builder.Logging.AddDbLogger(builder.Configuration.GetSection("Logging:Db"), builder.Configuration);
builder.Logging.AddMetricsLogger(builder.Configuration.GetSection("Logging:Metrics"), builder.Configuration);

```

*Logging:Db* configuration section may be present:
- *Directory*: where to store logging files for the *DbLogger* itself
- *RetainPolicyFileCount*: default value is **50**
- *MaxFileSize*: default value is **5 * 1024 * 1024**;
- *BufferTime*: the log entries will be written to the DB after the interval (5 seconds by default)
- *GeneralName*: the sub-directory name to store logging files. It's optional and the default value is *DBLogger*
- *DatabaseProvider*: the database provider name
- *Schema*: the DB schema to create the log table
- *ConnectionName*: the DB connection name
- *Disabled*: soft disable this logging channel

*Logging:Metrics* configuration section is as same as the *Logging:Db* but added:
- *SampleRate*: the sample rate to collect the log metrics, if not set, the collection time is 5 seconds.

```json
// appsettings.json
{
  "Logging": {    
    "Db": {
      "Directory": "C:\\Workspace\\Services\\logs",
      "DatabaseProvider": "PostgreSQL",
      "LogLevel": {
        "Default": "Warning"
      }
    },
    "Metrics": {
      "Directory": "C:\\Workspace\\Services\\logs",
      "DatabaseProvider": "PostgreSQL",
      "SampleRate": "00:00:30",
      "LogLevel": {
        "Default": "Information"
      }
    }
  }
}
```


The library can be accessed via Nuget:
- [Juice.Extensions.Logging.EF](https://www.nuget.org/packages/Juice.Extensions.Logging.EF)
- [Juice.Extensions.Logging.EF.PostgreSQL](https://www.nuget.org/packages/Juice.Extensions.Logging.EF.PostgreSQL)
- [Juice.Extensions.Logging.EF.SqlServer](https://www.nuget.org/packages/Juice.Extensions.Logging.EF.SqlServer)