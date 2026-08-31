---
title: Asynchronous Programming - 第十四章 :async lambda
date: 2025-10-22 14:08:05
categories: 未来よりの返歌
top_img: https://github.com/CHI-KEKE/pics/blob/main/Codesss/Async/logoAsync.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Codesss/Async/logoAsync.png?raw=true
tags:
    - Asynchronous Programming
toc:
toc_number:
comments :
---

{% btn 'https://blog.darkthread.net/blog/linq-foreach-n-async/',LINQ ForEach() 與 async / await,far fa-hand-point-right %}

{% tabs Asynchronous%}

<!-- tab 問題的起點 — async lambda 與 Fire-and-Forget 陷阱-->

新的一年，動力滿格，誓言要變成更好的自己，我們開始報名課程、立下目標、說要健身、要學語言、要創業，那一刻就像丟出一個 Task 嚷嚷著我會做到的承諾。但接下來，沒有積極安排「等待與驗收」，例如固定回顧、截止日、交付物、找人督促，生活的 ForEach 將忙碌的人生繼續往下跑，我們會很快進入下一堆事情。最後你回頭看 results 可能是空的、有的只有一半（做到一半被別的事用掉了）、有時看起來做了很多，但沒有一個真正可用的成果，問題不在是否努力，是因為沒設計「誰來等你完成」!!!


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-14-foreach-trap/1_fire_and_forget_life.png)




<!-- endtab -->

<!-- tab 問題的起點 — async lambda 與 Fire-and-Forget 陷阱-->

```csharp
urls.ForEach(async url =>
{
    var response = await client.GetAsync(url);
    results.Add(url, await response.Content.ReadAsStringAsync());
});
```

實際上這段 code 有兩個問題


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-14-foreach-trap/2_trap_2.png)



- `Fire and Forget`：`ForEach` 本身不會等待內部的 `async` 動作完成，等於是「丟出去就不管了」

async 「先把控制權還出去，等結果準備好再回來」，所以拿到的是「承諾會有結果的 `Task`」，而不是結果本身

1. async Lambda 被建立，這個 Lambda 的型別是：Func<string, Task>，它「不是馬上做完事情」，而是「回傳一個 Task」
2. 執行到 await client.GetAsync(url)，HTTP Request 發出去，還沒拿到回應時，這個方法「暫停」，執行緒被釋放回 Thread Pool（重點）
3. HTTP 回來後，繼續往下跑，讀取 response content，組成字串整個 Lambda 結束
4. Task 標記為 Completed「未來的結果」才正式完成，但 ForEach 跑完，整個方法會直接往下執行，所以可能會看到results 有時是空的、有時只有一半、有時還沒加完就被用掉了

👉 問題不在 async 本身，而在「誰有在等這個 Task」


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-14-foreach-trap/3_foreach_async_just_task_trap.png)


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-14-foreach-trap/5_release_the_control.png)



![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-14-foreach-trap/6_task_lifetime.png)




- `Thread Safety`：results 是共用的字典，在多個非同步任務同時 `Add()` 時，可能會出現執行緒安全（`Thread Safety`）問題



![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-14-foreach-trap/4_thread_safty_problem.png)




<!-- endtab -->

<!-- tab 解法 1：簡單但不平行-->

```csharp
foreach (var url in urls)
{
    var response = await client.GetAsync(url);
    results.Add(url, await response.Content.ReadAsStringAsync());
}
```

這樣的寫法雖然「安全又可預期」，但每個請求都要等上一個完成後才能繼續下一個，是「排隊」的概念



![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-14-foreach-trap/7_solA_step_by_step.png)



<!-- endtab -->

<!-- tab 解法 2：真正的非同步 + 平行執行-->

```csharp
var tasks = urls.Select(async url => {
    var response = await client.GetAsync(url);
    var content = await response.Content.ReadAsStringAsync();
    return (Url: url, Content: content);
});

var result = await Task.WhenAll(tasks);
var dicts = result.ToDictionary(x => x.Url, x => x.Content);
```

這段才是「真正的非同步」，他透過 `Select()` 建立一組非同步任務（每個 URL 對應一個 `Task`），並且使用 `Task.WhenAll()` 等待所有任務完成，回傳的結果可以整齊地整理成字典（Dictionary）


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-14-foreach-trap/9_solb_concurrency.png)



<!-- endtab -->

<!-- tab LINQ 的 Select 不會「馬上執行」-->

```csharp
urls.Select(async url => { ... });
```

因為 LINQ 的 `Select()` 是 延遲執行（Deferred Execution） 的，它只是一個「規劃」，直到你真的去「取用結果」時（例如 `.ToList()`、`.Count()`、`await Task.WhenAll()`），它才會開始跑


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-14-foreach-trap/8_select_is_delay.png)



<!-- endtab -->

<!-- tab Code-->

```csharp
async Task Main()
{
	var client = new HttpClient();
	
	var urls = new List<string>()
	{
		"https://v2.jokeapi.dev/joke/Any?blacklistFlags=nsfw",
		"https://v2.jokeapi.dev/joke/Any?lang=ru"
	};

	//// Fire And Forget & ThreadSafe
	//var results = new Dictionary<string, string>();
	//urls.ForEach(async url =>
	//{
	//	var response = await client.GetAsync(url);
	//	results.Add(url, await response.Content.ReadAsStringAsync());
	//});
	
	//// 解法 1 但會一個個做不會平行執行
	//foreach (var url in urls)
	//{
	//	var response = await client.GetAsync(url);
	//	results.Add(url, await response.Content.ReadAsStringAsync());
	//}
	
	//// 解法 2 非同步 + 平行執行 (尚未執行)
	var tasks = urls.Select(async url => {
		var response = await client.GetAsync(url);
		var content = await response.Content.ReadAsStringAsync();
		//// return () 後續可以解構
		return (Url: url, Content: content);
	});
	
	//// 開始 enumerate tasks (消費)
	var result = await Task.WhenAll(tasks);
	//// 這裡就好分辨
	var dicts = result.ToDictionary(x => x.Url, x => x.Content);
	
	//// 若僅僅 select 會是延遲執行所以不會做任何事情
	dicts.Select(dic => {
		Console.WriteLine($"url : {dic.Key}\n content : {dic.Value}");
		return dic;
	}).ToList();
}
```


<!-- endtab -->



<!-- tab summary-->


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-14-foreach-trap/Mastering_Async_Waiting.png)


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-14-foreach-trap/10_table.png)



<!-- endtab -->


{% endtabs %}