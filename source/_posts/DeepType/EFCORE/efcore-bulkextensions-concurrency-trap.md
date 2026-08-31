---
title: EF Core 樂觀鎖的假象：BulkExtensions 讓 Rowversion 形同虛設
date: 2026-05-04
tags:
  - EF Core
  - BulkExtensions
  - Concurrency
  - 樂觀鎖
  - Race Condition
categories:
  - DeepType
  - EFCORE
---

## 前言

你有沒有遇過這樣的情況：

- DB 有 `Rowversion` 欄位
- EF Core Model 設定了 `IsRowVersion().IsConcurrencyToken()`
- 但實際測試發現並發更新時，**完全沒有任何衝突偵測**？

這篇文章從一個真實的促購活動加價購功能出發，完整拆解這個坑是怎麼形成的。

---

## 案例背景

系統有一張 `PromotionEngineSpecialPrice`（活動價格表），記錄加價購活動下每個 SKU 的加購價格。

業務流程是：
1. SCMAPIV2（後台 API）接收營運人員的批次更新請求
2. 驗證通過後，呼叫 PromotionWebAPI 的 `POST /api/cart-extra-purchase/batch-update`
3. PromotionWebAPI 查詢現有價格記錄，再批次更新價格與排序

這個流程看起來合理，但有一個隱藏的並發問題。

---

## 問題一：Lost Update（更新遺失）

### 情境

兩個請求 A、B 同時針對同一個活動的相同 SKU 發起更新：

```
T=0  A 查詢 DB → specialPrice { TypeValue: 100, rowversion: 0x001 }
T=0  B 查詢 DB → specialPrice { TypeValue: 100, rowversion: 0x001 }

T=1  B 寫入完成：TypeValue 改為 200，DB rowversion 變成 0x002
T=2  A 寫入完成：TypeValue 改為 150，DB rowversion 變成 0x003
     ← B 的更新被靜默覆蓋，沒有任何錯誤！
```

這就是 Lost Update，B 的寫入憑空消失。

---

## 問題二：DB 明明有 Rowversion，為何沒保護？

### DB Schema 與 EF 設定都是正確的

`PromotionEngineSpecialPrice` 有 `Rowversion` 欄位：

```csharp
// PromotionEngineSpecialPrice.cs
/// <summary>
/// 資料版本
/// </summary>
public byte[] PromotionEngineSpecialPrice_Rowversion { get; set; }
```

EF Core DbContext 也設定了樂觀鎖：

```csharp
// WebStoreDBContext.cs
entity.Property(e => e.PromotionEngineSpecialPrice_Rowversion)
    .IsRowVersion()
    .IsConcurrencyToken()   // ← EF Core 樂觀鎖標記
    .HasComment("資料版本");
```

**標準的 EF Core `SaveChangesAsync` 下，這樣的設定會生效：**

```sql
-- EF Core 產生的 UPDATE（正確）
UPDATE PromotionEngineSpecialPrice
SET TypeValue = 150, UpdatedDateTime = ...
WHERE Id = 1
  AND Rowversion = 0x001  -- ← 若已被改動，0 rows affected → DbUpdateConcurrencyException
```

若 B 已先改掉 rowversion，A 的 UPDATE 影響 0 筆，EF 拋出 `DbUpdateConcurrencyException`，保護機制生效。

---

## 問題三：BulkExtensions 繞過樂觀鎖

實際的寫入方法長這樣：

```csharp
// SpecialPriceRepository.cs
public async Task BulkUpdateAsync(List<PromotionEngineSpecialPrice> items)
{
    items.ForEach(item =>
    {
        this._webStoreReadWriteDBContext.PromotionEngineSpecialPrice.Attach(item);
        var entry = this._webStoreReadWriteDBContext.Entry(item);
        entry.State = EntityState.Modified;
    });

    await this._webStoreReadWriteDBContext.BulkSaveChangesAsync();  // ← 問題在這裡
}
```

**`BulkSaveChangesAsync` 是 EFCore.BulkExtensions 提供的方法，底層使用 SqlBulkCopy 或直接批次 SQL，預設不走標準 EF SaveChanges 流程，因此不會檢查 `IsConcurrencyToken`。**

產生的 SQL 大約是：

```sql
-- BulkExtensions 產生的 UPDATE（有問題）
UPDATE PromotionEngineSpecialPrice
SET TypeValue = 150, UpdatedDateTime = ...
WHERE Id = 1
-- ← 完全沒有 WHERE Rowversion = ... 的條件
```

不管 rowversion 是否已被改動，UPDATE 永遠成功，最後寫入的人贏。

---

## 問題四：AsNoTracking 讓問題雪上加霜

讀取資料的方法：

```csharp
// SpecialPriceRepository.cs
public List<PromotionEngineSpecialPrice> GetListWithSku(int promotionEngineId, List<long> skuIds, long shopId)
{
    var promotionEngineSpecialPriceQuery = this._webStoreReadWriteDBContext.PromotionEngineSpecialPrice
        .AsNoTracking()  // ← 不追蹤
        .Where(...)
        ...
    return promotionEngineSpecialPriceQuery.ToList();
}
```

`AsNoTracking()` 讓查出來的 entity 不被 DbContext 追蹤原始值。

標準 EF 樂觀鎖的工作原理需要 DbContext 記住「查詢時的 rowversion 原始值」，之後 SaveChanges 才能帶進 WHERE 條件。`AsNoTracking()` 後手動 `Attach` 雖然 rowversion 欄位值仍在，但 BulkExtensions 根本不讀它。

---

## 完整問題鏈

```
設計意圖                    實際執行
────────────────────────    ────────────────────────────────────
DB: Rowversion 欄位    ✅   DB 有欄位，但沒被利用
EF: IsConcurrencyToken ✅   設定正確，但被 BulkExtensions 繞過
讀: AsNoTracking       ⚠️   版本資訊讀得到，但 context 不追蹤
寫: BulkSaveChanges    ❌   直接 SQL，不走 EF 標準更新流程
────────────────────────    ────────────────────────────────────
最終結果：樂觀鎖完全沒有作用
```

---

## 這條路徑上的其他問題

### 問題 A：BatchCreateAsync 沒有 await

```csharp
// CartExtraPurchaseController.cs 第 60 行
// ❌ Fire-and-Forget！DB 還沒寫完就回 200
this._cartExtraPurchaseService.BatchCreateAsync(this._traceContext.ShopId, entity);
return this.Ok();

// ✅ 正確寫法（BatchUpdateAsync 是對的）
await this._cartExtraPurchaseService.BatchUpdateAsync(this._traceContext.ShopId, entity);
return this.Ok();
```

**後果：**
- DB 寫入失敗時，例外被靜默吞掉
- Caller 收到 200 OK，但資料根本沒進 DB

### 問題 B：Add/Update/Remove 三批次無原子性

```csharp
// 三個各自獨立的 try/catch
try { BatchCreateCartExtraPurchase(addRequest); }    // ← 失敗繼續跑
catch { messages.AddRange(errors); }

try { BatchUpdateCartExtraPurchase(updateRequest); } // ← 失敗繼續跑
catch { messages.AddRange(errors); }

try { BatchDeleteCartExtraPurchase(removeRequest); } // ← 失敗繼續跑
catch { messages.AddRange(errors); }
```

**後果：Add 失敗，但 Delete 仍執行 → 資料被刪除但新增沒成功，停在中間狀態。**

### 問題 C：錯誤訊息寫死類型名稱

```csharp
// CartExtraPurchaseController.cs 第 101 行
// ❌ 滿件加價購也走這裡，但訊息寫死「滿額加價購」
Message = $"批次更新滿額加價購商品失敗: {ex.Message}",
```

---

## 如何正確使用 EF Core 樂觀鎖

### 方式一：使用標準 SaveChangesAsync

```csharp
// 正確：讓 EF 追蹤並帶入 rowversion WHERE 條件
var item = await _context.PromotionEngineSpecialPrice
    .FirstOrDefaultAsync(x => x.Id == id);  // 有 Tracking，不用 AsNoTracking

item.TypeValue = newPrice;
// EF 自動產生：UPDATE ... WHERE Id = ? AND Rowversion = <原始值>
await _context.SaveChangesAsync();  // 若 0 rows affected → DbUpdateConcurrencyException
```

### 方式二：BulkExtensions 加入 Rowversion 條件

EFCore.BulkExtensions 有設定可開啟 ConcurrencyToken 檢查：

```csharp
var bulkConfig = new BulkConfig
{
    UseTempDB = true,
    // 明確指定要比對的並發欄位
    ConcurrencyColumnNames = new List<string>
    {
        nameof(PromotionEngineSpecialPrice.PromotionEngineSpecialPrice_Rowversion)
    }
};
await _context.BulkUpdateAsync(items, bulkConfig);
```

### 方式三：DB 層加 UPDLOCK/ROWLOCK

若需要悲觀鎖：

```sql
-- Stored Procedure 內加鎖
SELECT * FROM PromotionEngineSpecialPrice WITH (UPDLOCK, ROWLOCK)
WHERE PromotionEngineId = @Id AND ShopId = @ShopId
```

---

## 總結

| 層次 | 狀態 | 說明 |
|------|------|------|
| DB Schema | ✅ 有 Rowversion 欄位 | 基礎設施備好了 |
| EF Core 設定 | ✅ `IsRowVersion().IsConcurrencyToken()` | EF 層也設定好了 |
| 讀取方式 | ⚠️ `AsNoTracking()` | 版本值有讀到，但 context 不追蹤原始值 |
| 寫入方式 | ❌ `BulkSaveChangesAsync` | 繞過 EF 標準流程，樂觀鎖失效 |
| 最終效果 | ❌ 完全沒有並發保護 | Lost Update 問題存在 |

**核心教訓：**

> `IsRowVersion().IsConcurrencyToken()` 只在 **標準 EF Core `SaveChangesAsync`** 路徑下有效。  
> 只要使用 BulkExtensions、Raw SQL、Stored Procedure 等方式繞過 EF 標準流程，樂觀鎖就形同虛設。  
> **有設定 ≠ 有保護。**

---

## 相關程式碼位置

| 檔案 | 說明 |
|------|------|
| `PromotionEngineSpecialPrice.cs` | DB Model，含 Rowversion 欄位定義 |
| `WebStoreDBContext.cs` 第 528 行 | `IsRowVersion().IsConcurrencyToken()` 設定 |
| `SpecialPriceRepository.cs` 第 75 行 | `BulkUpdateAsync` 實作（問題根源） |
| `CartExtraPurchaseService.cs` 第 147 行 | `GetListWithSku` + `BulkUpdateAsync` 呼叫鏈 |
| `CartExtraPurchaseController.cs` 第 60 行 | `BatchCreateAsync` 缺少 `await` |
| `PromotionEngineService.cs` 第 991 行 | Add/Update/Remove 三批次無原子性 |
