---
title: Bookingtime Job 跨日誤判
date: 2026-10-25 09:06:00
categories: PromotionReward
top_img: https://pub-d7e550ef212547d888a6e01348459946.r2.dev/ecom/promotion/promotionreward.png
cover : https://pub-d7e550ef212547d888a6e01348459946.r2.dev/ecom/promotion/promotionreward.png
toc:
toc_number:
comments :
tags:
---

{% tabs Bookingtime Job 跨日誤判 %}


<!-- tab 背景說明 -->

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
.pr-diagram.pr-compact svg{min-width:420px;max-width:720px!important}
.pr-diagram:focus-visible{outline:2px solid var(--pr-accent);outline-offset:-3px}
.pr-figure figcaption{background:var(--pr-panel);color:var(--pr-muted);padding:14px 18px;font-size:13px;border-top:1px solid var(--pr-line)}
#article-container .pr-flow code{overflow-wrap:anywhere;color:var(--pr-bear);background:var(--pr-panel)}
.pr-flow table{font-size:14px;width:auto;max-width:100%}
.pr-flow th{background:var(--pr-panel);color:var(--pr-ink)}
.pr-flow th,.pr-flow td{white-space:normal;overflow-wrap:anywhere;border-color:var(--pr-line);padding:12px}
.pr-table-scroll{max-width:100%;overflow-x:auto;border:1px solid var(--pr-line);border-radius:14px;margin:20px 0}
.pr-table-scroll table{width:100%;min-width:760px;max-width:none;margin:0}
.pr-table-scroll th,.pr-table-scroll td{overflow-wrap:normal;word-break:normal}
.pr-table-scroll code{white-space:nowrap}
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
.pr-timeline{padding:24px 30px 12px;background:var(--pr-diagram-bg)}
.pr-timeline-item{position:relative;margin-left:10px;padding:0 0 22px 30px;border-left:2px solid var(--pr-line)}
.pr-timeline-item::before{content:'';position:absolute;left:-8px;top:4px;width:14px;height:14px;border-radius:50%;background:var(--pr-accent);border:2px solid var(--pr-diagram-bg)}
.pr-timeline-item:last-child{border-left-color:transparent;padding-bottom:0}
.pr-timeline-item strong{display:block;color:var(--pr-ink)}
.pr-timeline-item p{margin:4px 0 0;color:var(--pr-muted)}
.pr-timeline-item.pr-alert::before{background:var(--pr-danger)}
[data-theme="dark"] .pr-flow{--pr-ink:#ece5d6;--pr-muted:#c6bca8;--pr-line:#575744;--pr-panel:#2b3025;--pr-accent:#b4cd92;--pr-bear:#e1b68c;--pr-diagram-bg:#242a20;--pr-danger:#db967f;--pr-danger-bg:#3b2925}
[data-theme="dark"] .pr-lead{background:linear-gradient(150deg,#4b6538,#354d29)}
[data-theme="dark"] .pr-diagram .cluster rect{fill:#343d2d!important;stroke:#8e9f78!important}
[data-theme="dark"] .pr-diagram .cluster-label span{color:#ece5d6!important}
[data-theme="dark"] .pr-note.pr-success{background:#293525}
@media(max-width:600px){.pr-lead{padding:22px 52px 22px 18px}.pr-lead::after{font-size:25px;right:12px}.pr-lead::before{right:16px;font-size:16px}.pr-note,.pr-evidence{padding:15px}.pr-diagram{padding:18px 10px}.pr-flow th,.pr-flow td{padding:9px 7px;font-size:13px}.pr-record-grid{grid-template-columns:1fr}.pr-timeline{padding:20px 16px 12px}}
</style>

<div class="pr-lead"><p>訂單在 11/05 23:54 成立，NMQ Job 到午夜後才寫入回饋紀錄，<code>UpdateTime</code> 落在 11/06 00:02。前後只隔幾分鐘，稽核卻說 <code>BookingTime</code> 差了 24 小時。Job 沒有跑上一整天；問題是正流程拿訂單日期排程，舊稽核卻可能拿 Job 寫入紀錄的日期重算。同一筆回饋，兩邊翻了不同頁的日曆。</p></div>

## NMQ Job 晚幾分鐘寫入，稽核為何多算 24 小時？

<figure class="pr-figure"><div class="pr-diagram" tabindex="0" role="region" aria-label="訂單成立時間與 NMQ Job 寫入時間跨日，正流程與舊稽核採用不同日期基準的流程圖">
{% mermaid %}
flowchart TB
    Order["11/05 23:54 訂單成立"] --> Booking["正流程：以 OrderTime 日期排程"]
    Order --> Job["午夜後：NMQ Job 處理回饋"]
    Job --> Update["11/06 00:02 寫入紀錄 UpdateTime"]
    Update --> Audit["舊稽核：以 UpdateTime 日期重算"]
    Booking --> Gap["兩個日期基準差一天"]
    Audit --> Gap
    Gap --> Alarm["稽核報差 24 小時"]
    classDef event fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef boundary fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef danger fill:#f9e9e3,stroke:#bb765f,color:#704737
    class Order,Booking event
    class Job,Update,Audit,Gap boundary
    class Alarm danger
{% endmermaid %}
</div><figcaption>案卷的 00:02 是回饋紀錄的 <code>UpdateTime</code>，不是 NMQ Job 的精確啟動時間。圖要指出的是：Job 處理與寫入落在訂單成立的隔日，而日期基準選錯，才讓稽核重算結果可能差一天。</figcaption></figure>

</div>


<!-- endtab -->


<!-- tab BookingTime 的用途 -->

<div class="pr-flow">

<div class="pr-lead"><p>先認識這場戲的主角：<code>BookingTime</code> 是回饋的預約號碼牌。Dispatcher 看號碼牌，決定何時輪到給點或給券；稽核器則翻出規則，檢查號碼牌當初有沒有印錯。同一筆時間被兩個單位拿去問不同的問題，難怪只看欄位名稱，案情還是霧煞煞。</p></div>

## 同一張號碼牌，兩個單位來查

<figure class="pr-figure"><div class="pr-diagram" tabindex="0" role="region" aria-label="BookingTime 寫入後分別用於派工查詢與稽核比對的資料流">
{% mermaid %}
flowchart TB
    Calculate["正流程計算 BookingTime"] --> Stored["DDB 儲存 BookingTimeUTC"]
    Stored -->|依日期與時間窗口查詢| Dispatcher["Dispatcher：輪到這筆了嗎？"]
    Dispatcher --> Reward["建立給點或給券 Task"]
    Stored -->|讀取實際排程時間| Auditor["稽核器：這筆約對時間了嗎？"]
    Expected["依回饋條件重算預期時間"] --> Auditor
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef waiting fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef result fill:#fbf7ef,stroke:#b49b78,color:#493b2d
    class Calculate,Stored,Expected service
    class Dispatcher,Auditor waiting
    class Reward result
{% endmermaid %}
</div><figcaption>派工像櫃檯叫號：Dispatcher 透過 <code>BookingDateUTC-BookingTimeUTC-index</code> 找出時間窗口內的紀錄。稽核像事後對帳：重算預期時間，再比對 <code>BookingTimeUTC</code>。一個問「輪到誰」，一個問「號碼對不對」。</figcaption></figure>

</div>

<!-- endtab -->


<!-- tab 正流程（給點）如何設定 BookingTime -->

<div class="pr-flow">

<div class="pr-lead"><p>先看正流程怎麼排時間。案發當時，線上訂單若走「加 N 天」分支，程式從訂單成立的 <code>OrderTime</code> 取日期，加上回饋天數，排在目標日午夜。NMQ Job 晚幾分鐘才寫入紀錄，不會讓這筆預約自動順延一天；這是第一張對照用的日曆。</p></div>

## 正流程先抽哪張時間規則？

<figure class="pr-figure"><div class="pr-diagram" tabindex="0" role="region" aria-label="線上訂單依回饋條件選擇 BookingTime 計算方式的決策流程">
{% mermaid %}
flowchart TB
    Start["線上訂單計算 BookingTime"] --> Fixed{"有指定回饋日期？"}
    Fixed -->|有| ByDate["使用 RewardDateTime"]
    Fixed -->|沒有| Status{"訂單狀態是 WaitingToShipping？"}
    Status -->|是| Immediate["保留 OrderTime"]
    Status -->|否| Days{"RewardDays 大於 0？"}
    Days -->|否| Zero["OrderTime 加上 RewardDays"]
    Days -->|是| Delayed["取 OrderTime 的日期，再加 N 天"]
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef result fill:#fbf7ef,stroke:#b49b78,color:#493b2d
    class Start,Delayed service
    class Fixed,Status,Days decision
    class ByDate,Immediate,Zero result
{% endmermaid %}
</div><figcaption>這張圖是 <code>EComOrderPromotionRewardCalculateService.GetRewardConditionSetting()</code> 的線上訂單分流。先看有沒有指定日期，再看訂單狀態與 <code>RewardDays</code>。規則有先後，不能一見「加 N 天」就直接跳到結局。</figcaption></figure>

<div class="pr-note">案發版本的算式是 <code>OrderTime.Date.AddDays(N)</code>，結果落在目標日的 00:00。<code>UpdateTime</code> 記錄的是 Job 何時寫入這筆回饋，不能代替訂單成立時間。下一幕要看稽核器是否拿同一個起點來對答案。</div>

</div>

<!-- endtab -->

<!-- tab 稽核器如何驗證 BookingTime -->

<div class="pr-flow">
<div class="pr-lead"><p>輪到稽核器出場。已出貨、指定日期、券不足、未出貨，各有各的比對規則。這次的未出貨回饋卡在一個選擇題：用訂單成立的 <code>OrderTime</code> 起算，還是用 NMQ Job 寫入紀錄時留下的 <code>UpdateTime</code> 起算？兩者剛好分居午夜兩側，選錯起點，預期日期就跟著換頁。</p></div>

## 稽核先分流，才輪到對時間

| 回饋類型 | 判斷順序 | 本文關注的入口 |
| --- | --- | --- |
| 給點 | 略過指定狀態 → 已出貨且非指定日 → 指定日期 → 其餘 | 未出貨時進入 <code>VerifyBookingTimeForUnshippedOrder</code> |
| 給券 | 略過指定狀態 → 近期券不足 → 已出貨 → 指定日期 → 其餘 | 沒有先命中券不足、已出貨或指定日時，才進入未出貨稽核 |

<div class="pr-note">有些紀錄根本不用進考場：<code>MatchWithoutQuota</code>、<code>Unmatch</code>、<code>Cancel</code>、<code>Reward</code> 會直接略過。給點與給券的關卡順序也不同，給券會先查券不足。先看入場券，再談時間算式。</div>

## 未出貨這題，誰提供起跑線？

<figure class="pr-figure"><div class="pr-diagram" tabindex="0" role="region" aria-label="未出貨 BookingTime 稽核選擇 OrderTime 或 UpdateTime 的決策流程">
{% mermaid %}
flowchart TB
    Start["未出貨 BookingTime 稽核"] --> Offline{"線下訂單？"}
    Offline -->|是| Order["以 OrderTime 加 RewardDays"]
    Offline -->|否| Initial{"未記錄延遲，且 RewardDays 大於 0？"}
    Initial -->|是| Order
    Initial -->|否| Update["以 UpdateTime 加 RewardDays"]
    Order --> Days{"RewardDays 大於 0？"}
    Update --> Days
    Days -->|是| Date["取目標日期"]
    Days -->|否| Time["保留算出的時分秒"]
    Date --> Compare["與 DDB BookingTime 比對"]
    Time --> Compare
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef result fill:#fbf7ef,stroke:#b49b78,color:#493b2d
    class Start,Order,Update service
    class Offline,Initial,Days decision
    class Date,Time,Compare result
{% endmermaid %}
</div><figcaption>線下訂單走 <code>OrderTime</code>；線上訂單若是加 N 天、又沒有改期記號，也走 <code>OrderTime</code>。其餘未出貨情境才輪到 <code>UpdateTime</code>。這張圖聚焦在稽核選擇哪個時間基準。</figcaption></figure>

<div class="pr-note pr-danger">案發時，未出貨稽核要求時間差<strong>小於 1 小時</strong>。若起算日期選到隔天，預期排程就可能整整晚一天，24 小時差異自然過不了這道門檻。</div>
</div>

<!-- endtab -->

<!-- tab 原因 -->

<div class="pr-flow">
<div class="pr-lead"><p>真正的轉折不是「Job 晚了幾分鐘」本身，而是這幾分鐘剛好跨過午夜。正流程用 11/05 的訂單日期排 <code>BookingTime</code>；舊稽核若用 NMQ Job 寫入紀錄後留下的 11/06 <code>UpdateTime</code> 重算，再把時分秒歸零，兩張日曆就差了一天。Job 沒遲到 24 小時，稽核的答案卻可能晚了 24 小時。</p></div>

## Job 跨日後寫入，為什麼讓稽核多算一天？

<figure class="pr-figure"><div class="pr-diagram" tabindex="0" role="region" aria-label="以 UTC 加八時區和加七天示意 OrderTime 與 UpdateTime 跨日後的預期日期差異">
{% mermaid %}
flowchart TB
    Order["訂單成立：11/05 23:54"] --> OrderDate["正流程：用 11/05 加 7 天"]
    OrderDate --> OrderResult["得到 11/12 00:00"]
    Update["NMQ Job 寫入紀錄：11/06 00:02"] --> UpdateDate["舊稽核：用 11/06 加 7 天"]
    UpdateDate --> UpdateResult["得到 11/13 00:00"]
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef waiting fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef danger fill:#f9e9e3,stroke:#bb765f,color:#704737
    class Order,OrderDate,OrderResult service
    class Update,UpdateDate waiting
    class UpdateResult danger
{% endmermaid %}
</div><figcaption>這張是<strong>假設加 7 天、以 UTC+8 時鐘計算的示意圖</strong>：正流程與舊稽核若分別取訂單日期、Job 寫入紀錄的日期，排到午夜後就差 24 小時。案卷未列出該筆的 <code>RewardDays</code>，所以這是機制示範，不能當成原案的實際算式。</figcaption></figure>

## 真正的案卷，留下了哪些時間？

| DDB 欄位 | 原始 UTC 值 | 換算為 UTC+8 |
| --- | --- | --- |
| <code>OrderTimeUTC</code> | 2025-11-05 15:54:52Z | 11/05 23:54:52 |
| <code>UpdateTimeUTC</code> | 2025-11-05 16:02:14Z | 11/06 00:02:14 |
| <code>BookingTimeUTC</code> | 2025-11-12 16:00:00Z | 11/13 00:00:00 |

<div class="pr-note pr-danger">案卷 VSTS 546845 記下「差異 24.00 小時」。<code>OrderTimeUTC</code> 是訂單成立時間；<code>UpdateTimeUTC</code> 是 NMQ Job 處理回饋時寫下的紀錄更新時間，不是 Job 的精確啟動時間。兩者換成 UTC+8 後確實跨過午夜。不過案卷沒有列出這筆的 <code>RewardDays</code>；若硬套上圖的「加 7 天」，算出的 UTC 值也對不上實際 <code>BookingTimeUTC</code>。機制能解釋誤判方向，不能把示意圖冒充現場計算結果。</div>


</div>

<!-- endtab -->

<!-- tab HasDelayBookingTime 判斷邏輯 -->

<div class="pr-flow">
<div class="pr-lead"><p><code>UpdateTime</code> 變晚了，不代表預約真的改期。這點很像群組訊息：有人半夜補一句話，不等於整場活動改到明天。稽核得翻 <code>RewardHistoryList</code>，找程式留下的指定改期記號，再決定要不要改用更新時間起算。沒有記號，就不能自行幫預約辦一場「改期發布會」。</p></div>

## 有更新紀錄，就算真的改期嗎？

<figure class="pr-figure"><div class="pr-diagram pr-compact" tabindex="0" role="region" aria-label="HasDelayBookingTime 依回饋歷史中的改期關鍵字判斷是否曾延遲">
{% mermaid %}
flowchart TB
    History["檢查 RewardHistoryList"] --> Keyword{"找到改期記號？"}
    Keyword -->|有| Delayed["HasDelayBookingTime = true"]
    Keyword -->|沒有| Initial["HasDelayBookingTime = false"]
    Delayed --> Update["線上未出貨：可走 UpdateTime 基準"]
    Initial --> Order["線上加 N 天：走 OrderTime 基準"]
    classDef service fill:#edf3e4,stroke:#7a9a5e,color:#3f542d
    classDef decision fill:#f5e8d3,stroke:#a06a41,color:#65442c
    classDef result fill:#fbf7ef,stroke:#b49b78,color:#493b2d
    class History service
    class Keyword decision
    class Delayed,Initial,Update,Order result
{% endmermaid %}
</div><figcaption><code>HasDelayBookingTime()</code> 只認既有程式寫入的固定字串 <code>設定BookingDataTime</code>。找到記號也還沒直接大結局；實際時間基準仍要依線上或線下，以及 <code>RewardDays</code> 的分支順序決定。</figcaption></figure>


</div>

<!-- endtab -->

<!-- tab 各情境演練 -->

<div class="pr-flow">
<div class="pr-lead"><p>分支一多，規則就像口袋裡的耳機線，拿出來自動打結。拆成三個問題就好：這筆要不要稽核？拿哪個時間起算？最後怎麼比？表格是攻略地圖，但關卡有順序，不能直接傳送到喜歡的結局。</p></div>

## 八種情境，別拿錯攻略

<div class="pr-table-scroll" role="region" tabindex="0" aria-label="稽核情境、預期時間來源與比對方式對照表，可水平捲動">
<table>
<colgroup><col style="width:34%"><col style="width:28%"><col style="width:38%"></colgroup>
<thead><tr><th>情境</th><th>預期時間的來源</th><th>稽核如何比</th></tr></thead>
<tbody>
<tr><td><code>MatchWithoutQuota</code>、<code>Unmatch</code>、<code>Cancel</code>、<code>Reward</code></td><td>不計算</td><td>直接略過</td></tr>
<tr><td>線上未出貨、加 N 天、沒有改期記號</td><td><code>OrderTime + RewardDays</code></td><td>取訂單日期，加 N 天後與 <code>BookingTime</code> 比</td></tr>
<tr><td>線上未出貨、加 N 天、有改期記號</td><td><code>UpdateTime + RewardDays</code></td><td>取更新日期，加 N 天後與 <code>BookingTime</code> 比</td></tr>
<tr><td>線下未出貨</td><td><code>OrderTime + RewardDays</code></td><td>加 N 天時取日期並比對 <code>BookingTime</code></td></tr>
<tr><td>未出貨、<code>RewardDays = 0</code></td><td>線上以 <code>UpdateTime</code>；線下以 <code>OrderTime</code></td><td>保留時分秒，比對 <code>BookingTime</code></td></tr>
<tr><td>指定日期回饋</td><td>活動設定的 <code>RewardDateTime</code></td><td>與 <code>BookingTime</code> 精確比對</td></tr>
<tr><td>已出貨回饋</td><td>最後出貨時間加回饋天數，且不早於活動有效日</td><td>比對 <code>BookingDate</code> 的日期</td></tr>
<tr><td>券不足重試</td><td>依 <code>UpdateTime</code> 排到當天 22:00、當天或隔天 10:00</td><td>與 <code>BookingTime</code> 精確比對</td></tr>
</tbody></table>
</div>

<div class="pr-note">攻略表只列出各關卡怎麼算，沒有開放任意跳關。給點遇到「已出貨且指定日期」，會先走指定日期稽核；給券則先查券不足，再查已出貨。這次警報要看的，是未出貨分支有沒有誤把 Job 寫入時間當成訂單時間。</div>

## 不同關卡，不能共用同一把尺

<div class="pr-record-grid" role="group" aria-label="BookingTime 稽核三種比對方式">
<div class="pr-record"><strong>案發版本：未出貨</strong><p>時間差必須小於 1 小時。剛好差 1 小時也不算過關。</p></div>
<div class="pr-record"><strong>指定日期</strong><p>照活動設定的日期時間比對 <code>BookingTime</code>，不靠未出貨分支的時間基準。</p></div>
<div class="pr-record"><strong>券不足重試</strong><p>依重試規則排出時間，再與 <code>BookingTime</code> 精確比對。</p></div>
</div>
</div>

<!-- endtab -->

<!-- tab 案件 -->

<div class="pr-flow">
<div class="pr-lead"><p>最後把鏡頭拉遠：訂單在午夜前成立，NMQ Job 在午夜後處理並寫入回饋紀錄。真正讓稽核報出 24 小時差異的，是正流程與舊稽核可能採用了不同的起算日期。後來的修正提交把未延遲情境改回 <code>OrderTime</code>；但提交不等於已在正式環境上線，這段不能擅自替劇情加彩蛋。</p></div>

<div class="pr-record-grid" role="group" aria-label="跨日稽核案件摘要">
<div class="pr-record"><strong>案發現場</strong><p>HK-Prod 訂單先成立，NMQ Job 跨日後寫入回饋紀錄。</p></div>
<div class="pr-record"><strong>警報喊了什麼</strong><p><code>BookingTime</code> 與預期不符，差異為 24.00 小時。</p></div>
<div class="pr-record"><strong>後來修了哪裡</strong><p>未延遲的線上加 N 天回饋，稽核改用 <code>OrderTime</code> 起算。</p></div>
</div>

## 案發到修正，時間線說了什麼？

<figure class="pr-figure"><div class="pr-timeline" role="list" aria-label="訂單成立、NMQ Job 寫入回饋紀錄、稽核告警與修正提交的時間線">
<div class="pr-timeline-item" role="listitem"><strong>2025/11/05 23:54，UTC+8</strong><p>訂單成立，午夜倒數中。</p></div>
<div class="pr-timeline-item" role="listitem"><strong>2025/11/06 00:02，UTC+8</strong><p>NMQ Job 處理後寫入回饋紀錄，<code>UpdateTime</code> 落在隔日；這不是 Job 啟動時間。</p></div>
<div class="pr-timeline-item pr-alert" role="listitem"><strong>原始稽核警報</strong><p>工作項目記錄 <code>BookingTime</code> 與預期相差 24 小時。警報先響，原因還得查。</p></div>
<div class="pr-timeline-item" role="listitem"><strong>2026/01/27，稽核修正提交</strong><p>未延遲的線上加 N 天回饋，稽核改用 <code>OrderTime</code> 起算。</p></div>
</div><figcaption>警報記在 VSTS 546845；稽核修正對應 Git 提交 <code>0dadda61</code>。提交日期是程式碼履歷，不是正式環境的上線打卡紀錄。</figcaption></figure>

<div class="pr-note pr-success">已能確定的修正方向：初始且沒有延遲記號的線上加 N 天回饋，正流程與稽核都該以 <code>OrderTime</code> 起算。Job 何時寫入紀錄，不能悄悄把原本的訂單日期換掉。至少這次，查票與排班終於拿同一份時刻表。</div>

<details class="pr-evidence"><summary>要完整復盤，案卷還缺哪些拼圖？</summary>

1. 該筆回饋實際設定的 <code>RewardDays</code>，以及適用的回饋條件。
2. 應用程式讀入三個 UTC 欄位時的 <code>DateTime</code> 時區轉換與日期基準。
3. 修正版本在 HK-Prod 的實際上線時間與上線後的稽核結果。
4. 該筆 NMQ Job 的啟動時間與完整執行日誌；目前只有紀錄寫入時留下的 <code>UpdateTime</code>。

在這些資料補齊之前，能確定的是「24 小時警報」和「修正程式選擇的時間基準」。加 7 天只是示意算式，不能假扮成這筆訂單的實測結果。故事可以說得好懂，證據邊界還是要畫清楚。

</details>

<div class="pr-note"><strong>補充：現行版本的分桶</strong><p>2026/08/31 的提交 <code>38b01eb9</code> 才加入 <code>RewardBookingTimeAllocator</code>，將加 N 天的目標日分到 00:00、01:00 或 02:00。現行未出貨稽核對此使用 <code>Math.Max(bucketCount, 3)</code> 小時的容忍值。這是後續版本的規則，不用來重算 2025 年 11 月的警報。</p></div>
</div>

<!-- endtab -->

{% endtabs %}
