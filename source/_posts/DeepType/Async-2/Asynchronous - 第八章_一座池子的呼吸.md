---
title: Asynchronous Programming - 第八章:一座池子的呼吸
date: 2024-10-26 09:22:05
categories: 未来よりの返歌
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-8-threadpool/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-8-threadpool/1_landing.png
tags:
    - Asynchronous Programming
toc:
toc_number:
comments :
---

{% tabs 一座池子的呼吸%}

<!-- tab 一座池子的呼吸-->

在雲端深處，棲息著一個無形的池子，我們稱它為 ThreadPool，它不眠不休，日夜吞吐無數任務，是城市裡最沉默卻最可靠的工廠。每一個建立 `Task` 的瞬間，就像把工作單投入池子，任它激起層層漣漪，有人接手、有人閒置、有人待命

我們在這裡低聲詠唱，希望當高峰來臨，池子能無限擴張，當潮水退去，池子也能自我收束，忠實而溫順，既不餓死任務，也不浪費資源。當你想起這座池子，它從未離開，只是默默在背後，調度著一切的並行與秩序

<!-- endtab -->

<!-- tab Task.Run 觀察 ThreadPool 的重複使用-->

```CSHARP
void Main()
{
	Task.Run(() => DoSomething());
	Task.Run(() => DoSomething());
	Task.Run(() => DoSomething());
	Task.Run(() => DoSomething());
	Task.Run(() => DoSomething());
	
	Thread.Sleep(1000);
	Console.WriteLine("稍微等等後繼續.......");
	Thread.Sleep(1000);

	Task.Run(() => DoSomething());
	Task.Run(() => DoSomething());
	Task.Run(() => DoSomething());
	Task.Run(() => DoSomething());
	Task.Run(() => DoSomething());
}


public static void DoSomething()
{
	Console.WriteLine($"dooooooooo.........ThreadId: {Thread.CurrentThread.ManagedThreadId}");
}


// dooooooooo.........ThreadId: 16
// dooooooooo.........ThreadId: 18
// dooooooooo.........ThreadId: 19
// dooooooooo.........ThreadId: 17
// dooooooooo.........ThreadId: 21
// 稍微等等後繼續.......
// dooooooooo.........ThreadId: 21
// dooooooooo.........ThreadId: 21
// dooooooooo.........ThreadId: 19
// dooooooooo.........ThreadId: 18
// dooooooooo.........ThreadId: 15
```

- `Task.Run` 在背後其實也是呼叫 `ThreadPool` 取得執行緒，因此會看到重複使用相同的 `Thread ID`（例如 10, 9, 11）
- 執行多個 `Task` 時，`ThreadPool` 會並行排程任務，將可用的執行緒分配給尚未完成的工作
- 稍微 Sleep 一段時間後再丟新的 `Task`，多數情況下會看到相同的 `Thread ID` 被再次使用，表示執行緒沒有立刻銷毀，而是維持一段時間等待可重用
- 實際執行下，`Thread ID` 分配不會每次都相同，因為 `ThreadPool` 會根據當下系統負載、佇列狀況動態調整


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-8-threadpool/2_threadpool_cycle.png)


<!-- endtab -->

<!-- tab Starvation-Avoidance Mechanism 怕你餓機制-->

`ThreadPool` 的核心價值在於「有限資源的重複利用」。然而，如果所有可用的執行緒（`Threads`）都被某些長時間執行或阻塞的任務佔用，就會導致後續新任務無法被及時處理，甚至可能引發 資源競爭死鎖（`Deadlock`） 或 任務飢餓（`Starvation`）

為了解決這種情況，`ThreadPool` 內建了 `Starvation-Avoidance Mechanism`（飢餓避免機制），只要它發現工作佇列（`Queue`）裡的 `等待項目持續無法減少`，就會考慮動態 `擴充額外的 Worker Threads`，嘗試打破當前的阻塞局面



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-8-threadpool/3_starvation.png)



<!-- endtab -->

<!-- tab Throughput-->

Thread 裡的 Throughput 不是「跑得快不快」，而是「同一段時間內，你到底能消化掉多少工作」；ThreadPool 只在乎你加人後，有沒有真的多做完事情

1. 先定義「工作完成」的事件，在 ThreadPool 的視角，一個 work item / task 結束，就算「完成 1 份工作」
2. 選一個觀察時間窗（sampling window），例如用 0.5 秒、1 秒、或一段固定期間去看
3. 算這段時間內完成了幾個工作，記作 CompletedWorkItems（完成數）
4. 完成數除以時間，得到吞吐量 => 每秒完成幾個工作
5. 調整 worker threads 後再觀察一次
   - throughput 上升：代表多開 threads 是有效的
   - throughput 不變或下降：代表多開 threads 沒幫忙（可能還害你切換更多、資源更吃）


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-8-threadpool/5_throughput.png)


## 情境 A：Throughput 上升（加 threads 有用）

你有很多「短小、可並行、CPU 還吃得下」的工作，原本 10 threads：每秒做完 100 個，加到 12 threads：每秒做完 120 個，throughput 從 100/s → 120/s，上升，所以 ThreadPool 會覺得「這方向對」

## 情境 B：Throughput 下降（加 threads 反而害你）

工作是 CPU-bound，CPU 已經 100%，原本 10 threads：每秒做完 100 個，加到 20 threads：每秒只做完 85 個（因為 context switch、cache miss、排程成本暴增），throughput 從 100/s → 85/s，下降，所以 ThreadPool 會覺得「過頭了」




<!-- endtab -->

<!-- tab Climbing Heuristic - 爬山捷思法-->

剛才提到 `ThreadPool` 會透過 `Starvation-Avoidance Mechanism` 避免等待被執行的工作餓死，但光是臨時增加 `Threads` 可能導致過度濫用。為了找到「恰到好處」的執行緒數量，`ThreadPool` 採用了 `Hill-Climbing Heuristic`（爬山捷思法）。所謂 `Hill-Climbing`，就像在一座未知地形的山上尋找最高點



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-8-threadpool/6_hill_climb.png)


- 山的高度代表系統的 `Throughput`（吞吐量）
- 你不知道最高點在哪裡，只能根據當下位置和變化趨勢，一步步嘗試往更高處移動

在 `ThreadPool` 中，這個算法會持續觀察

- 目前有多少任務在排隊？
- `Worker Threads` 的數量是多少？
- 調整 `Threads` 後，整體 `Throughput` 有沒有變化？

當 `Queue` 中有等待工作，且執行已持續一段時間（例如超過 0.5 秒），`ThreadPool` 可能會 增加一個或多個 `Worker Threads` 來嘗試提高吞吐量。如果觀察到 `Throughput` 確實上升，則代表「往山頂爬對了方向」，系統可能會繼續增加 `Threads`。如果 Throughput 開始下降或維持不變，代表「已超過最佳點或卡在局部高峰」，此時就會嘗試減少 Threads，避免額外的資源開銷



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-8-threadpool/6_2_hill_climb_strategy.png)



<!-- endtab -->

<!-- tab 為什麼增加 Threads 並不一定帶來更高 Throughput？-->

## Context Switching 成本

當 `Threads` 數量增加超過 CPU 核心數時，CPU 必須不停在多個執行緒之間切換，`每次切換都要保存和恢復執行緒的狀態（如暫存器、堆疊）`，這些額外操作會吃掉寶貴的 CPU 週期

想像一位廚師同時接了 5 個訂單，每隔幾分鐘就要放下手上的刀具，去處理下一個客人，來回切換太頻繁，反而沒辦法專注把一份菜切完。這就是多執行緒切換的開銷

## 有限資源的競爭與記憶體壓力

每個 `Thread` 都需要佔用 CPU、記憶體（例如堆疊空間）及快取，當 Threads 太多時，這些資源的競爭會導致快取失效率增加、記憶體管理成本上升，甚至可能觸發額外的 Garbage Collection 或分頁換出（paging），讓效能反而變差


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-8-threadpool/4_context_switch.png)



<!-- endtab -->

<!-- tab 實驗一.實驗設計-->


`塞入 200 個工作，每個工作要設定 1 min 完成，觀察 Threads 數量增長情形`

ThreadPool 不會因為你一次丟很多工作就立刻開滿執行緒，而是邊跑邊觀察「塞不塞車」，再慢慢補人，避免一開始就把系統拖垮

[實驗設計者](https://blog.darkthread.net/blog/threadpool-thread-management/)

```CSHARP

void Main()
{
	//// 觀察當下的資源
	bool stop = false;
	ThreadPool.GetMinThreads(out int minWorkerThreads, out int minCompletionPortThreads);
	ThreadPool.GetMaxThreads(out int maxWorkerThreads, out int maxCompletePortThreads);
	Console.WriteLine($"ThreadPool Min : minWorkerThreads {minWorkerThreads}, minCompletionPortThreads {minCompletionPortThreads}");
	Console.WriteLine($"ThreadPool Max : MaxWorkerThreads {maxWorkerThreads}, MaxCompletionPortThreads {maxCompletePortThreads}");
	
	const int totalTasksCount = 200;
	var remainingCount = totalTasksCount;
	var running = 0;
	var startDatetime = DateTime.Now;

	// 固定的監控排程
	// 每秒印一次 
	// ThreadPool.ThreadCount（目前有多少執行緒）
	// running（真的在跑的任務數）
	// PendingWorkItemCount（還在排隊的工作）
	// 使用 flag 來偵測是否停止運作
	Task.Factory.StartNew(() => {
		Console.WriteLine("Time | Threads | Running | Pending ");
		Console.WriteLine("-----+---------+---------+---------");
		while(stop == false)
		{
			Console.WriteLine($"{(DateTime.Now - startDatetime).TotalSeconds,3:n0}s | {ThreadPool.ThreadCount,7} | {running,7} | {ThreadPool.PendingWorkItemCount,7}");
			Thread.Sleep(1000);
		}
	});

	// 將 200 個工作丟進 ThreadPool
	// 標記「我開始跑了」
	// 睡 60 秒（模擬長時間任務）
	// 標記「我跑完了」
	Enumerable.Range(0, totalTasksCount).ToList().ForEach(i =>
	{
		Task.Run(() => {
			Interlocked.Increment(ref running);
			Thread.Sleep(60000);
			Interlocked.Decrement(ref remainingCount);
			Interlocked.Decrement(ref running);
		});
	});
	
	/// remainingCount = 0 時跳出
	while(remainingCount > 0)
	{
		Thread.Sleep(100);
	}
	
	stop = true;
}

```

```bash
ThreadPool Min : minWorkerThreads 20, minCompletionPortThreads 1
ThreadPool Max : MaxWorkerThreads 32767, MaxCompletionPortThreads 1000
Time | Threads | Running | Pending 
-----+---------+---------+---------
  0s |      11 |       8 |     191
  1s |      21 |      20 |     181
  2s |      23 |      22 |     180
  3s |      24 |      23 |     180
  4s |      25 |      24 |     179
  5s |      27 |      26 |     177
  6s |      28 |      27 |     176
  7s |      30 |      29 |     174
  8s |      31 |      30 |     173
  9s |      32 |      31 |     172
 10s |      34 |      33 |     170
 11s |      34 |      33 |     170
 12s |      36 |      35 |     168
 13s |      37 |      36 |     167
 14s |      38 |      37 |     166
 15s |      39 |      38 |     165
 16s |      40 |      39 |     164
 17s |      41 |      40 |     163
 18s |      43 |      42 |     161
 19s |      44 |      43 |     160
 20s |      46 |      45 |     158
 21s |      47 |      46 |     157
 22s |      48 |      47 |     156
 23s |      49 |      48 |     155
 24s |      51 |      50 |     153
 25s |      52 |      51 |     152
 26s |      53 |      52 |     151
 27s |      54 |      53 |     150
 28s |      56 |      55 |     148
 29s |      57 |      56 |     147
 30s |      58 |      57 |     146
 31s |      60 |      59 |     144
 32s |      61 |      60 |     143
 33s |      62 |      61 |     142
 34s |      63 |      62 |     141
 35s |      65 |      64 |     139
 36s |      66 |      65 |     138
 37s |      68 |      67 |     136
 38s |      69 |      68 |     135
 39s |      71 |      70 |     133
 40s |      73 |      72 |     131
 41s |      74 |      73 |     130
 42s |      75 |      74 |     129
 43s |      76 |      75 |     128
 44s |      77 |      76 |     127
 45s |      78 |      77 |     126
 46s |      80 |      79 |     124
 48s |      81 |      80 |     123
 49s |      82 |      81 |     122
 50s |      83 |      82 |     121
 51s |      84 |      83 |     120
 52s |      85 |      84 |     119
 53s |      87 |      86 |     117
 54s |      88 |      87 |     116
 55s |      90 |      89 |     114
 56s |      91 |      90 |     113
 57s |      92 |      91 |     112
 58s |      93 |      92 |     111
 59s |      95 |      94 |     109
 60s |      96 |      95 |     108
 61s |      97 |      96 |      88
 62s |      97 |      96 |      86
 63s |      98 |      97 |      84
 64s |      99 |      98 |      82
 65s |      99 |      98 |      80
 66s |      99 |      98 |      78
 67s |     100 |      99 |      76
 68s |     100 |      99 |      74
 69s |     101 |     100 |      72
 70s |     102 |     101 |      70
 71s |     103 |     102 |      68
 72s |     103 |     102 |      66
 73s |     104 |     103 |      64
 74s |     105 |     104 |      62
 75s |     106 |     105 |      60
 76s |     107 |     106 |      58
 77s |     108 |     107 |      56
 78s |     108 |     107 |      54
 79s |     109 |     108 |      52
 80s |     109 |     108 |      50
 81s |     110 |     109 |      48
 82s |     111 |     110 |      46
 83s |     111 |     110 |      45
 84s |     112 |     111 |      42
 85s |     113 |     112 |      41
 86s |     114 |     113 |      39
 87s |     115 |     114 |      37
 88s |     115 |     114 |      35
 89s |     116 |     115 |      33
 90s |     116 |     115 |      31
 91s |     117 |     116 |      28
 92s |     118 |     117 |      26
 93s |     118 |     117 |      25
 94s |     119 |     118 |      23
 95s |     120 |     119 |      20
 96s |     120 |     119 |      19
 97s |     121 |     120 |      17
 98s |     121 |     120 |      15
 99s |     122 |     121 |      13
100s |     122 |     121 |      11
101s |     123 |     122 |       9
102s |     124 |     123 |       7
103s |     125 |     124 |       5
104s |     126 |     125 |       3
105s |     126 |     123 |       0
106s |     126 |     122 |       0
107s |     126 |     121 |       0
108s |     126 |     120 |       0
109s |     126 |     119 |       0
110s |     126 |     118 |       0
111s |     126 |     117 |       0
112s |     126 |     115 |       0
113s |     126 |     113 |       0
114s |     126 |     112 |       0
115s |     126 |     111 |       0
116s |     126 |     110 |       0
117s |     126 |     109 |       0
118s |     126 |     107 |       0
119s |     126 |     106 |       0
120s |     126 |      85 |       0
121s |     126 |      84 |       0
122s |     126 |      82 |       0
123s |     126 |      80 |       0
124s |     126 |      78 |       0
125s |     126 |      76 |       0
126s |     126 |      74 |       0
127s |     126 |      72 |       0
128s |     126 |      70 |       0
129s |     122 |      68 |       0
130s |     121 |      66 |       0
131s |     120 |      64 |       0
132s |     119 |      62 |       0
133s |     118 |      60 |       0
134s |     115 |      58 |       0
135s |     115 |      56 |       0
136s |     113 |      54 |       0
137s |     111 |      52 |       0
138s |     111 |      50 |       0
139s |     109 |      48 |       0
140s |      92 |      46 |       0
141s |      89 |      44 |       0
142s |      88 |      42 |       0
143s |      86 |      40 |       0
144s |      84 |      38 |       0
145s |      82 |      37 |       0
146s |      80 |      34 |       0
147s |      79 |      32 |       0
149s |      76 |      31 |       0
150s |      75 |      28 |       0
151s |      73 |      26 |       0
152s |      69 |      25 |       0
153s |      67 |      22 |       0
154s |      65 |      20 |       0
155s |      63 |      18 |       0
156s |      60 |      16 |       0
157s |      58 |      14 |       0
158s |      56 |      13 |       0
159s |      53 |      11 |       0
160s |      51 |       9 |       0
161s |      50 |       7 |       0
162s |      48 |       5 |       0
163s |      46 |       3 |       0
164s |      45 |       1 |       0
```

<!-- endtab -->

<!-- tab 實驗一. 結果觀察-->

## 1️⃣ 初始設置

- ThreadPool 最小工作 Thread 數（MinWorkerThreads）是 20
- 最大工作 Thread 數（MaxWorkerThreads）是 32767
- 最小 I/O 完成埠 Thread 數（MinCompletionPortThreads）是 1
- 最大 I/O 完成埠 Thread 數（MaxCompletionPortThreads）是 1000

ThreadPool 就像開了一家工廠，規定至少要有 20 個工人 stand by，最多可以請到 32767 個人來做事。I/O  Thread 則是處理檔案或網路之類的工作，跟 CPU 工作 Thread 分開算


## 2️⃣ 開始執行時持續派出工人

ThreadPool 一開始從大概 11 條 Thread（主執行緒也算在內），很快增加到 21、23、24 條 Thread ，接著持續增加，最後約在 105 秒達到最高峰 126 條 Thread 。ThreadPool 有一個保護機制：當有太多任務排隊時，會慢慢增加 Thread ，每秒大概只能多生 1~2 條 Thread ，避免突然開太多，電腦被榨乾。

## 3️⃣ Pending 的任務的消化

一開始 Pending（排隊任務）有 191 件。前 60 秒，Pending 慢慢減少，但因為每個任務都要做 60 秒，所以前期沒有任務完成，新的 Thread 只能慢慢補充。60 秒後開始有任務做完，Pending 開始明顯下降。前 60 秒很像一堆客人進餐廳，工人數量不夠，要慢慢叫外場工讀生加班，但廚房還沒煮完第一批，沒人下班，排隊還是很多。


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-8-threadpool/7_test1.png)


## 4️⃣ Starvation

任務執行時間固定是 60 秒（Sleep）。所以一開始 ThreadPool 只能靠 Starvation 機制偵測「工作都卡在隊列沒人做」→ 觸發每秒生 1~2 條 Thread 。到第 60 秒，第一批任務做完，有 Thread 釋放回池子，新的任務就用這些人手， Thread 數成長速度自然變慢

## 5️⃣ ThreadPool 就不再增加 Thread

大概 97~105 秒時，Pending 已經到 0， Thread 數還在最高點 126 條。Queue 沒有新任務進來，ThreadPool 就不再增加 Thread 。這時 Running 也逐漸下降，代表任務執行中也慢慢收尾。這就像工廠訂單做到一個段落，沒新訂單，就不會再找更多工人，但老闆也不會馬上解雇人，會慢慢等人閒到沒事做再請他們回家。

## 6️⃣ Thread 數的收斂

從 105 秒以後，Pending 一直是 0。 Thread 數從 126 條開始慢慢減少，經過 20~60 秒逐步降到 100 以下。最後 Thread 數會慢慢收斂到一個比較合理的水位（可能接近最小工作 Thread 或系統內部的 Idle  Thread 策略）


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-8-threadpool/7_2_peak_and_delay.png)


我們發現了，ThreadPool 真的是慢慢擴充，慢慢釋放，不是瞬間滿載瞬間消失。如果瞬間爆量（200 個任務同時來），預設的 MinThreads（20 條）明顯不夠，會導致很多任務必須排隊等待。每秒只能多生 1~2 條 Thread ，對短任務影響更大，因為排隊時間比執行時間還長，效能就會差。



<!-- endtab -->

<!-- tab 實驗二.開頭即設定 200 條最低執行緒數量-->

```CSHARP

ThreadPool.SetMinThreads(200, 1)

```

結果時間縮短為 60 秒左右，也就是 200 個任務等於一個任務所需要完成的時間

<!-- endtab -->

<!-- tab 實驗三.實驗設計-->

`每個任務時間縮短到 2 秒，每隔 30 秒塞 200 筆工作`

200 件工作做完後約 20 秒，Thread 數由 202 降到 4，第 30 秒又來 200 件工作時，Thread 數瞬問回到 200 條，與預期相符

```CSHARP
void Main()
{
	bool stop = false;
	ThreadPool.SetMinThreads(200, 1);
	ThreadPool.GetMinThreads(out int minWorkerThreads, out int minCompletionPortThreads);
	ThreadPool.GetMaxThreads(out int maxWorkerThreads, out int maxCompletePortThreads);
	Console.WriteLine($"ThreadPool Min : minWorkerThreads {minWorkerThreads}, minCompletionPortThreads {minCompletionPortThreads}");
	Console.WriteLine($"ThreadPool Max : MaxWorkerThreads {maxWorkerThreads}, MaxCompletionPortThreads {maxCompletePortThreads}");
	
	const int totalTasksCount = 200;
	var remainingCount = totalTasksCount;
	var running = 0;
	var startDatetime = DateTime.Now;

	// 固定的監控排程
	Task.Factory.StartNew(() => {
		Console.WriteLine("Time | Threads | Running | Pending ");
		Console.WriteLine("-----+---------+---------+---------");
		while(stop == false)
		{
			Console.WriteLine($"{(DateTime.Now - startDatetime).TotalSeconds,3:n0}s | {ThreadPool.ThreadCount,7} | {running,7} | {ThreadPool.PendingWorkItemCount,7}");
			Thread.Sleep(1000);
		}
	});


	//// 丟任務給 ThreadPool
	//Enumerable.Range(0, totalTasksCount).ToList().ForEach(i =>
	//{
	//	Task.Run(() => {
	//		Interlocked.Increment(ref running);
	//		Thread.Sleep(60000);
	//		Interlocked.Decrement(ref remainingCount);
	//		Interlocked.Decrement(ref running);
	//	});
	//});

	// 大的， 跑 3 輪，每次 200 件
	var insertTask = Task.Factory.StartNew(() => {
		
		for(int i = 0; i < 3;i++)
		{
			Console.WriteLine($"Insert batch : {i} at {(DateTime.Now - startDatetime).TotalSeconds,3:n0}");

			Enumerable.Range(1, 200).ToList().ForEach(i => {
				Interlocked.Increment(ref remainingCount);
				Task.Run(() =>
				{
					Interlocked.Increment(ref running);
					Thread.Sleep(2000); // 單筆執行 2 秒
					Interlocked.Decrement(ref running);
					Interlocked.Decrement(ref remainingCount);
				});
			});
	
			Thread.Sleep(30000); // 間隔 30 秒
		}
	});
	
	/// 還沒結束 阻擋
	while(insertTask.Status != TaskStatus.RanToCompletion || remainingCount > 0)
	{
		Thread.Sleep(100);
	}
	
	stop = true;
	
	Console.WriteLine("All done.");
}

```

<!-- endtab -->

<!-- tab 實驗三.觀察實驗結果-->

```bash
ThreadPool Min : minWorkerThreads 200, minCompletionPortThreads 1
ThreadPool Max : MaxWorkerThreads 32767, MaxCompletionPortThreads 1000
Time | Threads | Running | Pending 
-----+---------+---------+---------
Insert batch : 0 at   0
  0s |       4 |       0 |       0
  1s |     201 |     199 |       2
  2s |     202 |      32 |       0
  3s |     202 |       1 |       0
  4s |     202 |       0 |       0
  5s |     202 |       0 |       0
  6s |     202 |       0 |       0
  7s |     202 |       0 |       0
  8s |     202 |       0 |       0
  9s |     202 |       0 |       0
 10s |     202 |       0 |       0
 11s |     202 |       0 |       0
 12s |     202 |       0 |       0
 13s |     202 |       0 |       0
 14s |     202 |       0 |       0
 15s |     202 |       0 |       0
 16s |     202 |       0 |       0
 17s |     202 |       0 |       0
 18s |     202 |       0 |       0
 19s |     202 |       0 |       0
 20s |     202 |       0 |       0
 21s |     202 |       0 |       0
 22s |       7 |       0 |       0
 23s |       7 |       0 |       0
 24s |       7 |       0 |       0
 25s |       7 |       0 |       0
 26s |       7 |       0 |       0
 27s |       7 |       0 |       0
 28s |       7 |       0 |       0
 29s |       7 |       0 |       0
Insert batch : 1 at  30
 30s |     200 |     198 |       2
 31s |     202 |     200 |       1
 32s |     202 |       2 |       0
 33s |     202 |       0 |       0
 34s |     202 |       0 |       0
 35s |     202 |       0 |       0
 36s |     202 |       0 |       0
 37s |     202 |       0 |       0
 38s |     202 |       0 |       0
 39s |     202 |       0 |       0
 40s |     202 |       0 |       0
 41s |     202 |       0 |       0
 42s |     202 |       0 |       0
 43s |     202 |       0 |       0
 44s |     202 |       0 |       0
 45s |     202 |       0 |       0
 46s |     202 |       0 |       0
 47s |     202 |       0 |       0
 48s |     202 |       0 |       0
 49s |     202 |       0 |       0
 50s |     202 |       0 |       0
 51s |     202 |       0 |       0
 52s |       9 |       0 |       0
 53s |       9 |       0 |       0
 54s |       9 |       0 |       0
 55s |       9 |       0 |       0
 56s |       9 |       0 |       0
 57s |       9 |       0 |       0
 58s |       7 |       0 |       0
 59s |       7 |       0 |       0
Insert batch : 2 at  60
 60s |     200 |     198 |       2
 61s |     202 |     200 |       1
 62s |     203 |       2 |       0
 63s |     203 |       0 |       0
 64s |     203 |       0 |       0
 65s |     203 |       0 |       0
 66s |     203 |       0 |       0
 67s |     203 |       0 |       0
 68s |     203 |       0 |       0
 69s |     203 |       0 |       0
 70s |     203 |       0 |       0
 71s |     203 |       0 |       0
 73s |     203 |       0 |       0
 74s |     203 |       0 |       0
 75s |     203 |       0 |       0
 76s |     203 |       0 |       0
 77s |     203 |       0 |       0
 78s |     203 |       0 |       0
 79s |     203 |       0 |       0
 80s |     203 |       0 |       0
 81s |     203 |       0 |       0
 82s |     203 |       0 |       0
 83s |       9 |       0 |       0
 84s |       8 |       0 |       0
 85s |       8 |       0 |       0
 86s |       8 |       0 |       0
 87s |       8 |       0 |       0
 88s |       8 |       0 |       0
 89s |       8 |       0 |       0
 90s |       8 |       0 |       0
 91s |       8 |       0 |       0
 92s |       8 |       0 |       0
 93s |       8 |       0 |       0
 94s |       8 |       0 |       0
 95s |       8 |       0 |       0
 96s |       8 |       0 |       0
 97s |       8 |       0 |       0
 98s |       8 |       0 |       0
 99s |       8 |       0 |       0
 ...
```

## 1️⃣ 依照 MinThreads 迅速建立新執行緒來滿足需求

剛開始插入一批 200 筆工作，我們看到一開始 ThreadPool.ThreadCount 很少（例如 4），當任務進來後，執行緒數量瞬間往上飆到 200+，running 很快達到 200，Pending 幾乎歸零，原因是 ThreadPool 看到要執行的工作量超過可用執行緒，就會依照 MinThreads 迅速建立新執行緒來滿足需求


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-8-threadpool/7_3_minthreads_burn.png)


## 2️⃣ ThreadPool 預設不會馬上銷毀多餘的執行緒

2 秒後大部分工作做完，工作做完後 running 很快降到 0，但 ThreadPool.ThreadCount 還是維持在 200 多，並沒有馬上下降，因為 ThreadPool 預設不會馬上銷毀多餘的執行緒，它會先讓這些執行緒空閒一段時間，等到超過「空閒逾時（idle timeout）」才開始回收

## 3️⃣ 開始收執行緒

約 20 秒後開始收執行緒，觀察到大約第 20 ~ 22 秒之後，執行緒數量從 200 多慢慢掉回到大概 7 ～ 8 條，因為 .NET ThreadPool 的預設執行緒空閒逾時（Idle Timeout）大約是 15~20 秒（不同版本的 .NET 有些微差異）超過這段時間沒工作用到執行緒時，ThreadPool 才會逐步把多餘的執行緒回收。


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-8-threadpool/7_4_burst_and_idle.png)



## 4️⃣ 又丟新的 200 筆工作

30 秒後又丟新的 200 筆，新一批工作丟進去後，執行緒數量又瞬間從 7 條跳回 200 多條

<!-- endtab -->


<!-- tab summary-->


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-8-threadpool/7_6_test_result_table.png)


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-8-threadpool/8_final.png)


<!-- endtab -->

{% endtabs %}