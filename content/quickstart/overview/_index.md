---
title: "Overview"
date: 2023-04-12T21:18:43+07:00
draft: false
weight: 1
---

In this chapter, we are going to start using `Juice` to build applications following these patterns:
- Microservice with DDD/CQRS pattern
- Monolithic modular application

However, if we have a reasonable approach, both of them may have the same structure and implementations but are different in the final target.
So we will consider the following similar components (equivalent to separate sub-projects) before returning to focus on the patterns above.
- Domain
- Infrastructure
- Api

Please follow up [Layers in DDD microservices](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/ddd-oriented-microservice#layers-in-ddd-microservices) for more information and **be careful when considering using the DDD and/or CQRS pattern**. The directory tree will look like this:

```csharp
|-- src
|   |-- {namespace}.{feature}   // Juice.Example(.Domain)
|   |-- {namespace}.{feature}.{infrastructure}  // Juice.Example.EF
|   |-- {namespace}.{feature}.Api.Contracts  // Juice.Example.Api.Contracts
|   |-- {namespace}.{feature}.Api  // Juice.Example.Api
|   |-- {namespace}.{feature}.App  // Juice.Example.App (target Microservice)
|   |-- {namespace}.{feature}.Module  // Juice.Example.Module (target Monolithic modular)
|   |-- {namespace}.{feature}.Extensions.{extension}   // optional
|-- test
|   |-- {namespace}.{feature}.Test  // Juice.Example.Test
```

