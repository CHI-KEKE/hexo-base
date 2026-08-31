---
title: PayProcessContext 流程控制
date: 2026-06-28 15:00:00
categories: Payment
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-context/landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-context/landing.png
tags:
toc:
toc_number:
comments :
---

{% tabs PayProcessContext 流程控制%}


<!-- tab 三種走向-->

第三方金流（PMW 系列）在結帳時，付款結果有三種走向：

| 結果 | 說明 | Pipeline 後段 |
|:---|:---|:---|
| **Redirect** | 需要導向金流頁面付款（如 QFPay 掃碼、Stripe 3DS） | ❌ 中斷，等 Callback |
| **Success（同步）** | PMW 當下回傳成功 | ✅ 繼續執行 |
| **Fail** | PMW 當下回傳失敗 | ❌ 中斷，執行取消流程 |


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-context/1-three-result.png)



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-context/3-result-context-table.png)



1. **`context.IsFinish`**：控制 Pipeline 是否繼續執行後段 Processor
2. **`context.ThirdPartyPaymentInfo`**：存放 Redirect URL，Pipeline 結束後由 Controller 讀取回傳前端



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-context/2-brake-system-and-share-context.png)


<!-- endtab -->


<!-- tab Redirect URL 如何傳到前端-->

URL **不透過 return value 傳遞**，而是寫入 `context` 物件，Pipeline 結束後 Controller 統一讀取。

```
ProcessPayment()
  → 寫入 context.ThirdPartyPaymentInfo.WebPaymentUrl（以及 AppPaymentUrl）

Pipeline 中斷（IsFinish = true）

Controller（讀取 context）
  → 包進 ApiResultEntity.Data 回傳給前端
```

## 完整傳遞路徑

```
CompleteForNewCartV2 (WebAPI)
  └─▶ PayProcessService.CreateTradesOrder(context)
        └─▶ ExecuteProcess("ThirdPartyProcess", isCheckFinish: true)
              └─▶ ThirdPartyPayApiProcessor.Process()
                    └─▶ TradesOrderPaymentService.ProcessPayment()
                          └─▶ PMW Pay API 回傳 Status = Redirect
                                └─▶ payChannelService.UpdateProcessContextForRedirect(context, paymentResult)
                                      ├─ context.IsFinish       = true
                                      ├─ context.Is3DSecure     = false（通常）
                                      └─ context.ThirdPartyPaymentInfo = {
                                             TransactionId,
                                             AppPaymentUrl,   ← App Deep Link
                                             WebPaymentUrl    ← Web 轉導 URL
                                         }

  ← Pipeline break，回到 Controller ──────────────────────

  └─▶ TradesOrderLiteController.CreateTradesOrder()
```

## Controller 依情況組 Response

### 3DS 驗證流程

```csharp
if (context.Is3DSecure == true)
{
    return Json(new {
        ReturnCode = "WaitingFor3d",
        Data = context.ThirdPartyPaymentInfo
    });
}
```

## 一般 Redirect 金流

```csharp
// 特殊處理：PayMe / BoCPay（Mobile 裝置）
// Mobile 改用 Deep Link：WebPaymentUrl = AppPaymentUrl

return Json(new {
    ReturnCode = "Success",
    Data = context.ThirdPartyPaymentInfo   // ← URL 在 Data 裡
});
```

## ReturnCode 語意

| ReturnCode | Data 內容 | 前端行為 |
|:---|:---|:---|
| `"Success"` + `Data.WebPaymentUrl` | Redirect URL | Redirect 使用者到金流頁 |
| `"WaitingFor3d"` + `Data` | ThirdPartyPaymentInfo | 進入 3DS 驗證流程 |
| `"Success"`（無 Data） | 空 | 直接跳訂單完成頁 |

<!-- endtab -->


<!-- tab IsFinish 機制-->

## ExecuteProcess 的中斷邏輯

```csharp
private void ExecuteProcess(PayProcessContextEntity context, string processName, bool isCheckFinish = false)
{
    var processorList = ResolveKeyed(processName);

    foreach (var processor in processorList.OrderBy(i => i.Metadata.Order))
    {
        processor.Value.Process(context);   // 執行 Processor

        if (isCheckFinish && context.IsFinish)
        {
            break;   // ← IsFinish = true 時立即中斷，後面的 Processor 不執行
        }
    }
}
```

> `CreateTradesOrder` 呼叫時傳入 `isCheckFinish: true`，
> 所以每個 Processor 執行完都會檢查 `context.IsFinish`。

## Redirect 情境的 Pipeline 執行示意

```
ThirdPartyProcess Pipeline（isCheckFinish = true）
  │
  ├─ [1]  XSSFilterProcessor                ✅ 執行
  ├─ [2]  ArrangeDataProcessor              ✅ 執行
  ├─ ...  （驗證、建單、扣點、扣券...）        ✅ 執行
  ├─ [30] HamiPointProcessor                ✅ 執行
  │
  ├─ [31] ThirdPartyPayApiProcessor         ✅ 執行
  │         └─▶ ProcessPayment()
  │               └─ Status == Redirect
  │                     └─ context.IsFinish = true  ← 設定
  │
  │  ← 執行完後檢查：IsFinish == true → break
  │
  ╳  [32] CreateGlobalInvoiceProcessor      ❌ 不執行
  ╳  [33] TransferOrderProcessor            ❌ 不執行
  ╳  [34] CreateRegularOrderProcessor       ❌ 不執行
  ╳  [35] AfterOrderProcessor               ❌ 不執行
```

## 三種情境對比

```
情境一：Redirect（WaitingToPay）  情境二：同步 Success       情境三：Fail
──────────────────────────────────────────────────────────────────────
ThirdPartyPayApiProcessor         ThirdPartyPayApiProcessor  ThirdPartyPayApiProcessor
  └─ IsFinish = true                └─ IsFinish 不變(false)    └─ 拋出例外
  └─ break                                                     └─ Pipeline 中斷

CreateGlobalInvoice  ❌            CreateGlobalInvoice  ✅    CreateGlobalInvoice  ❌
TransferOrder        ❌            TransferOrder        ✅    TransferOrder        ❌
CreateRegularOrder   ❌            CreateRegularOrder   ✅    CreateRegularOrder   ❌
AfterOrderProcessor  ❌            AfterOrderProcessor  ✅    AfterOrderProcessor  ❌

後段由誰補執行？                   當下 Pipeline 直接完成     CancelTradesOrderGroup
→ Callback 後的                                              （取消訂單）
  ThirdPartyFinishProcess
```

![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-context/4-how-finish-intterupt.png)


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-context/7-how-route-1-fronted-driect-to.png)



<!-- endtab -->


<!-- tab 後段 Processor 執行時機-->

Redirect 情境下被跳過的 4 個 Processor，職責各不相同：

| Processor | 職責 |
|:---|:---|
| `CreateGlobalInvoiceProcessor` | 建立發票（呼叫發票系統） |
| `TransferOrderProcessor` | 寫入 ERP 傳送佇列 |
| `CreateRegularOrderProcessor` | 建立定期購訂單（轉正） |
| `AfterOrderProcessor` | 後置處理（通知、Tag 等） |

## 各情境的執行時機

| 情境 | 後段 4 個 Processor | 執行時機 |
|:---|:---:|:---|
| **Redirect（WaitingToPay）** | ❌ 跳過 | Callback 成功後 → `ThirdPartyFinishProcess` |
| **同步 Success** | ✅ 執行 | 當下 Pipeline 繼續執行 |
| **Fail** | ❌ 跳過 | 不需要，直接走取消流程 |
| **Pending** | ❌ 跳過（IsFinish=true） | 同 Redirect，等後續排程或 Callback |

## Redirect 成功後的補執行路徑

```
第三方金流付款完成
  ↓
GET /V2/PayChannel/PayChannelReturn
  ↓
PayChannelController.PayChannelReturn()
  ↓
TradesOrderPaymentService.FinishPayment()
  ↓
QueryPayment() → PMW 回傳 0000 Success
  ↓
ExecuteProcess("ThirdPartyFinishProcess")
  ↓
CreateGlobalInvoiceProcessor   ✅
TransferOrderProcessor         ✅
CreateRegularOrderProcessor    ✅
AfterOrderProcessor            ✅
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-context/5-after-processors.png)



<!-- endtab -->


<!-- tab 完整結帳流程總覽-->

```
使用者送出結帳
  └─▶ CompleteForNewCartV2
        └─▶ Pipeline 執行（前段：驗證、建單、扣點、扣券...）
              └─▶ ThirdPartyPayApiProcessor
                    └─▶ ProcessPayment → 打 PMW Pay API
                          │
                          ├─【Redirect（WaitingToPay）】
                          │   context.ThirdPartyPaymentInfo.WebPaymentUrl = 金流頁 URL
                          │   context.IsFinish = true → Pipeline 中斷
                          │   Controller 回傳 URL 給前端
                          │   前端 redirect → 使用者在金流頁付款
                          │   ↓
                          │   金流方 Callback → FinishPayment → ThirdPartyFinishProcess
                          │   → 建發票、轉 ERP、轉正訂單、後置處理
                          │
                          ├─【Success（同步）】
                          │   Pipeline 繼續執行後段 4 個 Processor
                          │   當下完成
                          │   Controller 回傳成功
                          │
                          └─【Fail】
                              CancelTradesOrderGroup（取消訂單 + 退還點數/券/購物金）
                              拋例外 → Controller 回傳 ThirdPartyPayReserveFail
```


## 為什麼用 context 傳 URL 而不直接 return？

Pipeline 是 foreach 迴圈執行多個 Processor，
每個 Processor 的 `Process()` 回傳 void，
只能透過 **共享的 context 物件**傳遞狀態與資料。



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-context/6-why-not-direct-return-url.png)



## 為什麼要有 IsFinish？

Redirect 情境下，訂單已建立但付款未完成，
後段的 **發票建立、ERP 轉單、訂單轉正** 都需要付款確認後才能執行。
IsFinish 確保這些操作不會在付款完成前被提前觸發。




![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-context/8-how-redirect-to-finish.png)




<!-- endtab -->


{% endtabs %}
