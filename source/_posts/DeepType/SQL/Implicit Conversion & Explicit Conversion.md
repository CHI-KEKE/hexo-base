---
title: Implicit Conversion & Explicit Conversion
date: 2025-10-07 08:27:05
categories: 情報の存在論
top_img: https://i.imgur.com/NfCnwwU.png
cover : https://i.imgur.com/NfCnwwU.png
tags:
    - 
toc:
toc_number:
comments :
---


{% tabs Conversion %}

<!-- tab 關於婚姻與育兒...-->

生小孩後的夫妻，就像兩個本來型別不同的欄位，一個是「我覺得應該這樣做」的 string，一個是「我以為你希望我這樣做」的 bigint，兩人的期待、習慣、壓力都不一致，但因為忙、因為累、因為怕吵架，沒人敢明確地把自己的需求「顯式轉型」說出來，於是關係裡就開始出現「隱式轉換」

「算了，我先做吧，不然會更麻煩。」

...

「我應該能撐一下，他今天也很累。」

...

「這沒什麼啦，我忍一下就過了。」

看似體貼，能運作，但表面好看，內裡不對等

累 → 生氣
生氣 → 委屈
委屈 → 無聲
無聲 → 彼此以為「沒事」


需要轉型本身並沒有錯誤，資料的來源、格式、責任歸屬、生命周期都跨越多個系統與團隊，此時資料庫遇到 JOIN 或比較時，就會產生轉換需求，重要的是要能有溝通的認知，預防性做好準備

![family](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/family.png)



<!-- endtab -->

<!-- tab Implicit Conversion-->

當兩個不同型別的值需要比較時，資料庫必須強制把它們變成「同一種型別」，而這個自動決定的型別若不夠精確，就會導致邏輯錯誤


## 精確度陷阱（Double Precision Trap）

1. 發現型別不同，在 JOIN ... ON a.k = b.k 中，資料庫看到 a.k 是 string、b.k 是 bigint
2. 套用內建的「隱式轉換規則」，大多數引擎採用「升格為 float/double」的策略，因為浮點數可同時容納字串數字、整數數字
3. 把兩邊都轉成 double 再比，雙方變成 IEEE-754 double 型別

![A](https://github.com/CHI-KEKE/pics/blob/main/SQL/Convert/implicit_string_bigint.png?raw=true)

4. double 無法精確表示大整數 → 發生「格子化」，超過 2^53 的整數會掉到同一個 double 格子（間隔不是 1，而是 2、4、8...）
5. 不同的原始數字變成相同的 double → 比較結果錯誤, JOIN 誤判為相等 → 產生錯誤比對、重複資料或錯配


double 不是為了安全比較整數而設計的，只是為了讓轉型比較「能運作」，不是「正確運作」，因為多數開發者以為 double 很「大」，但忽略了它並不是「精確」，尤其超過 15–16 位的整數一定失真，這個陷阱不是少數引擎才有，而是所有使用 IEEE-754 的語言與資料庫都固有的限制。double（IEEE-754）只能精確表示到 2^53（≈ 9.0e15）以內的整數。超過這個範圍，double 會以「格子」的方式表示數字，每個格子相距不是 1，而是 2、4、8、16、32…（距離隨數字變大而變大），於是兩個原本不同的整數（例如 111111111111111111 與 111111111111111110），一轉成 double 就被「四捨五入」到同一個格子，變成相等，JOIN 就被誤判連上了，導致「重複或錯配」

❌❌ 跨型別比較 → 被轉成 double → 精度不足 → 不同值落在同一個 double 表示 → JOIN 判斷為相等


![C](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Convert/implicit_double_iswrong.png)


## 為什麼隱式轉換一定會影響效能

如果 a.k、b.k 型別一致，上面有 index，則資料庫最理想的執行方式是

- Index Seek / Hash Join / Merge Join
- 用索引直接定位「可能相等的值」

這是 O(log n) 或接近線性的快法


實際被執行的邏輯變成
```sql
ON CAST(a.k AS double) = CAST(b.k AS double)
```
索引是建在「原始型別」上的，不是建在 double 結果上，所以對 Optimizer 來說原本的索引 ≠ 現在要比較的東西，索引失效（或只能部分使用），資料庫只能全表掃描（Full Scan），每一列做 runtime conversion，再一筆一筆比對，CPU 使用率上升、記憶體壓力上升、IO 次數增加



## JOIN 不是唯一會被害的地方

只要牽涉跨型別比較，任何操作都有可能中招，而不只 JOIN

`WHERE a.k = b.k`
`GROUP BY`
`DISTINCT`
`ORDER BY`

都會因為比較前的隱式轉換而失真，它是一個「系統性錯誤來源」，不是單一語法風險

```SQL
-- a.k 是 String: '111111111111111111'
-- b.k 是 BigInt:  111111111111111110
SELECT a.k, b.k
FROM tbl_str AS a
JOIN tbl_int AS b
ON a.k = b.k;     -- 很多引擎會把兩邊轉成 double 再比
-- 可能得到一筆錯誤的匹配（看似相等）
```


![AAA](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Convert/implicit_join_not_right.png)

<!-- endtab -->

<!-- tab Explicit Conversion-->


在 JOIN 條件中使用 CAST，乍看之下是為了解決型別不一致的問題，但實際上，這樣做會讓原本可以利用索引的欄位變成「運算結果」，導致索引失效，查詢效能大打折扣

```SQL
SELECT a.k, b.k
FROM tbl_str a
JOIN tbl_int b
  ON a.k = CAST(b.k AS String);   -- 兩邊都是字串，'...111' ≠ '...110'，不會連上
```
這樣寫確實解決了「型別精確度」的問題，但也悄悄引爆了另一連串效能瓶頸


1. 讀到 JOIN 條件 `a.k = CAST(b.k AS String)`
   - 資料庫看到右邊不是欄位，而是一個即時計算的表達式
2. 比對兩邊型別
   - 兩邊都變成字串，精確度問題（數字精度誤差）被解決
3. 優化器開始決定如何執行 JOIN
   - 發現 CAST(b.k AS string) 不是索引欄位
4. 索引失效 → 只能掃表 → 這使得 b.k 的索引完全派不上用場
5. 逐筆運算 CAST
   - 資料庫只能一筆一筆做 CAST，再比對 a.k
6. 產生低效查詢 → 越多資料 → 越多 CAST → 越慢


- 掃描範圍變大（可能全表掃描 Full Table Scan）
- CPU 計算變多（每筆都要做型別轉換），如果表中有百萬筆資料，那 CAST(b.k AS string) 就會被呼叫百萬次
- 記憶體使用變高（暫時儲存轉換後結果）

這種「為了解決 A 問題而製造出 B 問題」的情境，其實很常見。寫 SQL 時，表面上跑得動不代表背後沒在冒煙，尤其在資料量大的情況下，一點小轉型，就能成為效能殺手


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Convert/explicit_low_performance.png)


<!-- endtab -->

<!-- tab 先轉型好再 JOIN（避免在 JOIN 內 CAST）-->


與其在 JOIN 裡動手腳，不如先把該轉的型別轉好，再來 JOIN，這樣才能保留索引、保住效能

```sql
SELECT a.k, b2.k
FROM tbl_str a
JOIN (
    SELECT k, CAST(k AS CHAR) AS k_str
    FROM tbl_int
) b2
ON a.k = b2.k_str;
```

這樣就能讓 b.k 保留它的索引，CAST 也只做一次，查詢速度會是天壤之別!


![FFF](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Convert/fix_subquery.png)


## Temporary Table

```SQL
-- 將字串欄位轉成 BIGINT，結果存到臨時表
SELECT CAST(k AS BIGINT) AS k
INTO #tmp_str       -- # 表示「本地臨時表」
FROM tbl_str;

SELECT a.k, b.*
FROM #tmp_str AS a
JOIN tbl_int AS b
  ON a.k = b.k;
```

#tmp_str 是「本地臨時表（Temporary Table）」，只在當前連線（Session）中有效，SQL Server 在執行 SELECT INTO 時會自動建立表結構 ，b.k 是 BIGINT，a.k 也已經轉成 BIGINT，所以 JOIN 條件型別一致，SQL Server 可以使用 tbl_int.k 上的索引（Index Seek），而轉型只發生一次（在建立臨時表時），JOIN 不會再重複轉換


若臨時表很大、JOIN 的資料量不少，可以加索引來進一步加速
```SQL
CREATE INDEX IX_tmp_str_k ON #tmp_str(k);
```


![CCC](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Convert/fix_temp.png)


<!-- endtab -->

<!-- tab 誤用 cfn-->


在某次專案中，開發目的是要透過傳入的 IdList（資料庫型別為 INT），去查出對應資料。但原始程式碼卻誤用了 cfn_SplitToBigintTableV2，雖然它確實能把字串拆成 Table，但拆出來的型別是 BIGINT (long)，真的可惜，就差一點細心了

[程式邏輯參考](https://gitlab.91app.com/commerce-cloud/nine1.promotion/nine1.promotion.web.api.frontend/-/merge_requests/10/diffs#52825afabb3b6ba[…]66437389f9a3376)

這會導致 隱式轉型，而這會對效能造成不小的影響，因為資料庫在處理 JOIN 時，會發現兩邊型別不同，只能「悄悄」幫你做型別轉換，這個轉換會讓原本有機會用上的索引直接失效，最後變成全表掃描 + 逐筆轉型 + 效能雪崩


![F](https://github.com/CHI-KEKE/pics/blob/main/SQL/Convert/impliciy_wrong_case.png?raw=true)


<!-- endtab -->


<!-- tab 拆出 GroupBy 案例-->

## GroupBy All in One

```sql
SELECT 
    CAST(h.OrderDate AS DATE) AS OrderDateOnly,
    SUM(CASE WHEN d.ProductID = 776 THEN d.OrderQty ELSE 0 END) AS Product_776_qty,
    SUM(CASE WHEN d.ProductID = 777 THEN d.OrderQty ELSE 0 END) AS Product_777_qty
FROM Sales.SalesOrderHeader AS h
INNER JOIN Sales.SalesOrderDetail AS d ON h.SalesOrderID = d.SalesOrderID
GROUP BY CAST(h.OrderDate AS DATE)
ORDER BY CAST(h.OrderDate AS DATE);
```

這樣每一筆資料在做 GROUP BY 和 ORDER BY 的時候，都要重複計算多次 CAST(h.OrderDate AS DATE)
執行計劃可能是這樣，每一筆都會經過一次 Compute Scalar（做 CAST 運算），再進入分組邏輯
```bash
Clustered Index Scan → Compute Scalar (CAST) → Hash Aggregate (GROUP BY)
```

## GroupBy 拆查詢

```sql
use AdventureWorks2022

DROP TABLE IF EXISTS #BUYBUYTEMP;
SELECT CAST(OrderDate AS DATE) AS OrderDateOnly,
	   ProductID,
	   OrderQty
INTO #BUYBUYTEMP
FROM Sales.SalesOrderHeader(NOLOCK)
INNER JOIN Sales.SalesOrderDetail(NOLOCK)
ON SalesOrderDetail.SalesOrderID = SalesOrderHeader.SalesOrderID

SELECT OrderDateOnly,
	   SUM(CASE WHEN ProductID = '776' THEN OrderQty ELSE 0 END) AS Product_776_qty,
	   SUM(CASE WHEN ProductID = '777' THEN OrderQty ELSE 0 END) AS Product_777_qty
FROM #BUYBUYTEMP
group by OrderDateOnly
order by OrderDateOnly
```

這裡 CAST() 只會做一次，存在中間結果集中（記憶體暫存或 tempdb），後續的 GROUP BY 直接用「已經轉好的值」來分組，這樣「CAST」的開銷不會被重複放大，而且 SQL Optimizer 能更有效率地調度資源（例如記憶體 Grant）

```bash

#step1
Clustered Index Scan → Compute Scalar (CAST) → Output to temp result

#step2
Hash Aggregate (GROUP BY on already-cast column)
```

![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Convert/cast_seperate.png)

<!-- endtab -->

<!-- tab summary-->

![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Convert/notes.png)

<!-- endtab -->

{% endtabs %}