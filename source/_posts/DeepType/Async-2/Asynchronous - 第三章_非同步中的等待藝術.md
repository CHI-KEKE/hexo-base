---
title: Asynchronous Programming - 第三章:非同步中的等待藝術
date: 2024-07-25 13:00:05
categories: 未来よりの返歌
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-3-taskrun/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-3-taskrun/1_landing.png
tags:
    - Asynchronous Programming
toc:
toc_number:
comments :
---

{% tabs Asynchronous3%}

<!-- tab 全自動咖啡機-->

小遙家裡有一台很貴的全自動咖啡機。只要放入咖啡豆，按下按鈕，五分鐘後，一杯完美的拿鐵便會端坐在托盤上，溫度與比例都恰到好處

<br>

但她還是習慣站在一旁等

<br>

她不相信機器；在咖啡流出前的幾十秒，她總會打開上蓋，觀察豆槽有沒有卡住，再輕拍一下；聽見磨豆的聲音時，也會試著微調參數、瞄一下壓力表。她甚至曾試著在機器運作時，手動幫忙攪拌牛奶。直到有一天，她打翻了一杯還沒加糖的咖啡，才驚覺：其實什麼都不做，才是真的在等那杯咖啡完成

有些事情，本質上就像 I/O-bound 的任務，對方的成長、一段關係的發酵、時機的成熟、情緒的消化，它們需要的是時間與環境，不是你額外派一個「焦慮的自己」去乾等


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-3-taskrun/2_no_interfere_coffee.png)



<!-- endtab -->

<!-- tab Task.Run(...) 的本質-->


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-3-taskrun/4_task_run.png)


`Task.Run` 的本質不是「非同步」，而是把你原本會卡住目前執行緒的同步工作，轉交給 `Thread Pool` 的背景執行緒去扛，讓現在這條執行緒先活下來


```CSHARP
public static Task Run(Action action)
{
    return Task.Factory.StartNew(
        action, 
        CancellationToken.None,
        TaskCreationOptions.DenyChildAttach,
        TaskScheduler.Default);
}
```

1. 呼叫 `Task.Run(action)`，丟進來的是一段「同步 Action」，不是 `async`，也沒有 I/O 魔法，內部實際呼叫 `Task.Factory.StartNew(...)`

- `action`：同步程式碼
- `CancellationToken.None`：這個 `Task` 本身不支援取消
- `TaskCreationOptions.DenyChildAttach`：避免子 `Task` 掛到父 `Task` 上，讓行為更可預期
- `TaskScheduler.Default`：關鍵在這裡，指向 `ThreadPoolTaskScheduler`，也就是丟到 .NET Thread Pool

2. `Thread Pool` 拿一條背景執行緒來執行你的 `Action`
3. 這條 Thread 會被你那段同步程式碼「整條佔住」，不管你裡面是 for 迴圈、CPU 計算、還是 blocking I/O
4. `Action` 跑完後，`Thread Pool` 回收執行緒，`Task` 進入 `RanToCompletion`，結果（或例外）被封裝在 `Task` 裡，你可以 `await` 它，因為它「回傳的是 Task」，但這只是語法層面的 `await`，不代表底層是 `async I/O`


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-3-taskrun/4_task_run_underthehood.png)


`Task.Run` 只是把同步阻塞，從 A 執行緒搬到 B 執行緒，阻塞本身完全沒有消失，`Task.Run` 的價值，在於「不要卡住重要執行緒」，而不是讓工作變快，每一次 `Task.Run`，都是在跟整個系統借一條 Thread，用完不還就會出事


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-3-taskrun/5_block_no_disappear.png)



<!-- endtab -->

<!-- tab CPU-bound vs I/O-bound-->

await Task.Run(...) 的真正意義，是把「等待的責任」從重要的執行緒（UI / Request）轉交給 Thread Pool 裡的一條背景執行緒，讓該卡的去卡、不該卡的先走


```CSHARP
public async Task CompressFileAsync(string path)
{
    await Task.Run(() => PerformCompression(path));
}
```

#### 誰在等？

Thread Pool 的背景執行緒，不是呼叫端那條重要的執行緒

#### 為什麼這對 CPU-bound 有幫助?

因為 CPU-bound 的本質就是「一定有人要被卡」，Task.Run 只是選擇讓比較不重要的人去卡


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-3-taskrun/6_why_good_for_cpu_bound.png)



#### I/O-bound 為什麼不該用 Task.Run


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-3-taskrun/7_no_for_iobound.png)



I/O-bound 本來就靠 OS 的事件通知（IOCP），等待期間沒有任何 Thread 被佔用，此時，若在本身有非同步 API 的前提下，硬用 Task.Run() 包裝這種 async 方法

```CSHARP
await Task.Run(() => _httpClient.GetAsync(url));
```

等於是多抽一條 Thread Pool Thread，但什麼事也沒做，只是等 async Task 自己完成，這在高併發下，這會直接拖垮 Thread Pool



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-3-taskrun/7_2_iobound_exhaust.png)


<!-- endtab -->


<!-- tab summary-->


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-3-taskrun/8_table.png)


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-3-taskrun/9_final.png)


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-3-taskrun/taskrun.png)


<!-- endtab -->

{% endtabs %}