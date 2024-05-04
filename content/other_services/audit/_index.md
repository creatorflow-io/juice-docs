---
title: "Audit services"
date: 2023-04-03T20:23:30+07:00
draft: false
weight: 1
version: latest
---

{{< versions doc=audit >}}

### Quick access
- [Data audit](#data-audit)
    + [Configure entity model](#configure-entity-model)
    + [Implement an auditable DbContext](#implement-an-auditable-dbcontext)
    + [Handle DataEvent](#handle-dataevent)
- [Default audit service](#default-audit-service)
    + [Core concept](#core-concept)
    + [Usage](#usage)
    + [Configuration](#configuration)
    + [Programming API](#programming-api)


### Data audit
We already have a data audit framework based on EF Core and fully implemented in the *DbContextBase* abstract class. It will send a *DataEvent* via MediatR's notification, the object contains:
- *Name*: type of change (**Inserted/ Modified/ Deleted**)
- *AuditRecord*: contains all needed data audit information (**User, Database, Schema, Table, KeyValues, OriginalValues, CurrentValues**)

#### What will you do?

1. ##### Configure entity model

Only entities marked as auditable will have audit events triggeredd on their changes. You can mark an entity as auditable by two ways:
- Inherit from *IAuditable* interface
- Call **IsAuditable()** on *EntityTypeBuilder* for your entity

```csharp

    public class SampleEntity: IAuditable{

    }

    // OR

    // Inside DbContext
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<SampleEntity>(entity =>
        {
            ...
            entity.IsAuditable();
            ...
        });

    }

```

2. ##### Implement an auditable DbContext

**If your DbContext inherits from the abstract class *DbContextBase*, please bypass this step**. But if you want to implement a DbContext yourself, it must inherit *IAuditableDbContext* interface and you can follow these steps to configure:

- ConfigureServices: we need *User* info and *IMediator* service to function.
    + If you are using pooled DbContext, your DbContext's *constructor* must clean, it must contains only DbContextOptions argurment. So we will use PooledDbContextFactory pattern and implement a factory that inherit *IDbContextFactory* and call ConfigureServices() inside.

    ```csharp
        // Dependency injection
        services.AddPooledDbContextFactory<SampleDbContext>();
        services.AddScoped<SampleDbContextScopedFactory>();
        services.AddScoped(sp => sp.GetRequiredService<SampleDbContextScopedFactory>().CreateDbContext());

    ``` 

    ```csharp
        // Factory
        public class SampleDbContextScopedFactory : IDbContextFactory<SampleDbContext>
        {

            private readonly IDbContextFactory<SampleDbContext> _pooledFactory;
            private readonly IServiceProvider _serviceProvider;

            public SampleDbContextScopedFactory(
                IDbContextFactory<SampleDbContext> pooledFactory,
                IServiceProvider serviceProvider)
            {
                _pooledFactory = pooledFactory;
                _serviceProvider = serviceProvider;
            }

            public SampleDbContext CreateDbContext()
            {
                var context = _pooledFactory.CreateDbContext();
                context.ConfigureServices(_serviceProvider);
                return context;
            }
        }

    ```

    + Otherwise, we can call *ConfigureServices()* in DbContext's constructor directly.

- Configure model to mark auditable entities.
    + Call *modelBuilder.ConfigureAuditableEntities()* inside *OnModelCreating(ModelBuilder modelBuilder)* to mark all entities that inherit *IAuditable* as auditable.
    + Call **IsAuditable()** on *EntityTypeBuilder* for your entity (step 1).

- Store the audit information before you *SaveChanges* and finally dispatch data change events.

The code block below is an example for an auditable DbContext.
```csharp
    public class SampleDbContext : DbContext,
        IAuditableDbContext
    {
        #region Auditable context
        public string? User { get; protected set; }

        private IHttpContextAccessor? _httpContextAccessor;
        public List<AuditEntry>? PendingAuditEntries { get; protected set; }
        #endregion

        protected IMediator? _mediator;

        protected ILogger? _logger;

        protected DbOptions? _options;

        /// <summary>
        /// Please call <c>ConfigureServices(IServiceProvider serviceProvider)</c> directly in your constructor
        /// or inside <c>IDbContextFactory.CreateDbContext()</c> if you are using PooledDbContextFactory</para>
        /// to init internal services</para>
        /// </summary>
        /// <param name="options"></param>
        public SampleDbContext(DbContextOptions options)
            : base(options)
        {

        }

        public virtual void ConfigureServices(IServiceProvider serviceProvider)
        {
            // Get user from HttpContext
            var httpContextAccessor = serviceProvider.GetService<IHttpContextAccessor>();
            User = httpContextAccessor?.HttpContext?.User?.FindFirst(ClaimTypes.Name)?.Value;

            try
            {
                _mediator = serviceProvider.GetService<IMediator>();
            }
            catch (Exception ex)
            {
            }
        }
        
        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);
            
            // Mark all entities that inherit IAuditable as auditable
            modelBuilder.ConfigureAuditableEntities();

            // OR/ AND
            // Mark specified entities as auditable

            modelBuilder.Entity<SampleEntity>(entity =>
            {
                ...
                entity.IsAuditable();
                ...
            });
        }

        private void ProcessingChanges()
        {
            if (PendingAuditEntries == null)
            { return; }
            _mediator.DispatchDataChangeEventsAsync(this).GetAwaiter().GetResult();
        }

        public override async Task<int> SaveChangesAsync(bool acceptAllChangesOnSuccess, CancellationToken cancellationToken = default(CancellationToken))
        {
            this.SetAuditInformation(_logger);
            PendingAuditEntries = _mediator != null ? this.TrackingChanges(_logger)?.ToList() : default;

            try
            {
                return await base.SaveChangesAsync(acceptAllChangesOnSuccess, cancellationToken);
            }
            finally
            {
                ProcessingChanges();
            }
        }

        public override int SaveChanges(bool acceptAllChangesOnSuccess)
        {
            this.SetAuditInformation(_logger);
            PendingAuditEntries = _mediator != null ? this.TrackingChanges(_logger)?.ToList() : default;

            try
            {
                return base.SaveChanges(acceptAllChangesOnSuccess);
            }
            finally
            {
                ProcessingChanges();
            }
        }

    }
```

3. ##### Handle DataEvent
- Implement your own handler by inheriting *MediatR.INotificationHandler\<DataEvent\<T\>\>*, so you can store anything you need proactively.
- Use default audit service that we provided below.

### Default audit service

In this part, we provide not only data audit service but access logging service.

#### Core concept

1. ##### AuditContext

The AuditContext contains information about user access log, data audit entries. It will be initialized while a request is starting, be fulfilled while the request pipeline is invoking, and be committed by the *IAuditService* when the request pipeline is finished.

2. ##### IAuditContextAccessor

The accessor to access AuditContext in the scope.

3. ##### IAuditService

The service that processes the audited information to store or send it to other systems.

4. ##### AuditMiddleware

The middleware that inits the AuditContext at the start of the request. It also collects the important information about the request:
 - *User*, *DateTime*, *Action*
 - **RequestInfo**: *RequestId*, *Headers*, *Host*, *Method*, *Path*, *RemoteIpAddress*
 - **ServerInfo**: *MachineName*, *OSVersion*, *SoftwareVersion*, *AppName*
 - **ResponseInfo**: *StatusCode*, *Headers*, *ElapsedMilliseconds*.
    + The *Data* was intended to be filled by other middleware that you will implement yourself.
    + The *Message*, *Error* may be set by this middleware if an exception was thrown while invoking the next steps of the pipeline and there is no other middleware filling it. Otherwise, it may be filled by a custom middleware like the *Data*.

It must be placed after *Authentication* middleware to have *User* information.

NOTE: **It often increases by 30ms - 70ms in request response time because saves the audit information with the EF repository or gRPC** (tested with *SQL Server* and *PostgreSQL* on Docker, it depends on whether there are any DataAudit entries or not, and database provider).

The figure below is the request processing flow with *AuditMiddleware* injected.

{{< figure src="/images/services/Audit-process.svg" title="Audit process" >}}

#### Usage

We currently provide two ways to use the audit services.

1. **EF repository**

Use the *AuditMiddleware* to collect audit information and directly persist into the EF repository.

```csharp
// Minimal application Program.cs
using Juice.Audit.AspNetCore.Extensions;
...

var builder = WebApplication.CreateBuilder(args);

// configure Audit services with default usage
builder.Services.ConfigureAuditDefault(builder.Configuration, options =>
{
    // configure database options
    //options.DatabaseProvider = "PostgreSQL";
});

// MediatR service is required for Audit services
builder.Services.AddMediatR(typeof(Program));

...
var app = builder.Build();
...

// if specified, the authentication middleware must place before the AuditMiddleware to have User info.
// app.UseAuthentication();

// use AuditMiddleware to handle audit for specified request
app.UseAudit("yourAppName", options =>
{
    options.Include("", "GET");
    options.Exclude("/Index");
});

```

2. **gRPC repository**

Use the *AuditMiddleware* to collect audit information and remotely persist to the audit server by gRPC.

In this usage, we will use *ConfigureAuditGrpcClient()* instead of *ConfigureAuditDefault()* for the application.

```csharp
// Minimal application Program.cs
using Juice.Audit.AspNetCore.Extensions;
...

var builder = WebApplication.CreateBuilder(args);

// configure Audit services to use gRPC client
builder.Services.ConfigureAuditGrpcClient(options =>
{
    options.Address = new Uri("https://localhost:7285"); // the audit server endpoint
    // configure grpc client factory options
    options.ChannelOptionsActions.Add(o =>
    {
        o.HttpHandler = new SocketsHttpHandler
        {
            PooledConnectionIdleTimeout = Timeout.InfiniteTimeSpan,
            KeepAlivePingDelay = TimeSpan.FromSeconds(60),
            KeepAlivePingTimeout = TimeSpan.FromSeconds(30),
            EnableMultipleHttp2Connections = true
        };
    });
});

// MediatR service is required for Audit services
builder.Services.AddMediatR(typeof(Program));

...
var app = builder.Build();
...

// if specified, the authentication middleware must place before the AuditMiddleware to have User info.
// app.UseAuthentication();

// use AuditMiddleware to handle audit for specified request
app.UseAudit("yourAppName", options =>
{
    options.Include("", "GET");
    options.Exclude("/Index");
});

```

We also need an audit server to handle the audit messages.

```csharp
// Audit server Program.cs
using Juice.Audit.AspNetCore.Extensions;
...

var builder = WebApplication.CreateBuilder(args);

// configure Audit services as a server
builder.Services.ConfigureAuditGrpcHost(builder.Configuration,
    options =>
    {
        options.DatabaseProvider = "PostgreSQL";
    });

// MediatR and Grpc services is required for Audit services
builder.Services.AddMediatR(typeof(Program));
builder.Services.AddGrpc(o => o.EnableDetailedErrors = true);

...
var app = builder.Build();
...
// map gPRC endpoint to handle audit messages
app.MapAuditGrpcServer();

```

Please follow this [link](https://learn.microsoft.com/en-us/aspnet/core/grpc/performance?view=aspnetcore-7.0) to read about rRPC performance tips.

#### Configuration

1. ##### EF repositories

You can configure *DbOptions* to specify database provider, schema... See [DbOptions]({{<ref "core/entityframework/#dboptions" >}}) for more information.

2. ##### Audit filters

You can configure *AuditFilterOptions* to specify the paths and methods you want to audit, as well as request/response headers to store in the access log.

- Path and method filtering to processing
    + You can describe multiple paths, zero or more methods per path. The path and method are **case insensitive**
    + You can define a rule with specified priority and then add it to *Filters* or load the rule from configuration
    + You can use **\*** to describe a single segment
    + You can use **#** to describe zero or more segments
- Response status filtering
    + You can describe multiple response status codes so that *AuditMiddleware* will process the request upon completion
    + The status codes described will be associated with the filter path at the same time
- Request/response headers filtering to store
    + You can use **\*** to describe a single segment
    + You can use **#** to describe zero or more segments
    + Request headers will be stored by default: **:authority:, accept-#, content-*, x-forwarded-#, referer, user-agent**
    + Response headers will be stored by default: **content-\***

Some main methods of *AuditFilterOptions* that help us build the filter options:
- *Clear*: clear all existing filter entries
- *Include*: append a filter entry to include path, methods...
- *Exclude*: append a filter entry to exclude path, methods...
- *Merge*: merge new filter entries if they do not already exist
- *IsExists*: check if the filter entry already exists
- *StoreEmptyRequestHeaders*: clear all request header filters
- *StoreRequestHeaders*: add new request header filters
- *StoreEmptyResponseHeaders*: clear all response header filters
- *StoreResponseHeaders*: add new response header filters

---
**NOTE**   
 - Leave **blank** (of *Path*, *Methods*, *StatusCodes*) will have the same meaning as **any**
 - Rules added **later** by filter builder will have **higher priority** by default
 - Rules added by configuration section will have the same priority level of 0 by default
---

The code blocks below are two ways to configure the audit filter

```cs
//Program.cs
var configs = new AuditFilterOptions();
builder.Configuration.Bind("Audit", configs);
app.UseAudit("yourAppName", options =>
{
    // describe the paths and methods to logging user acess and data audit
    options.Include("", "GET"); // include any GET request
    options.Exclude("/Index"); // exclude requests to /Index

    // include all POST, PUT request to the path that:
    // - starts with /kernel
    // - has one or more segment after
    // Ex: /kernel/foo, /kernel/foo/bar, /kernel/foo/bar/barbar ...
    options.Include("/kernel/*/#", "POST", "PUT");

    // merge filter entries from appsettings
    options.Merge(configs.Filters);
    // Only store the request headers like: accept, accept-encoding, accept-language-x,
    // content-type, content-length... but not content, content-type-y
    options.StoreEmptyRequestHeaders()
        .StoreRequestHeaders("accept-#", "content-*");
});
```

```cs
//appsettings.json
"Audit:Filters": [ // path filter antries
    {
        "Methods": [ "POST", "PUT", "DELETE" ],
        "Priority": -1
    },
    {
        "IsExcluded": true,
        "Path": "/#/negotiate"
    },
    {
        "IsExcluded": true,
        "Path": "/error/*"
    },
    {
        "StatusCodes": [400, 500]
    }
]

```

The model presents the filter entry here
```cs
class PathFilterEntry
{
    public int Priority { get; set; } = 0;
    public bool IsExcluded { get; set; } = false;
    public string Path { get; set; } = string.Empty;
    public string[] Methods { get; set; } = Array.Empty<string>();
    public int[] StatusCodes { get; set; } = Array.Empty<int>();
}
```

#### Programming API

1. ##### Fulfill response information

The *Data*, *Msg*, *Err* of *ResponseInfo* can only be assigned values **once**. In *AuditMiddleware*, we do not try to set *Data* but rather *Msg* and *Err* by handling the pipeline call exception. To set *Msg*, *Err*, *Data* in other middlewares or action filter, we can use *IAuditContextAccessor* service to access the *AuditContext* and then try to set the response information.

```csharp
    auditContextAccessor.AuditContext?.UpdateResponseInfo(response => 
    {
        response.TrySetMessage("An useful message");
        response.TrySetData("{\"foo\":\"bar\"}");
    });
```

2. ##### Add more audit entries
By default, we use *DataEvenNotificationtHandler* to handle *DataEvent* and add *DataAudit* entries, but you can add *DataAudit* entries manually by access the *AuditContext*.

```csharp
    auditContextAccessor.AuditContext?.AddAuditEntries(entry1, entry2...);
```

3. ##### Additional metadata
If the current audit information is not enough, you can add more information to the *Metadata* by access the *AuditContext*.


```csharp
    auditContextAccessor.AuditContext?.AccessRecord?.SetMetadata(key, value);
    // OR
    auditContextAccessor.AuditContext?.AccessRecord?.SetMetadataJson(jsonString);
```

The library can be accessed via Nuget:
- [Juice.Audit](https://www.nuget.org/packages/Juice.Audit)
- [Juice.Audit.Api.Contracts](https://www.nuget.org/packages/Juice.Audit.Api.Contracts)
- [Juice.Audit.Api](https://www.nuget.org/packages/Juice.Audit.Api)
- [Juice.Audit.EF](https://www.nuget.org/packages/Juice.Audit.EF)
- [Juice.Audit.EF.SqlServer](https://www.nuget.org/packages/Juice.Audit.EF.SqlServer)
- [Juice.Audit.EF.PostgreSQL](https://www.nuget.org/packages/Juice.Audit.EF.PostgreSQL)