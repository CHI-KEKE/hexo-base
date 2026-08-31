---
title: EF Core - Transaction
date: 2025-05-19 22:07:05
categories: 資料疆界的航圖
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction/1_half.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction/1_half.png
tags:
    - EF CORE 
toc:
toc_number:
comments :
---


[Using Transactions](https://learn.microsoft.com/zh-tw/ef/core/saving/transactions?WT.mc_id=DOP-MVP-37580)


{% tabs Transaction%}

<!-- tab 關於 Transaction-->


在系統開發中，最可怕的狀況不是資料存不進去，而是「資料只成功了一半」

你可能會覺得沒什麼——多一筆少一筆資料，好像影響不大。但實際上，這種「半成品」的狀態，才是造成系統錯誤與災難的根源



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction/1_half.png)



- 使用者下單成功了，庫存卻沒扣 → 超賣
- 客戶付款完成，但系統沒記帳 → 客訴、查帳風暴
- 發票開出來了，卻沒連到訂單 → 財報對不起來

這些問題不是 bug，也不是 crash，而是資料邏輯不一致，讓系統表面正常、實際混亂。這時候，就需要一種「全部成功或全部失敗」的保護機制 —— 這就是 交易（Transaction） 的存在價值。

交易就像你畫圖時的一顆橡皮擦：如果中間畫錯了，可以整張擦掉重來，而不是留下殘破不堪的塗鴉。


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction/1_eraser.png)


<!-- endtab -->

<!-- tab SaveChanges() 自動包成 Transaction-->


在 EF Core 中，只要你呼叫 SaveChanges() 或 SaveChangesAsync()，它就會自動包一個隱含的交易（Implicit Transaction）。這代表：

```CSHARP
using (var context = new MyDbContext())
{
    context.Users.Add(new User { Name = "小明" });
    context.Orders.Add(new Order { Item = "筆電" });

    context.SaveChanges(); // 任何一個失敗，全部操作都會 rollback
}
```

<br>

- 適合簡單 CRUD
- 乾淨簡單、效能佳
- 若發生例外，自動 Rollback


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction/2_1_savechanges.png)


<!-- endtab -->

<!-- tab 使用 BeginTransaction() 控制交易範圍，可以包多次 SaveChanges()-->

有些情境你會需要呼叫多次 SaveChanges()，但仍然希望它們「要嘛全成，要嘛全不成」。這時就可以使用 BeginTransaction() 取得 IDbContextTransaction：
```CSHARP
using (var context = new MyDbContext())
{
    using (var transaction = context.Database.BeginTransaction())
    {
        try
        {
            var user = new User { Name = "小明" };
            context.Users.Add(user);
            context.SaveChanges(); // 執行後 user.Id 才有值

            var order = new Order { UserId = user.Id, Item = "滑鼠" };
            context.Orders.Add(order);
            context.SaveChanges(); // 第二次 SaveChanges()

            transaction.Commit();
        }
        catch
        {
            transaction.Rollback();
        }
    }
}
```

<br>

- 可多次 SaveChanges()
- 手動控制 Commit / Rollback
- 單一 DbContext

<br>

但你可能會問，幹嘛不一次 SaveChanges() 就好，原因可能有

- 你需要先 SaveChanges() 才能取得資料庫生成的值（如 Identity ID）
- 你需要在中間進行額外邏輯、檢查、呼叫外部 API
- 有些資料之間有順序、依賴、條件判斷


![d](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction/2_2_begintransaction.png)


是否「一定需要」包 transaction？取決於資料關聯的重要性與穩定性

| 情境                     | 是否需要 Transaction           | 原因                                 |
| ---------------------- | -------------------------- | ---------------------------------- |
| 1. 新增會員 → 寄歡迎信         | **可以不用**                   | 資料獨立，寄信失敗不影響主資料                    |
| 2. 新增 User → 新增 Order  | **建議使用**                   | 兩者有強關聯，失敗時要 Rollback               |
| 3. 修改 A → 寫入 AuditLog  | **強烈建議使用**                 | 若 Log 寫入失敗，主資料就不能 commit，否則會造成審計遺漏 |
| 4. 新增 Order → 呼叫外部金流服務 | **需要自訂補償機制或用 Transaction** | 若金流失敗，Order 不想保留                  |



![6](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction/3_decision_scenerio.png)




<!-- endtab -->

<!-- tab 多個 DbContext 共用同一個交易-->


大型系統中，往往有多個 DbContext（例如：User 資料庫、Order 資料庫），你會希望這兩邊能「一起成功或一起失敗」。這可以透過共用同一個資料庫連線與交易物件，不過條件建立在:

- 使用相同連線
- 共用交易

```CSHARP
var connection = new SqlConnection("your-connection-string");
connection.Open();

using (var context1 = new UserDbContext(connection))
using (var context2 = new OrderDbContext(connection))
{
    using (var transaction = context1.Database.BeginTransaction())
    {
        context2.Database.UseTransaction(transaction.GetDbTransaction());

        context1.Users.Add(new User { Name = "小華" });
        context1.SaveChanges();

        context2.Orders.Add(new Order { Item = "鍵盤" });
        context2.SaveChanges();

        transaction.Commit(); // 兩個 Context 的操作一起提交
    }
}
```


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction/4_1_multicontext.png)



但實務上，「很少」這樣做，

1. DbContext 拆分背後通常代表邏輯或資料來源的分離，不管是 CQRS、模組劃分、DDD 聚合分離，DbContext 拆開的目的就是要「邏輯分開」。如果還要「共用連線」，那就有點「拆了又綁」的違和感。
2. 共用連線會帶來額外的維護負擔，你得自己手動開啟連線、關閉連線、處理例外情況，容易出現一方 Dispose 影響另一方的問題（特別是 scope 或 async context 下）
3. 絕大多數需求下，一個 DbContext 就夠了



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction/4_2_seldom_saredconnection.png)



對同一個資料庫的操作，幾乎都可以在單一 DbContext 中完成
若有多模組需求，也可以透過 Extension / Partial Class / Repository 等方式切出邏輯，而不需強拆成多個 DbContext


<!-- endtab -->

<!-- tab 使用 TransactionScope 自動處理多資源交易-->

TransactionScope 是 .NET 的「環境式交易管理器」，它會根據程式中所涉及的資源（SQL、File、Message Queue…）自動升級成 分散式交易（透過 MSDTC）或維持在本地交易（例如單一 SQL Server）


只需要
```CSHARP
using (var scope = new TransactionScope())
{
    // 這裡的所有連線都會自動納入交易
    scope.Complete();
}
```

| 使用情境                                                   | 原因                                       |
| ------------------------------------------------------ | ---------------------------------------- |
| 1. **單體系統中，多個 `DbContext` + Dapper + ADO.NET 一起寫入資料庫** | 可用 `TransactionScope` 把他們包起來，避免交易手動管理太複雜 |
| 2. **你明確知道都連到同一個資料庫**，但用的是不同資料存取技術                     | 避免你要手動開交易物件                              |
| 3. **你願意接受 MSDTC（分散式交易協調器）帶來的效能與部署複雜性**                | 跨資料庫或跨主機才有意義                             |


```CSHARP
using (var scope = new TransactionScope())
{
    using (var context1 = new UserDbContext())
    using (var context2 = new OrderDbContext())
    {
        context1.Users.Add(new User { Name = "小花" });
        context1.SaveChanges();

        context2.Orders.Add(new Order { Item = "耳機" });
        context2.SaveChanges();
    }

    scope.Complete(); // 呼叫這行才會真的提交交易
}
```


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction/4_3_transactionscope.png)



- 自動偵測參與的資源
- 自動決定 Commit / Rollback
- 必須使用 TransactionScopeAsyncFlowOption.Enabled 才能搭配 async


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction/4_4_transactionscope_case.png)



<!-- endtab -->

<!-- tab SaveChanges() 測試-->

```CSHARP
public async Task AddItem()
{
    var RD1 = new Rd()
    {
        RdName = "Amber",
        DeptCode = "PQ2",
    };

    _dbContext.Rds.Add(RD1);

    var RD2 = new Rd()
    {
        RdName = "grger",
        DeptCode = "fffffffffffffffffffffffffffff", // 超出欄位長度
    };

    _dbContext.Rds.Add(RD2);
    await _dbContext.SaveChangesAsync();  // 發生錯誤，兩筆皆 rollback
}
```

<br>

執行這段程式後，會收到錯誤訊息：

> String or binary data would be truncated.
![SQLUpdateException](https://github.com/CHI-KEKE/pics/blob/main/EF/Transaction/sqlUpdateTooLong.png?raw=true)

兩筆資料都沒進資料庫！這代表 EF Core 在呼叫 SaveChanges() 的時候，自動幫你包了一層交易（Transaction）。發生錯誤時，自動 Rollback。好貼心



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction/5_1_rollback.png)



甚至可以用 SQL Profiler 看到，BEGIN TRANSACTION -> 然後一連串 SQL 執行 -> 接著是 ROLLBACK TRANSACTION
![SQL Profiler](https://github.com/CHI-KEKE/pics/blob/main/EF/Transaction/sqlProfiler1.png?raw=true)




![d](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction/5_2_sql_profiler.png)




<!-- endtab -->

<!-- tab BeginTransaction() 測試-->

```CSHARP
public async Task BeginTransactionScopeTest()
{
    using (var transaction = this._dbContext.Database.BeginTransaction())
    {
        await DoJob1();
        await DoJob2();
        await transaction.CommitAsync();
    }
}

private async Task DoJob2()
{
    var rd = new Rd()
    {
        RdName = "Chicken",
        DeptCode = "RD04",
    };

    _dbContext.Add(rd);
    await _dbContext.SaveChangesAsync();
}

private async Task DoJob1()
{
    var rd = new Rd()
    {
        RdName = "Bird",
        DeptCode = "RD055555555555555",  // 又太長，欄位哀嚎
    };

    _dbContext.Add(rd);
    await _dbContext.SaveChangesAsync();
}
```

<br>

由於 DoJob1() 發生錯誤，整個交易還沒 Commit 就失敗，自然也會 Rollback。
結果兩筆資料都沒進資料庫。
SQL Profiler 證實有 BEGIN TRANSACTION -> 沒有 COMMIT -> 最後收場的是 ROLLBACK
![SQLUpdateException](https://github.com/CHI-KEKE/pics/blob/main/EF/Transaction/sqlUpdateTooLong2.png?raw=true)

檢查資料庫，第一筆資料並未寫入，證明有 Transaction 且 Rollback 了。由 SQL Profiler 也成功蒐集到證據，兩次 sp_executesql 前有 BEGIN TRANSACTION，之後有 ROLLBACK TRANSACTION：
![SQL Profiler](https://github.com/CHI-KEKE/pics/blob/main/EF/Transaction/sqlProfiler2.png?raw=true)


<!-- endtab -->

<!-- tab TransactionScope 測試-->

```CSHARP
public async Task TransactionScopeInit()
{
    using (var scope = new TransactionScope(TransactionScopeOption.Required,new TransactionOptions { IsolationLevel = IsolationLevel.ReadCommitted}, TransactionScopeAsyncFlowOption.Enabled))
    {
        await DoJob1();
        await DoJob2();
        scope.Complete();
    }
}
```

<br>

只要你沒呼叫 scope.Complete()，或者中途有一個爆了，就會自動回到初始狀態。資料沒寫入，SQL Profiler 出現 ROLLBACK。
![SQL Profiler](https://github.com/CHI-KEKE/pics/blob/main/EF/Transaction/sqlProfileTransactionScope.png?raw=true)


<!-- endtab -->

<!-- tab summary-->


![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Transaction/transaction.png)


在資料寫入的世界裡，有一種義氣叫做 Transaction。不論你是單純地用 SaveChanges()，還是自己手動開局 (BeginTransaction())，甚至升級為能跨越一切的 TransactionScope，他們的共同信念只有一句「我們是資料兄弟，要衝一起衝，有人滑倒就全體退場！」

<!-- endtab -->


{% endtabs %}
