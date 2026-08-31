---
title: DynamoDB - 讓流量走散，是最溫柔的防禦
date: 2025-11-06 17:54:05
categories: noSQL
top_img: https://github.com/CHI-KEKE/pics/blob/main/noSQL/tags.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/noSQL/tags.png?raw=true
tags:
    - noSQL
toc:
toc_number:
comments :
---

{% tabs DynamoDB - 讓流量走散，是最溫柔的防禦 %}

<!-- tab DynamoDB 是怎麼把「高併發壓力」攤平的-->

![woodhouse](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/woodhouse.png)

不要硬撐，不要集中，讓壓力自然就是分流，是最溫柔也最有效的防禦

如果把所有時間自我認同都投入單一面向上，一旦單點開始延遲、停滯、壓力襲來，整個人不是失落而是直接瓦解，開始焦慮、開始失去存在感

把自己拆成很多 partition，工作只是其中一塊、關係只是其中一塊，興趣、休息、身體、學習，各自扛一點


某一塊出問題 ≠ 整個人生當機

人生若綁定特定的 hot key ，可能是只從某個人那裡獲得肯定、只用成績證明自己、只靠收入撐安全感、情緒只剩一個出口，那個地方一爆，你會覺得為什麼我明明很努力，卻撐不住？不是你不夠強，而是你把所有流量都導到同一個點

我不需要承受，只要分散

<!-- endtab -->

<!-- tab DynamoDB 是怎麼把「高併發壓力」攤平的-->

DynamoDB 一開始就把資料切碎，讓很多台機器各自扛一小部分，DynamoDB 真正在做的是資料本來就分散在很多 partition，當流量來了會調整 partition 數量與吞吐，這會讓「請求平均打到不同 partition」


假設我有有一張 DynamoDB table，實際上這張 table 被切成很多小塊（partition），而每個  partition 背後是一組儲存 + 計算資源，AWS 在幫你管

📦📦📦📦📦 （每個小盒子就是一個 partition）

每個 partition 大概能扛一定量的 讀取以及一定量的 寫入，單一 partition 被打爆 ≠ 整張表被打爆

request 的分流是靠的是 Partition Key，不同使用者的請求，天生就會打到不同地方，這就是「壓力攤平」的第一層意思。而 DynamoDB 最適合拿來扛這種「大量小讀寫、延遲要穩」的戰場

<!-- endtab -->

<!-- tab 流量真的變很大時，DynamoDB 會做什麼-->

他會偵測某張 table / index 的吞吐需求上升，因此增加 partition 數量，接著重新分配資料到更多 partition，每個 partition 後面都有 AWS 管好的資源，但假如有特定的 hot key，會讓攤平失效，因為全部打在同一個 partition，這時 DynamoDB 沒辦法幫你分流


相比之下，以 RDS 而言，master 寫入是單點、replica 多半只能讀，而 DynamoDB 沒有「主節點」的概念，寫入本身就是分散的

<!-- endtab -->

<!-- tab 購物車-->

購物車只是「暫存狀態」，選擇使用 Redis 是完全合理、甚至更適合的選擇，DynamoDB 並不是一定比 Redis 好，它解的是「不同層級的問題」


#### Redis

Redis 在解的問題是希望快、簡單、短命資料，非常適合 session、暫存購物車、還沒進入「交易流程」的狀態，因為它的一些性質


- key-value
- TTL 天生強
- in-memory，延遲極低


而 Redis 缺點是維護成本可能較高，failover / data loss 成為風險，人還要顧，以成本考量來說，Redis 在付的錢是 EC2 規格（CPU + RAM）、節點數（主 + replica），通常 24 小時都要開著，容量是「預先買好」的，所以就算半夜沒人用，還是在付錢，通常較花錢的地方是為了撐尖峰，要提前開大、記憶體不能超賣、HA / Failover 要多節點，不能像 DDB 一樣用完就縮

#### DynamoDB 呢?

他能做到的是「撐得住尖峰、可預期、可長期保存的狀態資料」，DDB 比 Redis 多的不是快，而是扛超高併發寫入（尤其多 AZ）、資料是 持久化 的、條件寫入 / 交易 / TTL 都在，可以把它當成「不是快取的狀態存儲」

關於 $$ ，DynamoDB 是依據實際讀寫次數付錢，考量資料儲存量、（選擇性）備份、Global Table，因此沒流量 ≈ 幾乎沒成本。DynamoDB 在流量有高低起伏、尖峰很高、離峰很低、request 很碎（很多小寫入），在不想為尖峰預買資源是省錢的選擇，例如活動檔期爆量而平時很少人用，所以 可以用 DDB on-demand 直接撐，而且是 serverless

<!-- endtab -->

<!-- tab 回饋活動-->

資料的特性是 key 明確：promotionId_orderId，會被 MQ job 寫入，後續訂單狀態變更時更新，並且會被稽核時讀取，也就是說單筆資料會被反覆讀/改、不需要 join，不太需要「跨 promotion 任意篩選」

但缺點就是後續維護時，要查某段時間所有即將回饋的訂單，或某 promotion 下所有異常回饋的訂單紀錄就比較麻煩，若要根據其他欄位查詢就需要設 GSI，因為 DynamoDB 不是「搜尋引擎」，它是「分散式 key 定位系統」，在 SQL 的世界裡，資料在同一個 DB、可以掃表、有 index 可以後補，DynamoDB 的世界是

partition A → node A
partition B → node B
partition C → node C

如果說：「幫我找 status=PAID 的資料」，`等於是每個 node 都要掃`，成本、延遲、穩定性都爆炸，所以 DynamoDB 設計時的思維不是資料有哪些欄位，而是我會用什麼 key 查

<!-- endtab -->

<!-- tab 管庫存-->

庫存不是只有「扣減」那麼簡單，還有

- 訂單
- 退貨
- 補貨
- 批次調整
- 人工後台操作
- 跨表關聯
- 財務對帳

`這是一整個交易系統，不是單一 key-value 層級問題`

若 SKU 超熱（熱 key），一堆流程同時改同一筆庫存，還要支援後台各種查詢，這時候
Transaction 成本高、熱點明顯，且 Debug 困難，實際上 還是會讓 SQL 當庫存最終事實（source of truth），而 DynamoDB 做「輔助狀態 / 快速判斷 / 預處理」

<!-- endtab -->

<!-- tab 資料存取模式明確、可預期的系統-->

DynamoDB 最適合那種「你知道要怎麼查資料」的系統。例如：你永遠是用「UserId 查 User 資料」、或是「DeviceId + 時間」去查一段歷史資料
DynamoDB 查資料靠的是 PartitionKey（主鍵查詢），它不擅長做「模糊搜尋」、「多條件查詢」、「JOIN 多表」，所以如果查詢模式很固定，它就能非常快

<!-- endtab -->

<!-- tab 「不適合」用 DynamoDB-->

| 不適合情境                     | 原因                                  |
| ------------------------- | ----------------------------------- |
| 需要複雜 SQL、`JOIN`、`Aggregation` | DynamoDB 是 Key-Value 模型，查詢不靈活       |
| 資料關聯很強（多表關係）              | 沒有外鍵、沒有 JOIN                        |
| 需要強 Transaction（多筆資料一致性）  | 支援有限，效能會下降                          |
| 需要模糊搜尋（例如 `LIKE`、全文檢索）      | DynamoDB 查詢方式固定，要搭配 OpenSearch 才能做到 |

<br>
<br>

#### 🍊 跨條件查詢的限制情境

查同一活動下，所有尚未給予回饋的訂單

```sql
SELECT * 
FROM PromotionReward_Task 
WHERE PromotionId = 'P202511'
  AND RewardStatus = 'WaitingToReward'
```

DDB 只能用 `Partition Key` + `Sort Key` 查（或 `GSI`），若 PK = `PromotionId`、SK = `OrderId`，就只能指定單筆 `OrderId`。你要掃整個活動的所有訂單，只能用 `Query(PromotionId)` → 全量掃描數萬筆，再自己過濾 RewardStatus。

解法是建立 GSI，（Global Secondary Index）：GSI1 PK = `MemberId`, SK = `OrderTimeUTC`，但這會增加寫入成本（每次寫入也更新 GSI）難以維護多維查詢（活動 + 會員 + 狀態）

<!-- endtab -->

{% endtabs %}