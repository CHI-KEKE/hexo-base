---
title: Stripe
date: 2024-07-02 23:11:05
categories: Others
top_img: https://i.imgur.com/QYGzWvo.png
cover : https://i.imgur.com/QYGzWvo.png
tags:
    - C#
toc:
toc_number:
comments :
---

# The Payment Intents API

Payment Intents API 是 Stripe 提供的一個用於處理複雜支付流程的工具。
思考：為什麼需要一個專門的 API 來處理支付流程？傳統的支付處理方式有什麼局限性？

主要功能：

處理變化的支付狀態
追蹤整個支付生命週期
在需要時觸發額外的認證步驟

思考：這些功能如何提高支付流程的可靠性和安全性？

優勢：

自動處理認證
避免重複收費
解決冪等性問題
支持強客戶認證（SCA）

思考：這些優勢如何簡化開發流程？它們如何影響用戶體驗？

應用場景：

每個 PaymentIntent 通常對應一個購物車或客戶會話

思考：這種對應關係如何幫助管理複雜的電子商務場景？


````cURL

curl https://api.stripe.com/v1/payment_intents \
  -u "{YOUR_STRIPE_TEST_SECRET_KEY}:" \
  -d amount=1099 \
  -d currency=usd

````


## Passing the client secret to the client side

每個 PaymentIntent 都有一個唯一的客戶端密鑰（client secret）

## Off-session / On-session


Off-session 的定義：

"Off-session" 指的是在用戶不直接參與的情況下進行的支付。
這種支付發生在用戶不在您的網站或應用程序上時。


與 On-session 的對比：

On-session：用戶直接參與的支付，例如在網站上完成購買。
Off-session：用戶不直接參與的支付，通常是預先授權的。


Off-session 支付的應用場景：

訂閱服務的定期扣款
分期付款的後續扣款
預訂服務的延後收費（如酒店）
根據使用情況進行的自動扣款（如雲服務）


為什麼需要區分：

安全考量：Off-session 支付可能需要額外的安全措施。
法規遵循：某些地區（如歐盟）對 off-session 支付有特殊要求。
授權率：銀行對 off-session 支付的處理方式可能不同。


## Dynamic statement descriptor

主要用途：

允許您為每筆交易自定義出現在客戶信用卡對賬單上的描述。
幫助客戶更容易識別和回憶特定的交易。


默認情況：

如果不使用動態描述符，所有交易都會顯示您 Stripe 帳戶的默認描述符。


## Storing information in metadata

主要用途：

關聯自定義信息：將您系統中的數據與 Stripe 交易連接起來。
便於追蹤和報告：在 Stripe 儀表板和報告中查看自定義信息。
增強欺詐預防：為 Radar for Fraud Teams 提供額外信息。


實際應用場景：

訂單追蹤：將您的訂單 ID 與 Stripe 支付關聯。
客戶信息：存儲額外的客戶數據（非敏感信息）。
產品詳情：添加購買的產品信息。
內部參考：添加內部追蹤或參考號碼。