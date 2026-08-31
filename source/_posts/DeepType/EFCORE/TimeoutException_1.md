---
title: EF Core - TimeoutException (Part 1)
date: 2025-05-28 08:26:05
categories: 資料疆界的航圖
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout/1_landing.png
tags:
    - EF CORE
toc:
toc_number:
comments :
---


{% tabs EF Core - TimeoutException%}

<!-- tab 監控-->

某天早上，監控系統跳出好幾個 TimeoutException，我們小組群組裡只聽到一句：「啊，應該是網路問題吧，再跑一次就好了。」

這句話讓我背脊一涼。

TimeoutException 並不是資料庫在耍脾氣，它是壓力太大在吶喊。這篇文章，要來好好梳理這些 "吶喊" 背後的真正原因

<!-- endtab -->

<!-- tab 查詢資料太多，撐爆記憶體-->


有時候你只是開個後台報表，卻默默查了幾十萬筆資料。久而久之，伺服器就跟泡麵一樣——泡過頭，膨脹爛掉

<br>

## 1️⃣ 真的需要全部資料嗎？


> 商業情境：報表開發初期為求簡便，可能會習慣「全撈」資料來呈現，但實際使用者只會看最近的記錄。
```CSHARP
var threeMonthsAgo = DateTime.UtcNow.AddMonths(-3);
var recentOrders = await dbContext.Orders
    .Where(o => o.CreatedDate >= threeMonthsAgo)
    .ToListAsync();
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout/2_dont_take_all.png)



##　2️⃣ 使用投影（Select）只抓必要欄位

> 商業情境：畫面僅顯示使用者名稱與註冊時間，但卻把個人簡歷、地址等大型欄位也撈出。

```CSHARP
var userSummaries = await dbContext.Users
    .Select(u => new { u.Id, u.Name, u.CreatedDate })
    .ToListAsync();
```



## 3️⃣ 使用精準的查詢條件（WHERE）

> 商業情境：訂單查詢提供了多個條件（如狀態、付款方式），卻只使用模糊查詢造成效能低落。

```CSHARP
var paidOrders = await dbContext.Orders
    .Where(o => o.Status == "Paid" && o.PaymentMethod == "CreditCard")
    .ToListAsync();
```

<br>
<br>

## 4️⃣ 分批處理（分頁查詢 Paging）

> 商業情境：後台管理系統的使用者清單，分頁顯示，每頁 100 筆

```CSHARP
int page = 2;
int pageSize = 100;

var pagedUsers = await dbContext.Users
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();
```

<br>
<br>

## 5️⃣ 使用游標式分頁（Keyset Pagination）取代 Offset

> 商業情境：產品列表頁翻到第 100 頁，使用 Skip() 效能變差。可改以「最後一筆 ID」作為游標。

```CSHARP
var lastSeenId = 5000;

var nextUsers = await dbContext.Users
    .Where(u => u.Id > lastSeenId)
    .OrderBy(u => u.Id)
    .Take(100)
    .ToListAsync();
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout/3_pagination.png)



## 6️⃣ 是否該考慮資料分區（Partition）或歸檔


> 商業情境：交易紀錄超過五年，已不再常用，仍全部放在同一張表中。

建議長期資料定期歸檔、分表或使用資料分區機制


<!-- endtab -->

<!-- tab 命中索引卻還是慢？-->


我們都知道在常被查詢或過濾的欄位上建立索引可以加速查詢。但實務上

> 「WHERE 明明命中了索引，為什麼查詢還是慢？索引不是就是為了加速嗎？」

這背後的關鍵是，查到了位置，不代表資料已經在手上。還得看你查的欄位是不是也包含在索引中。


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout/4_index.png)


## 📚 想像比喻：你在圖書館找一本書

1. 你用目錄卡（索引）找到書的位置（WHERE 命中索引）
2. 你走到那個書架，找出那本書
3. 但這本書很厚，你還是得把整本書翻開來找那一頁（SELECT 的欄位不在索引裡）

👉 如果你只想看書的推薦序，因為沒有索引(不知道一班來說會在哪)還得把整本書查閱一遍，因此查得再快也只是「定位快」，資料還是要翻半天。



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout/4_3_covering_index.png)




## 🧠 真實資料庫中發生了什麼？

```csharp
SELECT Address FROM Users WHERE Email = 'abc@example.com';
```

- Email 有建立索引 ✅
- Address 沒有在索引內 ❌


1. Email 命中索引 → 快速定位該筆資料在主表的 RowId
2. 資料庫根據索引 Row Pointer → 回主資料表撈出 Address
3. 這種動作稱為 Key Lookup / Bookmark Lookup


> 如果你查詢很多筆資料，這個回主表的動作就會重複很多次，造成大量磁碟 I/O，查詢效能反而變差。

在常查詢或過濾的欄位上加索引。


| 查詢行為                 | 描述          | 效能影響        |
| -------------------- | ----------- | ----------- |
| 命中索引但 SELECT 欄位不在索引中 | 快速定位，但仍需回主表 | 中等，有 I/O 負擔 |
| 完全沒命中索引              | 全表掃描        | 慢，資源消耗高     |
| 使用 Covering Index    | 所有資料都在索引中取得 | 最快，無需回主表    |


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout/4_2_keylookup.png)



<!-- endtab -->

<!-- tab 複合索引中，查詢條件順序會影響是否命中索引嗎？-->


答案是會，順序影響很大。

在建立複合索引時，索引的順序決定了哪些查詢可以被有效利用。
```sql
CREATE INDEX IX_Users_Email_CreatedDate ON Users (Email, CreatedDate);
```
✅ 以下查詢能有效使用該索引：

```sql
SELECT * FROM Users WHERE Email = 'abc@example.com';
SELECT * FROM Users WHERE Email = 'abc@example.com' AND CreatedDate > '2024-01-01';
```
❌ 但下面的查詢就無法使用索引：

```sql
SELECT * FROM Users WHERE CreatedDate > '2024-01-01';
```

索引只能從第一個欄位開始連續使用，不能跳欄位


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout/4_4_composite_index.png)



<!-- endtab -->

<!-- tab 複雜 JOIN / 多表格查詢-->

## JOIN 本質是什麼？


```sql
SELECT * 
FROM Orders o
JOIN Customers c ON o.CustomerId = c.Id
```

這句話的意思是：每筆訂單去找對應的顧客，組成新資料表。想像一下 Orders 1000 筆、Customers 100 筆、Regions 10 筆

沒有限制條件或索引，JOIN 出來的中間表可能是 1000 × 100 × 10 = 百萬級資料。


- 沒有索引 → Hash Join（建立 Hash Table，再比對）
- LEFT JOIN → 資料不一定命中，更慢
- 多層 JOIN → 中間表爆量，記憶體炸裂


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout/5_join_explode.png)


## 資料庫怎麼執行 JOIN？

- 從主表（假設是 Orders）掃出一筆
- 根據 CustomerId 去 Customers 表中 找出對應的 Id
- 把兩筆資料組合起來

這個「去找對應資料」的動作，能不能用索引來加速？

## 使用有索引的欄位進行配對

- Customers.Id 是主鍵，自動有索引
- 資料庫可以透過索引「快速定位」符合條件的那筆客戶資料
- JOIN 效能好，通常會使用 Nested Loop Join + Index Seek


## 未使用有索引的欄位來配對

- 如果你 JOIN 的欄位不是索引，資料庫必須對 Customers 整張表逐筆掃描
- 加上 JOIN 是多對多匹配，資料量乘倍增加
- 最終導致全表掃描（Table Scan）或高成本的 Hash Join

> Hash Join 是一種「先建表，再比對」的方式。資料庫會先把其中一張表放進記憶體中，用某欄位建立Hash Table，再用另一張表來比對。



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout/5_2_nested_hash.png)



## 拆查詢，再組起來

將複雜 JOIN 拆成數個小查詢，逐步取得必要資料 → 每次都能命中索引、資料量小。最後在應用層用記憶體或 LINQ 合併。



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout/6_split.png)


```csharp
// 1. 查詢訂單
var orders = await dbContext.Orders.AsNoTracking().ToListAsync();

// 2. 查詢訂單中會出現的顧客
var customerIds = orders.Select(o => o.CustomerId).Distinct().ToList();

var customers = await dbContext.Customers
    .Where(c => customerIds.Contains(c.Id))
    .AsNoTracking()
    .ToListAsync();

// Dictionary 加速比對
var customerDict = customers.ToDictionary(c => c.Id);

var result = orders.Select(o =>
{
    var customer = customerDict.GetValueOrDefault(o.CustomerId);
    return new {
        OrderId = o.Id,
        CustomerName = customer?.Name ?? "(未知客戶)"
    };
}).ToList();

```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout/6_2_dictionary.png)


<!-- endtab -->

<!-- tab Summary-->


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout/7_final.png)


TimeoutException 不該被忽略，就像 GPS 持續轉圈圈，背後可能是資料撈太多、JOIN 過重、索引設計錯誤或條件不精準。
讓我們別再忽視這些訊號，好好回應資料庫的呼救聲。也許它不是不想回你，只是需要一點空氣


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Timeout/timeout1.png)



<!-- endtab -->


{% endtabs %}

