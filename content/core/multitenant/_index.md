---
title: "Multi-tenant"
date: 2023-04-03T20:24:21+07:00
draft: false
weight: 15
---

### MultiTenant
This core feature is useful to verify other core features that we have built because it is developed as a microservice.
It's integrated with [Finbuckle](https://www.finbuckle.com/MultiTenant) to identity the tenant. We provide multiple choices for tenant data store:
- File store
- EFCore store
- gRPC store

We also provide the gRPC API to manage tenant information and tenant settings, so MultiTenant itself is a service with the full implementation of:
- MultiTenant DBContext
- UnitOfWork pattern
- Audit
- gRPC API
- DDD/CQRS and Domain Events pattern
- Integration with event bus
- Per-tenant configuration/options
- xUnit helper

To use multi-tenant, we must register services with **Tenant** type

```csharp 
    var tenantBuilder = builder.Services
        // default Finbuckle services
        .AddMultiTenant<Tenant>(options =>
        {
            // configure tenant options
        })
        .WithBasePathStrategy(options => options.RebaseAspNetCorePathBase = true)
        // AND/OR other strategies
        ;
```
After that, we have 2 choices:

1. Use multi-tenant as a separate serice by building a multi-tenant server and use gRPC client to get tenant information.

So we will register services for tenant host inside tenant microservice.

```csharp {linenos=false,hl_lines=[2],linenostart=1}
    ...
    using Juice.MultiTenant.Api.DependencyInjection;
    ...

    tenantBuilder.ConfigureTenantHost(builder.Configuration, options =>
        {
            options.DatabaseProvider = "PostgreSQL";
            options.ConnectionName = "PostgreConnection";
            options.Schema = "App";
        });
```

Then register services for tenant client inside other microservices.

```csharp {linenos=false,hl_lines=[2],linenostart=1}
    ...
    using Juice.MultiTenant.Grpc.Finbuckle.DependencyInjection;
    ...

    tenantBuilder.ConfigureTenantClient(builder.Configuration, builder.Environment.EnvironmentName);
```

2. Use multi-tenant service directly.

- Default EF store build

```csharp {linenos=false,hl_lines=[2],linenostart=1}
    ...
    using Juice.MultiTenant.EF.DependencyInjection;
    ...

    tenantBuilder.ConfigureTenantEFDirectly(builder.Configuration, options =>
        {
            options.DatabaseProvider = "PostgreSQL";
            options.ConnectionName = "PostgreConnection";
            options.Schema = "App";
        }, builder.Environment.EnvironmentName);
```

- Or manually build tenant service with multiple store options.

```csharp {linenos=false,hl_lines=[2,"5-7"],linenostart=1}
    ...
    using Juice.MultiTenant.Finbuckle.DependencyInjection;
    ...

    tenantBuilder.JuiceIntegration()
        .WithGprcStore(tenantGrpcEndpoint)
        .WithDistributedCacheStore()
        // .WithEFStore(...)
        // you can add many stores at the same time
        ;
```


The library can be accessed via Nuget and npmjs:
- [Juice.MultiTenant](https://www.nuget.org/packages/Juice.MultiTenant)
- [Juice.MultiTenant.EF](https://www.nuget.org/packages/Juice.MultiTenant.EF)
- [Juice.MultiTenant.EF.SqlServer](https://www.nuget.org/packages/Juice.MultiTenant.EF.SqlServer)
- [Juice.MultiTenant.EF.PostgreSQL](https://www.nuget.org/packages/Juice.MultiTenant.EF.PostgreSQL)
- [Juice.MultiTenant.Api.Contracts](https://www.nuget.org/packages/Juice.MultiTenant.Api.Contracts)
- [Juice.MultiTenant.Api](https://www.nuget.org/packages/Juice.MultiTenant.Api)
- [Juice.MultiTenant.Grpc](https://www.nuget.org/packages/Juice.MultiTenant.Grpc)
- [@juice-js/tenants](https://www.npmjs.com/package/@juice-js/tenants)

### MultiTenant DBContext

Please follow up on [Data Isolation with Entity Framework Core](https://www.finbuckle.com/MultiTenant/Docs/v6.10.0/EFCore) to understand this section.

We usually have two options for storing per-tenant application data:
- Separate DB with different DB connection strings, it can be done by per-tenant setting
- Shared DB with an extra TenantId column to determine tenant data

In the second way, we will create a TenantDbContext that implement **IMultiTenantDbContext** interface. 
So we provide a **MultiTenantDbContext** abstraction class that implement these interfaces:
- ISchemaDbContext
- IAuditableDbContext
- IUnitOfWork
- IMultiTenantDbContext

You can inherit **MultiTenantDbContext** abstraction and mark your entity **IsMultiTenant()**, and/or **IsAuditable()**.

```csharp {linenos=false,hl_lines=[1,2,5,10,18,19],linenostart=1}
    using Juice.EF.Extensions;
    using Juice.EF.MultiTenant;
    ...

    public class TenantContentDbContext : MultiTenantDbContext
    {
        public DbSet<TenantContent> TenantContents { get; set; }
        public TenantContentDbContext(IServiceProvider serviceProvider, DbContextOptions options) : base(options)
        {
            ConfigureServices(serviceProvider);
        }

        protected override void ConfigureModel(ModelBuilder modelBuilder)
        {
            modelBuilder.Entity<TenantContent>(options =>
            {
                options.ToTable(nameof(TenantContent), Schema);
                options.IsMultiTenant();
                options.IsAuditable();
            });
        }
    }
```
Or using the **[MultiTenant]** attribute in entity class
```csharp {linenos=false,hl_lines=[1,2],linenostart=1}
    using Finbuckle.MultiTenant;
    [MultiTenant]
    public class TenantContent
    {
        ...
    }
```

See [DBContextBase]({{< ref "core/entityframework">}}) for more information.

The library can be accessed via Nuget:
- [Juice.EF.MultiTenant](https://www.nuget.org/packages/Juice.EF.MultiTenant)