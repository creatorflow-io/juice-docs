---
title: "Measurement"
date: 2024-09-30T08:36:48+07:00
draft: false
weight: 7
---

This service is useful for measure the execution time of a method by create the execution scopes and checkpoints in your code, and then print the details of execution time step by step to determine where is the slowly code.
You can use it with new transient instance to meansure inside a class, method, or use dependency injection pattern to meansure cross services.

```csharp
    ITimeTracker timeTracker = new TimeTracker();
    using (timeTracker.BeginScope("Test"))
    {
        // Do something
        await Task.Delay(12);
        timeTracker.Checkpoint("Checkpoint 0");

        using (timeTracker.BeginScope("Inner Test"))
        {
            // Do something
            await Task.Delay(15);

            using (timeTracker.BeginScope("Inner Inner Test"))
            {
                // Do something
                timeTracker.Checkpoint("Checkpoint 1.1");
                await Task.Delay(20);
                timeTracker.Checkpoint("Checkpoint 1.2");
            }
        }
        timeTracker.Checkpoint("Checkpoint 1");
        using (timeTracker.BeginScope("Inner Test 2"))
        {
            // Do something
            await Task.Delay(10);
            timeTracker.Checkpoint("Checkpoint 1.3");
            timeTracker.Checkpoint("Checkpoint 1.4");
        }
    }
    Console.WriteLine(timeTracker.ToString());

    // Output:
     _______________________________________________________
    |         Scope         |  Depth   |    Elapsed Time    |
    |-------------------------------------------------------|
    | « Test                |    1     |        ›  429.7 µs |
    |  › Checkpoint 0       |    1     |         + 16.21 ms |
    |  « Inner Test         |    2     |        ›  16.67 ms |
    |   « Inner Inner Test  |    3     |        ›  31.08 ms |
    |    › Checkpoint 1.1   |    3     |           + 4.2 µs |
    |    › Checkpoint 1.2   |    3     |         + 31.89 ms |
    |   » Inner Inner Test  |    3     |           32.10 ms |
    |  » Inner Test         |    2     |           46.77 ms |
    |  › Checkpoint 1       |    1     |         + 46.83 ms |
    |  « Inner Test 2       |    2     |        ›  63.46 ms |
    |   › Checkpoint 1.3    |    2     |         + 15.07 ms |
    |   › Checkpoint 1.4    |    2     |           + 2.1 µs |
    |  » Inner Test 2       |    2     |           15.07 ms |
    | » Test                |    1     |           78.13 ms |
    |-------------------------------------------------------|
    | Total                 |          |           79.36 ms |
    '-------------------------------------------------------'
```

Inside the **Elapsed Time** column:
- '›' symbol means the time elapsed from begin of the measurement
- '+' symbol means the time elapsed from begin of the scope or the last checkpoint