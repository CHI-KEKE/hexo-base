---
title: HttpRequestMessage
date: 2025-08-30 08:52:01
categories: 穿越雲層的信使
top_img: https://i.imgur.com/hZWviQz.png
cover : https://i.imgur.com/hZWviQz.png
tags:
    - Http
toc:
toc_number:
comments :
---

在資訊的天空裡，每一次請求都是一隻信鴿。它揹著我們的訊息，穿過纜線與路由器，飛往未知的彼端。

但如果我們只急著把字條塞進它的翅膀，信鴿或許能起飛，卻未必能安全抵達。它需要一個完整的「航行指令」：要飛向哪裡、攜帶什麼、如何證明身份、以何種姿態展現誠意。

HttpRequestMessage 就是這份航行指令。它讓我們能在雲端的長風中，為信使裝備地圖、護照與行李，確保訊息能準確無誤地抵達遠方。

![HttpRequestMessage](https://i.imgur.com/hZWviQz.png)


## 📨 HttpRequestMessage

HttpRequestMessage 的存在就是為了讓 HTTP 請求能「物件化」並且「延後執行」，這是一種「Command Pattern」的應用：

<br/>

<font color=#D3D3D3 style="font-size: 15px;">&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;組好請求指令 → 再交給執行者 HttpClient → 再送出
 </font>

<br/>


這樣做的好處是：

- 請求可以先組好但晚點再送
- 可以重複使用或修改
- 可以做攔截、紀錄、包裝（例如透過 DelegatingHandler）

HttpRequestMessage 是一個「請求的包裹」，你可以把它裝滿所有你要送給伺服器的資訊。它代表的是「一個完整、可自訂的 HTTP 請求」，包括：

- 要去哪裡（網址 URL）
- 用什麼方法（GET、POST、PUT、DELETE...）
- 帶什麼內容（Body）
- 附上什麼證明身份的資料（Headers、Authorization）
- 跟誰說明你是誰、你想幹嘛

我們把這些「請求的基本元素」全部包裝起來變成一個物件。

<br>
<br>

## 📨 不使用 HttpRequestMessage

<br>

### ❌ 無法同時設定「Authorization」和「自訂 Header」

你可以用 HttpClient.DefaultRequestHeaders.Authorization 設定授權沒錯，
但如果你還要加像 Stripe-Account 這種特殊 Header，就開始麻煩了。

而且這些 Header 是屬於 request 的，不是 client 的，你不能讓 HttpClient.DefaultRequestHeaders 一直保留著會造成之後所有請求都帶這些 Header，可能造成資安問題或 API 錯誤。

<br>

### ❌ 無法設定非標準的 HTTP 方法（像 PATCH、PUT、DELETE）

如果你用的是 client.PostAsync()、client.GetAsync()，你只能發送固定的方法，
無法像 HttpRequestMessage(HttpMethod.Delete, url) 這樣自由選擇。

<br>

### ❌ 無法一個 request 加上多種控制（Headers + Body + Method）

例如下面這段 你是沒辦法用 PostAsync() 實作的：

```CSHARP
var request = new HttpRequestMessage(HttpMethod.Post, fullUrl);
request.Headers.Authorization = ...;
request.Headers.Add("Stripe-Account", acct);
request.Content = new StringContent(...);
```

<br>

你用 PostAsync() 的話，最多只能設定 url 和 content，像這樣：

```CSHARP
var content = new StringContent(...);
var response = await client.PostAsync(url, content);
```

那你會發現：

- ❌ 無法設定 Stripe-Account Header
- ❌ 沒有 request 物件可控
- ❌ 沒辦法「一次把完整的 request 組好」

<br>

因此
```CSHARP
var request = new HttpRequestMessage(HttpMethod.Post, fullUrl);
// 設定 Authorization
var byteArray = Encoding.ASCII.GetBytes($"{_apiKey}:");
client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Basic", Convert.ToBase64String(byteArray));
// 加入 Stripe-Account header
request.Headers.Add("Stripe-Account", acct);
// 設定 body
request.Content = content = new StringContent($"type=card&card[number]={cardNo}&card[exp_month]=12&card[exp_year]=2025&card[cvc]=123", Encoding.UTF8, "application/x-www-form-urlencoded");

// 發送
var response = client.Send(request);
if (response.IsSuccessStatusCode)
{
    var responseBody = response.Content.ReadAsStringAsync().GetAwaiter().GetResult();
    Console.WriteLine($"{responseBody}資料從Stripe拿到了了!!");

}
else
{
    Console.WriteLine($"请求失败: {response.StatusCode}");
    throw new ApplicationException($"請求Stripe API 發生錯誤,暫停運作, 停在 pm : {paymentMethod}, Error : {response.Content.ToString()}");
}
```


<br>
<br>

## 📨 結語

當我們只是用 PostAsync() 匆匆寄出，訊息或許能穿過雲層，卻像隨手放飛的紙飛機，方向與命運難以掌握。而當我們選擇 HttpRequestMessage，就等於給信使一份清晰的飛行計畫：它知道該攜帶哪些憑證、該經過哪些航道，甚至能在途中調整姿態。

在這片無形的天空裡，準確比速度更重要，掌控比僥倖更可靠。所以，下一次當你要送出訊息，不妨先停下來，為信使繫好雲端的行囊，讓它能帶著完整的心意，穿越雲層，抵達世界的另一端。