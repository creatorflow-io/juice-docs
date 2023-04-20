---
title: "Modular"
date: 2023-04-14T15:36:00+07:00
draft: false
weight: 20
---

This small feature is useful for build a monolithic application from multiple separated features.
Developers can register services and configure their modules by themselves.

Firstly, you need to implement the class that inherits from **ModuleStartup** for your features.

```csharp {linenos=false,hl_lines=[1,3,5,6,9,15],linenostart=1}
    using Juice.Modular;
    ...
    namespace Juice.Test
    {
        [Feature(Required = true)]
        public class TestModuleStartup : ModuleStartup{

            // override ConfigureServices
            public override void ConfigureServices(IServiceCollection services, IMvcBuilder mvc, IWebHostEnvironment env, IConfigurationRoot configuration)
            {
                // configure your services
            }

            // override Configure
            public override void Configure(IApplicationBuilder app, IEndpointRouteBuilder routes, IWebHostEnvironment env)
            {
                // configure application pipeline
            }
        }
    }
```

The **Feature** attribute's arguments:
- *Required*: specify this feature is required and will start automatically
- *Name*: specify your feature's name. If it's not defined, the default name will be used. 
    The default name will be namespace then replace the "**.**" with "**_**". 
    
    Ex: Your namespace is **Juice.Test** so the feature name will be **Juice_Test** by default.
- *Description*: describe your feature
- *Dependencies*: an array of dependent features
- *IncompatibleFeatures*: an array of incompatible features. The application will throw an exception on startup if incompatible features are registered, so you must try to solve.

Then in your **Program.cs**

```csharp {linenos=false,hl_lines=[1,6,16],linenostart=1}
    using Juice.Conventions.StartupDiscovery.Extensions;
    ...

    var builder = WebApplication.CreateBuilder(args);

    builder.AddDiscoveredModules();
    ...

    var app = builder.Build();

    // customize application's middlewares order
    // app.UseMultiTenant();

    // app.UseRouting();
    ... 
    app.ConfigureDiscoverdModules(app.Environment);

    ...
    app.Run();

```