---
title: "Configuration"
date: 2023-04-03T20:23:10+07:00
draft: false
weight: 3
---

#### Per-tenant configuration
If you are trying to develop a multi-tenant application, you may need to access the tenant configuration.
So we provide *ITenantsConfiguration* interface to access tenant configuration and a default JSON configuration source
that will read configuration from *{ConfigurationDirectory}/tenants/{TenantIdentifier}/{FileName}*.

```
    public interface ITenantsConfiguration : IConfiguration
    {
    }
```

The *ITenantsConfiguration* is inherited from *IConfiguration*. So after registering the service on application startup
you can inject it and use like *IConfiguration*.

```csharp {linenos=false,hl_lines=[2,6],linenostart=1}
    ...
    using Juice.Extensions.Configuration;
    ...
    // register service and configuration source to access 
    // {WorkingDirectory}/tenants/{TenantIdentifier}/appsettings.Development.json
    services.AddTenantsConfiguration().AddTenantsJsonFile("appsettings.Development.json");
```

NOTE: It requires [ITenant service]({{< ref "core/multitenant/_index.md" >}})

You can also implement *ITenantsConfigurationSource* interface to use a custom configuration source like DB, gRPC...

The library can be accessed via Nuget:
- [Juice.Extensions.Configuration](https://www.nuget.org/packages/Juice.Extensions.Configuration)

#### Scalared configuration

We also provide *IConfigurationSection* extension methods to get scalared config so you can access a dictionary configuration.

```csharp {linenos=false,hl_lines=[2,15],linenostart=1}
    ...
    using Juice.Extensions.Configuration;
    ...

    // appsettings.json
    // "Options": {
    //      "Dict": {
    //          "A": 2,
    //          "B": false
    //      }
    // }

    services.Configure<ScalaredOptions>(options =>
    {
        var config = configuration.GetSection("Options").GetScalaredConfig<ScalaredOptions>();
        if (config != null)
        {
            options.Dict = config.Dict;
        }
    });
```

The library can be accessed via Nuget:
- [Juice](https://www.nuget.org/packages/Juice)
