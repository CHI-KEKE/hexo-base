---
title: Asynchronous - 第十五章:Host Task
date: 2026-01-05 08:23:05
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

<!-- tab Task-based Asynchronous Pattern（TAP）中的 Hot Task 行為-->


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-15-hottask/1_real_async.png)


TAP 的核心不是「延遲執行」，而是「把等待時間變成可並行」，`async` 方法在呼叫的那一刻就開始執行，是因為非同步的價值在於「提早把 I/O 交出去跑」，而不是等你 `await` 才開始，我們用一個例子來觀察

## Step 1：呼叫 DownloadFileAsync(url)
```csharp
getContentTasks.Add(DownloadFileAsync(url));
```
這一步不是僅是宣告任務，而是真的執行這個方法


## Step 2：進入 DownloadFileAsync 方法本體
```csharp
async Task DownloadFileAsync(string url)
{
    await httpClient.GetAsync(url);
}
```
方法一進來就跑，直到遇到第一個 await


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-15-hottask/2_2_steps.png)



## Step 3：遇到 HttpClient.GetAsync()

這是一個 I/O 非同步，會立刻把 HTTP 請求交給 OS / 網路層，並開始下載，此時 `Task` 進入「未完成，但已啟動」狀態

## Step 4：回傳一個「已經在跑的 Task」

`DownloadFileAsync(url)` 回傳的是一個 Hot Task，背後的下載 早就開始了，有沒有 `await`，只影響「你什麼時候等結果」，不影響「事情有沒有開始做」



![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-15-hottask/3_return_task_for_result.png)



<!-- endtab -->

<!-- tab 如果真的「不想馬上開始」會怎樣-->

TAP 不給你煞車，是因為煞車應該由「架構層」負責，不是方法本身，如果需要控制啟動時機，就不該直接呼叫 async method，而是用 `Func<Task>`、factory method 或 queue / semaphore


```csharp
List<Func<Task>> jobs = new();

jobs.Add(() => DownloadFileAsync(url));

// 真的要跑時才呼叫
await jobs[0]();
```


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-15-hottask/4_no_brake.png)



<!-- endtab -->

<!-- tab 為什麼 new Task(...) 不會自動執行-->


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-15-hottask/5_cold_task_shell.png)


`new Task(...)` 的存在，是為了讓你「先決定什麼時候跑」，而不是「一建立就失控地跑」。

因為用的是 Task constructor，此時 `Task`的狀態是 `TaskStatus.Created`，表示 CLR 知道這是一個 `Task`，但 `ThreadPool` 完全不知道它的存在，這是「cold task」設計，預設不會開始

```csharp
var task = new Task<int>(() => {
    Thread.Sleep(...);
    return time;
});
```

這種類型的 `Task` 只有當你 `.Start()` 時才會真正執行，完全符合「建立」與「開始」分離的模式，當我們手動 new 一個 Task，只是一個「殼」，通常用在背景工作、手動控制並行度，非 async / blocking 邏輯要放進 `Task` 執行

```csharp
// 這一行才會把工作丟進 ThreadPool，排程執行 delegate，Task 狀態轉為 Running
task.Start();
```

| 建立方式                  | 是否自動開始            | 用途                 |
| --------------------- | ----------------- | ------------------ |
| `DownloadFileAsync()` | ✔ 會自動開始（HOT task） | TAP async 非同步，立即啟動 |
| `Task.Run(() => ...)` | ✔ 會自動開始（HOT）      | 把 CPU-bound 放到背景執行 |
| `new Task(() => ...)` | ✘ 不會開始（COLD task） | 需要手動控制 Run/Start   |

👉 async method 本質是 HOT TASK，一呼叫就開始
👉 new Task(...) 是 COLD TASK，必須 Start() 才開始


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-15-hottask/6_table.png)


<!-- endtab -->



<!-- tab summary-->


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-15-hottask/7_final.png)


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-15-hottask/async-hottask.png)



<!-- endtab -->


{% endtabs %}