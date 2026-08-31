---
title: NMQ 並發 Race Condition 導致訂單子單轉單遺失分析
date: 2026-05-07
tags: [SQL, Race Condition, NOLOCK, NMQ, Incident]
categories: DeepType/SQL
---

## 事件摘要

一筆 HK 市場訂單（TradesOrderGroupId: `2708256`）的兩筆子單中，只有一筆成功轉單至 ERP，另一筆（`TradesOrderSlaveId: 8877652`）轉單失敗，導致 WebStore 的 `OrderSlaveFlow` 狀態永久卡在 `WaitingToTrans`，且 ERPDB 中完全沒有該子單的資料。

---

## 現象觀察

查詢 WebStore 的 `OrderSlaveFlow`，同一筆訂單的兩筆子單狀態截然不同：

| OrderSlaveFlow_Id | TradesOrderSlaveId | StatusDef | TransToERPDateTime | SalesOrderSlaveId |
|---|---|---|---|---|
| 8877547 | 8877653 | `WaitingToShipping` | 2026-05-07 08:16:38 | 8875983 |
| 8877546 | 8877652 | `WaitingToTrans` | `NULL` | `NULL` |

- **8877653** ✅ 正常轉單，有 `SalesOrderSlaveId`，狀態已推進至 `WaitingToShipping`
- **8877652** ❌ 轉單失敗，無任何 ERP 資料，狀態凍結在 `WaitingToTrans`

查詢 ERPDB 確認：

```sql
SELECT * FROM ERPDB.dbo.OrderSlaveFlow 
WHERE OrderSlaveFlow_TradesOrderSlaveId = 8877652
-- 結果：0 筆，確認完全未轉入 ERPDB
```

---

## 轉單機制說明

轉單流程由 NMQ Job `TransferOrderToERP` 觸發，主要分三步：

```
Step1: csp_ImportWebStoreDBTradesOrdersToERPDBSourceTablesByOrderId_Mall
       └─ 將 WebStore OrderSlaveFlow (WaitingToTrans) 複製至 ERPDB

Step2: TradesOrderTransToSalesOrderWithFlow
       └─ 依 ERPDB 資料建立 SalesOrderSlave

Step3: UpdateDataAfterTradesOrderTransToSalesOrderWithFlow
       └─ 回寫狀態、建立費用單等後處理
```

### CSP 的關鍵 WHERE 條件

```sql
FROM [WebStoreLS].WebStoreDB.dbo.OrderSlaveFlow v1 WITH (NOLOCK)
WHERE OrderSlaveFlow_StatusDef IN ('WaitingToTrans', 'TransToCancel', 'WaitingToThirdPartyTrans')
  AND [OrderSlaveFlow_TradesOrderGroupId] = @runId
  AND NOT EXISTS (
    SELECT 1 FROM ERPDB.dbo.OrderSlaveFlow WITH (NOLOCK)   -- ← NOLOCK！
    WHERE [OrderSlaveFlow_TradesOrderSlaveId] = v1.[OrderSlaveFlow_TradesOrderSlaveId]
  );
```

### CSP 的 CATCH block

```sql
BEGIN CATCH
    IF @@TRANCOUNT > 0
        ROLLBACK TRANSACTION;
    SELECT error_number(), error_message();  -- ← 回傳兩個欄位
END CATCH
```

---

## 關鍵時間線

```
08:15:58.797  兩筆 TradesOrderSlave 狀態皆變為 Finish
08:15:58.843  兩筆 OrderSlaveFlow 同時建立（CreatedDateTime）
08:16:32.503  8877652 的 OrderSlaveFlow UpdatedDateTime
              （此時才更新為 WaitingToTrans，UpdatedTimes=1）
08:16:33.511  NMQ Worker 1 收到 task request（taskId: 0c694576）
08:16:33.514  NMQ Worker 2 收到 task request（taskId: 81d29d5b）
08:16:35.181  Worker 1 啟動 process 18084
08:16:35.118  Worker 2 啟動 process 8868
08:16:38.393  Worker 1 ::Doing
08:16:38.403  Worker 2 ::Doing
08:16:38.430  Worker 1 Step1 (CSP) 開始執行     ← 兩者僅差 6ms
08:16:38.436  Worker 2 Step1 (CSP) 開始執行
              ↕ 兩個 CSP transaction 同時執行中
08:16:38.586  Worker 2 Step2 開始（Step1 已成功 commit）
08:16:38.646  Worker 1 Step1 ❌ exception → ROLLBACK
08:16:38.691  Worker 1 retry → 直接跳至 Step3（跳過 Step1+Step2！）
08:16:38.716  Worker 2 Step3 開始
08:16:42.912  Worker 1 ✅ done（但 8877652 未處理）
08:16:43.004  Worker 2 ✅ done
```

---

## Root Cause 分析

### 問題一：NOLOCK 髒讀（Dirty Read）Race Condition

兩個 Worker 的 CSP 幾乎同時執行，導致以下交錯：

```
Worker 1 (08:16:38.430)              Worker 2 (08:16:38.436)
────────────────────────────────────────────────────────────
BEGIN TRANSACTION
INSERT 8877652 into ERPDB
  (uncommitted, 尚未 commit)
                                     NOT EXISTS check (NOLOCK)
                                       → 髒讀到 8877652 已存在
                                       → 只 INSERT 8877653
                                     COMMIT ✅（只有 8877653）

INSERT 8877653 into ERPDB
  → ❌ PK Conflict！
    （Worker 2 已 commit 8877653）
→ CATCH block
→ ROLLBACK（8877652 消失）
→ SELECT error_number(), error_message()
  ← 回傳「兩個欄位」
→ C# EntityFramework 拋出：
  "The data reader has more than one field."
```

**核心問題：** CSP 的 NOT EXISTS 使用 `WITH(NOLOCK)`，導致 Worker 2 讀到 Worker 1 尚未 commit 的 dirty data，誤以為 8877652 已存在而跳過，最終 Worker 1 rollback 後 8877652 在兩邊都不見了。

### 問題二：NMQ Retry 跳過 Step1+Step2

Worker 1 Step1 失敗後，retry 邏輯**直接從 Step3 開始**，跳過了 Step1（轉入 ERPDB）和 Step2（建立 SalesOrderSlave）：

```
第一次執行：Step1 ❌ → retry
retry：Step3 ✅ → done（8877652 完全沒處理）
```

這是 retry 機制的設計缺陷，應該從 Step1 重試而非從 Step3 跳入。

### 問題三：同一筆 TG 被觸發兩個 NMQ Job

同一筆 `TradesOrderGroupId = 2708256` 在幾乎同一時間點被發了兩個 `TransferOrderToERP` task，這是觸發整個 race condition 的根本原因。

---

## 為何直接 Re-run CSP 無效

`SourceTradesOrderGroup` 的 INSERT **沒有 NOT EXISTS 保護**：

```sql
INSERT INTO [ERPDB].[dbo].[SourceTradesOrderGroup]
...
FROM [WebStoreLS].[WebStoreDB].[dbo].[TradesOrderGroup] WITH (NOLOCK)
INNER JOIN (SELECT DISTINCT tmpGroupId from @tmpOrder) t1
    ON [TradesOrderGroup_Id] = tmpGroupId;
-- 沒有 WHERE NOT EXISTS！
```

Re-run 時的執行流程：

```
1. OrderSlaveFlow INSERT → 8877652 符合條件 → 進入 @tmpOrder ✅
2. SourceTradesOrderGroup INSERT
   → TradesOrderGroup_Id = 2708256 已存在
   → ❌ PK Violation
   → CATCH → ROLLBACK（連帶 rollback 步驟 1）
   → SELECT error_number(), error_message()  ← 又爆
```

---

## 資料確認

| 資料表 | GroupId/SlaveId | 狀態 |
|--------|----------------|------|
| `ERPDB.dbo.SourceTradesOrderGroup` | 2708256 | ✅ 存在 |
| `ERPDB.dbo.SourceTradesOrderSlave` | 8877653 | ✅ 存在 |
| `ERPDB.dbo.SourceTradesOrderSlave` | 8877652 | ❌ **缺失** |
| `ERPDB.dbo.OrderSlaveFlow` | 8877652 | ❌ **缺失** |

---

## 修復方向

### 短期（資料補救）

由於 CSP 整包無法重跑，需要寫 **Operation SQL**，僅補齊 8877652 缺失的 slave-level 資料：

1. `ERPDB.dbo.OrderSlaveFlow`（8877652）
2. `ERPDB.dbo.SourceTradesOrderSlave`（8877652）
3. 其他 slave-level 關聯表（如 `SourceTradesOrderSlavePromotion`、`SourceAuthorized` 等）

### 長期（根本修復）

| # | 問題 | 修復方向 |
|---|------|----------|
| 1 | 同一 TG 被發兩個 NMQ job | 調查重複觸發來源，加入 job deduplication |
| 2 | CSP NOT EXISTS 使用 NOLOCK 造成髒讀 | 改用 `READCOMMITTED` 或加入 Application Lock |
| 3 | NMQ retry 跳過 Step1+Step2 | 修正 retry 邏輯，從失敗的 step 重試 |
| 4 | CSP group-level INSERT 缺乏冪等保護 | 加入 `IF NOT EXISTS` guard |

---

## 學習重點

> **NOLOCK 不等於「安全的讀取」**
>
> `WITH(NOLOCK)` 會讀取其他 transaction **尚未 commit 的 dirty data**。
> 在並發場景下，這可能導致：
> - 誤判資料「已存在」而跳過（本案例）
> - 讀到後來被 rollback 的資料
> - 幻象讀、不一致的資料狀態
>
> 在有並發風險的去重判斷（NOT EXISTS）上，應避免使用 NOLOCK。

