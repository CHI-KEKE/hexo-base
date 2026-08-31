---
title: Datetime Flow
date: 2025-10-09 11:00:34
categories: 情報の存在論
top_img: https://i.imgur.com/NfCnwwU.png
cover : https://i.imgur.com/NfCnwwU.png
tags:
    - 情報の存在論
toc:
toc_number:
comments :
---


{% tabs Datetime Flow%}


![S](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/DatetimeFlow/landing.png)


<!-- tab Flow-->

所有與時間相關的資料問題，第一步一定是先切出正確的時間工作區，再談任何分析、轉換與計算

![time](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/DatetimeFlow/1__working_zone.png)


在資料查詢與分析中，「流程感」會具體化成這件事 : 先用時間，把資料切成一塊安全、可工作的區間

所以在任何進階操作之前，第一步永遠是時間參數化（所有後續的地基），因此在做下面任何事情之前

- 格式轉換（YEAR / MONTH / CONVERT / 字串化）
- 計算邏輯（DATEDIFF、CASE、狀態判斷）
- 資料聚合（GROUP BY、AVG、SUM）

我們都應該先完成一件事，用「可走索引的時間區間參數」把資料切出來。而這三類操作有一個共通點，它們都會降低索引的價值，甚至直接讓索引失效，因此正確順序不是我要算月報 → GROUP BY MONTH → 再來想效能，而是先用時間縮小資料範圍 → 再隨便你怎麼算


<!-- endtab -->

<!-- tab 時間參數化-->

任何進階分析、轉換、統計之前，第一件事永遠是：先用「可走索引的時間區間參數」把資料切出一塊安全的工作區

- 格式轉換（YEAR / MONTH / CONVERT / 字串化）
- 計算邏輯（DATEDIFF、CASE、狀態判斷）
- 資料聚合（GROUP BY、AVG、SUM）

這三件事有一個共同點，它們都會破壞索引的使用價值，或至少讓索引變得沒那麼重要

所以正確的順序不是：我要算月報 → GROUP BY MONTH → 再來想效能，而是先用時間參數把資料量縮到「合理範圍」 → 再隨便你怎麼算


![G](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/DatetimeFlow/1__index__killers.png)


<!-- endtab -->


<!-- tab 當你對時間欄位動手算，流程就已經反了-->


## ❌ 非 SARGable：先算再篩

```sql
USE WebStoreDB;

SELECT VipMember_CreatedDateTime, 
       DAY(VipMember_CreatedDateTime), 
       VipMember_Id
FROM VipMember(NOLOCK)
WHERE VipMember_ValidFlag = 1
  AND DAY(VipMember_CreatedDateTime) = 1; -- Non-SARGable，無法有效利用索引
```

`VipMember_CreatedDateTime` 本來是一個「可排序、可用索引搜尋」的欄位，DAY(VipMember_CreatedDateTime)在這裡 SQL Server 必須對每一筆資料先算 DAY()，算完後才能知道是不是等於 1，因為索引裡存的是「原始欄位值」，不是「DAY() 計算後的結果」，所以 Optimizer 無法用索引快速定位，只能把整張表（或整個索引）掃過一次，結果就是 Index Scan / Table Scan → 效能直接爆炸

你等於對 SQL Server 說：「每一筆你都幫我算完，我再告訴你要不要」


![A](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/DatetimeFlow/2.__anti_nonsargeble.png)


## ✅ SARGable：先切範圍，再處理

```sql
DECLARE @firstDayOfMonth date = '2025-10-01'; -- 先準備一個明確、可比較的邊界值

SELECT VipMember_CreatedDateTime,
       VipMember_Id
FROM dbo.VipMember WITH (READCOMMITTED)
WHERE VipMember_ValidFlag = 1
  AND VipMember_CreatedDateTime >= @firstDayOfMonth               -- 10/1 00:00
  AND VipMember_CreatedDateTime <  DATEADD(DAY, 1, @firstDayOfMonth); -- 10/2 00:00
```


![B](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/DatetimeFlow/2.___fix___sargeble.png)


<!-- endtab -->


<!-- tab 迴圈實現同一件事反覆發生-->

當發現需求其實是「同一件事重複做，只是某個參數在規律變化」，那問題就不再是 SQL，而是要不要交給迴圈處理

## 先抽離「你真正要做的事」

你不是在查 1 月，也不是在查 2 月，而是「給我一個月份，我就能查出該月份的資料」

## 辨識「哪些是固定的，哪些是會變的」

- 固定的：查詢邏輯，表、欄位、條件結構
- 會變的：只有「月份」

## 把「會變的東西」抽成參數

- 月份 → `@month`
- 年份 → `@year`
- 月初 → `@startDate` , 用 `DATEFROMPARTS(@year, @month, 1)` 產生 `@startDate`
- 月末 → `@endDate`, `@endDate = DATEADD(DAY, 1, @startDate)`（半開區間）


![C](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/DatetimeFlow/3__get_to_parameters.png)



## 邏輯設計


![D](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/DatetimeFlow/3__iterative__flow.png)


```bash
設定年度、月份 = 1
 │
 ▼
[WHILE 迴圈]
 ├─► 設定 @startDate = 當年某月1號
 │    @endDate = 當年某月2號
 │
 ├─► 從 VipMember 找出該日註冊資料
 │    → INSERT 進暫存表
 │
 └─► 月份 + 1，繼續下一回合
 │
```
```sql
USE WebStoreDB;

DROP TABLE IF EXISTS #VipMember_FirstDay;
CREATE TABLE #VipMember_FirstDay(
    CreateDate DATETIME,
    Id BIGINT
);

DECLARE @year INT = YEAR(GETDATE()),
        @month INT = 1,
        @startDate DATETIME,
        @endDate DATETIME;

WHILE @month <= 12
BEGIN
    SET @startDate = DATEFROMPARTS(@year, @month, 1);
    SET @endDate = DATEADD(DAY, 1, @startDate);

    INSERT INTO #VipMember_FirstDay(CreateDate, Id)
    SELECT VipMember_CreatedDateTime, VipMember_Id
    FROM VipMember(NOLOCK)
    WHERE VipMember_CreatedDateTime >= @startDate
        AND VipMember_CreatedDateTime < @endDate;

    SET @month = @month + 1;
END

SELECT * FROM #VipMember_FirstDay;
```

![G](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/DatetimeFlow/3__iterative__flow__code.png)


<!-- endtab -->

<!-- tab 發現重複運算，其實是在提醒你「邏輯混在一起了」-->


```sql
CONVERT(...)
SELECT / GROUP BY / ORDER BY
```
當看到這種程式碼，其實是在暗示資料轉換、資料彙總，都被寫在同一個層級裡了!


## ❌ 月報的反流程寫法

「我想知道『今年』每一個月實際收到多少訂單金額，用來做營運分析與決策。」 

- GROUP BY CONVERT(CHAR(7), TradesOrderGroup_DateTime, 120)與 ORDER BY CONVERT(CHAR(7), TradesOrderGroup_DateTime, 120) 都會讓 SQL Server 對每筆資料進行字串轉換
- CONVERT() 出現在 SELECT、GROUP BY ORDER BY，SQL Server 會視為三次運算，CPU 多做了三倍的事

```sql
USE WebStoreDB;

SELECT 
    CONVERT(CHAR(7), TradesOrderGroup_DateTime, 120) AS 月份,
    SUM(TradesOrderGroup_TotalPayment) AS 總金額
FROM TradesOrderGroup(NOLOCK)
WHERE TradesOrderGroup_DateTime >= DATEFROMPARTS(YEAR(GETDATE()), 1, 1)
    AND TradesOrderGroup_DateTime < DATEFROMPARTS(YEAR(GETDATE()) + 1, 1, 1)
GROUP BY CONVERT(CHAR(7), TradesOrderGroup_DateTime, 120)
ORDER BY CONVERT(CHAR(7), TradesOrderGroup_DateTime, 120);
```


![E](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/DatetimeFlow/4___anti__threetimes.png)



## ✅ 有流程感的月報寫法

- 用暫存表先篩出「今年的資料」
- 後續的分組與轉換只在後面上運算
- 用 CTE 預先轉換一次月份 CONVERT() 只執行一次

```sql
use WebStoreDB

-- 避免 where 無法吃到索引
DROP TABLE IF EXISTS #yyyyy;
select TradesOrderGroup_Id,
		TradesOrderGroup_DateTime,
		TradesOrderGroup_TotalPayment
into #yyyyy
from TradesOrderGroup(nolock)
where TradesOrderGroup_DateTime >= DATEFROMPARTS(YEAR(GETDATE()),1, 1)
AND TradesOrderGroup_DateTime < DATEFROMPARTS(YEAR(GETDATE()) + 1, 1, 1);

-- 避免重複 convert 運算
WITH AA AS(
	SELECT convert(char(7),TradesOrderGroup_DateTime, 120) AS MON,
			TradesOrderGroup_TotalPayment
	FROM #yyyyy
)
select MON,SUM(TradesOrderGroup_TotalPayment) AS Payment
from AA
GROUP BY MON
```


![F](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/DatetimeFlow/4___fix__toflow.png)



## ✅ 另一個例子 - 出貨天數

「今年每個月，平均一筆訂單從『建立』到『實際出貨』要花幾天？」 


![R](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/DatetimeFlow/4__shipping_case.png)


```sql
SELECT 
    OrderSlaveFlow_Id,
    OrderSlaveFlow_CreatedDateTime,
    OrderSlaveFlow_ShippingOrderSlaveDateTime
INTO #FlowTemp
FROM OrderSlaveFlow (NOLOCK)
WHERE OrderSlaveFlow_CreatedDateTime >= DATEFROMPARTS(YEAR(GETDATE()), 1, 1)
  AND OrderSlaveFlow_CreatedDateTime < DATEFROMPARTS(YEAR(GETDATE()) + 1, 1, 1);

WITH CTE AS (
    SELECT 
        YEAR(OrderSlaveFlow_CreatedDateTime) AS Y,
        MONTH(OrderSlaveFlow_CreatedDateTime) AS MON,
        DATEDIFF(DAY, OrderSlaveFlow_CreatedDateTime, OrderSlaveFlow_ShippingOrderSlaveDateTime) AS timespentforShipping
    FROM #FlowTemp
)
SELECT 
    Y, MON,
    AVG(timespentforShipping) AS 平均出貨時間
FROM CTE
GROUP BY Y, MON;
```


![Q](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/DatetimeFlow/4___report__code.png)


<!-- endtab -->


<!-- tab summa-->


![hk](https://github.com/CHI-KEKE/pics/blob/main/SQL/DatetimeFlow/summa.png?raw=true)


<!-- endtab -->

{% endtabs %}
