---
title: SQL - Store Procedure
date: 2025-10-06 22:52:05
categories: SQL 職場歷險記
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

## OFFSET 與 FETCH：分頁查詢（Paging）

```sql
-- 第 1 頁：顯示第 1~10 筆
SELECT * FROM Employees
ORDER BY EmployeeID
OFFSET 0 ROWS
FETCH NEXT 10 ROWS ONLY;

-- 第 2 頁：顯示第 11~20 筆
SELECT * FROM Employees
ORDER BY EmployeeID
OFFSET 10 ROWS
FETCH NEXT 10 ROWS ONLY;
```

👉 可以想成像 Netflix：「往右滑 10 部影片，再顯示接下來的 10 部。」


## 在交易中進行分頁 (Paging with Transaction)

```sql
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRANSACTION;

DECLARE
    @StartingRowNumber INT = 1,
    @RowCountPerPage INT = 3;

WHILE (SELECT COUNT(*) FROM HumanResources.Department) >= @StartingRowNumber
BEGIN
    SELECT DepartmentID, Name, GroupName
    FROM HumanResources.Department
    ORDER BY DepartmentID ASC
    OFFSET @StartingRowNumber - 1 ROWS
    FETCH NEXT @RowCountPerPage ROWS ONLY;

    SET @StartingRowNumber = @StartingRowNumber + @RowCountPerPage;
END;

COMMIT TRANSACTION;
```

## 使用運算式 (Expressions)

```sql
DECLARE
    @StartingRowNumber TINYINT = 1,
    @EndingRowNumber TINYINT = 8;

SELECT DepartmentID, Name, GroupName
FROM HumanResources.Department
ORDER BY DepartmentID ASC
OFFSET @StartingRowNumber - 1 ROWS
FETCH NEXT @EndingRowNumber - @StartingRowNumber + 1 ROWS ONLY
OPTION (OPTIMIZE FOR (@StartingRowNumber = 1, @EndingRowNumber = 20));
```




## 💼 FK

外部索引鍵並不一定要唯一

<br>
<br>

## 💼 OLAP / OLTP

OLTP 是為了「快速記錄真實世界的每一筆變動」
OLAP 是為了「快速分析這些變動的長期趨勢」

換句話說：

OLTP 重在「準確、即時、細節完整」
OLAP 重在「彙總、分析、決策支援」


| 類別          | 比喻                                   |
| ----------- | ------------------------------------ |
| **OLTP 系統** | 就像你的「收銀系統」：每筆交易（客人買什麼、多少錢）都要即時登錄。    |
| **OLAP 系統** | 就像「財務報表或經營儀表板」：你要看出哪個月賣最多、哪個產品成長趨勢好。 |

| 面向         | OLTP（Online Transaction Processing）       | OLAP（Online Analytical Processing）    |
| ---------- | ----------------------------------------- | ------------------------------------- |
| **主要用途**   | 處理日常交易（新增、修改、刪除）                          | 分析歷史資料（彙總、比較、統計）                      |
| **資料設計目標** | 資料一致性、避免重複、確保交易完整                         | 查詢速度、聚合效能高                            |
| **資料模型**   | 正規化 (Normalized)                          | 維度模型 (Star Schema / Snowflake)        |
| **資料範例**   | 客戶訂單、庫存變化、付款紀錄                            | 每月銷售報表、年度趨勢分析                         |
| **查詢型態**   | 單筆交易、短查詢                                  | 大量彙總、複雜計算                             |
| **典型 SQL** | `INSERT / UPDATE / DELETE / SELECT TOP 1` | `SELECT SUM(), AVG(), GROUP BY, CUBE` |
| **交易數量**   | 高頻率、筆數多、單筆小                               | 低頻率、批次處理、資料量大                         |
| **使用者角色**  | 一線系統（前台、客服、ERP）                           | 管理層、分析師、BI 系統                         |
| **設計考量**   | ACID、鎖定策略、索引查詢                            | ETL、資料倉儲、快取彙總                         |


## 連接鏈條

```SQL
SELECT DISTINCT 
    SalesOrderData_SalesOrderSlaveId
FROM #tmpSalesOrderDataResponse 
LEFT JOIN dbo.SalesOrderSlavePayProfileType WITH (NOLOCK) 
    ON SalesOrderSlavePayProfileType_SalesOrderSlaveId = SalesOrderData_SalesOrderSlaveId
    AND SalesOrderSlavePayProfileType_ValidFlag = 1
LEFT JOIN dbo.PayProfile WITH (NOLOCK) 
    ON PayProfile_TypeDef = SalesOrderSlavePayProfileType_PayProfileTypeDef 
    AND PayProfile_ValidFlag = 1
WHERE PayProfile_StatisticsTypeDef = @PayProfileStatisticsTypeDef;
```


PayProfile 能不能連上去，完全取決於 SalesOrderSlavePayProfileType 那層有沒有東西。
若該訂單沒有對應的 SalesOrderSlavePayProfileType，那 ppt.* 欄位全是 NULL，
進而 PayProfile_TypeDef = NULL → 這條 JOIN 對不到任何 PayProfile → p.* 全部 NULL。


| 情況                                   | SalesOrderSlavePayProfileType | PayProfile | 最終結果         |
| ------------------------------------ | ----------------------------- | ---------- | ------------ |
| ✅ 有資料，且對應 PayProfile 符合條件            | 有                             | 有且符合       | ✅ 出現在結果      |
| ⚠️ 有資料，但對應 PayProfile 不符合條件          | 有                             | 有但不符       | ❌ 被 WHERE 過濾 |
| ❌ 完全沒有 SalesOrderSlavePayProfileType | 無                             | 不會被 JOIN   | ❌ 被 WHERE 過濾 |


把這三張表想成三段電路：

#tmpSalesOrderDataResponse --(線A)--> SalesOrderSlavePayProfileType --(線B)--> PayProfile

- 線A（第一個 JOIN）壞掉 → 第二段（PayProfile）永遠接不上
- 所以哪怕 PayProfile 本身有電（有資料）
- 只要中間 SalesOrderSlavePayProfileType 那條線沒通，整個電路就不亮


## 🧱 Filegroup 的目的

#### ① 把資料表分散在不同的磁碟，提高效能

假設交易量非常大的表（如 Orders）放到快速 SSD，歷史表、Log 表放到便宜 HDD → 不同 Filegroup 可以綁不同實體磁碟，提高整體 I/O 效能

PRIMARY (SSD)
CashFlowGroup (SSD)
ArchiveGroup (HDD)


#### ② 做分區表（Partition Table）必備

若你要做分區（Partition），例如每個月一個 partition，partition 指定到不同 filegroup，SQL Server 一定要求你有多個 filegroup 才能做 partition。
沒有 filegroup = 無法 partition

雖然你 SELECT 時還是：
```sql
SELECT * FROM TradesOrder
WHERE CreatedAt >= '2024-03-01'
```

但是 SQL Server 只會掃描「3 月的 partition」，而不是把整張大表都掃過。


#### ③ 獨立備份 / 還原 filegroup

大型系統（尤其 TB 級）常需要只備份活躍資料（Active Filegroup）、歷史資料放在 Archive Filegroup，必要時再掛載、只還原某個 filegroup（ex：歷史區域壞掉），這種需求必須靠 Filegroup 才能做到。



## 不是「所有情況 100% 都比反向快」

高基數欄位
Partition Key 跟查詢不一致
小表
模糊查詢沒有真正替代寫法

## SARGable


SARGable = 可被索引最佳化的查詢條件

SARGable → Index Seek（快）

non-SARGable → Index Scan / Table Scan（慢）

差距是：

Seek = O(log n)

Scan = O(n)


## UNION ALL


UNION ALL效能比UNION好，因為UNION會使用類使DISTINCT的演算法

## 反正規化

反正規化（Denormalization） = 刻意把資料表重複一些欄位，讓查詢時不需要 JOIN 或不需要一直存取同一張熱門表

📌 它是刻意做的，不是設計錯誤。
📌 目的：效能、效能、還是效能。


🔥 案例 1：TradesOrderThirdPartyPayment 需要會員 Email

原本：

SELECT t.*, m.Email
FROM TradesOrderThirdPartyPayment t
JOIN CrmMember m ON t.MemberId = m.MemberId


CrmMember 是 Hot Table，JOIN 會讓：

CPU 飆高

IO 飆高

執行變慢

✔ 改成反正規化

在 TradesOrderThirdPartyPayment 直接新增欄位：

MemberEmail
MemberName


寫入訂單時就存好；

未來查詢時：

SELECT MemberEmail FROM TradesOrderThirdPartyPayment


→ 完全不需 JOIN
→ 壓力從 Hot 下來
→ 效能顯著提升（真實案例 10 倍以上）

## where條件使用 OR 來連接條件


SELECT job_id FROM   job  WHERE  job_priority = 3 OR job_priority = 1; 

--可以這樣查詢 
SELECT job_id FROM   job WHERE  job_priority = 3 
UNION ALL 
SELECT job_id FROM   job WHERE  job_priority = 1; 


## cfn_SplitToBigintTableV2
抽成  CSP
LINQ改法:
先將原 List 用 string.Join 轉成逗號分隔的字串。
用函式 cfn_SplitToBigintTableV2
將字串轉成 temp table
temp table 拿來當條件

範例一 連結 SalePageRepository.cs 行1122
temp table Any (Exists)


## Func vs Expression Func


Func 在 LINQ 或 Lambda Expression 中並無法直接轉為 SQL，造成 EF 下查詢條件沒吃到，結果資料表全拉。
Predicate 要用 Expression<Func<Table, bool>> 不能用 Func<Table, bool>，直接丟 Func 他不會轉 Query 語法 會直接送去查完再下條件。
請大家記得，傳遞條件時，一定要用 Expression<Func<T>>
參考文章: https://karatejb.blogspot.com/2016/10/c-func-vs-expression-func.html 

