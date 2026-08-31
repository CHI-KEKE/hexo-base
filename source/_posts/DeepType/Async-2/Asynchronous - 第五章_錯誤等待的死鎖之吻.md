---
title: Asynchronous Programming - 第五章:錯誤等待的死鎖之吻
date: 2024-08-31 10:03:05
categories: 未来よりの返歌
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-5-deadloc/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-5-deadloc/1_landing.png
tags:
    - Asynchronous Programming
toc:
toc_number:
comments :
---

{% tabs Asynchronous2%}

<!-- tab love-->


![wait](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/love.png)


他們曾經相愛，也曾經深信，只要彼此都願意等，愛就會回來

她把訊息輸入對話框，卻沒有按下傳送，想等他主動說第一句話。她想：「他如果真的在乎，就會找我。」
他打開視窗看了又看，也沒傳訊息，心想：「她如果還有感覺，就會先聯絡我。」

它們每天打開彼此的聊天室又關上，不是沒有思念，而是不願先伸出手。怕輸、怕低頭、怕被拒絕，在各自的世界靜靜地等，畫了一個誰也跨不過的等待邊界，兩個 `.Result()` 卡在彼此門前，沒有人願意給對方 `await` 的空間


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-5-deadloc/2_wait_message.png)


<!-- endtab -->

<!-- tab 為什麼 .Result 有時會讓整個程式卡住-->

[async deadlock](https://abstreamace.com/sglab/2020/06/11/%E8%B8%A9%E5%9D%91%E8%A8%98%EF%BC%9A%E5%82%B3%E8%AA%AA%E4%B8%AD%E7%9A%84-async-deadlock/)

[await 與 Task.Result/Task.Wait () 的 Deadlock 問題](https://blog.darkthread.net/blog/await-task-block-deadlock/)


在 ASP.NET 舊版框架中，若在同步方法中呼叫非同步方法

```CSHARP
var result = GetDataAsync().Result;
```

可能會遇到一個非常棘手的問題：`Deadlock`

事情是這樣發生的

- 主執行緒呼叫 `.Result`，進入「同步等待」狀態，卡住不動
- `GetDataAsync()` 裡面用 `await`，遇到像 `await Task.Delay(...)` 時，會中斷流程、把控制權讓出。
- 等待完成後，預設會「切回原本的執行緒」來繼續執行。


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-5-deadloc/3_why_dead.png)


問題來了，原本的執行緒（主執行緒）正在 `.Result` 那裡等。但同時，你要回去的執行緒被你自己卡住了

互相等待 → ❌ 死鎖

這一切的核心關鍵是：`SynchronizationContext`，它會記住「原本在哪條執行緒開始」，然後強迫 `await` 結束後一定要回來


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-5-deadloc/3_2_dead_three_steps.png)


<!-- endtab -->

<!-- tab ASP.NET Core 不再死鎖-->

ASP.NET Core 拿掉了 `SynchronizationContext`，變成

> "任務完成後，誰有空就接，不一定要回原來的執行緒"


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-5-deadloc/4_synchronizecontext.png)


`await` 後續不需要再等主執行緒空出來，所以就算主執行緒 `.Result` 卡在那，也不會阻止 `await` 的後續被別的執行緒執行，這樣就不會造成死鎖了

然而，只要用了 .Result 或 .Wait()，主執行緒就還在等待任務完成。只是這次，它不再是唯一能「收尾」的人，換句話說：主執行緒會等結果，但不是非得由它自己完成任務，因為 .NET Core 的處理方式是，任務完成後可以由任意 ThreadPool 執行緒跑完 `await` 後續的邏輯，再把結果交回 `.Result`



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-5-deadloc/5_aspnetcoore_change.png)


```csharp
class Program
{
    static void Main()
    {
        //// Main() 在 Thread 1
        Console.WriteLine($"Main start, ThreadId = {Thread.CurrentThread.ManagedThreadId}");

        // 主執行緒「在等結果」
        var result = DoAsync().Result;
        Console.WriteLine($"Result = {result}");
        Console.WriteLine("Main end");
    }

    static async Task<string> DoAsync()
    {
        Console.WriteLine($"Before await, ThreadId = {Thread.CurrentThread.ManagedThreadId}");
        await Task.Delay(1000);
        ///After await, ThreadId = 5
        Console.WriteLine($"After await, ThreadId = {Thread.CurrentThread.ManagedThreadId}");
        return "Done";
    }
}
```

<!-- endtab -->

<!-- tab 如果我用 await 而不是 .Result-->


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-5-deadloc/6_free_thread.png)


await 不會阻塞執行緒，它會把「後面要做的事」記下來（continuation），等有空的執行緒再來做，當下執行緒（例如主執行緒）就被「釋放」，可以去處理別的事，如此一來，少量執行緒也能處理大量請求 → 非同步的威力！


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-5-deadloc/7_await_til_the_end.png)





<!-- endtab -->


<!-- tab summary-->


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-5-deadloc/async-deadlock.png)


<!-- endtab -->


{% endtabs %}
