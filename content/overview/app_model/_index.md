---
weight: 2
title: "Application models"
date: 2023-03-31T10:33:27+07:00
---

#### Monolithic

If you are starting to build an application without any existing product reference or new products without an idea to start with, monolithic architecture may be a good fit. Be careful with microservices if you are not sure about your product’s size, how many users you have to serve or what specific services need to scale… because it can be complicated and unnecessary.

The most common problems when we deploy a monolithic application are:

- Synchronizing settings between servers. It can be done manually or configured centrally (via shared folder, shared DB…)
- Shared sessions between servers can be made easy by using distributed cache
- Distributed logs are hard to mine
- In most cases, we have to clone large application to scale out instead of components. It is time to separate components into microservices

And in large applications, development also matters. Many developers will work independently but consistently, so we must split a big project into many sub-projects for features then use it as reference packages in final project. A modular model can be a good choice; all developers will register their services and configuration at module startup.

#### Microservices

Sometimes we need to rebuild all dependent projects to ensure that their references have the same version and the final application will work. Whenever we have a small change, no one can guarantee that the change will work for other features. We only know when it happens and not during build time. So a small change can lead to a large app that must be tested. It can affect multiple teams at once instead of a specific team.

Besides the problem of scaling the processing capacity of the monolithic application, we may need to scale up more human resources than necessary.

Microservices are needed when:

- You need to scale up your system or part of it to serve more users
- You don’t have many people to maintain and develop the existing project or you have many people to start a new project and deliver on time
- You want to integrate with 3rd party or internal services easily

Unfortunatelly, this model has even more problems that need to be resolved besides the traditional problems of monolithic app:
- Notifications between services require an external event bus instead of an internal bus.
These integration events are added by the domain business follow and sent after the domain process success
- Integrated events may be duplicated lead to repeated event processing

#### SaaS

Now a day, SaaS is growing up quickly to help ICT companies can provide their products and services to many customers rapidly and easily.
We can provice multi-tenant services using infrastructure, in-app implementation, or mixed mode.

In most cases, we need separte tenant configuration, data, and resources. So we have common problems:
- Tenant identification
- Configuration/Option by tenant but centralized
- Data and resources can be either separated or shared between tenants

#### How we can help

Choose the development model according to actual needs and conditions. We provide several application template:
- MVC
- API
- Modular: register and work as a module in MVC and API app
- Microservice: work independently