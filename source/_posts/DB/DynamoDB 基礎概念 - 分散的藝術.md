---
title: DynamoDB 基礎概念 - 分散的藝術
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

{% tabs DynamoDB 基礎概念 %}

<!-- tab 故事：圖書館的智慧-->

![library](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/woodhouse.png)

想像一座巨大的圖書館，每天有成千上萬的人來借還書。

傳統的圖書館只有一個櫃檯，所有人都要排隊等候同一個館員處理。高峰時段，隊伍排得很長，有人只是想借一本輕小說，卻要等前面的人處理完十幾本厚重的專業書籍。

聰明的館長決定改革：

1. **把圖書館分成多個區域**（Partition）：小說區、科技區、藝術區...每個區都有獨立的櫃檯
2. **每本書都有明確的歸屬**（Partition Key）：根據書的類別編號，自動知道該去哪個櫃檯
3. **每個櫃檯都能獨立工作**（分散式處理）：不會因為科技區很忙，就影響到小說區的借書速度

這就是 DynamoDB 的核心思想：**不要讓所有人擠在同一個地方，把壓力分散開來，每個人都能快速得到服務。**

當某個區域（partition）突然湧入大量讀者，館長不需要叫所有館員都跑去幫忙，只要在那個區域增加一個臨時櫃檯就好。這就是 **Serverless** 和 **Auto Scaling** 的概念。

但如果所有人都只想借《哈利波特》（Hot Key），即使有再多櫃檯也沒用，因為那本書只有一本。這時候館長會怎麼做？買更多複本、設定借閱冷卻時間（Throttling），或是引導讀者去借其他類似的書。

<!-- endtab -->

<!-- tab 📘 RCU & WCU：圖書館的服務能量-->

## 🔢 容量單位的本質

在 DynamoDB 的世界裡，每一次操作都會消耗「能量」，這個能量就是容量單位。

#### 📖 RCU (Read Capacity Unit)

**RCU = 讀取容量單位**，代表「每秒能處理多少次讀取」

| 讀取類型 | 1 RCU 可以讀取的資料量 | 適用情境 |
|---------|---------------------|---------|
| **強一致性讀取** | 4 KB | 需要立即看到最新資料（如：付款後立即查詢餘額） |
| **最終一致性讀取** | 8 KB | 可以接受短暫延遲（如：瀏覽商品列表） |

<br>
<br>

#### 實際案例計算

```plaintext
情境：電商網站的商品瀏覽

假設：
- 每個商品資訊 = 2 KB
- 每秒 1000 個使用者在瀏覽商品
- 使用最終一致性讀取（省一半 RCU）

計算：
1. 每次讀取需要 RCU = 2 KB ÷ 8 KB = 0.25 RCU（向上取整 = 1 RCU）
2. 每秒需要的總 RCU = 1000 次讀取 × 0.25 RCU = 250 RCU

如果改用強一致性讀取：
1. 每次讀取需要 RCU = 2 KB ÷ 4 KB = 0.5 RCU（向上取整 = 1 RCU）
2. 每秒需要的總 RCU = 1000 次讀取 × 0.5 RCU = 500 RCU

結論：使用最終一致性讀取可以省一半的 RCU！
```

<br>
<br>

#### ✍️ WCU (Write Capacity Unit)

**WCU = 寫入容量單位**，代表「每秒能處理多少次寫入」

| 操作類型 | 1 WCU 可以寫入的資料量 |
|---------|---------------------|
| **標準寫入** | 1 KB |
| **交易寫入** | 1 KB（但消耗 2 倍 WCU） |

<br>
<br>

#### 實際案例計算

```plaintext
情境：促銷活動的訂單寫入

假設：
- 每筆訂單資料 = 3 KB
- 促銷期間每秒產生 500 筆訂單

計算：
1. 每筆訂單需要 WCU = 3 KB ÷ 1 KB = 3 WCU
2. 每秒需要的總 WCU = 500 筆 × 3 WCU = 1500 WCU

如果需要交易寫入（確保強一致性）：
1. 每筆訂單需要 WCU = 3 × 2 = 6 WCU
2. 每秒需要的總 WCU = 500 筆 × 6 WCU = 3000 WCU
```

<!-- endtab -->

<!-- tab ☁️ Serverless：你不用管伺服器-->

**Serverless ≠ 沒有伺服器**  而是 **你不需要管理伺服器**

<div style="display: flex; gap: 40px;"><div>

**傳統 MySQL/PostgreSQL**

你需要處理：
- ❌ 選擇 EC2 規格（CPU、RAM）
- ❌ 設定磁碟空間
- ❌ 安裝資料庫軟體
- ❌ 設定備份機制
- ❌ 監控 CPU/記憶體使用率
- ❌ 手動擴容（停機風險）
- ❌ 作業系統安全修補
- ❌ 資料庫版本升級

</div>
<div>

**DynamoDB (Serverless)**

AWS 幫你處理一切：
- ✅ 自動分配計算資源
- ✅ 自動擴展儲存空間
- ✅ 自動備份與復原
- ✅ 自動監控與警報
- ✅ 無需停機擴容
- ✅ 自動安全修補
- ✅ 零維護成本

</div>
</div>

#### 活動促銷系統

雙11大促銷，流量從平時的 100 QPS 暴增到 10,000 QPS

**RDS**

// 1. 需要提前一週準備擴容計畫
// 2. 活動前一天晚上停機擴容（風險高）
// 3. 活動結束後，資源閒置但仍需付費
// 4. 如果預估錯誤，可能當機或資源浪費

- EC2 r5.2xlarge (8 vCPU, 64 GB RAM): $0.504/小時
- 每月成本 = $0.504 × 24 × 30 = $362.88
- 儲存空間 1 TB SSD: $115/月
- 備份與快照: $50/月
- 總計: ~$528/月（固定成本，無論流量多少）

**DynamoDB**

完全不需要擔心容量，DynamoDB 自動根據流量擴容，流量自動分散到多個 partition

DynamoDB (On-Demand)：
- 平常流量（低峰）:
  - 每月 100 萬次讀取 × $0.25/百萬 = $0.25
  - 每月 50 萬次寫入 × $1.25/百萬 = $0.625
  - 儲存 100 GB × $0.25/GB = $25
  - 總計: ~$26/月

- 促銷高峰（一個月 3 天）:
  - 峰值期間成本 ~$150
  - 其他時間成本 ~$26
  - 總計: ~$176/月

<!-- endtab -->

<!-- tab 🚦 Throttling：被限流了怎麼辦-->

#### Throttling = 節流 = 被拒絕服務

當你的請求超過 DynamoDB 的容量限制時，AWS 會暫時拒絕你的請求，並回傳錯誤：

```plaintext
ProvisionedThroughputExceededException: 
You exceeded your maximum allowed provisioned throughput for a table or for one or more global secondary indexes.
```

| 原因 | 說明 | 範例情境 |
|------|------|---------|
| **容量不足** | Provisioned 模式下 WCU/RCU 設定太低 | 促銷活動流量暴增，但只設定 100 WCU |
| **Hot Partition** | 大量請求集中在同一個 Partition Key | 所有使用者都在搶同一個限量商品 |
| **Burst Capacity 耗盡** | 短時間內突發流量過大 | 秒殺活動開始的瞬間 |
| **GSI 容量不足** | Global Secondary Index 的容量比主表低 | 主表夠用，但查詢 GSI 被限流 |

<br>
<br>

- 使用 Exponential Backoff 重試
- 避免 Hot Partition
- 使用 DynamoDB Accelerator (DAX,  DynamoDB 的記憶體快取)

<!-- endtab -->

<!-- tab 🔑 Partition Key：資料的身分證-->

**Partition Key = 決定資料存放位置的關鍵**

在 DynamoDB 中，每筆資料都必須有一個 Partition Key（也叫 Hash Key），這個 Key 決定了：
1. 資料會被存在哪個 partition
2. 查詢時如何快速定位資料
3. 流量如何分散

```plaintext
你的 Table
    ↓
【Partition 1】  【Partition 2】  【Partition 3】  【Partition 4】
 UserId: 001      UserId: 002      UserId: 003      UserId: 004
 UserId: 005      UserId: 006      UserId: 007      UserId: 008
 UserId: 009      UserId: 010      UserId: 011      UserId: 012
    ↓                ↓                ↓                ↓
 不同的實體機器    不同的實體機器    不同的實體機器    不同的實體機器
```

DynamoDB對 Partition Key 做 Hash、將 Hash 值轉換成數字、模除 partition 總數，得到目標 

<br>
<br>

#### ✅ 好的 Partition Key

| 原則 | 說明 | 範例 |
|------|------|------|
| **高基數（High Cardinality）** | 有很多不同的值 | ✅ UserId (百萬個不同值)<br>❌ Status (只有 3 個值：Active/Inactive/Deleted) |
| **均勻分佈** | 存取頻率平均 | ✅ OrderId (每筆訂單都不同)<br>❌ Date (今天的訂單都集中在同一天) |
| **符合查詢模式** | 你最常用的查詢欄位 | ✅ 如果常查「某使用者的訂單」→ UserId 當 PK<br>❌ 如果常查「某時間區間的訂單」→ UserId 不適合 |

<br>
<br>

#### ❌ 糟糕的 Partition Key 範例

使用固定值，所有資料都有相同的 Partition Key，後果是所有資料都擠在同一個 partition，完全無法發揮 DynamoDB 的分散式優勢

<!-- endtab -->

{% endtabs %}