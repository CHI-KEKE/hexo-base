---
title: EF Core - 交易的界線
date: 2025-05-25 11:04:05
categories: 
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction_corss/1_lost.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction_corss/1_lost.png
tags:
    - 
toc:
toc_number:
comments :
---


{% tabs EF Core - 交易的界線%}

<!-- tab 迷航-->

「啊…我有開 TransactionScopeAsyncFlowOption.Enabled 嗎？」
「咦？這兩個 Context 是不是不同連線？」
「等一下，我是不是跨資料庫了？那 MSDTC 呢？」

交易的邊界，就像洋流與領海，有時你以為你還在安全區，其實早已飄出防線。

這篇筆記，就是一趟從這場 Bug 航行而來的實驗紀錄 —— 我們用四種實際情境來測試 EF Core 的交易能力：

- 同資料庫、不同 DbContext
- 跨資料庫
- 共用連線 vs 不共用連線
- TransactionScope vs BeginTransaction

如果你也曾在交易邊界迷航過，歡迎登船


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction_corss/1_lost.png)


<!-- endtab -->

<!-- tab 實驗開航前：DBContext 的布陣-->

為了這趟「交易航線實驗」，我們準備了三個 DbContext：

- AdventureWorks2022：一個普通的 DbContext。
- AdventureWorks2022V2：與 AdventureWorks2022 使用相同的資料庫，但是不同的 DbContext 實例。
- NexCommerce_CouponContext：指向另一個資料庫。

```CSHARP
builder.Services.AddDbContext<AdventureWorks2022>(options => options.UseSqlServer(configurationManager.GetConnectionString("AdventureWorks2022")));
builder.Services.AddDbContext<AdventureWorks2022V2>(options => options.UseSqlServer(configurationManager.GetConnectionString("AdventureWorks2022")));
builder.Services.AddDbContext<NexCommerce_CouponContext>(options => options.UseSqlServer(configurationManager.GetConnectionString("NexCommerce_Coupon")));
```
這讓我們可以模擬「同庫多 Context」與「跨庫交易」兩種常見情境


![v](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction_corss/1_ships.png)


<!-- endtab -->

<!-- tab 實驗一：TransactionScope 實測雙 DbContext 同一資料庫-->


![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction_corss/2_1_same_db_differentcontext_scope.png)



我們嘗試用 TransactionScope 包住兩個 DbContext（AdventureWorks2022 與 AdventureWorks2022V2）的儲存行為，看是否能夠成功提交，或者在其中一個失敗時一併 Rollback

## 實驗

```CSHARP
public async Task TestTransactionScopeWith2SameDBContext()
{
    var transactionOptions = new TransactionOptions
    {
        IsolationLevel = IsolationLevel.ReadCommitted,
        Timeout = TransactionManager.DefaultTimeout
    };
    using (var transactionScope = new TransactionScope(
        TransactionScopeOption.Required,
        transactionOptions,
        TransactionScopeAsyncFlowOption.Enabled))
    {
        try
        {
            this._adventureWorks2022DbContext.Rds.Add(new Rd { RdName = "adven1", DeptCode = "AD-1" });
            await this._adventureWorks2022DbContext.SaveChangesAsync();
            this._adventureWorks2022DbContextV2.Rds.Add(new Rd { RdName = "adven2", DeptCode = "AD-2" });
            await this._adventureWorks2022DbContextV2.SaveChangesAsync();
            transactionScope.Complete();
        }
        catch (Exception ex)
        {
            System.Console.WriteLine($"{ex.Message} \n {ex.InnerException?.Message}");
        }
    }
}
```

<br>

✅ 結果：成功儲存！
![TransactionScopeSameDB](https://github.com/CHI-KEKE/pics/blob/main/EF/Transaction/TransactionScopeSameDB.png?raw=true)


![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction_corss/2_2_pass_scope.png )


<br>

但我們再進一步讓 V2 寫入錯誤資料（例如超過長度限制的欄位值），觀察是否會 Rollback

```CSHARP
this._adventureWorks2022DbContextV2.Rds.Add(new Rd { RdName = "adven2", DeptCode = "TooLONG!!!!!!!!!!!!!!!!" });
```

<br>

💥 結果：例外發生，兩邊都沒有寫入(觀察資料庫無新增資料)

```Bash
String or binary data would be truncated in table 'AdventureWorks2022.dbo.RD', column 'DeptCode'...
```
![TransactionScopeSameDBRollbackException](https://github.com/CHI-KEKE/pics/blob/main/EF/Transaction/TransactionScopeSameDBRollbackException.png?raw=true)


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction_corss/2_3_rollback_for_too_long.png)


<br>


## 經過

- `new TransactionScope(..., TransactionScopeAsyncFlowOption.Enabled)` 建立 `ambient transaction`，並讓它能穿過 `async/await`

- `ctx1.SaveChangesAsync()`：EF Core 會開啟連線（預設 `Enlist=true`），偵測到 `Transaction.Current` 後把這條連線 `enlist` 進 `ambient transaction`

- `ctx2.SaveChangesAsync()`：再開一條（可能是另一條）連線，也同樣 `enlist` 進同一個 `ambient transaction`

- `Complete()` 代表你允許提交；如果沒呼叫 `Complete()` 或中途丟例外，scope dispose 時就會回滾整個 `ambient transaction`


ctx1 成功寫入，ctx2 因欄位長度失敗丟例外，因為兩個都在同一個 `ambient transaction` 裡，所以最後整包 rollback，DB 看不到任何新增資料



## 連線池是否視為同一組」

連線字串一樣不是「才能同交易」的規則，它比較像是「比較容易不升級/比較容易共用到同一個連線池行為」的加分項，同一個 TransactionScope 能不能包住兩個 DbContext 不靠「字串一樣」，靠的是每條連線在 Open() 時能不能 enlist 到 Transaction.Current（預設會），字串完全一致在實務上常被拿來當條件，是因為它會影響「連線池是否視為同一組」；但即使字串一樣，你仍可能開到兩條不同的實體連線，而這就牽涉到後面「會不會升級」的風險，SQL Server provider 偵測同一 DB 會促成 promotable transaction」，關鍵是「資源管理員數量/連線開啟模式」而不是字串字面相等

## AsyncFlowOption.Enabled

用 async/await 還想靠 ambient transaction 串起來，就要開 AsyncFlow，不然交易上下文會在 await 後斷掉


## MSDTC

TransactionScope 可能在「開了第二個耐久資源」時自動升級成分散式交易（2PC / MSDTC）。即使同一個 SQL Server / 同一個 DB，只要你在同一個 scope 中讓多條連線同時 enlist，有些情況仍可能被升級。「只涉及一個資料來源通常不會升級」這句在很多簡單案例成立，但不保證，會受 provider 版本、連線開啟/關閉時機、是否同時開多連線等影響



<!-- endtab -->

<!-- tab 實驗二：TransactionScope 跨資料庫-->


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction_corss/2_4_dif_db_scope.png )


這次我們把 AdventureWorks2022 與 NexCommerce_CouponContext 放在同一個 TransactionScope 中


## 實驗

```CSHARP
/// <summary>
/// 因為跨 DB 要使用 TransactionScope 會因為沒開啟支援分散式交易噴掉
/// </summary>
public async Task TestTransactionScopeWithDiffDbsDBContext()
{
    var transactionOptions = new TransactionOptions
    {
        IsolationLevel = IsolationLevel.ReadCommitted,
        Timeout = TransactionManager.DefaultTimeout
    };
    using (var transactionScope = new TransactionScope(
        TransactionScopeOption.Required,
        transactionOptions,
        TransactionScopeAsyncFlowOption.Enabled))
    {
        try
        {
            //// adventureWorks2022 DB
            this._adventureWorks2022DbContext.Rds.Add(new Rd { RdName = "BBB",DeptCode = "avb" });
            await this._adventureWorks2022DbContext.SaveChangesAsync();

            //// nexCommerce DB
            this._nexCommerce_CouponContext.Coupons.Add(new Coupon
            {
                CouponId = 12,
                DiscountAmount = 100,
                MinAmount = 100
            });
            await this._nexCommerce_CouponContext.SaveChangesAsync();
            transactionScope.Complete();
        }
        catch (Exception ex)
        {
            System.Console.WriteLine($"{ex.Message} \n {ex.InnerException?.Message}");
        }
    }
}
```

<br>

🚫 失敗訊息：
```bash
Implicit distributed transactions have not been enabled. If you're intentionally starting a distributed transaction, set TransactionManager.ImplicitDistributedTransactions to true.
```

💡 原因是：跨資料庫的操作會啟用「分散式交易」，這需要額外設定（例如 MSDTC）並明確開啟：
![DistributedException](https://github.com/CHI-KEKE/pics/blob/main/EF/Transaction/DistributedException.png?raw=true)


## 經過

- `TransactionScope` 建立 `ambient transaction`，並用 Enabled 讓它能跨 await。

- 第一個 DbContext `SaveChangesAsync()`：第一條 SQL 連線 enlist 進 `ambient transaction`。

- 第二個 DbContext `SaveChangesAsync()`：開了第二個資料庫連線（另一個 DB），系統判定「有兩個耐久資源」→ 交易升級成分散式交易（DTC）。

- 在 .NET（特別是 .NET 7+ 這條路徑）預設 `TransactionManager.ImplicitDistributedTransactions` 是 false，所以一旦要升級就直接丟 `NotSupportedException`，就看到那句錯誤


## 跨「兩個資料庫」放進同一個 TransactionScope

同一個 TransactionScope 內碰到兩個不同 DB（兩個獨立的耐久資源）時，很容易觸發升級成分散式交易（DTC）；而在較新的 .NET 上，預設不允許「隱式」升級，所以你會看到那句 Implicit distributed transactions have not been enabled...


## 升級到 DTC


- 程式面：要不要允許升級（TransactionManager.ImplicitDistributedTransactions = true）
- 環境面：Windows / 網路 / MSDTC 設定是否允許跨機器、跨網段、權限等


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction_corss/2_5_implicit_msdc_fail.png)



<!-- endtab -->

<!-- tab 實驗三：BeginTransaction() 建出來的交易只能用在建立它的那條 connection 上-->


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction_corss/3_begin_tran_dif_conn.png)


改用 BeginTransaction()。這次我們觀察若 DbContext 雖然指向同一 DB，但使用不同的連線，交易會成功嗎？

<br>

```CSHARP
/// <summary>
/// 連線到相同 DB 但使用不同的連線時，會遇到連線錯誤
/// </summary>
public async Task TestBeginTransactionDiffConnectionFail()
{
    using (var connection = this._adventureWorks2022DbContext.Database.GetDbConnection())
    {
        var transaction = this._adventureWorks2022DbContext.Database.BeginTransaction();
        try
        {
            this._adventureWorks2022DbContext.Add(new Rd
            {
                RdName = "BeginTran1",
                DeptCode = "BT-1"
            });

            await this._adventureWorks2022DbContext.SaveChangesAsync();

            //// 
            /// ✅ 確實把「同一個 DbTransaction 物件」交給 ctx2 了 
            /// ❌ 但 ctx2 的 connection 不是建立這個 transaction 的那條(其實 _adventureWorks2022DbContextV2已經注入 有自己的連線)，因此不允許用
            this._adventureWorks2022DbContextV2.Database.UseTransaction(transaction.GetDbTransaction());
            this._adventureWorks2022DbContextV2.Add(new Rd
            {
                RdName = "BeginTran2",
                DeptCode = "BT-2"
            });

            await this._adventureWorks2022DbContextV2.SaveChangesAsync();
            await transaction.CommitAsync();
        }
        catch (Exception ex)
        {
            System.Console.WriteLine($"{ex.Message} \n {ex.InnerException?.Message}");
            await transaction.RollbackAsync();
        }
    }
}
```

<br>

```bash
The specified transaction is not associated with the current connection. Only transactions associated with the current connection may be used.
```

想共享同一個 BeginTransaction 的交易，就必須共享同一條 connection
![ConnectionNotAssociate](https://github.com/CHI-KEKE/pics/blob/main/EF/Transaction/ConnectionNotAssociate.png?raw=true)


像是在 A 銀行櫃檯開了一張「本次交易單」，然後跑去 B 銀行櫃檯說「用這張單幫我加一筆」，B 櫃檯回你這不是我這邊的交易單


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction_corss/3_2_diff_conn_fail.png)


## 經過

- 在 `_adventureWorks2022DbContext` 上呼叫 `BeginTransaction()`，這會在 ctx1 的 connection 上開一個 `DbTransaction`
- `ctx1.SaveChangesAsync()` 正常，因為 ctx1 的命令都走同一條 connection。
- 對 `_adventureWorks2022DbContextV2` 呼叫 `UseTransaction(transaction.GetDbTransaction())`，但 ctx2 目前拿著的是另一條 connection（DI 注入的那條/它自己開的）。
- EF Core/ADO.NET 檢查到給的 transaction 並不是 ctx2 這條 connection 建立的，所以直接丟 The specified transaction is not associated with the current connection...


<!-- endtab -->

<!-- tab 實驗四：BeginTransaction + 共用連線-->


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction_corss/3_3_shared_conn.png)


那如果我們手動建立第二個 DbContext 並共用連線，能讓交易撐下去嗎

<br>

```CSHARP
/// <summary>
/// 連線到相同 DB 並共用連線，成功儲存
/// </summary>
public async Task TestBeginTransactionSameConnectionSuccess()
{
    using (var connection = this._adventureWorks2022DbContext.Database.GetDbConnection())
    {
        var transaction = this._adventureWorks2022DbContext.Database.BeginTransaction();
        try
        {
            this._adventureWorks2022DbContext.Add(new Rd
            {
                RdName = "BeginTran1",
                DeptCode = "BT-1"
            });

            await this._adventureWorks2022DbContext.SaveChangesAsync();

            //// 使用相同的連線建立Context
            var v2optionBuilder= new DbContextOptionsBuilder<AdventureWorks2022V2>()
                            .UseSqlServer(connection)
                            .Options;

            using var v2DbContext = new AdventureWorks2022V2(v2optionBuilder);

            v2DbContext.Database.UseTransaction(transaction.GetDbTransaction());
            v2DbContext.Add(new Rd
            {
                RdName = "BeginTran2",
                DeptCode = "BT-2"
            });

            await v2DbContext.SaveChangesAsync();
            await transaction.CommitAsync();
        }
        catch (Exception ex)
        {
            System.Console.WriteLine($"{ex.Message} \n {ex.InnerException?.Message}");
            await transaction.RollbackAsync();
        }
    }
}
```

<br>

✅ 結果：交易成功，共兩筆資料寫入。這證明：只要 DbContext 使用相同連線並共用交易物件，即使是不同實例，也能協同作業。
![sameConnectionSuccessInsert](https://github.com/CHI-KEKE/pics/blob/main/EF/Transaction/sameConnectionSuccessInsert.png?raw=true)


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction_corss/3_4_shared_success.png)



## 經過


- `var connection = ctx1.Database.GetDbConnection()`，拿到 ctx1 目前用的那個 DbConnection 物件（重點是同一個 instance）
- `var transaction = ctx1.Database.BeginTransaction()，`在 這條 connection 上開一個 DbTransaction，交易綁在這條線
- `ctx1.Add(...) + ctx1.SaveChangesAsync()`，ctx1 用同一條 connection + 同一個 transaction 寫入第一筆
- `new DbContextOptionsBuilder().UseSqlServer(connection)`，建立了一個新的 DbContext（v2DbContext），但它用的是 同一個 connection instance
- `v2DbContext.Database.UseTransaction(transaction.GetDbTransaction())`，v2DbContext 也開始用同一個 DbTransaction。
- `v2DbContext.SaveChangesAsync()`，第二筆也進同一個 transaction。



<!-- endtab -->

<!-- tab summary-->


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction_corss/transaction_cross.png)


很多時候，我們並不是不懂交易怎麼運作，只是預設它「會幫我處理好」。

但事實是

- 你用兩個 DbContext，就已經把自己放進一艘不確定的船；
- 你跨資料庫，就已經進入需要特許航道的海域；
- 你用了 BeginTransaction，卻讓它漂流在不同連線上，自然找不到彼此。

每一次踩坑後的 Rollback，都是一次「喔原來不能這樣」的醒悟。這篇文章不是要給出什麼神解法，反而是提醒我們：
EF Core 給了我們很大的彈性，但這份彈性，也需要我們主動去畫出界線、維持一致性。畢竟，程式的問題，多半不是出在「這樣寫可不可以」，而是「我們知不知道它底下怎麼跑」。就像一艘船能不能出港，不只看船造得漂不漂亮，還要看你知不知道洋流怎麼走


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction_corss/4_table.png)


<!-- endtab -->


{% endtabs %}