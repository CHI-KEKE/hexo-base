---
title: SQL - 時の雫
date: 2025-10-26 11:01:34
categories: 
top_img: https://i.imgur.com/NfCnwwU.png
cover : https://i.imgur.com/NfCnwwU.png
tags:
    - SQL 職場歷險記
toc:
toc_number:
comments :
---


![SQL](https://i.imgur.com/NfCnwwU.png)

從時間欄位出發 → 做運算或分段 → 推導出狀態或統計結果

<br>
<br>


## 💼 三種常見的時間分析模式

| 類型                            | 說明              | 範例           | 關鍵函式                               |
| ----------------------------- | --------------- | ------------ | ---------------------------------- |
| **① 時間差模式 (Time Gap)**        | 用「時間差」判斷狀態或剩餘時間 | HR 系統判斷誰快退休  | `DATEDIFF`, `DATEADD`, `CASE WHEN` |
| **② 時間分群模式 (Time Bucketing)** | 把連續的時間切成段落來統計   | 每月、每日、每週銷售分析 | `MONTH`, `DATEPART`, `DATEADD`     |
| **③ 時間對齊模式 (Time Rounding)**  | 把時間「取整」對齊成標準區間  | 每分鐘、每小時登入量   | `DATEADD(DATEDIFF(...))`           |

<br>
<br>

## 💼 查詢的共通術式

| 技術重點                          | 說明                 |
| ----------------------------- | ------------------ |
| `DATEADD` / `DATEDIFF`        | 處理時間偏移與差距（加減天、月、年） |
| `DATEPART` / `MONTH` / `YEAR` | 擷取時間單位（分群依據）       |
| `CASE WHEN`                   | 將數值轉成語意化狀態         |
| CTE (`WITH`)                  | 建立暫存結果，讓查詢更乾淨易讀    |
| 分群 (`GROUP BY`)               | 對時間區間進行統計彙總        |
| 篩選 (`WHERE`)                  | 控制分析期間（區間分析）       |

<br>
<br>

## 👴 時間差模式：從距離推導狀態


「還有幾天退休？」一個簡單的問題，其實是一場時間運算的練習。

假設公司 HR 想知道哪些員工即將退休（退休年齡為 65 歲），我們可以

- 先算出員工的退休日期：生日 + 65 年
- 再用 DATEDIFF() 算出「今天」與「退休日」的天數差
- 若結果為負數，代表已過退休日
- 最後用 CASE WHEN 將結果轉成語意化標籤


| 技術點                                           | 說明                                          |
| --------------------------------------------- | ------------------------------------------- |
| **CTE (Common Table Expression)**             | 用 `WITH AAA AS(...)` 建立暫時性結果集，方便閱讀與維護。      |
| **`DATEADD(YEAR,65,VipMemberInfo_Birthday)`** | 計算出每位員工滿 65 歲的日期。                           |
| **`DATEDIFF(DAY,GETDATE(),...)`**             | 算出今天距離退休日還有幾天。若是負數，代表已經超過退休日。               |
| **`CASE WHEN` 判斷式**                           | 根據 `wait_retire_days` 的結果，標示為「待退弟兄」或「好好工作」。 |

```sql
USE WebStoreDB;

WITH AAA AS(
SELECT VipMemberInfo_FullName,VipMemberInfo_Birthday,DATEDIFF(DAY,GETDATE(),DATEADD(YEAR,65,VipMemberInfo_Birthday)) AS wait_retire_days
FROM VipMemberInfo(NOLOCK)
WHERE VipMemberInfo_ValidFlag = 1
AND VipMemberInfo_Birthday IS NOT NULL
)
SELECT VipMemberInfo_FullName,VipMemberInfo_Birthday,wait_retire_days,CASE
		when (wait_retire_days <= 0) THEN N'待退弟兄' ELSE N'好好工作'
	   END AS N'退休狀態'
FROM AAA
```

<br>
<br>

## 🏪 時間分群模式：把時間軸切成段落

- 每月訂單數量、每日銷售額
- 每月註冊會員數、登入數
- 每小時玩家登入人數

<br>

### 月份

**方法一：MONTH 函式**
```sql
use WebStoreDB;

with aaa as(
SELECT MONTH(TradesOrderGroup_DateTime) AS Year_Mon,*
FROM TradesOrderGroup(NOLOCK)
WHERE TradesOrderGroup_ValidFlag = 1
AND TradesOrderGroup_DateTime >= '2025-01-01')
select Year_Mon as N'年月',count(*) as N'訂單數量'
from aaa
GROUP by Year_Mon
```

- 運算結果為整數，排序與計算都快
- CPU 成本低
- 不吃索引（欄位被函式包住）
- 只顯示月份（無法區分跨年資料）

👉 適合中型資料量的快速月報分析。

**📅 方法二：CONVERT 函式**
```sql
with aaa as(
SELECT CONVERT(VARCHAR(7), TradesOrderGroup_DateTime, 120) AS Year_Mon,*
FROM TradesOrderGroup(NOLOCK)
WHERE TradesOrderGroup_ValidFlag = 1
AND TradesOrderGroup_DateTime >= '2025-01-01')
select Year_Mon as N'年月',count(*) as N'訂單數量'
from aaa
GROUP by Year_Mon
```

- 同樣不吃索引（函式包欄位）
- 但還要多做「日期 → 字串」的轉換
- 字串運算成本比整數高，而且排序時需要字元比較（collation）
- 結果雖然可讀性高，但效能會稍微差一點
- 相對較適合跨年份分析，結果包含年份和月份

👉 適合報表輸出（例如要顯示 '2025-01' 這種格式），但犧牲效能。

<br>

### 對齊模式：每分鐘、每小時的統計


- 每分鐘登入人數統計
- 每 5 分鐘 API 呼叫次數
- 每小時交易數量

```sql
WITH AAA AS(
	SELECT dateadd(MINUTE,DATEDIFF(MINUTE,0,Task_CreatedDatetime),0) AS task_by_min
	FROM Task(NOLOCK)
	WHERE Task_JobId = 6
      AND Task_CreatedDatetime >= '2025-10-26 00:00:00'
         AND Task_CreatedDatetime < '2025-10-27 00:00:00'
)
select task_by_min,COUNT(*)
from AAA
GROUP BY task_by_min
```

<br>

### 周
```sql
USE ERPDB;

WITH A AS (
    SELECT SalesOrderGroup_DateTime, SalesOrderGroup_TotalPayment
    FROM SalesOrderGroup(NOLOCK)
    WHERE SalesOrderGroup_ValidFlag = 1
        AND SalesOrderGroup_DateTime >= '2025-06-01'
        AND SalesOrderGroup_DateTime < '2025-07-01'
)
SELECT 
    DATEPART(WEEK, SalesOrderGroup_DateTime) - 
    DATEPART(WEEK, DATEADD(MONTH, DATEDIFF(MONTH, 0, SalesOrderGroup_DateTime), 0)) + 1 AS 當月第幾週,
    AVG(SalesOrderGroup_TotalPayment) AS 週平均消費
FROM A
GROUP BY DATEPART(WEEK, SalesOrderGroup_DateTime) - 
         DATEPART(WEEK, DATEADD(MONTH, DATEDIFF(MONTH, 0, SalesOrderGroup_DateTime), 0)) + 1;
```