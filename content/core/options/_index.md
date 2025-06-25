---
title: "Options"
date: 2023-04-03T20:22:57+07:00
draft: false
weight: 4
---

#### Mutable Options

To change options in the application, we can use *IOptionsMutable\<T\>* interface:

```csharp {linenos=false,linenostart=1}
    public interface IOptionsMutable<out T> : IOptionsSnapshot<T> where T : class, new()
    {
        // Update T field by field
        Task<bool> UpdateAsync(Action<T> applyChanges);
    }
```

To use *IOptionsMutable\<T\>* we must also register some required services.

```csharp {linenos=false,hl_lines=[2,5,11],linenostart=1}
    ...
    using Juice.Extensions.Options.DependencyInjection;
    ...
    // your changed options will be write to {WorkingDirectory}/appsettings.Development.json
    services.UseOptionsMutableFileStore("appsettings.Development.json");

    // OR separated appsettings file for strongly typed
    // services.UseOptionsMutableFileStore<Options>($"appsettings.Separated.Development.json");

    // configure your Options class before use
    services.ConfigureMutable<Options>(configuration.GetSection("Options"));

```

After that, we can inject *IOptionsMutable\<Options\>* and use it like *IOptionsSnapshot\<Options\>*.
We can also call *UpdateAsync* method to update options.

```csharp
    var options = serviceProvider.GetRequiredService<IOptionsMutable<Options>>();
    await options.UpdateAsync(o => {
        //o.Something = "something else";
    });
```

#### Per-tenant Options

Use *ITenantsOptions\<T\>* to access per-tenant configured options.

```cs
    // Per-tenant options snapshot
    public interface ITenantsOptions<T> : IOptionsSnapshot<T> where T : class, new()
    {
    }
```

We will register tenant configuration services to use per-tenant options accessor.
```csharp {linenos=false,hl_lines=[2,6, 9],linenostart=1}
    ...
    using Juice.Extensions.Options.DependencyInjection;
    ...

    // register tenant configuration services
    services.AddTenantsConfiguration().AddTenantsJsonFile("appsettings.Development.json");

    // configure typed options
    services.ConfigureTenantsOptions<Options>("Options");

    ...

    // use ITenantsOptions like IOptionsSnapshot
    var options = serviceProvider.GetRequiredService<ITenantsOptions<Options>>();

```
NOTE: It requires [ITenant service]({{< ref "core/multitenant/_index.md" >}})
#### Per-tenant Mutable Options

This is combined of per-tenant options and mutable options.
```cs
    public interface ITenantsOptionsMutable<T> : ITenantsOptions<T>, IOptionsMutable<T>
        where T : class, new()
    {
    }
```

```csharp {linenos=false,hl_lines=[2,6, 9, 12],linenostart=1}
    ...
    using Juice.Extensions.Options.DependencyInjection;
    ...

    // register tenant configuration services
    services.AddTenantsConfiguration().AddTenantsJsonFile("appsettings.Development.json");

    // register multi-tenant mutable configuration source
    services.UseTenantsOptionsMutableFileStore("appsettings.Development.json");

    // configure multi-tenant mutable options
    services.ConfigureTenantsOptionsMutable<Options>("Options");

    ...

    // use ITenantsOptionsMutable like IOptionsSnapshot
    var options = serviceProvider.GetRequiredService<ITenantsOptionsMutable<Options>>();

    // now we can update Options
    await options.UpdateAsync(o => {
        //o.Something = "something else";
    });

```
NOTE: It requires [ITenant service]({{< ref "core/multitenant/_index.md" >}})

The library can be accessed via Nuget:
- [Juice.Extensions.Options](https://www.nuget.org/packages/Juice.Extensions.Options)