---
title: "Mediator"
date: 2023-04-03T20:23:50+07:00
draft: false
weight: 7
---

We provide IdentifiedCommand pattern to deduplicate incoming events with built-in EF backend (SqlServer/PostgreSQL) or Redis backend. Please follow the link [Implement idempotent Commands](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/microservice-application-layer-implementation-web-api#implement-idempotent-commands) for more information.

```csharp
    await mediator.Send(new IdentifiedCommand<OriginCommand>(originCommand, originCommandId));
```
The request manager will verify *originCommandId* does not exist before sending *originCommand* to your **OriginCommandHandler**.

#### EF backend
Please register request management services with your application schema before use.

```csharp {linenos=false,hl_lines=[2,"5-10"],linenostart=1}
    ...
    using Juice.MediatR.RequestManager.EF.DependencyInjection;
    ...

    services.AddRequestManager(configuration, options =>
        {
            options.DatabaseProvider = "SqlServer"; // PostgreSQL
            options.Schema = your-app-schema; 
            options.ConnectionName = "SqlServerConnection";
        });

    // migrate DB if needed
    var context = resolver.ServiceProvider.GetRequiredService<ClientRequestContext>();
    var pendingMigrations = await context.Database.GetPendingMigrationsAsync();

    if (pendingMigrations.Any())
    {
        Console.WriteLine($"[{schema}][ClientRequestContext] You have {pendingMigrations.Count()} pending migrations to apply.");
        Console.WriteLine("[ClientRequestContext] Applying pending migrations now");
        await context.Database.MigrateAsync();
    }
```

The library can be accessed via Nuget:
- [Juice.MediatR.RequestManager.EF](https://www.nuget.org/packages/Juice.MediatR.RequestManager.EF)
- [Juice.MediatR.RequestManager.EF.SqlServer](https://www.nuget.org/packages/Juice.MediatR.RequestManager.EF.SqlServer)
- [Juice.MediatR.RequestManager.EF.PostgreSQL](https://www.nuget.org/packages/Juice.MediatR.RequestManager.EF.PostgreSQL)

#### Redis backend

If you already have Redis so you can use it as a backend.

```csharp {linenos=false,hl_lines=[2,"5-8"],linenostart=1}
    ...
    using Juice.MediatR.Redis.DependencyInjection;
    ...

    services.AddRedisRequestManager(options =>
    {
        options.ConnectionString = configuration.GetConnectionString("Redis");
    });

```

The library can be accessed via Nuget:
- [Juice.MediatR.Redis](https://www.nuget.org/packages/Juice.MediatR.Redis)
