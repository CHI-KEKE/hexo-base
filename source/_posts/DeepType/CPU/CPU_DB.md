---
title: CPU - DB
date: 2025-11-08 23:00:00
categories: Server
top_img: https://github.com/CHI-KEKE/pics/blob/main/Server/cat.jpg?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Server/cat.jpg?raw=true
tags:
    - Server
toc:
toc_number:
comments :
---

{% tabs CPU - DB%}

<!-- tab Node-->

一個 Node = 一個獨立運作的資料庫執行單位，實務上 通常對應一台 Instance（VM / EC2 / RDS instance）

一個「節點」就是一台資料庫伺服器（Server），在分散式或高可用架構中會有多台，也就是多節點佈署

```sql
     +-------------------------+
     |     App / API Server    |
     +-----------+-------------+
                 |
                 v
   ┌─────────────────────────────┐
   │         DB Cluster          │
   ├─────────────────────────────┤
   │ RW Node (Primary)           │  ← 負責寫入、交易、更新
   │    ↑  log shipping          │
   │    ↓  replication           │
   │ RO Node #1 (Replica)        │  ← 專供查詢分流
   │ RO Node #2 (Replica)        │  ← 專供查詢分流
   └─────────────────────────────┘
```


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/1__node.png)


<!-- endtab -->


<!-- tab AlwaysOn Availability Group-->


在 AlwaysOn 裡

- Primary Replica → 一台 SQL Server Instance
- Secondary Replica → 另一台 SQL Server Instance

假設你有三台機器，把它們組成一個 `AlwaysOn Availability Group`

```bash
VM-01：SQL Server Instance
VM-02：SQL Server Instance
VM-03：SQL Server Instance
```

① VM-01，跑一個 SQL Server Instance，他是 Primary Replica，負責寫入資料、接收交易（INSERT / UPDATE / DELETE），這一整台 = 1 個 Node
② VM-02，跑一個 SQL Server Instance，角色是 Secondary Replica（同步），負責即時接收 Primary 的 log，可設定成讀取用，這一整台 = 1 個 Node
③ VM-03，跑一個 SQL Server Instance，角色是 Secondary Replica（非同步），負責延遲接收 log，作為災難備援，這一整台 = 1 個 Node

```bash
Availability Group
 ├─ Node 1：Primary SQL Server
 ├─ Node 2：Secondary SQL Server
 └─ Node 3：Secondary SQL Server
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/2__awalys_on.png)


<!-- endtab -->

<!-- tab AWS RDS 的 Multi-AZ + Read Replica 架構-->


`Multi-AZ` = Availability Zone，同一個 Region 裡，彼此獨立的機房群，例如 ap-northeast-1（東京）有 AZ-a、AZ-b 、AZ-c


在 AWS RDS 裡，一個 DB Instance = 一個 Node，Multi-AZ 是為了「高可用（Failover）」，Read Replica 是為了「讀取擴充（Scale Read）」

假設你建立一個 MySQL / PostgreSQL / SQL Server 的 RDS

```bash
AZ-a
 └─ DB Instance (Primary)

AZ-b
 └─ DB Instance (Standby)   ← Multi-AZ

AZ-c
 └─ DB Instance (Read Replica)
```
這三個 都是獨立存在、獨立運作的 DB Instance

① Primary DB Instance（主資料庫），Application 連線的目標，所有 寫入（INSERT / UPDATE / DELETE）也可以讀，1 個 Node
② Standby DB Instance（Multi-AZ），AWS 自動建立跟 Primary 同步複寫，但不能給你連線，平常你「看不到它在幹嘛」，用途只有一個，Primary 掛掉時，立刻接手，1 個 Node
③ Read Replica（讀取副本），使用者「手動建立」，跟 Primary 非同步複寫，可以被 Application 連線，只能讀（SELECT），用途把大量讀流量導走，1 個 Node

| 項目       | Multi-AZ | Read Replica |
| -------- | -------- | ------------ |
| 目的       | 高可用      | 擴充讀取         |
| 複寫方式     | **同步**   | **非同步**      |
| 能不能連     | ❌ 不能     | ✅ 能          |
| 掛掉會接手寫入？ | ✅ 會      | ❌ 不會         |
| 你要手動切？   | ❌ AWS 自動 | ❌（需你升級）      |




![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/3_multiaz_replica.png)


![d](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/3_multi_repli_table.png)


<!-- endtab -->

<!-- tab RW / RO-->

## 1️⃣ 負載分擔（Load Balancing）

主節點（RW）負責交易、寫入；
從節點（RO）負責查詢報表、API 查詢等。

→ 避免查詢拖慢交易性能

<br>
<br>

## 2️⃣ 高可用性 High Availability

若 RW 節點掛掉，系統可自動切換其中一個 RO 節點升格為新的 RW → 保證服務不中斷

<br>
<br>

## 3️⃣ 容災與異地備援（DR）

RO 節點可能部署在異地區域（region），確保主機毀損時能快速接手

<!-- endtab -->

<!-- tab CPU 效能消耗來源-->

## 邏輯運算（Logical Operations）

- 比較（=, >, <, LIKE）
- 排序（ORDER BY）
- 分組（GROUP BY）
- Hash 計算（Hash Join / Hash Aggregate）
- 表達式計算（CASE、函式、運算）

<br>
<br>

## 資料轉換

（Row → Page → Memory Structure），CPU 花在「把資料變成資料庫能用的形狀」，因為資料庫不是直接拿磁碟上的 bytes 來用，每一層都要 CPU

```bash
Disk Page
 ↓ 讀進來
Buffer Pool Page
 ↓ 解析
Row Structure
 ↓
Execution Engine 使用
```


![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/4_cpu_transfer.png)


**SELECT * 吃 CPU（不是 IO）**

即使資料已經在 memory，但資料庫還要解析 row header、解碼欄位轉成 internal row format，並填進結果集，因此欄位越多，CPU 越高

**資料型別轉換**
```bash
INT → VARCHAR
或
VARCHAR → INT
```

每一 row 都有型別轉換，CPU silently 被吃掉


**Row 太寬（Wide Row）**

```sql
CREATE TABLE Logs (
  Id INT,
  Message NVARCHAR(MAX),
  Payload NVARCHAR(MAX),
  Metadata NVARCHAR(MAX)
);
```

資料庫需要處理 row offset，跳過 large object pointer，建立 row descriptor，因此 Row 越胖，CPU 越高


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/5_wide_and_data_transfer.png)


## 計畫決策（Query Plan Iterator Pattern）

每個算子都要一筆筆「拉取（pull）」資料再傳給上層節點，CPU 花在「一筆一筆拉資料、呼叫下一層」

**Iterator Model**

```bash
SELECT
 ↑
Nested Loop
 ↑
Index Seek
```
每一筆 row 都是一連串 function call
CPU 不是被「算」，而是被「呼叫次數」吃光。


資料庫 CPU 的消耗，不是來自「讀磁碟」，而是來自「計算、轉換、與一筆一筆執行計畫節點的成本」



![xc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/6_iterator_model.png)


<!-- endtab -->

<!-- tab CPU HIGH 原因分析-->

## 大量 join（尤其是 Nested Loop 或 Hash Join）

Nested Loop 是「O(n×m)」級別的比對行為，外層表每一筆都去內層表找匹配，每個比對都佔 CPU 指令
Hash Join 事先建立一個 Hash Table，再以雜湊比對，Hash 運算需要對每筆 key 做雜湊（Hash Function 計算），大量記憶體分配與比對（尤其當 Hash Bucket 滿或溢出），Hash Join = 高記憶體 + 高 CPU，犧牲 CPU 計算換取 I/O 減少


![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/7_nested_loop_join.png)


## 🔹 大範圍掃描（Table Scan / Index Scan）**

每一筆都要「Evaluate Predicate」（判斷 WHERE 條件），若表非常大，即使資料都在快取中，也要對每一筆跑一次比對，即使 I/O 不慢，CPU 仍要解析每一筆 row → expression → 評估結果
Table Scan 是「O(n) 次邏輯判斷」，CPU 被大量消耗在條件運算。


![ff](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/8_table_scan_index_seek.png)


## 🔹 計算聚合（SUM、COUNT、GROUP BY）

要建「Group Hash Table」或排序後群組，每筆資料都要計算雜湊、插入群組結構，SUM、AVG 等需要累積計算；COUNT 需要每筆增加，若 GROUP BY 欄位多或資料筆數大，Hash Table 膨脹 → CPU 緊繃

<br>
<br>

## 🔹 過多的排序（Sort 操作）

Sort 是純 CPU 工作，屬於「O(n log n)」運算，SQL 會將資料搬入工作記憶體（Sort Memory），執行比較與交換，若記憶體不夠會溢出至 tempdb（那才是 I/O high），即使在記憶體內部，排序演算法（如 QuickSort）也非常吃 CPU


![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/9_sort_aggre.png)


## 🔹 複雜的 Scalar Function、子查詢或計算欄位

每筆 row 都要呼叫一次函數（成千上萬次），SQL function 是逐行執行（無法向量化），沒有批次優化，若函數內部又有查詢、日期轉換、字串處理 → 全部都是 CPU-bound


也就是說資料庫忙著思考，當 stored procedure 的 query plan 被 SQL Server 儲存或重複使用，但它的計畫選擇不佳，也會導致每次執行都吃大量 CPU

| 類型                           | 解釋                                                         |
| ---------------------------- | ---------------------------------------------------------- |
| **Parameter Sniffing**       | SQL Server 針對第一次執行的參數建立計畫，後面用相同計畫處理不同參數，導致效率差（例如某參數導致掃全表）。 |
| **Missing Index / Index 不佳** | 缺乏適合的索引，導致 Table Scan 或 Hash Match。                        |
| **Join Strategy 錯誤**         | 預估行數錯誤，導致 Hash Join 被選中、內存壓力上升。                            |
| **過度重複執行**                   | 該 procedure 被高頻呼叫（例如每個 request 都查會員券），導致 CPU 累積上升。         |
| **複雜聚合與排序**                  | 例如 DISTINCT、GROUP BY、ORDER BY、大量 CASE WHEN。                |



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/10_scalar.png)


<!-- endtab -->

<!-- tab 檢查 Query Plain 方式-->

- 檢查是否有 Table Scan
- 是否有 `Estimated Number of Rows` 與 `Actual Number of Rows` 相差很大（估算錯誤）
- 檢查是否有 `Sort`、`Hash Match`、`Nested Loop` 等昂貴運算

<br>
<br>

CPU High 的本質不是「跑太多 SQL」，而是「每支 SQL 想太多」，就像一個人看書,若書有章節索引（索引良好），能快速找到重點，若沒有章節（缺索引），他就得從頭翻到尾（Table Scan），若還反覆翻同一本書（高頻呼叫），那 CPU 就會爆


![gg](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/11_summ_tABLE.png)


<!-- endtab -->

<!-- tab 搶券優化 - 關於 tempTable 濫用-->

[Pull Request](https://bitbucket.org/nineyi/nineyi.databases/pull-requests/7971/diff)

這支 SP 為取得會員已領取的折價券資料，核心流程是：

- 先查主檔 ECoupon（優惠券定義）
- 再查子檔 ECouponSlave（每一張券子紀錄）
- 篩出該會員已領取的券（ECouponSlave_IsTake = 1）
- 回傳券與子券的資訊

因此移除不必要的 temp table，可以減少一次 IO，透過移除中間 materialization（建立、插入、讀取 temp table 的過程），將不再觸發額外的 Hash/Sort/TempDB 操作

<!-- endtab -->

<!-- tab 後台付款查詢 - 關於索引-->

https://91app.slack.com/archives/G06A3GDC7/p1759195224449909


```SQL
SELECT TOP(2) [s].[SalesOrderPayProfileType_TotalFee] AS [SalesOrderPayProfileTypeTotalFee]
FROM [SalesOrderPayProfileType] AS [s]
WHERE (([s].[SalesOrderPayProfileType_ValidFlag] = CAST(1 AS bit)) AND ([s].[SalesOrderPayProfileType_SalesOrderId] = @__salesOrderId_0)) AND ([s].[SalesOrderPayProfileType_PayProfileTypeDef] = @__payProfileType_1)
```

從程式碼看出，查詢多付款的查詢沒有合適的索引，需要加上 `SalesOrderPayProfileType_SalesOrderId`

https://91appinc.visualstudio.com/DB%E6%95%88%E8%83%BD%E5%84%AA%E5%8C%96/_workitems/edit/463329


![j](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/12_REAL_CASE.png )


<!-- endtab -->

<!-- tab summary-->


![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/az.png)

<!-- endtab -->


{% endtabs %}