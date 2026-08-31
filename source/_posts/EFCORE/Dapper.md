---
title: Dapper vs EF Core
date: 2025-05-23 22:07:05
categories: 
top_img: /images/ocean.png
cover : /images/ocean.png
tags:
    - 
toc:
toc_number:
comments :
---

![ocean](/images/ocean.png)

<br>
<br>

> 如果資料存取是場人生選擇，那 EF Core 就像「全包式旅行團」：行李、住宿、行程幫你安排好，你只要準時上車；而 Dapper 則像「背包客自由行」：想去哪、怎麼走、要不要迷路，全靠你自己。

在 .NET 的世界裡，EF Core 與 Dapper 是資料存取的兩大門派，一個幫你包辦大小事，一個讓你操控每一行 SQL。

你可能常常陷入這種掙扎：

「欸現在這隻微服務要查一堆訂單、明細、客戶資料，我要不要用 EF？還是會變龜速大怪獸？」
「有三層關聯欸，用 Dapper 要手動 map 到哭出來，欄位一多手都斷了吧…」

<br>

## 🌊 ORM

<br>

在開發應用程式時，資料通常儲存在資料庫（Database）中，而程式則是用物件（Object）在操作資料。
但是資料庫與程式是兩種世界

> 資料庫用的是「表格」（table）來存資料，C# 用的是「物件」和「類別」

這兩個東西的結構不同、語言不同、邏輯也不同。

<br>

### 👉 ORM（Object-Relational Mapping）本質是什麼？

<br>

它是一座橋梁，幫你自動把「資料表」轉成「物件」，也幫你把你操作的物件轉回資料表格式，這個過程就叫做 Mapping（對應）。

<br>

### 🧱 ORM 的組成元素（以 EF Core 為例）

1. 資料映射（Mapping）
把資料表的欄位一一對應到 C# 的類別屬性。例如：
```CSHARP
public class Product {
    public int Id { get; set; }
    public string Name { get; set; }
}
```

| Id | Name   |
| -- | ------ |
| 1  | Apple  |
| 2  | Banana |

ORM 幫你把資料表的每一列變成一個 Product 物件。

<br>
<br>

2. 關聯導覽（Navigation Properties）
當資料表有外鍵（Foreign Key）關係時，ORM 會幫你把這些關聯也變成物件屬性，讓你可以像操作物件一樣處理資料之間的關係。
```CSHARP
public class Order {
    public int Id { get; set; }
    public Customer Customer { get; set; }  // Navigation Property
}
```
你就可以這樣查資料：
```CSHARP
var order = db.Orders.Include(o => o.Customer).First();
Console.WriteLine(order.Customer.Name);
```

<br>
<br>

3. 資料狀態追蹤（Change Tracking）
EF Core 會記得你從資料庫撈出來的物件有沒有被更動（例如某個屬性被改過），這樣你在呼叫 SaveChanges() 時，它就只會更新你改過的資料，效率高又安全。他像是有記憶的秘書，你說「幫我存檔」，它就知道「只有這個名字被改了，我只更新這一欄就好」。

<br>
<br>

4. 自動儲存（SaveChanges）
你對物件做修改後，呼叫 dbContext.SaveChanges()，EF Core 就會把所有變更轉換成 SQL 語句，自動更新到資料庫裡。
```CSHARP
var product = db.Products.First();
product.Name = "New Name";
db.SaveChanges();  // EF Core 自動產生 SQL 更新語句
```

<br>
<br>

5. Migration、自動建表
當你改了 C# 的類別（例如新增屬性），你可以用 Migration 功能讓資料表也同步更新，不用手動寫 SQL 建表。
```PLAINTEXT
dotnet ef migrations add AddPriceToProduct
dotnet ef database update
```

<br>
<br>

## 🌊 那 Dapper 呢？Dapper 是 ORM 嗎？

<br>

✅ 答案：Dapper 是 micro ORM（微型 ORM）

它只做了一件事：資料映射（Mapping），它幫你把 SQL 查詢結果自動轉成 C# 的物件，但不會幫你記住物件狀態、也不會幫你建立資料表或處理關聯

如果 EF Core 是全自動的智慧機器人（會記、會改、會存），那 Dapper 就是很快的打字員 —— 你給他 SQL，他幫你抄成物件，但你要自己記得東西改了沒、要不要存。

| 特性               | EF Core（全功能 ORM） | Dapper（微型 ORM） |
| ---------------- | ---------------- | -------------- |
| 自動 Mapping       | ✅                | ✅              |
| 關聯導覽             | ✅                | ❌              |
| Change Tracking  | ✅                | ❌              |
| 自動儲存 SaveChanges | ✅                | ❌              |
| 自動建表 / Migration | ✅                | ❌              |
| SQL 自由度          | 中（抽象）            | 高（你寫 SQL）      |
| 效能               | 中（有 overhead）    | 高（幾乎原生）        |


> Dapper 的宗旨是：SQL in, Object out.

你寫 SQL 查資料，它幫你把結果快速轉成 C# 物件。它確實有「資料表 → 物件」這一段，也支援 Mapping，但它不做關聯導覽、不追蹤變化、也不會幫你更新資料庫。

所以我們可以說：

✅ Dapper 是 ORM 的一小塊功能（Mapping）。
❌ Dapper 不是完整的 ORM 框架。

<br>
<br>

## 🌊 複雜模型 + 關聯查詢

<br>

當兩張資料表之間有「一對多」的關聯時（例如一張訂單有多個明細），你常會想要一次查出主資料（Order）和子資料（OrderItems）。這叫做關聯查詢，也是 ORM 的一個大重點。

<br>

### EF

你只要這樣寫，背後其實 EF Core 幫你做了這幾件事
```csharp
var order = context.Orders
    .Include(o => o.OrderItems)
    .FirstOrDefault(o => o.Id == id);
```

**✅ 第一步：組出 SQL JOIN 查詢**
EF Core 看你用了 .Include(o => o.OrderItems)，知道你要把主表跟子表一起查，所以它會幫你產生類似這樣的 SQL，你完全不用寫 SQL，它幫你產生好。
```SQL
SELECT o.Id, o.CustomerName, ...
       oi.Id, oi.OrderId, oi.ProductName, ...
FROM Orders o
LEFT JOIN OrderItems oi ON o.Id = oi.OrderId
WHERE o.Id = @id
```

**✅ 第二步：自動建立物件、填資料**
查出來的資料是扁平的（flat row），EF Core 要把它「還原」成 C# 的物件結構：
```CSHARP
class Order {
    public int Id { get; set; }
    public string CustomerName { get; set; }
    public List<OrderItem> OrderItems { get; set; }
}

class OrderItem {
    public int Id { get; set; }
    public string ProductName { get; set; }
}
```
EF Core 自動幫你：

- 建立 Order 物件
- 建立多個 OrderItem 物件
- 把這些 OrderItem 放進 Order.OrderItems 清單中

他會幫你配對好，順好關係，你完全不需要處理這些繁雜邏輯。

**✅ 第三步：追蹤物件狀態（Change Tracking）**

這時候 EF Core 還會開始追蹤這些物件：

- 你有沒有更改 CustomerName
- 你有沒有刪除某個 OrderItem
- 你有沒有新增一筆 OrderItem
- 到時候你只要呼叫 SaveChanges()，EF Core 會自己知道該怎麼下 SQL 去更新資料庫。

<br>
<br>

### Dapper

Dapper 沒有 .Include()、沒有自動建立關聯關係，全部得你自己來：
```SQL
SELECT o.Id, o.CustomerName, oi.Id, oi.OrderId, oi.ProductName
FROM Orders o
LEFT JOIN OrderItems oi ON o.Id = oi.OrderId
WHERE o.Id = @id;
```
然後用 Dapper 的 Query 搭配 SplitOn 把資料手動組合起來：
```CSHARP
var orderDict = new Dictionary<int, Order>();
var order = connection.Query<Order, OrderItem, Order>(
    sql,
    (o, oi) =>
    {
        if (!orderDict.TryGetValue(o.Id, out var currentOrder))
        {
            currentOrder = o;
            currentOrder.OrderItems = new List<OrderItem>();
            orderDict.Add(currentOrder.Id, currentOrder);
        }

        if (oi != null)
            currentOrder.OrderItems.Add(oi);

        return currentOrder;
    },
    new { id = id },
    splitOn: "Id"  // 要告訴 Dapper 從哪欄開始分出第二個物件
).FirstOrDefault();
```
這段程式碼你要自己維護、測試、處理重複資料、確保資料對應正確。

| 步驟            | EF Core | Dapper      |
| ------------- | ------- | ----------- |
| 自動產生 JOIN     | ✅       | ❌（要自己寫）     |
| 自動建立主物件 + 子物件 | ✅       | ❌（要用手動對應邏輯） |
| 自動追蹤狀態、存檔     | ✅       | ❌（你要寫更新邏輯）  |
| 實作難度          | 低       | 中到高         |
| 效能            | 中       | 高           |


<br>
<br>

## 🌊 資料變更 + 自動儲存

當你從資料庫撈出一筆資料，想要更新其中某些欄位並儲存回去時，有兩種處理方式：

- 狀態導向（State-based）：追蹤你改了哪些屬性，只更新那些部分（EF Core）
- 命令導向（Command-based）：你明確告訴系統「要改哪些欄位」（Dapper）

<br>

### EF Core 的本質：擁有「記憶力」的物件

EF Core 背後有個核心機制叫做 Change Tracker（變更追蹤器），它會在你查詢資料後，記住「原始的屬性值」，當你改了某個屬性，它就知道這裡被改了。

```CSHARP
var order = context.Orders.First(o => o.Id == 1);  // 撈資料並追蹤
order.Status = "Shipped";                          // 修改其中一個欄位
context.SaveChanges();                             // 自動找出變更，產生 UPDATE 語句
```
EF Core 幫你完成的細節：

✅ 知道是哪個物件被修改
✅ 知道是哪個屬性改了值
✅ 產生只更新必要欄位的 SQL
✅ 執行 SQL
✅ 更新後自動清除變更狀態

👉 你只專注在「業務邏輯」，不用關心 SQL 細節。

<br>
<br>

### Dapper 的本質：你是主控者，但也要包辦所有細節

Dapper 不會追蹤你改了什麼，因為它只是執行 SQL 的工具。你要自己明確告訴它：「我現在要更新哪些欄位」。

```csharp
var order = new Order { Id = 1, Status = "Shipped" };
connection.Execute(
    "UPDATE Orders SET Status = @Status WHERE Id = @Id",
    new { order.Status, order.Id }
);
```
你要自己處理的事情：

❌ 要自己寫 UPDATE SQL
❌ 要手動列出所有欄位
❌ 欄位多時容易錯、難維護
❌ 你改了哪個屬性，Dapper 不知道，它不幫你記

| 功能             | EF Core | Dapper     |
| -------------- | ------- | ---------- |
| 自動偵測資料變更       | ✅ 有     | ❌ 無        |
| 自動產生 UPDATE 語法 | ✅       | ❌（手寫）      |
| 僅更新變更欄位        | ✅       | ❌（除非你特別處理） |
| 欄位多時是否好維護      | ✅ 超好    | ❌ 非常痛苦     |
| 效能表現           | 中       | 高（但要犧牲方便性） |

<br>
<br>

## 🌊 快速開發 CRUD

大多數應用程式 80% 的資料操作都是這四件事。如果能快速搞定 CRUD，就可以省下大量時間、專注在業務邏輯上。

### EF Core：高效率、免寫 SQL 的 CRUD 幫手

你只要有：

- Model 類別（C# class）
- 設定好 DbContext

你就可以直接操作物件來做 CRUD，完全不寫 SQL：
```csharp
context.Add(obj);             // 新增
context.Find<T>(id);          // 查詢（依主鍵）
context.Update(obj);          // 更新（EF Core 會追蹤狀態）
context.Remove(obj);          // 刪除
context.SaveChanges();        // 一鍵執行所有變更
```
✅ 優點
無須 SQL 基礎也能操作資料

更接近物件導向思維，開發速度快

維護成本低（改欄位只改 class）

<br>
<br>

### Dapper：靈活但需手動操作的 SQL 工具

Dapper 本質就是一個「高速 SQL 執行工具」，它不幫你生成 SQL、也不追蹤資料變更，你必須每個 CRUD 都自己寫 SQL。

```csharp
connection.Execute("INSERT INTO Products (Name, Price) VALUES (@Name, @Price)", new { obj.Name, obj.Price });

connection.QueryFirstOrDefault<Product>("SELECT * FROM Products WHERE Id = @Id", new { id });

connection.Execute("UPDATE Products SET Name = @Name, Price = @Price WHERE Id = @Id", new { obj.Name, obj.Price, obj.Id });

connection.Execute("DELETE FROM Products WHERE Id = @Id", new { obj.Id });
```

<br>
<br>

## 🌊 支援 Migration（Schema 跟著走）

在現實開發中，資料表的結構可能會會變更，例如：

- 新增欄位（如：加上 PhoneNumber）
- 修改欄位型別
- 加上索引或外鍵
- 新增/刪除資料表
- 這些結構變更如果沒管理好，很容易讓「程式對不上資料庫」，導致錯誤、難以維護。

### EF Core：有「自動進化」的資料庫管理機制 —— Migration

當你修改了 C# class，例如：
```csharp
public class User {
    public int Id { get; set; }
    public string Name { get; set; }
    public string PhoneNumber { get; set; }  // 新增欄位
}
```
只要下命令
```BASH
dotnet ef migrations add AddColumn
dotnet ef database update
```
EF Core 就會：

- 檢查現在的資料表跟 Model 差在哪裡
- 自動產生 SQL Script（如 ALTER TABLE ... ADD COLUMN）
- 自動執行 SQL 更新資料庫結構

✅ 優點

- 免手動寫 SQL
- 版本可追蹤（每次 Migration 都有紀錄）
- 可以多人協作同步資料表變更
- 程式與資料庫結構保持一致

<br>
<br>

### Dapper：你要自己當建築師 + 工人

Dapper 不提供任何 Schema 管理功能，所以：

- 你要自己手動寫 SQL ALTER TABLE
- 你要確保每個環境都記得執行更新
- 你要自己管理版本（Excel、Notion？）
- 改了 Model，SQL 沒改 → 就會爆炸

<br>
<br>

## 🌊 那什麼時候換 Dapper 上場？

EF Core 很萬能，但有點胖；Dapper 是個乾淨俐落的 SQL 工具人。以下情境交給他，最實在：

| 不適合 EF Core 的場景  | 為什麼改用 Dapper？          |
| ---------------- | ---------------------- |
| 查詢為主的 API 系統     | Dapper 寫 SQL 查資料最省事    |
| 資料變更少、查詢量大       | EF 的 tracking 開銷太大     |
| 報表系統 / Dashboard | Dapper 查表快速、少 overhead |
| 微服務效能要求高         | Dapper 精準操控 SQL 效能最強   |
| 工程師對資料庫很熟、愛寫 SQL | Dapper 不限制你，怎麼寫怎麼來     |

<br>
<br>

## 🌊 結語：選哪一個，取決於你要什麼

你可以這樣理解：

想快速起專案、專注在商業邏輯 → EF Core
想自己寫 SQL、控制效能到最細節 → Dapper

想兩者都用？也行啊，讀用 Dapper、寫用 EF 的混合派也很多！
最終選擇不只是「用哪個工具」，而是「你的系統架構、團隊熟悉度、可維護性」能不能承接得住。