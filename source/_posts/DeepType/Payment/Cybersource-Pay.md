---
title: Cybersource-Pay
date: 2026-07-02 18:19:00
categories: Payment
top_img: https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Pay/landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Pay/landing.png
tags:
toc:
toc_number:
comments :
---

{% tabs Cybersource_Pay%}


<!-- tab 表單導向，無直接 API 呼叫-->

Cybersource 的 Pay 設計與多數金流根本不同。PMW **不呼叫 Cybersource 任何 API**，他的做法是



1. 組建一份已簽名的 HTML 表單（FormPost）
2. 前台讓瀏覽器直接將這份表單 POST 到 Cybersource 托管頁
3. 消費者在 Cybersource 頁面完成付款後，Cybersource 將結果 POST 回前台 callback URL



![5](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Pay/1-no-api-call.png)


```
mweb 前台
  └─ POST /api/Pay → PaymentMiddlewareService
        └─ PMW CybersourcePlugin.Pay()
              └─ 組建 CybersourcePaymentFormEntity
              └─ HMAC-SHA256 簽名（SecureAcceptanceHelper）
              └─ 回傳 FormPostAction（ActionUrl + FormData）
                    ↓
              瀏覽器 POST → https://secureacceptance.cybersource.com/pay
                                  ↓
                            消費者輸入信用卡完成付款
                                  ↓
                            Cybersource POST → /V2/PayChannel/CreditCardOnce/Cybersource/Callback
                                  ↓
                            CybersourcePayChannelService.ProcessResponseFormAfterPayment()
                                  ↓
                            前台 redirect → /V2/PayChannel/CreditCardOnce/Cybersource/{TGCode}?shopId=&k=
```

> **設計哲學**：卡號等敏感資訊完全不經過商戶系統（mweb、PMW），由 Cybersource 的托管頁直接收取，大幅縮減 PCI DSS 合規範圍


![5](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Pay/3-the-pay-process.png)


<!-- endtab -->


<!-- tab 兩套認證體系-->



Cybersource 有兩套平行的認證機制，用途不同

| 認證集 | 欄位 | 用途 |
|:---|:---|:---|
| **REST API 認證** | `merchantId` / `merchantKeyId` / `secretKey` | 呼叫 Cybersource REST API（Query、Refund 用） |
| **Secure Acceptance Profile 認證** | `profileId` / `profileAccessKey` / `profileSecret` | 組建表單、HMAC-SHA256 簽名（Pay 用）|

Pay 流程**只使用 Secure Acceptance Profile 認證**，REST API 認證在 Pay 時僅傳入 Header 但未使用。



![5](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Pay/2-two-auth.png)


<!-- endtab -->


<!-- tab 流程概覽-->


```
mweb GetHeader()          → 從 ShopSecret 讀取 6 個認證欄位 → 放入 HTTP Headers
mweb GetPayExtendInfo()   → { lang, uniqueKey } → 放入 request.ExtendInfo

PMW CybersourcePlugin.Pay()
  ├─ CybersourceConfiguration     → 從 Headers 建立認證物件
  ├─ CybersourcePaymentFormEntity → 組建 11 個表單欄位
  │    └─ ToDictionary()          → 所有欄位 key 串成 signed_field_names
  ├─ SecureAcceptanceHelper.sign()
  │    ├─ buildDataToSign()       → "key=value,key=value,..." 格式
  │    └─ HMACSHA256 + Base64     → signature
  └─ RequiredFormPostAction()     → ReturnCode=2003, ActionUrl, FormData

mweb UpdateProcessContextForRedirect()
  └─ PostData = FormData, RedirectHttpMethod = "POST"

Browser → POST FormData → Cybersource Secure Acceptance URL
  └─ 消費者輸入信用卡，Cybersource 完成付款

Cybersource → POST /PayChannel/ReturnPost/CreditCardOnce/Cybersource
  └─ PayChannelReturnPost()                ← 不需要 session（無 [RequireLoginGoLoginPage]）
       └─ ProcessResponseFormAfterPayment()
            ├─ Request.Form["req_reference_number"] → TGCode
            ├─ Request.Form["req_transaction_uuid"] → UniqueKey
            └─ 收集所有欄位到 ExtendData
       └─ return Content(HTML form + auto-submit)
            └─ 組 <form action='/V2/PayChannel/CreditCardOnce/Cybersource/{TGCode}?k='>
               把所有欄位包成 extendData.{key} 格式

Browser 自動 POST（帶著 session cookie）
  └─ PayChannelReturn(tgCode, k, extendData)  ← 需要 session（有 [RequireLoginGoLoginPage]）
       └─ FinishPayment(extendData != null → Form-Data 路徑)
            └─ GetQueryPaymentExtendInfo() → 解析所有 extendData
            └─ PMW QueryPayment → 驗簽 + decision 判斷
                 ├─ "ACCEPT"                → ReturnCode = 1000 → /V2/Pay/Finish
                 └─ "DECLINE"/"ERROR"/"CANCEL" → ReturnCode = Failed → 錯誤頁
```


<!-- endtab -->


<!-- tab Pay 行為概覽-->


## Phase 1：mweb 端準備（CybersourcePayChannelService）


#### 1-1. 從 ShopSecret 讀取認證（GetHeader）

- Cybersource_ProfileId
- Cybersource_ProfileAccessKey
- Cybersource_ProfileSecret
- Cybersource_MerchantId
- Cybersource_SecretKey
- Cybersource_MerchantKeyId

#### 1-2. 組建 Pay ExtendInfo（GetPayExtendInfo）


- uniqueKey
- lang


## Phase 2：PMW 端處理（CybersourcePlugin.Pay）

| 環境 | PaymentUrl |
|:---|:---|
| Test | `https://testsecureacceptance.cybersource.com/pay` |
| Production | `https://secureacceptance.cybersource.com/pay` |


#### 2-2. 組建表單（CybersourcePaymentFormEntity）

| 欄位（表單 key）| 值來源 | 說明 |
|:---|:---|:---|
| `access_key` | `config.ProfileAccessKey` | Secure Acceptance 識別用 |
| `profile_id` | `config.ProfileId` | Profile 識別 |
| `transaction_uuid` | `request.ExtendInfo.UniqueKey` | 防重複提交唯一碼（≤ 50 字元）|
| `signed_field_names` | 所有欄位 key 逗號串接（含自身）| 告知 Cybersource 哪些欄位被簽名 |
| `unsigned_field_names` | `""` | 本實作無未簽名欄位 |
| `signed_date_time` | `DateTime.UtcNow.ToString("yyyy-MM-dd'T'HH:mm:ss'Z'")` | UTC 時間戳，Cybersource 驗簽用 |
| `locale` | `request.ExtendInfo.Lang` | 語系（如 `zh-tw`）|
| `transaction_type` | `"sale"` | 授權 + 請款一次完成（非兩階段）|
| `reference_number` | `request.TradesOrderGroupCode` | TG Code，對應訂單號 |
| `amount` | `request.Amount.ToString()` | 付款金額 |
| `currency` | `request.Currency` | 幣別（如 `HKD`）|


> `signed_field_names` 的值是以上所有欄位 key 的逗號串接字串（包含 `signed_field_names` 本身），且在 `ToDictionary()` 中先把所有 key 寫入後才回填此欄位值。


#### 2-3. HMAC-SHA256 簽名（SecureAcceptanceHelper）

```csharp
var formData = payment.ToDictionary();
var signature = SecureAcceptanceHelper.sign(config.ProfileSecret, formData);
formData.Add("signature", signature);
```

#### 2-4. 組建回傳（RequiredFormPostAction）


PMW 回傳 `ReturnCode = 2003（WaitingToPay）`，這是 Pay 流程的**正常結果**，代表「請前台引導瀏覽器 POST 此表單」。


## Phase 2 完整回傳結構

```json
{
  "ReturnCode": "2003",
  "ReturnMessage": null,
  "TransactionId": null,
  "RequiredAction": "FormPost",
  "ActionUrl": "https://testsecureacceptance.cybersource.com/pay",
  "FormData": {
    "access_key": "abc123def456",
    "profile_id": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
    "transaction_uuid": "K20260702001",
    "signed_field_names": "access_key,profile_id,transaction_uuid,signed_field_names,unsigned_field_names,signed_date_time,locale,transaction_type,reference_number,amount,currency",
    "unsigned_field_names": "",
    "signed_date_time": "2026-07-02T03:56:00Z",
    "locale": "zh-tw",
    "transaction_type": "sale",
    "reference_number": "TG260702M00043",
    "amount": "618.30",
    "currency": "HKD",
    "signature": "Base64EncodedHMACSignature=="
  }
}
```



![5](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Pay/4-two-ways-sign.png)


<!-- endtab -->


<!-- tab Cybersource Callback 兩跳機制-->




## 為什麼 Callback 需要兩跳？

Cybersource Profile 設定的 callback URL 是固定的，**不含 TGCode**：

```
POST /PayChannel/ReturnPost/CreditCardOnce/Cybersource   ← Cybersource 只知道這個
```

但後續真正做驗簽與付款確認的 `PayChannelReturn` route 需要 TGCode 在 URL 裡，且需要 login session：

```csharp
[RequireLoginGoLoginPage]   // ← 需要 browser session cookie
public ActionResult PayChannelReturn(string payMethod, string payChannel,
    string tgCode, string k, IDictionary<string, string> extendData = null)
```

所以 `PayChannelReturnPost` 扮演**中繼轉接站**的角色。



![5](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Pay/5-two-break-from-cybersource-callback.png)



## 第一跳：Cybersource → PayChannelReturnPost

```csharp
// PayChannelController.PayChannelReturnPost()
// Route: /PayChannel/ReturnPost/{payMethod}/{payChannel}
// 注意：此 action 沒有 [RequireLoginGoLoginPage]，Cybersource callback 不帶 session

public ActionResult PayChannelReturnPost(string payMethod, string payChannel)
{
    IPayChannelService payChannelService = this._payChannelServiceResolver.Resolve(payChannel);
    var shopId = ViewBag.ShopId;

    // 從 Request.Form 讀取 Cybersource POST 的所有欄位
    PayChannelResponseEntity response = payChannelService.ProcessResponseFormAfterPayment(shopId, Request);

    // 組建隱藏表單，讓 browser 自動 POST 到下一個 URL
    var formBuilder = new StringBuilder();
    formBuilder.AppendLine($"<form id='autoSubmitForm' action='{response.RedirectUrl}' method='post'>");
    foreach (var pair in response.ExtendData)
    {
        // 所有 Cybersource 欄位轉成 extendData.{key} 格式
        formBuilder.AppendLine($"<input type='hidden' name='extendData.{HttpUtility.UrlEncode(pair.Key)}' value='{HttpUtility.UrlEncode(pair.Value.ToString())}' />");
    }
    formBuilder.AppendLine("</form>");
    formBuilder.AppendLine("<script type='text/javascript'>document.getElementById('autoSubmitForm').submit();</script>");

    return Content(formBuilder.ToString());  // 回傳 HTML 給 browser
}
```

**`ProcessResponseFormAfterPayment()` 做的事：**

```csharp
// CybersourcePayChannelService.ProcessResponseFormAfterPayment()
var tgCode    = request.Form["req_reference_number"];   // TGCode 從 Form-Data 取出
var uniqueKey = request.Form["req_transaction_uuid"];   // UniqueKey 從 Form-Data 取出

var formData = new Dictionary<string, object>();
foreach (var key in request.Form.AllKeys)
    formData.Add(key, request.Form[key]);   // 全部 Cybersource 欄位收走

return new PayChannelResponseEntity
{
    RedirectUrl = $"/V2/PayChannel/CreditCardOnce/Cybersource/{tgCode}?shopId={shopId}&k={uniqueKey}",
    TradesOrderGroupCode = tgCode,
    UniqueKey = uniqueKey,
    ExtendData = formData
};
```

> Cybersource 回傳欄位有 `req_` 前綴（表示「request echo」）。TGCode 對應 `req_reference_number`，UniqueKey 對應 `req_transaction_uuid`。






![5](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Pay/6-two-hops.png)






## 為什麼用 `return Content(HTML)` 而不是直接 POST 到 PayChannelReturn？

| 方案 | 可行？ | 問題 |
|:---|:---:|:---|
| `return Redirect(url)` | ❌ | HTTP 302 redirect 永遠變成 GET，POST body 消失 |
| `HttpClient.SendAsync()` 打 PayChannelReturn | ❌ | Server 發出的 request 不帶 browser session cookie，`[RequireLoginGoLoginPage]` 直接擋掉 |
| **`return Content(HTML form + auto-submit)`（現況）** | ✅ | Browser 自己 POST，天然帶著 session cookie |

`return Content()` 只是回傳一段 HTML，**browser 從來沒有離開**，session cookie 全程保留。browser 執行 JavaScript 自動 submit form，這次 POST 等同使用者自己按按鈕，cookie 自然帶著。


## 第二跳：PayChannelReturnPost HTML → PayChannelReturn

browser 收到 HTML 後自動 POST：

```html
<form method="post" action="/V2/PayChannel/CreditCardOnce/Cybersource/TG260702M00043?shopId=2&k=K2026...">
  <input type="hidden" name="extendData.decision"              value="ACCEPT" />
  <input type="hidden" name="extendData.signature"             value="xxx==" />
  <input type="hidden" name="extendData.req_reference_number"  value="TG260702M00043" />
  <input type="hidden" name="extendData.req_transaction_uuid"  value="K2026..." />
  <!-- ... 所有 Cybersource 欄位 ... -->
</form>
<script>document.getElementById('autoSubmitForm').submit();</script>
```

ASP.NET MVC model binding 自動將 `extendData.{key}` 格式解析為 `IDictionary<string, string> extendData` 參數。

`PayChannelReturn` 收到後：
- `tgCode`、`k` 來自 URL
- `extendData`（含 `decision`、`signature` 等）來自 POST body
- 呼叫 `FinishPayment(extendData != null → Form-Data 路徑)` → 驗簽 → ReturnCode



![5](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Pay/7-auto-submits.png)



<!-- endtab -->


<!-- tab Secure Acceptance 托管頁模式：Pay() 不知道結果-->


Cybersource Pay 採用托管頁（Hosted Page）模式，PMW 的 `Pay()` **只組表單、不呼叫任何 API**，回傳 `ReturnCode = 2003`（FormPostAction）後，付款結果完全由 Cybersource callback 決定

```
PMW Pay()  →  ReturnCode = 2003（組完表單就結束）
                    ↑
              完全不知道付款會成功或失敗

付款結果唯一來源：Cybersource callback POST decision 欄位
```



**優點**

| 優點 | 說明 |
|:---|:---|
| ✅ PCI DSS 合規範圍最小 | 卡號完全不經過 mweb / PMW，由 Cybersource 托管頁收取，大幅降低合規成本 |
| ✅ 無需自建 3DS 驗證流程 | Cybersource 托管頁內建 3D Secure，商戶完全不需處理 |
| ✅ Pay() 邏輯極簡 | 純 CPU 計算（簽名），無外部 API 呼叫，不會因金流商 API 超時而失敗 |
| ✅ 簽名防竄改 | HMAC-SHA256 雙向驗證，Pay 表單與 callback 結果都有防偽 |



**缺點**

| 缺點 | 說明 |
|:---|:---|
| ❌ Pay() 無法即時知道失敗 | 結果必須等 callback 打回來才知道，PMW 層完全非同步 |
| ❌ callback 可能不來 | 消費者關閉 Cybersource 頁面 → callback 不觸發 → 只能靠 ReCheck Job 輪詢補救 |
| ❌ Create Search API 有索引延遲 | ReCheck Job 路徑有 eventual consistency 問題，不能立刻查到結果 |
| ❌ 付款 UI 完全由 Cybersource 控制 | 商戶無法自訂卡號輸入頁的 UI，只能調整 Cybersource Profile 設定 |

<!-- endtab -->


<!-- tab transaction_type 固定為 `"sale"`-->



`sale = authorization + capture` 一次完成，不走先授權後請款的兩階段流程。適合零售電商直接扣款，無需人工審核後再請款的場景



## 簽名雙向驗證

| 方向 | 誰簽 | 誰驗 |
|:---|:---|:---|
| Pay 表單 → Cybersource | PMW（`SecureAcceptanceHelper`）| Cybersource |
| Cybersource callback → mweb | Cybersource | PMW（`SecureAcceptanceHelper`）|

相同的 `ProfileSecret` 用於兩個方向，確保表單內容未被竄改。


<!-- endtab -->


<!-- tab 兩條 QueryPayment 路徑-->



Cybersource 的 `QueryPayment` 有兩條完全不同的路徑，由 `extendData` 是否存在來分支：

```csharp
// CybersourcePlugin.QueryPayment()
bool isSearchByTG = request.ExtendInfo.ContainsKey("TradesOrderGroupCode");
if (isSearchByTG)
    // 路徑 A：由 Create Search API 查詢
    var response = await this._cybersourceHttpClient.QueryPaymentAsync(request, headers);
else
    // 路徑 B：直接解析 Form-Data（驗簽）
    return this.ConvertToQueryPaymentResponse(request);
```

| 路徑 | 觸發條件 | 說明 |
|:---|:---|:---|
| **Form-Data 路徑**（路徑 B） | callback 時 extendData 有帶回（`extendData != null`） | 直接驗 Cybersource 回傳的 signature，解析 `decision` 欄位 |
| **Create Search API 路徑**（路徑 A） | ReCheck Job 輪詢，無 extendData，帶 `TradesOrderGroupCode` | 呼叫 Cybersource REST API 搜尋索引查詢 |




![5](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Pay/8-query-two-paths.png)



## 為什麼要優先走 Form-Data 路徑？

Cybersource 的 `Create Search API` 本質是**搜尋索引查詢**，而搜尋索引**不是即時同步**的（有 eventual consistency 問題）：

```
消費者付款完成
    ↓  （立刻）
Cybersource 主交易庫寫入
    ↓  （有延遲，幾秒～數十秒不等）
搜尋索引同步完成
    ↓
Create Search API 才查得到這筆
```

若 callback 一到就立刻呼叫 Create Search API，有機率**索引尚未同步 → 查無此交易 → 系統誤判付款失敗 → 消費者無法進付款完成頁**

Form-Data 路徑完全不碰 API，直接用 Cybersource callback 夾帶的 form-data 本身驗簽，**繞過索引延遲問題**




![5](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Pay/9-search-api-delay.png)


<!-- endtab -->


<!-- tab summary-->




![5](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Pay/final-table.png)





<!-- endtab -->



{% endtabs %}