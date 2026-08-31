---
title: Asynchronous Programming - 第十三章:Task.WhenAll
date: 2025-10-21 22:00:05
categories: 未来よりの返歌
top_img: https://github.com/CHI-KEKE/pics/blob/main/Codesss/Async/logoAsync.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Codesss/Async/logoAsync.png?raw=true
tags:
    - Asynchronous Programming
toc:
toc_number:
comments :
---

{% tabs Task.WhenAll%}

<!-- tab 跑腿-->

你叫三個小弟幫你跑腿

Gina 去買雞蛋
Bill 去買牛奶
Lily 去買麵包

你不會等 Gina 回來才讓 Bill 去，你會同時叫他們出門，然後你說：「等你們全部都回來，我們再一起做早餐！」這就是 `Task.WhenAll` ，等全部人回來，再開始做接下來的事

<br>

它的本質是，把很多 Task 包裝成「一個新的 Task」，這個新 Task 會在所有子 Task 都完成後才完成 (to execute multiple asynchronous tasks concurrently，and wait for all of them to complete)


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-13-taskwhenall/1_three_buy.png)


<!-- endtab -->

<!-- tab 什麼時候觸發了任務執行?-->

```csharp
async Task Main()
{
	var tasks = new List<Task>();
	foreach (var i in Enumerable.Range(1, 3))
	{
		tasks.Add(DoWorkAsync(i));
	}

	Console.WriteLine("All tasks added!");
	await Task.WhenAll(tasks);
	Console.WriteLine("All done!");
}

async Task DoWorkAsync(int id)
{
	Console.WriteLine($"Task {id} started");
	await Task.Delay(1000);
	Console.WriteLine($"Task {id} finished");
}
```


```bash
Task 1 started
Task 2 started
Task 3 started
All tasks added!
Task 3 finished
Task 1 finished
Task 2 finished
All done!
```


👉 可以看到，Task started 都發生在 All tasks added! 之前，代表任務在被「建立（呼叫）」時就開始跑了，不等 WhenAll 才啟動



![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-13-taskwhenall/2_task_start_by_add.png)



![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-13-taskwhenall/3_safetime.png)


<!-- endtab -->

<!-- tab 下載檔案預覽-->

流程大概是這樣子的 : 「給一串網址 → 同時下載所有文字檔 → 全部下載完成後印出內容預覽」

```csharp
async void Main()
{
	var urls = new List<string>()
	{
		"https://www.gutenberg.org/cache/epub/1342/pg1342.txt",
		"https://www.gutenberg.org/cache/epub/1661/pg1661.txt"
	};
	
	await DonwloadFilesAsync(urls);
}

public static async Task DonwloadFilesAsync(IList<string> urls)
{
	var getContentTasks  = new List<Task<byte[]>>();
	foreach(var url in urls)
	{
		// 呼叫 DownloadFileAsync(url) → ⚠️ 這一步會「建立並啟動」一個非同步任務。
		// 把那個回傳的 Task<byte[]> 加進 getContentTasks List。
		// 所以在這一步已經啟動任務
		// 相當於 var task = DownloadFileAsync(url); // 🚀 下載開始！ getContentTasks.Add(task); // 🧺 放進任務清單
		getContentTasks.Add(DownloadFileAsync(url));
	}

	var contents = await Task.WhenAll(getContentTasks);
	
	foreach(var content in contents)
	{
		$"Sunccessfully donwloaded conetnt, length : {content.Length}".Dump();
		var fileString = System.Text.Encoding.UTF8.GetString(content);
		var preview = fileString.Substring(0, Math.Min(50, fileString.Length));
		$"Preview : {preview}".Dump();
	}
}

public static async Task<byte[]> DownloadFileAsync(string url)
{
	using(var client = new HttpClient())
	{
		byte[] content = await client.GetByteArrayAsync(url);
		File.WriteAllBytes(@"C:\Users\Allen Lin\Desktop\MyLab\Lab.同步異步\x.txt",content);
		return content;
	}
}

```

1. 建立多個任務，DownloadFileAsync(url) 會立即啟動下載（非同步），不會卡住主線程。所以這段 `foreach `很快就會跑完
2. `Task.WhenAll` 等待全部完成，當你呼叫 `await Task.WhenAll(getContentTasks)` 時，程式會同時等所有下載任務完成，等全部都完成後，才會回傳所有下載結果
3. 處理結果，回傳的 contents 是一個「陣列」，裡面包含每個下載結果（`byte[]`）

<br>

## 🔹 改為依序非同步等待

```CSHARP
foreach (var url in urls)
{
    var content = await DownloadFileAsync(url);
    // 處理內容
}
```

1. 先等第一個下載完
2. 再下載第二個
3. 一個一個排隊

👉 速度會慢很多，因為網路下載可以同時進行，沒必要等一個完成再開始下一個。所以 `Task.WhenAll` 幫你「並行處理任務」可以極大化效能



![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-13-taskwhenall/4_download_sample.png)


<!-- endtab -->

<!-- tab Validators-->

假設現在有一個「付款流程」需要做多項驗證 (Validation)

AValidator：檢查交易紀錄是否存在
BValidator：檢查金額是否合理
CValidator：檢查使用者身份是否合法

每個驗證都要去查資料庫或呼叫外部 API，這些動作通常是「IO-bound」（例如讀寫資料、網路請求）而這類任務最適合用非同步 (async/await) 執行

```csharp
async Task Main()
{
	var context = new PayProcessContext()
	{
		TransactionId = 123
	};
	var validators = new List<IValidator>()
	{
		new AValidaor(),
		new BValidaor(),
		new CValidaor()
	};
	
	//// A 方法 一個個等待做完
	foreach (var v in validators)
	{
		var (isValid,errorMessage) = await v.IsValidAsync(context);
		if (!isValid)
			Console.WriteLine($"驗證失敗：{errorMessage}");
		else
			Console.WriteLine("驗證通過");
	}

	//// B 方法 一次排出多個非同步,一次等他做完
	var tasks = validators.Select(v => v.IsValidAsync(context)).ToList();
	var result = await Task.WhenAll(tasks);
	foreach (var (isValid,errorMessage) in result)
	{
		if (!isValid)
			Console.WriteLine($"驗證失敗：{errorMessage}");
		else
			Console.WriteLine("驗證通過");
	}
}

public class PayProcessContext
{
	public int TransactionId { get; set; }
}


public interface IValidator
{
	Task<(bool, string)>IsValidAsync(PayProcessContext context);
}

public class AValidaor : IValidator
{
	public async Task<(bool, string)> IsValidAsync(PayProcessContext context)
	{
		await Task.Delay(1000);
		return (true,string.Empty);
	}
}

public class BValidaor : IValidator
{
	public async Task<(bool, string)> IsValidAsync(PayProcessContext context)
	{
		await Task.Delay(1000);
		return (false,"B 發現了嚴重的流程問題ㄚㄚㄚ ");
	}
}

public class CValidaor : IValidator
{
	public async Task<(bool, string)> IsValidAsync(PayProcessContext context)
	{
		await Task.Delay(1000);
		return (false, "C 發現了嚴重的流程問題ㄚㄚㄚ ");
	}
}
```


```bash
驗證通過
驗證失敗：B 發現了嚴重的流程問題ㄚㄚㄚ 
驗證失敗：C 發現了嚴重的流程問題ㄚㄚㄚ 

//// 方法 1 需要 3秒
//// 方法 2 需要 1.XXX 秒
```

<!-- endtab -->

<!-- tab Task.WhenAll 最高?-->

因為多個任務幾乎同時啟動，也幾乎同時結束，即使只用一個執行緒，也能達到「同時在處理」的效果，就是所謂的「Concurrency（並行）」，對程式來說，這就是趕快把工作都派出去做，派完可以自己專心做其他事，如此一來會節省時間，不過需要注意如果對方是 DB、或比較怕一次性處理多個請求的服務，就有可能較容易發生處理失敗


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-13-taskwhenall/5_io_bound.png)



另外如果任務是「自己在 CPU 裡算東西」，程式並不會釋放執行緒，不管你用多少個 Task，這些運算都要「真的佔用 CPU 時間」，若用 `Task.WhenAll`，系統可能會開多個執行緒同時計算 → 結果是 CPU 爆滿、效能下降，在這種情況下，除非你有多核心 CPU 且希望平行計算（`Parallelism`），否則應該就用同步執行（`foreach`），因為 CPU-bound 的任務在 async 之下沒有加速效果


因此，並不是你想併行處理多個任務就 Task.WhenAll 最高這件事，仍然是場景應用的!!


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-13-taskwhenall/6_cpu_bound.png)



<!-- endtab -->

<!-- tab WaitAll 與 WhenAll-->

```csharp
// A：同步等待（阻塞 Thread）
Task.WaitAll(task1, task2, task3);

void Main()
{
    var t1 = Task.Delay(1000);
    var t2 = Task.Delay(1000);
    Task.WaitAll(t1, t2);  // 🚫  Main Thread 被卡住直到兩個都完成
    Console.WriteLine("All done!");
}

// B：非同步等待（不阻塞 Thread）
await Task.WhenAll(task1, task2, task3);

async Task Main()
{
    var t1 = Task.Delay(1000);
    var t2 = Task.Delay(1000);
    await Task.WhenAll(t1, t2);  // ✅ 不會阻塞 Main Thread 
    Console.WriteLine("All done!");
}
```

| 項目          | `Task.WaitAll()`      | `Task.WhenAll()`                     |
| ----------- | --------------------- | ------------------------------------ |
| 類型          | 同步 API                | 非同步 API                              |
| 是否阻塞 Thread | ✅ 會阻塞目前執行緒            | ❌ 不會阻塞（await 等待）                     |
| 回傳型態        | `void`                | `Task` 或 `Task<T[]>`                 |
| 可否用 `await` | ❌ 不行                  | ✅ 可以                                 |
| 適合用在哪裡      | 傳統同步程式（Main, Console） | 非同步程式（async 方法）                      |
| 若任務拋出例外     | 直接在 `WaitAll` 時丟出     | 會包成 `AggregateException` 在 await 時丟出 |


你在等三個朋友（小明、小華、小美）買早餐

- Task.WaitAll → 你( Main Thread )站在門口發呆，不幹別的，遙望著著等待三個人回來
- Task.WhenAll → 你先交代他們去買早餐，然後自己去煮水、準備餐具。三個都回來後自動通知你可以弄早餐囉!


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-13-taskwhenall/7_wait_when.png)


<!-- endtab -->

<!-- tab 展現阻塞的差異-->

## 🔹 Task.WhenAll

```csharp
async Task Main()
{
	$"main thread 開始 whenall 測試 : ThreadId : {Thread.CurrentThread.ManagedThreadId}".Dump();
	var tasks = Enumerable.Range(1,3).Select(x => Do.DoWork(x)).ToArray();
	var result = Task.WhenAll(tasks);
	foreach(var i in Enumerable.Range(1,3))
	{
		$"main thread 繼續作業中 no.{i} ThreadId : {Thread.CurrentThread.ManagedThreadId}".Dump();
		await Task.Delay(500);
	}
	await result;
	"Main WhenAll 完成".Dump();
}

public static class Do
{
	public static async Task DoWork(int id)
	{
		Console.WriteLine($"工作 {id} 開始於執行緒 {Thread.CurrentThread.ManagedThreadId}");
		await Task.Delay(2000);
		Console.WriteLine($"工作 {id} 結束於執行緒 {Thread.CurrentThread.ManagedThreadId}");
	}
}
```

```bash
main thread 開始 whenall 測試 : ThreadId : 1
工作 1 開始於執行緒 1
工作 2 開始於執行緒 1
工作 3 開始於執行緒 1
main thread 繼續作業中 no.1 ThreadId : 1
main thread 繼續作業中 no.2 ThreadId : 5
main thread 繼續作業中 no.3 ThreadId : 5
工作 3 結束於執行緒 5
工作 1 結束於執行緒 17
工作 2 結束於執行緒 7
Main WhenAll 完成
```

三個 DoWork 任務同時啟動，但主執行緒仍然持續印出 主線程還在工作中...，主線程沒有被卡住；等待 2 秒後，任務都完成，才印出最後一句  

👉 這就是「非阻塞」

<br>

## 🔹 Task.WaitAll
```csharp
async Task Main()
{
	$"main thread 開始 whenall 測試 : ThreadId : {Thread.CurrentThread.ManagedThreadId}".Dump();
	var tasks = Enumerable.Range(1,3).Select(x => Do.DoWork(x)).ToArray();
	Task.WaitAll(tasks);
	foreach(var i in Enumerable.Range(1,3))
	{
		$"main thread 繼續作業中 no.{i} ThreadId : {Thread.CurrentThread.ManagedThreadId}".Dump();
		await Task.Delay(500);
	}

	"Main WhenAll 完成".Dump();
}

public static class Do
{
	public static async Task DoWork(int id)
	{
		Console.WriteLine($"工作 {id} 開始於執行緒 {Thread.CurrentThread.ManagedThreadId}");
		await Task.Delay(2000);
		Console.WriteLine($"工作 {id} 結束於執行緒 {Thread.CurrentThread.ManagedThreadId}");
	}
}
```

```plaintext
main thread 開始 whenall 測試 : ThreadId : 1
工作 1 開始於執行緒 1
工作 2 開始於執行緒 1
工作 3 開始於執行緒 1
工作 3 結束於執行緒 11
工作 2 結束於執行緒 12
工作 1 結束於執行緒 14
main thread 繼續作業中 no.1 ThreadId : 1
main thread 繼續作業中 no.2 ThreadId : 11
main thread 繼續作業中 no.3 ThreadId : 11
Main WhenAll 完成
```

Task.WaitAll 這行一執行後，主執行緒就「停住」；會發現 2 秒內主線程什麼都不做；直到三個任務都結束才印出 Main 繼續執行



![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-13-taskwhenall/8_block_nonblock.png)



<!-- endtab -->

{% endtabs %}