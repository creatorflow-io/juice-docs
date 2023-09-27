---
title: "Logging"
date: 2023-04-03T21:02:02+07:00
draft: false
weight: 1
---

#### Purposes

- Separate log folder for services
- Limit log file size
- Limit the number of log files
- Separate log file for job

#### Usage

1. **Configure log builder**
Call *AddBgServiceFileLogger()* on the logging builder to add a custom file logger provider.
```cs
using Juice.BgService.Extensions.Logging;
...

var builder = WebApplication.CreateBuilder(args);

builder.Logging.AddBgServiceFileLogger(builder.Configuration.GetSection("Logging:File"));

```

*Logging:File* configuration section may be present:
- *Directory*
- *RetainPolicyFileCount*: default value is **50**
- *MaxFileSize*: default value is **5 * 1024 * 1024**;
- *ForkJobLog*: default value is **true**

2. ##### Separate logs by scope

To separate logs folder, we will init new log scope with specified properties:
- *ServiceId*: Guid value
- *ServiceDescription*: string value

You may want to store *_logScope* to dispose it later.
```cs
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
using (_logger.BeginScope(new List<KeyValuePair<string, object>>
{
    new KeyValuePair<string, object>("JobId", JobId),
    new KeyValuePair<string, object>("JobDescription", JobDescription)
})){
    // job processing
}
```