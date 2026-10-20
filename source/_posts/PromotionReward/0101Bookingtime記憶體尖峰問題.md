---
title: 凌晨0101的記憶體尖峰問題
date: 2026-10-20 11:10:00
categories: PromotionReward
top_img: https://pub-d7e550ef212547d888a6e01348459946.r2.dev/ecom/promotion/promotionreward.png
cover : https://pub-d7e550ef212547d888a6e01348459946.r2.dev/ecom/promotion/promotionreward.png
toc:
toc_number:
comments :
tags:
---

{% tabs 凌晨0101的記憶體尖峰問題 %}

<!-- tab 問題現象與根因 -->

<div class="pr-flow">

<style>
.pr-flow{--pr-ink:#493b2d;--pr-muted:#71624e;--pr-line:#e4d8bf;--pr-panel:#fbf7ef;--pr-accent:#57753f;--pr-bear:#a06a41;--pr-diagram-bg:#fffdf7;--pr-danger:#bb765f;--pr-danger-bg:#f9e9e3;color:var(--pr-ink);line-height:1.9;overflow-wrap:anywhere}
.pr-flow *{box-sizing:border-box}
.pr-flow h2,.pr-flow h3,.pr-flow h4{color:var(--pr-ink)}
.pr-flow h2{margin-top:2.4em;margin-bottom:1em;font-size:1.28em}
.pr-flow h3{margin-top:2.2em;margin-bottom:1em;font-size:1.1em;border-bottom:1px dashed var(--pr-line);padding-bottom:10px}
.pr-lead{position:relative;overflow:hidden;padding:28px 72px 28px 26px;border-radius:24px;background:radial-gradient(circle at 12% 10%,#ffffff12,transparent 50%),linear-gradient(150deg,#6f9250,#547a3d 60%,#3f5f2e);color:#fffbed;margin:20px 0 30px;box-shadow:0 6px 18px #59462212}
.pr-lead::before{content:'⏱️';position:absolute;right:22px;top:14px;font-size:20px;opacity:.78}
.pr-lead::after{content:'🐻';position:absolute;right:18px;bottom:15px;font-size:32px}
.pr-lead p{margin:0;font-size:1em;line-height:1.9}
.pr-figure{border:1px solid var(--pr-line);border-radius:20px;overflow:hidden;margin:24px 0;background:var(--pr-diagram-bg)}
.pr-diagram{padding:24px 16px;overflow-x:auto;background:var(--pr-diagram-bg)}
.pr-diagram .mermaid-wrap{margin:0}
.pr-diagram svg{display:block;margin:auto;width:100%;min-width:620px;max-width:none!important;height:auto}
.pr-diagram.pr-wide svg{min-width:900px}
.pr-diagram.pr-compact svg{min-width:420px;max-width:720px!important}
.pr-diagram:focus-visible{outline:2px solid var(--pr-accent);outline-offset:-3px}
.pr-figure figcaption{background:var(--pr-panel);color:var(--pr-muted);padding:14px 18px;font-size:13px;border-top:1px solid var(--pr-line)}
#article-container .pr-flow code{overflow-wrap:anywhere;color:var(--pr-bear);background:var(--pr-panel)}
.pr-flow table{font-size:14px;width:auto;max-width:100%}
.pr-flow th{background:var(--pr-panel);color:var(--pr-ink)}
.pr-flow th,.pr-flow td{white-space:normal;overflow-wrap:anywhere;border-color:var(--pr-line);padding:12px}
.pr-note{border-left:4px solid var(--pr-bear);border-radius:0 16px 16px 0;padding:18px 20px;background:var(--pr-panel);margin:24px 0}
.pr-note.pr-danger{border-left-color:var(--pr-danger);background:var(--pr-danger-bg)}
.pr-note.pr-success{border-left-color:var(--pr-accent);background:#edf3e4}
.pr-evidence{border:1px solid var(--pr-line);border-radius:16px;padding:16px 20px;margin:24px 0;background:var(--pr-panel)}
.pr-evidence summary{cursor:pointer;color:var(--pr-accent);font-weight:700}
.pr-evidence[open] summary{margin-bottom:18px}
.pr-record-grid{display:grid;grid-template-columns:repeat(3,minmax(0,1fr));gap:14px;margin:22px 0}
.pr-record{border:1px solid var(--pr-line);border-radius:18px;background:var(--pr-panel);padding:18px 20px;box-shadow:0 5px 18px #59462208}
.pr-record strong{display:block;color:var(--pr-ink);margin-bottom:8px}
.pr-record p{margin:0;color:var(--pr-muted)}
[data-theme="dark"] .pr-flow{--pr-ink:#ece5d6;--pr-muted:#c6bca8;--pr-line:#575744;--pr-panel:#2b3025;--pr-accent:#b4cd92;--pr-bear:#e1b68c;--pr-diagram-bg:#242a20;--pr-danger:#db967f;--pr-danger-bg:#3b2925}
[data-theme="dark"] .pr-lead{background:linear-gradient(150deg,#4b6538,#354d29)}
[data-theme="dark"] .pr-diagram .cluster rect{fill:#343d2d!important;stroke:#8e9f78!important}
[data-theme="dark"] .pr-diagram .cluster-label span{color:#ece5d6!important}
[data-theme="dark"] .pr-note.pr-success{background:#293525}
@media(max-width:600px){.pr-lead{padding:22px 52px 22px 18px}.pr-lead::after{font-size:25px;right:12px}.pr-lead::before{right:16px;font-size:16px}.pr-note,.pr-evidence{padding:15px}.pr-diagram{padding:18px 10px}.pr-flow th,.pr-flow td{padding:9px 7px;font-size:13px}.pr-record-grid{grid-template-columns:1fr}}
</style>

<div class="pr-lead">
<p>Promotion Worker 每天凌晨 01:01 都會迎來一波集中的回饋派工。問題不在 CronJob 突然多跑，而是大量延後回饋紀錄被排到同一個整點，Dispatcher 又在同一輪查詢、分組、上傳 S3 並建立 NMQ Task，因而形成固定時刻的記憶體尖峰。</p>
</div>

## 尖峰如何形成

<figure class="pr-figure"><div class="pr-diagram pr-compact" tabindex="0" role="region" aria-label="凌晨一點零一分記憶體尖峰形成時間軸">
{% mermaid %}
flowchart LR
    Orders["多筆訂單命中延後回饋"] --> Normalize["BookingTime 集中到整點"]
    Normalize --> Wait["DDB 累積 WaitToReward"]
    Wait --> Cron["01:01 CronJob 啟動"]
    Cron --> Dispatch["同一輪查詢與建立大量 Task"]
    Dispatch --> Peak["Worker 記憶體形成尖峰"]
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef waiting fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef danger fill:#f9e9e3,stroke:#bb765f,color:#704737
    class Orders,Normalize service
    class Wait,Cron waiting
    class Dispatch,Peak danger
{% endmermaid %}
</div><figcaption>固定出現在 01:01 的尖峰，與前一小時查詢窗口及整點 BookingTime 聚集相互對應。</figcaption></figure>

根因可拆成三層。第一層是 `RewardDays > 0` 的計算會把時間收斂到日期整點。第二層是許多紀錄因此共用相同的 `BookingTime`。第三層是 Dispatcher 會先為所有分組準備 S3 Payload 與 Task Request，再逐一送往 NMQ，單次批量越大，處理期間保留的物件也越多。

<div class="pr-record-grid" role="group" aria-label="尖峰問題的三層原因">
<div class="pr-record"><strong>時間集中</strong><p>延後回饋把日期計算結果收斂到整點。</p></div>
<div class="pr-record"><strong>資料累積</strong><p>大量 <code>WaitToReward</code> 紀錄落在相同查詢窗口。</p></div>
<div class="pr-record"><strong>批量放大</strong><p>查詢、分組、S3 上傳與 Task 建立集中在同一輪。</p></div>
</div>

<div class="pr-note pr-danger">DDB 主單筆數不等於 NMQ 回饋 Task 數。Dispatcher 會依 <code>ShopId + MemberId + PromotionEngineId</code> 分組，每個群組建立一個 Task；群組內的多筆訂單則放入同一份 S3 清單。</div>

</div>

<!-- endtab -->

<!-- tab BookingTime 從哪裡來 -->

<div class="pr-flow">
<div class="pr-lead"><p>並非所有回饋都會集中在午夜。真正會進入分桶的是 <code>RewardDays &gt; 0</code> 的延後回饋；即時回饋保留原始訂單時間，指定日期回饋則保留營運設定的時間值。</p></div>

## 四條時間來源

<figure class="pr-figure"><div class="pr-diagram" tabindex="0" role="region" aria-label="BookingTime 來源與是否分桶的決策流程">
{% mermaid %}
flowchart TB
    Start["計算 BookingTime"] --> Timing{"回饋時間類型"}
    Timing -->|RewardDays 等於零| Immediate["保留原始訂單時間"]
    Timing -->|指定日期| ByDate["使用 RewardDateTime"]
    Timing -->|RewardDays 大於零| Delayed["以日期加 N 天"]
    Delayed --> Allocate["RewardBookingTimeAllocator 分桶"]
    Retry["優惠券庫存不足重試"] --> Fixed["固定排到 10:00 或 22:00"]
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef success fill:#e8f1dc,stroke:#6f9250,color:#3f542d
    classDef result fill:#fbf7ef,stroke:#b49b78,color:#493b2d
    class Start,Delayed,Retry service
    class Timing decision
    class Allocate success
    class Immediate,ByDate,Fixed result
{% endmermaid %}
</div><figcaption>三桶分配只處理加 N 天回饋，不會改寫即時回饋、指定日期回饋與券不足重試時間。</figcaption></figure>

| 情境 | BookingTime 計算方式 | 是否進入三桶分配 |
| --- | --- | --- |
| `RewardDays = 0` | 保留 `orderTime` | 否 |
| `RewardDays > 0` | 以訂單日期或重算基準加 N 天 | 是 |
| `RewardTimingType = ByDateTime` | 使用 `RewardDateTime` | 否 |
| 優惠券庫存不足重試 | `CalculateNextRetryBookingTime` 排到 10:00 或 22:00 | 否 |

## 會套用分桶的實作路徑

<figure class="pr-figure"><div class="pr-diagram pr-wide" tabindex="0" role="region" aria-label="四條使用共同 BookingTime 分桶元件的實作路徑">
{% mermaid %}
flowchart LR
    ECom["線上訂單初次計算"] --> Allocator["RewardBookingTimeAllocator"]
    CRM["線下訂單初次計算"] --> Allocator
    Point["給點時間重算"] --> Allocator
    Coupon["給券時間重算"] --> Allocator
    Allocator --> Booking["00:00、01:00 或 02:00"]
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef success fill:#e8f1dc,stroke:#6f9250,color:#3f542d
    class ECom,CRM,Point,Coupon service
    class Allocator decision
    class Booking success
{% endmermaid %}
</div><figcaption>四條路徑使用相同分桶元件與相同 Seed 規則，避免初次計算與後續重算得到不同時間。</figcaption></figure>

| 路徑 | 實作位置 | 分桶 Seed |
| --- | --- | --- |
| 線上訂單初次計算 | `EComOrderPromotionRewardCalculateService.GetRewardConditionSetting` | `PromotionEngineId_TradesOrderGroupCode` |
| 線下訂單初次計算 | `OthersCrmSalesOrderPromotionRewardCalculateService.GetRewardConditionSetting` | `PromotionEngineId_TradesOrderGroupCode` |
| 未出貨或出貨後給點重算 | `LoyaltyPointService.SetBookingTimeAsync` | `PromotionEngineId_TradesOrderGroupCode` |
| 未出貨或出貨後給券重算 | `CouponService.SetBookingTimeAsync` | `PromotionEngineId_TradesOrderGroupCode` |

</div>

<!-- endtab -->

<!-- tab CronJob 與完整派工 -->

<div class="pr-flow">
<div class="pr-lead"><p>K8s CronJob 只負責建立 Dispatcher Task。查詢 DynamoDB、依會員與活動分組、上傳 S3，以及建立真正的給點或給券 Task，都由 <code>BookingRewardLoyaltyPointsDispatcherV2Job</code> 接手。</p></div>

## 每小時 01 分如何喚起 Dispatcher

<figure class="pr-figure"><div class="pr-diagram pr-wide" tabindex="0" role="region" aria-label="CronJob 建立 Dispatcher Task 的時序圖">
{% mermaid %}
sequenceDiagram
    autonumber
    participant Cron as K8s CronJob
    participant Script as CreateTask Script
    participant Admin as NMQ Admin API
    participant Queue as NMQ
    participant Dispatcher as Booking Dispatcher
    Cron->>Script: 每小時 01 分啟動
    Script->>Script: 產生含時區的 ExecuteTime
    Script->>Admin: POST /api/v1/task
    Admin->>Queue: 建立 Dispatcher Task
    Queue-->>Dispatcher: 交付 BookingRewardLoyaltyPointsDispatcherV2
    Note over Cron,Dispatcher: CronJob 只建立入口 Task，尚未建立給點或給券 Task
{% endmermaid %}
</div><figcaption>Production 排程為 <code>1 * * * *</code>，時區設定為 <code>Asia/Taipei</code>。</figcaption></figure>

## Dispatcher 如何把資料轉成回饋 Task

<figure class="pr-figure"><div class="pr-diagram pr-wide" tabindex="0" role="region" aria-label="Dispatcher 查詢分組與建立回饋 Task 的完整資料流">
{% mermaid %}
flowchart TB
    Execute["解析 ExecuteTime"] --> Window["計算前一小時整點窗口"]
    Window --> Query["依日期分區與 BookingTime 查詢 DDB"]
    Query --> Filter["保留 WaitToReward"]
    Filter --> Any{"有符合紀錄？"}
    Any -->|否| Stop["本次結束"]
    Any -->|是| Group["依 ShopId、MemberId、PromotionEngineId 分組"]
    Group --> Sort["群組內依 OrderTime 排序"]
    Sort --> Source["判斷 ECom 或 Others 來源"]
    Source --> Upload["上傳 memberDataList 至 S3"]
    Upload --> RewardType{"RewardType 是 ECoupon？"}
    RewardType -->|是| Coupon["建立 Coupon PriorityLow Request"]
    RewardType -->|否| Point["建立 LoyaltyPoints PriorityLow Request"]
    Coupon --> Collect["加入 taskRequests"]
    Point --> Collect
    Collect --> More{"還有下一個群組？"}
    More -->|是| Sort
    More -->|否| Create["逐一呼叫 Task.CreateAsync"]
    Create --> Worker["回饋 Worker 下載 S3 清單並依序處理"]
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef success fill:#e8f1dc,stroke:#6f9250,color:#3f542d
    classDef result fill:#fbf7ef,stroke:#b49b78,color:#493b2d
    class Execute,Window,Query,Filter,Group,Sort,Source,Upload,Collect service
    class Any,RewardType,More decision
    class Coupon,Point,Create,Worker success
    class Stop result
{% endmermaid %}
</div><figcaption>箭頭表示本次 Dispatcher 的處理順序。所有群組的 S3 上傳與 Request 組裝完成後，才進入逐一建立 NMQ Task 的階段。</figcaption></figure>

<div class="pr-note">查詢窗口的計算是 <code>startTime = ExecuteTime.AddHours(-1).ToHour()</code>，<code>endTime = ExecuteTime.ToHour()</code>。DDB 查詢使用日期分區鍵與 <code>BookingTimeUTC</code> 排序鍵。</div>

</div>

<!-- endtab -->

<!-- tab 三桶分配實作 -->

<div class="pr-flow">
<div class="pr-lead"><p>目前 Production 將加 N 天回饋分成三個整點桶。相同活動與訂單群組會穩定落到同一桶，目標是把原本集中在午夜的資料拆散，同時讓正流程、重算與稽核得到一致結果。</p></div>

## 從 Seed 到 BookingTime

<figure class="pr-figure"><div class="pr-diagram pr-wide" tabindex="0" role="region" aria-label="三桶 BookingTime 分配流程">
{% mermaid %}
flowchart LR
    Seed["PromotionEngineId 與 TradesOrderGroupCode"] --> MD5["計算 MD5"]
    MD5 --> Bytes["取前四個 byte"]
    Bytes --> UInt["轉成 uint32"]
    UInt --> Mod["對 bucketCount 取餘數"]
    Mod --> B0["bucket 0，00:00"]
    Mod --> B1["bucket 1，01:00"]
    Mod --> B2["bucket 2，02:00"]
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef success fill:#e8f1dc,stroke:#6f9250,color:#3f542d
    class Seed,MD5,Bytes,UInt service
    class Mod decision
    class B0,B1,B2 success
{% endmermaid %}
</div><figcaption>Prod 的 <code>BookingTimeAllocatorBucketCount</code> 目前為 3，輸出為目標日期的 00:00、01:00 或 02:00。</figcaption></figure>

```csharp
var bucketIndex = (int)(BitConverter.ToUInt32(hash, 0) % (uint)bucketCount);
var bookingTime = dateOnlyBase.Date.AddHours(startHour + bucketIndex);
```

## 實作帶來的改善範圍

<figure class="pr-figure"><div class="pr-diagram pr-compact" tabindex="0" role="region" aria-label="分桶前後 BookingTime 分布比較">
{% mermaid %}
flowchart TB
    subgraph Before["分桶前"]
        All["全部延後回饋"] --> Midnight["00:00"]
    end
    subgraph After["分桶後"]
        Split["延後回饋"] --> H0["00:00"]
        Split --> H1["01:00"]
        Split --> H2["02:00"]
    end
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef danger fill:#f9e9e3,stroke:#bb765f,color:#704737
    classDef success fill:#e8f1dc,stroke:#6f9250,color:#3f542d
    class All,Split service
    class Midnight danger
    class H0,H1,H2 success
{% endmermaid %}
</div><figcaption>這張圖只表示 BookingTime 已被分散，不代表 Dispatcher 實際負載一定形成三個等量批次。</figcaption></figure>

這項實作已於 2026 年 9 月 8 日合併至 master，merge commit 為 `0ac39b3c`，工作項目為 `VSTS661981`。它改善的是寫入時間分布；實際派工分布仍會受到查詢區間邊界影響。

</div>

<!-- endtab -->

<!-- tab 雜湊與整數安全 -->

<div class="pr-flow">
<div class="pr-lead"><p><code>MD5.ComputeHash</code> 回傳的是 16 bytes，不是可直接拿來分桶的整數。實作只取前四個 byte 轉成 <code>uint32</code>，先完成取餘數，再轉成 <code>int</code>，才能避免負數桶號。</p></div>

## 型別轉換的正確順序

<figure class="pr-figure"><div class="pr-diagram pr-wide" tabindex="0" role="region" aria-label="雜湊值轉成安全桶號的正確與錯誤路徑">
{% mermaid %}
flowchart TB
    Hash["MD5 的前四個 byte"] --> UInt["BitConverter.ToUInt32"]
    UInt --> Choice{"先做哪一步？"}
    Choice -->|先取餘數| SafeMod["uint32 % bucketCount"]
    SafeMod --> SafeInt["結果再轉 int"]
    SafeInt --> Safe["桶號落在合法範圍"]
    Choice -->|先轉 int| Overflow["超過 int.MaxValue 時變成負數"]
    Overflow --> Negative["負數 remainder 可能產生負桶號"]
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef success fill:#e8f1dc,stroke:#6f9250,color:#3f542d
    classDef danger fill:#f9e9e3,stroke:#bb765f,color:#704737
    class Hash,UInt service
    class Choice decision
    class SafeMod,SafeInt,Safe success
    class Overflow,Negative danger
{% endmermaid %}
</div><figcaption>C# 的百分比運算是 remainder。被除數為負數時，結果也可能為負數。</figcaption></figure>

```csharp
// 正確：先在 uint 範圍取餘數，最後才轉成 int
var bucketIndex = (int)(BitConverter.ToUInt32(hash, 0) % (uint)bucketCount);
```

以 Seed `28343_TG20193021` 為例，MD5 前四個 byte 轉出的 `uint32` 是 `4,273,969,079`。若先強制轉成 `int`，結果會變成 `-20,998,217`，再對 3 取餘數便可能得到 `-2`。

<div class="pr-note pr-success">單元測試需涵蓋大量不同 Seed，並驗證 <code>bucketIndex</code> 永遠滿足 <code>0 &lt;= bucketIndex &lt; bucketCount</code>。固定 Seed 也應重複得到相同桶號。</div>

</div>

<!-- endtab -->

<!-- tab Auditor 相容性 -->

<div class="pr-flow">
<div class="pr-lead"><p>BookingTime 改成三桶後，未出貨訂單的稽核不能再假設時間固定是 00:00。目前該路徑已改用相同的 <code>RewardBookingTimeAllocator</code> 重算預期值，其他稽核則依各自的時間語意維持原有比較方式。</p></div>

## 各稽核路徑如何比較時間

<figure class="pr-figure"><div class="pr-diagram pr-wide" tabindex="0" role="region" aria-label="BookingTime Auditor 各驗證路徑決策圖">
{% mermaid %}
flowchart TB
    Verify["PromotionRewardBookingTimeAuditor"] --> Type{"驗證情境"}
    Type -->|已出貨訂單| Shipped["比較 BookingDate"]
    Type -->|未出貨訂單| Unshipped["用相同 Allocator 重算預期 BookingTime"]
    Type -->|指定日期回饋| DateTime["依 RewardDateTime 比對"]
    Type -->|券不足重試| Coupon["依 10:00 或 22:00 比對"]
    Unshipped --> Tolerance["容忍範圍至少涵蓋三桶"]
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef success fill:#e8f1dc,stroke:#6f9250,color:#3f542d
    classDef result fill:#fbf7ef,stroke:#b49b78,color:#493b2d
    class Verify service
    class Type decision
    class Unshipped,Tolerance success
    class Shipped,DateTime,Coupon result
{% endmermaid %}
</div><figcaption>只有未出貨訂單需要與三桶配置保持一致，券不足重試仍是另一套固定時刻機制。</figcaption></figure>

| 方法 | 比對方式 | 三桶影響 |
| --- | --- | --- |
| `VerifyBookingTimeForShippedOrder` | 比對日期 | 同日桶位不改變日期 |
| `VerifyBookingTimeForUnshippedOrder` | 共用 Allocator 重算預期值 | 已同步三桶邏輯 |
| `VerifyBookingTimeForRewardDateTime` | 依 `RewardDateTime` 比對 | 不受加 N 天分桶影響 |
| `VerifyBookingTimeForInsufficientECoupon` | 依 10:00 或 22:00 比對 | 不屬於本次分桶 |

<div class="pr-note">目前未出貨稽核容忍時數採 <code>Math.Max(bucketCount, 3)</code>。這能包住現行三桶偏移，但配置若持續增加，仍需確認容忍語意與精確邊界是否符合預期。</div>

</div>

<!-- endtab -->

<!-- tab 三桶為何可能不如預期 -->

<div class="pr-flow">
<div class="pr-lead"><p>三個 BookingTime 不一定形成三次各三分之一的派工。Dispatcher 使用 DynamoDB <code>BETWEEN</code> 查前一小時，而 <code>BETWEEN</code> 同時包含左右邊界，因此落在整點的桶可能被相鄰兩個查詢窗口命中。</p></div>

## 預期窗口與實際窗口

<figure class="pr-figure"><div class="pr-diagram pr-wide" tabindex="0" role="region" aria-label="DynamoDB 閉區間造成整點桶重疊的時間窗口圖">
{% mermaid %}
flowchart TB
    B0["bucket 0，00:00"] --> Q1["01:01 查詢，00:00 到 01:00"]
    B1["bucket 1，01:00"] --> Q1
    B1 --> Q2["02:01 查詢，01:00 到 02:00"]
    B2["bucket 2，02:00"] --> Q2
    B2 --> Q3["03:01 查詢，02:00 到 03:00"]
    Q1 --> Load1["第一波可能同時承接兩桶"]
    Q2 --> Load2["第二波包含新桶與邊界重掃可能"]
    Q3 --> Load3["第三波可能只剩邊界重掃"]
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef danger fill:#f9e9e3,stroke:#bb765f,color:#704737
    classDef result fill:#fbf7ef,stroke:#b49b78,color:#493b2d
    class B0,B1,B2 service
    class Q1,Q2,Q3 decision
    class Load1,Load2 danger
    class Load3 result
{% endmermaid %}
</div><figcaption>連線表示閉區間可能命中的桶，不表示相鄰兩次一定重複建立 Task。是否重掃還取決於紀錄當時是否仍為 <code>WaitToReward</code>。</figcaption></figure>

```csharp
var startTime = entity.ExecuteTime.AddHours(-1).ToHour();
var endTime = entity.ExecuteTime.ToHour();

// DynamoDB QueryOperator.Between 同時包含 startTime 與 endTime
Values = new object[] { startTime, endTime };
```

| Dispatcher 時間 | 閉區間可能命中的桶 | 對負載的影響 |
| --- | --- | --- |
| 01:01 | 00:00 與 01:00 | 第一波可能接近兩桶資料 |
| 02:01 | 01:00 與 02:00 | 01:00 邊界資料若仍待處理，可能再次命中 |
| 03:01 | 02:00 與 03:00 | 沒有第四桶，但 02:00 邊界仍在查詢範圍 |

## 修正方向

<figure class="pr-figure"><div class="pr-diagram" tabindex="0" role="region" aria-label="查詢邊界問題的三種修正方向">
{% mermaid %}
flowchart TB
    Issue["整點桶落在閉區間邊界"] --> Choice{"選擇修正層級"}
    Choice -->|查詢層| HalfOpen["改為左閉右開區間"]
    Choice -->|時間層| Offset["BookingTime 避開整點邊界"]
    Choice -->|派工層| Idempotent["增加派工鎖或冪等檢查"]
    HalfOpen --> Preferred["優先方案，直接消除窗口重疊"]
    Offset --> Sync["同步調整 Auditor 與測試"]
    Idempotent --> Limit["降低重派風險，但不降低第一波兩桶負載"]
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef success fill:#e8f1dc,stroke:#6f9250,color:#3f542d
    classDef result fill:#fbf7ef,stroke:#b49b78,color:#493b2d
    classDef danger fill:#f9e9e3,stroke:#bb765f,color:#704737
    class Issue danger
    class Choice decision
    class HalfOpen,Preferred success
    class Offset,Idempotent,Sync,Limit result
{% endmermaid %}
</div><figcaption>左閉右開區間讓每個整點只屬於一個窗口，最直接對齊三桶設計。</figcaption></figure>

建議優先將查詢語意改成 `BookingTime >= startTime` 且 `BookingTime < endTime`，並補上整點與跨日整合測試。在完成修正與 Production 數據複驗前，較精確的結論是：三桶已降低單一 00:00 的資料聚集，但尚不能宣稱 Dispatcher 負載穩定形成三次各三分之一。

<details class="pr-evidence"><summary>建議補齊的整合測試</summary>

1. `ExecuteTime = 01:01` 時，只撈取 `00:00 <= BookingTime < 01:00`。
2. `BookingTime = 01:00:00` 時，不會被 01:01 與 02:01 兩次查詢同時命中。
3. 00:00、01:00、02:00 三桶分別落在三個不同的 Dispatcher 執行。
4. 23:00 到 00:00 的跨日窗口，能正確對應 `BookingDateUTC-BookingTimeUTC-index` 分區鍵。

</details>

</div>

<!-- endtab -->

{% endtabs %}
