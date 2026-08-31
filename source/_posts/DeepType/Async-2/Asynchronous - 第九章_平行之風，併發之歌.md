---
title: Asynchronous Programming - 第九章:平行之風，併發之歌
date: 2025-06-26 09:22:05
categories: 未来よりの返歌
top_img: https://github.com/CHI-KEKE/pics/blob/main/Codesss/Async/logoAsync.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Codesss/Async/logoAsync.png?raw=true
tags:
    - Asynchronous Programming
toc:
toc_number:
comments :
---

{% tabs Asynchronous2%}

<!-- tab 穿梭林間-->

有些風，獨自穿梭林間，沿著樹影與地面爬行，沒有分身，沒有回音，只將時間吹得靜謐
有些風，輕輕裂成數萬縷絲線，從山脊滲入溪谷，從枝頭滲入土壤，它們彼此不糾纏，卻同時為大地帶來不同的聲響



![wind](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/wind.png)



也有些種子，懂得在雲後潛伏，等待雨季將它們喚醒；一顆未必只能開一朵花，它可以把等待切碎，併發成無數顆更小的種子，在時間的縫隙裡生根發芽

當我們談論分割與合流，等待與釋放，這片風與影子的地圖，就是平行之風，併發之歌
那些藏在核心深處的秘密，終究會在一次又一次的呼吸裡，被我們拆解、重組、散播，直到有一天，學會怎樣用一秒去換取另一秒




![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async--9-parallel/2_parallel_concurrency.png)

<!-- endtab -->

<!-- tab 實驗 : Single Thread / Parallel.For / Task-based-->

[用.NET展現多核威力(1) - 從ThreadPool翻船談起](https://blog.darkthread.net/blog/multicore-1/)

這個實驗透過一個簡單的數學運算（Math.Log10），使用

`單執行緒（Single Thread）`
`平行迴圈（Parallel.For）`
`Task-based 併發（async/await + Task.WhenAll）`

這三種方式，分別執行相同的 100 萬次運算，並多輪重複測量`執行時間`

目的是比較 順序執行 vs. 多核心平行 vs. 多任務併發 在 CPU-bound 工作下的效能差異，瞭解平行化與併發在沒有 I/O 等待、沒有共享資源時，會帶來什麼開銷與優勢，體會 ThreadPool、Context Switching、Task 排程等背後的隱藏成本

藉由實驗結果，我們希望可以回答

- 什麼情況適合單執行緒？
- 什麼情況可以用平行迴圈發揮多核心效能？
- 什麼情況下 Task-based 非但沒好處還可能更慢？



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async--9-parallel/3_tests.png)



```CSHARP

async Task Main()
{
	// 設定測試參數
	//// TIME：每一種做法要執行 一百萬次
	//// Round：整組測試跑 5 輪
	//// 用多輪是為了讓行為「穩定可觀察」，而不是偶發一次
	//// const int TIME = 1_000_000;
	//// int Round = 5;
	
	//// 建 Loop 實作多輪測試
	for (int i = 0; i < Round; i++)
	{
		Console.WriteLine($"This is Round : {i + 1}");

		// SingleTread
		//// 把一段同步程式碼包成 Action，傳進 MeasureTime，裡面只是一個單純的 for，每圈做一次數學運算
		var singleThreadTime = TimeHelper.MeasureTime(() =>
		{
			for (int j = 0; j < TIME; j++)
			{
				double d = Math.Log10(Convert.ToDouble(i));
			}
		});

		Console.WriteLine($"Single Thread : {singleThreadTime} milliSeconds");

		// Parallel.For
		//// 呼叫 Parallel.For，runtime 會自動切割 0 ~ TIME，分派到多個 ThreadPool 執行緒，每個 iteration 都是獨立的 
		var parallelForTime = TimeHelper.MeasureTime(() =>
		{
			Parallel.For(0, TIME, k => {
				double d = Math.Log10(Convert.ToDouble(k + 1));
			});
		});
		
		Console.WriteLine($"parallelForTime : {parallelForTime} milliSeconds");

		// ASync
		//// 每一次建立一個 Task，會立即在 ThreadPool 上排程執行，共產生一百萬個「尚未執行完成的 Task」，並在同步點「等所有工作跑完」
		var taskRunAsyncTime = await TimeHelper.MeasureTimeAsync(async () =>
		{
			var tasks = Enumerable.Range(0, TIME).Select(async l => {
				double d = Math.Log10(Convert.ToDouble(l + 1));
			});
			
			await Task.WhenAll(tasks);
		});
		
		Console.WriteLine($"Task-based: {taskRunAsyncTime:N0}ms");
		Console.WriteLine();
	}
}

public static class TimeHelper
{
	public static long MeasureTime(Action action)
	{
		var stopWatch = new Stopwatch();
		stopWatch.Start();
		action();
		stopWatch.Stop();
		return stopWatch.ElapsedMilliseconds;
	}

	public static async Task<long> MeasureTimeAsync(Func<Task> action)
	{
		Stopwatch stopwatch = new Stopwatch();
		stopwatch.Start();

		await action();

		stopwatch.Stop();
		return stopwatch.ElapsedMilliseconds;
	}
}

```

<!-- endtab -->

<!-- tab 測試紀錄-->

```bash
This is Round : 1
Single Thread : 30 milliSeconds
parallelForTime : 8 milliSeconds
Task-based: 69ms

This is Round : 2
Single Thread : 5 milliSeconds
parallelForTime : 6 milliSeconds
Task-based: 73ms

This is Round : 3
Single Thread : 4 milliSeconds
parallelForTime : 6 milliSeconds
Task-based: 72ms

This is Round : 4
Single Thread : 4 milliSeconds
parallelForTime : 9 milliSeconds
Task-based: 85ms

This is Round : 5
Single Thread : 4 milliSeconds
parallelForTime : 11 milliSeconds
Task-based: 83ms
```

<!-- endtab -->

<!-- tab Single Thread-->

單執行緒平均時間第一次 4 但後續皆 ~ 30 毫秒，執行穩定

這邊只有一條執行緒，CPU 核心 A 開始跑，流程是「絕對線性的」

```bash
j=0 → 算完
j=1 → 算完
j=2 → 算完
...
j=999999 → 算完
```

單執行緒沒有額外的排程、執行緒切換、或執行緒池分派開銷，CPU 只要照順序完成迴圈裡的計算即可。這種情況下，效能幾乎只受到 CPU 時脈與記憶體快取的影響。適合執行小規模、簡單且不需要分拆的 CPU 任務，像是短時間的批次運算、輕量的邏輯處理，或需要維持執行緒上下文一致性時（例如需要在 UI 執行緒中執行）



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async--9-parallel/4_single_thread.png)


<!-- endtab -->

<!-- tab Parallel.For-->


平均時間約 6 ~ 11 毫秒，明顯比單執行緒快 2~5 倍。

在這裡事告訴 runtime，這個迴圈裡的每一圈彼此沒關係，是請想辦法同時跑，Runtime 看到 0 ~ 1,000,000
它會在心裡想：「我這台電腦有 8 核心，那我大概可以同時尻 8 條執行緒。」，於是它會切成類似

```bash
Thread A：0 ~ 124,999
Thread B：125,000 ~ 249,999
Thread C：250,000 ~ 374,999
...
```


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async--9-parallel/5_parallel.png)



但這個切法不是你控制的，是 runtime 決定的，而他會把「區塊」丟給 ThreadPool，ThreadPool 裡本來就有一堆工人，有空的就接一塊來做，所以是真的同時在算，不是輪流，並且他們會動態補洞，假設 Core 1 很快做完，Core 3 還在算，那 Core 1 不會發呆，它會「你那邊還沒做完？我幫你拿一點來做」，這叫 Work-Stealing
目標是讓 CPU 核心不要閒著



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async--9-parallel/5_2_work_steal.png)



所以多核心系統一次只用一條線（CPU 的一個核心）去做，會很浪費電腦的多核心能力。Parallel.For 會把迴圈工作，切成一小塊一小塊，丟給不同的核心同時去做

- 一般 for：一個人做 100 件事。
- Parallel.For：10 個人分工，每人做 10 件事，大家同時開始。這樣就能用到 CPU 的多核心，達到 平行處理，速度通常會更快

要注意的是，如果每筆任務非常小，平行化的好處會被 Context Switching、排程、資料同步的成本吃掉。如果每筆任務彼此之間需要共享資源（例如對同一個集合寫入），就必須加鎖（lock），加鎖就會導致多執行緒之間搶鎖，效能會急速下降


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async--9-parallel/5_parallel.png)



<!-- endtab -->


<!-- tab Task-based (Concurrency)-->


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async--9-parallel/6_task_based.png)


平均時間約 69 ~ 85 毫秒，遠遠落後於單執行緒與 Parallel.For。

Task 是 併發（Concurrency） 的抽象，它是為了讓一個執行緒可以同時「管理」多個任務進度而存在，而不是要讓每個任務都佔用獨立執行緒。在典型的 I/O Bound 情境（例如呼叫 API、等待檔案寫入），Task 透過 async/await 可以`在等待期間「釋放執行緒」`，讓執行緒去跑其他工作，等結果回來時再接手後續邏輯，這樣就能極大化 ThreadPool 的效益。

但在這個案例，每次都要建立一個獨立的 Task，對於 100 萬次來說，就會產生 100 萬個 Task 實例

每一個 Task「實際經歷了這些事情」

- 1️⃣ 配置一個 Task 物件（heap allocation）
- 2️⃣ 建立狀態機（就算你沒有 await）
- 3️⃣ 加入 ThreadPool 的工作佇列
- 4️⃣ 被某個 ThreadPool thread 拿到
- 5️⃣ 執行一行 Math.Log10
- 6️⃣ 標記完成
- 7️⃣ 觸發 continuation（給 WhenAll 用）

為了算一行數學，付出了 7 個步驟

重點是，這些任務本身極短且完全沒有 I/O 等待，這樣的操握導致他沒時間釋放執行緒，反而產生了巨大的排程與 Context Switching 開銷




![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async--9-parallel/6_2_costs.png)





一條執行緒可以「輪流服務很多 Task」，才是 Task + async 的本命用途!!!!!


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async--9-parallel/6_3_real_tasks.png)



實驗結果得知，他們的使用場景大致是這樣子

`Single Thread` 👉 工作很小、很快、順序重要
`Parallel.For` 👉 工作可以切、每塊夠大、CPU 算很久
`Task-based async` 👉 工作會「等」，而不是「算」

<!-- endtab -->


<!-- tab summary-->



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async--9-parallel/7_table.png)



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async--9-parallel/8_final.png)



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async--9-parallel/async-parallel.png)



<!-- endtab -->


{% endtabs %}