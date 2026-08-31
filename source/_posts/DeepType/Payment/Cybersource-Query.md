---
title: Cybersource-Query
date: 2026-07-03 10:44:00
categories: Payment
top_img: https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Query/landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Query/landing.png
tags:
toc:
toc_number:
comments :
---

{% tabs Cybersource-Query%}


<!-- tab 雙路徑設計-->


Cybersource `QueryPayment` 有兩條完全不同的執行路徑，由 `request.ExtendInfo` 是否包含 `TradesOrderGroupCode` 決定。

```
QueryPayment 入口
│
├── ExtendInfo 有 TradesOrderGroupCode？
│   ├── YES → 【路徑 A】Create Search API 查詢（ReCheck Job 路徑）
│   └── NO  → 【路徑 B】Form-Data 簽名驗證（Callback 快速路徑）
```

**背景設計決策（程式碼原始 comment）：**
```
//// 由於 Cybersource 的 Create Search API 會有同步資料的延遲
//// 為避免因為同步資料的延遲導致付款後無法進入付款完成頁
//// 討論後由前台直接將 Cybersource 回傳 Form Data 帶入 PMW 解析
//// 若前台有帶入則直接透過解析 Form Data 回傳結果
//// 若無帶入則透過 Create Search API 查詢結果後回傳
```


![2](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Query/1-query-two-paths.png)



![2](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Query/2-extendinfo-tg-decide.png)


<!-- endtab -->

<!-- tab ExtendInfo 怎麼決定-->


## mweb 觸發邏輯（GetQueryPaymentExtendInfo）

mweb 的 `GetQueryPaymentExtendInfo()` 根據 `extendData` 是否存在決定走哪條路徑：

```csharp
// CybersourcePayChannelService.GetQueryPaymentExtendInfo()
public IDictionary<string, object> GetQueryPaymentExtendInfo(
    PayProcessContextEntity context,
    string queryString,
    bool isFromWorker,
    IDictionary<string, string> extendData = null)
{
    if (extendData == null)
    {
        // 路徑 A：ReCheck Job，帶 TGCode 讓 PMW 去查 API, internalfinish 的 extendData 為 null
        return new Dictionary<string, object>()
        {
            { "TradesOrderGroupCode", context.TradesOrderGroup.TradesOrderGroup_Code }
        };
    }

    // 路徑 B：callback 帶回的 form-data，URL decode 後直接傳給 PMW 驗簽,ReturnPost => PaychannelReturn 會帶 extendData
    var result = new Dictionary<string, object>();
    foreach (var pair in extendData)
    {
        string decodedValue = HttpUtility.UrlDecode(pair.Value);
        result[pair.Key] = decodedValue;
    }
    return result;
}
```

| 呼叫情境 | `extendData` | 走哪條路徑 |
|:---|:---:|:---|
| PayChannelReturn（Cybersource callback 帶回 form-data） | 有值 | 路徑 B |
| ReCheck Job / 後台補查 | `null` | 路徑 A |


![2](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Query/3-formdata-callback.png)


<!-- endtab -->


<!-- tab Form-Data 快速路徑（Callback 情境）怎麼判斷付款狀態-->



## 路徑 B：Form-Data 快速路徑（Callback 情境）

### 觸發條件
`request.ExtendInfo` **不**包含 `TradesOrderGroupCode`，直接帶 Cybersource callback 的所有 form-data 欄位。

### Step 1：HMAC-SHA256 驗簽

```csharp
var config = new CybersourceConfiguration(this._configuration, headers);
var signature = SecureAcceptanceHelper.sign(config.ProfileSecret, request.ExtendInfo);
bool isValid = signature == request.ExtendInfo.Get("signature");
if (isValid == false)
{
    throw new SecurityException("'signature' is not match");
}
```

- 使用與 Pay 表單相同的 `ProfileSecret` 重新計算簽名
- 計算結果與 Cybersource 回傳的 `signature` 欄位比對
- 不符 → 立即拋 `SecurityException`，防止偽造 callback

### Step 2：解析 `decision` 欄位

```csharp
var decision = request.ExtendInfo.Get("decision");
var returnCode = decision switch
{
    "ACCEPT"                        => ReturnCodes.Success,      // 1000
    "DECLINE" or "ERROR" or "CANCEL"=> ReturnCodes.Failed,       // 3000
    _                               => ReturnCodes.WaitingToPay  // 2003
};
```

| `decision` | ReturnCode | 說明 |
|:---|:---:|:---|
| `ACCEPT` | `1000` Success | 付款成功 |
| `DECLINE` | `3000` Failed | 發卡行拒絕 |
| `ERROR` | `3000` Failed | 系統錯誤 |
| `CANCEL` | `3000` Failed | 消費者取消 |
| 其他 | `2003` WaitingToPay | 未知狀態 |

### Step 3：組建回應

| 欄位 | 來源 |
|:---|:---|
| `TransactionId` | `request.ExtendInfo["transaction_id"]` |
| `ReturnMessage` | `request.ExtendInfo["message"]` |
| `ExtendInfo.Source` | `"FormData"` |
| `ExtendInfo.card_type_name` | `request.ExtendInfo["card_type_name"]` |
| `ExtendInfo.req_payment_method` | `request.ExtendInfo["req_payment_method"]` |



![2](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Query/4-formcallback-decision.png)


<!-- endtab -->

<!-- tab 路徑 A：Create Search API（ReCheck Job 情境）-->

### 觸發條件
`request.ExtendInfo` 包含 `TradesOrderGroupCode`（= TGCode），由 ReCheck Job 觸發。

### API 呼叫

```
POST /tss/v2/searches
Authorization: HTTP_SIGNATURE
Content-Type: application/json

{
  "query": "clientReferenceInformation.code:{TGCode}"
}
```

認證使用 **HTTP Signature**（`HttpSignatureHelper.GenerateHttpSignatureHeaders()`），帶 `x-merchant-id`、`x-merchant-key-id`、`x-secret-key` 三個 headers。

> 注意：路徑 A 使用 **REST API 認證**（merchantId / merchantKeyId / secretKey），與路徑 B 的 **Secure Acceptance 認證**（ProfileSecret）是完全不同的認證體系。

### 回傳結果判斷邏輯（4 步驟，順序不可調換）

```csharp
// Step 1：有無任何交易紀錄？
bool hasEmbedded = this.Embedded is not null &&
                   this.Embedded.TransactionSummaries?.Any() == true;
if (!hasEmbedded)
    → ReturnCode = 2003 WaitingToPay（查詢時資料尚未同步）

// Step 2：有無成功的 capture？
// Cybersource 官方判斷：applications 中有 name=="ics_bill" 且 reasonCode=="100"
var transaction = TransactionSummaries.FirstOrDefault(
    item => item.ApplicationInformation.Applications.Any(app => app.Name == "ics_bill")
         && item.ApplicationInformation.ReasonCode == "100"
);
if (transaction != null)
    → ReturnCode = 1000 Success，TransactionId = transaction.Id

// Step 3：有無確定失敗（只認 ESYSTEM）？
var failFlags = new[] { "ESYSTEM" };
transaction = TransactionSummaries.FirstOrDefault(
    item => item.ApplicationInformation.ReasonCode != "100"
         && failFlags.Contains(item.ApplicationInformation.RFlag)
);
if (transaction != null)
    → ReturnCode = 3000 Failed，ReturnMessage = application 錯誤串接

// Step 4：其他一律維持待付款
→ ReturnCode = 2003 WaitingToPay
```



![2](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Query/5-recheck-path.png)



### 為什麼判斷順序是「先找成功、再找失敗」？

Cybersource Secure Acceptance 付款失敗時**不一定**將消費者導回商戶頁，可在 Cybersource 頁面重試。因此一筆 TGCode 的搜尋結果中，可能同時存在多筆 transaction：

```
TG123456 的查詢結果：
  - transaction #1：3D 驗證失敗（未重試）
  - transaction #2：ACCEPT（重試成功）
```

**如果先找失敗再找成功 → 誤判失敗**。因此必須先找成功。




![2](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Query/6-recheck-success-fisrt.png)



### 失敗判定只認 ESYSTEM 的設計意圖

| rFlag | 含義 | 判斷結果 |
|:---|:---|:---:|
| `ESYSTEM` | 系統錯誤，無法重試 | `3000` Failed |
| 其他失敗 rFlag | 可能還在重試中（如 3D 驗證失敗）| `2003` WaitingToPay |

設計理念：**只有確定無法重試的系統錯誤才視為失敗，保留消費者在 Cybersource 頁面重試的可能性。**




![2](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Query/7-esystem-fail-only.png)



## ReturnCode 對照表

| ReturnCode | 說明 | 路徑 A 觸發情境 | 路徑 B 觸發情境 |
|:---:|:---|:---|:---|
| `1000` | Success | `ics_bill` application + `reasonCode == "100"` | `decision == "ACCEPT"` |
| `2003` | WaitingToPay | 無資料（索引延遲）/ 非 ESYSTEM 失敗 / Step 4 fallback | `decision` 非已知值 |
| `3000` | Failed | `rFlag == "ESYSTEM"` | `decision` 為 DECLINE / ERROR / CANCEL |

<!-- endtab -->


<!-- tab QueryPaymentResponseExtendInfo 欄位對照-->

| 欄位 | 路徑 A（Query） | 路徑 B（FormData） |
|:---|:---|:---|
| `Source` | `"Query"` | `"FormData"` |
| `card_type_name` | `""` | Cybersource form-data 的 `card_type_name` |
| `req_payment_method` | `""` | Cybersource form-data 的 `req_payment_method` |
| `paymentInformation_paymentType_type` | `transaction.PaymentInformation.PaymentType.Type` | `""` |
| `paymentInformation_paymentType_method` | `transaction.PaymentInformation.PaymentType.Method` | `""` |

<!-- endtab -->

<!-- tab 完整流程圖-->

```
【路徑 B — Callback 快速路徑】

Cybersource callback POST
  └─ PayChannelReturnPost（無 session 限制）
       └─ ProcessResponseFormAfterPayment()
            ├─ req_reference_number → TGCode
            └─ 所有 form-data → ExtendData
  └─ return Content(HTML form) → browser auto-submit
  └─ PayChannelReturn（需要 session）
       └─ GetQueryPaymentExtendInfo(extendData != null)
            └─ URL decode 所有 extendData 欄位
  └─ PMW QueryPayment()
       ├─ isSearchByTG = false（無 TradesOrderGroupCode）
       ├─ SecureAcceptanceHelper.sign() 驗簽
       └─ decision → ReturnCode
  └─ GetTradesOrderThirdPartyPaymentUpdateEntityOnReturnSuccess()
       └─ TransactionId = queryResult.TransactionId
       └─ 更新 TradesOrderThirdPartyPayment

【路徑 A — ReCheck Job 路徑】

ReCheck Job 觸發
  └─ GetQueryPaymentExtendInfo(extendData = null)
       └─ { TradesOrderGroupCode: "TG123456" }
  └─ PMW QueryPayment()
       ├─ isSearchByTG = true
       └─ _cybersourceHttpClient.QueryPaymentAsync()
            └─ POST /tss/v2/searches { query: "clientReferenceInformation.code:TG123456" }
  └─ CybersourceCreateSearchResponseEntity.ConvertToQueryPaymentResponse()
       ├─ Step 1：有無資料？→ 無 → 2003
       ├─ Step 2：ics_bill + reasonCode 100？→ 有 → 1000
       ├─ Step 3：ESYSTEM？→ 有 → 3000
       └─ Step 4：其他 → 2003
```



![2](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Query/8-paths-compare.png)


<!-- endtab -->


<!-- tab 情境分析-->


### Q1：付款剛完成，前台立刻查詢（路徑 B）

```
Cybersource callback → form-data（decision=ACCEPT, signature=xxx）
→ GetQueryPaymentExtendInfo(extendData 有值) → URL decode 所有欄位
→ PMW 驗簽 OK → decision=ACCEPT → ReturnCode=1000
✅ 即時確認，不受索引延遲影響
```

### Q2：消費者關掉 Cybersource 頁面（callback 未觸發）

```
→ 無 callback，ExtendData 為 null
→ ReCheck Job 觸發路徑 A
→ /tss/v2/searches 查詢
→ 若資料尚未同步 → 2003（排程重試）
→ 若查到成功 → 1000
⚠️ 需等到索引同步，有一定延遲
```

**真實 Log（已驗證）— 索引尚未同步情境：**

```json
// Request：extend_info 有 TradesOrderGroupCode → 路徑 A
{
  "request_id": "0dc15ff6-4106-42b0-b955-c9d6a5201eb6",
  "transaction_id": null,
  "country": "HK",
  "extend_info": { "TradesOrderGroupCode": "TG260703K00025" }
}

// Response：HTTP 201 Created，count = 0，無 _embedded
{
  "searchId": "909d5d51-54b8-4e33-bc61-0f040c91d293",
  "query": "clientReferenceInformation.code:TG260703K00025",
  "count": 0,
  "totalCount": 0,
  "_links": { "self": { "href": "https://api.cybersource.com/tss/v2/searches/..." } }
}
```

```
→ hasEmbedded = false（count = 0，無 _embedded）
→ Step 1 命中 → ReturnCode = 2003 WaitingToPay
→ ReCheck Job 排程重試
```

> `transaction_id` 為 `null`（Pay 時 Cybersource 不回 TransactionId），`count: 0` 代表 Cybersource 搜尋索引尚未同步，是索引延遲的直接佐證。

### Q3：3D 驗證失敗，消費者在 Cybersource 頁面重試

```
路徑 A 查詢結果：
  transaction #1：3D 失敗（rFlag 非 ESYSTEM）
→ Step 2：無 ics_bill reasonCode 100 → 找不到成功
→ Step 3：rFlag 非 ESYSTEM → 找不到確定失敗
→ Step 4：2003 WaitingToPay
✅ 系統等待，消費者可繼續重試
```

### Q4：同一訂單有多筆 transaction（重試付款場景）

```
transaction #1：3D 失敗
transaction #2：ACCEPT 成功
→ Step 2 先找成功 → 找到 #2（ics_bill + reasonCode 100）→ 1000
✅ 正確判斷為成功，不被 #1 失敗記錄干擾
```

### Q5：簽名驗證失敗（路徑 B）

```
form-data 被竄改或 ProfileSecret 設定錯誤
→ signature 不符 → SecurityException("'signature' is not match")
→ PMW 回傳錯誤，不處理付款結果
🔒 防偽造 callback 機制
```

<!-- endtab -->

<!-- tab 雙路徑設計是對 Create Search API 的妥協-->


路徑 B（Form-Data）的存在，本質上是承認了路徑 A（Create Search API）在付款完成後立刻查詢的場景下**不可信賴**。這是一個「因為 API 不夠即時，所以繞過 API」的設計決策。

這種雙軌制帶來的代價：**同一個 `QueryPayment` 方法，在兩種情境下行為完全不同**，呼叫方必須清楚知道自己在哪條路徑上，增加了心智負擔。

<!-- endtab -->

<!-- tab 路徑 B 的 `TransactionId` 來自 Cybersource form-data 的 `transaction_id`-->


路徑 B 成功時，`TransactionId = request.ExtendInfo["transaction_id"]`，這是 Cybersource 在 callback form-data 裡回傳的欄位。

路徑 A 成功時，`TransactionId = transaction.Id`（Create Search 回傳的 `transactionSummaries[].id`）。

兩條路徑的 `TransactionId` 應指同一筆交易，但**來源不同**，若 Cybersource 在兩個 API 回傳的 id 格式不一致，可能造成後續退款查找問題。




![2](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Query/9-why-two-ways.png)



![2](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Query/10-transcionid-diff.png)


<!-- endtab -->

<!-- tab WaitingToPay 語義過載-->



`ReturnCode = 2003` 在路徑 A 中代表三種完全不同的現實：

| 情境 | 現實意義 |
|:---|:---|
| `_embedded` 為空 | 資料還沒同步，稍後可能成功 |
| 有失敗 transaction 但非 ESYSTEM | 消費者可能在重試 |
| Step 4 fallback | 不明原因，系統不確定狀態 |

這三種情境對後端系統的處理策略應該不同（等待時間、是否告警等），但都用同一個 ReturnCode 表達，**ReCheck Job 無法區分**。




![2](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Query/11-2003-block.png)


<!-- endtab -->

<!-- tab ESYSTEM 白名單過於保守-->


`failFlags = new[] { "ESYSTEM" }` 只認一種確定失敗，其餘一律 WaitingToPay。

實際上 Cybersource 有多種確定性失敗的 rFlag（參考 Cybersource 文件 000001630），例如卡片餘額不足、卡號無效等情境，也可能有不可重試的狀態。**只認 ESYSTEM 可能導致某些確定失敗的訂單長期卡在 WaitingToPay，佔用 ReCheck 資源。**



<!-- endtab -->

<!-- tab 路徑 B 缺少 `decision` 非預期值的告警-->


```csharp
_ => ReturnCodes.WaitingToPay  // switch 的 default case
```

當 Cybersource 回傳未知的 `decision` 值時，靜默返回 `WaitingToPay`，沒有任何日誌或告警。若 Cybersource 新增了 `decision` 值，系統會悄悄誤判，難以排查。





![2](https://raw.githubusercontent.com/CHI-KEKE/Payment/refs/heads/master/pics/Cybersource/Query/12-silent-failure.png)


<!-- endtab -->

{% endtabs %}
