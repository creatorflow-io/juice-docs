---
title: "Domain"
date: 2023-04-13T10:48:09+07:00
draft: false
weight: 3
---

This layer will name `{namespace}.{feature}`.

The project tree will look like this:

```csharp
|-- Domain
|   |-- AggregrateModel
|       |-- AbcAggregrate
|           | Abc.cs
|           | IAbcRepository.cs     // suggested
|       |-- XyzAggregrate
|   |-- Commands
|       |-- CreateAbcCommand.cs
|   |-- Events
|       |-- AbcCreatedDomainEvent.cs
|   |-- CommandHandlers     // move it into API project if you are not implementing the Repository
|       |-- CreateAbcCommandHandler.cs      // to handle create Abc command (as same as Manager)
|-- DependencyInjection     // optional
|   |-- YourServiceCollectionExtensions.cs  // to register your services
```

In the tree above:
- **Abc** is an aggregarte model that implement **IAggregrateRoot<TNotification>**.

```csharp {linenos=false,hl_lines=[1,2,21],linenostart=1}
    using Juice.Domain;
    public class Abc: IAggregrateRoot<INotification>{   // INotification is an interface of MediatR
        public Abc() { } // parameterless constructor is needed for EF

        public Abc(string name, string correlationId)
        {
            Name = name;
            CorrelationId = correlationId;
        }
        
        [Key]
        public Guid Id { get; init; }
        public string Name { get; init; }
        public string CorrelationId { get; init; }

        public void UpdateName(string newName){
            var originName = Name;
            // validation for new name then update it into Name
            ...
            // add a domain event to notice that name was changed
            this.AddDomainEvent(new AbcNameChangedDomainEvent(originName, newName));
        }
    }
```

- **IAbcRepository** will define methods to CRUD Abc (and entities within its boundaries) and all you need.
- The *Commands* folder will contain commands to process specified use-cases. A command must contains:
    - **Identifier**: to identify processing object
    - **Data**: enough to process the use-case
- The *CommandHandlers* folder will contain classes to handle the commands above
    - Use **IAbcRepository** to read domain objects by **Identifier** or init domain objects from the command's data
    - Call domain method to process the use-case
    - Use **IAbcRepository** to persist data back to data backend 
