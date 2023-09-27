---
title: "Entity framework"
date: 2023-04-11T09:01:19+07:00
draft: false
weight: 9
---

### DbContextBase
We resolved **Audit**, **UnitOfWork** and **dynamic schema** problems with an abstract DbContext that implemented these interfaces:
- **IAuditableDbContext**
- **IUnitOfWork**
- **IScehmaDbContext**

We'll explain these interfaces below, but remember to call `ConfigureServices( serviceProvider )` for your legacy DbContext in two ways:

- Inside constructor
```csharp {linenos=false,hl_lines=[5],linenostart=1}
    // call inside constructor
    public TimerDbContext(IServiceProvider serviceProvider, DbContextOptions<TimerDbContext> options)
        : base(options)
    {
        ConfigureServices(serviceProvider);
    }

```

- Or inside context factory if you are using pooled DbContext
```csharp    
    public class TimerDbContextScopedFactory : IDbContextFactory<TimerDbContext>
    {

        private readonly IDbContextFactory<TimerDbContext> _pooledFactory;
        private readonly IServiceProvider _serviceProvider;

        public TimerDbContextScopedFactory(
            IDbContextFactory<TimerDbContext> pooledFactory,
            IServiceProvider serviceProvider)
        {
            _pooledFactory = pooledFactory;
            _serviceProvider = serviceProvider;
        }

        public TimerDbContext CreateDbContext()
        {
            var context = _pooledFactory.CreateDbContext();
            context.ConfigureServices(_serviceProvider);
            return context;
        }
    }

```
Then register PooledDbContext

```csharp
    services.AddPooledDbContextFactory<TimerDbContext>((serviceProvider, options) =>
    {
        // configure DB options
    });

    services.AddScoped<TimerDbContextScopedFactory>();
    services.AddScoped(sp => sp.GetRequiredService<TimerDbContextScopedFactory>().CreateDbContext());
```
#### DbOptions

The abstract class *DbOptions* contains:
- DatabaseProvider
- ConnectionName
- Schema
- JsonPropertyBehavior: (UpdageCHANGES, UpdateALL)

The *DbOptions\<T\>* class derived from *DbOptions* will specify a DbContext type that it is used for.

#### Working process
Database context classes derived from **DbContextBase** will:
- Try configure *User* infomation by get from HttpContext if exists
- Set the *_options* if there is an instance of *DbOptions\<YourDbContext\>* was registered
- Set the *Schema* if it is specified in *_options*
- Before call **SaveChanges** or **SaveChangesAsync** of original **DbContext** class
    - Track changes and update to **PendingAuditEntries**
    - Set basic audit info (CreatedUser/ModifiedUser/CreatedDate/ModifiedDate) into auditable entity model
- Call **base.SaveChanges()** or **base.SaveChangesAsync()**
- After call base methods
    - Send domain events collected from entities that inherit **IAggregrateRoot** interface
    - Dispatch **PendingAuditEntries** to notice **DataEvent** through **MediatR**

### IAuditableDbContext

We provide **IAuditableDbContext** interface and its extensions to track changes, update audit columns, and finally, dispatch data change events.
```
    public interface IAuditableDbContext
    {
        List<AuditEntry>? PendingAuditEntries { get; }

        string? User { get; }
    }
```

Database context classes derived from IAuditableDbContext will:
- Set **User** info when init context instance.
- Before call **SaveChanges** or **SaveChangesAsync** of original **DbContext** class
    - Track changes and update to **PendingAuditEntries**
    - Set basic audit info (CreatedUser/ModifiedUser/CreatedDate/ModifiedDate) into auditable entity model
- Dispatch **PendingAuditEntries** to notice **DataEvent** through **MediatR** after called **SaveChanges** or **SaveChangesAsync**, so you can handle **DataEvent** to process audit records.

There are two ways to designate an entity type as auditable:

1. Inherit from *IAuditable* interface
```csharp {linenos=false,hl_lines=[1,2],linenostart=1}
    using Juice.Domain;
    public class Content: IAuditable{
        ...
    }
```
2. Use fluent API *IsAuditable()*

If your entity does not contain basic audit info properties (CreatedUser/ModifiedUser/CreatedDate/ModifiedDate)
you can not access them but these columns are still available in your DB table.
```csharp {linenos=false,hl_lines=[1,2,9],linenostart=1}
    using Juice.EF.Extensions;
    public class ContentDbContext: DbContextBase{
        ...
        protected override void ConfigureModel(ModelBuilder modelBuilder)
        {
            modelBuilder.Entity<Content>(options =>
            {
                options.ToTable(nameof(Content), Schema);
                options.IsAuditable();
            });
        }
    }
```

You can see [Audit service]({{<ref "other_services/audit/_index.md" >}}) for more details.

### ISchemaDbContext
This interface has only one purpose which is support to migrating the DBContext with an optional schema.
After running **Add-Migration** command, you must modify the generated migration file to customize the schema.

```csharp {linenos=false,hl_lines=[2,6,"11-14",17,18,22],linenostart=1}
    ...
    using Juice.EF;
    ...
    public partial class InitEventLog : Migration
    {
        private readonly ISchemaDbContext _schema;

        public InitEventLog() { }

        // add one more constructor to inject ISchemaDbContext service
        public InitEventLog(ISchemaDbContext schema)
        {
            _schema = schema;
        }
        protected override void Up(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.EnsureSchema(
                name: _schema.Schema);

            migrationBuilder.CreateTable(
                name: "IntegrationEventLog",
                schema: _schema.Schema,
                columns: table => new
                ...
```

You also need to replace the original **IMigrationsAssembly** with **DbSchemaAwareMigrationAssembly** in **DbContextOptionsBuilder**

```csharp {linenos=false,hl_lines=[2,7],linenostart=1}
    ...
    using Juice.EF.Migrations;
    ...
    services.AddDbContext<IntegrationEventLogContext>(options =>
    {
        ...
        options.ReplaceService<IMigrationsAssembly, DbSchemaAwareMigrationAssembly>();
    });
```

### IUnitOfWork
You can read more about [Unit of Work](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design#implementing-unit-of-work) before continue.
We define an **IUnitOfWork** interface to easy to work with DbContext transaction.

```csharp
    public interface IUnitOfWork
    {
        IDbContextTransaction? GetCurrentTransaction();

        bool HasActiveTransaction { get; }

        Task<IDbContextTransaction?> BeginTransactionAsync();

        Task CommitTransactionAsync(IDbContextTransaction transaction);

        void RollbackTransaction();
    }
```

The library can be accessed via Nuget:
- [Juice.EF](https://www.nuget.org/packages/Juice.EF)