---
title: Asynchronous Programming - 第四章:彼岸未歸的異步任務
date: 2024-08-31 10:03:05
categories: 未来よりの返歌
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-4-async-exception/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-4-async-exception/1_landing.png
tags:
    - Asynchronous Programming
toc:
toc_number:
comments :
---

{% tabs Asynchronous2%}

<!-- tab textMessage-->


![textMessage](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/textmessage.png)


她在深夜寫了一段訊息，對著那個熟悉的對話框，輕輕按下「傳送」

網路似乎是卡住了，畫面顯示「正在傳送…」，她關上了手機，心想：「應該已經送出了吧。」

隔天，她等不到回應。那人說沒收到。

她打開訊息記錄，那串話仍顯示為「未送達」，就像訊息從未存在過

一個沒有被接收的訊息，不會自動變成世界的理解；一個沒有被等待的結果，也無法保證會如你所想地完成



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-4-async-exception/2_message.png)



<!-- endtab -->

<!-- tab 在同步環境中強行等待非同步 Task 完成-->

在實務開發中，我們經常面臨一種情境：必須等待某個非同步任務完成，才能繼續執行接下來的邏輯。但如果我們當下的環境是同步方法（例如 Main() 或某些事件處理函式），該怎麼辦？



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-4-async-exception/3_sync_async.png)



這時，許多開發者會選擇以下方式強行「同步化」等待

- `.Wait()`
- `.Result`
- `GetAwaiter().GetResult()`


它們都會強迫等待 Task 結束，但錯誤處理機制卻有天壤之別，而這個差異，正是許多初學者第一次遇到的陷阱。
以下，我們將模仿實驗，實際觀察其錯誤行為與影響，像是 `.Wait()` 和 `.Result` 不是把錯誤丟給你，而是「包一層再丟」，讓你更難看清真相


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-4-async-exception/4_force_to_sync_3.png)


<!-- endtab -->

<!-- tab 實驗篇-->

[實驗出處](https://blog.darkthread.net/blog/getawaiter-getresult-vs-result/)

你有一個 async 方法
```csharp
static async Task<string> SumulateDatabaseConnectionAsync()
{
	Thread.Sleep(2000);
	if (DateTime.Now > new DateTime(2020, 12, 12))
	{
		throw new InvalidDataException("db 連壞掉啦!!");
	}
	
	return "db 連到啦";
}

```

你身處同步世界
```csharp
static void Main()
```

我們用同一個會「一定丟錯」的 async 方法，換三種同步等待方式，看 catch 到的是什麼鬼東西
```CSHARP
static void Main()
{
	TestSip("Wait", () => {
		SumulateDatabaseConnectionAsync().Wait();
	});

	TestSip("Result", () =>
	{
		var result = SumulateDatabaseConnectionAsync().Result;
	});

	TestSip("使用 GetAwaiter().GetResult()", () =>
	{
		var result = SumulateDatabaseConnectionAsync().GetAwaiter().GetResult();
	});


	TestSip("Fire And Forget", async () => {
		await SumulateDatabaseConnectionAsync();
	});
	
 	WriteColorLine("主程序結束!", ConsoleColor.Cyan);
}

static void TestSip(string testName, Action callback)
{
	WriteColorLine("=========================================", ConsoleColor.Yellow);
	WriteColorLine($"測試 : {testName}", ConsoleColor.Green);
	WriteColorLine($"開始時間 : {DateTime.Now:HH:mm:ss}", ConsoleColor.Gray);

	try
	{
		callback();
		WriteColorLine($"結束時間 : {DateTime.Now:HH:mm:ss}", ConsoleColor.Gray);
		WriteColorLine("操作成功完成", ConsoleColor.Green);
	}
	catch (Exception ex)
	{
		WriteColorLine($"結束時間 : {DateTime.Now:HH:mm:ss}", ConsoleColor.Red);
		WriteColorLine($"錯誤 : {ex.Message}", ConsoleColor.Red);
		if (ex.InnerException != null)
		{
			WriteColorLine($"內部錯誤 : {ex.InnerException.Message}", ConsoleColor.Red);
		}
	}
}

static void WriteColorLine(string message, ConsoleColor colorName)
{
	Console.ForegroundColor = colorName;
	Console.WriteLine(message);
	Console.ResetColor();
}
```

<!-- endtab -->

<!-- tab 結果分析-->

![結果分析](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Async/asyncToSyncExceptionCatch.png?raw=true)

## .Wait() / .Result

`.Wait()` 和 `.Result` 是 「同步阻塞 async Task」 的做法，在 .NET 設計上一個 Task 可能同時有多個例外，所以它一定用 AggregateException 當外殼，它不是把錯誤丟給你，而是先幫你包一個你沒要的殼再丟。.Wait() / .Result 會把真正的例外包進 AggregateException，當我們 catch (Exception ex) 時 ex.Message 是 假的，真正錯誤藏在 InnerException



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-4-async-exception/5_2_wait_result_agg.png)




## GetAwaiter().GetResult()

它模擬的就是 await 的行為，只是少了語法糖，async 裡丟什麼，你就接到什麼，而 stack trace 也比較乾淨



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-4-async-exception/5_3_getawaiter.png)



## Fire-and-Forget（async lambda）

你以為你在等，其實你只是假裝沒看到錯誤。TestSip 接的是 Action，async lambda 會變成 async void，例外不會回到 TestSip、也 catch 不到，直接交給 runtime


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-4-async-exception/5_4_fire_and_forget.png)


<!-- endtab -->


<!-- tab summary-->


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-4-async-exception/6_table.png)


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-4-async-exception/7_final.png)



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-4-async-exception/async=ghost.png)


<!-- endtab -->

{% endtabs %}