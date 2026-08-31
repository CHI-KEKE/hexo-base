---
title: Query Expression Syntax vs Method Chaining Syntax
date: 2024-06-06 23:59:34
categories: LINQ
top_img: https://i.imgur.com/f2eWUWv.png
cover : https://i.imgur.com/cBfeEDY.png
tags:
    - LINQ
toc:
toc_number:
comments :
---

當我們在 C# 中使用 LINQ 查詢語法（Query Syntax）時，這種查詢語法看起來簡潔且易於理解，但實際上，.NET 的 Common Language Runtime（CLR）並不直接支援這種語法。CLR 主要還是處理方法呼叫 (method calls) 和其他基本的程式碼結構。


簡單的例子
```csharp

int[] numbers = [ 5, 10, 8, 3, 6, 12 ];

//Query syntax:（查詢語法）
IEnumerable<int> numQuery1 =
    from num in numbers
    where num % 2 == 0
    orderby num
    select num;

//Method syntax:（方法語法）
IEnumerable<int> numQuery2 = numbers.Where(num => num % 2 == 0).OrderBy(n => n);

```

看起來是兩種完全不同的寫法，但 本質上查詢語法只是語法糖 (syntactic sugar)。C# 編譯器會把查詢語法轉換成方法語法呼叫 → 最終都變成 Enumerable.Where、Enumerable.OrderBy 的呼叫。CLR 並不懂查詢語法，只懂方法呼叫。所以 Query Syntax 只是寫法比較貼近 SQL，適合閱讀；Method Syntax 才是底層真正執行的方式。

<br>
<br>

## 🐛 LINQPad Lambda translations

像 LINQPad 會幫我們做「即時翻譯」查詢語法，顯示對應的 Lambda / Method Syntax：

![Image](https://i.imgur.com/RFfvHEt.png)

<br>
<br>

## 🐛 支援範圍的不同

查詢語法其實只能對應「常見的 SQL 風格子句」：from、where、select、join、orderby、group。但有些 LINQ 運算子並沒有對應的查詢語法：

✅ 查詢語法可用：where、select、orderby、join、group by
❌ 查詢語法沒有：Distinct、Skip、Take、Zip、Aggregate… 這些就只能用方法語法

```csharp
List<StudentGrade> grades = GetStudentGrades();

var topStudents = grades.OrderByDescending(g => g.Grade)
                         .Select(g => g.StudentName)
                         .Distinct()
                         .Take(5);
```

Query Syntax 是為了貼近 SQL，適合表達「篩選 + 投影 + 分組」，而 Method Syntax 更完整，能用到所有 LINQ 擴充方法。

<br>
<br>

## 🐛 查詢分解成多個步驟、可重用

方法語法比查詢語法更容易拆解成多個步驟，方便組合與重用
```csharp
var items = orders.SelectMany(o => o.Items);

var grouped = items.GroupBy(i => i.ItemName)
                   .Select(g => new { ItemName = g.Key, TotalPrice = g.Sum(i => i.Price) });

var topItems = grouped.OrderByDescending(i => i.TotalPrice)
                      .Take(3);
```

- 查詢分段更清晰
- 中間結果（grouped）可以重複使用或 debug
- 更符合「方法鏈式」的思維

<br>
<br>

## 🐛 查詢的執行時機

LINQ 有「延遲執行」的特性：查詢只是組裝好，真正取資料要等到「實體化 (materialize)」時才會執行（例如 ToList()、First()、Count()）。

```csharp
IQueryable<Customer> customers = GetCustomers();

// 還沒去 DB
var query = from c in customers
            where c.Country == "USA"
            select new { c.Name, c.Email };

// 這裡才真正發 SQL 到資料庫
var result = query.Take(10).ToList();
```

Query Syntax 和 Method Syntax 在執行時機上沒有本質差別，但 Method Syntax 因為能明確插入 .ToList()、.ToArray()，更能控制「查詢在何時執行」。

<br>
<br>

## 🐛 Join

### 單表 Inner Join

```csharp

//// Query Syntax
var result = from c in customers
             join o in orders on c.CustomerID equals o.CustomerID
             select new { CustomerName = c.Name, OrderID = o.OrderID, OrderDate = o.Date };

//// Method Syntax
var result = customers.Join(
    orders,
    c => c.CustomerID,
    o => o.CustomerID,
    (c, o) => new { CustomerName = c.Name, OrderID = o.OrderID, OrderDate = o.Date }
);

```

join 關鍵字在 Query Syntax 就像 SQL 的 JOIN 子句，結構清晰；而 Method Syntax 必須顯式指定：

outerKeySelector (c => c.CustomerID)
innerKeySelector (o => o.CustomerID)
resultSelector ((c, o) => ...)

在 CRM 系統中，客戶與訂單的關聯是一對多，用 JOIN 很常見。Query Syntax 一看就懂，Method Syntax 顯得繁瑣。


###　多表 Join

```csharp

//// Query Syntax
var result = from c in Customers
             join o in Orders on c.CustomerID equals o.CustomerID
             join p in Products on o.ProductID equals p.ProductID
             select new { c.Name, o.OrderID, p.ProductName };

//// Method Syntax
var result = Customers.Join(
                 Orders,
                 c => c.CustomerID,
                 o => o.CustomerID,
                 (c, o) => new { c, o }
             ).Join(
                 Products,
                 co => co.o.ProductID,
                 p => p.ProductID,
                 (co, p) => new { co.c.Name, co.o.OrderID, p.ProductName }
             );

```

Method Syntax 在多表連接時會出現「匿名型別套匿名型別」的情況 (co.c、co.o)，可讀性急速下降；Query Syntax 卻仍然貼近 SQL 的直覺。
例如在電商網站裡，要查詢「客戶 → 訂單 → 商品」，Query Syntax 看起來就像一段熟悉的 SQL，而 Method Syntax 會讓人迷路。

### Left Join (GroupJoin + DefaultIfEmpty)

```csharp

//// Query Syntax
var result = from c in Customers
             join o in Orders on c.CustomerID equals o.CustomerID into customerOrders
             from o in customerOrders.DefaultIfEmpty()
             select new { c.Name, OrderID = o?.OrderID ?? 0 };

//// Method Syntax
var result = Customers.GroupJoin(
                 Orders,
                 c => c.CustomerID,
                 o => o.CustomerID,
                 (c, customerOrders) => new { c, customerOrders }
             ).SelectMany(
                 co => co.customerOrders.DefaultIfEmpty(),
                 (co, o) => new { co.c.Name, OrderID = o?.OrderID ?? 0 }
             );

```

Query Syntax 使用 join … into + DefaultIfEmpty，就能自然寫出 Left Join。
Method Syntax 必須用 GroupJoin 產生「客戶 + 訂單群組」，再展開 SelectMany，語法很重。
報表系統中要列出「所有客戶，即使沒有訂單也要顯示」。Query Syntax 可讀性好得多。

<br>
<br>

## 🐛 let（引入中間變數）

Query Syntax 提供 let，允許你在查詢中計算一次值，後續多次使用。

假設我們有一個 Character 類別，表示遊戲中的角色，其中包含角色的名稱、等級和經驗值。我們想要查詢所有達到 5 級及以上的角色，並計算他們升到下一級所需的經驗值。

```csharp

//// Query Syntax
var querySyntax = from character in characters
                  let experienceToNextLevel = GetExperienceToNextLevel(character.Level, character.Experience)
                  where character.Level >= 5
                  orderby experienceToNextLevel ascending
                  select new { character.Name, ExperienceToNextLevel = experienceToNextLevel };

//// Method Syntax
var methodSyntax = characters
    .Select(character => new
    {
        character.Name,
        character.Level,
        character.Experience,
        ExperienceToNextLevel = GetExperienceToNextLevel(character.Level, character.Experience)
    })
    .Where(x => x.Level >= 5)
    .OrderBy(x => x.ExperienceToNextLevel)
    .Select(x => new { x.Name, x.ExperienceToNextLevel });

```

let 可以「命名中間計算結果」，避免重複計算，也讓查詢更好讀。在 Method Syntax 裡，必須用匿名類型 Select 包裝變數，然後再投影出結果。
遊戲角色查詢、財務報表計算（例如 let 稅額 = 金額 * 稅率），Query Syntax 會更直覺。s

<br>
<br>

## 🐛 Cross Join（多來源組合）

這個與 JOIN 不同的是，我們可能不是要交集篩選他們的資料，而是單純從不同資料來源，列出不同的排列組合

```csharp

var levels = new[] { "Level 1", "Level 2", "Level 3" };
var enemies = new[] { "Enemy A", "Enemy B", "Enemy C" };

//// Query Syntax
var querySyntax = from level in levels
                  from enemy in enemies
                  select $"{enemy} in {level}";

//// Method Syntax
var methodSyntax = levels.SelectMany(level => enemies, (level, enemy) => $"{enemy} in {level}");

```

Query Syntax 用兩個 from 就能直接表達「交叉組合」。
Method Syntax 必須用 SelectMany，語意上不如 Query Syntax 清楚。
RPG 遊戲中，關卡 × 敵人 → 所有可能組合
電商中，顏色 × 尺寸 → 所有商品規格組合



| 場景                                  | Query Syntax  | Method Syntax               |
| ----------------------------------- | ------------- | --------------------------- |
| **簡單篩選/投影**                         | ✅ 直覺（像 SQL）   | ✅ 同樣好用                      |
| **多表 Join**                         | ✅ 可讀性好        | ❌ 容易變巢狀難讀                   |
| **Left Join**                       | ✅ 語法糖清晰       | ❌ 必須 GroupJoin + SelectMany |
| **引入中間變數 (let)**                    | ✅ 可命名暫存變數     | ❌ 需匿名型別包裝                   |
| **Cross Join (排列組合)**               | ✅ 多個 from 就搞定 | ❌ 需要 SelectMany             |
| **複雜運算子 (Distinct/Take/Aggregate)** | ❌ 不支援         | ✅ 完整支援                      |

- JOIN / let / Cross Join → Query Syntax（清晰度高，像 SQL）
- 複雜運算、管線式操作 → Method Syntax（功能最完整）
- 專案實務：多數團隊會混用，以「可讀性」為首要考量。