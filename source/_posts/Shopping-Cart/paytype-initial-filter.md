---
title: 購物車初步篩選可用金流
date: 2026-09-04 08:35:00
categories: Shopping-Cart
top_img: https://pub-d7e550ef212547d888a6e01348459946.r2.dev/ecom/shoppingcart/payment-availible-landing.png
cover : https://pub-d7e550ef212547d888a6e01348459946.r2.dev/ecom/shoppingcart/payment-availible-landing.png
tags:
toc:
toc_number:
comments :
---

{% tabs shopping-cart-paytype-initial-filter %}

<!-- tab 系統架構 -->

<style>
.hl-hero{position:relative;overflow:hidden;border-radius:18px;padding:36px 40px;margin:0 0 30px;background:linear-gradient(160deg,#f4f2ec 0%,#eae8dc 65%,#e3dfcd 100%);border:1.5px solid #d7cfba;}
.hl-hero::after{content:"";position:absolute;top:-30px;right:-30px;width:120px;height:120px;border-radius:50%;background:radial-gradient(circle at 35% 35%,#c9403c,#8f1c1a);opacity:.16;}
.hl-hero .hh-tag{display:inline-block;font-size:.72rem;letter-spacing:.14em;text-transform:uppercase;background:#0e3746;color:#f4f2ec;padding:5px 16px;border-radius:3px;margin-bottom:14px;font-weight:700;}
.hl-hero .hh-title{margin:0 0 10px;font-size:1.4rem;font-weight:800;color:#182228;}
.hl-hero .hh-desc{max-width:88%;font-size:.92rem;line-height:1.85;color:#3d4d52;position:relative;}
.hl-hero .hh-desc code{background:#e3ddc9;padding:1px 6px;border-radius:4px;color:#0e3746;font-weight:700;}
.hl-hero .hh-badges{display:flex;gap:10px;flex-wrap:wrap;margin-top:18px;}
.hl-hero .hh-badge{background:#f4f2ec;border:1.5px solid #b9c3c6;padding:6px 14px;border-radius:3px;font-size:.78rem;font-weight:600;color:#233238;box-shadow:0 6px 14px -8px rgba(14,55,70,.3);}
.hl-card-grid{display:flex;flex-wrap:wrap;gap:16px;margin:16px 0 26px;}
.hl-card-grid .hc-card{flex:1 1 220px;background:#f4f2ec;border:1.5px solid #d7cfba;border-radius:14px;padding:20px 20px 22px;box-shadow:0 8px 20px -8px rgba(14,55,70,.22);border-top:4px solid #0e3746;}
.hl-card-grid .hc-badge{display:inline-flex;align-items:center;justify-content:center;width:34px;height:34px;border-radius:6px;background:#0e3746;color:#f4f2ec;font-weight:800;font-size:.85rem;margin-bottom:10px;}
.hl-card-grid .hc-card.red{border-top-color:#be2623;}
.hl-card-grid .hc-card.red .hc-badge{background:#be2623;color:#fff;}
.hl-card-grid .hc-card.tan{border-top-color:#a9946a;}
.hl-card-grid .hc-card.tan .hc-badge{background:#a9946a;color:#fff;}
.hl-card-grid .hc-title{font-weight:800;font-size:.96rem;color:#182228;margin-bottom:6px;}
.hl-card-grid .hc-desc{font-size:.85rem;line-height:1.75;color:#3d4d52;}
.hl-card-grid .hc-desc code{background:#e3ddc9;padding:1px 6px;border-radius:4px;color:#0e3746;font-size:.85em;font-weight:600;}
.hl-pill-flow{display:flex;align-items:center;flex-wrap:wrap;gap:12px;margin:20px 0 26px;}
.hl-pill-flow .hp-step{background:#f4f2ec;border:2px solid #0e3746;color:#0e3746;border-radius:4px;padding:10px 20px;font-size:.85rem;font-weight:700;box-shadow:0 6px 14px -8px rgba(14,55,70,.3);}
.hl-pill-flow .hp-step.red{border-color:#be2623;color:#8f1c1a;}
.hl-pill-flow .hp-step.tan{border-color:#a9946a;color:#6b5a37;}
.hl-pill-flow .hp-arrow{color:#0e3746;font-size:1.15rem;font-weight:900;}
.hl-callout{border-radius:8px;padding:16px 20px;margin:16px 0;display:flex;gap:12px;align-items:flex-start;border:1.5px solid transparent;border-left-width:5px;}
.hl-callout .hc-icon{font-size:1.2rem;flex:none;}
.hl-callout .hc-body{flex:1;min-width:0;}
.hl-callout p{margin:0;font-size:.9rem;color:#233238;line-height:1.7;}
.hl-callout p+p,.hl-callout p+ul,.hl-callout ul+p{margin-top:10px;}
.hl-callout p strong{color:#182228;}
.hl-callout.success{background:#e3edea;border-color:#9db8b0;border-left-color:#0e3746;}
.hl-callout.warn{background:#f3ecd8;border-color:#d9c48f;border-left-color:#a9946a;}
.hl-callout.danger{background:#f8e2e0;border-color:#e29c98;border-left-color:#be2623;}
.hl-callout.info{background:#e6edef;border-color:#a9bcc2;border-left-color:#37606b;}
.hl-table-box{display:inline-block;max-width:100%;background:#f4f2ec;border:1.5px solid #d7cfba;border-radius:10px;overflow:hidden;box-shadow:0 8px 20px -8px rgba(14,55,70,.2);margin:16px 0 26px;}
.hl-table{border-collapse:collapse;width:auto;max-width:100%;}
.hl-table th{background:#0e3746;color:#f4f2ec;font-size:.8rem;text-align:left;padding:12px 16px;border-bottom:1.5px solid #d7cfba;font-weight:700;}
.hl-table td{padding:12px 16px;font-size:.85rem;color:#3d4d52;border-bottom:1px solid #e3ddc9;line-height:1.6;vertical-align:top;}
.hl-table tr:last-child td{border-bottom:none;}
.hl-table code{background:#e3ddc9;padding:1px 6px;border-radius:4px;color:#0e3746;font-size:.88em;font-weight:600;}
.hl-steps{display:flex;flex-direction:column;gap:12px;margin:16px 0 22px;}
.hl-steps .hs-card{background:#f4f2ec;border:1px solid #d7cfba;border-radius:8px;padding:14px 18px 14px 58px;position:relative;box-shadow:0 6px 16px -10px rgba(14,55,70,.2);}
.hl-steps .hs-num{position:absolute;left:14px;top:14px;width:28px;height:28px;border-radius:6px;background:linear-gradient(135deg,#154a5c,#0e3746);color:#f4f2ec;font-weight:800;font-size:.85rem;display:flex;align-items:center;justify-content:center;}
.hl-steps .hs-card.warn .hs-num{background:linear-gradient(135deg,#c9bc98,#a9946a);}
.hl-steps .hs-title{font-weight:800;font-size:.9rem;color:#182228;margin-bottom:4px;display:flex;align-items:center;gap:8px;flex-wrap:wrap;}
.hl-steps .hs-desc{font-size:.84rem;color:#3d4d52;line-height:1.65;margin:0;}
.hl-steps .hs-desc code{background:#e3ddc9;padding:1px 6px;border-radius:4px;color:#0e3746;font-size:.9em;font-weight:600;}
.hl-checklist{list-style:none;padding:0;margin:16px 0;}
.hl-checklist li{position:relative;padding:10px 0 10px 32px;border-bottom:1px dashed #d7cfba;color:#3d4d52;}
.hl-checklist li:last-child{border-bottom:none;}
.hl-checklist li::before{content:"✓";position:absolute;left:0;top:12px;width:20px;height:20px;border-radius:4px;background:#0e3746;color:#f4f2ec;font-size:.72rem;display:flex;align-items:center;justify-content:center;font-weight:900;}
.hl-checklist li code{background:#e3ddc9;padding:1px 6px;border-radius:4px;color:#0e3746;font-weight:600;}
.hl-seal-divider{display:flex;align-items:center;justify-content:center;gap:14px;margin:34px 0;color:#a9946a;}
.hl-seal-divider::before,.hl-seal-divider::after{content:"";flex:1;height:1px;background:linear-gradient(90deg,transparent,#a9946a,transparent);}
.hl-seal-divider span{font-size:1.3rem;}
</style>

<div class="hl-hero" style="padding:30px 32px;">
<div class="hh-tag">System Guide</div>
<div class="hh-title">`carts/create` 金流裁決在系統中的位置</div>
<div class="hh-desc">目前討論的是 `Shopping carts/create` 這條購物車建立與結帳前置裁決鏈。這條鏈的設計重點，是在正式進入促銷換價與金物流重算前，先完成一次可用付款方式的安全裁決。本文聚焦的位置在 `GetPayTypeIsAvaliableProcessor`，它位在 `CartCreateProcessor` 之後，位在初始金物流交集與後續重算之前。</div>
</div>

```mermaid
flowchart TB
subgraph Channel["Channel Layer"]
web["Web / mWeb"]
app["iOS / Android App"]
end
subgraph Shopping["Shopping System"]
api["CartsController.Create"]
create["CartCreateProcessor"]
filter["GetPayTypeIsAvaliableProcessor"]
end
subgraph Dependency["Policy & Config"]
risk["Risk Control Blacklist"]
cfg["PayType / PSP Config"]
end
web --> api
app --> api
api --> create
create --> filter
risk --> filter
cfg --> filter
```

<div class="hl-callout info">
<span class="hc-icon">🎯</span>
<div class="hc-body">
<p><strong>本文討論定位</strong></p>
<p>`GetPayTypeIsAvaliableProcessor` 的本質意義，是把「商店理論上可開放的付款方式」轉成「這一次請求在當下情境可安全放行的付款方式」。它輸出的 `EnablePayProfileTypeDef` 不是顯示資料，而是後續所有金物流裁決共同遵守的邊界條件。</p>
</div>
</div>

```mermaid
flowchart LR
subgraph Stage2["Stage 2：初步篩選可用金流"]
s0["All PayProfile Set"]
s1["裝置與版本過濾"]
s2["結帳情境限縮"]
s3["黑名單交集"]
s4["EnablePayProfileTypeDef"]
end
s0 --> s1 --> s2 --> s3 --> s4
```

<!-- endtab -->

<!-- tab 核心流程 -->

<div class="hl-hero">
<div class="hh-tag">GetCart Pipeline · Stage 2</div>
<div class="hh-title">`GetPayTypeIsAvaliableProcessor` 在做的事</div>
<div class="hh-desc">`carts/create` 進入促銷與金物流計算之前，會先跑一層「初步篩選可用金流」。這一層的目標只有一件事，產出當下情境真正可用的 `EnablePayProfileTypeDef`，避免前端看到可點選但其實不支援的付款方式。</div>
<div class="hh-badges">
<span class="hh-badge">目標輸出：`EnablePayProfileTypeDef`</span>
<span class="hh-badge">執行位置：GetCart Pipeline 第二階段</span>
<span class="hh-badge">核心策略：全集合起手，逐步剔除</span>
</div>
</div>

<div class="hl-callout info">
<span class="hc-icon">ℹ️</span>
<div class="hc-body">
<p><strong>初步篩選可用金流的本質邏輯</strong></p>
<p>系統先把 `PayProfileTypeDefEnum.All` 當作初始集合，再用條件判斷逐步排除不符合的金流。主體用 `^=` 做位元剔除，特定情境則直接覆寫成純白名單。這個設計同時兼顧擴充性與安全性。</p>
</div>
</div>

<div class="hl-pill-flow">
<div class="hp-step">起點：`All` 全集合</div>
<div class="hp-arrow">➜</div>
<div class="hp-step red">條件不符：`^=` 剔除位元</div>
<div class="hp-arrow">➜</div>
<div class="hp-step tan">特殊情境：直接覆寫白名單</div>
<div class="hp-arrow">➜</div>
<div class="hp-step">輸出：`EnablePayProfileTypeDef`</div>
</div>

<div class="hl-card-grid">
<div class="hc-card">
<div class="hc-badge">01</div>
<div class="hc-title">裝置與版本相容性過濾</div>
<div class="hc-desc">像 `GooglePay`、`OpenWallet`、`FamilyMartOnlinePay`、`RazerPay`、`OPPayLater` 這類支付，會先檢查來源裝置與 `AppVersion`。版本不足或環境不符時，直接從集合中剔除。</div>
</div>
<div class="hc-card red">
<div class="hc-badge">02</div>
<div class="hc-title">結帳情境限縮</div>
<div class="hc-desc">`Express` 情境直接限縮為 `CreditCardOnce | LinePay | PoyaPay`。`PayPage` 情境只保留 `CreditCardOnce`。這是後端硬裁決，不依賴前端偏好。</div>
</div>
<div class="hc-card tan">
<div class="hc-badge">03</div>
<div class="hc-title">風控黑名單交集</div>
<div class="hc-desc">若會員命中風控限制，系統把已過濾集合再與黑名單允許清單做交集，只保留風控允許的金流，風控規則優先度最高。</div>
</div>
</div>

<div class="hl-callout warn">
<span class="hc-icon">🧮</span>
<div class="hc-body">
<p><strong>三種常見運算模式</strong></p>
<ul class="hl-checklist">
<li><strong>模式 A，負向排除：</strong>以 `All` 起手，條件不符就 `^=` 關閉位元。</li>
<li><strong>模式 B，白名單條件取反：</strong>先判斷是否在支援白名單內，不在白名單才剔除，例如 `ApplePay`。</li>
<li><strong>模式 C，定向覆寫：</strong>特殊結帳情境直接寫死白名單，略過一般剔除流程。</li>
</ul>
</div>
</div>

<!-- endtab -->

<!-- tab 裝置環境與 App 版本矩陣 -->

<div class="hl-seal-divider"><span>🧭</span></div>

<div class="hl-hero" style="padding:28px 30px;">
<div class="hh-tag">Section 1</div>
<div class="hh-title">重點一：裝置環境與 App 版本矩陣</div>
<div class="hh-desc">版本檢查不是字串比較，而是把版號轉成 `System.Version` 再比對。判斷原則是 App 來源才做下限檢查，非 App 環境走安全放行。香港 `GooglePay` 另有最低版本門檻。</div>
</div>

<div class="hl-table-box">
<table class="hl-table">
<tr><th>付款方式</th><th>環境限制</th><th>版本門檻</th><th>補充</th></tr>
<tr><td><code>GooglePay</code></td><td>Android App</td><td>市場條件版號門檻</td><td>HK 額外要求版本下限</td></tr>
<tr><td><code>OpenWallet</code></td><td>App 專用</td><td><code>24.1.0</code></td><td>低於門檻剔除</td></tr>
<tr><td><code>FamilyMartOnlinePay</code></td><td>App 專用</td><td><code>24.1.0</code></td><td>低於門檻剔除</td></tr>
<tr><td><code>RazerPay</code></td><td>App 專用</td><td><code>24.2.05</code></td><td>跨境市場常見</td></tr>
<tr><td><code>OPPayLater</code></td><td>App 專用</td><td><code>25.7.0</code></td><td>台灣市場先買後付</td></tr>
</table>
</div>

<!-- endtab -->

<!-- tab `PaymentServiceProvider` 解耦與版本分流 -->

<div class="hl-hero" style="padding:28px 30px;">
<div class="hh-tag">Section 2</div>
<div class="hh-title">重點二：`PaymentServiceProvider` 解耦與版本分流</div>
<div class="hh-desc">前端看到的付款方式代碼，與後端實際收單通道已解耦。系統依 `MinAppVer` 與 `MaxAppVer` 做版本區間比對，選出對應 PSP。若設定缺失或版本異常，退回 `FallbackProfiles`，確保流程不會中斷。</div>
</div>

<div class="hl-steps">
<div class="hs-card">
<div class="hs-num">1</div>
<div class="hs-title">讀取 PSP 設定清單</div>
<p class="hs-desc">讀取 `PaymentServiceProviderProfileEntity`，每筆定義一段可用版本區間。</p>
</div>
<div class="hs-card warn">
<div class="hs-num">2</div>
<div class="hs-title">版本區間比對</div>
<p class="hs-desc">以 `currentVersion >= MinAppVer` 且 `currentVersion < MaxAppVer` 進行匹配。</p>
</div>
<div class="hs-card">
<div class="hs-num">3</div>
<div class="hs-title">無匹配時安全降級</div>
<p class="hs-desc">改用 `FallbackProfiles`，在舊版與新版 PSP 之間自動回退。</p>
</div>
</div>

<!-- endtab -->

<!-- tab `Express` 與 `PayPage` 的強制限縮 -->

<div class="hl-hero" style="padding:28px 30px;">
<div class="hh-tag">Section 3</div>
<div class="hh-title">重點三：`Express` 與 `PayPage` 的強制限縮</div>
<div class="hh-desc">這一段不是做交集，是直接覆寫。`Express` 目標是縮短授權路徑並降低中斷點。`PayPage` 目標是失敗訂單補款成功率，因此只留信用卡一次付。</div>
</div>

<div class="hl-table-box">
<table class="hl-table">
<tr><th>結帳型態</th><th>策略</th><th>最終可用金流</th></tr>
<tr><td><code>Normal</code></td><td>走一般過濾規則</td><td>依商店與環境條件決定</td></tr>
<tr><td><code>Express</code></td><td>後端直接覆寫白名單</td><td><code>CreditCardOnce</code>、<code>LinePay</code>、<code>PoyaPay</code></td></tr>
<tr><td><code>PayPage</code></td><td>後端直接覆寫白名單</td><td><code>CreditCardOnce</code></td></tr>
</table>
</div>

<!-- endtab -->

<!-- tab 黑名單會員的最終交集限制 -->

<div class="hl-hero" style="padding:28px 30px;">
<div class="hh-tag">Section 4</div>
<div class="hh-title">重點四：黑名單會員的最終交集限制</div>
<div class="hh-desc">當風控回傳黑名單允許清單後，系統對目前可用集合做最後交集。實作是把不在允許清單中的項目移除，等同把高風險金流全面關閉。</div>
</div>

<div class="hl-callout danger">
<span class="hc-icon">🛑</span>
<div class="hc-body">
<p><strong>交集實作重點</strong></p>
<p><code>enabledPayProfileTypeList.RemoveAll(i =&gt; allowPayTypeList.Contains(i) == false);</code></p>
<p>這段保證黑名單限制一定生效，即使商店本身有開啟更多付款方式也不會放行。</p>
</div>
</div>

<!-- endtab -->

<!-- tab 輸出 `EnablePayProfileTypeDef` 串接下游 -->

<div class="hl-hero" style="padding:28px 30px;">
<div class="hh-tag">Section 5</div>
<div class="hh-title">重點五：輸出 `EnablePayProfileTypeDef` 串接下游</div>
<div class="hh-desc">初步篩選結果會寫入 `context.Data.EnablePayProfileTypeDef`，後續處理器都以它為金流母集，不再回頭看未過濾的原始支付集合。</div>
</div>

<div class="hl-card-grid">
<div class="hc-card">
<div class="hc-badge">P1</div>
<div class="hc-title">分期計算處理器</div>
<div class="hc-desc">先看 `EnablePayProfileTypeDef` 是否包含分期型態，再決定是否查期數與利率資料。</div>
</div>
<div class="hc-card tan">
<div class="hc-badge">P2</div>
<div class="hc-title">初始金物流交集</div>
<div class="hc-desc">把商品頁支援的付款方式，與 `EnablePayProfileTypeDef` 再做一次對齊，剔除環境不允許組合。</div>
</div>
<div class="hc-card red">
<div class="hc-badge">P3</div>
<div class="hc-title">整車重算與最終展示</div>
<div class="hc-desc">後續重算流程在這個母集內再套入運費、促銷與折抵規則，輸出前端最終顯示清單。</div>
</div>
</div>

<div class="hl-callout success">
<span class="hc-icon">📌</span>
<div class="hc-body">
<p><strong>這份整理對應的範圍</strong></p>
<ul class="hl-checklist">
<li>第 1 點，初步篩選可用金流的集合運算邏輯。</li>
<li>第 2 點，裝置與版本相容性過濾。</li>
<li>第 3 點，`PaymentServiceProvider` 解耦與版本分流。</li>
<li>第 4 點，結帳情境限縮與黑名單交集。</li>
<li>第 5 點，結果寫回 `EnablePayProfileTypeDef` 並串接下游。</li>
</ul>
</div>
</div>

<!-- endtab -->

{% endtabs %}
