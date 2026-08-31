---
title: Asynchronous Programming - 第十二章:章魚的八爪例外：多執行緒的捕獲術
date: 2025-07-07 09:06:05
categories: 未来よりの返歌
top_img: https://github.com/CHI-KEKE/pics/blob/main/Codesss/Async/logoAsync.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Codesss/Async/logoAsync.png?raw=true
tags:
    - Asynchronous Programming
toc:
toc_number:
comments :
---

{% tabs Asynchronous%}

<!-- tab 章魚-->

在深海裡，章魚的八隻腕足總是同時探向不同的方向，纏繞著礁岩、觸碰著漩渦，也悄悄在黑暗中編織出一張未被看見的網。我們的非同步程式，就像這隻章魚，當你派出無數任務游向未知，若沒有好好等待（await），有些錯誤就會像溜走的墨汁般散開，再也無法追溯



![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-12-exception/1_octopus.png)




若選擇用同步阻塞強行捕捉，例外也會被包裹成難解的聚合結，像是八爪同時纏住獵物卻黏成一團。學會如何等待，如何攤開多執行緒裡潛藏的觸手，才是馴服這隻章魚、看清每一道潛伏例外的唯一方法。

🐙
![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-12-exception/2_three_types.png)



<!-- endtab -->

<!-- tab 實驗一：沒 await，什麼都接不到-->

```CSHARP
public static async Task ExceptionResultTest()
{
    try
    {
        TaskThatThrowsExceptionStringAsync();
    }
    catch(Exception ex)
    {
        throw;
    }
}

private static async Task<string> TaskThatThrowsExceptionStringAsync()
{
    throw new NotImplementedException("內部錯誤#####################################");
}
```

執行後什麼都沒有。錯誤直接遺失，因為 try...catch 並沒有真正等待這個 Task 的完成。async 方法呼叫後立即傳回 Task。try 區塊在呼叫後就結束了，例外發生時已離開 try 區塊。


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-12-exception/3_lose_exception.png)


<!-- endtab -->

<!-- tab 實驗二：用 .Result 或 .Wait() 阻塞取得結果-->

```CSHARP
public static async Task<string> ExceptionResultTest()
{
    try
    {
        return TaskThatThrowsExceptionStringAsync().Result;
    }
    catch(Exception ex)
    {
        throw;
    }
}

private static async Task<string> TaskThatThrowsExceptionStringAsync()
{
    throw new NotImplementedException("內部錯誤#####################################");
}

```

成功抓到錯誤。但訊息是 AggregateException: One or more errors occurred。需要看 InnerException 才知道真正錯誤。.Result 與 .Wait() 是同步阻塞。若 Task 出現例外，CLR 會將例外包成 AggregateException。這是 .NET 的多任務例外聚合機制


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-12-exception/4_aggregation.png)


<!-- endtab -->

<!-- tab 實驗三：使用 await 等待結果-->

```CSHARP
public static async Task<string> ExceptionResultTest()
{
    try
    {
        return await TaskThatThrowsExceptionStringAsync();
    }
    catch(Exception ex)
    {
        throw;
    }
}

private static async Task<string> TaskThatThrowsExceptionStringAsync()
{
    throw new NotImplementedException("內部錯誤#####################################");
}

```
直接捕捉到 NotImplementedException。訊息清楚明確，堆疊追蹤與除錯資訊完整。await 會將 Task 中發生的例外重新拋出（unwrap）。因此例外型態不被包成 AggregateException，除錯體驗佳


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-12-exception/5_await_unwrap.png)



<!-- endtab -->

<!-- tab 實驗四：Task 不回傳值也一樣-->

```CSHARP
public static async Task ExceptionResultTest()
{
    try
    {
        await TaskThatThrowsExceptionStringAsync();
    }
    catch(Exception ex)
    {
        throw;
    }
}

private static async Task TaskThatThrowsExceptionStringAsync()
{
    throw new NotImplementedException("內部錯誤#####################################");
}

```

await 發現 Task 狀態 = Faulted → 把裡面的 Exception 拿出來，catch 成功攔到例外 ， throw 表示保留原本的 stack trace，原封不動往上丟


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-12-exception/6_await_under_the_hood.png)

<!-- endtab -->

<!-- tab 多個 Task 的例外處理-->

如果一次執行多個 Task，用 .WaitAll() 或 .Wait()，所有例外都會被聚合：
```CSHARP

public static async Task ExceptionMultipleResultTest()
{
    try
    {
        MultipleTasks().Wait();
    }
    catch (AggregateException ex)
    {
        foreach (var inner in ex.InnerExceptions)
        {
            Console.WriteLine($"==============={inner.Message}=====================");
        }
        throw;
    }
}

private static async Task MultipleTasks()
{
    var tasks = new List<Task>
    {
        Task.Run(() => throw new InvalidOperationException("錯誤 A")),
        Task.Run(() => throw new NotImplementedException("錯誤 B"))
    };

    await Task.WhenAll(tasks);
}

```

若用 await Task.WhenAll()，則第一個出現的例外就會被拋出，且同樣是原型別，不是 AggregateException



![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-12-exception/7_waitall_whenall.png)


<!-- endtab -->

<!-- tab ✅ 最佳實踐-->

- 非同步永遠要 await, 除非有特別需求（如 Fire-and-Forget），否則不要只是丟出 Task
- 同時多任務，盡量用 await Task.WhenAll, 不要用 .WaitAll()，除非需要同步阻塞
- 理解聚合例外的機制, .Wait() 與 .Result 的 AggregateException 並非 bug，而是多執行緒任務例外安全聚合機制
- 在 Library 或 API 層面提供非同步介面, 讓呼叫端決定是否要 await，保留彈性



![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-12-exception/rules.png)


<!-- endtab -->

{% endtabs %}