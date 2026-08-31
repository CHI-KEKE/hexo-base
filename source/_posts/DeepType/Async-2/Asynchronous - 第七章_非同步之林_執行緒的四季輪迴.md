---
title: Asynchronous Programming - 第七章:非同步之林：執行緒的四季輪迴
date: 2024-10-26 09:22:05
categories: 未来よりの返歌
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-7-thread-cycle/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-7-thread-cycle/1_landing.png
tags:
    - Asynchronous Programming
toc:
toc_number:
comments :
---

{% tabs Asynchronous2%}

<!-- tab forest-->

在這片名為非同步之林的廣闊森林裡，藏著一家智慧餐廳。這間餐廳一天到晚接單不斷，每張訂單都是一個任務，廚房裡的人力有限，卻得讓所有客人都吃得快又好、不讓誰在門口久等，還得同時兼顧廚房資源的高效運用。


![forest](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/forest2.png)


這家餐廳裡，有忙碌的櫃台接待員（Main Thread），有後廚大廚團隊（Worker Threads），有專門通知外送到達的取餐小組（I/O Completion Threads），偶爾，老闆還會臨時請來私廚（Custom Thread），用一場又一場執行緒的調度，撐起這座非同步之林裡飲食服務


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-7-thread-cycle/2_multi_threads.png)



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-7-thread-cycle/3_three_roles.png)


<!-- endtab -->

<!-- tab Main Thread-->

## 🎵 Main Thread = 櫃台接待員

負責第一時間接待客人、記錄每張訂單、回答客人問題。櫃台小姐（Main Thread）不能被卡住，一卡住，後面來的客人全部排隊大塞車！所以櫃台只是登記完，馬上就把訂單丟給後廚（ThreadPool）去做。

✅ 在 ASP.NET 裡，Main Thread 就像每次收到使用者的 HTTP 請求的 Request Thread；桌面程式裡就是 UI 執行緒，負責畫面互動。


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-7-thread-cycle/4_main_thread.png)




<!-- endtab -->

<!-- tab Worker Threads-->

## 🎵 Worker Threads = 後廚大廚團隊

這是餐廳的主要戰力，一群大廚（Worker Threads）在後台幫你煮菜（跑 CPU Bound 工作）。廚師是可彈性調度的，客人多時，廚房會動態叫更多人加班（ThreadPool 的 Hill-Climbing 調節）。客人少時，廚房有些廚師會先回家休息（釋放執行緒，避免浪費資源）。

✅ 例如 Task.Run 就是告訴櫃台「這道菜需要人現場煮，丟給後廚去做」。


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-7-thread-cycle/5_worker_threads.png)


<!-- endtab -->

<!-- tab I/O Completion Threads-->

#### 🎵 I/O Completion Threads = 外送取餐通知

有些菜是不用自己煮的，比如外包餐點（非同步 I/O），櫃台下單後是外面的合作店家準備。餐點做好後，不需要廚師跑去取餐，而是外送通知來了（OS 通知），專門的通知員（I/O Completion Thread）負責回報「外送到了」。廚師不需要卡著等外送，省下人力。

✅ 例如 HttpClient、FileStream.ReadAsync，背後都是 OS 幫你監聽，準備好再通知，不佔用廚師（Worker Thread）


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-7-thread-cycle/6_complete_thread.png)



<!-- endtab -->

<!-- tab Custom Thread-->

## 🎵 Custom Thread = 老闆找來的私廚

有些老闆想要做特殊料理（例如長期監控、背景資料同步），就會另外請一個私廚（new Thread()）。這位私廚不屬於餐廳原本的後廚團隊（ThreadPool），所以沒辦法自動被餐廳調度。你要自己負責這位私廚什麼時候上班、什麼時候休息，還要自己付錢，請多了還會佔掉廚房空間。

✅ 通常只有少數需要長期跑、不想被 ThreadPool 管理的情境才會用到


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-7-thread-cycle/7_custom_thread.png)




| 類型                         | 角色                                                 | 特點                                                |
| -------------------------- | -------------------------------------------------- | ------------------------------------------------- |
| **Main Thread (主執行緒)**     | 譬如 `ASP.NET` 的 Request 處理執行緒、桌面程式的 UI 執行緒          | 不能被卡太久，否則 UI 畫面卡住、Web Request 超時                  |
| **Worker Threads**         | ThreadPool 的主要執行緒，跑 `Task.Run`、`QueueUserWorkItem` | 處理 CPU Bound 的工作或短時間非同步任務                         |
| **I/O Completion Threads** | 專門處理非同步 I/O 的回呼                                    | 像 File IO、Socket、HttpClient 用 OS 的非同步 API，完成後這邊回呼 |
| **Custom Thread**          | 你自己 new `Thread()` 建的獨立執行緒                         | 不進 ThreadPool，不受 ThreadPool 管理，成本高，一般很少需要         |




![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-7-thread-cycle/8_thread_role_summary.png)



<!-- endtab -->

<!-- tab 「人力底線」與「人力上限」-->

## 🎵 「人力底線」與「人力上限」

在這間 24 小時營業的智慧餐廳裡，負責烹調訂單的就是「後廚」——也就是執行緒集區（`ThreadPool`）。
即使廚房裡的工作（任務）數量會忽高忽低，老闆（`.NET Runtime`）還是得先準備好人力調度的兩條底線：

- 1️⃣ 人力底線（`GetMinThreads`）
- 2️⃣ 人力上限（`GetMaxThreads`）


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-7-thread-cycle/9_thread_pool_adjust.png)



## 🍳 後廚人力底線 — GetMinThreads

`ThreadPool.GetMinThreads(out workerThreads, out completionPortThreads)`

這個設定就像是規定「後廚最少要留幾位廚師 standby」。就算此刻沒有新訂單進來，也要先留住這些廚師，確保當訂單瞬間湧入時，不會因為重新招人（建立執行緒）而延誤上菜（執行任務）。

在 GetMinThreads 中有兩個維度：

- `workerThreads`：CPU Bound 工作（純運算）
- `completionPortThreads`：I/O Bound 工作（非同步 I/O）

```CSHARP

int minWorker, minIOC;
ThreadPool.GetMinThreads(out minWorker, out minIOC);
minWorker.Dump();
minIOC.Dump();

/// 20
//// 1
```

這台機器預設至少會有 20 個背景工作執行緒隨時待命處理 CPU 密集任務，以及至少 1 個 I/O 完成執行緒，用於非同步 I/O 事件的回呼，這個最小值通常跟 CPU 核心數有關（包含超執行緒 Hyper-Threading）。
目的在於降低執行緒從無到有的建立成本，換取在高峰期能即時出菜。



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-7-thread-cycle/9_1_min_threads.png)



## 🍳 2️⃣ 後廚人力上限 — GetMaxThreads

`ThreadPool.GetMaxThreads(out workerThreads, out completionPortThreads)`

另一個極限就是「最多能請多少臨時工進廚房幫忙」，也就是執行緒集區能夠擴充到的最大執行緒數量。如果有大量訂單同時進來，後廚就會動態招募更多廚師，但絕不會超過這個人力上限，以免廚房人太多彼此擠到走不開，反而降低效率、耗盡系統資源。

```CSHARP

int maxWorker, maxIOC;
ThreadPool.GetMaxThreads(out maxWorker, out maxIOC);
maxWorker.Dump();
maxIOC.Dump();

//// 32767
//// 1000
```

ThreadPool 最多可以容納 32,767 個 CPU Bound 執行緒，最多可容納 1,000 個 I/O Completion 執行緒


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-7-thread-cycle/9_2_max_thread.png)


總結是這樣

- 最小值太低 ➜ 容易接單卡住，導致吞吐量下降
- 最小值太高 ➜ 閒置執行緒也會佔用資源
- 最大值太低 ➜ 並發能力受限
- 最大值太高 ➜ 太多執行緒反而彼此爭奪 CPU，Context Switching 開銷反噬效能


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-7-thread-cycle/9_3_thread_min_max_tradesoff.png)



<!-- endtab -->

<!-- tab 「ThreadPool 實驗-->

在一個應用程式（Process）裡，執行程式碼的最小單位就是 執行緒（Thread）。而多執行緒（Multi-Thread）就是指，同一時間內，這個應用程式裡有多個執行緒在同時執行不同的工作

在 .NET 的世界，可以自己建立 Thread，或更常見的，把工作丟給 ThreadPool（執行緒集區）自動幫你安排哪位廚師來做，省去手動管理執行緒的麻煩

```CSHARP

void Main()
{
	$"start ThreadId: {Thread.CurrentThread.ManagedThreadId}".Dump();
	Task.Run(() => DoSomethingLong("啟動 Multi-Thread"));
	$"end ThreadId: {Thread.CurrentThread.ManagedThreadId}".Dump();
}

static void DoSomethingLong(string para)
{
	$"{para}  ThreadId: {Thread.CurrentThread.ManagedThreadId}".Dump();
}

// start ThreadId: 1
// end ThreadId: 1
// 啟動 Multi-Thread  ThreadId: 5
```

- 1️⃣ 先輸出開始時的 Thread ID，通常是 1，代表這是主執行緒（Main Thread）。
- 2️⃣ 呼叫 Task.Run，它會把工作排進 ThreadPool，呼叫點（Main Thread）不會被卡住你的背景工作會在別的執行緒執行
- 3️⃣ 馬上輸出結束時的 Thread ID，可以看到開始和結束的 Thread ID 相同，證明主執行緒執行到這裡沒有被阻塞。

start 和 end 仍然是同一個 Thread ID，證明 Main Thread 自己沒有被阻塞。



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-7-thread-cycle/10_thread_1_id.png)


<!-- endtab -->


<!-- tab summary-->


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-7-thread-cycle/11_final.png)



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-7-thread-cycle/thread-cycle.png)


<!-- endtab -->



{% endtabs %}