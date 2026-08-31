---
title: Redis - 專注於低延遲的收費站
date: 2025-08-17 08:53:11
categories: 落葉下的存檔
top_img: https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true
tags:
toc:
toc_number:
comments :
---

{% tabs Cache - 專注於低延遲的收費站%}

<!-- tab Redis-->

想像一條高速公路，如果有太多交流道、紅綠燈、車子搶道，雖然看起來能同時容納很多車，但實際上常常塞爆，而 Redis 的設計是，入口很多 + 收費站只有一個，但每台車過站只要 0.001 秒，除非突然來了一台「超寬超慢的車」，這麼設計是因為，Redis 幾乎只做單純的操作，例如 : 從記憶體拿資料、在記憶體改資料，並且結構非常單純（key → value），他不需要管 `join`、`constraint`、`transaction` 等複雜邏輯，所以它可以保證「每個操作都很短、可預期」

Redis 把所有資料放在記憶體裡，所以車子（請求）一旦進來，就能以極快的速度通過。這種設計讓 Redis 在需要低延遲的場景裡，成為最可靠的快取層。它不追求資料永遠存在（Persistence），而是保證當下的快與穩

<!-- endtab -->

<!-- tab 單執行緒會不會撐不住高併發-->

真正會壓垮 Redis 的不是請求數，而是「單一請求做太多事」，Redis 怕的不是 QPS 高，而是超大的 value、O(n) 操作，這會一次卡住 CPU 幾十毫秒

假設有個很多使用者的電商網站、一台 Redis Server，當使用者很多時表示連線很多

這時同時間可能有幾百、幾千、幾萬個 client 連線到 Redis，這些連線是「同時存在」的，因此 Redis 並不是一次只能接一個人進門

當 request 進來後，Redis 內部會將所有 socket request 透過 event loop（epoll / kqueue）放進「待處理佇列」，此時 request 雖然是`「併發進來」`但還沒`開始執行`

真正關鍵的地方在於，執行指令的時候，Redis 的規則是「任何時間點，只執行一個 command」，因此流程會像這樣

```bash
request A  ──▶ 執行
request B  ──▶ 等
request C  ──▶ 等
```

需要注意每個 command 通常只花幾微秒（µs）所以實際感覺好像是同時被處理，所以在高流量電商中， Redis 還撐得住的原因必須滿足這三件事同時成立

記憶體操作本來就快
沒有鎖
沒有 thread 切換成本
大多數指令是 O(1)

Redis 不是靠「同時跑很多人」，而是靠「每個人都秒過」

<!-- endtab -->

<!-- tab 與其他服務相比-->

Redis 能單執行緒，是因為它「每個請求都極短、極單純、幾乎不會阻塞」，只要其他服務做不到這三件事，就「一定需要多執行緒」!

Redis 刻意限制自己只做單純的事

- 資料在 RAM
- 操作是 GET、SET、INCR、小範圍 list / hash 操作
- 幾乎都是 CPU-bound
- O(1) 或極小 O(n)

所以說 Redis 保證「每個 request 不會卡住 CPU 太久」

<br>

## 一般後端服務

一個「看似簡單」的 API，實際可能在做很多事情

- 驗證 JWT / Session
- 打 Redis
- 打資料庫
- 組資料
- 呼叫第三方金流 / 推播
- JSON 序列化
- 回應 HTTP

這裡面有什麼特性？

❌ 會等 I/O
❌ 等 DB
❌ 等外部 API
❌ 不可控延遲


如果這種服務也用單執行緒，例如有一個 request A → 呼叫第三方金流（等 1000ms），在單執行緒模型下，這 1000ms 內，所有 request 全部卡死，這時候 QPS = 0，使用者全轉圈圈，所以這類服務必須多執行緒 / `async` / `worker pool`，多執行緒的本質是在解決「當一個 request 在等，我還能處理別人。」，但為了做到這件事，代價就是 `lock`、`context switch`、`race condition`等提升複雜度的問題

<br>

## SQL Database

我們用一個 最普通的 SELECT 來拆
```sql
SELECT *
FROM orders
WHERE user_id = 123
AND status = 'PAID'
ORDER BY created_at DESC
LIMIT 20;
```
這一行在 SQL DB 內部就做了這些事

<br>

#### 1️⃣ SQL Parser（先看你在寫什麼）

你 SQL 寫得對不對？語法是否合法？

<br>

#### 2️⃣ Query Optimizer

DB 要決定用哪個 index？index 掃描 or 全表掃描？join 順序？成本估算（cost-based），這一步就不是 O(1)，而且每一筆 query 都可能不同

<br>

#### 3️⃣ Storage Engine（資料在哪）

資料可能在 buffer pool（記憶體）、disk（SSD / HDD），DB 要決定要不要讀磁碟、要不要 preload page，一碰磁碟，延遲直接跳級

<br>

#### 4️⃣ Transaction & Lock

SQL DB 必須處理 ACID、row lock / table lock、MVCC、rollback log / undo log，這些都是 Redis 幾乎不用管的

<br>

#### 5️⃣ 回傳結果前整理

排序、limit、format、network 傳輸

<br>

所以 SQL DB 天生就「不能單執行緒」，如果 SQL DB 也只用一條 thread，會怎樣？

假設：

query A：掃 100 萬筆資料（500ms）
query B：簡單 SELECT（1ms）

在單執行緒下 B 必須等 A 跑完，整個系統體感會直接爆炸，所以 SQL DB 必須多 worker、多 thread、同時跑很多 query，把「慢的」跟「快的」隔離，SQL DB 的價值，正是 Redis 刻意放棄的東西，資料正確性、關聯一致性、Crash recovery、長期保存、複雜查詢能力

「Redis 快，是因為它幾乎不需要做決策；而 SQL DB 慢，是因為它必須每次都做出正確的決策」

<!-- endtab -->

<!-- tab 🍂 併發問題與 Optimistic lock-->

Redis 是單執行緒，所以「每一條指令」是原子性的，但「多條指令之間」仍然可能被插隊，問題真正出在「讀 → 想 → 寫」不是一條指令，例如典型的 Read–Modify–Write 問題

使用者 A
```bash
HGETALL product:123   → 看到 stock = 10
（心裡想：我要扣 1）
HSET stock = 9
```

使用者 B 同時在做
```bash
HGETALL product:123   → 也看到 stock = 10
（心裡想：我要扣 1）
HSET stock = 9
```

```bash
時間 →
A: HGETALL
B: HGETALL
A: HSET stock=9
B: HSET stock=9
```

每一行 都沒有併發，但「整個邏輯」被交錯執行了，所以最後 `lost update`，也就是說單執行緒 ≠ 邏輯不可被打斷

## WATCH + MULTI

WATCH 幫我們處理「讀到的資料是不是已經過期了」的問題

#### 1️⃣ WATCH product:123

「我接下來要根據這筆資料做決策，如果有人動它，請告訴我。」

<br>

#### 2️⃣ HGETALL product:123

你現在拿到的是一個 快照，但 Redis 不保證這個快照之後還是最新的

<br>

#### 3️⃣ 檢查 lastUpdateTs

這是你應用層的版本號，用途就是確保你不是拿舊資料在算

<br>

#### 4️⃣ MULTI

從這一刻開始指令只會排隊，還不會真的執行

<br>

#### 5️⃣ EXEC（關鍵點）

Redis 在這一刻會做檢查「從 WATCH 到現在，product:123 有沒有人動過？」

- ❌ 有 → 整包丟掉
- ✅ 沒有 → 整包一次執行

這裡才是真正的「鎖判斷點」


<!-- endtab -->

<!-- tab 🍂 樂觀鎖-->

Optimistic Lock 假設「大多數時候不會衝突」，只有在真的撞到時才重來，只要確保不擋別人、不鎖資源撞到才放棄，非常適合讀多、寫少，衝突機率低，延遲敏感的場景，Redis Transaction 保證的是「指令要嘛全送進去、要嘛一條都不送」，不是「中間錯了自動回滾狀態」，所以型別錯誤、邏輯錯誤不會幫你回復，這也是為什麼 Redis Transaction 是「工具」，不是「安全網」

<!-- endtab -->

<!-- tab 🍂 我真的需要 WATCH + MULTI 嗎-->

很多情況，答案是 ❌

我們有另一個指令
```bash
HINCRBY product:123 stock -1
```
這已經是原子性、快、不會 lost update，而且還更不容易寫錯

`HINCRBY` 的價值不是「幫你加減數字」，而是把「讀 → 算 → 寫」鎖死成一次不可插隊的行為，我們定位一個 Hash key，例如 product:123，接著定位 Hash 裡的一個欄位例如 stock，當我們在 Redis 內部讀出這個欄位的值，若不存在，當作 0，在同一個執行緒、同一段流程中加減，而 -1 表示扣庫存，此時立刻把結果寫回記憶體，這中途不可能被其他指令插隊，並且回傳更新後的值，最後拿到的是「確定成功後的結果」

假設同時有 100 個使用者，每個人都送這一行，Redis 只會一個一個執行，結果一定是 10 → 9 → 8 → 7 → ...，不可能出現兩個人都把 10 改成 9

<!-- endtab -->

<!-- tab 🍂 多條件一致性-->

讓我們再次上升一個維度，`HINCRBY` 只能處理「單一欄位的數值變動」，無法保證多條件一致性，當我們需要庫存 > 0 AND 狀態 = ON_SALE，就該考慮 Lua Script 或回到資料庫，因為 `HINCRBY` 不能判斷條件，我們如果先 `HGET` 再 `HINCRBY`，就又回到「多指令被插隊」的世界，`HINCRBY` 很像一把「超鋒利的小刀」，但你不能拿它去開瑞士刀等級的需求

## Lua Script

Lua Script 的價值，是讓你把「多步業務邏輯」包裝成「一條不可插隊的指令」，他讓 Redis 保證一個 Lua Script 從頭到尾執行完前，不會插進任何其他指令，但使用起來要小心，雖然 Lua Script 很強，但「慢一次，所有人都等你」。因為 Redis 是單執行緒，Lua 跑太久 = 全世界卡住，腳本不能跑迴圈掃大量資料或做複雜運算，Lua 適合「小而快的原子邏輯」

<!-- endtab -->

<!-- tab 🍂 庫存到底該放 Redis 還是 SQL-->

我們需要考慮的是庫存的「真實來源」與「即時來源」分離，Redis 適合「撐流量」，SQL 適合「負責任」

假設庫存資訊只放 Redis， Redis 一掛掉，資料可能遺失，就很難做審計、回溯，所以這作法只適合活動、限時秒殺、短期資料

再假設每次都打 RDBMS ，高流量時可能會撐不住，因為高併發可能導致鎖爆，TPS 上不去造成使用者體感差，因此純 RDBMS 只適合低流量後台系統

<br>

## 主流做法

| 元件             | 負責什麼       |
| -------------- | ---------- |
| Redis          | 即時扣庫存、擋超賣  |
| SQL DB         | 最終一致性、訂單真相 |
| Background Job | 對帳、回補、修正   |


流程大致上是

- 下單 → Redis（Lua 原子扣庫存）
- 成功 → 建立訂單（SQL），失敗 / 取消 → 補回 Redis
- 定期對帳 Redis vs SQL

我們用 Redis 承受「瞬間不準」，用 SQL 保證「最後一定準」

<!-- endtab -->

{% endtabs %}