---
title: "Plugins"
date: 2023-09-23T15:54:24+07:00
draft: false
weight: 6
---

We have implemented a plugin management service that supports application load and execute service from plugin by load its assembly dynamically.

{{< figure src="/images/services/Plugins-process.svg" title="Plugin sequence diagram" >}}

To implement plugin pattern, you first understanding the [AssemblyLoadContext](https://learn.microsoft.com/en-us/dotnet/core/dependency-loading/understanding-assemblyloadcontext).

#### Core concept

1. **Application**

The application will host all services including the *IPluginServiceProvider* and *IPluginsManager* services. Application accesses *Plugin* services through *IPluginServiceProvider*, and manages plugins (**Load/Unload**) via *IPluginsManager*.

2. **PluginsManager**

This service manages all plugins by Load/Unload them. Some *Shared Services* can be shared between *Application* and *Plugins* when we load them.

---
**NOTE**
- Any service tha can be called from *Application* or passed from *Application* is a shared service
- Any services shared with plugin must be excluded from the plugin directory
- We can share the service instead of its implementation classes, so we can use different implementations between *Application* and *Plugins* 
---

These services will be shared with plugin by default:
- IServiceCollection
- IConfiguration

3. **Plugin**

The plugin will implement our shared services. To avoid libraries version conflicts and easy of implementation, it should have a reference to the **Microsoft.AspNetCore.App** so *Application* and *Plugin* will share this framework.

On load, all *Startup* classes will be discovered and their *ConfigureServices()* method called with *IServiceCollection* and *IConfiguration* argurments. So the plugin can have an owned appsettings file.

After loaded, plugin will have its own *ServiceProvider*, which is useful when resolving services from the plugin.

#### Usage

To use plugin services, please call the *AddPlugins()* on *IServiceCollection* instance.

```cs
using Microsoft.Extensions.DependencyInjection;
...
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddPlugins(options =>
{
    options.AbsolutePaths = pluginPaths;
    options.ConfigureSharedServices = (services, sp) =>
    {
        // SharedService will be shared to plugins
        services.AddScoped(sp1 =>
        {
            var s = sp.GetRequiredService<SharedService>();
            var l = sp.GetRequiredService<ILoggerFactory>().CreateLogger("shared");
            l.LogInformation("Shared service created {0}", s.Id);
            return s;
        });
    };
});
```

#### Plugin implementation

The quote below is a simple project file of a plugin. Please refer to [Dotnet tutorials](https://learn.microsoft.com/en-us/dotnet/core/tutorials/creating-app-with-plugin-support) for details.

```cs
// Plugin.csprj
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFrameworks>net6.0,net7.0</TargetFrameworks>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <EnableDynamicLoading>true</EnableDynamicLoading> <!-- This element is important -->
  </PropertyGroup>
  
  <ItemGroup Label="AspNetCore">
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
  </ItemGroup>

  
  <ItemGroup>
    <ProjectReference Include="..\Juice.Plugins.Tests.Common\Juice.Plugins.Tests.Common.csproj" />

    <ProjectReference Include="..\Juice.Plugins.Tests.PluginBase\Juice.Plugins.Tests.PluginBase.csproj">
      <Private>false</Private> <!-- This element is important -->
      <ExcludeAssets>runtime</ExcludeAssets> <!-- This element is important -->
    </ProjectReference>
  </ItemGroup>

</Project>
```

```cs
// Plugin Startup.cs
public class Startup
{
    public void ConfigureServices(IServiceCollection services, IConfiguration configuration)
    {

        services.AddLogging(builder =>
        {
            builder.AddConsole();
            builder.AddConfiguration(configuration.GetSection("Logging"));
        });

        services.AddScoped<MessageService>();
        services.AddScoped<ICommand, HelloCommand>();
    }
}
```

The full source code for this sample is available on [github](https://github.com/creatorflow-io/Juice/tree/master/services/plugins/test/).

The library can be accessed via Nuget:
- [Juice.Plugins](https://www.nuget.org/packages/Juice.Plugins)