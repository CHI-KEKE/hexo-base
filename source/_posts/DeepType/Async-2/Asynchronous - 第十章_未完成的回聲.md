---
title: Asynchronous Programming - 第十章:未完成的回聲
date: 2025-07-04 09:22:05
categories: 未来よりの返歌
top_img: https://github.com/CHI-KEKE/pics/blob/main/Codesss/Async/logoAsync.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Codesss/Async/logoAsync.png?raw=true
tags:
    - Asynchronous Programming
toc:
toc_number:
comments :
---

[簡介.NET 4.0 的多工執行利器 --Task](https://blog.darkthread.net/blog/net4-task/)

[C# 學習筆記：多執行緒 (6) - TPL](https://www.huanlintalk.com/2013/06/csharp-notes-multithreading-6-tpl.html)


{% tabs 未完成的回聲%}

<!-- tab 派遣出去的夜-->

在每一個任務被派遣出去的夜裡，總有一些聲音，無法即刻歸來。它們在平行的執行緒裡交錯穿行，有的抵達了終點，返回一聲輕快的完成；有的在等待中腐蝕，成為無聲的阻塞；有的半途拋下誓言，消失在取消的荒野裡。你以為程式碼寫好了，流向已然明朗，卻不知每個 Task 都像流浪的信件，封存著可能，也埋藏著未竟


![night](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/night.png)



TPL，不只是一套平行的工具，它是一場關於 等待與命運 的實驗，每一個 WaitAll、WhenAny、Result，都是開啟分支與結局的咒語。當結果未歸，回聲仍在。在執行緒的深處，有些任務，終將完成；有些回聲，永遠未完


<!-- endtab -->

<!-- tab 簡單的建立一個 Task-->

先用最簡單的 Task.Run 建立一個任務，看看它跟一般程式碼的執行順序有什麼不一樣。

```CSHARP

static void Main(string[] args)
{
    SimpleTask();
}

public static void SimpleTask()
{
    Task.Run(() =>
    {
        Thread.Sleep(1000);
        Console.WriteLine("Try Task!!!!!!!!");
    });

    Console.WriteLine("Main Thread!!!!!");
}

```

執行結果
```bash
Main Thread!!!!!
```

為什麼只印出 Main Thread!!!!!？

這是因為 Task 預設是 非同步（Asynchronous） 執行的，你叫他去做事（開背景工作）後，主程式不會停下來等他做完。主程式執行緒繼續往下跑，直接把 Main Thread!!!!! 印出來。而背景執行的 Task 需要一點時間（Thread.Sleep(1000)），結果來不及印，主程式就跑完關閉了

Task.Run 的本質是「把工作丟給別的執行緒去做，自己不等結果就繼續往下跑」，所以如果主程式先結束，背景任務根本來不及完成

1. Main() 呼叫 SimpleTask()，目前還在「主執行緒（Main Thread）」上跑
2. Task.Run(...) 被呼叫，CLR 把 lambda 內容丟到 Thread Pool，這一步「只是在排程任務」，不是在執行內容
3. Task.Run 立刻回傳，主執行緒完全沒有等，Task 還在背景準備或剛開始跑
4. 主執行緒立刻執行並輸出 Console.WriteLine("Main Thread!!!!!")
5. SimpleTask() 結束 → Main() 結束，Console 應用程式直接關閉
6. 背景 Task 還在 Sleep，就被強制終止


![night](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-10-tpl/2_quietly_disappear.png)



<!-- endtab -->

<!-- tab 那要怎麼「等他」？-->

一個方法是 Main 多加一個 `Console.Read()` 或 `Console.ReadLine()`，讓主執行緒在結束前卡住，等你看完輸出

```CSHARP

static void Main(string[] args)
{
    SimpleTask();
    Console.Read(); // 加這行
}

```

✅ 執行結果
```bash
Main Thread!!!!!
Try Task!!!!!!!!
```

因為多了 `Console.Read()`，程式不會馬上結束，所以等到背景任務跑完後，也會看到 Try Task!!!!!!!!。



![night](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-10-tpl/3_consoleread_stop_main.png)



<!-- endtab -->

<!-- tab 等待兩個任務-->

現在我們試試看同樣的概念，開兩個 Task，然後確保兩個都做完後再繼續執行主流程

```CSHARP

public static void DoubleTask()
{
    var task1 = Task.Run(() =>
    {
        Thread.Sleep(2000);
        Console.WriteLine("Task 1 完成溜");
    });

    var task2 = Task.Run(() =>
    {
        Thread.Sleep(3000);
        Console.WriteLine("Task 2 完成溜");
    });

    Task.WaitAll(task1, task2);

    Console.WriteLine("Main Thread!!!!!");
}

```

看到上一個例子，我們知道 Task 開始執行我們就不會停在那邊等他完成，會繼續跑主流程，因此這邊我們用 WaitAll 阻塞，預期上，直到 task1、task2 完成前，Main Thread 不會冒出來

✅ 執行結果
```bash
Task 1 完成溜
Task 2 完成溜
Main Thread!!!!!
```

當你同時啟動多個 Task 時，主執行緒一樣不會等它們。Task.WaitAll 是一個「同步阻塞（Blocking）」的方法，會卡在那裡，直到你列出的所有 Task 都執行完才往下跑


![night](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-10-tpl/4_task_waitall.png)



<!-- endtab -->

<!-- tab 等待結果回傳再繼續跑-->

```CSHARP

public static void ResultTest()
{
    var taskResult = Task.Run(() =>
    {
        Thread.Sleep(2000);
        return "Done!!";
    });

    var sw = new Stopwatch();
    sw.Start();
    Console.WriteLine(taskResult.Result);
    sw.Stop();
    Console.WriteLine($"Time : {sw.ElapsedMilliseconds}");
}

```

✅ 執行結果
```PLAINTEXT
Done!!
Time : 2000ms
```

taskResult 是一個 Task 執行狀況，它裡面包了一個「還沒做好的結果」。當你呼叫 .Result 時，程式會 阻塞（Blocking），如果結果還沒算好，就會在那裡等。等到結果算好了，就把值回傳給你


![night](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-10-tpl/5_task_result.png)


<!-- endtab -->

<!-- tab 實驗：觀察 Task 的執行狀態-->

- `Task.WhenAny` 怎麼做多任務競賽（誰先完成）
- `CancellationToken` 怎麼拋出取消例外
- `try/catch/finally` 怎麼正確分辨 `Completed`、`Canceled`、`Faulted` 三種狀態
- 怎麼把同步輸入（`Console.ReadKey`）包成 `Task` 跟 `Delay` 比賽

<br>

## 1️⃣ 先準備好取消功能

```CSHARP
var cts = new CancellationTokenSource();
var token = cts.Token;
```

## 2️⃣ 開一個 Task，裡面邊等輸入邊計時

用 `Console.ReadKey()` 讓使用者按 A/B/C 其中一個按鍵。每秒檢查一次，用 `Task.WhenAny` 確認使用者輸入 or 時間到哪一個先


![night](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-10-tpl/6_whenany.png)



## 3️⃣ 如果輸入了，就依輸入執行不同分支

- A：正常完成，回傳 "OK"
- B：拋例外，模擬異常狀況
- C：呼叫取消，丟出 OperationCanceledException
- 其他：回傳 Unknown

## 4️⃣ 5 秒都沒輸入就回傳 "No Input!"

這時 Task 也算是正常完成

## 5️⃣ 主流程 try/catch/finally 分別處理三種可能結果

- OperationCanceledException → Canceled
- 其他 Exception → Faulted
- 正常 → Completed

<br>

## 完整實作

```CSHARP

public static async Task TaskStatusTest()
{
    var cts = new CancellationTokenSource();
    var token = cts.Token;
    System.Console.WriteLine("Please Choose a Status : A : Completed, B : Fault : C : Cancel");

    try
    {
        var result = await Task.Run(async () =>
        {
            // 因為 ReadKey 是同步阻塞的，不丟到背景執行緒，async 設計會破功
            var inputTask = Task.Run(() => System.Console.ReadKey()); // 讓他跑, 不 await
            for (int i = 0; i < 5; i++)
            {
                System.Console.WriteLine($"Waiting for the :{i + 1} sec...");
                var deplayTask = Task.Delay(1000); // 不 await void
                var completedTask = await Task.WhenAny(inputTask, deplayTask); // await 取回 真正先完成的那個 Task, 而非裁判自己

                if (completedTask == inputTask)
                {
                    var userInputResult = inputTask.Result;
                    switch (userInputResult.Key)
                    {
                        case ConsoleKey.A:
                            return "OK";
                        case ConsoleKey.B:
                            throw new ApplicationException("MyMyException");
                        case ConsoleKey.C:
                            cts.Cancel();
                            token.ThrowIfCancellationRequested();
                            return "Cancel";
                        default:
                            return "Unknown Input";
                    }
                }
            }

            return "No Input!";
        }, token);

        System.Console.WriteLine("Completed! Result={0}", result);
    }
    catch (OperationCanceledException)
    {
        System.Console.WriteLine("Canceled!");
    }
    catch (Exception ex)
    {
        System.Console.WriteLine("Faulted!");
        System.Console.WriteLine("Error: {0}", ex.Message);
    }
    finally
    {
        System.Console.WriteLine("Async Run...");
    }
}
```


![night](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-10-tpl/7_complete_falted_cancelled.png)


<!-- endtab -->

<!-- tab 結果與分析-->


Task 沒有「狀態設定器」，只有「結果推論器」，CLR 只是根據你離開方法的方式，自動幫你分類。只要你看到一個 Task 是 Completed，那代表「裡面的程式碼真的沒有炸掉」。這是 async/await 能被信任的根本原因，Task 的狀態是由「你怎麼結束那段程式碼」決定的!


## 🔵 1. 什麼都不輸入 → 正常結束

```bash
Please Choose a Status : A : Completed, B : Fault : C : Cancel
Waiting for the :1 sec...
Waiting for the :2 sec...
Waiting for the :3 sec...
Waiting for the :4 sec...
Waiting for the :5 sec...
Completed! Result=No Input!
Async Run...
```

Console.ReadKey() 沒有輸入，delayTask 每次都先完成
5 次迴圈後，直接回傳 "No Input!"，代表任務是正常完成的，狀態是 Completed


![night](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-10-tpl/8_completed.png)



## 🟢 2. 5 秒內輸入 A → 正常完成

```bash
Please Choose a Status : A : Completed, B : Fault : C : Cancel
Waiting for the :1 sec...
Waiting for the :2 sec...
Waiting for the :3 sec...
Waiting for the :4 sec...
aCompleted! Result=OK
Async Run...
```

Console.ReadKey() 輸入 A 後，inputTask 先完成。Task 直接走到 return "OK"。狀態是 Completed。

## 🔴 3. 5 秒內輸入 B → 發生例外

```bash
Please Choose a Status : A : Completed, B : Fault : C : Cancel
Waiting for the :1 sec...
Waiting for the :2 sec...
BFaulted!
Error: MyMyException
Async Run...
```

B 會執行 throw new ApplicationException。這會讓 Task 變成 Faulted 狀態（裡面有 Exception）。進入 catch (Exception) 區塊，印出錯誤訊息。


![night](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-10-tpl/9_falted.png)


## 🟡 4. 5 秒內輸入 C → 取消

```bash
Please Choose a Status : A : Completed, B : Fault : C : Cancel
Waiting for the :1 sec...
Waiting for the :2 sec...
CCanceled!
Async Run...
```

輸入 C 時，呼叫 cts.Cancel() 並且 ThrowIfCancellationRequested()。這會拋出 OperationCanceledException。Task 變成 Canceled 狀態。進入 catch (OperationCanceledException)


![night](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-10-tpl/10_cancelled.png)


<!-- endtab -->



<!-- tab summary-->


![night](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-10-tpl/async-10-tpl.png)



<!-- endtab -->



{% endtabs %}