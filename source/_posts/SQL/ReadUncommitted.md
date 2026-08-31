---
title: ReadUncommitted - 應用
date: 2025-11-11 10:31:05
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

它允許「髒讀 (dirty read)」，即能讀到尚未 commit 的資料，讀取過程也不會對任何資料加 lock（不會阻塞其他 transaction）。在高併發的電商場景中，這確實可以用來提升查詢效能，但只能用在「對準確性要求不高」的情境


- 查詢只讀、不影響交易邏輯
- 查詢結果可容忍暫時誤差
- 若使用 ORM可在 connection 層設定 transaction isolation
- WITH (NOLOCK)

## 1️⃣ 商品瀏覽頁（商品列表、搜尋結果）

使用者瀏覽商品時，看到的價格或庫存即使延遲幾秒、甚至略有偏差，也不影響體驗。

範例：

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT id, name, price, stock FROM products WHERE category_id = 10 LIMIT 50;
```


使用者只是在「逛」，非購買瞬間，資料一致性不必嚴格 → 減少 shared lock，提升商品列表查詢速

## 2️⃣ 熱門商品排行榜（依銷量或瀏覽量排序）

排行榜本身是近似實時資料，但不需要精確到 transaction level。

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT product_id, SUM(quantity) as total_sales
FROM order_items
GROUP BY product_id
ORDER BY total_sales DESC
LIMIT 10;
```

數字偶爾誤差 ±1 筆無妨，效能提升更重要

## 3️⃣ 商品瀏覽數或點擊統計查詢

用於即時顯示熱門關鍵字、即時點擊統計

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT keyword, COUNT(*) as clicks FROM search_logs
GROUP BY keyword ORDER BY clicks DESC;
```

log 類資料量大、寫入頻繁，不加鎖查詢可避免阻塞

## 4️⃣ 類似商品推薦（基於用戶行為的快取查詢）

推薦引擎查詢中讀取暫時性資料，如最近瀏覽、加購紀錄

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT product_id FROM view_history WHERE user_id = 123 ORDER BY view_time DESC LIMIT 10;
```

這類查詢的準確性影響不大，反而需要速度


## 後台儀表板即時流量監控

顯示當前訂單量、線上人數、瀏覽量等近似統計。

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT COUNT(*) FROM sessions WHERE is_active = true;
```

監控數據可誤差 1～2 秒，不需鎖定

## 8️⃣ 後台資料分析（非即時報表生成）

後台分析如「上週銷售趨勢」、「使用者地區分布」。

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT region, SUM(amount) FROM orders GROUP BY region;
```

這類報表非即時決策，不會造成資料一致性問題

