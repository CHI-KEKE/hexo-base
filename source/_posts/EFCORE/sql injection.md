---
title: SQL Injection
date: 2025-10-25 22:25:34
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

## 💼 FromSqlInterpolated

```csharp
var user = "johndoe";
var blogs = context.Blogs
    .FromSqlInterpolated($"EXECUTE dbo.GetMostPopularBlogsForUser {user}")
    .ToList();
```

FromSqlInterpolated 接受 FormattableString（即 $"..."）做為參數。EF Core 會把內插的值（{user}）自動轉成 DbParameter（例如 @p0），並把 SQL template 與參數分開傳給底層 provider（SqlClient）。用在執行 Stored Procedure 或 raw SQL 且要保留可讀性的情境很好用。如果有多個參數，寫法同樣直觀：$"EXEC proc {p1}, {p2}"。

FromSqlInterpolated 並不是把整個字串插入後再送出；EF 會解析 FormattableString.Format 與 FormattableString.GetArguments()，建立 SQL 模板與參數列表。資料庫接收到的是「帶參數的查詢」，因此使用者輸入不會被當作 SQL 執行。這保留了 SQL Server 的執行計畫共享（plan cache）優勢，因為模板相同只是參數不同

<br>
<br>

## 💼 FromSqlRaw

```csharp
var user = "johndoe";
var blogs = context.Blogs
    .FromSqlRaw("EXECUTE dbo.GetMostPopularBlogsForUser {0}", user)
    .ToList();
```

FromSqlRaw 常用於你已經有一個格式化佔位符的 SQL template（{0}, {1}）。EF Core 會把這些佔位符映射至內部產生的參數（@p0, @p1）。比 FromSqlInterpolated 少一些語法糖，但在程式碼生成、或模板是動態構造（例如從外部組合格式字串）時很方便。如果使用 DbParameter（如 SqlParameter）來控制型別/長度/方向，也可以傳入參數物件陣列到 FromSqlRaw。

<br>
<br>

## 💼 SqlParameter

```csharp
var nameParam = new SqlParameter("@Name", SqlDbType.NVarChar, 50) {
    Direction = ParameterDirection.Input,
    Value = "johndoe"
};

var countParam = new SqlParameter("@TotalCount", SqlDbType.Int) {
    Direction = ParameterDirection.Output
};

context.Database.ExecuteSqlRaw("EXEC dbo.GetBlogCountByUser @Name, @TotalCount OUTPUT", nameParam, countParam);
int count = (int)countParam.Value;
```

SqlParameter 屬於 ADO.NET 的參數類別，能明確指定 SqlDbType、長度、是否為 NULL、Direction（輸入/輸出）、以及精確的 ParameterName。必須在要接收 OUTPUT 值或需要控制型別（避免隱式轉換、避免 encoding 問題）時使用。明確指定參數型別可以避免 SQL Server 的隱式型別轉換（implicit conversion）導致索引無法使用或執行計畫不佳。Output 參數在 Stored Procedure 中是 Database 層回傳額外資訊的標準方式：執行 EXEC proc @a, @b OUTPUT 後由 SqlParameter.Value 讀取。

<br>
<br>

## 💼 容易被注入

```csharp
var user = "johndoe'; DROP TABLE Users; --";
var blogs = context.Blogs
    .FromSqlRaw($"EXEC GetPopularBlogs '{user}'") // ❌ 直接把字串拼進 SQL
    .ToList();
```

這裡 $"EXEC GetPopularBlogs '{user}'" 是在 C# 層把 user 的值直接拼進 SQL 字串，然後把整個字串交給 EF。EF 得到的是一段已經包含使用者輸入的 SQL，底層無法把插入的片段切分成「SQL template + parameter」，所以資料庫會把 '; DROP TABLE Users; -- 視為 SQL 語法，於是執行 DROP TABLE。攻擊者藉由在輸入中插入語句終止符（;）、註解符（--）等，結合你拼字串的缺陷達成注入。

<br>
<br>

## 💼 為什麼「參數化」能防注入？

### 字串 vs 參數：不同角色

當你把 user 輸入直接拼到 SQL 字串裡，整段字串會被 SQL Server 當成要「解析執行的程式碼」。攻擊字串可以插入額外 SQL 指令（例如 ; DROP TABLE ...）。而當你使用參數（@p0、@Name），你其實把該資料交給資料庫當值（value） 處理，資料庫不會再把它當成 SQL 指令解析，僅當作字串值傳入。

### EF Core 的處理（在做什麼）

FromSqlInterpolated 背後建立一個 FormattableString，EF 會把內插值轉成 DbParameter（例如 @p0），並藉由 Database provider 傳給 SQL Server。FromSqlRaw("... {0}", user) 則把 {0} 映射成參數，同樣產生 @p0。SqlParameter 讓你直接控制用什麼型別、長度、是否為 Output 等。

### DB 層面保障

資料庫執行的是「參數化查詢/預備語句」，資料本身不會被解析成 SQL 語法片段，因而無法在伺服器端被當作命令執行。

<br>
<br>

## 💼 動態表名/欄位/ORDER BY 不能用參數


SQL 的參數（parameter）設計是代替 值（value），不是代替 語法結構（identifier/keywords）。像表名、欄位名、ORDER BY 的 ASC/DESC、甚至 LIMIT/OFFSET 的欄位選擇，都是 SQL 語法的一部分，資料庫解析器必須在解析階段就知道結構，因此參數無法代表這類結構。如果直接把使用者輸入當成表名或欄位拼進 SQL，攻擊者可以傳入惡意表名或混入 SQL 片段，造成注入或資料暴露。

```csharp
// 危險：把 userInput 直接拼成 SQL 的欄位或表名
string orderBy = Request.Query["orderBy"]; // 來自使用者
var sql = $"SELECT * FROM {orderBy}";     // ❌ 直接拼接表名
```

### 安全作法：白名單（whitelist）或預先映射（mapping）

- 白名單：事先寫好允許的表名/欄位清單，使用者只能從這個清單選擇。
- 映射：不要把輸入直接對應 SQL；用 key → SQL 字串的 mapping。例如使用者傳 "date"，後端把它映射成 OrderBy = "CreatedAt"。
```csharp
var allowedColumns = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase) {
    ["name"] = "Name",
    ["date"] = "CreatedAt",
    ["popularity"] = "PopularityScore"
};

string userChoice = Request.Query["orderBy"] ?? "date";
if (!allowedColumns.TryGetValue(userChoice, out var column)) {
    // fallback 或回傳錯誤
    column = "CreatedAt";
}

string sql = $"SELECT * FROM Blogs ORDER BY {column} DESC"; // column 由白名單決定
```

<br>
<br>

## 💼 不要靠 escaping 代替參數化

Escaping（對特殊字元加斜線或換成 entity）看起來像個捷徑，但它很容易出錯：不同 DB 對 escaping 規則不同（'、"、Unicode、長度、編碼），並且可能被忘記或被錯誤地雙重處理。此外，攻擊者會找出繞過你 escaping 的技巧。例如自己寫 Replace("'", "''") 當成防 injection（雖然對簡單情況短期有效，但不完整，且常被錯誤使用）。或是因用不同層級的 escaping 導致「雙重 escaping」或「未 escaping」。
```csharp
var unsafe = userInput.Replace("'", "''");
var sql = $"SELECT * FROM Users WHERE Name = '{unsafe}'"; // 仍不建議


/// 正確（參數化）：
var sql = "SELECT * FROM Users WHERE Name = @name";
var param = new SqlParameter("@name", userInput);
context.Database.ExecuteSqlRaw(sql, param);
```

<br>
<br>

## 💼 最小權限原則（Least Privilege）

參數化能降低注入風險，但若程式被攻破或出現其他漏洞（例如邏輯錯誤、內部帳密洩漏），擁有過高權限的 DB 帳號會讓攻擊者造成更大破壞（DROP、DELETE、權限提升等）。最小權限減少了「一旦被攻破能做多少事」。Web 應用用的 DB 帳號只給 SELECT/INSERT/UPDATE/DELETE 所需權限；如果不需要 DROP、ALTER、BACKUP，就別給。並且在管理腳本、部署才使用更高權限帳號。使用 Role-based 權限管理（例如 SQL Server 的 role 或 schema-level 限制）。

<br>
<br>

## 💼 ORM 與執行計畫快取（plan cache）

資料庫會把 SQL 的執行計畫（query plan）快取以提升效能。參數化查詢通常能讓多次相同模板但不同參數的查詢重複使用同一份執行計畫，節省解析與編譯成本；反之，若每次都拼不同字串（literal），就會造成計畫碎片化（plan cache bloat），降低效能，甚至引發「parameter sniffing」或不佳的選擇性估算問題。parameter sniffing，DB 在建立執行計畫時會「嗅」當前的參數值來估算資料分布，這在多數情況是好事（可生成最佳計畫），但若後續使用完全不同資料分布的參數，原計畫未必最佳，可能導致效能不佳。