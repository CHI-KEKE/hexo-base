---
title: EF Core - TimeoutException PART 2
date: 2025-05-28 08:26:05
categories: 資料疆界的航圖
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout2/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout2/1_landing.png
tags:
    - EF CORE
toc:
toc_number:
comments :
---



{% tabs EF Core - TimeoutException PART 2%}

<!-- tab Pre-->

本篇 ~ 繼續 ~ TimeoutException 可能發生的原因及改善方式，從 Lock、Isolation Level、Retry、連線池，到 Change Tracker 與批次處理


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout2/1_whisper.png)


<!-- endtab -->

<!-- tab 為什麼資料庫會「卡住」？—— 談 Lock-->


## 🔒 資料庫為什麼會 Lock？

當兩個交易（Transaction）同時存取資料時，為了確保資料一致性，資料庫會自動幫我們上鎖。像是：

- 其他交易正在修改這筆資料（而還沒提交 Commit）
- 你要修改的資料已經被排他鎖定

這時候，後來的交易就只能等別人釋放鎖，不然就只能拋錯。等待超過 Command Timeout，就會拋出 TimeoutException 或 SqlException。

<br>
<br>

##　關鍵例外訊息可能會有哪些？


```bash
System.TimeoutException

Microsoft.Data.SqlClient.SqlException

常見訊息：Timeout expired. The timeout period elapsed prior to completion of the operation or the server is not responding.

或：Transaction (Process ID xx) was deadlocked on resources with another process and has been chosen as the deadlock victim.
```

這些錯誤看起來很兇，其實是資料庫在告訴你：「欸，有人先來排隊，但你等太久我就不等了囉！」


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout2/2_deadlock.png)


<!-- endtab -->

<!-- tab 方向一. 從交易層級（Transaction Isolation Level）開始說起-->

所謂設定「交易隔離層級（Isolation Level）」是指

> 告訴資料庫我們希望這個交易要多“ 保護資料的一致性 ”，或是能不能「容忍髒讀」。

但這些設定在哪裡呢?

例如常使用的 SaveChanges() 儲存操作，它預設會自己包一個「隱藏版的交易」，而這個交易的隔離層級是由資料庫決定的（像 SQL Server 是預設 ReadCommitted）。而 ReadCommitted 只讀取「已經被其他交易提交的資料」，不讀取還沒 Commit 的。這雖然保證了資料一致性，但也比較容易造成鎖定。

今天我們若可以接受髒讀，可以主動設定較寬鬆的隔離層級來降低被鎖機率，就需要自己開 Transaction 包起來
```CSHARP
using var transaction = context.Database.BeginTransaction(System.Data.IsolationLevel.ReadUncommitted);

try
{
    // 所有的查詢與 SaveChanges 都包在這個隔離層級中
    context.MyEntities.Add(new MyEntity { Name = "Test" });
    context.SaveChanges();

    transaction.Commit();
}
catch
{
    transaction.Rollback();
    throw;
}
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout2/2_isolationlevel.png)


<!-- endtab -->

<!-- tab 方向二. 設計　Retry 機制-->


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout2/3_temporary_fail.png)


遇到 TimoutExcpetion 被 lock 住，也可以嘗試重試，但可以拆解成幾個面向思考

<br>

## 最大重試次數（Max Retry Count）

避免無限 Retry 造成資源浪費或風暴式災難

```CSHARP
const int maxRetry = 3;
```

<br>

## 延遲策略（Delay / Backoff）

每次失敗後要延遲多久時間再重試。例如我們可以設計

- 固定延遲：每次都等 2 秒
- 指數回退（Exponential Backoff）：等 2、4、8 秒
- 加上隨機 jitter（抖動）防止所有程式同時重試

```CSHARP
var delay = TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)) + TimeSpan.FromMilliseconds(Random.Shared.Next(100, 500));
```

<br>

## 重試條件是甚麼?（Retry On What）

不是所有錯都該 Retry，像參數錯、邏輯錯就不該 Retry。

```CSHARP
bool IsTransient(Exception ex)
{
    if (ex is SqlException sqlEx)
    {
        return sqlEx.Number == -2 || sqlEx.Number == 1205 || sqlEx.Number == 4060;
    }
    return ex is TimeoutException;
}
```



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout2/3_retry.png)



## 避免副作用（Idempotency）


如果你的 Retry 會「重複寫入資料」，就要考慮

- 資料是否可重複執行
- 要不要先判斷資料是否已寫入（去重）
- 是否加唯一鍵（避免插入相同資料）


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout2/3_idempotency.png)


<!-- endtab -->

<!-- tab 方向三. 避免長時間交易-->

你可能聽過「不要開太久的交易」，其實這不是指你 BeginTransaction() 本身時間設限，而是你交易裡頭做的事，導致資料庫資源一直鎖住

先來了解一下有兩個 timeout 會影響交易存活

## A. CommandTimeout（指令逾時時間）

這會影響你的 SaveChanges()、SQL 查詢指令多久等不到就爆錯。EF Core 預設是 30 秒（除非你另外設定）

```CSHARP
optionsBuilder.UseSqlServer(connStr, opt => opt.CommandTimeout(60));
```


## B. TransactionScope.Timeout

這是專門用來控制 TransactionScope 的最大生存時間（預設是 1 分鐘）！
```CSHARP
var options = new TransactionOptions
{
    Timeout = TimeSpan.FromSeconds(30),
    IsolationLevel = IsolationLevel.ReadCommitted
};

using (var scope = new TransactionScope(TransactionScopeOption.Required, options))
{
    // ... 操作
    scope.Complete();
}
```

如果超過時間，會自動 Rollback。建議避免在交易中做耗時的事（呼叫 Web API、讀檔、Thread.Sleep），保持交易越短越好並且只把必要的 DB 操作包在交易中



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout2/4_transactiontime.png)



<!-- endtab -->

<!-- tab 連線池用盡 / 資料庫連不穩-->


EF Core 預設使用 ADO.NET 的連線池機制。若應用程式短時間內開啟太多連線、卻沒及時釋放，就會導致：

```bash
Timeout expired. The timeout period elapsed while attempting to obtain a connection from the pool.
```

原因常見有：

- 使用 await 卻忘了加 using，導致 DbContext 未釋放
- 長時間背景工作（如批次處理）未妥善管理 DbContext
- 並發量高，連線數不夠用
- 資料庫本身忙碌（例如 CPU 滿載）



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout2/5_connection_exhausion.png)



<br>

## 改善方式

#### 養成用完就釋放的習慣

```CSHARP
await using var context = new MyDbContext(); // EF Core 7 支援 await using
```

或使用 AddDbContext + Scoped 生存週期搭配依賴注入。


#### 使用 DbContext Pooling（EF Core 支援）

如果你想重複利用已初始化的 DbContext，可用 Pool 機制來減少建構開銷

```CSHARP
services.AddDbContextPool<MyDbContext>(options => 
    options.UseSqlServer(connectionString));
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout2/5_2_dbcontext_pool.png)


<!-- endtab -->

<!-- tab 沒用 AsNoTracking()-->

AsNoTracking 是一個指令，告訴 EF Core：「我只是要讀資料，不會去修改它們，請不要幫我追蹤這些資料。」
很多時候，我們只是查資料出來顯示在前端畫面上（例如查詢商品清單、報表、訂單列表），並不會修改資料。這時候，如果你沒用 AsNoTracking()，EF Core 會白白幫你「追蹤」這些資料，浪費效能。

<br>

## EF Core 的 Change Tracker 是怎麼運作的？

每次你用 context.Users.ToList() 查資料時，EF Core 會把結果加到一個叫做「ChangeTracker」的物件中。

這個物件會記住：

- 這筆資料是哪個表格的？
- 原始值是什麼？
- 有沒有被改過？

這樣做的目的是：「萬一你之後要呼叫 SaveChanges()，我可以幫你知道哪些資料有被改動，然後自動幫你存回去！」

因此大量查詢時會吃掉大量記憶體與 CPU，重複查同一筆資料會回傳同一個實體，有時候你不想要這種行為（因為可能你要比較不同快照）。讀取用查詢也會引入不必要的狀態追蹤機制，尤其在 API、報表、DataGrid 等情境中，這是多餘的。



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout2/6_change_tracker.png)



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout2/6_2_as_no_tracking.png)



<!-- endtab -->

<!-- tab 大量更新或刪除資料-->

在 EF Core 中，如果你直接對大量資料執行更新（Update）或刪除（Delete）操作，可能會遇到效能瓶頸，甚至導致應用程式記憶體暴增或逾時。

這是因為 EF Core 預設的操作流程是這樣的：

1. 先查出符合條件的所有資料。
2. 把這些資料加入 Change Tracker（用來追蹤每一筆資料的狀態變化）。
3. 對每筆資料做變更。
4. 呼叫 SaveChanges() 時，EF Core 會一筆一筆產生 SQL 指令，並依序送到資料庫執行。

這種做法在處理少量資料時沒什麼問題，但如果你要處理幾百、幾千，甚至幾萬筆資料，效能就會急遽下降，因為：

- 記憶體被 Change Tracker 撐爆（每筆資料都在記憶體中追蹤）。
- 每筆資料都是獨立發出 SQL 指令，無法有效合併成一筆批次 SQL。
- SaveChanges() 的執行時間會拖很長。

<br>

## 使用 ExecuteSqlRaw() 或 ExecuteUpdate() / ExecuteDelete()（EF Core 7+）

從 EF Core 7 開始，你可以使用 ExecuteUpdate() 和 ExecuteDelete() 方法來執行批次操作。這些方法會直接產生 SQL 語法並送到資料庫執行，不經過 Change Tracker，也不需要先查出資料

批次更新（更新所有未啟用的使用者，把他們設為已啟用）
```CSHARP
// EF Core 7+
context.Users
    .Where(u => !u.IsActive)
    .ExecuteUpdate(u => u.SetProperty(x => x.IsActive, true));
```

批次刪除（刪除 30 天前的暫存資料）
```CSHARP
// EF Core 7+
context.TempData
    .Where(x => x.CreatedAt < DateTime.Now.AddDays(-30))
    .ExecuteDelete();
```

或使用原生 SQL 指令
```CSHARP
context.Database.ExecuteSqlRaw("DELETE FROM TempData WHERE CreatedAt < {0}", DateTime.Now.AddDays(-30));
```

<br>
<br>

## 分批處理（Batch）

如果你仍需要修改資料並依賴 Change Tracker（例如進行複雜的物件狀態轉換、觸發領域事件等），可以考慮分批處理 + 清空 Change Tracker。

```CSHARP
const int batchSize = 100;
var total = await context.Users.CountAsync();

for (int i = 0; i < total; i += batchSize)
{
    var users = await context.Users
        .Skip(i).Take(batchSize)
        .ToListAsync();

    foreach (var user in users)
    {
        user.IsActive = true; // 做你要的事
    }

    await context.SaveChangesAsync();
    context.ChangeTracker.Clear(); // 避免記憶體爆炸
}
```

控制記憶體用量、減少 ChangeTracker 負擔、安全地逐批處理



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout2/7_chunking.png)





> EF Core 在大量更新刪除時，預設方式很沒效率，請善用 ExecuteUpdate() / ExecuteDelete()，或「分批處理 + 清空 ChangeTracker」，才能讓系統跑得順、跑得快

<!-- endtab -->

<!-- tab Summary-->

每次 Timeout，不只是錯誤訊息，而是系統在對你說話。它可能在說：「我被鎖太久了」「有人沒還我資源」「你一次丟太多給我了」「我連線用光了」……

當我們開始理解錯誤背後的故事，就能用正確的方式回應它：

- 該設的隔離層級就設
- 該用 AsNoTracking() 就不要手軟
- 該重試的時候別忘了考慮副作用
- 該批次處理時也別讓 Change Tracker 扛太多

優雅地處理錯誤，從來都不是一件冷冰冰的工程技巧，而是讓系統、資料與團隊彼此合作的橋梁。願這篇文章，能幫你修好這座橋，讓 Timeout 不再是令人焦慮的未知，而是一個可以好好對話的訊號


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout2/8_table.png)


<!-- endtab -->


{% endtabs %}