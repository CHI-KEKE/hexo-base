---
title: 回饋活動 - 為何用 DynamoDB
date: 2026-10-17 20:39:00
categories: PromotionReward
top_img: https://pub-d7e550ef212547d888a6e01348459946.r2.dev/ecom/promotion/promotionreward.png
cover : https://pub-d7e550ef212547d888a6e01348459946.r2.dev/ecom/promotion/promotionreward.png
toc:
toc_number:
comments :
tags:
---

{% tabs 回饋活動 - 1 %}



<!-- tab 架構 -->

<div class="pr-flow">

<style>
.pr-flow{--pr-ink:#493b2d;--pr-muted:#71624e;--pr-line:#e4d8bf;--pr-panel:#fbf7ef;--pr-accent:#57753f;--pr-bear:#a06a41;--pr-diagram-bg:#fffdf7;color:var(--pr-ink);line-height:1.9;overflow-wrap:anywhere}
.pr-flow *{box-sizing:border-box}
.pr-flow h2,.pr-flow h3,.pr-flow h4{color:var(--pr-ink)}
.pr-flow h3{margin-top:2.5em;margin-bottom:1.1em;font-size:1.15em;border-bottom:1px dashed var(--pr-line);padding-bottom:12px}
.pr-lead{position:relative;overflow:hidden;padding:28px 72px 28px 26px;border-radius:24px;background:radial-gradient(circle at 12% 10%,#ffffff12,transparent 50%),linear-gradient(150deg,#6f9250,#547a3d 60%,#3f5f2e);color:#fffbed;margin:20px 0 30px;box-shadow:0 6px 18px #59462212}
.pr-lead::before{content:'🌼';position:absolute;right:22px;top:14px;font-size:20px;opacity:.7}
.pr-lead::after{content:'🐻';position:absolute;right:18px;bottom:15px;font-size:32px}
.pr-lead p{margin:0;font-size:1em;line-height:1.9}
.pr-figure{border:1px solid var(--pr-line);border-radius:20px;overflow:hidden;margin:24px 0;background:var(--pr-diagram-bg)}
.pr-diagram{padding:24px 16px;overflow-x:auto;background:var(--pr-diagram-bg)}
.pr-diagram .mermaid-wrap{margin:0}
.pr-diagram svg{display:block;margin:auto;width:100%;min-width:600px;max-width:none!important;height:auto}
.pr-diagram.pr-decisions svg{min-width:1000px}
.pr-diagram.pr-select svg{min-width:420px;max-width:640px!important}
.pr-diagram.pr-compare svg{min-width:270px;max-width:500px!important}
.pr-diagram:focus-visible{outline:2px solid var(--pr-accent);outline-offset:-3px}
.pr-figure figcaption{background:var(--pr-panel);color:var(--pr-muted);padding:14px 18px;font-size:13px;border-top:1px solid var(--pr-line)}
.pr-flow .pr-figure h4{margin:0;padding:17px 20px;background:var(--pr-panel);font-size:1em}
#article-container .pr-flow code{overflow-wrap:anywhere;color:var(--pr-bear);background:var(--pr-panel)}
.pr-flow table{font-size:14px;width:100%}
.pr-flow th{background:var(--pr-panel);color:var(--pr-ink)}
.pr-flow th,.pr-flow td{white-space:normal;overflow-wrap:anywhere;border-color:var(--pr-line);padding:12px}
.pr-flow .pr-version-table{table-layout:fixed}
.pr-flow .pr-version-table th:nth-child(1){width:28%}
.pr-flow .pr-version-table th:nth-child(2){width:28%}
.pr-note{border-left:4px solid var(--pr-bear);border-radius:0 16px 16px 0;padding:18px 20px;background:var(--pr-panel);margin:24px 0}
.pr-evidence{border:1px solid var(--pr-line);border-radius:16px;padding:16px 20px;margin:24px 0;background:var(--pr-panel)}
.pr-evidence summary{cursor:pointer;color:var(--pr-accent);font-weight:700}
.pr-evidence[open] summary{margin-bottom:18px}
.pr-evidence li{margin:10px 0}
.pr-flow .cozy-line-steps{list-style:none;margin:26px 0;padding:0 0 0 84px;position:relative}
.cozy-line-steps::before{content:'';position:absolute;left:30px;top:18px;bottom:22px;width:2px;background:linear-gradient(var(--pr-bear),var(--pr-accent));opacity:.5}
.pr-flow .cozy-line-steps li{position:relative;margin:0 0 26px;padding:0 0 4px}
.cozy-line-steps .cl-time{position:absolute;left:-84px;top:0;width:62px;line-height:38px;border:1px solid var(--pr-line);border-radius:24px;background:var(--pr-panel);text-align:center;color:var(--pr-bear);font-size:14px;font-weight:700}
.cozy-line-steps .cl-title{display:block;padding-top:4px;font-weight:700}
.cozy-line-steps .cl-desc{display:block;color:var(--pr-muted);margin-top:5px}
[data-theme="dark"] .pr-flow{--pr-ink:#ece5d6;--pr-muted:#c6bca8;--pr-line:#575744;--pr-panel:#2b3025;--pr-accent:#b4cd92;--pr-bear:#e1b68c;--pr-diagram-bg:#242a20}
[data-theme="dark"] .pr-lead{background:linear-gradient(150deg,#4b6538,#354d29)}
[data-theme="dark"] .pr-diagram .cluster rect{fill:#343d2d!important;stroke:#8e9f78!important}
[data-theme="dark"] .pr-diagram .cluster-label span{color:#ece5d6!important}

.pr-record-grid{display:grid;grid-template-columns:repeat(3,minmax(0,1fr));gap:14px;margin:22px 0}
.pr-record{border:1px solid var(--pr-line);border-radius:18px;background:var(--pr-panel);padding:18px 20px;box-shadow:0 5px 18px #59462208}
.pr-record strong{display:block;color:var(--pr-ink);margin-bottom:8px}
.pr-record p{margin:0;color:var(--pr-muted)}
@media(max-width:600px){.pr-lead{padding:22px 52px 22px 18px}.pr-lead::after{font-size:25px;right:12px}.pr-lead::before{right:16px;font-size:16px}.pr-note,.pr-evidence{padding:15px}.pr-diagram{padding:18px 10px}.pr-flow th,.pr-flow td{padding:9px 7px;font-size:13px}.pr-record-grid{grid-template-columns:1fr}}
</style>



<div class="pr-lead">
<p>回饋活動算出應給的點數或優惠券後，不一定會立刻發放。系統可能需要等待指定時間或訂單條件成立；發放後，也可能因取消或退貨而回收。因此，每筆訂單都需要留下目前處理到哪一步的紀錄，讓後續派工、重試與回收有依據。這些執行狀態保存在 DynamoDB。</p>
</div>

Promotion Engine 負責計算應給多少回饋；DynamoDB 保存回饋履約的狀態與當時結果，讓後續工作能接續處理。以下五張圖分別說明狀態總覽，以及線上、線下訂單的正向與逆向流程。

DynamoDB 在此承擔執行狀態帳本的角色，支援後續工作依紀錄判斷下一步：

- 等待：用 `WaitToReward` 與 `BookingTime` 記錄待發放的回饋。
- 派工：Dispatcher 依時間窗找出待處理紀錄。
- 發放：用 `Reward` 與 `IsReward` 記錄發放進度。
- 取消與回收：用 `Cancel`、`IsRecycle` 與 `RecyclePoints` 記錄逆向處理。
- 重試與冪等：依 Lock、條件更新與狀態判斷防止重複處理。
- 稽核與補償：保留歷史紀錄與 `UpdateReason`。

## 回饋處理流程

### 回饋狀態與觸發事件

這張圖先看訂單成立後如何進入 `Occupy`、`WaitToReward` 或 `Reward`，以及付款失敗或整張訂單取消時的 `Cancel`。`MatchWithoutQuota` 與 `Unmatch` 是圖中另外兩種判斷結果。

<figure class="pr-figure">
<div class="pr-diagram" tabindex="0" role="region" aria-label="回饋狀態與觸發事件">
{% mermaid %}
flowchart TB
    Place["訂單成立，尚未付款"] -->|OrderPlace| NeedQuota{"活動需要佔額？"}
    NeedQuota -->|否| Paid["訂單付款完成"]
    NeedQuota -->|是| ActiveBefore{"有命中活動？"}
    ActiveBefore -->|否| UnmatchBefore["Unmatch"]
    ActiveBefore -->|是| Full{"額度已滿？"}
    Full -->|否| Occupy["Occupy"]
    Full -->|是| NoQuota["MatchWithoutQuota"]
    Occupy --> Paid
    Paid -->|OrderCreate| ActiveAfter{"有命中活動？"}
    ActiveAfter -->|否| UnmatchAfter["Unmatch"]
    ActiveAfter -->|是| Wait["WaitToReward"]
    Wait --> Due{"到達給點或給券時間？"}
    Due -->|否| Wait
    Due -->|是| Reward["Reward"]
    PaymentFailed["訂單付款失敗"] --> Cancel["Cancel"]
    OrderCancelled["整張訂單取消"] --> Cancel
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef waiting fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef success fill:#e8f1dc,stroke:#6f9250,color:#3f542d
    classDef result fill:#fbf7ef,stroke:#b49b78,color:#493b2d
    classDef error fill:#f9e9e3,stroke:#bb765f,color:#704737
    class Place,Paid service
    class NeedQuota,ActiveBefore,Full,ActiveAfter,Due decision
    class Occupy,Wait waiting
    class Reward success
    class PaymentFailed error
    class OrderCancelled,Cancel,UnmatchBefore,UnmatchAfter,NoQuota result
{% endmermaid %}
</div>
<figcaption>黃色標示等待或佔額，綠色標示已發放。取消與未命中是另外的結果。</figcaption>
</figure>


圖中的付款後判斷若已經計算過，便沿用先前結果，不重新判斷活動是否命中。這張總覽依原圖保留狀態與事件，具體的發放與回收工作分別見下方流程。

### 線上訂單的正向流程

訂單成立時，`PromotionRewardOccupyDispatcherJob` 判斷活動是否需要佔額。需要佔額時，`PromotionRewardCouponJob` 或 `PromotionRewardLoyaltyPointsV2Job` 判斷資格並建立 `Occupy` 狀態；不需要佔額則直接等待付款完成事件。

<figure class="pr-figure">
<div class="pr-diagram" tabindex="0" role="region" aria-label="線上訂單正向流程">
{% mermaid %}
flowchart TB
    Place["訂單成立 OrderPlace"] --> OccupyDispatcher["佔額派工"]
    OccupyDispatcher --> NeedQuota{"需要佔額？"}
    NeedQuota -->|是| OccupyJob["回饋工作判斷資格並佔額"]
    NeedQuota -->|否| Create["訂單付款完成 OrderCreate"]
    OccupyJob -->|Occupy| Create
    Create --> RewardDispatcher["給點派工判斷活動資格"]
    RewardDispatcher --> Qualified{"符合活動資格？"}
    Qualified -->|否| End["結束"]
    Qualified -->|是| Record["回饋工作建立 DDB 紀錄"]
    Record --> Delayed{"狀態為 WaitToReward？"}
    Delayed -->|否，立即贈與| End
    Delayed -->|是| Booking["每小時檢查回饋時間"]
    Booking --> Due{"到達回饋時間？"}
    Due -->|否| Booking
    Due -->|是| Grant["回饋工作發放點數或優惠券"]
    Grant -->|Reward| End
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef waiting fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef success fill:#e8f1dc,stroke:#6f9250,color:#3f542d
    classDef result fill:#fbf7ef,stroke:#b49b78,color:#493b2d
    classDef error fill:#f9e9e3,stroke:#bb765f,color:#704737
    class Place,Create,OccupyDispatcher,OccupyJob,RewardDispatcher,Record,Booking service
    class NeedQuota,Qualified,Delayed,Due decision
    class Grant success
    class End result
{% endmermaid %}
</div>
<figcaption>線上訂單先處理佔額，付款後建立回饋紀錄；等待中的回饋再由定時派工接手。</figcaption>
</figure>


付款完成後由 `PromotionRewardLoyaltyPointsDispatcherV2Job` 判斷活動資格，回饋工作建立 DynamoDB 紀錄。若狀態是 `WaitToReward`，`BookingRewardLoyaltyPointsDispatcherV2Job` 每小時檢查回饋時間；到時再由 `PromotionRewardCouponJob` 或 `PromotionRewardLoyaltyPointsV2Job` 發放。原圖將立即贈與標成建檔後結束，沒有另外畫出發放動作。

### 線下訂單的正向流程

線下訂單由每日排程先撈取前一日匯入的 CRM 訂單，再判斷適用活動並建立回饋紀錄。等待與發放階段沿用線上訂單的每小時派工流程。

<figure class="pr-figure">
<div class="pr-diagram" tabindex="0" role="region" aria-label="線下訂單正向流程">
{% mermaid %}
flowchart TB
    Schedule["每日 04:00 排程"] --> Import["撈取昨日匯入的 CRM 訂單"]
    Import --> Match["判斷訂單適用的活動"]
    Match --> Record["回饋工作佔額並建立 DDB 紀錄"]
    Record --> Delayed{"狀態為 WaitToReward？"}
    Delayed -->|否，立即贈與| End["結束"]
    Delayed -->|是| Booking["每小時檢查回饋時間"]
    Booking --> Due{"到達回饋時間？"}
    Due -->|否| Booking
    Due -->|是| Grant["回饋工作發放點數或優惠券"]
    Grant -->|Reward| End
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef waiting fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef success fill:#e8f1dc,stroke:#6f9250,color:#3f542d
    classDef result fill:#fbf7ef,stroke:#b49b78,color:#493b2d
    classDef error fill:#f9e9e3,stroke:#bb765f,color:#704737
    class Schedule,Import,Match,Record,Booking service
    class Delayed,Due decision
    class Grant success
    class End result
{% endmermaid %}
</div>
<figcaption>線下訂單由每日排程匯入；已建立的待發放紀錄沿用每小時派工流程。</figcaption>
</figure>


撈單工作為 `PromotionRewardActiveCrmShopDispatcherJob`，活動判斷由 `PromotionRewardBatchDispatcherV2Job` 處理。建立紀錄與發放由 `PromotionRewardCouponJob` 或 `PromotionRewardLoyaltyPointsV2Job` 處理；等待中的訂單由 `BookingRewardLoyaltyPointsDispatcherV2Job` 檢查。

### 線上訂單的逆向流程

付款失敗會取消先前佔額。訂單取消或退貨則先判斷是否需要回收，只有需要時才執行點數或優惠券回收。

<figure class="pr-figure">
<div class="pr-diagram" tabindex="0" role="region" aria-label="線上訂單逆向流程">
{% mermaid %}
flowchart TB
    Failed["訂單付款失敗 OrderFailed"] --> OccupyDispatcher["佔額派工取消佔額"]
    OccupyDispatcher --> End["結束"]
    Reversed["訂單取消或退貨"] --> RecycleDispatcher["回收派工判斷回收需求"]
    RecycleDispatcher --> NeedRecycle{"需要回收？"}
    NeedRecycle -->|否| End
    NeedRecycle -->|是| Recycle["回收點數或優惠券"]
    Recycle --> End
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef waiting fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef success fill:#e8f1dc,stroke:#6f9250,color:#3f542d
    classDef result fill:#fbf7ef,stroke:#b49b78,color:#493b2d
    classDef error fill:#f9e9e3,stroke:#bb765f,color:#704737
    class Reversed,OccupyDispatcher,RecycleDispatcher,Recycle service
    class Failed error
    class NeedRecycle decision
    class End result
{% endmermaid %}
</div>
<figcaption>付款失敗會取消佔額；訂單取消或退貨先判斷是否需要回收。</figcaption>
</figure>


付款失敗由 `PromotionRewardOccupyDispatcher` 取消佔額。取消或退貨事件交給 `RecycleLoyaltyPointsDispatcherV2` 判斷，必要時再由 `PromotionRecycleCouponJob` 或 `RecycleLoyaltyPointsV2` 回收。

### 線下訂單的逆向流程

線下訂單每天掃描前一日匯入的 CRM 訂單，發現取消或退貨時先預約回收任務，再於 10:30 執行回收。

<figure class="pr-figure">
<div class="pr-diagram" tabindex="0" role="region" aria-label="線下訂單逆向流程">
{% mermaid %}
flowchart TB
    Morning["每日 04:00 排程"] --> Import["撈取昨日匯入的 CRM 訂單"]
    Import --> Scan["掃描訂單狀態並預約回收任務"]
    Scan --> Later["每日 10:30 排程"]
    Later --> Recycle["依預約任務回收點數或優惠券"]
    Recycle --> End["結束"]
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef waiting fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef success fill:#e8f1dc,stroke:#6f9250,color:#3f542d
    classDef result fill:#fbf7ef,stroke:#b49b78,color:#493b2d
    classDef error fill:#f9e9e3,stroke:#bb765f,color:#704737
    class Morning,Import,Scan,Later,Recycle service
    class End result
{% endmermaid %}
</div>
<figcaption>線下訂單先預約回收任務，再由後續排程執行。</figcaption>
</figure>


`PromotionRewardActiveCrmShopDispatcherJob` 負責撈單，`PromotionRewardBatchDispatcherV2` 掃描取消或退貨狀態並預約任務，`PromotionRecycleCouponJob` 或 `RecycleLoyaltyPointsV2` 依預約任務執行回收。

</div>

<!-- endtab -->



<!-- tab 選用 Ddb 有哪些好處-->

<div class="pr-flow">

<div class="pr-lead">
<p>一筆訂單的回饋可能經過佔額、等待、發放，之後還可能因退貨而回收。這些工作發生在不同時間，並由不同的 Worker 接手。系統需要反覆找回同一筆回饋紀錄，確認目前的處理進度，再決定下一步。</p>
</div>

## 訂單事件與定時派工都有明確的查找方式

付款完成、取消或退貨時，系統可以依活動、訂單與子單的識別資料找回對應紀錄。例如，收到某張訂單的退貨事件後，回收工作要先找出這張訂單當初領到的回饋，才能繼續處理。

定時派工則從另一個方向找資料：每小時挑出仍在等待、且 `BookingTime` 已到的紀錄。DynamoDB 適合預先為這兩種固定的讀取方式安排鍵值與索引。時間查詢仍須先指定分區鍵，再依排序鍵縮小範圍；單有 `BookingTime` 欄位，無法直接用 `Query` 找遍所有待發放紀錄。

<figure class="pr-figure">
<div class="pr-diagram" tabindex="0" role="region" aria-label="訂單事件與定時派工查找回饋紀錄的兩種路徑">
{% mermaid %}
flowchart TB
    Event["付款、取消或退貨事件"] --> OrderKey["活動與訂單識別資料"]
    OrderKey --> Lookup["查找對應回饋"]
    Clock["每小時派工"] --> TimeKey["預先設計的分區鍵與時間排序鍵"]
    TimeKey --> Due["查找已到期的紀錄"]
    Lookup --> Record["DynamoDB 回饋紀錄"]
    Due --> Record
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef result fill:#fbf7ef,stroke:#b49b78,color:#493b2d
    class Event,Clock,Lookup,Due service
    class OrderKey,TimeKey decision
    class Record result
{% endmermaid %}
</div>
<figcaption>兩種工作都要讀回饋紀錄，但查找入口不同。時間查詢必須事先安排鍵值或索引。</figcaption>
</figure>

## 多個 Worker 可以同時處理不同訂單

活動期間可能有許多訂單陸續付款，也可能有大量回饋在相近時間到期。每個 Worker 主要讀取或更新自己負責的回饋紀錄，因此可以把不同訂單分開處理，而不必讓所有工作排成一條長隊。

DynamoDB 會依分區鍵分布資料，讓分散在不同鍵值上的讀寫共同承擔流量。這項好處取決於鍵值設計：若大量工作集中在少數熱門鍵值，同樣可能遇到熱點與限流。

## 後續工作能知道回饋處理到哪一步

訂單成立時可能先留下 `Occupy`，付款後若還沒到發放時間，回饋紀錄會進入 `WaitToReward`；完成發放後再記錄 `Reward`。若訂單取消或退貨，後續工作也要根據當前紀錄判斷是否需要取消或回收。

狀態轉換由回饋程式決定，DynamoDB 保存每次處理後的結果。`WaitToReward`、`Reward`、`Cancel` 表示流程狀態；`IsReward`、`IsRecycle` 則是幫助判斷發放或回收進度的標記，兩者用途不同。

## 事件重送時，可以有條件地更新紀錄

派工或訂單事件可能再次送達，也可能有兩個 Worker 同時碰到同一筆回饋。如果只先讀取狀態、稍後再寫入，讀取與寫入之間的狀態可能已被另一個 Worker 改變。

DynamoDB 的 `ConditionExpression` 可以在寫入同一筆紀錄時檢查目前狀態，只有條件成立才更新。這能降低重複變更紀錄的風險。點數或優惠券若由外部系統發放，仍須另外處理「外部發放成功，但本地紀錄更新失敗」的情境，才能讓重試有可靠依據。

## 保存當時發放結果，供回收與查證

退貨可能發生在活動規則變更之後。若用現在的規則重新計算，結果可能與當初實際發放的點數或優惠券不同。回收工作需要讀取當時的發放結果，才能知道要回收什麼、回收多少。

回饋紀錄可以保存這份結果快照，以及後續處理所需的狀態與原因。若只在同一筆紀錄上覆蓋欄位，仍需另外保存歷次變更，才能追查完整的狀態歷史。

<div class="pr-record-grid" role="group" aria-label="回饋紀錄保存的三類資料">
<div class="pr-record"><strong>識別資料</strong><p>讓訂單事件能找回對應的回饋。</p></div>
<div class="pr-record"><strong>處理進度</strong><p>保存等待、發放、取消與回收所需的狀態。</p></div>
<div class="pr-record"><strong>當時結果</strong><p>保存實際發放內容，供退貨回收與查證。</p></div>
</div>

這些固定的查找方式、分散的單筆讀寫、狀態更新與結果快照，共同構成選用 DynamoDB 的理由。實際效能與重試正確性，仍取決於鍵值設計以及回饋發放流程如何處理失敗。

參考資料：[DynamoDB Query](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.html)、[分區鍵設計](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-uniform-load.html)、[條件寫入](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.ConditionExpressions.html)、[DynamoDB Streams](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html)。

</div>

<!-- endtab -->



<!-- tab 如果用其他 solution 呢-->

<div class="pr-flow">

<div class="pr-lead">
<p>先用同一筆訂單來比較：付款後先等待三天發點，第五天又因退貨需要回收。無論使用哪種方案，系統都要能找出到期工作、避免重複發點，並保留當時實際發放的結果，供退貨時查詢。</p>
</div>

## 同一筆回饋交給不同方案時

| 方案 | 回饋狀態放在哪裡 | 到期與並發工作怎麼做 |
| --- | --- | --- |
| MySQL／SQL Server | 回饋資料表 | 建索引找待處理紀錄，用交易或條件更新領取工作 |
| MongoDB | 回饋文件 | 建索引找待處理文件，用單筆條件更新控制狀態 |
| 只有 MQ | 沒有獨立的目前狀態 | 必須另找結果來源，或保存完整事件並重建狀態 |
| DynamoDB | 回饋項目 | 依查找方式設計鍵與索引，用條件寫入更新單筆狀態 |

## 關聯式資料庫（MySQL／SQL Server）

關聯式資料庫可以保存每筆回饋的狀態，也能透過索引查出「這張訂單的回饋」與「現在到期、尚未發放的回饋」。多個 Worker 同時派工時，可以用交易、條件更新或跳過已被領取的工作，避免重複處理。這些做法都可行，是否會出現查詢變慢或鎖等待，取決於索引、交易範圍與實際流量，不能只因使用 SQL 就認定會發生。

但這份資料目前只有回饋活動系統讀寫：其他系統不需要直接查詢回饋狀態，也沒有和它一起完成交易的需求。因此，把資料放進 SQL，不會因此取得跨系統關聯查詢或共用交易的額外效益。若選擇 SQL，仍須為這個獨立流程設計到期派工、併發領取與相關索引；未來若出現跨系統查詢需求，再依實際情況評估即可。

## 文件型資料庫（例如 MongoDB）

MongoDB 也能把回饋狀態、`BookingTime` 與發放結果存成一份文件，並用索引查詢到期工作。Worker 更新文件時，可以把「目前仍是 `WaitToReward`」放進更新條件，避免兩個 Worker 都成功改變同一筆狀態。這種單筆條件更新，不是 DynamoDB 獨有的能力。

這裡的取捨主要是導入成本。若團隊尚未使用 MongoDB，為只有回饋活動系統使用的資料新增一套資料庫，還需要處理連線、權限、監控與維運。MongoDB Atlas 可以部署在 AWS 上，但仍是另一個服務；AWS 的 Amazon DocumentDB 則是 MongoDB 相容的服務，兩者功能並非完全相同，選用前要確認所需功能。若目前沒有文件型資料庫才提供的特殊需求，這些額外工作就缺少明確收益。

## 只有 MQ 傳遞事件，不保存回饋狀態

MQ 可以把付款、發放與退貨事件送給不同 Worker，卻不能在沒有其他紀錄的情況下，直接回答「這筆訂單是否已發點、發了多少、是否回收過」。第五天收到退貨事件時，回收工作仍須從某處取得當時的發放結果。

訊息也可能重送，所以收到同一事件兩次時，Worker 必須有防止重複發放的依據。若改為長期保存完整事件，並在需要時重播事件來重建狀態，也可以做到；但那就需要另外設計事件保存、查詢與重播機制，而不是單靠無狀態的訊息處理。

## 為什麼這裡選 DynamoDB？

回饋活動反覆執行的讀寫方式相對固定：依訂單找紀錄、找出已到期的工作、更新單筆狀態，並讀取原本的發放結果。DynamoDB 可依這些查找方式設計鍵值與索引，再用條件寫入控制狀態轉換，也不需要管理資料庫伺服器。這符合目前只有回饋活動系統使用資料的範圍。

這不代表 DynamoDB 自動解決所有問題。到期派工仍要設計可查詢的分區鍵與排序鍵；點數或優惠券由外部系統發放時，「發放成功，但本地紀錄更新失敗」也仍須靠流程設計處理。選型理由是它符合目前的固定查找方式與維運需求，而非其他方案無法實作。

參考資料：[MySQL 派工鎖定](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking-reads.html)、[SQL Server 工作佇列](https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table)、[MongoDB 單筆更新](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/)、[MongoDB Atlas on AWS](https://www.mongodb.com/docs/atlas/reference/amazon-aws/)、[Amazon DocumentDB 相容性](https://docs.aws.amazon.com/documentdb/latest/devguide/compatibility.html)、[DynamoDB Query](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.KeyConditionExpressions.html)。

</div>

<!-- endtab -->






<!-- tab 為什麼 高併發 key-based 就適合 ddb?-->

<div class="pr-flow">

<div class="pr-lead">
<p>回饋活動的「高併發」有兩種情況：許多 Worker 同時處理不同訂單，以及事件重送時，多個 Worker 同時碰到同一筆回饋。前者需要把讀寫流量分散，後者需要確認誰可以更新狀態。能否事先知道要找哪筆紀錄，也是選用 DynamoDB 的關鍵。</p>
</div>

<figure class="pr-figure">
<div class="pr-diagram" tabindex="0" role="region" aria-label="不同訂單的流量分散與同筆回饋的條件更新對照">
{% mermaid %}
flowchart TB
    subgraph Different["不同訂單：分散讀寫"]
        OrderA["訂單甲"] --> RecordA["回饋紀錄甲"]
        OrderB["訂單乙"] --> RecordB["回饋紀錄乙"]
    end
    subgraph Same["同一筆回饋：檢查更新條件"]
        WorkerA["Worker A"] --> CheckA{"狀態仍可更新？"}
        WorkerB["Worker B"] --> CheckB{"狀態仍可更新？"}
        CheckA -->|先完成| Updated["一筆狀態轉換成功"]
        CheckB -->|已被改變| Rejected["另一筆條件不成立"]
    end
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef success fill:#e8f1dc,stroke:#6f9250,color:#3f542d
    classDef result fill:#fbf7ef,stroke:#b49b78,color:#493b2d
    class OrderA,OrderB,RecordA,RecordB,WorkerA,WorkerB service
    class CheckA,CheckB decision
    class Updated success
    class Rejected result
{% endmermaid %}
</div>
<figcaption>右側以先後到達為例。條件寫入控制的是 DynamoDB 紀錄，外部發放仍要另外處理重試。</figcaption>
</figure>

## 訂單事件來了，先找到對應回饋

付款、取消或退貨事件都帶有訂單相關的識別資料。Worker 要先依預先設計的鍵值或索引，找出這張訂單的回饋紀錄，再判斷它目前是等待發放、已發放，還是已回收。若已有完整主鍵，可以用 `GetItem` 讀取單筆紀錄；若是依另一組識別資料查找，就需要為那種查找方式設計索引。不能只因資料裡有訂單欄位，就認為一定能直接找到紀錄。

## 不同訂單同時處理，流量才有機會分散

假設活動期間有大量訂單付款，每個 Worker 主要更新自己負責的回饋紀錄。DynamoDB 會依分區鍵分配資料；若鍵值與請求分布得夠均勻，這些讀寫可以共同使用不同分區的容量，不必由單一位置承擔所有工作。

這不是「不同 key 就一定不會互相影響」。如果所有回饋都使用少數熱門鍵值，例如只用同一個活動識別值集中派工，仍可能形成熱點而被限流。效能要看實際的分區鍵與流量分布，不能只看 Worker 數量。

## 同一筆回饋同時被處理，要檢查更新條件

派工重疊或訊息重送時，兩個 Worker 可能都讀到 `WaitToReward`。更新時可用 `ConditionExpression` 檢查「目前仍是 `WaitToReward`」，條件成立的寫入才會成功；狀態已被改掉的 Worker 就不能再做同一筆狀態轉換。這是保護 DynamoDB 紀錄的方式。

但條件更新本身不能保證外部點數或優惠券服務只執行一次。若外部發放成功、本地狀態卻沒寫入，重試時仍須能查證先前的結果，並由發放流程處理重複請求。

## 定時派工需要另一種查找方式

訂單事件可以依訂單識別資料找回饋；每小時派工則要找出 `BookingTime` 已到、仍待發放的一批紀錄。這不是已知單筆主鍵的讀取。DynamoDB 的 `Query` 必須先指定分區鍵，才能依排序鍵查時間範圍，因此需要事先為到期查詢設計鍵值與索引，不能只靠資料有 `BookingTime` 欄位。

回饋活動適合 DynamoDB，是因為主要讀寫方式可事先定義，且多半圍繞單筆回饋狀態；大量不同訂單的流量可以透過良好的鍵值設計分散，同一筆紀錄的並發更新也能用條件寫入控制。實際能承受多少流量，以及外部發放如何安全重試，仍取決於具體設計。

參考資料：[DynamoDB 項目讀寫](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/WorkingWithItems.html)、[分區鍵設計](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-uniform-load.html)、[熱點與限流](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/throttling-key-range-limit-exceeded-mitigation.html)、[條件寫入](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.ConditionExpressions.html)、[Query 鍵值條件](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.KeyConditionExpressions.html)。

</div>

<!-- endtab -->





<!-- tab 其他活動類型呢-->

<div class="pr-flow">

<div class="pr-lead">
<p>同一張訂單可能同時命中滿額折與給點活動。滿額折在結帳時算出折扣，建單時就能保存折後價格；給點活動即使已算出應給多少，也可能要等付款完成或指定時間才發放，之後還可能因退貨而回收。兩者都要留下當時的結果，差別在於後續是否還有一筆回饋工作要繼續執行。</p>
</div>

<figure class="pr-figure">
<div class="pr-diagram" tabindex="0" role="region" aria-label="滿額折與延後發點的處理路徑對照">
{% mermaid %}
flowchart TB
    subgraph Discount["滿額折：結果留在訂單"]
        direction TB
        CartD["購物車計算折扣"] --> OrderD["建單時保存價格與活動結果"]
        OrderD --> ReturnD["退貨時查原訂單結果"]
    end
    subgraph Reward["給點：後續仍有回饋工作"]
        direction TB
        CartR["算出應給點數"] --> RecordR["建立回饋紀錄"]
        RecordR --> WaitR["等待指定條件或時間"]
        WaitR --> GrantR["實際發放"]
        GrantR --> ReturnR["退貨時依當時結果回收"]
    end
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef waiting fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef success fill:#e8f1dc,stroke:#6f9250,color:#3f542d
    classDef result fill:#fbf7ef,stroke:#b49b78,color:#493b2d
    class CartD,OrderD,CartR,RecordR service
    class WaitR waiting
    class GrantR success
    class ReturnD,ReturnR result
{% endmermaid %}
</div>
<figcaption>圖中以延後發點為例。滿額折保存訂單結果；給點活動還要保存可供派工、重試與回收的處理進度。</figcaption>
</figure>

## 常見活動分成哪些情況？

| 活動類型 | 訂單當下做什麼 | 後續需要什麼資料或工作 |
| --- | --- | --- |
| 第 N 件改價、滿額折、會員價、多件最低價免費 | 算出價格與折扣分攤，套用到購物車或訂單 | 保存當時的訂單結果，供查單、退貨與對帳 |
| 優惠碼折抵 | 輸入優惠碼後計算可用折扣 | 保存訂單折抵結果；用碼次數或配額可能由另一套機制管理 |
| 滿額贈、滿額折加贈品 | 決定優惠與贈品，建立訂單及贈品關聯 | 保存當時贈了什麼；實體贈品的出貨仍可能有自己的流程 |
| 滿額登記 | 判斷資格並處理登記 | 另外追蹤是否完成登記與後續資格判斷 |
| 給點、給券等回饋活動 | 算出應給的回饋，建立回饋紀錄 | 等待、發放、重試、取消或退貨回收 |

對應到程式中的活動類型：第 N 件改價是 `DiscountNthPieceWithPrice`，滿額折是 `DiscountReachPriceWithRate`，會員價是 `DiscountByMemberWithPrice`，多件最低價免費是 `MultiBuyLowestPriceFree`；優惠碼折抵是 `PromoCodeKOLReachPriceAmount`；贈品相關有 `FreeGiftReachPriceWithAmount` 與 `DiscountReachPriceWithFreeGift`；滿額登記則是 `RegisterReachPrice`。

## 滿額折：算完後，結果留在訂單

假設訂單原價 1,000 元，滿額折抵 100 元。結帳時計算出折扣後，建單流程會保存當時的價格、折扣分攤與命中的活動。日後若發生退貨，應以原訂單保存的結果處理，不能直接拿已經改過的活動規則重新計算這張舊訂單。

以這條線上建單流程為例，Promotion Frontend API 的 `POST /api/basket-calculate` 負責回傳當下的活動計算結果；mweb 的 `MapToTradesOrderProcessor` 將活動與贈品資訊放進訂單資料，`CreateTradesOrderProcessor` 再建立訂單及相關關聯。一般活動、活動引擎與贈品有各自的訂單關聯資料，讓後續能查出這筆訂單當時命中什麼活動、拿到什麼折扣或贈品。這些紀錄是訂單結果的追溯依據，不等於另有一筆待發放的回饋工作。

## 給點或給券：算完後，還要追蹤發放進度

假設同一張訂單另可獲得 100 點，但規則要求三天後才發放。活動計算只確定「應給 100 點」，後續還要記錄預計發放時間、派工、實際發放結果；第五天若退貨，還要知道當初是否真的發出，以及應回收多少。

即使某筆回饋立即發放，也可能遇到事件重送、外部發放成功但本地紀錄未更新，或日後退貨的情況。因此，回饋活動需要以訂單為單位保存處理進度，讓後續工作能接著做。這是前面幾個分頁所說的 DynamoDB 回饋紀錄。

## 其他活動也可能有自己的後續狀態

優惠碼可能要管理使用次數，實體贈品可能還要出貨，`RegisterReachPrice` 也有登記時間與後續處理。它們並非完全沒有狀態；只是要追蹤的事情不同，不必都使用回饋活動的 `WaitToReward`、`Reward`、`IsRecycle` 等發放與回收欄位。

直播活動關心即時互動，定期購關心週期性建單與付款。它們也可能需要保存流程狀態，但屬於不同業務流程，不能只因為有狀態，就套用回饋活動的資料模型。

區分這些活動時，先問「優惠是否已反映在當次訂單」，再問「之後是否還要獨立發放或回收」。前者需要保存可追溯的訂單結果；後者還需要保存可供派工、重試與回收的回饋處理狀態。

</div>

<!-- endtab -->




<!-- tab DynamoDB 空值注意事項 -->

<div class="pr-flow">

## 空值寫入要先分清楚是哪一種

<div class="pr-lead">
<p>遇到空值寫入問題時，先分辨欄位沒有出現在項目中、一般屬性被存成 <code>NULL</code>，或主鍵本身缺少有效值。這三種情況的結果不同。</p>
</div>

| 寫入情況 | DynamoDB 的處理 |
| --- | --- |
| 一般屬性使用 <code>NULL</code> 型別 | 支援儲存，例如低階 API 的 <code>{"NULL": true}</code> |
| 欄位未寫入項目 | 欄位不存在；部分 SDK 或物件映射器可能省略程式中的 <code>null</code> 欄位 |
| 分區鍵或排序鍵沒有有效值 | 主鍵須是字串、數字或二進位，不能用 <code>NULL</code> 代替 |

<div class="pr-note">排查時要先確認欄位是否為主鍵，再檢查 SDK 實際送出的資料。資料庫支援 <code>NULL</code> 屬性，不代表所有欄位與每種程式物件映射方式都會得到同樣結果。</div>

參考資料：[DynamoDB 支援的資料型別](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.NamingRulesDataTypes.html)、[低階 API 的 <code>NULL</code> 表示法](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Programming.LowLevelAPI.html)、[屬性存在與空值](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.OperatorsAndFunctions.html)。

</div>

<!-- endtab -->



{% endtabs %}
