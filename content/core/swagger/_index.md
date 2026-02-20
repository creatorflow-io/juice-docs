---
title: "Swagger"
date: 2023-04-03T20:23:38+07:00
draft: false
weight: 5
---

#### Include XML comments

This is a SwaggerGenOptions' extension to include xml comments from referenced assemblies.

```csharp {linenos=false,hl_lines=[2,8],linenostart=1}
    ...
    using Juice.Extensions.Swagger;
    ...

    services.ConfigureSwaggerGen(c =>
    {
        ...
        c.IncludeReferencedXmlComments();
    });
```
The library can be accessed via Nuget:
- [Juice.AspNetCore](https://www.nuget.org/packages/Juice.AspNetCore)

#### SwaggerIgnoreFilter

To ignore model properties from Swagger document generator but keep it in serialized JSON, 
we implement **ApiIgnoreAttribute** and *ISchemaFilter* interface to remove properties
those have **ApiIgnoreAttribute** from **OpenApiSchema**

```cs
    public class SwaggerIgnoreFilter : ISchemaFilter
    {
        public void Apply(OpenApiSchema schema, SchemaFilterContext context)
        {
            ...
        }
    }
```

Register filter into DI
```csharp {linenos=false,hl_lines=[2,6,12],linenostart=1}
    ...
    using Juice.Extensions.Swagger;
    ...

    // use default config
    services.AddSwaggerWithDefaultConfigs();

    // OR configure swagger yourself
    // services.AddSwaggerGen(c =>
    // {
    //     ...
    //     c.SchemaFilter<SwaggerIgnoreFilter>();
    // });
```
The library can be accessed via Nuget:
- [Juice.AspNetCore](https://www.nuget.org/packages/Juice.AspNetCore)

#### ApiIgnore attribute
This attribute is useful to Swagger skip generate API for specified properties when combined with the above **SwaggerIgnoreFilter**.

```csharp {linenos=false,hl_lines=[1,3],linenostart=1}
    using Juice.CompnentModel;
    public class ViewMode{
        [ApiIgnore]
        public int OnlyAppearInJSON { get; set; }
    }
```

The library can be accessed via Nuget:
- [Juice](https://www.nuget.org/packages/Juice.AspNetCore)