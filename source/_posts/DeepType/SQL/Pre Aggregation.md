---
title: Pre Aggregation
date: 2024-06-08 23:50:34
categories: 情報の存在論
top_img: https://i.imgur.com/NfCnwwU.png
cover : https://i.imgur.com/NfCnwwU.png
tags:
    - 情報の存在論
toc:
toc_number:
comments :
---


{% tabs Pre Aggregation %}

![akatsuki](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/naruto1.png)


<!-- tab 關於曉-->

曉組織從不會跑去逐村、逐個忍者詢問「你是不是叛忍」，那樣太慢

他們先整理來自各村的大量任務紀錄，任務失敗率、行蹤中斷、查克拉特徵異常、忍術使用頻率…把整個忍界的龐大資料先「彙整」出一小群行為模式明顯脫離正常範圍的忍者。

等到名單縮小之後，曉組織才拿這份「疑似叛忍清單」去比對外部情報：黑市名冊、走私紀錄、賞金通緝、各村密報……這就像 JOIN 小表。

先把大集合縮到只剩「關鍵群體」，外部情報才能精準派上用場；不然會浪費巨大成本也找不到目標


![aaa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Preaggragation/1__ninja__preaggragate.png)


<!-- endtab -->

<!-- tab 關於資料庫-->

## 不好的處理方式

假如有以下資料

| 表格 | 筆數          | 說明                                              |
| -- | ----------- | ----------------------------------------------- |
| a  | 1,300,000 筆 | 主資料表，存放每筆紀錄（包含 name、bid、cid、did、count、position） |
| b  | 5 筆         | 小表，包含欄位 col1                                    |
| c  | 5 筆         | 小表，包含欄位 col2                                    |
| d  | 5 筆         | 小表，包含欄位 col3                                    |

需求是針對每一組 (a.name, a.bid, a.cid, a.did)，計算 SUM(count) 與 AVG(position)，並且 JOIN b、c、d 取出對應的欄位 col1, col2, col3  

```sql
SELECT a.name, SUM(a.count), AVG(a.position), b.col1, c.col2, d.col3
FROM a
JOIN b ON a.bid = b.id
JOIN c ON a.cid = c.id
JOIN d ON a.did = d.id
GROUP BY a.name, b.id, c.id, d.id;
```

JOIN 是非常貴的操作。如果直接把 130 萬筆資料拿去 JOIN 三個小表，SQL 必須把 130 萬筆資料掃過一次又一次，產生超大的中間結果，最後再做 Group By 分組計算，你浪費超多力氣在絕大多數根本沒問題的忍者身上


![ccc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Preaggragation/2___bad__init.png)

![gg](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Preaggragation/2___bad__waste.png)


![ss](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Preaggragation/3___order___history__drown.png)


## ✅ 高效能版本（先彙總再 JOIN）

先壓縮資料量再做 JOIN，本質上是在減少 SQL 引擎必須處理的資料維度，降低中間結果膨脹，進而大幅提升效能

```sql
SELECT a.name, a_sum, a_avg, b.col1, c.col2, d.col3
FROM (
  SELECT name, SUM(count) AS a_sum, AVG(position) AS a_avg, bid, cid, did
  FROM a
  GROUP BY name, bid, cid, did
) a
JOIN b ON a.bid = b.id
JOIN c ON a.cid = c.id
JOIN d ON a.did = d.id;
```


![dd](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Preaggragation/2___fix___less__rows.png)



先在表 a 內做 GROUP BY，這一步會先把 130 萬筆資料，彙整成「每組條件唯一」的結果，例如，假設 name+bid+cid+did 的組合只有 5000 種，那結果就只剩 5000 筆，資料量立刻縮小幾百倍。這個彙總後的小資料集，再去 JOIN b、c、d，JOIN 的對象小得多（只有幾千筆），執行起來非常快，也更節省記憶體，最後再取出需要的欄位，即可得到結果


![ff](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Preaggragation/2___fix__with__preaggre.png)



## GROUP BY 是否一定能減少資料量

GROUP BY 能不能縮小資料量完全取決於「分組維度的獨特程度」

如果 a.name + bid + cid + did 的組合很多（例如每筆都不同），那 GROUP BY 後筆數不會減少，此時「先 GROUP 再 JOIN」就不會改善效能。但在大部分分析情境中，分組後資料筆數通常會大幅縮減，因此先 Group 是合理策略



![aa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Preaggragation/4___cardinality.png)


<!-- endtab -->


<!-- tab tempTable 拆查詢-->

我們想透過 customers JOIN orders，然後算每個客戶有多少訂單，問題是，當 orders 表非常大（例如幾百萬筆），而 customers 只有幾千筆時，這個 JOIN 實際上會建立一個巨大的中間結果（Intermediate Result Set）

```sql
SELECT c.customer_id, c.customer_name, COUNT(o.order_id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name;
```

customers = 1,000 筆
orders = 10,000,000 筆

在 JOIN 的階段，SQL 引擎可能會產生 數百萬筆中間結果，即使最後 GROUP BY 只剩下 1,000 筆結果，這個過程依然耗費了大量的 I/O、記憶體暫存空間 與 CPU 計算。這就是 JOIN 對效能的最大挑戰，JOIN 的過程中，會產生「資料乘積」的中間結果


# Temptable / CTE / Table Variable

這裡把「訂單數的統計」先算好、先壓縮成一個小表，這張表只有每個客戶一筆資料

```plaintext
| customer_id | order_count |
| ----------- | ----------- |
| 1           | 10          |
| 2           | 3           |
| 3           | 0           |
```

再拿這張「小小的統計表」去 JOIN customers，這樣 JOIN 的對象就不再是「幾百萬筆的 orders」，而是「幾千筆的 temp_order_counts」，自然效能提升非常明顯

JOIN 的花費 ≈ 左表筆數 × 右表筆數（在最壞情況下）。而如果能事先把右表（orders）壓縮成只有每個客戶一筆的彙總結果，那 JOIN 的複雜度就從「數百萬 × 數千」變成「數千 × 數千」


## Temporary Table

```sql
-- 建立暫存表（名稱前要加 #）
CREATE TABLE #temp_order_counts (
    customer_id INT,
    order_count INT
);

-- 將聚合結果插入暫存表
INSERT INTO #temp_order_counts
SELECT customer_id, COUNT(*) AS order_count
FROM orders
GROUP BY customer_id;

-- 再使用這個暫存表進行 JOIN
SELECT c.customer_id, c.customer_name, t.order_count
FROM customers c
LEFT JOIN #temp_order_counts t ON c.customer_id = t.customer_id;

-- 最後（可選）刪除暫存表
DROP TABLE #temp_order_counts;
```


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Preaggragation/5___temptable.png)


## CTE

```SQL
WITH temp_order_counts AS (
    SELECT customer_id, COUNT(*) AS order_count
    FROM orders
    GROUP BY customer_id
)
SELECT c.customer_id, c.customer_name, t.order_count
FROM customers c
LEFT JOIN temp_order_counts t ON c.customer_id = t.customer_id;
```


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Preaggragation/5___cte.png)



## Table Variable
```SQL
DECLARE @temp_order_counts TABLE (
    customer_id INT,
    order_count INT
);

INSERT INTO @temp_order_counts
SELECT customer_id, COUNT(*) AS order_count
FROM orders
GROUP BY customer_id;

SELECT c.customer_id, c.customer_name, t.order_count
FROM customers c
LEFT JOIN @temp_order_counts t ON c.customer_id = t.customer_id;
```


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Preaggragation/5__tablevariable.png)



<!-- endtab -->

<!-- tab summary-->


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Preaggragation/6__decision__flow.png)


![cc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Preaggragation/unnamed.png)


<!-- endtab -->


{% endtabs %}