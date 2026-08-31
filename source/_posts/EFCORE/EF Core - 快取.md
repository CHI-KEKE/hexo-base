---
title: EF Core - 快取
date: 2025-07-12 10:51:05
categories: 
top_img: https://github.com/CHI-KEKE/pics/blob/main/EF/ocean.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/EF/ocean.png?raw=true
tags:
    - 
toc:
toc_number:
comments :
---

![ocean](https://github.com/CHI-KEKE/pics/blob/main/EF/ocean.png?raw=true)

<br>
<br>

## 🌊 DbContext 是什麼？ — 它就是你的「臨時記事本」

<br>

在 Entity Framework Core（EF Core）裡，DbContext 是一個很關鍵的角色，背後有兩個重要概念：

- 它是工作單位（Unit of Work）
- 它自帶一級快取（First-Level Cache）

當你在一個業務流程裡使用 DbContext 時，它幫你做三件事：

✅ 變更追蹤（Change Tracking）

追蹤每個被載入的 Entity 誰改了什麼，幫你記錄要更新的欄位。

✅ 一級快取（First-Level Cache）

相同主鍵的資料，只要查過一次就放在 DbContext 的快取中，後續查詢就直接從快取拿，不會再打資料庫。

✅ 單一交易（Single Transaction）

呼叫 SaveChanges() 時，會把所有變更一次送到資料庫並用同一個交易包起來，確保一致性。換句話說，DbContext 就像一個臨時記事本（背包 🧳），把這段流程需要用到的資料和變更都記錄起來，最後一次打包送到資料庫。

<br>
<br>

## 🌊 EF Core 的一級快取是什麼？

所謂「快取」，指的是 DbContext 內部對載入過的資料做記憶體暫時存取：

只要是同一個 DbContext，查詢相同主鍵的資料時，就會直接從快取拿，而不是重跑 SQL 去打資料庫。

這個快取是 DbContext 專屬的，每個 DbContext 有自己的快取空間，互不干涉。


```CSHARP
var user1 = _dbContext.Users.Find(1); // 第一次從資料庫查
var user2 = _dbContext.Users.Find(1); // 從快取拿，不會再查一次
```

但是要注意：

只要還在同一個 DbContext，快取就存在，不會自己消失。如果外部資料庫有更新，除非你重新查（或用 AsNoTracking()），不然快取裡的值還是舊的。

<br>
<br>


## 🌊 實驗

### 🎯 實驗目的

這個實驗要證明 EF Core 的一級快取特性：

1. **不同 DbContext 之間的快取是獨立的**
2. **每個 DbContext 都有自己的變更追蹤和快取機制**
3. **跨 DbContext 的資料一致性問題**

<br>
<br>

### 🔧 實驗配置

在開始實驗前，確保您的 `Startup.cs` 或 `Program.cs` 中正確註冊了 DbContext：

```CSHARP
builder.Services.AddDbContext<JournalDbContext>(options => options.UseSqlite("Data Source=journal.db"));
```

以及確保您有對應的 Entity 類別：

```CSHARP
public class DailyRecord
{
    public int Id { get; set; }
    public DateTime Date { get; set; }
    public string User { get; set; }
    public string EventSummary { get; set; }
}

public class JournalDbContext : DbContext
{
    public DbSet<DailyRecord> Records { get; set; }
    
    public JournalDbContext(DbContextOptions<JournalDbContext> options) 
        : base(options)
    {
    }
}
```

<br>
<br>

### 實驗程式碼

```CSHARP
public class TestScopedModel : PageModel
{
    private readonly IServiceProvider _services;
    private readonly JournalDbContext _dbCtx1;

    public TestScopedModel(IServiceProvider services, JournalDbContext dbCtx1)
    {
        _services = services;
        _dbCtx1 = dbCtx1;
    }

    public void OnGet()
    {
        Console.WriteLine($"Onget~~~~~~~~~~~~~~~");

        // 🧹 清理環境：移除所有現有資料
        var allRecords = _dbCtx1.Records.ToList();
        if (allRecords.Any())
        {
            _dbCtx1.Records.RemoveRange(allRecords);
            _dbCtx1.SaveChanges();
            Console.WriteLine($"已移除 {allRecords.Count} 筆資料");
        }

        // 🌱 初始化：新增一筆種子資料
        if (!_dbCtx1.Records.Any())
        {
            _dbCtx1.Records.Add(new DailyRecord
            {
                Date = DateTime.Today,
                User = "Original",
                EventSummary = "Seed record"
            });
            _dbCtx1.SaveChanges();
        }

        // 📦 取得第一筆記錄的參考（此時已載入 dbCtx1 的快取）
        var rec = _dbCtx1.Records.First();
        
        // 🆕 建立新的 DbContext Scope（dbCtx2）
        using (var scope = _services.CreateScope())
        {
            var dbCtx2 = scope.ServiceProvider.GetRequiredService<JournalDbContext>();

            // 📊 第一次查詢：兩個 DbContext 的初始狀態
            Console.WriteLine($"Records.Count in dbCtx1 = {_dbCtx1.Records.Count()}");
            Console.WriteLine($"1st Record.User in dbCtx1 = {_dbCtx1.Records.First().User}");
            Console.WriteLine($"Records.Count in dbCtx2 = {dbCtx2.Records.Count()}");
            Console.WriteLine($"1st Record.User in dbCtx2 = {dbCtx2.Records.First().User}");

            // ➕ dbCtx2 新增一筆資料
            dbCtx2.Records.Add(new DailyRecord
            {
                Date = DateTime.Today,
                User = "Old User",
                EventSummary = "DBcTX2 Add Me -..-"
            });
            dbCtx2.SaveChanges();

            // ✏️ dbCtx1 修改現有資料
            rec.User = "New User";
            _dbCtx1.SaveChanges();

            // 📊 第二次查詢：觀察快取行為
            Console.WriteLine($"Records.Count in dbCtx1 = {_dbCtx1.Records.Count()}");
            Console.WriteLine($"1st Record.User in dbCtx1 = {_dbCtx1.Records.First().User}");
            Console.WriteLine($"Records.Count in dbCtx2 = {dbCtx2.Records.Count()}");
            Console.WriteLine($"1st Record.User in dbCtx2 = {dbCtx2.Records.First().User}");
        }
    }
}
```

<br>
<br>

### 實驗結果

```PLAINTEXT
Records.Count in dbCtx1 = 1
1st Record.User in dbCtx1 = Original
Records.Count in dbCtx2 = 1
1st Record.User in dbCtx2 = Original
Records.Count in dbCtx1 = 2
1st Record.User in dbCtx1 = New User
Records.Count in dbCtx2 = 2
1st Record.User in dbCtx2 = Original
```

<br>
<br>

### 結果分析

#### 第一次查詢結果分析

```PLAINTEXT
Records.Count in dbCtx1 = 1  ✅ dbCtx1 只看到自己建立的資料
1st Record.User in dbCtx1 = Original  ✅ 顯示原始值
Records.Count in dbCtx2 = 1  ✅ dbCtx2 從資料庫讀取到相同資料
1st Record.User in dbCtx2 = Original  ✅ 顯示原始值
```

**重點觀察：**
- 兩個 DbContext 都讀取到相同的資料庫狀態
- 此時資料庫中只有一筆 "Original" 記錄

<br>

#### 📊 第二次查詢結果分析

```
Records.Count in dbCtx1 = 2  // ⚠️ dbCtx1 看到了 dbCtx2 新增的資料
1st Record.User in dbCtx1 = New User  // ✅ 顯示自己修改後的值
Records.Count in dbCtx2 = 2  // ✅ dbCtx2 看到了自己新增的資料
1st Record.User in dbCtx2 = Original  // ❌ 仍顯示快取中的舊值！
```

**關鍵發現：**

🔴 **Count 會重新查詢資料庫**
- `_dbCtx1.Records.Count()` 和 `dbCtx2.Records.Count()` 都顯示 2
- 因為 `Count()` 方法會執行 SQL 查詢，所以能看到最新的資料庫狀態

🔴 **First() 使用快取資料**
- `_dbCtx1.Records.First().User` 顯示 "New User"（因為 `rec` 物件已被追蹤且修改）
- `dbCtx2.Records.First().User` 仍顯示 "Original"（使用快取中的舊資料）

<br>

### 快取機制詳解

#### 變更追蹤機制

```CSHARP
var rec = _dbCtx1.Records.First(); // ← 此物件被 dbCtx1 追蹤
rec.User = "New User";              // ← 修改被追蹤
_dbCtx1.SaveChanges();              // ← 變更寫入資料庫
```

- `rec` 物件在 `dbCtx1` 的變更追蹤器中
- 修改 `rec.User` 時，EF Core 自動記錄這個變更
- `SaveChanges()` 將變更同步到資料庫

<br>

#### 一級快取行為

**dbCtx1 的快取：**
- 載入 ID=1 的記錄到快取
- 追蹤到 User 欄位被修改為 "New User"
- `First()` 返回被修改的物件

**dbCtx2 的快取：**
- 獨立載入 ID=1 的記錄到自己的快取
- 完全不知道 dbCtx1 的修改
- `First()` 返回原始的 "Original" 值

<br>

### 實務影響

#### 資料不一致風險

```CSHARP
// 危險情境
var userFromCtx1 = dbCtx1.Users.First(u => u.Id == 1);
var userFromCtx2 = dbCtx2.Users.First(u => u.Id == 1);

// 兩個物件可能有不同的值！
if (userFromCtx1.Status != userFromCtx2.Status)
{
    // 這種情況下該相信誰？
}
```

<br>

#### 解決方案

**1. 使用 AsNoTracking() 避免快取**
```CSHARP
var latestUser = dbCtx2.Users
    .AsNoTracking()
    .First(u => u.Id == 1); // 強制從資料庫重新讀取
```

**2. 重新載入實體**
```CSHARP
dbCtx2.Entry(user).Reload(); // 重新從資料庫載入
```

**3. 使用短生命週期 DbContext**
```CSHARP
// 每次操作都用新的 DbContext
using var context = new JournalDbContext(options);
var freshData = context.Records.First();
```

<br>
<br>

## 🌊 實驗總結

### 🎯 核心發現

透過這個實驗，我們清楚看到了 EF Core 一級快取的三個重要特性：

1. **獨立性**：每個 DbContext 都有自己的快取空間，互不干涉
2. **持久性**：快取在 DbContext 生命週期內持續存在
3. **不同步性**：跨 DbContext 的資料可能出現不一致

<br>
<br>

### 對比

| 查詢方法 | 行為 | dbCtx1 結果 | dbCtx2 結果 |
|---------|------|------------|------------|
| `Count()` | 查詢資料庫 | 2 (最新) | 2 (最新) |
| `First().User` | 使用快取 | "New User" (已修改) | "Original" (舊快取) |

<br>
<br>

### 💡 實驗驗證了什麼？

這個實驗展示了為什麼我們需要：

- **短生命週期的 DbContext**（Scoped 註冊）
- **謹慎處理跨 DbContext 的資料操作**
- **適當使用 AsNoTracking() 避免過期快取**

<br>
<br>

##　參考文獻

[EF Core DbContext 快取特性實驗](https://blog.darkthread.net/blog/efcore-scoped-dbcontext-cache/)