---
title: EF Core 效能陷阱：IQueryable vs IEnumerable 的致命差異
date: 2025-11-15 08:06:05
categories: LINQ
top_img: https://i.imgur.com/cBfeEDY.png
cover: https://i.imgur.com/cBfeEDY.png
tags:
    - LINQ
toc: true
toc_number: true
comments: true
---

![EF Core Performance](https://i.imgur.com/cBfeEDY.png)

在 Entity Framework Core 開發中，看似簡單的 LINQ 查詢可能隱藏著巨大的效能陷阱。今天我們來深入分析兩個經典的效能問題，以及它們背後的根本原因。

<br>

## 🚨 案例一：SearchProducts 的效能災難

### 💥 問題程式碼

```csharp
public IEnumerable<Product> SearchProducts(string[] keywords)
{
    return _context.Products.Where(p => keywords.All(k => p.Name.Contains(k)));
}

// 呼叫方式
var results = SearchProducts(new[] { "red", "shoes" });
```

乍看之下這段程式碼很合理：搜尋包含所有關鍵字的產品。但實際上，這是一個效能炸彈！

### � 問題分析

**核心問題：`keywords.All(...)` 無法轉換成 SQL**

讓我們拆解這個表達式：
```csharp
keywords.All(k => p.Name.Contains(k))
```

在這個表達式中：
- `keywords` 是 C# 陣列（記憶體中的資料）
- `.All()` 是 LINQ to Objects 的方法
- `p.Name.Contains(k)` 雖然可以轉換成 SQL 的 `LIKE`，但外層的 `.All()` 根本無法轉成 SQL

### ⚠️ EF Core 的處理方式

當 EF Core 遇到無法轉換的表達式時，它會採取「最安全」的做法：

| 階段 | 發生什麼事 |
|------|-----------|
| **資料庫端** | `SELECT * FROM Products` (撈取整張表，可能幾萬筆) |
| **記憶體端** | 在 C# 中逐筆執行 `All()` 檢查 |
| **後果** | 🚨 效能爆炸、記憶體爆炸、回應時間線性成長 |

### 🔧 解決方案

#### ✅ 方法一：使用 Expression Tree 動態建構查詢

```csharp
public IQueryable<Product> SearchProducts(IEnumerable<string> keywords)
{
    IQueryable<Product> query = _context.Products;

    foreach (var keyword in keywords)
    {
        string k = keyword; // 重要：避免閉包問題
        query = query.Where(p => p.Name.Contains(k));
    }

    return query;
}
```

**優點：**
- ✅ 每個 `Contains` 都能轉換成 SQL `LIKE`
- ✅ 資料庫端直接篩選，不會拉整張表
- ✅ 可以利用索引（如果有建立全文索引）
- ✅ 支援多關鍵字查詢

#### ✅ 方法二：使用 EF.Functions.Like

```csharp
public IQueryable<Product> SearchProducts(IEnumerable<string> keywords)
{
    IQueryable<Product> query = _context.Products;

    foreach (var keyword in keywords)
    {
        string pattern = $"%{keyword}%";
        query = query.Where(p => EF.Functions.Like(p.Name, pattern));
    }

    return query;
}
```

### 📊 效能對比

| 方案 | 資料庫查詢 | 記憶體使用 | 效能評級 |
|------|-----------|-----------|----------|
| **原始程式碼** | `SELECT * FROM Products` | 極高（載入全表） | 💀 災難級 |
| **修正後** | `SELECT * FROM Products WHERE Name LIKE '%red%' AND Name LIKE '%shoes%'` | 低 | ⚡ 優秀 |

<br>

## 🚨 案例二：GetAdultUsers 的型別陷阱

### 💥 問題程式碼

```csharp
public IEnumerable<User> GetAdultUsers(Func<User, bool> predicate)
{
    return _context.Users.Where(predicate);
}

// 呼叫方式
var adultUsers = GetAdultUsers(u => u.Age > 18);
```

### � 問題分析

**核心問題：`Func<User, bool>` 不是 Expression**

EF Core 需要能夠解析 Lambda 表達式並轉換成 SQL，但：

- `Func<User, bool>` 是已編譯的委派（delegate）
- EF Core 無法「逆向工程」出 SQL
- 因此只能視為「本地函式」處理

### ⚠️ EF Core 的處理方式

| 階段 | 發生什麼事 |
|------|-----------|
| **資料庫端** | `SELECT * FROM Users` (撈取所有使用者) |
| **記憶體端** | 逐筆執行 `predicate(u)` |
| **後果** | 使用者越多越慢，嚴重拖垮效能 |

### 🔧 解決方案

#### ✅ 正確寫法：使用 Expression<Func<User, bool>>

```csharp
public IQueryable<User> GetAdultUsers(Expression<Func<User, bool>> predicate)
{
    return _context.Users.Where(predicate);
}

// 呼叫方式（看起來一樣，但型別不同！）
var adultUsers = GetAdultUsers(u => u.Age > 18).ToList();
```

### � Func vs Expression 的差異

| 類型 | 特性 | EF Core 處理方式 | 效能影響 |
|------|------|-----------------|----------|
| `Func<T, bool>` | 已編譯的委派 | 無法轉 SQL，在記憶體執行 | 💀 災難級 |
| `Expression<Func<T, bool>>` | 表達式樹 | 可解析成 SQL | ⚡ 優秀 |

<br>

## 🎯 核心概念：IQueryable vs IEnumerable

### 🔍 根本差異

這兩個案例都指向同一個核心問題：**混淆了 IQueryable 和 IEnumerable 的執行模式**

| 介面 | 執行位置 | LINQ 提供者 | 適用場景 |
|------|----------|-------------|----------|
| `IQueryable<T>` | 資料庫端 | LINQ to Entities | 大量資料查詢 |
| `IEnumerable<T>` | 記憶體端 | LINQ to Objects | 小量資料處理 |

### ⚡ 效能影響

```csharp
// ❌ 錯誤：強制轉換成 IEnumerable，後續操作在記憶體執行
var badQuery = _context.Users.AsEnumerable().Where(u => u.Age > 18);

// ✅ 正確：保持 IQueryable，操作在資料庫執行  
var goodQuery = _context.Users.Where(u => u.Age > 18);
```

### 🚨 常見的效能殺手

以下操作會強制執行查詢，將 IQueryable 轉換成 IEnumerable：

```csharp
// 這些方法會立即執行查詢
.ToList()
.ToArray() 
.First()
.FirstOrDefault()
.Count()
.Any()
.All() // ⚠️ 特別注意這個！

// 在查詢鏈中使用時要特別小心
var badExample = _context.Users
    .ToList() // 🚨 這裡就把所有用戶載入記憶體了
    .Where(u => u.Age > 18) // 🚨 這個篩選在記憶體執行
    .OrderBy(u => u.Name);
```

<br>

## 💡 最佳實踐

### ✅ 設計原則

1. **延遲執行到最後**：盡可能保持 `IQueryable<T>` 直到真正需要資料
2. **識別表達式型別**：確保使用 `Expression<Func<T, bool>>` 而非 `Func<T, bool>`
3. **避免混合操作**：不要在查詢鏈中混用資料庫操作和記憶體操作

### � 實用技巧

#### 動態查詢建構

```csharp
public IQueryable<Product> BuildProductQuery(ProductSearchCriteria criteria)
{
    var query = _context.Products.AsQueryable();

    if (!string.IsNullOrEmpty(criteria.Name))
        query = query.Where(p => p.Name.Contains(criteria.Name));

    if (criteria.MinPrice.HasValue)
        query = query.Where(p => p.Price >= criteria.MinPrice.Value);

    if (criteria.CategoryIds?.Any() == true)
        query = query.Where(p => criteria.CategoryIds.Contains(p.CategoryId));

    return query; // 仍然是 IQueryable，尚未執行
}
```

#### 分頁查詢

```csharp
public async Task<PagedResult<Product>> GetProductsAsync(int page, int pageSize)
{
    var query = _context.Products.Where(p => p.IsActive);
    
    var totalCount = await query.CountAsync(); // 第一次執行：取得總數
    
    var items = await query
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync(); // 第二次執行：取得分頁資料

    return new PagedResult<Product>
    {
        Items = items,
        TotalCount = totalCount,
        Page = page,
        PageSize = pageSize
    };
}
```

<br>

## � 總結

### 🔑 關鍵要點

> **凡是使用 C# 的 LINQ-to-Objects 操作（如 `Func`、`All`、`Any`、`ToList()`），都會讓 EF Core 無法轉換成 SQL，導致資料庫整張表被拉回記憶體處理 → 效能災難。**

### 📝 檢查清單

在撰寫 EF Core 查詢時，請確認：

- [ ] 是否正確使用 `Expression<Func<T, bool>>` 而非 `Func<T, bool>`
- [ ] 是否避免在查詢鏈中過早使用 `.ToList()` 等執行方法
- [ ] 是否確保複雜條件能夠轉換成 SQL
- [ ] 是否適當使用 `IQueryable<T>` 保持延遲執行

### 🚀 效能最佳化的黃金法則

**「資料庫擅長篩選，記憶體擅長計算」** —— 讓正確的組件處理適合的工作，才能發揮最佳效能。

<br>

## 🚨 案例三：複雜查詢中的隱藏陷阱

### 💥 問題程式碼

在實際專案中，我們經常會遇到這樣的複雜查詢：

```csharp
using (var transactionScope = new TransactionScope(TransactionScopeOption.Required, transactionOptions))
{
    using (var context = CRMDBEntities.CreateNew(isReadOnly: true))
    {
        var orders = (from order in context.CrmSalesOrder.AsNoTracking().Valids()
                join location in context.Location.AsNoTracking().Valids()
                    on order.CrmSalesOrder_LocationId equals location.Location_Id into locationGroup
                from location in locationGroup.DefaultIfEmpty()
                join slave in context.CrmSalesOrderSlave.AsNoTracking().Valids()
                    on new
                    {
                        a = order.CrmSalesOrder_Id,
                        b = order.CrmSalesOrder_ShopId,
                        c = order.CrmSalesOrder_CrmMemberId
                    }
                    equals new
                    {
                        a = slave.CrmSalesOrderSlave_CrmSalesOrderId,
                        b = slave.CrmSalesOrderSlave_ShopId,
                        c = slave.CrmSalesOrderSlave_CrmMemberId
                    } into slaveGroup
                where order.CrmSalesOrder_ShopId == shopId
                      && order.CrmSalesOrder_CrmMemberId == crmMemberId
                      && order.CrmSalesOrder_TradesOrderFinishDateTime >= minDateTime
                      && order.CrmSalesOrder_TypeDef == "Others"
                      && slaveGroup.Any()
                select new CrmSalesOrderDbResult
                {
                    CrmSalesOrder_Id = order.CrmSalesOrder_Id,
                    CrmSalesOrder_ShopId = order.CrmSalesOrder_ShopId,
                    CrmSalesOrder_OuterOrderCode1 = order.CrmSalesOrder_OuterOrderCode1,
                    // ... 其他欄位
                    Location_Name = (location == null ? string.Empty : location.Location_Name),
                    Location_IsOutlet = (location == null ? false : location.Location_IsOutlet),
                    CrmSalesOrderSlaveCount = slaveGroup.Count(),
                    CrmSalesOrderSlaves = slaveGroup.Select(x => new CrmSalesOrderSlaveDbResult
                    {
                        CrmSalesOrderSlave_Id = x.CrmSalesOrderSlave_Id,
                        CrmSalesOrderSlave_ProductCode = x.CrmSalesOrderSlave_ProductCode,
                        // ... 其他欄位
                    }).ToList(), // 🚨 問題在這裡！
                })
            .OrderByDescending(x => x.CrmSalesOrder_TradesOrderFinishDateTime)
            .Skip(startIndex)
            .Take(maxCount)
            .ToList();
    }
}
```

### 🔍 問題分析

這個查詢看似完整且合理，但隱藏著一個嚴重的效能陷阱：

**核心問題：在 Select 投影中使用 `.ToList()`**

```csharp
CrmSalesOrderSlaves = slaveGroup.Select(x => new CrmSalesOrderSlaveDbResult
{
    // ...
}).ToList(), // 🚨 這裡會導致 N+1 查詢問題
```

### ⚠️ 實際執行情況

| 階段 | 發生什麼事 | SQL 查詢次數 |
|------|-----------|-------------|
| **主查詢** | 執行外層查詢，取得訂單列表 | 1 次 |
| **子查詢執行** | 為每個訂單執行 `slaveGroup.Select().ToList()` | N 次（N = 訂單數量） |
| **總查詢數** | 1 + N 次查詢 | 可能數百次！ |

### 🔧 解決方案

#### ✅ 方法一：移除內部 ToList()，讓 EF Core 自動處理

```csharp
select new CrmSalesOrderDbResult
{
    // ... 其他欄位
    CrmSalesOrderSlaves = slaveGroup.Select(x => new CrmSalesOrderSlaveDbResult
    {
        CrmSalesOrderSlave_Id = x.CrmSalesOrderSlave_Id,
        CrmSalesOrderSlave_ProductCode = x.CrmSalesOrderSlave_ProductCode,
        // ... 其他欄位
    }) // ✅ 移除 .ToList()，讓 EF 在最後統一執行
}
```

#### ✅ 方法二：使用 Include 預載入關聯資料

```csharp
var orders = context.CrmSalesOrder
    .AsNoTracking()
    .Include(o => o.CrmSalesOrderSlaves) // 預載入關聯資料
    .Include(o => o.Location)
    .Where(order => order.CrmSalesOrder_ShopId == shopId
                 && order.CrmSalesOrder_CrmMemberId == crmMemberId
                 && order.CrmSalesOrder_TradesOrderFinishDateTime >= minDateTime
                 && order.CrmSalesOrder_TypeDef == "Others"
                 && order.CrmSalesOrderSlaves.Any())
    .OrderByDescending(x => x.CrmSalesOrder_TradesOrderFinishDateTime)
    .Skip(startIndex)
    .Take(maxCount)
    .ToList();
```

#### ✅ 方法三：分離查詢（適用於複雜場景）

```csharp
// 先取得主要訂單資訊
var orderIds = context.CrmSalesOrder
    .Where(/* 主要條件 */)
    .OrderByDescending(x => x.CrmSalesOrder_TradesOrderFinishDateTime)
    .Skip(startIndex)
    .Take(maxCount)
    .Select(o => o.CrmSalesOrder_Id)
    .ToList();

// 再批量取得關聯資料
var orderDetails = context.CrmSalesOrder
    .Include(o => o.CrmSalesOrderSlaves)
    .Include(o => o.Location)
    .Where(o => orderIds.Contains(o.CrmSalesOrder_Id))
    .ToList();
```

### 📊 效能對比

| 方法 | SQL 查詢次數 | 記憶體使用 | 網路往返 | 效能評級 |
|------|-------------|-----------|----------|----------|
| **原始程式碼** | 1 + N 次 | 中等 | 極高 | 💀 災難級 |
| **移除 ToList()** | 1-2 次 | 低 | 低 | ⚡ 優秀 |
| **使用 Include** | 1 次 | 中等 | 極低 | 🚀 最佳 |
| **分離查詢** | 2 次 | 低 | 低 | ✅ 良好 |

### 🎯 關鍵洞察

> **在 LINQ 查詢的 Select 投影中使用 `.ToList()`、`.ToArray()` 等執行方法，會導致 EF Core 無法優化查詢，進而產生 N+1 查詢問題。**

### 🚨 其他常見的投影陷阱

```csharp
// ❌ 錯誤：在投影中使用執行方法
select new OrderDto
{
    OrderId = o.OrderId,
    Items = o.OrderItems.ToList(), // 🚨 N+1 查詢
    ItemCount = o.OrderItems.Count(), // 🚨 額外查詢
    TotalAmount = o.OrderItems.Sum(i => i.Price) // 🚨 額外查詢
}

// ✅ 正確：讓 EF Core 統一處理
select new OrderDto
{
    OrderId = o.OrderId,
    Items = o.OrderItems, // ✅ 由 EF 優化
    ItemCount = o.OrderItems.Count(), // ✅ 在主查詢中計算
    TotalAmount = o.OrderItems.Sum(i => i.Price) // ✅ 在主查詢中計算
}
```

<br>

---

**參考資料：** [BitBucket PR #48759](https://bitbucket.org/nineyi/nineyi.webstore.mobilewebmall/pull-requests/48759/diff)