---
title: "File storage"
date: 2023-03-31T10:58:47+07:00
weight: 5
draft: false
---

In this section, we introduce *File storage* to build a file management system.

#### Purposes
- Provides a web API for large file uploads and a javascript library to working on web client
- Resumeable upload process
- Easily integrated with MAM system
- Multiple file transfer protocols on the backend
- Multiple storages on the same backend

#### Concept

{{< figure src="/images/services/Storage-process.svg" title="File storage sequence diagram" >}}

1. **IStorageResolver**

This service will resolves *Storage* according to its identity (it is storage's web endpoint by default).

2. **IStorage**

It is a proxy that contains backend storage endpoints so we can access storage using multiple endpoints and protocols.

3. **IStorageProvider**

It provides methods to access the endpoint depending on its protocol. By default, we have implemented FTP, Samba and LocalDisk access providers.

4. **IUploadManager**

It manages the upload process by storing the upload process to *IUploadRepo* to continue it and can load-balance between servers.

After a successful upload, it will try to add the uploaded file to *IFileRepo* for storage.

5. **StorageMiddleware**

This middleware handles requests, attemps to resolve *Storage* by requesting enpoint and operates it.

#### Usage

To use storage services, please call the *AddStorage()* on *IServiceCollection* instance. By default, we have implemented in-memory repositories to store upload processes.


```cs
// Program.cs
using Juice.Storage.Extensions;
using Juice.Storage.Local;
...
var builder = WebApplication.CreateBuilder(args);

// add storage services
builder.Services.AddStorage();

// use in-memory repos
builder.Services.AddInMemoryUploadManager(builder.Configuration.GetSection("Juice:Storage"));

// maintain timed-out uploads
builder.Services.AddInMemoryStorageMaintainServices(builder.Configuration.GetSection("Juice:Storage"),
    new string[] { "/storage", "/storage1" },
    options =>
    {
        options.CleanupAfter = TimeSpan.FromMinutes(5);
        options.Interval = TimeSpan.FromMinutes(1);
    });
// access FTP, Samba, LocalDisk enpoints
builder.Services.AddLocalStorageProviders();
...
var app = builder.Build();
...

app.UseStorage(options =>
{
    options.Endpoints = new string[] { "/storage", "/storage1" };
    options.SupportDownloadByPath = true;
});
app.Run();

```

#### Configuration

1. ##### In-memory storage options

```json
{
  "Juice:Storage": {
    "Storages": [
      {
        "WebBasePath": "/storage",
        "Endpoints": [
          {
            "Protocol": "LocalDisk",
            "Uri": "C:\\Workspace\\Storage"
          }
        ]
      },
      {
        "WebBasePath": "/storage1",
        "Endpoints": [
          {
            "Protocol": "Smb",
            "BasePath": "\\\\ip-or-name",
            "Uri": "\\\\ip-or-name\\path\\to\\storage",
            //"Identity": "demo",
            //"Password": "demo"
          }
        ]
      }
    ]
  }
}
```
In these options:
- *WebBasePath*: used to resolve storage if the client requests this path
- *Endpoints*: list of endpoints that will be used for the server to access the file storage (we currently support FTP, Smb, LocalDisk protocols)

2. ##### Storage maintain options

We can configure *StorageMaintainOptions* using configuration section or configure action (see the code block above).
- *CleanupAfter*: the waiting period before the upload times out
- *Interval*: the waiting time between checks

3. ##### Storage middleware options

We can configure *StorageMiddlewareOptions* directly using configure action.
- *Endpoints*: web path matches the storage identity
- *SupportDownloadByPath*: if enabled, clients can download the file if the correct file path is provided

4. ##### Upload Options

We can configure *UploadOptions* using configuration section or configure action.

- *SectionSize*: specify max size for each upload sections
- *DeleteOnAbort*: behavior when a client's' upload request is aborted

---
**NOTE**

- **You should configure your web server's security policies to match  *SectionSize***
---

5. ##### Authorization policies

If you have added *app.UseAuthorization()* then the storage authorization policies need to be configured.

```cs
// Program.cs
using Juice.Storage.Authorization;
...
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy(StoragePolicies.CreateFile, policy =>
    {
        // policy requirements
    });
    options.AddPolicy(StoragePolicies.DownloadFile, policy =>
    {
        // policy requirements
    });
});
```

The sample source code is available on [github](https://github.com/creatorflow-io/Juice/tree/master/services/storage/src/Juice.Storage.App).

The library can be accessed via Nuget and npmjs:
- [Juice.Storage](https://www.nuget.org/packages/Juice.Storage)
- [Juice.Storage.Local](https://www.nuget.org/packages/Juice.Storage.Local)
- [@juice-js/upload](https://www.npmjs.com/package/@juice-js/upload)