---
title: "Priority"
date: 2023-04-11T17:20:57+07:00
draft: false
weight: 3
---

Here is an approach for in-order testing.

First, we add the **[TestCaseOrderer]** attribute to the xUnit test class then add the **[TestPriority]** attribute to the test method. The test method with higher Priority will run first.

```csharp {linenos=false,hl_lines=[1,2,5,11],linenostart=1}
    using Juice.XUnit;
    [TestCaseOrderer("Juice.XUnit.PriorityOrderer", "Juice.XUnit")]
    public class EFTest
    {
        [Fact(TestPriority(99)]
        public async Task Shoud_run_first()
        {
            
        }

        [Fact(TestPriority(98)]
        public async Task Shoud_run_second()
        {
            
        }
    }
```

The library can be accessed via Nuget:
- [Juice.XUnit](https://www.nuget.org/packages/Juice.XUnit)