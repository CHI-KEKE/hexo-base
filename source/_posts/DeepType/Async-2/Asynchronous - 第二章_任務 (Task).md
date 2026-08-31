---
title: Asynchronous - 第二章:任務 (Task)
date: 2024-05-07 11:04:05
categories: 未来よりの返歌
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-2/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-2/1_landing.png
tags:
    - Asynchronous Programming
toc:
toc_number:
comments :
---

{% tabs Asynchronous2%}

<!-- tab breakfast-->

清晨，陽光還沒透進窗簾，我們站在廚房前，準備煮一頓早餐。那是個沒有 `await`、沒有 `Task` 的年代，一切流程只能靠一層層的 `callback` 撐起來。煮早餐這件事，得先等水滾、再控火候、再計時間，哪一步出錯，就得重來。每一個選擇都得接在前一個選擇的屁股後面，像在搭積木，一層接一層，沒有省略鍵


![breakfast](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/breakfast.png)


所謂的 `Callback`，其實本質就是：「這件事我現在做不完沒關係，我先把『之後要幹嘛』這個決定權丟出去，等你搞定了，再來叫我回來接著做。」事情還沒弄好，整個流程不卡死在原地

<br>

現實生活中有太多事都不是即刻完成的──水要時間才會滾、I/O 要等、網路請求要等、使用者要按個鈕、檔案要讀完……如果我們什麼都等完再做就成了 `Blocking`，導致執行緒被佔死、什麼事都不能做、UI 會卡、Server 吞吐會掉，`Blocking` 的問題不是「慢」，而是為了等一件事，犧牲了所有其他可能性


![12](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-2/2_non_blocking.png)


`Callback` 提供的方案是「我先把水放去燒，等你燒好了，再來叫我，我再接著走。」

```csharp
BoilWaterAsync(() =>
{
    BrewCoffee();
});
```


![12](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-2/3_callback.png)


<!-- endtab -->

<!-- tab callback in 早餐-->

煮水、烤麵包、沖咖啡、煎蛋，設計上如果要更精確地利用時間，應能讓 **程式知道「什麼時候可以開始下一步」**，我們不能一開始就同時烤麵包、煎蛋，因為你得等水煮好才能確定一切準備開始，我們也不能一個一個慢慢地依序做，烤完麵包才去煎蛋，煎蛋後才去煮開水，煮完水再去沖咖啡...，時間的利用沒有效率，只有一個人或許是合理的，但我們要開早餐店的話不能這樣運作

我們開始進行安排，寫下第一個 callback：水煮好後爐台空了我們開始做三件事。雖然每一件事不會即時完成，但我希望它們執行完成後，向我回報「早餐好了」，這些回報必須集中，並且要能知道「三件事都完成了嗎？」才能進一步呼叫最後的完成通知!

```csharp
// PrepareBreakfast 做完後要執行 onBreakfastReady
void PrepareBreakfast(Action onBreakfastReady)
{
    Console.WriteLine("開始準備早餐");

	// BoilWaterAsync 執行完呼叫其他行為
    BoilWaterAsync(() =>
    {
        Console.WriteLine("水煮好了，爐台空了，開始其他準備工作");

        int completedCount = 0;

        Action reportCompleted = () =>
        {
            if (Interlocked.Increment(ref completedCount) == 3)
            {
                Console.WriteLine("三件事都完成了，早餐準備好了");
                onBreakfastReady?.Invoke();
            }
        };

		// ToastBreadAsync / BrewCoffeeAsync / FryEggsAsync 做完後要執行 reportCompleted
        ToastBreadAsync(reportCompleted);
        BrewCoffeeAsync(reportCompleted);
        FryEggsAsync(reportCompleted);
    });
}

void BoilWaterAsync(Action callback)
{
    ThreadPool.QueueUserWorkItem(_ =>
    {
        Console.WriteLine("開始燒開水...");
        Thread.Sleep(3000);
        Console.WriteLine("開水燒好囉");
        callback?.Invoke();
    });
}

void ToastBreadAsync(Action callback)
{
    ThreadPool.QueueUserWorkItem(_ =>
    {
        Console.WriteLine("開始烤麵包...");
        Thread.Sleep(3000);
        Console.WriteLine("麵包完成");
        callback?.Invoke();
    });
}

void BrewCoffeeAsync(Action callback)
{
    ThreadPool.QueueUserWorkItem(_ =>
    {
        Console.WriteLine("開始沖咖啡...");
        Thread.Sleep(3000);
        Console.WriteLine("咖啡完成");
        callback?.Invoke();
    });
}

void FryEggsAsync(Action callback)
{
    ThreadPool.QueueUserWorkItem(_ =>
    {
        Console.WriteLine("開始煎蛋...");
        Thread.Sleep(3000);
        Console.WriteLine("煎蛋完成");
        callback?.Invoke();
    });
}


PrepareBreakfastAsync(() => {
	Console.WriteLine("所有早餐準備工作完成，可以開始享用了！");
});


// 開始準備早餐...
// 開始燒開水...
// 開水燒好囉
// 開水已經燒好，開始其他準備工作
// 開始煮咖啡...
// 開始烤麵包...
// 開始煎蛋...
// 麵包好囉
// 蛋好囉
// 咖啡好囉
// 早餐準備完成！
// 所有早餐準備工作完成，可以開始享用了！
```


<!-- endtab -->

<!-- tab callback 之難一. 線性程式碼回歸-->

我們為了把「有先後順序的非同步流程」串起來，讓事情一定照順序發生，等前一件事真的做完，再安全地做下一件事，避免順序亂掉

1. 呼叫 BoilWater 把一段「等水煮好後要做的事」包成 `callback` 傳進去，表示水還沒好，後面的事先不要動
2. 水煮好了，才會進到下一層 ToastBread，ToastBread 同樣的模式，烤麵包是另一個需要時間的動作，烤好之前，不會去煎蛋
3. FryEggs 完成後，最後才呼叫 Serve()

每一層 `callback` 都表示「我做完了，才輪到你」

```csharp
BoilWater(() =>
{
    ToastBread(() =>
    {
        FryEggs(() =>
        {
            Serve();
        });
    });
});

```

假如不寫成上面的 callback 層次，直接呼叫就沒有順序可言了! 水還在煮、麵包還沒烤、蛋還沒熟，你就已經上菜了，問題是「時間點不受你控制」

```csharp
BoilWater();
ToastBread();
FryEggs();
Serve();
```


<!-- endtab -->

<!-- tab callback 之難二. 自己管完成狀態-->

製作 callback 時，為了協調多個非同步 callback，會用「共享計數器」在全部完成時只觸發一次結束事件

```csharp
int completed = 0;
void Done()
{
    if (Interlocked.Increment(ref completed) == 3)
        AllDone();
}
```

1. completed 是「完成數量的共享狀態」，代表目前有幾個工作已經完成，預期會有 3 個非同步工作，各自在完成時呼叫 Done()
2. 每個工作完成時都呼叫 Done()，不管是 A、B、C 任一個，你不能假設順序，也不能假設誰最後完成
3. Interlocked.Increment，這一步是為了原子操作，需要保證在多執行緒同時呼叫 Done() 時不會遺失加 1、不會出現兩個人都以為自己是第 3 個，這裡是在對抗 Race Condition
4. 判斷是否「剛好是最後一個」== 3「誰加完後剛好變成 3，誰負責收尾」
5. 只呼叫一次 AllDone()，「完成的瞬間」才是唯一的同步點

這是一個「人工打造的 Join 點」，等所有平行任務完成 → 再繼續下一步，如果有多個層次就會顯得相當複雜



![12](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-2/3_callbackhell_manual_join.png)



<!-- endtab -->

<!-- tab callback 之難三. try/catch 沒用-->

如果在外層做 try / catch，會遺漏要 catch 得錯誤，有幾個原因

1. try / catch 只能接住「同一個呼叫堆疊（call stack）」裡拋出的例外，而 callback 裡的程式，根本不是在你現在這個 stack 上執行
2. try/catch 只抓「同步拋出」

**以為**
```bash
Main
 └── DoWork
      └── callback
           └── throw exception
```

**真相**
```bash
Main
 └── DoWork
      └── QueueUserWorkItem
           (return 了，stack 結束)

--- 很久以後 ---

ThreadPool Thread
 └── callback
      └── throw exception
```

DoWork() 早就 `return`，呼叫它的 `stack` 已經不存在，`callback` 是在「另一條 `thread`、另一個 `stack`」上執行，`try/catch` 根本不在同一條時間線上


```csharp
void Main()
{
    try
    {
        DoWork(() =>
        {
            throw new Exception("Boom");
        });
    }
    catch
    {
        Console.WriteLine("永遠不會進來");
    }

    Thread.Sleep(1000); // 讓背景 thread 有時間跑
}

public static void DoWork(Action action)
{
    ThreadPool.QueueUserWorkItem(_ =>
    {
        action(); // 例外在另一條 thread
    });
}
```
❌ catch 不會進來，Console App 直接 crash（Unhandled exception），錯誤直接「野放」


因此，callback 世界會這樣處理錯誤

```csharp
// 1️⃣ 自己 try/catch 在 callback 裡
ThreadPool.QueueUserWorkItem(_ =>
{
    try
    {
        DoSomething();
        callback(null);
    }
    catch (Exception ex)
    {
        callback(ex);
    }
});

// 2️⃣ callback 改成「錯誤回傳型」
void DoWork(Action<Exception> callback)

// 3️⃣ 呼叫端自己判斷

DoWork(ex =>
{
    if (ex != null)
    {
        Log(ex);
        return;
    }
});

```


![12](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-2/4_trycatch_miss.png)


<!-- endtab -->

<!-- tab 任務 ( Task )-->

Task 不是讓非同步變簡單，而是讓「時間線重新回到線性思考」。callback hell 的痛點，本質上不是「非同步很難」，而是人類不擅長用巢狀與狀態機思考時間，卻被迫用 callback 去描述「未來會發生的事」
Task 做的事，是把「未來」變成一個可被等待、可被組合、可被觀察的物件。你不用一直站在旁邊看著他工作，他會自己告訴你：「我好了」、「我失敗了」，甚至可以主動回報結果給你。你只需要在適當時機收割成果，或者處理錯誤就好。

`async/await` 就像是幫你寫了一套「任務筆記本」：你只要寫下「接下來要做什麼」，系統會幫你記得「做一半時停在哪」、「要等什麼人回來」、「接著做哪一步」。它幫你記住流程停在哪、在等誰、接下來要做什麼。把「暫停點」交給編譯器管理，我們只負責描述意圖，不用自己實作狀態機


![12](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-2/5_encap_task.png)


![12](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-2/6_state_machine.png)



- `await BoilWaterAsync()` 就表示：「先等水煮好，再繼續」
- `await Task.WhenAll(...)` 表示：「這幾件事可以一起做，等都做完再說」

每個方法都變成獨立、清爽、單一責任的小單元!

```CSHARP
public async Task PrepareBreakfastAsync()
{
	Console.WriteLine("開始準備早餐...");

	//// 開始燒水，但開水要先燒完才往下做，這時候的 Thread 被釋放去做其他事情，等開水燒完，某條 Thread 再回來繼續主流程
	await BoilWaterAsync();
	
	//// 全部做完才可以往下走
	await Task.WhenAll(BrewCoffeeAsync(),FryEggsAsync(),ToastBreadAsync());
	Console.WriteLine("早餐準備完成！");
}
```

模擬做早餐任務，每個任務都有一個 `await` 斷點，遇到這個斷點就會回到主程序並告訴主程序現在的狀態是甚麼，等到 `await` 的內容完成，就會再次從 `await` 下一步繼續走，因為狀態機已經記住剛才這個非同步方法執行到哪裡了

```CSHARP
public async Task BoilWaterAsync()
{
    Console.WriteLine("開始燒水...");
    await Task.Delay(2000); // 模擬燒水時間
    Console.WriteLine("水燒好了");
}

public async Task ToastBreadAsync()
{
    Console.WriteLine("開始烤麵包...");
    await FailMakingBreadAsync();
    Console.WriteLine("麵包烤好了");
}

public async Task BrewCoffeeAsync()
{
    Console.WriteLine("開始沖咖啡...");
    await Task.Delay(1500); // 模擬沖咖啡時間
    Console.WriteLine("咖啡沖好了");
}

public async Task FryEggsAsync()
{
    Console.WriteLine("開始煎蛋...");
    await Task.Delay(1500); // 模擬煎蛋時間
    Console.WriteLine("蛋煎好了");
}

```

執行結果
```CSHARP

void Main()
{
    Console.WriteLine("開始準備早餐流程");
    PrepareBreakfastAsync();
}


// 開始準備早餐流程
// 開始準備早餐...
// 開始燒水...
// 水燒好了
// 開始沖咖啡...
// 開始煎蛋...
// 開始烤麵包...
// 麵包烤好了
// 蛋煎好了
// 咖啡沖好了
// 早餐準備完成！
```


![12](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-2/7_task_whenall.png)



<!-- endtab -->

<!-- tab try/catch 有用了-->

Task 世界，錯誤會被捕捉並封裝在 Task 裡

```csharp
try
{
    await DoWorkAsync();
}
catch (Exception ex)
{
    // 一定會進來
}
```

因為 Exception 不再「直接炸 thread」，而是成為 Task 的 完成狀態（Faulted），await 會在「回到你這條時間線時」重新丟出，錯誤從「野放事件」變成「可被等待的結果」，這是 callback 世界做不到的


![12](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-2/8_await_task_exception_falted.png)


<!-- endtab -->


<!-- tab 自動錯誤拆包，不再面對可怕的 AggregateException-->

處理多個 `Task` 時，如果出現錯誤，很常會被 `AggregateException` 砸滿頭，還要手動 `.InnerExceptions` 把每個錯誤逐個拆出來處理。但 `await` 幫你做了這些事，它會自動把第一層包裝解開，讓你用熟悉的 try-catch 就能捕捉最裡層的例外。你不用再 `.Wait()`、什麼是 `.Result`，也不用手動檢查哪一個任務丟出例外。只要 `await` 它，錯誤就會直接丟給你 `catch`

```CSHARP

async void Main()
{
	try
	{
		await PrepareBreakfastAsync();
	}
	catch (AggregateException ex)
	{
		Console.WriteLine($"Caught AggregateException: {ex.Message}");
		foreach (var innerException in ex.InnerExceptions)
		{
			Console.WriteLine($"Inner Exception: {innerException.GetType().Name} - {innerException.Message}");
		}
	}
	catch (Exception ex)
	{
		Console.WriteLine($"Caught other Exception: {ex.GetType().Name} - {ex.Message}");
	}
}

public async Task PrepareBreakfastAsync()
{
	await Task.WhenAll(BoilWaterAsync(),ToastBreadAsync(),BrewCoffeeAsync(),FryEggsAsync());
}

private async Task BoilWaterAsync()
{
	await Task.Run(() => { throw new InvalidOperationException("無法燒開水"); });
}

private async Task ToastBreadAsync()
{
	await Task.Run(() => { throw new InvalidOperationException("無法烤麵包"); });
}

private async Task BrewCoffeeAsync()
{
	await Task.Run(() => { throw new InvalidOperationException("無法沖咖啡"); });
}

private async Task FryEggsAsync()
{
	await Task.Run(() => { throw new InvalidOperationException("無法煎蛋"); });
}


//// Caught other Exception: InvalidOperationException - 無法燒開水
```

<!-- endtab -->

<!-- tab 自動包裝 return，不必自己 new Task.Result-->

在 `async` 方法裡，當你用 `return` 回傳一個結果（像是字串），編譯器會自動幫你包裝成 `Task<T>` 的形式，你不需要自己 `return Task.FromResult(...)` 這麼麻煩

```CSHARP

async void Main()
{
	try
	{
		await PrepareBreakfastAsync();
	}
	catch (InvalidOperationException ex)
	{
		// 直接捕獲 InvalidOperationException，而不是 AggregateException
		Console.WriteLine($"Caught specific exception: {ex.Message}");
	}
	
}

public async Task PrepareBreakfastAsync()
{
	// 不需要自己包裝成 Task
	Console.WriteLine("早餐直接做好!");
}

// 早餐直接做好!

```

或者
```CSHARP

async void Main()
{
	try
	{
		var result = await PrepareBreakfastAsync();
		Console.WriteLine(result);
	}
	catch (InvalidOperationException ex)
	{
\		Console.WriteLine($"Caught specific exception: {ex.Message}");
	}
	
}

public async Task<string> PrepareBreakfastAsync()
{
	Console.WriteLine("早餐直接做好!");
	return "一人份早餐誰要吃";
}


// 早餐直接做好!
// 一人份早餐誰要吃

```


![12](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-2/8_auto_encap_task.png)


<!-- endtab -->


<!-- tab summary-->

![12](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-2/9_table.png)


![12](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-2/10_final.png)


![12](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-2/async-2.png)

<!-- endtab -->




{% endtabs %}