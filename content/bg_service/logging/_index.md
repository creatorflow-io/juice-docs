---
title: "Logging"
date: 2023-04-03T21:02:02+07:00
draft: false
weight: 1
version: latest
---
{{< versions doc=bg_logging >}}

### File logger provider
#### Purposes

- Separate log folder for services
- Limit log file size
- Limit the number of log files
- Separate log file for job

#### Usage

1. ##### Configure log builder
Call *AddBgServiceFileLogger()* on the logging builder to add a customized file logger provider.
```cs
using Juice.BgService.Extensions.Logging;
...

var builder = WebApplication.CreateBuilder(args);

builder.Logging.AddBgServiceFileLogger(builder.Configuration.GetSection("Logging:File"));

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
- *JobId*: string value
- *JobDescription*: string value

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
- [Juice.BgService.Extensions.Logging.File](https://www.nuget.org/packages/Juice.BgService.Extensions.Logging.File)

### SignalR logger provider

#### Purposes

- Send logs to signalR hub
- Log channel by *ServiceId*
- Support log *State*, *Contextual* and *Scopes*

#### Usage

1. ##### Configure log builder
Call *AddBgServiceFileLogger()* on the logging builder to add a customized file logger provider.
```cs
using Juice.BgService.Extensions.Logging;
...

var builder = WebApplication.CreateBuilder(args);

builder.Logging.AddBgServiceSignalRLogger(builder.Configuration.GetSection("Logging:SignalR"));

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
- *JobState*: if specified, we will send to the *StateAsync* method on the hub

Any [log scopes specified by the string](#write-log-scopes-to-file) will be sent to the log hub if it is supported.

```cs
  using (_logger.BeginScope(new List<KeyValuePair<string, object>>
  {
      new KeyValuePair<string, object>("JobId", jobId),
      new KeyValuePair<string, object>("JobDescription", "Invoke recurring tasks")
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
          new KeyValuePair<string, object>("JobState", "Succeeded")
      }))
      {   // will send to the state method, not logging method
          _logger.LogInformation("End");
      }
  }

```

3. ##### Hub implementation

The code block below is an example of signalR hub to handle logging

```cs
    public class LogHub : Hub
    {
        public async Task LoggingAsync(Guid serviceId, string? jobId, string message, LogLevel level, string? contextual, string[] scopes)
        {
            try
            {
                await Clients.Others.SendAsync("LoggingAsync", serviceId, jobId, message, level, contextual, scopes);
            }
            catch { }
        }

        public async Task StateAsync(Guid serviceId, string? jobId, string state, string message)
        {
            try
            {
                await Clients.Others.SendAsync("StateAsync", serviceId, jobId, state, message);
            }
            catch { }
        }
    }
```

---
**NOTE**

- If you are not supported for *scopes* in the logging method, please set the *ScopesSupported* to *False*
---