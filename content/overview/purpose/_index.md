---
weight: 1
title: "Purposes"
date: 2023-03-29T14:44:25+07:00
---

Most modern applications look similar like this:
![Block diagram](/images/overview/block-diagram.png "Block diagram")

Your applications may have more or fewer components:
- The service-bus may or may not be available
- You may choose single or multi-tenant model
- You can use local or remote authentication/authorization services

So we wanted a simple codebase for beginners. It includes the following characteristics:
- Simple and easy to use, we can start with just what we need
- Lightweight: it must quick to start and work on low-memory environments
- Ready to integrate: it can work with third-party libraries without conflict
- Ready to use: it has some built-in services that we can use right away

JUICE is not a complete framework. It cannot help you quickly build an app with less code
but provides implementation of some extensions and small tools that you normally use to build your apps.
It also contains examples of microservices for beginners.