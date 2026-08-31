---
title: TempTable
date: 2025-10-06 22:52:05
categories: 情報の存在論
top_img: https://i.imgur.com/NfCnwwU.png
cover : https://i.imgur.com/NfCnwwU.png
tags:
    - 情報の存在論
toc:
toc_number:
comments :
---

{% tabs TempTable%}

<!-- tab #temp（Local Temporary Table）-->

#temp 的本質，是把「一次查不完、邏輯又複雜的中間結果」變成一個可以被優化、重用、逐步加工的暫存資料集，讓 SQL Optimizer 有機會幫你把事情做對


![temp](https://github.com/CHI-KEKE/pics/blob/main/SQL/tempTable/Why_temptable.png?raw=true)


- 建立與隔離：在 `tempdb` 中開闢一塊專屬空間，僅限當前連線（Session）存取，確保資料不會跟別人的請求打架
- 資料灌入：利用 `SELECT INTO` 或 `INSERT INTO` 將中間運算結果存入
- 優化加速：視需求針對 `#temp` 建立索引（`Index`），這在處理萬筆以上資料時非常關
- 多次加工：在同一個連線內，後續的 Stored Procedure 或 SQL 指令可以反覆讀取、更新這份暫存資料。
- 生命週期終結：當連線關閉時自動銷毀，保持資料庫環境整潔


![aa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/tempTable/%23temp.png)


```SQL
CREATE TABLE #TEMP(
  NAME CHAR(20),
  ID   CHAR(10)
) 
```

## 電商雙 11 促銷結帳

想像我們要幫上萬名會員計算購物車折扣

- 第一步：先撈出所有符合活動資格的訂單到 #OrdersToBill
- 第二步：將這份名單去串接複雜的「會員分級表」與「階梯折扣表」，算出最終折扣並存入 #CustomerDiscount
- 第三步：最後才把這些算好的結果吐給前端顯示

```sql
-- Step 1: 篩出候選訂單
SELECT o.OrderId, o.CustomerId, o.TotalAmount
INTO #OrdersToBill
FROM dbo.Orders o
WHERE o.Status = 'ReadyToBill';

-- Step 2: 跟促銷規則做 Join 後彙整
SELECT ob.CustomerId, SUM(ob.TotalAmount * p.DiscountRate) AS DiscountTotal
INTO #CustomerDiscount
FROM #OrdersToBill ob
JOIN dbo.Promotions p ON p.RuleId = ob.PromoRuleId
GROUP BY ob.CustomerId;

-- Step 3: 輸出結果
SELECT * FROM #CustomerDiscount;

-- 用完可手動清理（選擇性）
DROP TABLE #CustomerDiscount, #OrdersToBill;
```

這批資料只對「這次報表 / 這次計算」有意義、下一個使用者跑一樣的報表，資料內容一定不同，而 @table variable 的問題是沒有（或非常弱的）統計資訊，Optimizer 常假設「大概只有 1 筆資料吧 🤷」


![temp](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/tempTable/%23temp_case.png)


## 📊 統計資訊 (Statistics) 與執行計畫

這是 #temp 勝過表變數 (@table) 的殺手鐧，它會主動告訴資料庫：「我這裡有十萬筆資料，請認真對待」。若糾結要用 @table 還是 #temp。簡單來說，@table 適合資料量極小（幾百筆以內）的情況；但當資料量大時，資料庫會誤判 @table 只有 1 筆資料，導致查詢變超慢。#temp 則擁有完整的統計資訊，能讓 SQL 引擎選對 Join 的方式


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/tempTable/%23temp_statistics.png)


## 🛠️ 索引 (Index) 的彈性

如果打算對這份暫存資料進行多次搜尋或 Join，記得幫它「開外掛」加索引。#temp 雖然是暫存的，但它在本質上就是一張「表」。如果在一個 Session 裡要對這張表做多次過濾，建立索引能讓原本要掃描全表的動作變成精準定位，這對於複雜報表的效能提升非常有感


![i](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/tempTable/%23temp_index.png)


## 🧹 手動清理 (Explicit Drop) 的好習慣

雖然系統會幫收尾，但「用完即丟」是工程師優雅的展現，也能減輕 tempdb 的負擔。在長連線（Connection Pooling）的機制下，雖然 Session 可能沒關閉，但邏輯已經跑完。手動執行 DROP TABLE #TEMP 可以立刻釋放 tempdb 的空間與記憶體資源，避免在高併發環境下讓暫存資料庫塞車


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/tempTable/%23temp_manual_clean.png)


有時過度使用暫存表會導致 IO 壓力過大（因為資料必須寫入磁碟上的 tempdb）。在某些情況下，使用 CTE (Common Table Expressions) 或是優化過的 Subquery 可能會更快，因為它們在記憶體中運算的可能性更高

<!-- endtab -->

<!-- tab ##temp（Global Temporary Table）-->

全域暫存表 (Global Temporary Table, ##temp) 的特性。與區域暫存表 (#temp) 相比，它最大的不同在於「共享性」與「生命週期」，可以把 ##temp 想像成一個公共的臨時佈告欄 📋：任何路過的人（不同的連線 Session）都能看到上面的內容並進行操作，直到最後一個關注這個佈告欄的人離開為止

Global Temporary Table 用意是建立一個跨連線的臨時資料共享中心，讓不同的處理程序能像接力賽一樣傳遞數據。在複雜的系統架構中，有時 A 程序算完的結果需要立刻給 B 程序接手（例如不同的排程或非同步的查詢），這時 ##temp 就是最簡單的橋樑，讓你不需要真的去實體資料庫建一張永久表

- 在「同一個 SQL Server 實例」中所有連線都可見（跨資料庫可用，但不能跨伺服器）；存在 tempdb；當建立它的那個 Session 結束且沒有其他 Session 使用時，才會被刪除
- ##Table是全域的資料表，所有人均可取用(其他資料庫也可使用)
- 名稱務必唯一（加日期與隨機碼），避免彼此覆蓋
- 要有清理機制（或控制生命週期），不然容易造成資料錯讀或安全風險


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/tempTable/%23%23temp.png)


```SQL
CREATE TABLE ##TEMP(
  NAME CHAR(20),
  ID   CHAR(10)
)
```

## 自動化物流排程

```sql
-- Session A：產出結果，給其他工作讀
SELECT OrderId, ShipDate
INTO ##ShippingToday_20251006_0D6F
FROM dbo.Shipments
WHERE CAST(ShipDate AS date) = '2025-10-06';

-- Session B（甚至是批次排程、另一個 DBA 的 SSMS）：可讀取
SELECT * FROM ##ShippingToday_20251006_0D6F;

-- 當 A 結束且 B 不再使用，這張表就會被清掉
```


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/tempTable/%23%23temp_case.png)


## 🛡️ 命名衝突與覆蓋風險

因為 ##temp 是全域的，如果兩個人都寫 CREATE TABLE ##MyData，第二個人就會失敗。實務上我們必須像範例中提到的，加上日期與隨機碼 _20251006_0D6F，這不是為了美觀，而是為了生存（系統穩定性）


## 安全性與權限漏洞

##temp 的權限相對鬆散，同一個伺服器上的其他資料庫使用者也可能讀到內容。如果資料包含客戶個資或交易細節，這會是一個潛在的安全破口


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/tempTable/%23%23temp_risk.png)


<!-- endtab -->

<!-- tab @t（Table Variable）-->

- 是變數，用 DECLARE @t TABLE(...) 宣告
- 作用域只限於目前的 Batch/Stored Procedure/Function 生命週期到該作用域結束即自動回收
- 底層也會用到 tempdb（不是純記憶體），小量資料多半在記憶體中，量大一樣可能寫到磁碟
- 小型清單、快速過濾、參數化傳遞
- 只是幾十到幾百筆的小清單，語意像參數；作用域自然結束就回收，不用 DROP
- 若變大且邏輯複雜，建議改 #temp 讓優化器有更好的統計
- 從 .NET 傳大清單可用 Table-Valued Parameter (TVP)，在 SQL 端再配合 #temp 做複雜處理，是電商常見高效組合


![p](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/tempTable/%40table.png)


## 秒殺／限時搶購（Flash Sale）名單過濾

從應用程式傳入少量候選 SkuId（數十筆以內），需要即時比對庫存與價格做快速檢查

```sql
DECLARE @SkuList TABLE (SkuId INT PRIMARY KEY);
-- 應用程式批次傳入候選 SKU
INSERT INTO @SkuList VALUES (101),(102),(103);

SELECT s.SkuId, p.Price, i.Stock
FROM @SkuList s
JOIN ProductPrice p ON s.SkuId = p.SkuId
JOIN Inventory i ON s.SkuId = i.SkuId;
```

- 資料量小，作用域只在該批請求內
- 不需建立索引或統計
- 系統會在執行結束時自動清除，乾淨又快速


## 黑名單／白名單過濾（高頻小清單）

像是活動可用的 CouponCode、或風控黑名單使用者 ID。這些清單通常只有數十～數百筆，每次檢查時由應用程式傳入
```sql
DECLARE @BlackList TABLE (UserId INT);
INSERT INTO @BlackList VALUES (101,105,108);

SELECT u.UserId, u.LoginTime
FROM Users u
WHERE u.UserId IN (SELECT UserId FROM @BlackList);
```

![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/tempTable/%40table_case.png)


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/tempTable/%40table_risk.png)


<!-- endtab -->

<!-- tab TVP (Table-Valued Parameters)-->

TVP 的本質，是讓應用程式「一次傳一整組結構化資料給 SQL」，而不是硬把集合塞成字串再請資料庫自己想辦法拆，TVP 做的事很就是直接把「表格」當成參數傳進來


流程會是這樣，Application 組好一張表 → 當參數傳給 SQL → SQL 當成正常資料表用


## Step 1：先在 SQL Server 定義「表的型別」

這不是 Table，是 Table 的「型別」，專門拿來當參數用，就像 C# 裡的 class / struct 定義

```sql
CREATE TYPE dbo.IdListType AS TABLE
(
    Id BIGINT
);
```

## Step 2：Stored Procedure 接收 TVP

TVP 一定要 READONLY，在 SQL 裡它看起來、用起來、Join 起來，都跟一張表一樣

```sql
CREATE PROCEDURE dbo.usp_GetOrdersByIds
    @Ids dbo.IdListType READONLY
AS
BEGIN
    SELECT o.*
    FROM Orders o
    JOIN @Ids i ON o.OrderId = i.Id;
END
```

## Step 3：應用程式端怎麼傳（概念）

以 .NET 為例，要做的事就是建一個 DataTable、塞進 Id 清單、當參數丟給 Stored Procedure，他不是字串、不是 JSON，是真的一張表



## 商品搜尋 / 訂單批次處理

前端一次選 5000 個商品，不可能用 IN (...)、不想用字串 Split，正確姿勢是

Application 組 TVP-> SQL 接收 TVP -> 轉存 #temp -> 加 index -> 開始算折扣、排序、排名

![t](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/tempTable/TVP.png)

<!-- endtab -->

{% endtabs %}
