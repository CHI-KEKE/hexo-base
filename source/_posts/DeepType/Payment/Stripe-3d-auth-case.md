---
title: Stripe 3D Secure 驗證失敗處理：從 canceled 卡死到根本觸發鏈設計
date: 2026-07-21 10:07:00
categories: Payment
top_img: https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/stripe/3d.png
cover : https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/stripe/3d.png
toc:
toc_number:
comments :

---

{% tabs Stripe-3d-auth-case%}

<!-- tab 案例全文-->

<style>
.s3d{display:block;font-family:'Segoe UI',-apple-system,BlinkMacSystemFont,'PingFang TC','Microsoft JhengHei',sans-serif;background:radial-gradient(1200px 600px at 10% -10%, rgba(108,140,255,.18), transparent 60%),radial-gradient(1000px 500px at 100% 0%, rgba(255,107,107,.12), transparent 55%),#0f1220;color:#e7e9f5!important;line-height:1.7;border-radius:20px;padding:1px 0 28px;margin:0 0 20px;overflow:hidden;}
.s3d *{box-sizing:border-box;}
.s3d a{color:#6c8cff!important;text-decoration:none;}
.s3d a:hover{text-decoration:underline;}
.s3d .s3d-hero{padding:48px 6vw 34px;text-align:center;position:relative;overflow:hidden;}
.s3d .s3d-hero::before{content:"";position:absolute;inset:0;background:linear-gradient(180deg, rgba(255,107,107,.08), transparent 70%);pointer-events:none;}
.s3d .s3d-eyebrow{display:inline-block;font-size:.78rem;letter-spacing:.16em;color:#4fd1c5!important;text-transform:uppercase;font-weight:700;margin-bottom:12px;}
.s3d .s3d-hero h1{font-size:1.7rem;margin:0 0 12px;color:#fff!important;font-weight:800;}
.s3d .s3d-hero p{color:#a6acc9!important;max-width:700px;margin:0 auto;font-size:1rem;}
.s3d .s3d-badges{display:flex;gap:10px;justify-content:center;flex-wrap:wrap;margin-top:20px;}
.s3d .s3d-badge{background:#1b2038;border:1px solid #2a3153;padding:6px 14px;border-radius:999px;font-size:.8rem;color:#a6acc9!important;}
.s3d .s3d-container{padding:0 5vw;}
.s3d nav.s3d-toc{background:#1b2038;border:1px solid #2a3153;border-radius:14px;padding:20px 22px;margin-bottom:36px;}
.s3d nav.s3d-toc h2{margin:0 0 12px;font-size:.9rem;letter-spacing:.08em;text-transform:uppercase;color:#4fd1c5!important;}
.s3d nav.s3d-toc ol{margin:0;padding-left:0;list-style:none;display:grid;grid-template-columns:repeat(2,1fr);gap:6px 18px;}
.s3d nav.s3d-toc li a{display:block;padding:5px 8px;border-radius:8px;color:#e7e9f5!important;font-size:.9rem;}
.s3d nav.s3d-toc li a:hover{background:#20263f;text-decoration:none;color:#6c8cff!important;}
.s3d nav.s3d-toc li a .s3d-num{color:#8f6cff!important;font-weight:700;margin-right:6px;}
.s3d section{margin-bottom:44px;}
.s3d .s3d-section-head{display:flex;align-items:center;gap:12px;margin-bottom:18px;}
.s3d .s3d-section-head .s3d-chip{flex:none;width:38px;height:38px;border-radius:11px;display:flex;align-items:center;justify-content:center;font-weight:800;background:linear-gradient(135deg,#6c8cff,#8f6cff);color:#fff!important;font-size:1rem;}
.s3d .s3d-section-head.danger .s3d-chip{background:linear-gradient(135deg,#ff6b6b,#b23b3b);}
.s3d .s3d-section-head h2{margin:0;font-size:1.3rem;font-weight:800;color:#fff!important;}
.s3d h3{font-size:1.08rem;margin:26px 0 10px;color:#fff!important;border-left:4px solid #6c8cff;padding-left:10px;}
.s3d p{color:#a6acc9!important;}
.s3d .s3d-card{background:#1b2038;border:1px solid #2a3153;border-radius:14px;padding:18px 22px;margin:14px 0;}
.s3d table{width:100%;border-collapse:collapse;background:#1b2038!important;border:1px solid #2a3153;border-radius:14px;overflow:hidden;margin:14px 0 22px;font-size:.88rem;}
.s3d thead th{background:linear-gradient(90deg, rgba(108,140,255,.18), rgba(143,108,255,.1))!important;color:#fff!important;text-align:left;padding:11px 14px;font-weight:700;border-bottom:1px solid #2a3153;}
.s3d tbody td{padding:10px 14px;border-bottom:1px solid #2a3153;color:#a6acc9!important;vertical-align:top;background:#1b2038!important;}
.s3d tbody tr:last-child td{border-bottom:none;}
.s3d tbody td strong,.s3d tbody td code{color:#e7e9f5!important;}
.s3d code{font-family:Consolas,'SFMono-Regular','Liberation Mono',Menlo,monospace;background:rgba(108,140,255,.16)!important;color:#c7d2ff!important;padding:2px 6px;border-radius:5px;font-size:.88em;}
.s3d pre.s3d-pre{background:#0d0f1c!important;border:1px solid #2a3153;border-radius:12px;padding:16px 18px;overflow-x:auto;margin:12px 0 22px;position:relative;color:#dce1f5!important;font-size:.84rem!important;line-height:1.65!important;font-family:Consolas,'SFMono-Regular','Liberation Mono',Menlo,monospace!important;white-space:pre!important;}
.s3d pre.s3d-pre::before{content:attr(data-lang);position:absolute;top:6px;right:12px;font-size:.66rem;letter-spacing:.08em;text-transform:uppercase;color:#a6acc9!important;font-family:'Segoe UI',sans-serif;}
.s3d .kw{color:#c792ea!important;} .s3d .str{color:#c3e88d!important;} .s3d .cm{color:#6b7394!important;font-style:italic;} .s3d .fn{color:#82aaff!important;} .s3d .type{color:#ffcb6b!important;} .s3d .num{color:#f78c6c!important;}
.s3d .s3d-callout{border-radius:12px;padding:14px 18px;margin:14px 0 22px;border:1px solid #2a3153;display:flex;gap:12px;}
.s3d .s3d-callout .s3d-icon{font-size:1.15rem;}
.s3d .s3d-callout.info{background:rgba(108,140,255,.08);border-color:rgba(108,140,255,.35);}
.s3d .s3d-callout.warn{background:rgba(255,180,84,.08);border-color:rgba(255,180,84,.35);}
.s3d .s3d-callout.danger{background:rgba(255,107,107,.08);border-color:rgba(255,107,107,.35);}
.s3d .s3d-callout.ok{background:rgba(79,209,140,.08);border-color:rgba(79,209,140,.35);}
.s3d .s3d-callout p{margin:0;color:#e7e9f5!important;}
.s3d .s3d-callout strong{color:#fff!important;}
.s3d .s3d-file-path{font-family:Consolas,monospace;font-size:.76rem;color:#a6acc9!important;background:#20263f!important;display:inline-block;padding:3px 10px;border-radius:6px;margin:4px 0 12px;}
.s3d .s3d-tag{display:inline-block;font-size:.72rem;font-weight:700;padding:2px 9px;border-radius:999px;letter-spacing:.03em;}
.s3d .s3d-tag.danger{background:rgba(255,107,107,.18)!important;color:#ff6b6b!important;}
.s3d .s3d-tag.ok{background:rgba(79,209,140,.18)!important;color:#4fd18c!important;}
.s3d .s3d-tag.warn{background:rgba(255,180,84,.18)!important;color:#ffb454!important;}
.s3d .s3d-tag.info{background:rgba(108,140,255,.18)!important;color:#6c8cff!important;}
.s3d .s3d-summary-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:14px;margin:16px 0 22px;}
.s3d .s3d-summary-grid .s3d-box{background:#1b2038;border:1px solid #2a3153;border-radius:14px;padding:16px 18px;position:relative;overflow:hidden;}
.s3d .s3d-summary-grid .s3d-box::before{content:"";position:absolute;top:0;left:0;width:4px;height:100%;}
.s3d .s3d-summary-grid .s3d-box.impact::before{background:#ff6b6b;}
.s3d .s3d-summary-grid .s3d-box.cause::before{background:#ffb454;}
.s3d .s3d-summary-grid .s3d-box.fix::before{background:#4fd18c;}
.s3d .s3d-summary-grid .s3d-box h4{margin:0 0 8px;color:#fff!important;font-size:.83rem;text-transform:uppercase;letter-spacing:.06em;}
.s3d .s3d-summary-grid .s3d-box p{margin:0;font-size:.88rem;}
@media (max-width:780px){.s3d .s3d-summary-grid{grid-template-columns:1fr;}}
.s3d .s3d-timeline{position:relative;margin:20px 0;padding-left:26px;border-left:2px solid #2a3153;}
.s3d .s3d-t-item{position:relative;padding-bottom:22px;}
.s3d .s3d-t-item:last-child{padding-bottom:0;}
.s3d .s3d-t-item::before{content:"";position:absolute;left:-32px;top:4px;width:12px;height:12px;border-radius:50%;background:#6c8cff;border:3px solid #0f1220;box-shadow:0 0 0 2px #2a3153;}
.s3d .s3d-t-item.warn::before{background:#ffb454;}
.s3d .s3d-t-item.danger::before{background:#ff6b6b;}
.s3d .s3d-t-item .s3d-t-time{font-family:Consolas,monospace;font-size:.78rem;color:#4fd1c5!important;font-weight:700;}
.s3d .s3d-t-item .s3d-t-title{font-size:.98rem;font-weight:700;color:#fff!important;margin:3px 0 4px;}
.s3d .s3d-t-item .s3d-t-desc{font-size:.86rem;color:#a6acc9!important;margin:0;}
.s3d .s3d-flow{display:flex;align-items:center;gap:8px;flex-wrap:wrap;margin:16px 0 22px;}
.s3d .s3d-flow .s3d-step{background:#1b2038;border:1px solid #2a3153;border-radius:10px;padding:9px 14px;font-size:.84rem;color:#e7e9f5!important;}
.s3d .s3d-flow .s3d-step.bad{border-color:rgba(255,107,107,.5);color:#ff6b6b!important;background:rgba(255,107,107,.08);}
.s3d .s3d-flow .s3d-arrow{color:#8f6cff!important;font-size:1.05rem;}
.s3d .s3d-flow.loop{border:1px dashed rgba(255,107,107,.4);border-radius:16px;padding:12px 10px;background:rgba(255,107,107,.04);}
.s3d .s3d-checklist{list-style:none;padding:0;margin:14px 0;}
.s3d .s3d-checklist li{display:flex;gap:10px;align-items:flex-start;padding:7px 0;color:#a6acc9!important;}
.s3d .s3d-checklist li::before{content:"✓";flex:none;width:18px;height:18px;border-radius:6px;background:rgba(79,209,140,.18);color:#4fd18c!important;font-size:.72rem;display:flex;align-items:center;justify-content:center;margin-top:2px;font-weight:700;}
.s3d .s3d-evidence{border:1px solid rgba(108,140,255,.35);background:linear-gradient(135deg, rgba(108,140,255,.06), rgba(143,108,255,.03));border-radius:14px;padding:18px 20px;margin:16px 0 22px;}
.s3d .s3d-evidence .s3d-evidence-head{display:flex;align-items:center;gap:8px;margin-bottom:10px;}
.s3d .s3d-evidence .s3d-evidence-head .s3d-tag{background:#6c8cff!important;color:#fff!important;}
.s3d footer{text-align:center;padding:30px 5vw 6px;color:#a6acc9!important;font-size:.82rem;border-top:1px solid #2a3153;margin-top:8px;}
@media (max-width:680px){.s3d nav.s3d-toc ol{grid-template-columns:1fr;} .s3d .s3d-summary-grid{grid-template-columns:1fr;}}
</style>

<div class="s3d">
<div class="s3d-hero">
<span class="s3d-eyebrow">Payment · Stripe · 異常處理</span>
<h1>3D Secure 驗證失敗處理</h1>
</div>
<div class="s3d-container">

<section id="s3d-1">
<div class="s3d-section-head danger">
<div class="s3d-chip">01</div>
<h2>概述與問題來源</h2>
</div>

<p>3D Secure 是信用卡的額外身份驗證機制。當持卡人在銀行驗證頁面操作失敗時，Stripe 回傳的其實是 <code>requires_payment_method</code> 狀態並附上 <code>last_payment_error</code>（<code>payment_intent_authentication_failure</code>），PMW 會把它判讀為 <code>Failed</code> 交給 MWeb。

<code>canceled</code> 並不是 3D 驗證失敗當下 Stripe 自然轉入的狀態，而是 MWeb 收到 <code>Failed</code> 後，依規則主動呼叫 Stripe Cancel API 所造成的結果

因此真正的問題出在呼叫 Cancel API 成功、Stripe 端已轉為 <code>canceled</code> 之後，MWeb 用來結案的 <code>CancelTradeOrder</code>（退點數、還購物金、回補庫存、轉單…）中途被中斷，本地訂單狀態沒能同步更新，仍停留在 <code>WaitingToPay</code>。當系統下一次重新查詢時，才會「撞見」Stripe 已是 <code>canceled</code> 的終態

而這個狀態剛好又不在 PMW 查詢邏輯的處理分支中，於是 PMW 與 MWeb 雙邊都無法辨識，訂單就此卡死。</p>

<div class="s3d-callout danger">
<span class="s3d-icon">🚨</span>
<p><strong>問題監控來源：</strong> <a href="https://91app.slack.com/archives/C7T5CTALV/p1738461751955299" target="_blank">Slack 3D 驗證失敗問題追蹤討論串</a>　·　分類：資料庫更新異常 / 狀態處理錯誤</p>
</div>

<div class="s3d-summary-grid">
<div class="s3d-box impact">
<h4>💥 影響</h4>
<p>訂單卡在 WaitingToPay，Redis Cache 過期後 WebAPI 持續拋出 <code>GetPayProcessDataProcessorException</code>，需仰賴 Console 人工介入才能結案。</p>
</div>
<div class="s3d-box cause">
<h4>🔍 根因</h4>
<p>並非「Stripe 直接回傳 canceled 沒人處理」這麼單純，而是「MWeb 呼叫 Cancel API 後、CancelTradeOrder 結案流程中途中斷，造成 Stripe 與 DB 狀態不同步」；下一次重新查詢才撞見 PMW 無分支可判讀的 <code>canceled</code></p>
</div>
<div class="s3d-box fix">
<h4>🛠️ 解法</h4>
<p>短期以 Timeout 機制強制轉單；長期需讓 PMW 把 <code>canceled</code> 對應到 <code>Expired</code>（天生不會重複呼叫 Cancel API），並補強結案流程的防重複執行保護</p>
</div>
</div>
</section>

</div>
</div>

<!-- endtab -->

<!-- tab 事件時間軸與技術分析-->

<div class="s3d">
<div class="s3d-container">

<section id="s3d-2">
<div class="s3d-section-head">
<div class="s3d-chip">02</div>
<h2>事件時間軸</h2>
</div>
<p class="s3d-file-path">實際案例時序（同一筆交易）</p>

<div class="s3d-timeline">
<div class="s3d-t-item">
<div class="s3d-t-time">01:16:55.049</div>
<div class="s3d-t-title">建立付款方式</div>
<p class="s3d-t-desc">建立 <code>payment_method</code> 與 <code>payment_intent</code>，流程正常啟動。</p>
</div>
<div class="s3d-t-item">
<div class="s3d-t-time">01:16:58.780</div>
<div class="s3d-t-title">WaitingToPay</div>
<p class="s3d-t-desc">Stripe 回傳 <code>next_action: redirect_to_url</code>，使用者被導向銀行 3D 驗證頁面。</p>
</div>
<div class="s3d-t-item">
<div class="s3d-t-time">01:21:10.334</div>
<div class="s3d-t-title">requires_action</div>
<p class="s3d-t-desc">仍需完成 3D Secure 驗證，<code>amount_received = 0</code>，系統持續等待用戶操作。</p>
</div>
<div class="s3d-t-item warn">
<div class="s3d-t-time">01:27:37.290</div>
<div class="s3d-t-title">驗證失敗</div>
<p class="s3d-t-desc">Stripe 回報 <code>payment_intent_authentication_failure</code>，PMW 收到失敗事件。</p>
</div>
<div class="s3d-t-item warn">
<div class="s3d-t-time">01:27:37.350</div>
<div class="s3d-t-title">API 取消成功</div>
<p class="s3d-t-desc">MWeb 呼叫 Stripe Cancel API 成功，預期訂單應轉入取消流程。</p>
</div>
<div class="s3d-t-item danger">
<div class="s3d-t-time">01:31:07.879</div>
<div class="s3d-t-title">狀態異常</div>
<p class="s3d-t-desc">後續查詢 Stripe 回傳狀態變為 <code>"canceled"</code>，PMW 無法解析此狀態，系統開始進入無限迴圈。</p>
</div>
</div>

<h3>關鍵狀態轉換</h3>
<div class="s3d-flow">
<div class="s3d-step">建立付款</div><span class="s3d-arrow">→</span>
<div class="s3d-step">等待 3D 驗證</div><span class="s3d-arrow">→</span>
<div class="s3d-step">驗證失敗</div><span class="s3d-arrow">→</span>
<div class="s3d-step">API 取消</div><span class="s3d-arrow">→</span>
<div class="s3d-step bad">canceled 狀態異常</div><span class="s3d-arrow">→</span>
<div class="s3d-step bad">系統循環</div>
</div>
</section>

<section id="s3d-3">
<div class="s3d-section-head">
<div class="s3d-chip">03</div>
<h2>技術問題分析</h2>
</div>

<h3>核心問題</h3>
<p>問題有兩層，缺一不可：① MWeb 的 CancelTradeOrder 結案流程中途中斷，導致 Stripe 已是 canceled 但本地 DB 仍是 WaitingToPay；② PMW 對 canceled 狀態沒有對應的判讀邏輯，讓這個不同步狀態被重新查詢到時無法收斂</p>
<ul class="s3d-checklist">
<li>MWeb 收到 Failed（3D 驗證失敗）後呼叫 Cancel API，Stripe 端成功轉為 canceled，但後續 CancelTradeOrder（結案）中斷，DB 訂單狀態沒能同步為 Fail</li>
<li>下一次重新查詢時，PMW 將 canceled 歸類為「未處理錯誤」，回傳非明確結果給 MWeb</li>
<li>MWeb 收到未預期的回應，進入「由 Console 處理」的分支，資料庫訂單狀態持續停留在 <code>WaitingToPay</code></li>
</ul>

<h3>系統行為異常對照</h3>
<table>
<thead><tr><th>組件</th><th>預期行為</th><th>實際行為</th><th>問題原因</th></tr></thead>
<tbody>
<tr><td><strong>MWeb（結案階段）</strong></td><td>呼叫 Cancel API 成功後，完整執行 CancelTradeOrder 並同步本地訂單狀態</td><td>CancelTradeOrder 中途中斷，DB 狀態未同步為 Fail</td><td>結案流程非原子化，中斷後無補償機制</td></tr>
<tr><td><strong>PMW（重查階段）</strong></td><td>明確處理 <code>"canceled"</code> 狀態並回傳可判讀的 ReturnCode</td><td>落入 else 分支，回傳「未處理錯誤」</td><td>狀態判斷邏輯中缺少 <code>canceled</code> 分支</td></tr>
<tr><td><strong>MWeb（重查後）</strong></td><td>依 ReturnCode 更新訂單狀態為取消</td><td>進入「由 Console 處理」的等待迴圈</td><td>收到 PMW 未分類的錯誤回應，找不到對應處理路徑</td></tr>
<tr><td><strong>資料庫</strong></td><td>狀態更新為 Canceled / Fail</td><td>維持 <code>WaitingToPay</code></td><td>源頭是結案中斷，PMW 無法辨識 canceled 只是讓問題浮現、無法收斂</td></tr>
</tbody>
</table>
</section>

</div>
</div>

<!-- endtab -->

<!-- tab 程式碼實證-->

<div class="s3d">
<div class="s3d-container">

<section id="s3d-4">
<div class="s3d-section-head">
<div class="s3d-chip">04</div>
<h2>程式碼實證</h2>
</div>
<p>對照 <code>nineyi.payment.middleware</code> 的 Stripe Plugin 與 <code>nineyi.webstore.mobilewebmall</code> 的 <code>PayChannelController</code> / <code>TradesOrderPaymentService</code>，可以精確定位問題發生的程式碼位置。</p>

<div class="s3d-evidence">
<div class="s3d-evidence-head"><span class="s3d-tag">PMW</span>StripePlugin.GetThirdPartyQueryPaymentDetail</div>
<p class="s3d-file-path">src/Plugins/NineYi.PaymentMiddleware.Plugins.Stripe/StripePlugin.cs</p>
<p>這個私有方法負責把 Stripe <code>PaymentIntent.status</code> 轉換成 PMW 標準 ReturnCode。可以看到判斷式只涵蓋 <code>succeeded</code>、<code>requires_payment_method</code>（含 last_payment_error）、<code>requires_action</code> / <code>requires_confirmation</code> 三類，<code>canceled</code> 完全沒有出現在任何條件分支中，只能落入最後的 <code>else</code>。</p>
<pre class="s3d-pre" data-lang="C#"><span class="kw">private</span> (<span class="type">string</span> returnCode, <span class="type">string</span> returnMessage, <span class="type">IDictionary</span>&lt;<span class="type">string</span>, <span class="type">object</span>&gt; extendInfo) GetThirdPartyQueryPaymentDetail(...)
{
    <span class="kw">var</span> status = paymentIntentResponseEntity.status;
&nbsp;
    <span class="kw">if</span> (status == <span class="str">"succeeded"</span>) { <span class="cm">/* ReturnCodes.Success */</span> }
    <span class="kw">else if</span> (status == <span class="str">"requires_payment_method"</span> &amp;&amp; last_payment_error != <span class="kw">null</span>)
    {
        <span class="kw">return</span> (ReturnCodes.Failed, ...);
    }
    <span class="kw">else if</span> (status == <span class="str">"requires_action"</span> || status == <span class="str">"requires_payment_method"</span> || status == <span class="str">"requires_confirmation"</span>)
    {
        <span class="kw">return</span> (ReturnCodes.WaitingToPay, status, <span class="kw">null</span>);
    }
    <span class="kw">else</span> <span class="cm">// ⚠️ "canceled" 落到這裡，未被顯式處理</span>
    {
        _logger.LogWarning($<span class="str">"Payment Exception. \nPaymentIntentResponseEntity: {json}"</span>);
        <span class="kw">return</span> (ReturnCodes.UnhandledException, status, <span class="kw">null</span>); <span class="cm">// "9001"</span>
    }
}</pre>
<div class="s3d-callout warn">
<span class="s3d-icon">⚠️</span>
<p><strong>與原始事故紀錄的差異：</strong> 事件描述中提到的是「錯誤碼 9999」，但依目前程式碼，<code>canceled</code> 實際落入的是 <code>ReturnCodes.UnhandledException = "9001"</code>（<code>ReturnCodes.UnknownException = "9999"</code> 是另一個保留碼）。無論是 9001 或 9999，兩者在 MWeb 端都不屬於「可辨識」的 ReturnCode，最終效果相同——訂單都會落入 Console 人工處理分支，事件本質未變。</p>
</div>
</div>

<div class="s3d-evidence">
<div class="s3d-evidence-head"><span class="s3d-tag">MWeb</span>TradesOrderPaymentService.FinishPayment</div>
<p class="s3d-file-path">WebStore/Frontend/BLV2/ThirdPartyPay/TradesOrderPaymentService.cs</p>
<p>MWeb 依 PMW 回傳的 <code>ReturnCode</code> 分派後續動作：<code>Success</code> 轉單、<code>Expired</code> 逾時處理、<code>Failed</code> 取消訂單、Worker 情境下的 <code>WaitingToPay</code> 判斷逾時。除此之外的任何 ReturnCode（包含本案的 <code>9001</code>）都會落入最後的 <code>else</code>，也就是文件中所說的「由 Console 處理」。</p>
<pre class="s3d-pre" data-lang="C#"><span class="kw">else</span>
{
    <span class="cm">//// For this scenario, order will process by the console
    //// ReturnCode from Payment Middleware Response included:
    ////  - 2003 : WaitingToPay 待付款
    ////  - 9000 : PayChannelError
    ////  - 9999 : Payment Middleware Exception</span>
    _logger.Info($<span class="str">"Payment Middleware 回傳資訊 - ReturnCode: {resultEntity.ReturnCode}, ReturnMessage: ..."</span>);
    _logger.Info(<span class="str">"訂單將由 Console 進行處理"</span>);
&nbsp;
    resultEntity.Message = Translation...OrderInProcessing;
}</pre>
<p>這段程式碼本身沒有拋出例外，只是把訂單「晾在原地」等待人工處理——這解釋了為何前台會持續顯示處理中，而不是直接報錯。</p>
</div>

<div class="s3d-evidence">
<div class="s3d-evidence-head"><span class="s3d-tag">MWeb</span>PayChannelController.InternalFinishPayment</div>
<p class="s3d-file-path">WebStore/WebAPI/Controllers/PayChannelController.cs</p>
<p>此 API 註解明確標示「提供內部 Console 使用」，僅允許來自公司內網（<code>IsFromCompany()</code>）的請求呼叫，用途正是本文「解決方案」章節提到的「強制轉單」動作——由客服 / RD 在 Console 手動觸發 <code>FinishPayment(isFromWorker: true)</code> 重新跑一次結果判斷流程。</p>
<pre class="s3d-pre" data-lang="C#">[HttpPost]
[Route(<span class="str">"PayChannel/InternalFinishPayment"</span>)]
<span class="kw">public</span> JsonResult InternalFinishPayment(InternalFinishPaymentRequestEntity request)
{
    <span class="kw">if</span> (<span class="kw">this</span>.IsFromCompany() == <span class="kw">false</span>)
    {
        <span class="kw">return</span> <span class="kw">this</span>.Json(ApiResultEntity&lt;<span class="type">object</span>&gt;.Success(<span class="kw">new</span> { }));
    }
&nbsp;
    <span class="kw">var</span> payType = PayChannelHelper.SplitPayProfileType(request.PayProfileType);
&nbsp;
    <span class="kw">var</span> result = _tradesOrderPaymentService.FinishPayment(
        request.ShopId, request.MemberId,
        payType.PayChannel, payType.PayMethod,
        request.TradesOrderGroupCode, request.UniqueKey,
        <span class="type">string</span>.Empty, <span class="kw">isFromWorker: true</span>
    );
&nbsp;
    <span class="kw">return</span> <span class="kw">this</span>.Json(ApiResultEntity&lt;<span class="type">object</span>&gt;.Success(<span class="kw">new</span> { result.ReturnCode, result.ReturnMessage, result.Message }));
}</pre>
<div class="s3d-callout info">
<span class="s3d-icon">💡</span>
<p><strong>關鍵：</strong> 由於呼叫時固定帶入 <code>isFromWorker: true</code>，即使查詢結果仍是 <code>WaitingToPay</code>，也會進入 <code>FinishPayment</code> 內「Worker 逾時判斷」分支，只要超過 Timeout 秒數就會強制取消付款請求並結束訂單，這正是「Timeout 機制進行資料庫狀態更新」的實際程式路徑。</p>
</div>
</div>

<div class="s3d-evidence">
<div class="s3d-evidence-head"><span class="s3d-tag">MWeb</span>StripePayChannelService.IsRequiredCancelPaymentRequestOnReturnFailed</div>
<p class="s3d-file-path">WebStore/Frontend/BLV2/PayChannel/StripePayChannelService.cs</p>
<p>這是本案「第一次」進入 Failed 分支時，真正觸發 Cancel API 呼叫的判斷式。只有當 Stripe 狀態為 <code>requires_payment_method</code> 且錯誤碼為 <code>payment_intent_authentication_failure</code>（也就是 3D 驗證失敗）時才會回傳 <code>true</code>：</p>
<pre class="s3d-pre" data-lang="C#"><span class="kw">public</span> <span class="type">bool</span> IsRequiredCancelPaymentRequestOnReturnFailed(PayProcessContextEntity context, QueryPaymentResultEntity queryResult)
{
    <span class="type">string</span> stripeStatus = queryResult.ExtendInfo.GetValueOrDefault(<span class="str">"status"</span>)?.ToString() ?? <span class="str">""</span>;
    <span class="type">string</span> stripeErrorCode = queryResult.ExtendInfo.GetValueOrDefault(<span class="str">"last_payment_error_code"</span>)?.ToString() ?? <span class="str">""</span>;
&nbsp;
    <span class="kw">return</span> stripeStatus == <span class="str">"requires_payment_method"</span> &amp;&amp; stripeErrorCode == <span class="str">"payment_intent_authentication_failure"</span>;
}</pre>
<p>命中後 <code>FinishPayment</code> 會呼叫 <code>CancelPayment(context)</code> → PMW <code>StripePlugin.Cancel()</code> → 呼叫 Stripe <code>CancelPaymentIntentAsync</code>，成功後 Stripe 端的 PaymentIntent 就會真正轉為 <code>canceled</code>。但緊接著的 <code>CancelTradeOrder(...)</code>（結案：退點數、還券、回補庫存、轉單…）若在這一步中斷（例外、NMQ 逾時、Retry 失敗等），本地訂單狀態就會停留在 <code>WaitingToPay</code>，而 Stripe 端已經是 <code>canceled</code>。</p>
<div class="s3d-callout warn">
<span class="s3d-icon">🔗</span>
<p><strong>完整觸發鏈：</strong> ① 第一次查詢 → <code>requires_payment_method + payment_intent_authentication_failure</code> → PMW 回 <code>Failed(3000)</code> → ② MWeb 命中本判斷式 → 呼叫 Cancel API → Stripe 端成功轉為 <code>canceled</code> → ③ <code>CancelTradeOrder</code> 執行中斷，DB 訂單狀態未落地 → ④ 下一次重試查詢 → Stripe 這次直接回 <code>canceled</code>（不再是 requires_payment_method）→ PMW 因無對應分支回傳 <code>UnhandledException(9001)</code> → MWeb 也無法識別 → 卡入「由 Console 處理」。</p>
</div>
</div>

<h3>ReturnCode 對照（本案相關）</h3>
<table>
<thead><tr><th>ReturnCode</th><th>常數名稱</th><th>意義</th><th>本案是否命中</th></tr></thead>
<tbody>
<tr><td><code>0000</code></td><td>Success</td><td>付款成功</td><td>—</td></tr>
<tr><td><code>2001</code></td><td>Expired</td><td>付款逾期／已取消（建議：canceled 應對應此碼，MWeb 該分支不會呼叫 Cancel API）</td><td>目前未命中，第 08 節建議改為命中</td></tr>
<tr><td><code>2003</code></td><td>WaitingToPay</td><td>待付款 / requires_action 等中間狀態</td><td>驗證中階段命中</td></tr>
<tr><td><code>3000</code></td><td>Failed</td><td>requires_payment_method + 有 last_payment_error</td><td>3D 驗證失敗當下命中（會觸發 MWeb 呼叫 Cancel API）</td></tr>
<tr><td><code>9001</code></td><td><strong>UnhandledException</strong></td><td>狀態不在已知分支中（目前 <code>canceled</code> 命中此碼）</td><td>✅ 命中，導致落入 Console 分支</td></tr>
<tr><td><code>9999</code></td><td>UnknownException</td><td>保留碼，事件敘述中提及但目前程式碼未直接產生此碼</td><td>否（程式碼實際回傳 9001）</td></tr>
</tbody>
</table>
</section>

</div>
</div>

<!-- endtab -->

<!-- tab 異常循環機制-->

<div class="s3d">
<div class="s3d-container">

<section id="s3d-5">
<div class="s3d-section-head danger">
<div class="s3d-chip">05</div>
<h2>異常循環機制</h2>
</div>
<div class="s3d-flow loop">
<div class="s3d-step">Query 狀態</div><span class="s3d-arrow">→</span>
<div class="s3d-step bad">收到 "canceled"</div><span class="s3d-arrow">→</span>
<div class="s3d-step bad">PMW 回傳 9001（未處理錯誤）</div><span class="s3d-arrow">→</span>
<div class="s3d-step">MWeb 落入 Console 分支</div><span class="s3d-arrow">→</span>
<div class="s3d-step bad">前台 / Worker 重試</div><span class="s3d-arrow">→</span>
<div class="s3d-step bad">無限循環</div>
</div>

<div class="s3d-callout danger">
<span class="s3d-icon">🔁</span>
<p><strong>最終結果：</strong> Redis Cache 過期、WebAPI 持續拋出 <code>GetPayProcessDataProcessorException</code>（因 <code>PayProcessContextEntity</code> 快取遺失），訂單處理卡死，只能由 Console 人工介入。</p>
</div>
</section>

</div>
</div>

<!-- endtab -->

<!-- tab 解決方案與預防措施-->

<div class="s3d">
<div class="s3d-container">

<section id="s3d-6">
<div class="s3d-section-head">
<div class="s3d-chip">06</div>
<h2>解決方案</h2>
</div>

<h3>立即處理方案</h3>
<p>使用 Timeout 機制進行資料庫狀態更新（實務上即透過 <code>PayChannel/InternalFinishPayment</code> 由 Console 觸發 <code>isFromWorker: true</code> 的 <code>FinishPayment</code>）：</p>
<ul class="s3d-checklist">
<li>狀態重置：將訂單狀態設為 Timeout</li>
<li>資料壓制：更新資料庫中的訂單狀態</li>
<li>強制轉單：執行轉單流程，完成訂單結案</li>
</ul>

<h3>長期修復方案</h3>
<table>
<thead><tr><th>修復項目</th><th>修復內容</th><th>優先級</th></tr></thead>
<tbody>
<tr><td><strong>PMW 狀態處理</strong></td><td>在 <code>GetThirdPartyQueryPaymentDetail</code> 加入 <code>"canceled"</code> 狀態的顯式判斷，對應 <code>Expired</code>（而非 Failed，避免 MWeb 重複呼叫 Cancel API，詳見第 08 節設計）</td><td><span class="s3d-tag danger">高</span></td></tr>
<tr><td><strong>取消申請單防重複</strong></td><td><code>DoProcessOnReturnForTimeoutTradesOrder</code> 呼叫的 <code>CreateCancelRequest</code> 目前無條件新增，需先檢查是否已有未關閉的取消申請單（詳見第 08 節設計）</td><td><span class="s3d-tag danger">高</span></td></tr>
<tr><td><strong>子步驟可安全重跑</strong></td><td>依第 08 節方案 A/B，讓 CancelTradeOrder（Failed 分支）具備進度追蹤或各步驟自行冪等，避免自動補跑造成重複退款 / 重複還購物金</td><td><span class="s3d-tag danger">高</span></td></tr>
<tr><td><strong>錯誤處理機制</strong></td><td>改善 MWeb 對未分類 ReturnCode 的自動回復機制，避免單純停在「由 Console 處理」</td><td><span class="s3d-tag warn">中</span></td></tr>
<tr><td><strong>監控告警</strong></td><td>加強 3D 驗證失敗率與 <code>canceled</code> 狀態出現次數的監控</td><td><span class="s3d-tag warn">中</span></td></tr>
<tr><td><strong>文件更新</strong></td><td>更新異常處理操作手冊，同步修正 ReturnCode 描述（9001 vs 9999）</td><td><span class="s3d-tag info">低</span></td></tr>
</tbody>
</table>
</section>

</div>
</div>

<!-- endtab -->

<!-- tab 根本觸發鏈與自動化根治設計-->

<div class="s3d">
<div class="s3d-container">

<section id="s3d-8">
<div class="s3d-section-head danger">
<div class="s3d-chip">08</div>
<h2>根本觸發鏈與自動化根治設計</h2>
</div>



<h3>建議修改一：PMW 顯式處理 canceled 狀態，對應 Expired 而非 Failed</h3>
<p class="s3d-file-path">src/Plugins/NineYi.PaymentMiddleware.Plugins.Stripe/StripePlugin.cs · GetThirdPartyQueryPaymentDetail</p>
<pre class="s3d-pre" data-lang="C#"><span class="kw">else if</span> (status == StripeStatusConstants.Cancelled) <span class="cm">// "canceled"</span>
{
    <span class="cm">// Stripe 端已是終態，且此時通常已無 last_payment_error 可帶（因為是先前流程呼叫 Cancel 造成的）。
    // 對應 Expired 而非 Failed，讓 MWeb 走「逾時／已取消」處理路徑——
    // 這條路徑天生不會再呼叫 Cancel API，避免對已終態的 PaymentIntent 重複取消。</span>
    <span class="kw">var</span> extendInfo = <span class="kw">new</span> Dictionary&lt;<span class="type">string</span>, <span class="type">object</span>&gt;();
    extendInfo.Add(<span class="str">"status"</span>, status);
&nbsp;
    <span class="kw">return</span> (ReturnCodes.Expired, status, extendInfo);
}
<span class="kw">else</span>
{
    <span class="cm">// 其餘真正未知的狀態才落入 UnhandledException</span>
    ...
}</pre>

<h3>建議修改二：確認 MWeb Expired 分支的既有行為（不需額外改動即可避免重複 Cancel）</h3>
<p class="s3d-file-path">WebStore/Frontend/BLV2/ThirdPartyPay/TradesOrderPaymentService.cs · FinishPayment / DoProcessOnReturnForTimeoutTradesOrder</p>
<pre class="s3d-pre" data-lang="C#"><span class="kw">else if</span> (resultEntity.ReturnCode == PaymentMiddlewareReturnCodeConstants.Expired)
{
    <span class="cm">//// 付款逾時，不關閉第三方訂單</span>
    thirdPartyPaymentEntity.IsClosed = <span class="kw">false</span>;
    <span class="kw">this</span>.EnsureRequestNotDuplicated(memberId, k);
    <span class="kw">this</span>.DoProcessOnReturnForTimeoutTradesOrder(context, thirdPartyPaymentEntity, payChannelService);
    <span class="cm">// ⚠️ 注意：這條路徑完全沒有呼叫 CancelPayment / Cancel API，
    // 所以把 canceled 對應到 Expired，天生規避了「重複打 Cancel API」的風險，
    // 不需要像 Failed 分支那樣額外去改 IsRequiredCancelPaymentRequestOnReturnFailed。</span>
}</pre>
<p><code>DoProcessOnReturnForTimeoutTradesOrder</code> 內部依序執行：更新 <code>TradesOrderThirdPartyPayment_StatusDef = Timeout</code> → <code>CancelExternalStoreCreditDeduction</code> → <code>_cancelRequestService.CreateCancelRequest(...)</code>（建立取消申請單）→ <code>UpdateOrderSlaveFlow</code>。註解也明確寫著「補庫存、退點、退券於 CancelOrderProcess 進行」——也就是說退點 / 退券 / 補庫存被拆到後續<strong>非同步的取消單處理流程</strong>執行，而不是像 <code>CancelTradeOrder</code> 一樣同步全部做完。</p>

<div class="s3d-callout ok">
<span class="s3d-icon">✅</span>
<p><strong>效果：</strong> 只要修改 PMW 這一處，MWeb 完全不需要改動任何 Cancel 相關判斷式，就能保證「查到 canceled」這件事永遠不會再多打一次 Stripe Cancel API。這比原本「對應 Failed + 額外修改 IsRequiredCancelPaymentRequestOnReturnFailed」更安全，因為它不依賴兩處程式碼同時改對，而是天生走一條不含 Cancel 呼叫的路徑。</p>
</div>

<div class="s3d-callout warn">
<span class="s3d-icon">⚠️</span>
<p><strong>仍要注意的殘留風險：</strong> <code>CancelExternalStoreCreditDeduction</code>（同前面分析，靠靜態旗標判斷，非真正冪等）與 <code>_cancelRequestService.CreateCancelRequest(...)</code>（無條件新增一筆取消申請單，未檢查是否已存在同一 <code>TradesOrderGroupId</code> 的待處理取消單）在這條路徑上一樣<strong>不是天生防重複的</strong>。改用 Expired 只解決了「不會重複呼叫第三方 Cancel API」這一項，若 <code>DoProcessOnReturnForTimeoutTradesOrder</code> 本身也被重跑，仍可能建立重複的取消申請單。建議在 <code>CreateCancelRequest</code> 前加一個查詢：若該 <code>TradesOrderGroupId</code> 已存在未關閉（<code>CancelRequest_IsClosed = false</code>）的取消申請單，就不要再建立新的一筆。</p>
</div>

<h3>建議修改三：CancelTradeOrder（Failed 分支專用）必須是可安全重跑的（而非「猜中斷點在哪裡」）</h3>

<div class="s3d-callout danger">
<span class="s3d-icon">🚨</span>
<p><strong>重新檢視後發現：整個 CancelTradeOrder 直接重跑「目前恰好安全、但不是被設計保證的」。</strong> <code>CancelTradeOrder</code> 的第一行就是把 <code>TradesOrderThirdPartyPayment_StatusDef</code> 寫成 <code>Fail</code>（<code>UpdateThirdPartyPaymentWithRetry</code>），而 <code>FinishPayment</code> 開頭又有守門：狀態非 <code>WaitingToPay</code> 就直接丟例外、不會再查詢 Stripe。這代表：本文情境（重試查詢仍能拿到 Stripe <code>canceled</code>）只可能發生在中斷點落在 <code>UpdateThirdPartyPaymentWithRetry</code> 提交<strong>之前</strong>——此時後面的 <code>CancelExternalStoreCreditDeduction</code>、<code>RefundPoint</code>、<code>RevertPromoCodePoolQuota</code> 都還沒執行過，重跑不會重複。但如果中斷點落在<strong>之後</strong>（例如卡在 <code>RefundPoint</code>），下一次重試會被 <code>FinishPayment</code> 的守門擋下，丟出 <code>TradesOrderThirdPartyPaymentException(NotWaitingToPay)</code>，症狀會變成另一種卡單，而不是本文描述的 canceled 迴圈——但只要日後步驟順序調整、流程非同步化、或多個 worker 併發處理同一筆訂單，這個「恰好安全」的隱性假設就會失效。</p>
</div>

<table>
<thead><tr><th>方案</th><th>做法</th><th>優缺點</th></tr></thead>
<tbody>
<tr>
<td><strong>A. 進度追蹤（Checkpoint / Saga）</strong></td>
<td>在 <code>TradesOrderThirdPartyPayment</code> 新增每一子步驟的完成標記（或獨立的 CancelStep 記錄表），每個子步驟成功後立即落地一筆，而不是只在最後寫一次總狀態。重跑時依標記跳過已完成步驟，只執行剩餘的。</td>
<td>改動集中在 <code>TradesOrderPaymentService</code>，不需動到下游各服務；但需新增資料結構、每步都要多一次 DB write。</td>
</tr>
<tr>
<td><strong>B. 各子步驟自行冪等（Check-before-Act）</strong></td>
<td>把 <code>CancelExternalStoreCreditDeduction</code>、<code>RefundPoint</code>、<code>RevertPromoCodePoolQuota</code>、<code>CreateCancelRequest</code> 等改為執行前先查「權威狀態」（外部購物金流水、點數退款紀錄、Promo Pool 配額紀錄、是否已有未關閉的取消申請單）確認尚未處理過才動作。</td>
<td>更貼近各服務職責、長期更穩健，且同時涵蓋 Failed 與 Expired 兩條路徑；但要動到多個下游服務（外部購物金 / 點數 / PromoCode Pool / 券中心 / 取消申請單），改動範圍較大。</td>
</tr>
</tbody>
</table>

<div class="s3d-callout ok">
<span class="s3d-icon">✅</span>
<p><strong>建議順序：</strong> 先做 PMW／MWeb 的 <code>Expired</code> 對應（本節開頭建議修改一、二），從根本避免「canceled 重試」誤觸 Cancel API；接着針對 Failed 與 Expired 兩條路徑，短期用方案 A（進度追蹤）讓各自的結案流程可安全重跑；中長期再逐步把方案 B 的 Check-before-Act 補進各下游服務，讓整個取消流程無論從哪個步驟中斷、被重跑幾次都收斂到同一個正確結果。</p>
</div>
</section>

<footer>
Stripe · Paytypes 知識庫　|　來源：<code>10-3D驗證失敗處理.md</code>　+　PaymentMiddleware / MWeb 原始碼實證
</footer>

</div>
</div>

<!-- endtab -->

{% endtabs %}
