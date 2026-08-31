---
title: EF Core - 資料是否存在
date: 2025-05-28 08:26:05
categories: 資料疆界的航圖
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Exists/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Exists/1_landing.png
tags:
    - EF CORE
toc:
toc_number:
comments :
---



{% tabs Exist%}

<!-- tab Exist-->

> 「當我們只想確認某人是否出現過，沒必要翻遍整本通訊錄。」

在資料庫的世界裡，我們經常會遇到一個簡單卻重要的問題：資料是否存在？

不論是確認某個用戶是否曾下單、某篇文章是否已發佈，這些判斷存在與否的查詢看似簡單，卻往往決定了系統的效能與延遲。尤其當資料表龐大、條件複雜時，使用錯誤的寫法可能導致嚴重的效能問題。

今天，我們就從 SQL 與 EF Core 的角度，來聊聊「資料是否存在」


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Exists/2_book.png)


<!-- endtab -->

<!-- tab SQL 寫法：COUNT vs EXISTS-->

如果只是想知道「是否有訂單」


## COUNT

```SQL
IF (SELECT COUNT(*) FROM Orders WHERE CustomerID = 'ALFKI') > 0
BEGIN
    PRINT '有訂單'
END
```

但其實效能更佳的寫法是使用 EXISTS
```SQL
IF EXISTS (SELECT 1 FROM Orders WHERE CustomerID = 'ALFKI')
BEGIN
    PRINT '有訂單'
END
```

SELECT 1 是慣用寫法，實際上裡面選什麼都不重要，重點是條件判斷。

## 📈 效能差異

- `COUNT(*)` 會掃描整個資料表，計算總筆數，即使只想知道「有沒有資料」。
- `EXISTS` 一旦找到第一筆符合條件的資料就會停止搜尋，大幅減少 CPU 與 I/O 成本。

這種差異在資料量大或查詢條件複雜時，會更加明顯



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Exists/3_early_sql.png)


<!-- endtab -->


<!-- tab 會員有沒有下過單-->

## EXISTS

```sql
SELECT CASE
    WHEN EXISTS (
        SELECT 1
        FROM orders
        WHERE customer_id = 100
    ) THEN 'Y'
    ELSE 'N'
END;
```


## COUNT

```sql
SELECT CASE
    WHEN COUNT(*) > 0 THEN 'Y'
    ELSE 'N'
END
FROM orders
WHERE customer_id = 100;
```

<!-- endtab -->

<!-- tab SQL 寫法：COUNT vs EXISTS-->


<!-- tab 指標-->

| 指標                | COUNT(\*)  | EXISTS    |
| ----------------- | ---------- | --------- |
| **Actual Rows**   | 可能掃到數萬筆    | 可能只需 1 筆  |
| **Logical Reads** | 高，掃描整個資料頁面 | 少，只讀到一筆就停 |
| **Elapsed Time**  | 隨資料量增加而增加  | 幾乎固定且低    |



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Exists/4_compare_3_index.png)



## 1️⃣ Actual Rows（實際資料列數）

🔍 定義：這個執行步驟實際吐出了幾筆資料

在 Execution Plan 中，與 Estimated Rows 比對，能看出是否估算準確。如果 Estimated Rows 和 Actual Rows 相差很大，表示 SQL Server 預估錯誤，可能導致選錯執行計畫。可以驗證查詢是否處理了大量不必要的資料（如 COUNT(*)）。分析是否有「過度資料處理」的問題


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Exists/5_1_actual_rows.png)


## 2️⃣ Logical Reads（邏輯讀取）

🔍 定義：SQL Server 讀取**資料頁面（data pages）**的次數，不管這些頁面來自記憶體（Buffer Pool）還是磁碟，有可以說**資料庫為了回答這個查詢，實際翻了幾頁資料來看**

SQL Server 裡，資料不是一筆一筆在底層讀的，它是以 Page 為單位在處理，通常一頁是 8KB。這是衡量查詢 I/O 成本最關鍵的指標之一。使用 `SET STATISTICS IO ON` 可以查詢。測試哪種語法能減少資料頁掃描次數。判斷是否有索引支援查詢條件。觀察是否使用 Index Seek vs Scan


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Exists/5_2_logical_read.png)


## 3️⃣ Elapsed Time（耗時）

🔍 定義：從查詢開始執行到結束所花費的時間（以毫秒為單位）。

使用 SET STATISTICS TIME ON 可取得。包含 CPU Time（實際運算時間）與 IO Wait、Network 等延遲。真正觀察查詢「體感」效能是否良好。用於比較不同寫法在實際環境的速度差異。


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Exists/5_3_elapsed_time.png)


<!-- endtab -->

<!-- tab EF Core Count vs Any-->


```CSHARP
public async Task AnyTest()
{
    Console.WriteLine("== Any ===");
    await _adventureWorks2022DbContext.Rds.AnyAsync();

    Console.WriteLine("== COUNT ===");
    await _adventureWorks2022DbContext.Rds.CountAsync();
}
```

若你只是想知道「有沒有資料」，建議使用 AnyAsync()，因為：

- `Any()` 對應 SQL 的 EXISTS
- `Count()` 對應 SQL 的 COUNT(*)

![MarkExists](https://github.com/CHI-KEKE/pics/blob/main/EF/Performance/MarkExists.png?raw=true)


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Exists/6_ef.png)


<!-- endtab -->

<!-- tab 複雜查詢-->


即便是有 JOIN 的複雜條件查詢，只要不關心實際內容，只想知道「有沒有資料」，也可以用 Any()

```CSHARP
var query = (from shopCategorySalepage in _webStoreDbReadOnlyContext.ShopCategorySalePage
            join Salepage in _webStoreDbReadOnlyContext.SalePage
                on shopCategorySalepage.ShopCategorySalePage_SalePageId equals Salepage.SalePage_Id
            where shopCategorySalepage.ShopCategorySalePage_ValidFlag == true
                && shopCategorySalepage.ShopCategorySalePage_CategoryId == shopCategoryId
                && Salepage.SalePage_ValidFlag == true
            select 1);

return query.Any();
```

這樣就能轉換成效能優化的 EXISTS 查詢，而不需實際撈取資料內容


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Exists/7_complex.png)



<!-- endtab -->

<!-- tab whisper-->

有時，我們不是要看清楚每一筆資料的模樣，而只是想知道它在不在？

在資料量龐大的現實世界中，「存在」這個問題，該用最輕巧的方式來確認。EXISTS 與 Any() 就是為了這樣的需求而生的工具


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Exists/Exist.png)


<!-- endtab -->


{% endtabs %}