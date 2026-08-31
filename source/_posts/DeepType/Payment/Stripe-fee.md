---
title: Stripe-Fee
date: 2026-07-10 11:09:00
categories: Payment
top_img: https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/stripe/app_fee_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/stripe/app_fee_landing.png
toc:
toc_number:
comments :
tags:
---

<style>
.pf-hero {
background: linear-gradient(135deg, #4a3fb0, #635bff 70%); color: #fff;
padding: 34px 32px 28px; border-radius: 16px; margin: 0 0 26px;
}
.pf-hero .pf-hero-eyebrow {
font-size: 0.78rem; letter-spacing: 3px; text-transform: uppercase; opacity: 0.85;
display: flex; align-items: center; gap: 8px;
}
.pf-hero .pf-hero-mark {
display: inline-flex; align-items: center; justify-content: center;
width: 20px; height: 20px; border-radius: 6px; background: #fff; color: #635bff;
font-weight: 800; font-size: 0.72rem; font-family: Consolas, monospace; letter-spacing: 0;
}
.pf-hero h1 { margin: 10px 0 8px; font-size: 1.5rem; color: #fff; }
.pf-hero p { margin: 0; opacity: 0.95; max-width: 760px; line-height: 1.8; font-size: 0.92rem; }
.pf-hero-meta { display: flex; gap: 22px; margin-top: 18px; flex-wrap: wrap; font-size: 0.82rem; opacity: 0.92; }
.pf-hero-meta span strong { display: block; font-size: 0.68rem; opacity: 0.75; font-weight: 400; letter-spacing: 1px; text-transform: uppercase; }
.pf-hero-meta code { background: rgba(255,255,255,0.18); color: #fff; padding: 2px 7px; border-radius: 4px; font-size: 0.9em; }

.pf-badge { display: inline-block; padding: 2px 10px; border-radius: 20px; font-size: 0.72rem; font-weight: 700; }
.pf-badge.ok { background: #e7f7ee; color: #1f8a52; }
.pf-badge.wait { background: #fdf3dd; color: #b7791f; }
.pf-badge.fail { background: #fdecea; color: #c0392b; }
.pf-badge.info { background: #e8f1fb; color: #2563a8; }

.pf-api-chip {
display: inline-flex; align-items: center; gap: 8px; background: #2b2a3a; color: #fff;
padding: 6px 12px; border-radius: 8px; font-family: Consolas, monospace; font-size: 0.82rem; margin: 6px 0;
}
.pf-api-chip .pf-api-method {
background: #635bff; color: #fff; font-weight: 700; padding: 1px 8px; border-radius: 4px; font-size: 0.74rem; font-family: inherit;
}

.pf-card-chips { display: flex; gap: 8px; flex-wrap: wrap; margin: 10px 0 18px; }
.pf-card-chip {
display: inline-flex; align-items: center; gap: 6px; padding: 5px 12px; border-radius: 20px;
font-size: 0.8rem; font-weight: 600; border: 1px solid #e3e0f5; background: #fff; color: #2b2a3a;
}
.pf-card-chip .pf-cc-dot { width: 8px; height: 8px; border-radius: 50%; display: inline-block; }
.pf-card-chip.visa .pf-cc-dot { background: #1a1f71; }
.pf-card-chip.mc .pf-cc-dot { background: #eb001b; }
.pf-card-chip.jcb .pf-cc-dot { background: #0e7c3a; }
.pf-card-chip.amex .pf-cc-dot { background: #2563a8; }
.pf-card-chip.unknown { border-style: dashed; color: #5c5a72; }
.pf-card-chip.unknown .pf-cc-dot { background: #b0aec7; }
</style>


{% tabs Stripe_fee%}

<!-- tab 費用機制總覽-->

<style>
.pf-note {
border-radius: 10px; padding: 14px 18px; margin: 16px 0; font-size: 0.9rem;
border-left: 4px solid #2563a8; background: #e8f1fb; color: #1c3f63; line-height: 1.8;
}
.pf-note .pf-n-label { font-weight: 700; margin-bottom: 4px; display: block; }
.pf-note code { background: #eeecff; color: #4a3fb0; padding: 1px 6px; border-radius: 4px; font-size: 0.85em; }
.pf-note.warn { border-left-color: #b7791f; background: #fdf3dd; color: #6b4c10; }
.pf-note.danger { border-left-color: #c0392b; background: #fdecea; color: #7d281d; }
.pf-note.ok { border-left-color: #1f8a52; background: #e7f7ee; color: #155c3a; }

.pf-table-wrap { display: inline-block; max-width: 100%; overflow-x: auto; margin: 14px 0 22px; border: 1px solid #e3e0f5; border-radius: 10px; }
.pf-table-wrap table { width: auto; border-collapse: collapse; font-size: 0.88rem; margin: 0 !important; }
.pf-table-wrap thead th { background: #4a3fb0; color: #fff; text-align: left; padding: 10px 14px; font-weight: 600; }
.pf-table-wrap tbody td { padding: 9px 14px; border-top: 1px solid #e3e0f5; vertical-align: top; color: #2b2a3a; }
.pf-table-wrap tbody tr:nth-child(even) { background: #faf9ff; }
.pf-table-wrap code { background: #eeecff; color: #4a3fb0; padding: 1px 6px; border-radius: 4px; font-size: 0.85em; }
.pf-table-wrap .pf-cell-sub { display: block; font-size: 0.78rem; color: #5c5a72; margin-top: 2px; }

.pf-flow-diagram { background: #fff; border: 1px solid #e3e0f5; border-radius: 12px; padding: 22px 24px; margin: 16px 0 24px; }
.pf-flow-step {
display: flex; align-items: flex-start; gap: 14px; padding: 10px 4px;
border-left: 2px dashed #e3e0f5; margin-left: 14px; padding-left: 20px; position: relative;
}
.pf-flow-step::before {
content: ""; position: absolute; left: -7px; top: 18px; width: 12px; height: 12px;
border-radius: 50%; background: #635bff; border: 2px solid #fff; box-shadow: 0 0 0 2px #635bff;
}
.pf-flow-step.branch::before { background: #b7791f; box-shadow: 0 0 0 2px #b7791f; }
.pf-flow-step.end::before { background: #1f8a52; box-shadow: 0 0 0 2px #1f8a52; }
.pf-flow-step .pf-fs-body { font-size: 0.92rem; color: #2b2a3a; }
.pf-flow-step .pf-fs-tag {
display: inline-block; font-size: 0.68rem; letter-spacing: 1px; font-weight: 700;
background: #eeecff; color: #4a3fb0; padding: 2px 8px; border-radius: 20px; margin-bottom: 4px;
}
.pf-flow-step .pf-fs-tag.branch { background: #fdf3dd; color: #b7791f; }
.pf-flow-step .pf-fs-tag.end { background: #e7f7ee; color: #1f8a52; }
.pf-flow-step code { background: #eeecff; color: #4a3fb0; padding: 1px 6px; border-radius: 4px; font-size: 0.85em; }

.pf-fee-flow { background: #fff; border: 1px solid #e3e0f5; border-radius: 12px; padding: 20px; margin: 16px 0 24px; }
.pf-fee-box { border: 1px solid #e3e0f5; border-radius: 8px; padding: 12px 14px; margin: 8px 0; font-size: 0.87rem; background: #fcfcff; color: #2b2a3a; line-height: 1.7; }
.pf-fee-box .pf-fb-title { font-weight: 700; color: #4a3fb0; margin-bottom: 4px; display: block; }
.pf-fee-box code { background: #eeecff; color: #4a3fb0; padding: 1px 6px; border-radius: 4px; }
.pf-fee-box.final { background: #eeecff; border-color: #635bff; }
.pf-fee-arrow { text-align: center; color: #635bff; font-size: 1.1rem; margin: 2px 0; }

.pf-stage-grid { display: flex; gap: 18px; margin: 16px 0 22px; flex-wrap: wrap; }
.pf-stage-card { flex: 1; min-width: 260px; background: #fff; border: 1px solid #e3e0f5; border-radius: 10px; overflow: hidden; box-shadow: 0 1px 4px rgba(70,60,150,0.06); }
.pf-stage-card .pf-stage-head { background: #4a3fb0; color: #fff; padding: 10px 16px; font-weight: 700; font-size: 0.92rem; }
.pf-stage-card .pf-stage-head .pf-stage-sub { display: block; font-weight: 400; font-size: 0.72rem; opacity: 0.85; margin-top: 2px; letter-spacing: 0.5px; }
.pf-stage-card .pf-stage-body { padding: 14px 16px 16px; font-size: 0.88rem; color: #2b2a3a; }
.pf-stage-card .pf-stage-body ul, .pf-stage-card .pf-stage-body ol { margin: 8px 0; padding-left: 20px; }
.pf-stage-card .pf-stage-body li { margin: 4px 0; line-height: 1.7; }
.pf-stage-card code { background: #eeecff; color: #4a3fb0; padding: 1px 6px; border-radius: 4px; font-size: 0.85em; }

.pf-branch-grid { display: flex; gap: 16px; margin: 16px 0 22px; flex-wrap: wrap; }
.pf-branch-card { flex: 1; min-width: 200px; border-radius: 10px; padding: 12px 14px; font-size: 0.86rem; border: 1px solid #e3e0f5; background: #fcfcff; color: #2b2a3a; }
.pf-branch-card .pf-bc-code { font-family: Consolas, monospace; font-weight: 700; font-size: 0.92rem; color: #4a3fb0; display: block; margin-bottom: 4px; }
.pf-branch-card.muted { background: #f5f4fb; border-color: #dedbf0; color: #5c5a72; }
.pf-branch-card.warn { background: #fdf3dd; border-color: #f0dba8; color: #6b4c10; }
.pf-branch-card.warn .pf-bc-code { color: #b7791f; }
.pf-branch-card code { background: #eeecff; color: #4a3fb0; padding: 1px 6px; border-radius: 4px; font-size: 0.85em; }

.pf-term { background: #221f36; color: #dcd9f5; border-radius: 12px; padding: 18px 22px 22px; margin: 16px 0 24px; box-shadow: 0 6px 20px rgba(40,32,90,0.22); overflow-x: auto; }
.pf-term .pf-term-bar { display: flex; align-items: center; gap: 6px; margin-bottom: 14px; }
.pf-term .pf-term-dot { width: 10px; height: 10px; border-radius: 50%; display: inline-block; }
.pf-term .pf-term-dot.r { background: #ff5f56; }
.pf-term .pf-term-dot.y { background: #ffbd2e; }
.pf-term .pf-term-dot.g { background: #27c93f; }
.pf-term .pf-term-title { margin-left: 8px; font-size: 0.74rem; color: #9b96c9; letter-spacing: 0.5px; }
.pf-term pre.pf-term-pre { margin: 0 !important; background: transparent !important; padding: 0 !important; border: none !important; box-shadow: none !important; font-family: Consolas, monospace !important; font-size: 0.84rem !important; line-height: 1.9 !important; white-space: pre !important; overflow-x: auto !important; color: #f3f1ff !important; text-shadow: none !important; }
.pf-term .tstep { color: #c7b8ff !important; font-weight: 700; }
.pf-term .tbadge { color: #ffd166 !important; font-weight: 700; }
.pf-term .tdb { color: #7fd1ff !important; font-weight: 600; }
.pf-term .tfinal { color: #7ee8a8 !important; font-weight: 700; }
.pf-term .tok { color: #ffffff !important; font-weight: 700; }
.pf-term .tdim { color: #b8b3d9 !important; }
</style>

Stripe 請款的時候，其實一次要跟商家收兩筆錢：一筆是<strong>平台的系統使用費</strong>，另一筆是<strong>信用卡本身的手續費</strong>。這兩筆錢的費率怎麼查、什麼時候確定、又是怎麼一起送給 Stripe 申報的，就是這篇要拆解的內容


## 兩種費用的差別

<div class="pf-table-wrap">
<table>
<thead><tr><th style="width:22%">費用類型</th><th>說明</th></tr></thead>
<tbody>
<tr><td><strong>系統使用費</strong><span class="pf-cell-sub"><code>SalesOrder</code></span></td><td>平台向商家收取的服務費，費率與商品分類、供應商有關</td></tr>
<tr><td><strong>金流手續費</strong><span class="pf-cell-sub"><code>PayProfile</code></span></td><td>Stripe 信用卡交易手續費，費率與「發卡國家＋卡片品牌」的組合有關</td></tr>
</tbody>
</table>
</div>


## 費用取得的時間點（在 Pipeline 中的位置）

這兩種費率不是一開始就知道，而是隨著下單 Pipeline 一步步往下跑，才逐漸被鎖定、查出來、最後算成金額：

<div class="pf-flow-diagram">
<div class="pf-flow-step">
<div class="pf-fs-body"><span class="pf-fs-tag">Pipeline 第 19 道</span><br><code>ArrangePayTypeExpressInfoProcessor</code> — 設定卡片品牌與發卡國代碼（這兩個值就是後面手續費計算的依據）</div>
</div>
<div class="pf-flow-step branch">
<div class="pf-fs-body"><span class="pf-fs-tag branch">Pipeline 第 26 道</span><br><code>GetOrderProcessingFeeProcessor</code> — 分別載入「系統使用費」與「金流手續費」兩種費率</div>
</div>
<div class="pf-flow-step end">
<div class="pf-fs-body"><span class="pf-fs-tag end">Pipeline 第 30 道</span><br><code>ThirdPartyPayApiProcessor</code> — 呼叫費用計算方法算出最終要向 Stripe 申報的手續費金額，隨請款請求一併送出</div>
</div>
</div>

### Brand / 發卡國代碼的資料來源（依情境不同）

手續費要算得準，關鍵就在「卡片品牌」與「發卡國家」這兩個值有沒有抓對。金流手續費率正是依「發卡國家＋卡片品牌」的組合去查，常見的卡片品牌如下：

<div class="pf-card-chips">
<span class="pf-card-chip visa"><span class="pf-cc-dot"></span>Visa</span>
<span class="pf-card-chip mc"><span class="pf-cc-dot"></span>Mastercard</span>
<span class="pf-card-chip jcb"><span class="pf-cc-dot"></span>JCB</span>
<span class="pf-card-chip amex"><span class="pf-cc-dot"></span>American Express</span>
<span class="pf-card-chip unknown"><span class="pf-cc-dot"></span>未知（全新卡查詢前）</span>
</div>

但這兩個值在不同付款情境下，來源完全不一樣：

<div class="pf-table-wrap">
<table>
<thead><tr><th>使用情境</th><th>資料來源</th></tr></thead>
<tbody>
<tr><td>記住信用卡（舊卡復用）</td><td>從 <code>PayTypeExpress</code> 資料表讀取後設定</td></tr>
<tr><td>定期購自動成單</td><td>從既有的定期購結帳資訊解析後設定</td></tr>
<tr><td>Google Pay / Apple Pay</td><td>即時向 Stripe API 查詢取得</td></tr>
<tr><td>新信用卡（首次付款）</td><td>付款完成後才能取得，因此計算手續費時暫時視為未知</td></tr>
</tbody>
</table>
</div>

<!-- endtab -->


<!-- tab ArrangePayTypeExpressInfoProcessor-->


前面提到 Pipeline 第 19 道會設定卡片品牌與發卡國代碼，但這一步<strong>不是「每次結帳都會設定」</strong>，而是有明確的觸發條件。這個 Processor 真正的角色是：把「使用者選擇的已記住付款工具」還原成付款所需的機敏資料，只有在使用者<strong>明確選擇快速結帳／記住的卡片</strong>時才會動作。

<div class="pf-note">
<span class="pf-n-label">💡 一句話定位</span>
它是 <code>identity</code> → <code>PayTypeExpress</code> 記錄 → 機敏付款資訊的還原器，跟「這次是不是輸入全新卡號」完全無關；有沒有觸發，取決於前端這次結帳有沒有帶 <code>identity</code> 上來。
</div>

<div class="pf-stage-grid">
<div class="pf-stage-card">
<div class="pf-stage-head">組別一：CheckoutDotCom / Stripe / KPay<span class="pf-stage-sub">快速結帳 / 記住信用卡專用</span></div>
<div class="pf-stage-body">
<ol>
<li>前提：<code>context.ThirdPartyPaymentInfo.ExtendInfo</code> 裡必須帶有 <code>identity</code>（代表前端已指定「用第幾張記住的卡」）</li>
<li>用 <code>identity</code> 查出對應的 <code>PayTypeExpress</code> 記錄，解密還原出機敏付款資訊（如 Stripe 的 <code>customer_id</code> / <code>payment_method</code>）寫回 context</li>
<li>只有 Stripe 這個分支才會額外多做一件事：把 <code>Brand</code> / <code>IssueCountryCode</code> 一併設進 <code>context.CreditCardInfo</code>，專門為了讓後面第 26 道能查到正確的金流手續費率</li>
</ol>
</div>
</div>
<div class="pf-stage-card">
<div class="pf-stage-head">組別二：CreditCardOnce / CreditCardInstallment<span class="pf-stage-sub">本土一次性 / 分期信用卡（限 Nine1Payment 金流）</span></div>
<div class="pf-stage-body">
<ol>
<li>邏輯類似，同樣需要 <code>identity</code> 才會觸發</li>
<li>從 <code>PayTypeExpress</code> 撈出 <code>CardToken</code> / <code>BankCode</code></li>
<li>處理的是「發卡行資訊可能過期需要更新」的問題，<strong>跟手續費計算無關</strong>，服務對象也不是 Stripe</li>
</ol>
</div>
</div>
</div>


<!-- endtab -->

<!-- tab GetOrderProcessingFeeProcessor -->

`GetOrderProcessingFeeProcessor` 就是前面提到的 <strong>Pipeline 第 26～27 道</strong>，它唯一的工作是：把「系統使用費」與「金流手續費」這兩種費率查出來，放進 <code>context.ProcessingFeeInfo</code>，讓後面第 30 道 <code>ThirdPartyPayApiProcessor</code> 能直接拿來算 <code>application_fee_amount</code>。它會不會執行、抓到的是不是正確的值，是這篇最值得深挖的地方。

<div class="pf-note">
<span class="pf-n-label">💡 一句話定位</span>
它是「兩種費率的查詢器」，只認 <code>PayProfileType</code> 決定要不要查、以及查哪一種公式，<strong>完全不判斷這次結帳是不是快速結帳、記不記得卡、是不是全新卡</strong>。
</div>

### 第一道關卡：只有 HK 環境才會執行 <span class="pf-badge info">HK Only</span>

```csharp
// GetOrderProcessingFeeProcessor.Process()
if (SettingHelper.DefaultCountry != "HK")
{
    return;
}
```

其他地區的商店，這個 Processor 一進來就直接跳過，<code>context.ProcessingFeeInfo</code> 全程維持空值。

### 兩條各自獨立的查詢邏輯

<div class="pf-stage-grid">
<div class="pf-stage-card">
<div class="pf-stage-head">GetSalesOrderProcessingFee<span class="pf-stage-sub">查「系統使用費」→ context.ProcessingFeeInfo.SalesOrder</span></div>
<div class="pf-stage-body">
<ol>
<li><code>CreditCardOnce_Stripe</code> / <code>GooglePay</code> / <code>ApplePay</code> 才會查，其餘付款方式回傳 <code>null</code></li>
<li>邏輯很單純：拿購物車第一個非贈品的商品分類 <code>SourceCategoryId</code>，查該分類 + 供應商對應的系統使用費率</li>
<li>這條路徑跟卡片品牌／發卡國完全無關，<strong>不會受全新卡或記住卡影響</strong></li>
</ol>
</div>
</div>
<div class="pf-stage-card">
<div class="pf-stage-head">GetPayProfileProcessingFee<span class="pf-stage-sub">查「金流手續費」→ context.ProcessingFeeInfo.PayProfile</span></div>
<div class="pf-stage-body">
<ol>
<li><code>CreditCardOnce_Stripe</code>：直接拿<strong>當下</strong>的 <code>context.CreditCardInfo.Brand</code> / <code>IssueCountryCode</code> 去查——這兩個值是不是正確</li>
<li><code>GooglePay</code> / <code>ApplePay</code>：呼叫 <code>GetBrandAndCountry()</code>，即時打 Stripe API（<code>RetrievePaymentMethod</code>）用 <code>payment_method</code> 查出<strong>真實</strong>卡片品牌與發卡國</li>
<li>其餘付款方式回傳 <code>null</code></li>
</ol>
<div class="pf-api-chip"><span class="pf-api-method">GET</span> /v1/payment_methods/{id}</div>
</div>
</div>
</div>

### 三種 PayProfileType 分支對照

<div class="pf-branch-grid">
<div class="pf-branch-card">
<span class="pf-bc-code">CreditCardOnce_Stripe</span> <span class="pf-badge wait">資料待確認</span>
兩種費率都查。金流手續費用「context 當下的 Brand / Country」，<strong>不會主動去問 Stripe 要真實資料</strong>
</div>
<div class="pf-branch-card">
<span class="pf-bc-code">GooglePay / ApplePay</span> <span class="pf-badge ok">資料最可靠</span>
兩種費率都查。金流手續費<strong>即時呼叫 Stripe API</strong> 取得真實 Brand / Country，資料最可靠
</div>
<div class="pf-branch-card muted">
<span class="pf-bc-code">其他付款方式</span> <span class="pf-badge fail">不計費</span>
兩個方法都直接回傳 <code>null</code>，<code>ProcessingFeeInfo</code> 維持空值，後面也不會計入 <code>application_fee_amount</code>
</div>
</div>


<!-- endtab -->

<!-- tab 全新卡的 Brand/Country 從哪來-->

前兩個 tab 一直圍繞著一個假設：全新卡（沒有 identity）在後端 Pipeline 裡「查不到」正確的卡片品牌／發卡國。實際追下去才發現，這個假設<strong>只對了一半</strong>——後端的確沒有主動去查，但<strong>前端在送出訂單之前，就已經先把真實資料準備好了</strong>。


## 送單前的隱藏一步：CardValidate

<div class="pf-flow-diagram">
<div class="pf-flow-step">
<span class="pf-fs-tag">前端 SubmitOrder()</span>
<div class="pf-fs-body">判斷條件：<code>!PayProcessData.HasCreditCard</code>（全新卡） <strong>且</strong> <code>PayProfileType === CreditCardOnce_Stripe</code>，才會進入下一步；記住的卡（HasCreditCard = true）不會走這條路</div>
</div>
<div class="pf-flow-step branch">
<span class="pf-fs-tag branch">前端 ValidateService.CardValidate()</span>
<div class="pf-fs-body">把使用者剛輸入的卡號、到期日、CVC 包成 <code>CardOptions</code>，POST 到後端 <code>/CreditCard/Validate</code></div>
</div>
<div class="pf-flow-step branch">
<span class="pf-fs-tag branch">後端 CreditCardController.Validate → StripeService.CreditCardValidation</span>
<div class="pf-fs-body">直接拿卡片資料呼叫<strong>真正的 Stripe 官方 API</strong>，換回這張卡真實的 <code>brand</code>、<code>country</code></div>
<div class="pf-api-chip"><span class="pf-api-method">POST</span> /v1/payment_methods</div>
</div>
<div class="pf-flow-step branch">
<span class="pf-fs-tag branch">前端拿到回應</span>
<div class="pf-fs-body">驗證成功後寫回 <code>PayProcessData.CreditCardInfo.Brand</code> / <code>IssueCountryCode</code>，這兩個值現在是<strong>真實卡片資訊</strong>，不再是預設值</div>
</div>
<div class="pf-flow-step end">
<span class="pf-fs-tag end">前端 AfterCheck() → Send()</span>
<div class="pf-fs-body">把整包 <code>PayProcessData</code>（也就是後端的 <code>PayProcessContextEntity</code>）送給 <code>CompleteForNewCartV2</code>，Pipeline 開始跑第 1 道時，<code>context.CreditCardInfo</code> 就已經帶著真實資料進來了</div>
</div>
</div>


<!-- endtab -->



<!-- tab 費率怎麼實際計算系統使用費-->

拿到兩種費率之後，接下來就是把它們套進公式，算出一個要跟 Stripe 申報的 <code>application_fee_amount</code>。這筆金額由三個部分加總而成：

<div class="pf-fee-flow">
<div class="pf-fee-box">
<span class="pf-fb-title">① 商品系統使用費</span>
所有商品（實付金額扣除購物金攤提後）× 系統使用費率，加總
</div>
<div class="pf-fee-box">
<span class="pf-fb-title">② 運費系統使用費</span>
所有運費（實付金額扣除購物金攤提後）× 系統使用費率，加總
</div>
<div class="pf-fee-box">
<span class="pf-fb-title">③ 金流手續費（僅 <code>Custom</code> 帳戶類型才會加上）</span>
總支付金額 × 金流手續費率 + 金流固定手續費
</div>
<div class="pf-fee-arrow">↓</div>
<div class="pf-fee-box final">
<span class="pf-fb-title">最終金額 application_fee_amount</span>
① ＋ ② ＋ ③，四捨五入到小數第 2 位；若計算結果大於 0 但小於 0.01，強制設為 0.01
</div>
</div>

<div class="pf-note">
<span class="pf-n-label">💡 PMW 收到金額後的用途</span>
PMW 會把算好的手續費金額（換算成最小貨幣單位、取整數）放進 Stripe 的請款請求裡的 <code>application_fee_amount</code> 欄位。Stripe 扣款成功後，這筆金額會直接從交易款項中撥給 91APP 的主帳號，作為平台向商家收取的服務費。
</div>



<!-- endtab -->



<!-- tab 付款成功後：費用資料如何存下來、如何交給 ERP-->


付款成功後，系統會把這次交易用到的費率資訊（付款流程類型、帳戶類型、卡片品牌、發卡國家、各項費率與固定費用）整批序列化，存進這筆訂單的第三方付款紀錄裡，供後續 ERP 轉單時讀取使用。實際負責這件事的是 <code>StripePayChannelService.GetTradesOrderThirdPartyPaymentInfo()</code>，它會把當次交易用到的費率組成一段 JSON，寫進 <code>TradesOrderThirdPartyPayment.Info</code> 欄位：

```json
{
    "PaymentFlow": "DirectCharge|DestinationCharge",
    "AccountType": "Standard|Custom|Express",
    "SubAccount": "acct_xxx",
    "CardBrand": "Visa|MasterCard|...",
    "CardIssueCountry": "HK|TW|...",
    "FeeRate": 0.015,
    "FixedFee": 0.00,
    "SourceId": "123",
    "SourceType": "...",
    "SCMSalesFeeRate": 0.05,
    "PaymentServiceProvider": "PaymentMiddleware"
}
```

<div class="pf-table-wrap">
<table>
<thead><tr><th style="width:20%">欄位</th><th>意義</th></tr></thead>
<tbody>
<tr><td><code>PaymentFlow</code></td><td>這筆交易走的是 <code>DirectCharge</code> 還是 <code>DestinationCharge</code></td></tr>
<tr><td><code>AccountType</code></td><td>子帳戶類型：<code>Standard</code> / <code>Custom</code> / <code>Express</code>，決定金流手續費要不要平台自己算</td></tr>
<tr><td><code>SubAccount</code></td><td>實際收款的 Stripe 子帳戶 ID</td></tr>
<tr><td><code>CardBrand</code> / <code>CardIssueCountry</code></td><td>這次刷卡用的卡片品牌與發卡國，也就是查金流手續費率的組合鍵</td></tr>
<tr><td><code>FeeRate</code> / <code>FixedFee</code></td><td><code>PayProfile</code> 查出來的金流手續費率與固定費</td></tr>
<tr><td><code>SourceId</code> / <code>SourceType</code></td><td>這個費率的來源，<code>SourceId</code> 對應 <code>PayProfileProcessingFee</code> 表的主鍵</td></tr>
<tr><td><code>SCMSalesFeeRate</code></td><td><code>SalesOrder</code> 查出來的系統使用費率</td></tr>
<tr><td><code>PaymentServiceProvider</code></td><td>固定為 <code>PaymentMiddleware</code>，標示這筆交易走哪一套金流串接</td></tr>
</tbody>
</table>
</div>

### ERP 轉單時的欄位映射

ERP 轉單 Job 會讀出 <code>TradesOrderThirdPartyPayment.Info</code> 這段 JSON，把它還原成一般會計看得懂的訂單欄位：

<div class="pf-table-wrap">
<table>
<thead><tr><th>Info 欄位</th><th>ERP 目標欄位</th><th>說明</th></tr></thead>
<tbody>
<tr><td><code>FeeRate</code></td><td><code>SalesOrderSlave_SCMCreditCardFeeRate</code> /<span class="pf-cell-sub"><code>SCMCreditCardFeeRate2</code></span></td><td>信用卡手續費率</td></tr>
<tr><td><code>SCMSalesFeeRate</code></td><td><code>SalesOrderSlave_SCMSalesFeeRate</code></td><td>系統使用費率</td></tr>
<tr><td><code>FixedFee</code></td><td><code>SalesOrderGroup_PaymentFixedFee</code></td><td>付款固定費用</td></tr>
<tr><td><code>CardBrand</code> / <code>CardIssueCountry</code></td><td><code>SalesOrder</code> 相關欄位</td><td>卡別與國家資訊</td></tr>
</tbody>
</table>
</div>

<!-- endtab -->

<!-- tab 費用資料流向全圖-->

把前面幾個 tab 拆解的內容串成一條完整的線，從 Pipeline 第 19 道設定卡片資訊開始，一路走到 ERP 生出費用單為止：

<div class="pf-term">
<div class="pf-term-bar"><span class="pf-term-dot r"></span><span class="pf-term-dot y"></span><span class="pf-term-dot g"></span><span class="pf-term-title">費用資料流向全圖</span></div>
<pre class="pf-term-pre"><span class="tstep">[Step #19]</span> ArrangePayTypeExpressInfoProcessor
    設定 context.CreditCardInfo.Brand / IssueCountryCode
         │
         ▼
<span class="tstep">[Step #26]</span> GetOrderProcessingFeeProcessor<span class="tbadge">（HK Only）</span>
    ┌──────────────────────────┐   ┌───────────────────────────────┐
    │ ProcessingFeeService     │   │ ProcessingFeeService           │
    │ .GetSalesFeeInfo()       │   │ .GetCreditCardPayProfileFeeInfo│
    │                          │   │                                │
    │ <span class="tdb">DB: SupplierContract</span>     │   │ <span class="tdb">DB: csp_GetPayProfileProcessing</span>│
    │     <span class="tdb">SalesFeeRate 表</span>      │   │     <span class="tdb">Fee SP（WebStoreDB）</span>       │
    └──────────────────────────┘   └───────────────────────────────┘
         │                                      │
         ↓ context.ProcessingFeeInfo.SalesOrder  ↓ context.ProcessingFeeInfo.PayProfile
         │                                      │
         └─────────────────┬────────────────────┘
                           ▼
<span class="tstep">[Step #30]</span> ThirdPartyPayApiProcessor
    → PaymentMiddlewareService.PaymentRequest()
        → StripePayChannelService.GetPayExtendInfo()
            → GetStripeApplicationFee()
                ① 商品費 = Σ(商品金額 - 購物金) × SalesOrder.Rate
                ② 運費費 = Σ(運費 - 購物金) × SalesOrder.Rate
                ③ Custom Only: + TotalPayment × PayProfile.Rate + PayProfile.FixedFee
                ─────────────────────────────────────────────
                → <span class="tok">ExtendInfo["application_fee_amount"] = ①+②+③</span>
        → HTTP POST PMW /api/v1/Pay/CreditCardOnce_Stripe/{tgCode}
            (帶 application_fee_amount)
                 │
                 ▼
            Stripe API POST /v1/payment_intents
                 <span class="tdim">(application_fee_amount * 100 → 整數，送給 Stripe)</span>
                 │
                 ▼
            <span class="tfinal">Stripe 扣款成功</span>
                 │
                 ▼
<span class="tstep">[付款成功後]</span> StripePayChannelService.GetTradesOrderThirdPartyPaymentInfo()
    → 序列化 FeeRate/SCMSalesFeeRate/FixedFee/CardBrand/CardIssueCountry
    → 存入 TradesOrderThirdPartyPayment.Info（JSON）
                 │
                 ▼
<span class="tstep">[TransferOrderProcessor → AfterOrderProcessor → ERP Job]</span>
    csp_TradesOrderTransToSalesOrderWithFlow_Mall
    → 讀取 ThirdPartyPayment_Info
    → 更新 SalesOrderSlave_SCMCreditCardFeeRate / SCMSalesFeeRate / PaymentFixedFee
    → <span class="tfinal">生成 ExpenseOrder（費用單）</span>
</pre>
</div>

<!-- endtab -->

{% endtabs %}
