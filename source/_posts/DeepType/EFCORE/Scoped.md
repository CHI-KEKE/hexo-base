---
title: EF Core - DbContext 生命週期
date: 2025-05-20 15:28:05
categories: 資料疆界的航圖
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/1_landing.png
tags:
    - EF CORE
toc:
toc_number:
comments :
---

{% tabs DbContext 生命週期%}

<!-- tab blow out-->

> 「這裡的風很大，請小心 DbContext 被吹出作用域。」

開發 ASP.NET Core 應用程式時，你一定碰過這段設定
```csharp
services.AddDbContext<MyDbContext>();
```
這一行註冊你的 DbContext，但你知道它的 生命週期（Lifetime）是什麼嗎？Scoped、Transient 差在哪？又該怎麼選擇？今天我們就來一趟資料庫生命週期的航行


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/1_landing.png)


<!-- endtab -->

<!-- tab EF Core 的預設生命週期：Scoped-->

[Entity Framework contexts](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-6.0&WT.mc_id=DOP-MVP-37580#entity-framework-contexts)

> By default, Entity Framework contexts are added to the service container using the    scoped lifetime because web app database operations are normally scoped to the client request.

簡單來說，EF Core 的 DbContext 預設會註冊成 Scoped，也就是，每個 HTTP Request 會建立一個 DbContext 實例。這個實例會被整個 Request 共用，直到請求結束後才釋放。這種設計既合理又安全，畢竟一個請求裡的資料庫操作應該是一體的，不應該在不同的 DbContext 之間跳來跳去。


![v](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/2_one_dbcontext_per_request.png)


<!-- endtab -->

<!-- tab Scoped vs Transient：生命週期的差異-->

若將 DbContext 改成 Transient ，表示每次注入都會新建一個 DbContext。即使在同一個 Request 裡的兩個 Repository，也會拿到不同的實例，現在來做個實驗


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/2_change_to_transient.png)



## DI 建立物件

Minimal API 透過 DI 注入 ARepository a、BRepository b，DI 也同時注入 AdventureWorks2022Context 給兩個 Repository

```CSHARP
builder.Services.AddDbContext<AdventureWorks2022Context>(options => options.UseSqlServer(configurationManager.GetConnectionString("AdventureWorks2022")));

builder.Services.AddScoped<ARepository>();
builder.Services.AddScoped<BRepository>();
```

## ARepository

```CSHARP
public class ARepository
{
    private readonly AdventureWorks2022Context _adventureWorks2022DbContext;
    public AdventureWorks2022Context DbContext => _adventureWorks2022DbContext;

    public ARepository(AdventureWorks2022Context adventureWorks2022DbContext)
    {
        this._adventureWorks2022DbContext = adventureWorks2022DbContext;
    }

    public async Task AddRDA()
    {
        var RD1 = new Rd()
        {
            RdName = "Elly",
            DeptCode = "PQ3",
        };

        this._adventureWorks2022DbContext.Rds.Add(RD1);

        ////// 會壞掉的 test
        //var RD2 = new Rd()
        //{
        //    RdName = "grger",
        //    DeptCode = "fffffffffffffffffffffffffffff",
        //};

        var RD2 = new Rd()
        {
            RdName = "grger",
            DeptCode = "ff",
        };

        this._adventureWorks2022DbContext.Rds.Add(RD2);
        await this._adventureWorks2022DbContext.SaveChangesAsync();
    }


    public string AllData => string.Join(",", _adventureWorks2022DbContext.Rds.Select(rd => rd.RdName).ToArray());
}
```

## BRepository

```CSHARP
public class BRepository
{
    private readonly AdventureWorks2022Context _adventureWorks2022DbContext;

    public AdventureWorks2022Context DbContext => _adventureWorks2022DbContext;

    public BRepository(AdventureWorks2022Context adventureWorks2022DbContext)
    {
        this._adventureWorks2022DbContext = adventureWorks2022DbContext;
    }

    public async Task AddRDB()
    {
        var RD1 = new Rd()
        {
            RdName = "Joanne",
            DeptCode = "RD05",
        };

        this._adventureWorks2022DbContext.Rds.Add(RD1);

        ////// 會壞掉的 test
        //var RD2 = new Rd()
        //{
        //    RdName = "grger",
        //    DeptCode = "fffffffffffffffffffffffffffff",
        //};

        var RD2 = new Rd()
        {
            RdName = "WGEWRGR",
            DeptCode = "ff",
        };

        this._adventureWorks2022DbContext.Rds.Add(RD2);
        await this._adventureWorks2022DbContext.SaveChangesAsync();
    }

    public string AllData => string.Join(",", _adventureWorks2022DbContext.Rds.Select(rd => rd.RdName).ToArray());
}
```

## 觀察是否共用同一個 DbContext

```csharp
System.Console.WriteLine($"Dbcontext in ARepository hashcode : {a.DbContext.GetHashCode()}");
System.Console.WriteLine($"Dbcontext in BRepository hashcode : {b.DbContext.GetHashCode()}");
```

確認兩個 Repository 取得的是不是同一個 DbContext 實例


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/3_2_same_context.png)


![v](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/3.2_hashcode.png)


## 在 A 的 DbContext 開交易

```csharp
await a.DbContext.Database.BeginTransactionAsync()
```

這個 Transaction 綁在 a.DbContext 底下那條連線/交易狀態 上


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/3_3_same_transaction.png)


## ARepository 做兩筆 Add + SaveChanges

```csharp
await a.AddRDA();
```

Add() 只是把 entity 丟進 EF 追蹤
SaveChangesAsync() 才會真的送 SQL 到 DB（也就是開始跟 DB 打交道、可能拿鎖）


## BRepository 也做 Add + SaveChanges

如果 B 用的是同一個 DbContext（Scoped），它會在同一個 Transaction 裡
如果 B 是另一個 DbContext（Transient），它其實是「另一條連線 / 另一個交易世界」

```csharp
System.Console.WriteLine($"Dbcontext in BRepository After Transaction : {b.DbContext.Database.CurrentTransaction}");
System.Console.WriteLine($"ARepo Transaction HashCode : {a.DbContext.Database.CurrentTransaction?.GetHashCode()} vs BRepo Transaction HashCode : {b.DbContext.Database.CurrentTransaction?.GetHashCode()}");
```

## Rollback

rollback 的是 Atransaction，所以只有「在 Atransaction 管得到的那條交易脈絡」裡的變更會被回滾


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/3_4_rollback.png)


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/3.4.2_safe_rollback.png)


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/3_5_rollback_fail.png)


## Overview


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/3_exp_overview.png)


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/3.1_flow.png)


想像有同時操作 ARepository 與 BRepository

- A 增加 "Elly"
- B 增加 "Joanne"

中間動作包在一個 Transaction 中，然後 Rollback

- 使用 `Scoped`：兩者 DbContext HashCode 相同，證明分享同一實例
- 使用 `Transient`：兩者是不同的 DbContext，Transaction 無法共用，甚至會出現新增加時 DB Lock 的問題


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/4_dblock.png)


```CSHARP
var app = builder.Build();

//// 透過 依賴注入（Dependency Injection），你取得了兩個 Repository 物件 ARepository 和 BRepository
app.MapGet("/", async (ARepository a, BRepository b) =>
{
    //// 確保資料庫已經建立。
    a.DbContext.Database.EnsureCreated();

    //// 如果 hashcode 一樣 ➜ 他們共用同一個 DbContext ➜ 可以共享 transaction
    /// 如果不一樣 ➜ 是兩個獨立 DbContext ➜ transaction 是隔離的（可能造成 rollback 不一致）
    System.Console.WriteLine($"Dbcontext in ARepository hashcode : {a.DbContext.GetHashCode()}");
    System.Console.WriteLine($"Dbcontext in BRepository hashcode : {b.DbContext.GetHashCode()}");
    System.Console.WriteLine($"Dbcontext in BRepository Before Transaction : {b.DbContext.Database.CurrentTransaction}");

    //// await using 做非同步資源釋放
    await using (var Atransaction = await a.DbContext.Database.BeginTransactionAsync())
    {
        //// 若有活躍的交易會有物件（正常會是 null）
        System.Console.WriteLine($"Dbcontext in BRepository After Transaction : {b.DbContext.Database.CurrentTransaction}");
        System.Console.WriteLine($"ARepo Transaction HashCode : {a.DbContext.Database.CurrentTransaction?.GetHashCode()} vs BRepo Transaction HashCode : {b.DbContext.Database.CurrentTransaction?.GetHashCode()}");

        await a.AddRDA();

        try
        {
            await b.AddRDB();
        }
        catch(Exception ex)
        {
            System.Console.WriteLine($"Error : {ex.Message}");
        }

        //// 將目前記憶體中（或 EF 追蹤中）已經寫入的資料印出來。
        /// SaveChanges()，資料才會進資料庫
        System.Console.WriteLine($"data before Rollback : A {a.AllData} vs B {b.AllData}");
        await Atransaction.RollbackAsync();
    }

    System.Console.WriteLine($"data after rollback : A {a.AllData} B {b.AllData}");
});
```

註冊為 Scoped，ARepository 與 BRepository 取得的 DbContext 是同一個物件，二者的操作也被包成同一個交易

![Scoped](https://github.com/CHI-KEKE/pics/blob/main/EF/LIFETIME/Scoped1.png?raw=true)


若改註冊為 Transient，ARepository、BRepository 的 DbContext 不同，甚至 ARepository 啟用交易後，可能還會引發資料庫鎖定導致 BRepository 無法寫入，兩邊也無法包成交易，不過寫入會不會互卡還關乎 Provider 是誰

![Transient](https://github.com/CHI-KEKE/pics/blob/main/EF/LIFETIME/Transient1.png?raw=true)


<!-- endtab -->

<!-- tab 為何 using dbContext()-->

你可能有看過這種操作

```CSHARP
using var context = new ProductContext();
```

這表示要自己處理 DbContext 的 Dispose，這在 Console Application 較為常見，因為這種環境沒有 DI 工具來自動管理資源。相對來說，在 ASP.NET Core MVC 或 API 裡，通常是用建構子注入 DbContext，對應的 context 由底層管理。如果看到 MVC Controller 裡用 using var context = ...，那應是一個不常見的狀況


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/5_dispose.png)


<!-- endtab -->

<!-- tab EF 會幫我們打理連線嗎？-->

看是誰創造這個連線

如果您對 EF 說
```CSHARP
services.AddDbContext<MyDbContext>(options => options.UseSqlServer("Server=.;Database=MyDb;Trusted_Connection=True;"));
```
這時是 EF 根據連線字串創造 SqlConnection，EF Core 會自動處理它的開啟、釋放、Dispose 等。

但如果是自己用 SqlConnection：
```CSHARP
var conn = new SqlConnection("Server=.;Database=MyDb;Trusted_Connection=True;");
services.AddDbContext<MyDbContext>(options => options.UseSqlServer(conn));
```
這時責任就落在你身上了，EF 不會覺得這是它的錯，就該自己把 conn.Dispose() 打理好



![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/5_2_dispose_responsible.png)



<!-- endtab -->

<!-- tab summary-->


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/6_final.png)


在應用程式的航道上，DbContext 就像船艙裡那位操舵的水手——他掌握著資料的流動節奏，也承載著交易的一致性與效能的平衡。選擇 Scoped、Transient，不是隨意撐起風帆，而是根據航線與氣候的判斷；在該共享的時候共享，在該獨立的時候獨立，才能讓應用不偏不倚，穩穩前行



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Scoped/scoped.png)


<!-- endtab -->


{% endtabs %}
