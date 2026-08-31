---
title: Stripe-Query
date: 2026-07-26 08:45:00
categories: Payment
top_img: https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/stripe/account.png
cover : https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/stripe/account.png
toc:
toc_number:
comments :
tags:
---

<style>
.sq-note { border-radius: 12px; padding: 16px 20px; margin: 16px 0 24px; border: 1px solid #2a3153; display: flex; gap: 12px; background: #1b2038; }
.sq-note .sq-icon { font-size: 1.2rem; flex: none; }
.sq-note p { margin: 0 0 10px; color: #e7e9f5; line-height: 1.8; }
.sq-note p:last-child { margin-bottom: 0; }
.sq-note code { background: rgba(108,140,255,.18); color: #c7d2ff; }
.sq-note.info { background: #1a2340; border-color: rgba(108,140,255,.45); }
.sq-note.warn { background: #2b2415; border-color: rgba(255,180,84,.45); }
.sq-note.danger { background: #2a191a; border-color: rgba(255,107,107,.45); }
.sq-note.ok { background: #142822; border-color: rgba(79,209,140,.45); }

.sq-tag { display: inline-block; font-size: .74rem; font-weight: 700; padding: 2px 10px; border-radius: 999px; letter-spacing: .03em; font-family: Consolas, monospace; }
.sq-tag.ok { background: rgba(79,209,140,.18); color: #1f9d63; }
.sq-tag.warn { background: rgba(255,180,84,.2); color: #b3720a; }
.sq-tag.danger { background: rgba(255,107,107,.2); color: #d43f3f; }
.sq-tag.info { background: rgba(108,140,255,.2); color: #4a63c9; }

.sq-table { width: 100%; border-collapse: collapse; background: #1b2038; border: 1px solid #2a3153; border-radius: 14px; overflow: hidden; margin: 14px 0 24px; font-size: .92rem; }
.sq-table th { background: linear-gradient(90deg, rgba(108,140,255,.35), rgba(79,209,197,.22)); color: #fff; text-align: left; padding: 11px 16px; font-weight: 700; border-bottom: 1px solid #2a3153; }
.sq-table td { padding: 10px 16px; border-bottom: 1px solid #2a3153; color: #d3d7ec; vertical-align: top; }
.sq-table tr:last-child td { border-bottom: none; }
.sq-table td code, .sq-table th code { background: rgba(108,140,255,.18); color: #c7d2ff; }

.sq-summary-grid { display: grid; grid-template-columns: repeat(3,1fr); gap: 14px; margin: 16px 0 24px; }
.sq-summary-grid .sq-box { background: #1b2038; border: 1px solid #2a3153; border-left: 4px solid #6c8cff; border-radius: 14px; padding: 16px 18px; color: #d3d7ec; }
.sq-summary-grid .sq-box.b { border-left-color: #4fd1c5; }
.sq-summary-grid .sq-box.c { border-left-color: #ffb454; }
.sq-summary-grid .sq-box h4 { margin: 0 0 8px; color: #fff; font-size: .82rem; text-transform: uppercase; letter-spacing: .05em; }
.sq-summary-grid .sq-box p { margin: 0; font-size: .9rem; color: #b7bcda; }
.sq-summary-grid .sq-box code { background: rgba(108,140,255,.18); color: #c7d2ff; }

.sq-compare-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; margin: 16px 0 24px; }
.sq-compare-grid .sq-cbox { background: #1b2038; border: 1px solid #2a3153; border-radius: 14px; padding: 18px 20px; }
.sq-compare-grid .sq-cbox.pi { border-top: 3px solid #6c8cff; }
.sq-compare-grid .sq-cbox.ch { border-top: 3px solid #4fd1c5; }
.sq-compare-grid .sq-chead { display: flex; align-items: center; gap: 8px; margin-bottom: 10px; }
.sq-compare-grid .sq-chead .sq-cicon { font-size: 1.1rem; }
.sq-compare-grid .sq-chead h4 { margin: 0; color: #fff; font-size: 1rem; }
.sq-compare-grid p { margin: 0; color: #c3c9e6; font-size: .9rem; line-height: 1.8; }
.sq-compare-grid code { background: rgba(108,140,255,.18); color: #c7d2ff; }
@media (max-width: 720px) { .sq-compare-grid { grid-template-columns: 1fr; } }

.sq-status-block { background: #1b2038; border: 1px solid #2a3153; border-left: 4px solid #6c8cff; border-radius: 14px; padding: 18px 22px; margin: 22px 0 28px; }
.sq-status-block.ok { border-left-color: #4fd18c; }
.sq-status-block.danger { border-left-color: #ff6b6b; }
.sq-status-block.warn { border-left-color: #ffb454; }
.sq-status-block.info { border-left-color: #4fd1c5; }
.sq-status-block h3 { margin: 0 0 14px !important; padding: 0 0 12px !important; border-bottom: 1px dashed #2a3153 !important; font-size: 1.02rem !important; color: #fff !important; }
.sq-status-block p { color: #c3c9e6; margin: 0 0 10px; }
.sq-status-block p:last-child { margin-bottom: 0; }
.sq-status-block table.sq-table { margin: 10px 0; }

.sq-diagram { background: #0d0f1c; border: 1px solid #2a3153; border-radius: 12px; padding: 16px 20px; margin: 14px 0 24px; overflow-x: auto; }
.sq-diagram-pre { margin: 0 !important; background: transparent !important; padding: 0 !important; border: none !important; box-shadow: none !important; font-family: Consolas, monospace !important; white-space: pre !important; color: #c3c9e6 !important; font-size: .84rem !important; line-height: 1.7 !important; }

.sq-banner { display: flex; align-items: center; gap: 12px; flex-wrap: wrap; border-radius: 12px; padding: 12px 18px; margin: 26px 0 4px; border: 1px solid #2a3153; border-left: 4px solid #6c8cff; background: #1b2038; }
.sq-banner.ok { border-left-color: #4fd18c; }
.sq-banner.warn { border-left-color: #ffb454; }
.sq-banner.danger { border-left-color: #ff6b6b; }
.sq-banner.info { border-left-color: #4fd1c5; }
.sq-banner-num { flex: none; width: 30px; height: 30px; border-radius: 9px; background: #20263f; border: 1px solid #2a3153; display: flex; align-items: center; justify-content: center; font-weight: 800; color: #4fd1c5; font-size: .88rem; }
.sq-banner-title { font-size: 1rem; font-weight: 700; color: #fff; flex: 1; min-width: 180px; }

.sq-scenario { background: #1b2038; border: 1px solid #2a3153; border-left: 4px solid #6c8cff; border-radius: 14px; padding: 0 22px 22px; margin: 26px 0 30px; }
.sq-scenario.ok { border-left-color: #4fd18c; }
.sq-scenario.warn { border-left-color: #ffb454; }
.sq-scenario.danger { border-left-color: #ff6b6b; }
.sq-scenario.info { border-left-color: #4fd1c5; }
.sq-scenario .sq-banner { margin: 0 -22px 18px; border-radius: 12px 12px 0 0; border: none; border-bottom: 1px solid #2a3153; }
.sq-scenario p, .sq-scenario li { color: #c3c9e6; }
.sq-scenario strong { color: #fff; }
.sq-scenario > p:first-of-type { margin-top: 0; }

.sq-chain { display: inline-block; font-family: Consolas, monospace; font-size: .84rem; background: #0d0f1c; border: 1px solid #2a3153; border-radius: 8px; padding: 8px 14px; color: #c3c9e6; margin: 10px 0; }

.sq-checklist { list-style: none; padding: 0; margin: 16px 0; }
.sq-checklist li { padding: 6px 0 6px 28px; color: #555; position: relative; }
.sq-checklist li::before { content: "✓"; position: absolute; left: 0; top: 6px; width: 18px; height: 18px; border-radius: 5px; background: rgba(79,209,140,.18); color: #1f9d63; font-size: .72rem; font-weight: 700; display: flex; align-items: center; justify-content: center; }
</style>

{% tabs Stripe-Query %}

<!-- tab 概述與用途-->

Stripe 信用卡付款有時候不是「打完 API 馬上知道結果」， 3D 驗證、系統可能需要事後再確認一次「這筆錢到底有沒有真的收到」。`QueryPayment` 就是拿著 Pay 時取得的交易編號，回頭去問 Stripe「這筆付款現在狀態是什麼」，再把 Stripe 的答案翻譯成 PaymentMiddleware 自己的標準回應碼，讓上游系統不用去研究 Stripe 原始的狀態機

三種時機：

<table class="sq-table">
<tr><th>使用時機</th><th>說明</th></tr>
<tr><td><strong>3D 驗證完成後</strong></td><td>Pay 回傳 <code>2003</code>，用戶完成 3D 驗證後，需呼叫此 API 確認最終結果</td></tr>
<tr><td><strong>非同步補查</strong></td><td>付款後主動確認 <code>PaymentIntent</code> 是否真正成功，確保資料一致性</td></tr>
<tr><td><strong>輪詢查詢</strong></td><td>持續確認等待中的付款狀態，直到收斂為明確結果</td></tr>
</table>

<div class="sq-summary-grid">
<div class="sq-box a">
<h4>🎯 核心目的</h4>
<p>把 Stripe <code>PaymentIntent.status</code> 轉換成 PaymentMiddleware 標準 <code>ReturnCode</code>，讓上游系統不需要理解 Stripe 原始狀態機。</p>
</div>
<div class="sq-box b">
<h4>🔑 查詢主鍵</h4>
<p><code>transaction_id</code> 必須是 <strong>PaymentIntent ID</strong>（<code>pi_xxx</code> 格式），不能是 Charge ID（<code>ch_xxx</code>）。</p>
</div>
<div class="sq-box c">
<h4>⚠️ 設計重點</h4>
<p>「查詢失敗」與「付款失敗」被刻意區分：Stripe API 呼叫失敗回傳 <code>2003</code>，只有明確的付款拒絕才回傳 <code>3000</code>。</p>
</div>
</div>

<!-- endtab -->

<!-- tab 入口 API 與資料結構-->

### 入口 API

```
POST /api/v1/QueryPayment/{payMethod}_{payChannel}

// Stripe 信用卡範例：
POST /api/v1/QueryPayment/CreditCardOnce_Stripe
```

### Request 結構

HTTP Body 對應 `QueryPaymentRequestEntity`：

<table class="sq-table">
<tr><th>欄位</th><th>型別</th><th>必填</th><th>說明</th></tr>
<tr><td><code>request_id</code></td><td>string</td><td>✅</td><td>本次查詢的唯一識別碼</td></tr>
<tr><td><code>transaction_id</code></td><td>string</td><td>✅</td><td><strong>Pay 時回傳的 PaymentIntent ID</strong>（<code>pi_xxx</code> 格式），查詢的主鍵</td></tr>
<tr><td><code>country</code></td><td>string</td><td>—</td><td>國家代碼</td></tr>
<tr><td><code>extend_info.payment_flow</code></td><td>string</td><td>✅</td><td><code>DirectCharge</code> 或 <code>DestinationCharge</code>，影響 Header 帶法</td></tr>
<tr><td><code>extend_info.stripe_account</code></td><td>string</td><td>✅</td><td>Stripe 子帳號 ID（DirectCharge 查詢時帶入 Header）</td></tr>
<tr><td><code>extend_info.query_string</code></td><td>string</td><td>❌</td><td>⚠️ <strong>目前程式碼未使用</strong>，傳入無任何效果（dead field）</td></tr>
</table>

```json
{
  "request_id": "唯一請求識別碼",
  "transaction_id": "pi_xxxxxxxxxxxxxxxxxxxxxxxx",
  "extend_info": {
    "payment_flow": "DirectCharge | DestinationCharge",
    "stripe_account": "acct_xxxxxxxxxx"
  }
}
```

### Response 結構

```json
{
  "request_id": "唯一請求識別碼",
  "return_code": "0000",
  "return_message": "succeeded",
  "transaction_id": "pi_xxxxxxxxxxxxxxxxxxxxxxxx",
  "extend_info": {
    "payment_intent_id": "pi_xxxxxxxxxxxxxxxxxxxxxxxx",
    "charge_id": "ch_xxxxxxxxxxxxxxxxxxxxxxxx"
  }
}
```

<!-- endtab -->

<!-- tab 內部執行流程-->

`StripePlugin.QueryPayment()` 收到請求後，依 `payment_flow` 決定是否帶入子帳號 Header，再呼叫 Stripe API 查詢，最後把結果交給 `GetThirdPartyQueryPaymentDetail()` 判斷狀態：

<div class="sq-diagram">
<pre class="sq-diagram-pre">QueryPayment(request)
        │
        ├─ payment_flow == DirectCharge？
        │       ├─ YES → subAcct = ExtendInfo.SubAccount
        │       └─ NO  → subAcct = null
        │
        ▼
[Stripe API] GET /v1/payment_intents/{transaction_id}
   DirectCharge      → Header: Stripe-Account: {sub_account}
   DestinationCharge → Header: 無（使用 Platform 主帳號查詢）
        │
        ├─ 成功 → GetThirdPartyQueryPaymentDetail(response)
        │              依 status 判斷，回傳 ReturnCode + ExtendInfo
        │
        ├─ ApiException（Stripe 回 4xx/5xx）
        │              ReturnCode = 2003（WaitingToPay）
        │
        └─ Exception（其他未知錯誤）
                       logger.LogError → throw → HTTP 500</pre>
</div>

### Stripe API 呼叫：`GET /v1/payment_intents/{id}`

<table class="sq-table">
<tr><th>項目</th><th>DirectCharge</th><th>DestinationCharge</th></tr>
<tr><td>Header</td><td><code>Stripe-Account: {sub_account}</code></td><td>無</td></tr>
<tr><td>查詢範圍</td><td>子帳號下的 PaymentIntent</td><td>主帳號下的 PaymentIntent</td></tr>
<tr><td>subAcct 判斷</td><td><code>request.ExtendInfo.SubAccount</code></td><td><code>null</code></td></tr>
</table>

<!-- endtab -->

<!-- tab 回應狀態判斷邏輯-->

`GetThirdPartyQueryPaymentDetail()` 是整支 API 的核心，依 `PaymentIntent.status` 與是否帶有錯誤資訊，決定回傳的 `ReturnCode`：

<div class="sq-status-block ok">

### ✅ status = `succeeded`（付款成功）— ReturnCode `0000`

<p>信用卡情境下，ExtendInfo 只帶回：</p>

<table class="sq-table">
<tr><th>欄位</th><th>來源</th></tr>
<tr><td><code>payment_intent_id</code></td><td><code>PaymentIntentResponseEntity.id</code></td></tr>
<tr><td><code>charge_id</code></td><td><code>charges.data[0].id</code></td></tr>
</table>

<p>若付款方式是 <strong>Mobile Wallet（GooglePay / ApplePay）</strong>，ExtendInfo 會額外多帶回卡片資訊：</p>

<table class="sq-table">
<tr><th>欄位</th><th>來源</th></tr>
<tr><td><code>card_brand</code></td><td><code>charges.data[0].payment_method_details.card.brand</code></td></tr>
<tr><td><code>card_country</code></td><td><code>charges.data[0].payment_method_details.card.country</code></td></tr>
<tr><td><code>card_exp_month</code></td><td><code>charges.data[0].payment_method_details.card.exp_month</code></td></tr>
<tr><td><code>card_exp_year</code></td><td><code>charges.data[0].payment_method_details.card.exp_year</code></td></tr>
<tr><td><code>card_last4</code></td><td><code>charges.data[0].payment_method_details.card.last4</code></td></tr>
</table>

</div>

<div class="sq-status-block danger">

### ❌ status = `requires_payment_method` 且有 `last_payment_error`（付款被拒）— ReturnCode `3000`

<table class="sq-table">
<tr><th>欄位</th><th>來源</th></tr>
<tr><td><code>status</code></td><td><code>requires_payment_method</code></td></tr>
<tr><td><code>last_payment_error_code</code></td><td><code>last_payment_error.code</code></td></tr>
<tr><td><code>last_payment_error_decline_code</code></td><td><code>last_payment_error.decline_code</code></td></tr>
<tr><td><code>last_payment_error_message</code></td><td><code>last_payment_error.message</code></td></tr>
<tr><td><code>last_payment_error_type</code></td><td><code>last_payment_error.type</code></td></tr>
</table>

<p><code>ReturnMessage</code> 直接使用 <code>last_payment_error.message</code>（Stripe 原始錯誤文字）。</p>

</div>

<div class="sq-status-block warn">

### ⏳ status = `requires_action` / `requires_payment_method`（無錯誤）/ `requires_confirmation`（等待中）— ReturnCode `2003`

<table class="sq-table">
<tr><th>status</th><th>意義</th></tr>
<tr><td><code>requires_action</code></td><td>仍在等待 3D 驗證</td></tr>
<tr><td><code>requires_payment_method</code>（無錯誤）</td><td>等待用戶提供付款方式</td></tr>
<tr><td><code>requires_confirmation</code></td><td>等待確認</td></tr>
</table>

<p>這三種情況 ExtendInfo 皆為 <code>null</code>，ReturnMessage 直接回傳原始 Stripe status 字串。</p>

</div>

<div class="sq-status-block warn">

### ⚠️ 其他 status（`canceled`、`processing` 等未處理狀態）— ReturnCode `9001`（UnhandledException）

<p>會觸發 <code>logger.LogWarning</code> 記錄完整 <code>PaymentIntentResponseEntity</code>，ReturnMessage 為原始 Stripe status 字串，ExtendInfo 為 <code>null</code>。</p>

</div>

<div class="sq-status-block danger">

### 🔴 Stripe API 呼叫失敗（`ApiException`，4xx/5xx）— ReturnCode `2003`（WaitingToPay）

<p><strong>設計注意：</strong>QueryPayment 遇到 Stripe API 錯誤時，回傳的是 <code>2003</code> 而<strong>不是</strong> <code>3000</code>。設計意圖是「查詢失敗 ≠ 付款失敗」，上游系統應判斷是否需要重試，不可直接認定為付款失敗。</p>
<p>ReturnMessage 格式為 <code>"status code: {http_status}, message: {error.message}"</code>，TransactionId 回傳空字串。</p>

</div>

<div class="sq-status-block danger">

### 🔴 其他未預期例外（`Exception`）— HTTP 500

<p><code>logger.LogError</code> 記錄後直接 <code>throw</code>，回傳 HTTP 500，代表非 Stripe 業務邏輯內的錯誤（例如網路逾時、連線失敗）。</p>

</div>

### 完整狀態對照表

<table class="sq-table">
<tr><th>PaymentIntent status</th><th>附加條件</th><th>ReturnCode</th><th>ReturnMessage</th><th>ExtendInfo</th></tr>
<tr><td><code>succeeded</code></td><td>—</td><td><span class="sq-tag ok">0000</span></td><td><code>"succeeded"</code></td><td><code>payment_intent_id</code> + <code>charge_id</code>（Mobile Wallet 另加卡片資訊）</td></tr>
<tr><td><code>requires_payment_method</code></td><td>有 <code>last_payment_error</code></td><td><span class="sq-tag danger">3000</span></td><td>Stripe 錯誤文字</td><td>錯誤詳情（含 <code>decline_code</code>）</td></tr>
<tr><td><code>requires_action</code></td><td>—</td><td><span class="sq-tag warn">2003</span></td><td><code>"requires_action"</code></td><td><code>null</code></td></tr>
<tr><td><code>requires_payment_method</code></td><td>無 <code>last_payment_error</code></td><td><span class="sq-tag warn">2003</span></td><td><code>"requires_payment_method"</code></td><td><code>null</code></td></tr>
<tr><td><code>requires_confirmation</code></td><td>—</td><td><span class="sq-tag warn">2003</span></td><td><code>"requires_confirmation"</code></td><td><code>null</code></td></tr>
<tr><td><code>canceled</code> / 其他</td><td>—</td><td><span class="sq-tag info">9001</span></td><td>原始 status 字串</td><td><code>null</code>（並觸發 <code>logger.LogWarning</code>）</td></tr>
<tr><td>Stripe 回傳 4xx/5xx</td><td><code>ApiException</code></td><td><span class="sq-tag warn">2003</span></td><td><code>"status code: {n}, message: ..."</code></td><td><code>null</code>，<code>transaction_id</code> 為空字串</td></tr>
<tr><td>未知例外</td><td><code>Exception</code></td><td><span class="sq-tag danger">HTTP 500</span></td><td>—</td><td>—</td></tr>
</table>

<!-- endtab -->

<!-- tab payment_intent_id 與 charge_id 的差異-->

Response 裡同時出現的 `payment_intent_id` 與 `charge_id`，代表的是 Stripe 付款流程中**兩個不同層級的物件**，先分開理解各自的角色，再看兩者的關係。

<div class="sq-compare-grid">
<div class="sq-cbox pi">
<div class="sq-chead"><span class="sq-cicon">🧭</span><h4>PaymentIntent（<code>pi_...</code>）</h4></div>
<p>代表「一次完整付款的意圖／流程」，是 Stripe 付款的最上層物件，貫穿整個生命週期（<code>requires_payment_method</code> → <code>requires_action</code> → <code>processing</code> → <code>succeeded</code>/<code>canceled</code> 等）。</p>
<p style="margin-top:10px;">即使卡片被拒後換卡重試，<code>id</code> 也不會變——PMW/mweb 用它來查詢狀態、取消付款（<code>GET /v1/payment_intents/{id}</code>、<code>POST /v1/payment_intents/{id}/cancel</code>）。</p>
</div>
<div class="sq-cbox ch">
<div class="sq-chead"><span class="sq-cicon">🧾</span><h4>Charge（<code>ch_...</code>）</h4></div>
<p>代表 PaymentIntent 底下「實際成功扣款的那一筆交易紀錄」，只有 <code>status = succeeded</code> 之後才會產生，是<strong>退款（Refund）的操作對象</strong>。</p>
<p style="margin-top:10px;">DirectCharge/DestinationCharge 退款都要先用 <code>GET /v1/payment_intents/{id}</code> 撈出 <code>charges.data[0].id</code> 才能對該筆 Charge 執行退款。一個 PaymentIntent 理論上最多對應一筆成功的 Charge；失敗重試的嘗試不會產生 Charge。</p>
</div>
</div>

<div class="sq-note info">
<span class="sq-icon">💡</span>
<p><strong>簡單比喻：</strong>PaymentIntent 是這筆訂單的付款流程/狀態機，Charge 是流程走到成功那一刻留下的扣款收據。</p>
<p>這也是為什麼只有 <code>succeeded</code> 狀態才會同時帶出 <code>payment_intent_id</code> + <code>charge_id</code>，其餘狀態（如 <code>requires_payment_method</code>、<code>canceled</code>）只有 <code>payment_intent_id</code>。</p>
</div>

<!-- endtab -->

<!-- tab DirectCharge / DestinationCharge 差異-->

同一筆 `transaction_id`，使用不同 `payment_flow` 查詢，**結果可能不同**，因為 Header 帶法會改變 Stripe 實際查詢的帳號範圍：

<table class="sq-table">
<tr><th>比較項目</th><th>DirectCharge</th><th>DestinationCharge</th></tr>
<tr><td><code>Stripe-Account</code> Header</td><td>✅ 帶入子帳號</td><td>❌ 不帶</td></tr>
<tr><td>查詢對象</td><td>子帳號下的 PaymentIntent</td><td>主帳號下的 PaymentIntent</td></tr>
<tr><td>subAcct 判斷</td><td><code>request.ExtendInfo.SubAccount</code></td><td><code>null</code></td></tr>
</table>

<table class="sq-table">
<tr><th>情況</th><th>行為</th></tr>
<tr><td>DirectCharge 帶正確 sub_account</td><td><span class="sq-tag ok">✅ 正常查詢</span></td></tr>
<tr><td>DirectCharge 帶錯誤 sub_account</td><td><span class="sq-tag danger">❌ Stripe 回 404</span>（PaymentIntent 不屬於此帳號）→ <code>2003</code></td></tr>
<tr><td>DestinationCharge 不帶 sub_account</td><td><span class="sq-tag ok">✅ 從主帳號查詢，可正常取得</span></td></tr>
<tr><td>DestinationCharge 誤傳 DirectCharge 的 payment_flow</td><td><span class="sq-tag danger">❌ 因不帶 Header 而查不到子帳號的 PaymentIntent</span> → <code>2003</code></td></tr>
</table>

<div class="sq-note danger">
<span class="sq-icon">🎯</span>
<p><strong>結論：</strong><code>payment_flow</code> 必須與 Pay 時使用的 flow 完全一致，否則會因為查詢帳號範圍錯位而拿到 <code>2003</code>，容易被誤判為「付款狀態不明」而非「查詢參數帶錯」。</p>
</div>

<!-- endtab -->

<!-- tab 情境決策樹-->

把整支 API 的判斷邏輯畫成一張樹狀圖，方便快速定位「收到某個回應時，究竟命中哪一條分支」：

<div class="sq-diagram">
<pre class="sq-diagram-pre">收到 QueryPayment 請求
        │
        ▼
GET /v1/payment_intents/{id}
        │
   ┌────┴──────────────────────────────────┐
   ▼                                       ▼
API 成功                               ApiException（4xx/5xx）
   │                                       │
   │                                   ReturnCode = 2003
   ▼                                   ReturnMessage = "status code: xxx, ..."
status？
   │
   ├─ "succeeded"
   │       └─ ReturnCode = 0000
   │          ExtendInfo: payment_intent_id + charge_id
   │
   ├─ "requires_payment_method" + last_payment_error
   │       └─ ReturnCode = 3000
   │          ExtendInfo: 錯誤詳情 + decline_code
   │
   ├─ "requires_action"
   ├─ "requires_payment_method"（無錯誤）
   ├─ "requires_confirmation"
   │       └─ ReturnCode = 2003（持續等待）
   │          ExtendInfo: null
   │
   └─ "canceled" / 其他
           └─ ReturnCode = 9001（UnhandledException）
              logger.LogWarning 記錄
              ExtendInfo: null</pre>
</div>

<!-- endtab -->

<!-- tab 8 種真實情境全解析-->

以下 8 種情境涵蓋了 QueryPayment 在真實環境中會遇到的所有代表性狀況，每個情境都附上觸發條件、Stripe 狀態變化，以及對應的 Request / Response 範例。

<table class="sq-table">
<tr><th>#</th><th>情境名稱</th><th>觸發來源</th><th>ReturnCode</th></tr>
<tr><td>1</td><td>3D 驗證完成 → 付款成功</td><td>Pay 回 2003，用戶完成 3D</td><td><span class="sq-tag ok">0000</span></td></tr>
<tr><td>2</td><td>3D 驗證中 → 用戶尚未操作</td><td>Pay 回 2003，立即查詢</td><td><span class="sq-tag warn">2003</span></td></tr>
<tr><td>3</td><td>3D 驗證完成 → 付款被拒</td><td>3D 通過但發卡行最終拒絕</td><td><span class="sq-tag danger">3000</span></td></tr>
<tr><td>4</td><td>未觸發 3D → 直接付款成功</td><td>Pay 直接 succeeded，主動確認</td><td><span class="sq-tag ok">0000</span></td></tr>
<tr><td>5</td><td>卡片被拒（明確原因）</td><td>Stripe 直接拒卡</td><td><span class="sq-tag danger">3000</span></td></tr>
<tr><td>6</td><td>查詢不存在的 PaymentIntent</td><td>transaction_id 錯誤或過期</td><td><span class="sq-tag warn">2003</span></td></tr>
<tr><td>7</td><td>PaymentIntent 已被取消</td><td>先 Cancel 後查詢</td><td><span class="sq-tag info">9001</span></td></tr>
<tr><td>8</td><td>Stripe 服務暫時異常</td><td>Stripe API 回 5xx</td><td><span class="sq-tag warn">2003</span></td></tr>
</table>

<div class="sq-scenario ok">
<div class="sq-banner ok">
<span class="sq-banner-num">1</span>
<span class="sq-banner-title">3D 驗證完成 → 付款成功</span>
<span class="sq-tag ok">ReturnCode 0000</span>
</div>

用戶在 Pay 時，Stripe 判斷此張信用卡需要 3D 驗證（SCA 要求），Pay API 回傳 `2003` 並附上 3D 驗證的導頁 URL。用戶完成銀行 3D 驗證後，系統回呼並查詢最終付款結果。

<span class="sq-chain">requires_action → （用戶完成 3D）→ succeeded</span>

**觸發條件：**Pay 回應 `return_code: 2003`、`PaymentIntent.status = requires_action`、用戶已完成銀行端的 3D 驗證。

**Request**
```json
{
  "request_id": "query-001",
  "transaction_id": "pi_3PabcXXXXXXXXXXXX",
  "extend_info": {
    "payment_flow": "DirectCharge",
    "stripe_account": "acct_1A2B3C4D5E"
  }
}
```

**Response**
```json
{
  "request_id": "query-001",
  "return_code": "0000",
  "return_message": "succeeded",
  "transaction_id": "pi_3PabcXXXXXXXXXXXX",
  "extend_info": {
    "payment_intent_id": "pi_3PabcXXXXXXXXXXXX",
    "charge_id": "ch_3PabcXXXXXXXXXXXX"
  }
}
```

</div>

<div class="sq-scenario warn">
<div class="sq-banner warn">
<span class="sq-banner-num">2</span>
<span class="sq-banner-title">3D 驗證中 → 用戶尚未完成操作</span>
<span class="sq-tag warn">ReturnCode 2003</span>
</div>

Pay 後系統立即或過短時間內查詢，用戶仍在銀行的 3D 驗證頁面尚未完成操作，PaymentIntent 狀態仍停留在 `requires_action`。

<span class="sq-chain">requires_action（持續中）</span>

**觸發條件：**Pay 回應 `return_code: 2003`、用戶尚未完成 3D 驗證、查詢時 PaymentIntent 仍為 `requires_action`。

**Request**
```json
{
  "request_id": "query-002",
  "transaction_id": "pi_3PabcXXXXXXXXXXXX",
  "extend_info": {
    "payment_flow": "DirectCharge",
    "stripe_account": "acct_1A2B3C4D5E"
  }
}
```

**Response**
```json
{
  "request_id": "query-002",
  "return_code": "2003",
  "return_message": "requires_action",
  "transaction_id": "pi_3PabcXXXXXXXXXXXX",
  "extend_info": null
}
```

<div class="sq-note info">
<span class="sq-icon">💡</span>
<p>上游系統應繼續輪詢，直到 status 從 <code>requires_action</code> 轉為 <code>succeeded</code> 或 <code>requires_payment_method</code>（含錯誤）。</p>
</div>

</div>

<div class="sq-scenario danger">
<div class="sq-banner danger">
<span class="sq-banner-num">3</span>
<span class="sq-banner-title">3D 驗證完成 → 付款最終被拒</span>
<span class="sq-tag danger">ReturnCode 3000</span>
</div>

用戶完成 3D 驗證，但發卡行在最終授權階段仍拒絕此筆交易（例如額度不足、帳戶異常等），Stripe 將 PaymentIntent 狀態設為 `requires_payment_method` 並附上 `last_payment_error`。

<span class="sq-chain">requires_action → （3D 完成）→ requires_payment_method（附 last_payment_error）</span>

**觸發條件：**用戶完成了 3D 驗證、發卡行在授權時拒絕、`PaymentIntent.status = requires_payment_method` 且 `last_payment_error != null`。

**Request**
```json
{
  "request_id": "query-003",
  "transaction_id": "pi_3PabcXXXXXXXXXXXX",
  "extend_info": {
    "payment_flow": "DestinationCharge",
    "stripe_account": "acct_1A2B3C4D5E"
  }
}
```

**Response**
```json
{
  "request_id": "query-003",
  "return_code": "3000",
  "return_message": "Your card has insufficient funds.",
  "transaction_id": "pi_3PabcXXXXXXXXXXXX",
  "extend_info": {
    "status": "requires_payment_method",
    "last_payment_error_code": "card_declined",
    "last_payment_error_decline_code": "insufficient_funds",
    "last_payment_error_message": "Your card has insufficient funds.",
    "last_payment_error_type": "card_error"
  }
}
```

</div>

<div class="sq-scenario ok">
<div class="sq-banner ok">
<span class="sq-banner-num">4</span>
<span class="sq-banner-title">未觸發 3D → 主動確認付款成功</span>
<span class="sq-tag ok">ReturnCode 0000</span>
</div>

Pay 時 Stripe 判斷為低風險交易，直接回傳 `succeeded`（`return_code: 0000`）。上游系統為確保資料一致性，仍主動呼叫 QueryPayment 二次確認。

<span class="sq-chain">succeeded</span>

**觸發條件：**Pay 已回傳 `return_code: 0000`、主動發起查詢以二次確認。

**Request**
```json
{
  "request_id": "query-004",
  "transaction_id": "pi_3PabcXXXXXXXXXXXX",
  "extend_info": {
    "payment_flow": "DirectCharge",
    "stripe_account": "acct_1A2B3C4D5E"
  }
}
```

**Response**
```json
{
  "request_id": "query-004",
  "return_code": "0000",
  "return_message": "succeeded",
  "transaction_id": "pi_3PabcXXXXXXXXXXXX",
  "extend_info": {
    "payment_intent_id": "pi_3PabcXXXXXXXXXXXX",
    "charge_id": "ch_3PabcXXXXXXXXXXXX"
  }
}
```

</div>

<div class="sq-scenario danger">
<div class="sq-banner danger">
<span class="sq-banner-num">5</span>
<span class="sq-banner-title">卡片被 Stripe 直接拒絕（有明確 decline_code）</span>
<span class="sq-tag danger">ReturnCode 3000</span>
</div>

Pay 階段，Stripe 建立 PaymentIntent 並立即確認，但信用卡被拒絕（例如卡號無效、已過期、被停用）。此時 Pay 本身就回傳 `3000`，QueryPayment 若再查詢，同樣取得拒絕狀態。

<span class="sq-chain">requires_payment_method（含 last_payment_error）</span>

**常見 decline_code 一覽：**

<table class="sq-table">
<tr><th>decline_code</th><th>原因</th></tr>
<tr><td><code>insufficient_funds</code></td><td>餘額不足</td></tr>
<tr><td><code>card_declined</code></td><td>發卡行拒絕（未提供細節）</td></tr>
<tr><td><code>expired_card</code></td><td>信用卡已過期</td></tr>
<tr><td><code>incorrect_cvc</code></td><td>CVV 錯誤</td></tr>
<tr><td><code>stolen_card</code></td><td>疑似被盜卡</td></tr>
<tr><td><code>do_not_honor</code></td><td>發卡行要求拒絕，原因不明</td></tr>
<tr><td><code>fraudulent</code></td><td>Stripe 風險判斷為詐欺</td></tr>
</table>

**Request**
```json
{
  "request_id": "query-005",
  "transaction_id": "pi_3PabcXXXXXXXXXXXX",
  "extend_info": {
    "payment_flow": "DirectCharge",
    "stripe_account": "acct_1A2B3C4D5E"
  }
}
```

**Response**
```json
{
  "request_id": "query-005",
  "return_code": "3000",
  "return_message": "Your card is expired.",
  "transaction_id": "pi_3PabcXXXXXXXXXXXX",
  "extend_info": {
    "status": "requires_payment_method",
    "last_payment_error_code": "expired_card",
    "last_payment_error_decline_code": "expired_card",
    "last_payment_error_message": "Your card is expired.",
    "last_payment_error_type": "card_error"
  }
}
```

</div>

<div class="sq-scenario warn">
<div class="sq-banner warn">
<span class="sq-banner-num">6</span>
<span class="sq-banner-title">查詢不存在的 PaymentIntent</span>
<span class="sq-tag warn">ReturnCode 2003</span>
</div>

傳入錯誤的 `transaction_id`（打錯 ID、ID 不屬於此帳號、測試環境 ID 用於正式環境等），Stripe 回傳 HTTP 404，程式碼進入 `ApiException` 處理。

**觸發條件：**`transaction_id` 格式錯誤、不存在或不屬於查詢帳號，Stripe 回傳 HTTP 404。

**程式碼處理路徑：**
```csharp
catch (ApiException ex)
{
    return new QueryPaymentResponseEntity
    {
        ReturnCode = ReturnCodes.WaitingToPay,   // "2003"
        ReturnMessage = $"status code: 404, message: No such payment_intent: '{pi_xxx}'"
    };
}
```

**Request**
```json
{
  "request_id": "query-006",
  "transaction_id": "pi_INVALID_OR_WRONG",
  "extend_info": {
    "payment_flow": "DirectCharge",
    "stripe_account": "acct_1A2B3C4D5E"
  }
}
```

**Response**
```json
{
  "request_id": "query-006",
  "return_code": "2003",
  "return_message": "status code: 404, message: No such payment_intent: 'pi_INVALID_OR_WRONG'",
  "transaction_id": "",
  "extend_info": null
}
```

<div class="sq-note warn">
<span class="sq-icon">⚠️</span>
<p>ReturnCode 為 <code>2003</code> 而<strong>非</strong> <code>3000</code>，設計上避免因查詢失敗而誤判付款失敗；<code>transaction_id</code> 回傳<strong>空字串</strong>（非傳入值）；上游系統應記錄此情況並人工確認，不應直接視為付款失敗。</p>
</div>

</div>

<div class="sq-scenario info">
<div class="sq-banner info">
<span class="sq-banner-num">7</span>
<span class="sq-banner-title">PaymentIntent 已被取消（Cancel 後再查詢）</span>
<span class="sq-tag info">ReturnCode 9001</span>
</div>

付款請求先被呼叫 Cancel API 取消，之後再查詢其狀態。Stripe PaymentIntent 狀態為 `canceled`，程式碼進入 `else`（UnhandledException）分支。

<span class="sq-chain">canceled</span>

**觸發條件：**先呼叫 `POST /api/v1/Cancel/...` 取消成功（ReturnCode `5000`），再呼叫 QueryPayment 查詢同一筆。

**程式碼處理路徑：**
```csharp
else
{
    _logger.LogWarning($"Payment Exception. PaymentIntentResponseEntity: ...");
    return (ReturnCodes.UnhandledException, status, null);   // "9001"
}
```

**Request**
```json
{
  "request_id": "query-007",
  "transaction_id": "pi_3PabcXXXXXXXXXXXX",
  "extend_info": {
    "payment_flow": "DirectCharge",
    "stripe_account": "acct_1A2B3C4D5E"
  }
}
```

**Response**
```json
{
  "request_id": "query-007",
  "return_code": "9001",
  "return_message": "canceled",
  "transaction_id": "pi_3PabcXXXXXXXXXXXX",
  "extend_info": null
}
```

<div class="sq-note info">
<span class="sq-icon">💡</span>
<p><code>9001</code> 是 <code>UnhandledException</code>，系統沒有針對 <code>canceled</code> 定義特定行為，同時會觸發 <code>logger.LogWarning</code> 並記錄完整 PaymentIntent JSON；上游系統若收到 <code>9001</code>，應自行判斷是否為已取消場景。</p>
</div>

</div>

<div class="sq-scenario warn">
<div class="sq-banner warn">
<span class="sq-banner-num">8</span>
<span class="sq-banner-title">Stripe 服務暫時異常（5xx）</span>
<span class="sq-tag warn">ReturnCode 2003</span>
</div>

Stripe 服務端發生暫時性錯誤（例如 503 Service Unavailable），查詢失敗並進入 `ApiException` 處理。

**觸發條件：**Stripe API 回傳 HTTP 5xx；若為網路超時或連線失敗，則會進入 `Exception`（非 `ApiException`）分支。

<ul class="sq-checklist">
<li><strong>5xx（ApiException）</strong> → ReturnCode <code>2003</code></li>
<li><strong>網路超時（Exception）</strong> → <code>logger.LogError</code> → <code>throw</code> → HTTP 500</li>
</ul>

**Response（5xx 情況）**
```json
{
  "request_id": "query-008",
  "return_code": "2003",
  "return_message": "status code: 503, message: Service temporarily unavailable.",
  "transaction_id": "",
  "extend_info": null
}
```

</div>

<!-- endtab -->

{% endtabs %}
