---
title: SQL - 文字列の森
date: 2025-10-09 08:57:34
categories: 
top_img: https://i.imgur.com/NfCnwwU.png
cover : https://i.imgur.com/NfCnwwU.png
tags:
    - 
toc:
toc_number:
comments :
---


![ocean](https://i.imgur.com/NfCnwwU.png)


<br>


## 💼 最長欄位

SQL Server 在執行 ORDER BY LEN(Video_Introduction) 時，必須對每一筆資料都算一次長度，然後再排序
```sql
USE InfoDB;

SELECT TOP 1 LEN(Video_Introduction),Video_Introduction
FROM Video(NOLOCK)
ORDER BY LEN(Video_Introduction) DESC
```


若僅僅想知道最長，不需要排序!
```sql
SELECT MAX(LEN(Video_Introduction)) FROM Video (NOLOCK);
```

<br>
<br>

## 💼 LIKE '%xxx%' 效能差？

索引是一種「字典排序的結構（B-tree）」，想像成一本電話簿（索引）

<br>

### 🔹 有開頭字串的 LIKE

你可以很快找到「開頭是 09」的電話，但如果你說「我要找結尾是 1234」的電話，電話簿就得一筆筆翻完才知道

📘 索引是有方向的（從左到右），開頭沒固定的查詢無法用索引來加速


| 執行策略           | 說明              | 效能   |
| -------------- | --------------- | ---- |
| **Index Seek** | 透過索引直接找到目標資料列   | 🚀 快 |
| **Index Scan** | 掃描整個索引（但不一定整張表） | 🐢 中 |
| **Table Scan** | 掃描整張資料表         | 🐌 慢 |

當你用 LIKE '%xxx%'，SQL Server 無法知道從哪裡開始搜尋，因此只能用 Table Scan

| 查詢               | 能否使用索引 | 原因                  |
| ---------------- | ------ | ------------------- |
| `LIKE 'abc%'`    | ✅ 可以   | 開頭是固定字串，索引能知道從哪裡開始找 |
| `LIKE 'abc%xyz'` | ✅ 部分可用 | 開頭仍固定，結尾條件在資料列比對時過濾 |
| `LIKE '%abc'`    | ❌ 不行   | 無法知道從哪裡開始找，只能掃描全表   |
| `LIKE '%abc%'`   | ❌ 不行   | 同上，開頭未知             |
| `=`（等於）          | ✅ 完全可用 | 精確比對，最有效率           |

假設有一個索引在 username 欄位上，索引的順序可能是
```plaintext
aaron
adam
alex
amy
ben
bob
```

SQL Server 可以從索引樹中快速定位「a」開頭的第一筆 (aaron)，然後一路往後掃到不符合的那筆為止 (ben 為止)，這樣的過程叫 Index Range Seek，效能 ok!

<br>

### 🔹 搭配其他條件

雖然 LIKE '%xxx%' 不能用索引，但若搭配其他條件，可以讓 SQL Server 少掃一些資料
```sql
SELECT * 
FROM logs
WHERE log_date >= '2025-01-01' 
  AND log_date < '2025-01-02'
  AND message LIKE '%error%';
```

即使 message 用不到索引，但 log_date 的條件可以讓 SQL Server 先從日期範圍內的資料掃描，縮小搜尋集!

<br>

### 🔹 先依索引條件撈取到 Temp Table 再做 LIKE

```sql
-- Step 1：先利用索引欄位 (log_date) 篩選資料進暫存表
SELECT *
INTO #tmpLogs
FROM logs WITH (NOLOCK)
WHERE log_date >= '2025-01-01';  -- ✅ log_date 有索引，能 Index Seek

-- Step 2：在暫存表內做 LIKE 搜尋（小範圍比對）
SELECT *
FROM #tmpLogs
WHERE message LIKE '%error%';  -- ❌ 暫存表沒索引，但資料量已縮小
```

- 第一步用索引條件縮小資料量（Index Seek）
- 第二步再在少量資料上做 LIKE

<br>
<br>

##　💼 字串拼接

<br>

### 🔹 大量字串串接（loop 累加）

```sql
DECLARE @result NVARCHAR(MAX) = '';
DECLARE @i INT = 1;

WHILE @i <= 50000
BEGIN
    SET @result = CONCAT(@result, 'X');
    SET @i += 1;
END
```

- 每次迴圈都產生新字串 → 記憶體重分配 → CPU 高
- 在沒有索引、無 schema 可改的情況下，這樣會極慢

<br>

✅ 改善：一次聚合字串，不逐筆累加
```sql
DECLARE @result NVARCHAR(MAX);

SELECT @result = STRING_AGG('X', '') 
FROM (SELECT TOP (50000) 'X' AS val FROM sys.objects) AS x;

SELECT LEN(@result);
```

- 使用 STRING_AGG()（SQL Server 2017+）
- 一次聚合而非逐次串接 → 省下大量 CPU 與 TempDB 資源
- 不需要改任何資料結構

<br>

###　🔹 WHERE 裡用 CONCAT 導致索引失效

```sql
SELECT *
FROM Users
WHERE CONCAT(FirstName, LastName) = 'JohnDoe';
```

✅ 改善寫法
```sql
SELECT *
FROM Users
WHERE FirstName = 'John' AND LastName = 'Doe';
```

- 邏輯上等價（除非你的資料允許不同組合的空格／大小寫）
- 這樣 SQL Server 可獨立比對兩個欄位，而不用做函數運算

<br>

### 🔹 隱性轉換

```sql
SELECT * 
FROM Customers
WHERE CONCAT(CustomerID, '') = '123';
```

CustomerID 是 INT，CONCAT() 會強制轉為 NVARCHAR → CPU 開銷 + 無法最佳化

<br>

✅ 改善寫法（型別正確）
```sql
SELECT * 
FROM Customers
WHERE CustomerID = TRY_CAST('123' AS INT);
```

- 明確轉型，避免隱性轉換
- Query Optimizer 能更精準預估成本（雖仍是掃描，但更穩定）

<br>

### 🔹 CONCAT + DISTINCT / GROUP BY 效能差

```sql
SELECT DISTINCT CONCAT(FirstName, LastName)
FROM Users;
```

✅ 改善寫法（利用 ROW_NUMBER 篩唯一）
```sql
SELECT FirstName, LastName
FROM (
    SELECT FirstName, LastName,
           ROW_NUMBER() OVER (PARTITION BY FirstName, LastName ORDER BY (SELECT NULL)) AS rn
    FROM Users
) AS x
WHERE rn = 1;
```

<br>

### 🔹 NULL 處理差異（CONCAT vs +）

| 用法             | 行為                           | 範例結果                            |
| -------------- | ---------------------------- | ------------------------------- |
| `+`（字串串接運算子）   | **只要任一邊是 NULL，整個結果就變成 NULL** | `'John' + NULL = NULL`          |
| `CONCAT()`（函數） | **自動將 NULL 視為空字串 ''**        | `CONCAT('John', NULL) = 'John'` |


```sql
SELECT CONCAT(FirstName, ' ', LastName) AS FullName
FROM Users;
```

<br>

✅ 改善寫法（手動控制 NULL）
```sql
SELECT 
    FirstName + 
    CASE WHEN LastName IS NOT NULL THEN ' ' + LastName ELSE '' END AS FullName
FROM Users;
```

<br>

### 🔹 不讓 DB 做格式化工作
```sql
SELECT CONCAT(FirstName, ' (', Email, ') - ', Phone) AS DisplayName
FROM Users;
```
這是純輸出格式化工作，占用 DB CPU

<br>
<br>

## 💼 REPLACE

<br>

### 🔹 清除不需要的特殊符號

例如清掉換行符號、HTML 標籤、控制字元、或不可列印字元
```sql
-- 清除換行符號與 Tab
SELECT REPLACE(REPLACE(Description, CHAR(13), ' '), CHAR(10), ' ') AS CleanText
FROM Products;
```

- 常見於清理匯入文字資料（CSV、爬蟲資料、Excel 上傳內容）
- CHAR(13) 是 CR，CHAR(10) 是 LF。很多文字欄位會有這些換行符號

<br>

### 🔹 移除多餘空白或特殊符號
```sql
-- 把電話號碼清理成純數字
SELECT REPLACE(REPLACE(REPLACE(Phone, '-', ''), '(', ''), ')', '') AS CleanPhone
FROM Customers;
```

- 清理使用者輸入或外部系統匯入的資料格式
- 尤其是電話、身分證、信用卡號、稅號等

<br>

### 🔹 移除 HTML 或特殊標籤

```sql
SELECT REPLACE(REPLACE(REPLACE(Content, '<b>', ''), '</b>', ''), '&nbsp;', ' ') AS PlainText
FROM Articles;
```

<br>

### 🔹 資料格式化與轉換（Formatting / Transformation）

```sql
SELECT REPLACE('2025/10/09', '/', '-') AS DateFormatted;
-- 結果：2025-10-09
```

<br>

### 🔹 清理金額或數值中的千分位符號
```sql
SELECT CAST(REPLACE('1,234,567.89', ',', '') AS DECIMAL(18,2)) AS Amount;
```

<br>

### 🔹 調整資料分隔符號
```sql
SELECT REPLACE(Address, ',', ';') AS ExportAddress
FROM Customers;
```

<br>
<br>

## 💼 CHARINDEX

用來「找出某個字串在另一個字串中出現的位置」的函數

<br>

### 🔹 取出字串的某部分（搭配 SUBSTRING）
```sql
SELECT 
    SUBSTRING(Email, CHARINDEX('@', Email) + 1, LEN(Email)) AS Domain
FROM Users;
```

- CHARINDEX('@', Email) 找出 @ 在哪裡
- SUBSTRING(..., +1, LEN(...)) 從下一個字元開始取到結尾

結果就是 email 的 domain，例如：gmail.com

<br>

### 🔹 判斷欄位是否含特定字元後再做運算

```sql
SELECT 
  CASE 
     WHEN CHARINDEX('-', ProductCode) > 0 THEN '有連字號'
     ELSE '無連字號'
  END AS Result
FROM Products;
```

- 資料清理與格式驗證（例如：代碼中是否含非法符號）
- 實務上常搭配 REPLACE() 或 PATINDEX() 一起使用

