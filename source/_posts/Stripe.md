---
title: Stripe (串接 ApplyPay 所需要知道的事)
date: 2024-08-26 10:50:00
categories: Others
top_img: https://i.imgur.com/QYGzWvo.png
cover : https://i.imgur.com/QYGzWvo.png
tags:
    - ThirdParty
toc:
toc_number:
comments :
---

最近在串接 Stripe 的 ApplePay，花了一段時間理解 Stripe 的付款流程，這邊整理一下 ApplePay Server side 這邊要知道的一些事情備忘

# 名詞解釋

Platform Account : 平台帳戶，底下可能會有多個子帳戶
Connected Account : 底下的子帳戶，就是你管理的商戶們

Direct Charge : Platform Account 與 Connected Account 是以 Standard 模式關聯在一起，客戶付款時，錢會直接流向 Connected Account，通常若商店選擇這種方式表示他們有能力處理與 Stripe 談到較好費率的能力，不需要電商平台處理

Destination Charge : 


# Direct charge 概念：

1. Direct chargen 所指的方面是在 Stripe Connect 平台中，最終用戶（客戶）直接與商家（關聯帳戶）進行交易的模式。

2. 客戶感知：
在這種模式下，客戶可能不知道有一個中間平台參與其中。他們認為自己是直接與商家交易。

1. 收費歸屬：
交易記錄會出現在商家的 Stripe 帳戶中，而不是平台的主帳戶中。這增加了交易的透明度。

1. 商家餘額：
每次成功的交易都會直接增加商家（關聯帳戶）的 Stripe 餘額。這意味著商家可以更直接地管理其資金。

1. 平台收益：
雖然主要資金流向商家，但平台可以通過收取每筆交易的應用程式費用來獲得收入。這些費用會增加平台的 Stripe 餘額。

適用場景：
這種模式適合那些希望為商家提供更多自主權，同時仍從每筆交易中獲得一定收益的平台。

![Image](https://i.imgur.com/C027xgJ.png)

# Connect platforms using the Payment Methods API

這一篇主要說明如何在 Direct Charge 模式下做付款的流程

首先，必須在 Connected Account 啟用你打算使用的支付方法，例如 ApplePay、一般信用卡
付款流程需先提供一些資訊後藉由 PaymentMethod API 建立 PaymentMethod Id，接著到真正的執行付款行為時，透過 把 PaymentMethod Id 帶在 Payment Intent API 的 Header 上完成結帳

但，要注意的是，每次產生的 Payment Method Id 是一次性的，也就是每次顧客結帳都要再帶一次付款資訊

## clone PaymentMethod

但是，以信用卡來說，我們想做到 "記住常用信用卡" 的這個概念，這部分的實作就是透過 clone PaymentMethod，我們想像，自己是 Uber Eats，顧客在你的平台註冊並保存了信用卡信息。但實際收款的是各個餐廳，不是你的平台，並且信用卡資訊也不會流到餐廳那裏去，clone Payment method 的做法就是在我們的平台帳戶先建立好可重複使用的 PaymentMethodId，接著可以 clone 到三商巧福的子帳號進行付款，下次還可以使用同一張卡在去麥當勞的子帳號去結帳

所以步驟是這樣:

1. 顧客在平台添加信用卡：

    使用 Stripe.js 在前端安全收集信用卡信息
    調用 Stripe API 建立 PaymentMethod

```JAVASCRIPT

const paymentMethod = await stripe.paymentMethods.create({
  type: 'card',
  card: cardElement,
});

```

2. 將 PaymentMethod Attach 到 Customer:

```JAVASCRIPT

const customer = await stripe.customers.create({
  email: 'customer@example.com',
  payment_method: paymentMethod.id,
});

```

3. 顧客從餐廳下單時：
   
    獲取餐廳的 Stripe Connect 帳戶 ID（假設為 'acct_restaurant123'）
    Clone PaymentMethod 到餐廳的 Stripe 帳戶

```JAVASCRIPT

const clonedPaymentMethod = await stripe.paymentMethods.create({
  customer: customer.id,
  payment_method: paymentMethod.id,
}, {
  stripeAccount: 'acct_restaurant123',
});

```

4. 餐廳使用 Cloned 的 PaymentMethod Id 支付：

```JAVASCRIPT

const paymentIntent = await stripe.paymentIntents.create({
  amount: 1000, // 金額，比如 10.00 美元
  currency: 'usd',
  payment_method: clonedPaymentMethod.id,
  confirm: true,
}, {
  stripeAccount: 'acct_restaurant123',
});

```


#　iOS, Andriod, React Native 意義是甚麼

因為對 APP 不熟，順便了解一下文件上列出了這些類型的意義是什麼

iOS：指原生 iOS 應用開發（使用 Swift 或 Objective-C）。
Android：指原生 Android 應用開發（使用 Java 或 Kotlin）。
React Native：指使用 React Native 框架開發的跨平台移動應用。

因此，React Native 專門設計用於開發可以在 iOS 和 Android 上運作的 APP



# 怎麼串接 Stripe ApplePay

1. Register for an Apple Merchant ID

需註冊為 Apple Developer 帳號
https://idmsa.apple.com/IDMSWebAuth/signin?appIdKey=891bd3417a7776362562d2197f89480a8547b108fd934911bcbea0110d07f757&path=%2Faccount%2Fresources%2Fidentifiers%2Fadd%2Fmerchant&rv=1

然後取得一個 merchantId

2. Create a new Apple Pay certificate

這一步是要產生憑證在 Stripe 帳戶上，如此以來，可以建立比較安全的付款資料傳輸

位置 : https://dashboard.stripe.com/settings/payments/apple_pay

下載 CSP 後加入第一步的資訊，產生憑證後掛上

一個 Apple MerchantId 對應一組 Certificate

3. Create a Payment Method

若我們今天要在後端 Server 做付款，可以在 ApplePay Client Side 先產 Payment Method Id 往後丟


4. Create the Payment Intent

```CSHARP

// Set your secret key. Remember to switch to your live secret key in production.
// See your keys here: https://dashboard.stripe.com/apikeys
StripeConfiguration.ApiKey = "{YOUR_STRIPE_TEST_SECRET_KEY}";

var options = new PaymentIntentCreateOptions
{
    Amount = 1099,
    Currency = "hkd",
};

var service = new PaymentIntentService();
var paymentIntent = service.Create(options);
// Pass the client secret to the clientS

```

產出 Client Secret 後要再丟回給 APP 前端處理付款確認


最後要注意 : 留意證書到期提醒（有效期 25 個月）



# Stripe hosted onboarding for Custom accounts

Stripe 提供了一個網頁表單，用於收集自定義連接帳戶的身份驗證信息。這個表單會根據帳戶的功能、國家和業務類型動態調整所需信息。它是為自定義帳戶收集身份驗證信息的推薦解決方案。

流程：

1. 通過 Stripe API 創建一個新的連接帳戶。
2. 為這個新帳戶生成一個 Account Link。
3. 將您的商家用戶重定向到這個 Account Link URL。
4. 商家在 Stripe 頁面上填寫必要信息。
5. 填寫完成後，商家被重定向回您的平台。
6. 您的平台檢查帳戶狀態，確認是否需要額外操作。
7. 如果一切正常，該商家的連接帳戶就可以開始使用了。

![Image](https://i.imgur.com/Hbxxp1F.png)


# Using OAuth with Standard accounts that your platform controls

OAuth 的用途：

允許 Stripe 用戶連接到你的平台。

OAuth 連接流程：

a. 用戶從你的網站點擊連接鏈接。
b. 用戶在 Stripe 網站上提供連接資訊。
c. 用戶被重定向回你的網站，帶有授權碼。
d. 你的網站向 Stripe 請求完成連接並獲取用戶的帳戶 ID。

創建 OAuth 鏈接流程：

在平台設置中啟用 OAuth。
獲取 client_id 和設置 redirect_uri。
創建一個包含必要參數的 URL。