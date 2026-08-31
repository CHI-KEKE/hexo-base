---
title: PMW 金流付款失敗完整流程
date: 2026-06-28 11:00:00
categories: Payment
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-failMessage/landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-failMessage/landing.png
tags:
    
toc:
toc_number:
comments :
---

{% tabs 金流-PMW付款失敗流程 %}


<!-- tab 兩條失敗路徑-->


PMW 系列金流的付款失敗，分為兩個完全不同的入口

| | 路徑一：建單時直接失敗 | 路徑二：PayChannelReturn 失敗 |
|:---|:---|:---|
| **觸發時機** | `CompleteForNewCartV2` → PMW 當下回 Fail | 第三方付款頁完成後 redirect 回 mweb |
| **訂單狀態** | 訂單從未成立，立即回滾 | 訂單已成立（進入 WaitingToPay） |
| **ErrorCode** | `ThirdPartyPayReserveFail` | 無 errorCode，顯示翻譯後文字 |
| **前端機制** | XState `useErrorAction.tsx` errorCode Map | `payChannelCallback/index.tsx` 直接渲染 |
| **用戶看到的文案** | 泛用「付款方式付款失敗」 | 各金流自訂文案（Stripe 最細緻） |


![d](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-failMessage/1-2-two-fail-path.png)



![d](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-failMessage/2-diff-of-two-paths.png)


## 適用金流

以下金流共用同一套 PMW 架構：

- **Stripe**（`CreditCardOnce_Stripe`）
- **Razer 信用卡系列**（`CreditCardOnce_Razer`、`CreditCardInstallment_Razer`、`PayNow_Razer`、`OnlineBanking_Razer`、`TNG_Razer`、`Boost_Razer`、`GrabPay_Razer`）
- **Cybersource**（`CreditCardOnce_Cybersource`）
- **QFPay**（`QFPay`）



![d](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-failMessage/5-pmw-paytypes.png)



<!-- endtab -->


<!-- tab 路徑一：建單時 PMW 失敗-->


```
前端結帳
  ↓
POST /TradesOrder/CompleteForNewCartV2
  ↓
TradesOrderLiteController.CreateTradesOrder(context)
  ↓
PayProcessService.CreateTradesOrder(context)   ← Processor Pipeline
  ↓
ThirdPartyPayApiProcessor.Process(context)
  ↓
TradesOrderPaymentService.ProcessPayment(context)
  ↓
PaymentProviderService.PaymentRequest(context)  ← 打 PMW HTTP API
  ↓
PMW 回傳 PaymentResultEntity { Status: Fail, Message: "原始錯誤" }
  ↓
CheckPaymentResultStatus()
  → throw TradesOrderProcessStatusException(ThirdPartyPayReserveFail, paymentResult.Message)
  ↓
Controller catch → 回傳 JSON
  ↓
前端 useErrorAction.tsx → 固定 i18n 文案 → 顯示彈窗
```

## ThirdPartyPayApiProcessor 統一入口

```csharp
case PayProfileTypeDefEnum.CreditCardOnce_Stripe:
case PayProfileTypeDefEnum.CreditCardOnce_Razer:
case PayProfileTypeDefEnum.CreditCardInstallment_Razer:
// ...其他 Razer 系列...
case PayProfileTypeDefEnum.CreditCardOnce_Cybersource:
case PayProfileTypeDefEnum.QFPay:
    PaymentResultEntity paymentResult = _tradesOrderPaymentService.ProcessPayment(context);
    CheckPaymentResultStatus(paymentResult);  // ← 失敗時 throw
    break;
```

失敗邏輯：

```csharp
// ThirdPartyPayApiProcessor.cs line 99
private void CheckPaymentResultStatus(PaymentResultEntity paymentResult)
{
    if (paymentResult.Status == PaymentStatusDefEnum.Fail)
    {
        _logger.Info($"Payment failed response body: {JsonConvert.SerializeObject(paymentResult)}");
        throw new TradesOrderProcessStatusException(
            TradesOrderResultEnum.ThirdPartyPayReserveFail,
            paymentResult.Message);  // ← PMW 原始訊息（後端傳出去，但前端不顯示）
    }
}
```

## Controller 回傳格式

```csharp
case TradesOrderResultEnum.ThirdPartyPayReserveFail:
    result.Message = tradesOrderProcessStatusException.Message;  // PMW 原始訊息
    result.ReturnCode += $"_{nameof(TradesOrderResultEnum.ThirdPartyPayReserveFail)}";
    break;
```

```json
{
  "ReturnCode": "API0004_ThirdPartyPayReserveFail",
  "Message": "（PMW 原始錯誤訊息）",
  "Data": "（新的 checkoutUniqueKey）"
}
```


![d](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-failMessage/3-path1-fail.png)



<!-- endtab -->


<!-- tab 路徑一：前端顯示機制-->

## 前端如何處理 ThirdPartyPayReserveFail

前端在 `useErrorAction.tsx` 中用 errorCode 字串作為 key，對應到固定顯示規則：

```typescript
// useErrorAction.tsx line 480
TradesOrderProcessStatusException_ThirdPartyPayReserveFail: () => ({
    message: I18n.t('client.shopping.shopping_cart.error.payment_reserve_fail', {
        payment: I18n.t('client.shopping.shopping_cart.payment_method'),
        //              ↑ 泛用「付款方式」，不區分 Stripe / QFPay / Cybersource
    }),
    // ⚠️ 沒有 actionHint → 彈窗無「返回購物車」按鈕
}),
```

最終顯示：**「付款方式付款失敗，請重新選擇付款方式」**

## 關鍵特性

| 特性 | 說明 |
|:---|:---|
| PMW 原始 Message | **被前端完全忽略**，不顯示給用戶 |
| 支付方式名稱 | 泛用「付款方式」，不區分各金流 |
| actionHint | **無**，彈窗只有關閉按鈕，無「返回購物車」 |
| checkoutUniqueKey | 從 `Data` 欄位更新，讓用戶可以重新結帳 |

<!-- endtab -->



<!-- tab 路徑二：PayChannelReturn 失敗-->

## 何時走 PayChannelReturn 路徑

第三方金流（如 Stripe 3DS、QFPay 掃碼）需要把用戶導向金流頁面付款，
付款頁完成後 redirect 回 mweb 的 `PayChannelReturn` action。

> 這代表訂單**已經成立**，只是付款結果還不知道。

## 呼叫鏈

```
第三方金流付款完成
  ↓
GET /V2/PayChannel/PayChannelReturn?payChannel=...&tgCode=...&k=...
  ↓
PayChannelController.PayChannelReturn()
  ↓
TradesOrderPaymentService.FinishPayment(isFromWorker: false)
  ↓
PaymentMiddlewareService.QueryPayment()  ← 打 PMW Query API 查詢結果
  ↓
payChannelService.GetErrorMessage(responseEntity)  ← 各金流自訂邏輯
  ↓
依 ReturnCode 分支設 ViewBag.Error
  ↓
return View("PayChannelCallback")
  ↓
前端 payChannelCallback/index.tsx 顯示 Dialog
```

## Controller 如何設 ViewBag.Error

```csharp
switch (paymentResult.ReturnCode)
{
    case "0000":  // Success
        return new RedirectResult($"/V2/Pay/Finish/?k={paymentResult.TradeOrderGroupUniqueKey}&shopId={shopId}");

    case "3000":  // Failed
    case "2001":  // Expired
        ViewBag.Error = paymentResult.Message;  // ← GetErrorMessage() 翻譯後文字
        ViewBag.ErrorRedirectUrl = $"/V2/ShoppingCart/Index?shopId={shopId}&err=PaymentFail";
        break;

    case "2003":  // WaitingToPay
    default:
        ViewBag.Error = paymentResult.Message;  // ← 通常是 OrderInProcessing
        ViewBag.ErrorRedirectUrl = $"/V2/TradesOrder/TradesOrderList?shopId={shopId}&err=PaymentWaitingToPay";
        break;
}
// Exception 路徑：固定 Translation.PaymentInConfirmation，不走 GetErrorMessage
```

## View 到前端的傳遞

```html
<!-- PayChannelCallback.cshtml -->
<script>
  window.nineyi.ServerData = {
    Message: '@ViewBag.Error',
    RedirectUrl: '@Html.Raw(ViewBag.ErrorRedirectUrl)'
  }
</script>
```

```tsx
// payChannelCallback/mobile/index.tsx
<Dialog isOpen={isOpenDialog} isShowClose={false}
        confirmText={I18n.t('client.common.confirm')}
        onConfirmDialog={() => { window.location.href = serverData.RedirectUrl; }}>
    {serverData.Message}   {/* ← 直接渲染後端傳來的翻譯文字，不再套 i18n Map */}
</Dialog>
```

> ✅ 前端**直接顯示**後端 `Message`，不像路徑一會忽略後端內容改用固定 i18n key



![d](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-failMessage/4-path-2-fail.png)


<!-- endtab -->


<!-- tab 路徑二：各金流 GetErrorMessage 差異-->

各金流顯示的文案不同，Stripe 最細緻，最多三種


## Stripe（`StripePayChannelService`）

**讀取 PMW ExtendInfo 中的 `last_payment_error_code` 決定文案：**

```csharp
switch (errorCode)
{
    case "incorrect_cvc":
    case "invalid_cvc":
    case "incorrect_number":
    case "invalid_expiry_year":
        return Translation.Backend.Service.StripePayChannel.ErrorCardIncorrect;
        // → 「卡片資訊有誤，請確認後重新輸入」

    case "payment_intent_authentication_failure":
        return Translation.Backend.Service.StripePayChannel.ErrorAuthenticationFailure;
        // → 「3DS 驗證失敗」

    default:
        return Translation.Backend.Service.StripePayChannel.ErrorCardDeclined;
        // → 「卡片被拒，請嘗試其他付款方式」
}
```


## Razer CC（`RazerPayChannelService`）

```csharp
case "3000": return Translation.Backend.V2.PayChannel.ErrorFail;  // 「付款失敗」
default:     return string.Empty;   // 非 Failed 一律空字串
```





![d](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-failMessage/6-razer-errorcodes-diff.png)




## QFPay（`QFPayPayChannelService`）

```csharp
case "3000": return Translation.Backend.V2.PayChannel.ErrorFail;         // 「付款失敗」
case "2001": return Translation.Backend.V2.PayChannel.ErrorMessage;      // 「付款逾時」
default:     return Translation.Backend.V2.PayChannel.OrderInProcessing; // 「訂單處理中」
```


## Cybersource（`CybersourcePayChannelService`）

```csharp
if (ReturnCode == "0000") return string.Empty;
return Translation.Backend.V2.PayChannel.ErrorFail;  // 非 Success 全部同一個「付款失敗」
```


## 各金流文案對照表

| 金流 | `3000` Failed | `2001` Expired | default | 特殊邏輯 |
|:---|:---|:---|:---|:---|
| **Stripe** | 卡片被拒 | N/A | 卡片被拒 | 讀 `last_payment_error_code`，最多 3 種文案 |
| **Razer CC** | 付款失敗 | `""` 空字串 | `""` 空字串 | 非 Failed 一律空字串 |
| **QFPay** | 付款失敗 | 付款逾時 | 訂單處理中 | 標準三段式 |
| **Cybersource** | 付款失敗 | 付款失敗 | 付款失敗 | 非 Success 全部相同 |

## ReturnCode × 前端行為

| ReturnCode | Dialog 顯示 | 按確認後跳往 |
|:---|:---|:---|
| `0000` Success | 不顯示 Dialog，直接跳轉 | `/V2/Pay/Finish/`（訂購完成頁） |
| `3000` Failed | GetErrorMessage 自訂文字 | `/V2/ShoppingCart/Index?err=PaymentFail`（購物車） |
| `2001` Expired | GetErrorMessage 自訂文字 | `/V2/ShoppingCart/Index?err=PaymentFail`（購物車） |
| `2003` WaitingToPay | 固定「交易進行中」 | `/V2/TradesOrder/TradesOrderList?err=PaymentWaitingToPay` |
| Exception | 固定「付款確認中」 | `/V2/TradesOrder/TradesOrderList?err=OrderProcessing` |

> Dialog 行為：頁面載入後自動開啟，沒有關閉按鈕，只能按「確認」後跳轉



![d](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Arch/ecom/Payment-failMessage/7-query-frontend-redriectTo-pages.png)






<!-- endtab -->


{% endtabs %}
