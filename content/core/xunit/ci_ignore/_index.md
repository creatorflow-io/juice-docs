---
title: "CI-Ignore"
date: 2023-04-11T17:21:32+07:00
draft: false
weight: 4
---

Some special tests require special conditions so we have to ignore them in the CI test.
So we can add **[IgnoreOnCIFact]** or **[IgnoreOnCITheory]** attribute to specified tests to bypass.

```csharp {linenos=false,hl_lines=[1,4,10],linenostart=1}
    using Juice.XUnit;
    public class EFTest
    {
        [IgnoreOnCIFact(TestPriority(99)]
        public async Task Shoud_run_first()
        {
            
        }

        [IgnoreOnCITheory(TestPriority(98)]
        [InlineData("SqlServer")]
        [InlineData("PostgreSQL")]
        public async Task Shoud_run_second(string provider)
        {
            
        }
    }
```

The library can be accessed via Nuget:
- [Juice.XUnit](https://www.nuget.org/packages/Juice.XUnit)