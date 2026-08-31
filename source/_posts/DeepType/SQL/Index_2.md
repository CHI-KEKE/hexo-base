---
title: Index - 發揮作用...了嗎
date: 2025-10-12 13:51:05
categories: 情報の存在論
top_img: https://i.imgur.com/NfCnwwU.png
cover : https://i.imgur.com/NfCnwwU.png
tags:
    - 情報の存在論
toc:
toc_number:
comments :
---

{% btn 'https://medium.com/@henry-chou/5%E5%80%8B%E6%A6%82%E5%BF%B5%E6%8B%AF%E6%95%91%E9%BE%9C%E9%80%9F%E7%9A%84sql%E6%9F%A5%E8%A9%A2-query-d10e06b8bae3',5個概念拯救龜速的SQL查詢(Query),far fa-hand-point-right %}

<br>

{% tabs Index2 %}


![P](https://github.com/CHI-KEKE/pics/blob/main/SQL/Index/landing.png?raw=true)


<!-- tab 索引沒有覆蓋（Covering Index)-->

索引是「查詢條件、資料分佈、索引結構三件事剛好對齊時，Optimizer 才會願意用它」，假設我們今天建立了一個 Index
，我們以為它可以幫忙「快速定位資料」，Optimizer 看查詢條件會嘗試判斷能不能用這個 Index 做 Seek、用了之後要不要回主表（Key Lookup）、成本是不是比 Scan 還低，Optimizer 依據 Statistics 估算回傳列數，不是實際跑，而是「猜」，如果猜錯，整個執行計畫就歪掉，只要任何一環不理想、Index 欄位不夠、欄位順序不對、資料分佈太平均、統計過期，則 Optimizer 就會放棄你以為「應該要用」的索引

因此會發生已經建好索引，而且查詢條件也寫得很明確，但實際發生的是出現 Key Lookup、Index Scan / Table Scan、查詢隨資料量成長而變慢，因為照你的設計用索引不划算!


![G](https://github.com/CHI-KEKE/pics/blob/main/SQL/Index/1___index_is_about_cost.png?raw=true)


<!-- endtab -->


<!-- tab 索引沒有覆蓋查詢（Covering Index）-->

事情已經發生到「你就在現場」了，卻因為少帶一樣東西，只好再跑一趟

你已經請好假、準時到機場、行李也託運了，輪到你登機...「護照呢？」

在托運行李裡 😇😇😇

![baggage](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/baggage2.png)

<br>

索引也一樣，他只能回答它「帶在身上的資訊」，缺的欄位一定得回主表補，回表一多，索引的優勢就被吃光，Covering 是把「查詢會用到的資料一次拿齊」，讓 Seek 結束就真的結束

```sql
CREATE INDEX IX_Trip_Checkin
ON Trip(DepartureDate, Destination, Airline);

SELECT PassengerName, SeatNo, BoardingGroup
FROM Trip
WHERE DepartureDate = '2025-12-01'
  AND Destination = 'Tokyo'
  AND Airline = 'ANA'
ORDER BY BoardingGroup DESC;
```

這個索引裡 沒有包含 BoardingGroup 欄位。所以 SQL Server 雖然可以用這個索引快速找到
「今天、飛東京、搭 ANA 的旅客」，但它還得 回主表 (Key Lookup)去拿 BoardingGroup 因此造成大量的隨機 I/O

雖然有索引 + Key Lookup 還是比全表掃描快很多（特別是資料量大時），只是當「命中筆數變多」，Lookup 成本會開始反咬你


![G](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/2___keylookup.png)


![C](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/2____keylookup___problem.png)


## ✅ 解法：改成「Covering Index」

```sql
CREATE INDEX IX_Trip_Checkin_Cover
ON Trip(DepartureDate, Destination, Airline)
INCLUDE (BoardingGroup, PassengerName, SeatNo);
```

SQL Server 不用回主表，就像你護照就在口袋裡，輪到你，直接登機!

而 Covering Index 是用空間與維護成本，去換「某些查詢的穩定速度」，因此每多 INCLUDE 一個欄位，表示索引頁變大、INSERT / UPDATE / DELETE 更慢、Buffer Pool 壓力變高，因此 Covering Index 只該為「高頻、痛點查詢」服務


![F](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/2___convering__index.pn)


<!-- endtab -->

<!-- tab 複合索引的順序不對（Leftmost Prefix 原則)-->


一本很厚的參考書，它的編排方式是

1️⃣ 先依語言分章（Language）
2️⃣ 每一章裡再依 類別分節（Category）
3️⃣ 每一節裡，內容再依 時間排序（EventTime）

這本書是實體印刷的，頁碼是固定的，你只能這樣找，先翻到「中文」那一章，再在該章裡找「體育」這一節，最後在這一節裡，依時間往前或往後翻，這樣，每一次翻頁，搜尋範圍都會快速縮小。在 SQL Server 裡，複合索引 也是有順序的。這順序會直接影響查詢能不能用上索引


![F](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/3___book__is_sequence.png)


複合索引就像一本只能「照目錄順序翻」的書，只要跳過最前面的分類，後面的資訊再齊全也派不上用場

Leftmost Prefix Rule 說的是書就是這樣排的，你只能照它的目錄結構來找，一旦你跳過最前面的分類（Language），書沒有任何線索告訴你該從哪一頁開始，就只能從頭一頁一頁翻，搜尋自然就退化成「掃描整本書」

```sql
CREATE INDEX IX_EventCategory
ON EventCategory(Language, Category, EventTime);
```

能有效支援的條件是：
| 查詢條件                                                        | 能用到索引嗎？              |
| ----------------------------------------------------------- | -------------------- |
| WHERE Language = ...                                        | ✅ 可以                 |
| WHERE Language = ... AND Category = ...                     | ✅ 可以                 |
| WHERE Language = ... AND Category = ... AND EventTime = ... | ✅ 可以                 |
| WHERE Category = ... AND EventTime = ...                    | ❌ 不行（漏掉最左邊 Language） |
| WHERE Language = ... AND EventTime = ...                    | ⚠️ 部分使用（可能效能不佳）      |


![H](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/3__index__broke.png)


![KK](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/3___sequence__rule.png)


## 情況一：完全沒照順序（漏掉最左邊）

```sql
WHERE Category = 'Sport'
```

「我知道你要找『體育』，但我整本書裡每一種語言都有體育，你又沒告訴我從哪一章開始……」

結果導致不能 Index Seek、不能快速定位起點、退化成 Index Scan（幾乎等於沒用索引），這種情況，實務上可以把它當作「沒用索引」來看

## 情況二：有用到最左邊，但中間斷掉

```sql
WHERE Language = 'zh-TW' AND EventTime >= '2025-01-01'
```

索引其實還是能幫一點忙，可以先跳到 Language = 'zh-TW' 那一整段，但在這一段裡，資料是先依 Category 排，再依 EventTime，所以 EventTime 不能再精準 Seek，結果通常是 Seek + Scan（掃 zh-TW 這一整段），或看起來像 Seek，但實際 I/O 還是不小，不是沒用索引，而是「只用了一半」

沒照順序，但有用到最左邊欄位，還算有用索引，但效果打折，連最左邊欄位都沒用到，幾乎等於沒用索引（Index Scan / Table Scan）


<!-- endtab -->

<!-- tab 索引欄位的「資料分佈」不好（Cardinality）-->

「我很努力」「我很認真」「我很善良」

這些就像 IsActive = 1，幾乎每個人都有，所以選擇性太低，世界不會因此特別為你走 Index Seek，這不代表它不重要，而是它不足以讓你被快速選中，屬於低基數

而真正讓人生加速的，是高選擇性那一欄例如

- 你獨有的經驗
- 你不可替代的視角
- 你能直接解決的問題

![C](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/4___i_am_kind.png)


Cardinality（基數）就是「某個欄位有多少種不同的值」的程度

- IsActive 只有 0/1 兩種 → 低 Cardinality
- UserId 幾乎每筆都不同 → 高 Cardinality

Query Optimizer 要決定走哪個 Index（索引）時，需要先估計「這個條件會篩出多少列」。這個估計，主要靠 Statistics（統計資料）。如果 Statistics 過期或不準導致估計錯，則可能選錯索引或選錯執行計畫，導致變慢，而Optimizer 透過 Statistics 來估 Cardinality，如果估錯，就可能選錯索引


## 💠 低 Cardinality 欄位（例如 IsActive）

```sql
SELECT *
FROM EventCategory
WHERE IsActive = 1;
```

IsActive 只有 0/1，選擇性（Selectivity）低。如果大多數資料都是 IsActive = 1（例如 95% 都是 1），即使有 IsActive 的索引，也不一定會使用 Index Seek，因為掃幾乎整張表跟用索引再回表（Key Lookup）成本差不多甚至更高。這種情況，Optimzer 常會選 Index Scan 或 Table Scan，這不是「壞」，只是比較符合成本模型



## 高 Cardinality 欄位（例如 UserId）

```sql
SELECT *
FROM EventCategory
WHERE UserId = @userId AND IsActive = 1;
```

UserId 幾乎唯一，選擇性高 → 很適合走 Index Seek。但如果 Statistics 過舊（例如剛大量匯入/刪除資料，或資料分佈變化很大），Optimizer 就可能低估或高估回傳列數，導致選錯索引（例如選擇掃描一個大索引）

改善就是先把統計更新到新鮮的狀態
```sql
-- 精準但較慢（完整掃描）
UPDATE STATISTICS dbo.EventCategory WITH FULLSCAN;

-- 一般快速更新
UPDATE STATISTICS dbo.EventCategory;

-- 全資料庫批次更新比較省事
EXEC sp_updatestats;
```

## Filtered Index

若你只會查活躍資料，考慮 Filtered Index，索引鍵只有一個欄位：SomeSelectiveColumn，這是唯一的 Index Key Column。WHERE IsActive = 1 是「篩選條件」，不是索引鍵，Filtered Index 的意思是只有 IsActive = 1 的資料列會被包含進這個索引，其他資料列（IsActive = 0 或 NULL）全部忽略，這不會讓 IsActive 成為索引的一部分，它只是一個 filter predicate。

```sql
CREATE INDEX IX_EventCategory_IsActive_1
ON dbo.EventCategory(SomeSelectiveColumn)  -- 可放你常查的更具選擇性的欄位
WHERE IsActive = 1;
```

這樣針對 IsActive = 1 的查詢，索引更小、更精準


若還有其他選擇性高的條件，如 CategoryId 或 UpdatedAt，把選擇性高的欄位放在複合索引（Composite Index）的前面
```sql
CREATE INDEX IX_EventCategory_CategoryId_IsActive
ON dbo.EventCategory(CategoryId, IsActive);
```


![C](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/4___filtered__index.png)


<!-- endtab -->

<!-- tab Equality 與 Range 的順序-->

若條件一個是等號（=），一個是範圍（BETWEEN、>、<、LIKE 'x%'），把等號欄位放前面，再放範圍欄位，Seek 能力最好。

```sql
-- 常見條件：Category = 'Sport' AND PublishedAt >= @d
CREATE INDEX IX_Articles_Category_PublishedAt
ON dbo.Articles(Category, PublishedAt);
```


![G](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/4____cardio__order.png)


<!-- endtab -->

<!-- tab Optimizer-->


![GG](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/5___optimizer.png)


![k](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/5___optimizer__refresh.png)

<!-- endtab -->

<!-- tab summary-->

![t](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/6___table.png)


![p](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/6___final.png)


![sum](https://github.com/CHI-KEKE/pics/blob/main/SQL/Index/unnamed.png?raw=true)

<!-- endtab -->

{% endtabs %}
