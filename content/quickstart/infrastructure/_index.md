---
title: "Infrastructure"
date: 2023-04-13T10:49:24+07:00
draft: false
weight: 4
---

The main purpose of this layer is to implement repository interfaces which were defined in **Domain** layer.
If you need to customize the entity classes to fit the infrastructure (like EF) or optimize the DB, you can do it here.

It's named `{namespace}.{feature}.{infrastructure}`. Eg: **Juice.Abc.EF**

Basically, we will create a DbContext that inherits [DbContextBase]({{< ref "core/entityframework">}}) to work with the database. Then use the created DbContext to implement repositories.

The project tree will look like this:

```csharp
|-- Dtos    // optional
|-- Migrations   // only if you have selected a DB backend for your application 
|-- Repositories
|   |-- AbcRepository.cs   
|-- DependencyInjection     // optional
|   |-- YourServiceCollectionExtensions.cs  // to register your infrastructure services
|-- YourDbContext.cs
```

### Separate migrations

If you want to support multiple DB backends like SqlServer, PostgreSQL... then you need to [separate migrations into other projects](https://learn.microsoft.com/en-us/ef/core/managing-schemas/migrations/projects?tabs=dotnet-core-cli). Eg: **Juice.Abc.EF.SqlServer** and  **Juice.Abc.EF.PostgreSQL**.

These are steps to separete migrations:
- Firstly, you must implement an **IDesignTimeDbContextFactory** for your DBContext

```csharp {linenos=false,hl_lines=[2,21,45,55],linenostart=1}
    ...
    using Juice.Extensions.DependencyInjection;
    ...

    public class AbcDbContextFactory : IDesignTimeDbContextFactory<AbcDbContext>
    {
        public WorkflowDbContext CreateDbContext(string[] args)
        {
            Environment.SetEnvironmentVariable("ASPNETCORE_ENVIRONMENT", "Development");
            var resolver = new DependencyResolver
            {
                CurrentDirectory = AppContext.BaseDirectory
            };

            resolver.ConfigureServices(services =>
            {

                // Register DbContext class
                var configService = services.BuildServiceProvider().GetRequiredService<IConfigurationService>();

                var configuration = configService.GetConfiguration(args);

                var provider = configuration.GetSection("Provider").Get<string>() ?? "SqlServer";
                var connectionName =
                    provider switch
                    {
                        "PostgreSQL" => "PostgreConnection",
                        "SqlServer" => "SqlServerConnection",
                        _ => throw new NotSupportedException($"Unsupported provider: {provider}")
                    }
                ;
                var connectionString = configuration.GetConnectionString(connectionName);

                services.AddDbContext<AbcDbContext>(options =>
                {
                    switch (provider)
                    {
                        case "PostgreSQL":
                            AppContext.SetSwitch("Npgsql.EnableLegacyTimestampBehavior", true);

                            options.UseNpgsql(
                               connectionString,
                                x =>
                                {
                                    x.MigrationsAssembly("Juice.Abc.EF.PostgreSQL");
                                });
                            break;

                        case "SqlServer":

                            options.UseSqlServer(
                                connectionString,
                                x =>
                                {
                                    x.MigrationsAssembly("Juice.Abc.EF.SqlServer");
                                });
                            break;
                        default:
                            throw new NotSupportedException($"Unsupported provider: {provider}");
                    }

                });

            });

            return resolver.ServiceProvider.GetRequiredService<AbcDbContext>();
        }
    }
```

- Create a web application then add references to all infrastructure projects. Eg: **Juice.Abc.Host**
    - Register your DbContext into **IServiceCollection**
    - Set it as a **startup project**
- In the **Package Manager Console**
    - Select the **Default project** to your backend project as same as the provider that you want to generate the migration. Eg: **Juice.Abc.EF.PostgreSQL**
    - Exec command:

```powershell
Add-Migration InitDb -Context AbcDbContext -OutputDir Migrations -Args "--provider PostgreSQL"
``` 

- Well done!

### CQRS

The typical CQRS implementation will segregate the Read/Write model and also segregate the Read/Write DB. So it can be used together with complex pattern [Event Sourcing](https://learn.microsoft.com/en-us/previous-versions/msp-n-p/jj591559(v=pandp.10)) or is it simply complicated to implement for newbies.

However, a simpler approach is to use ReadOnly DB to query or read models. It's supported by several types of databases.
So you can create a DBContext intended to be read-only and use it with a read-only connection string.