---
title: QueryString
date: 2025-10-05 22:38:11
categories: 蒼き盾
top_img: https://github.com/CHI-KEKE/pics/blob/main/Security/wolf.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Security/wolf.png?raw=true
tags:
    - 蒼き盾
toc:
toc_number:
comments :
---

{% btn 'https://blog.darkthread.net/blog/dont-pass-data-in-querystring/',資安小常識 - 為什麼不建議在網址用 QueryString (?a=...) 傳資料？,far fa-hand-point-right %}

{% tabs QueryString%}

<!-- tab UberEats-->

下班時間將至，lazy 在 UberEats 下完訂單後，在「備註欄」寫上

「不要香菜」
「門口放就好」
「到樓下打電話」

他突然想到，食物到家的時間很可能來不及接應，於是又打上

「我現在不在家，密碼是 5740，管理室說可以直接進，如果我沒接電話就直接放我房門口就好」

後來發現外送平台有紀錄、客服看得到、內部系統會存、有時截圖還會被轉來轉去...

![ubereats](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/ubereats.jpg)

<br>

Query String 的本質不是「傳資料」，而是把「你要哪個資源、用什麼條件」這件事講清楚，讓這個請求可以被辨識、被重播、被理解，他正是為了讓「請求能被識別與重現」

當伺服器接到一個網址時，必須能知道「使用者要查什麼、做什麼、要哪一份資料」它是對伺服器的**指令**，不是傳輸內容的容器


![ss](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Querystring/1__q_is_resource.png)


在 HTTP 的哲學裡，它屬於 **資源定位的一部分（part of URI）**

<br>

URL（包含 Query String）代表「資源的位址」，若帶上其他提交的資訊相當於告訴外送員送到哪裡的同時，順便分享你的秘密!



![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Querystring/1__message.png)


<!-- endtab -->

<!-- tab 凡走過必留下痕跡-->

因為 Query String 是 URL 的一部分，而 URL 是網路世界的地址。只要一個系統要「找得到你、記得你、能重播請求」，它就必須完整記錄整條 URL。所以留下痕跡是本身的 feature

<br>

## 瀏覽器的歷程記錄

讓使用者能回到曾經拜訪的同一頁，瀏覽器把完整 URL 存起來，是為了能「重現你的行為」。因為網址不同就代表不同內容

![QueryStringHistoy01](https://github.com/CHI-KEKE/pics/blob/main/Security/querystring01.png?raw=true)

<br>

## 書籤（Bookmark）

讓頁面能被完整重訪，瀏覽器不知道哪些參數重要、哪些不重要，它只能誠實地保存整條 URL

<br>

## 開發者工具（Network tab）

讓開發者能觀察完整的 HTTP 請求，對開發者工具來說，URL 是診斷的依據，所以必須顯示最原始的請求內容，包含 Query String。



![aaa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Querystring/1__history.png)



## Referer Header（引用頁面）

目的是讓伺服器知道「流量從哪來」，HTTP 設計 Referer，是為了幫網站分析使用者行為、做流量追蹤。所以瀏覽器在發新請求時，會把上一個 URL（含 Query）放進 Header

如果在上面帶上敏感資訊，就像拜訪別人家，順手送上了你的**個資**當伴手禮 🎁


![sss](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Querystring/2__referrer.png)


## Web Server Access Log

目的是紀錄所有請求，方便除錯、追蹤與統計。伺服器 Log 是維運的「黑盒子」
它要能回答：

- 哪個 URL 被訪問最多？
- 哪裡 404？
- 哪個 API 出錯？

因此它必須把完整請求（包含 Query）記下來，確保「請求可追蹤」

例如 IIS、Nginx、Apache 預設都會記錄完整 URL

```bash
IIS：%SystemDrive%\inetpub\logs\LogFiles\W3SVC...
Nginx：/var/log/nginx/access.log
```

<br>

## Reverse Proxy 與 CDN Log

目的是快取與負載分流。CDN 需要用 URL（包含 Query）來判斷「這是不是相同的請求」。

例如 `Cloudflare`、`Azure Front Door`、`NGINX Proxy` 都會記錄完整 URL

<br>

## 分析工具（Analytics）

目的是追蹤使用者行為與流量來源。`Google Analytics`、`Application Insights` 等工具要知道

- 使用者在哪個頁面進來
- 哪些 Query 組合最多人使用
- 本質是 Query String 是「行為分析的線索」。

只是這些工具不知道哪些是「查詢條件」，哪些是「個資」



![aa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Querystring/2__server_everywhere.png)



<font color=#D3D3D3 style="font-size: 20px;">一旦在 Query String 加上姓名、Email、電話等隱私，甚至身分證號或信用卡資料，等同一次把個資灑得到處都是!
 </font>

<!-- endtab -->

<!-- tab 反射型 XSS（Reflected XSS）-->

XSS 的本質就是語意轉換錯誤（string → code）

`瀏覽器 Request`
   ↓
`URL Query String（惡意字串）`
   ↓
`Server（沒有消毒）`
   ↓
`HTML Response（把字串直接輸出）`
   ↓
`瀏覽器解析成 JavaScript 並執行`

沒有被存進 DB、沒有長期存在於系統，只存在於「這一次請求與回應之間」，就像聲音打到牆壁再彈回來一樣


![wall](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Querystring/3__reflected_xss.png)


<br>

當網站直接把 URL 參數（`Query String`）原樣插進網頁內容（例如直接 `innerHTML`、或在 server side 用 `Html.Raw` 輸出），就會產生 `反射型 XSS（Reflected XSS）` 或內容被竄改的風險。攻擊者只要誘導使用者點擊特製 URL，就能在被害者的瀏覽器上執行惡意 JavaScript、顯示詐騙訊息或盜用 session

<br>

把不受信任的輸入當作 HTML 或 JavaScript 輸出，瀏覽器會解析並執行它。就是把使用者的字串交給瀏覽器當成「程式碼」而非「單純文字」

innerHTML 會把字串解析成 DOM，若字串內含 
```html
<script> 、 onclick 、 <img src=x onerror=...>
```

就會被執行或觸發

HTML 如下
```html
<!-- 不安全：直接把 query string 放到 innerHTML -->
<div id="msg"></div>
<script>
  const params = new URLSearchParams(location.search);
  const raw = params.get('msg') || '';
  // 危險做法：把外來字串交給 innerHTML，會被當成 HTML/JS 解析
  document.getElementById('msg').innerHTML = raw;
</script>
```


![aa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Querystring/3__html.png)


<!-- endtab -->

<!-- tab localStorage 偷東西-->

一旦攻擊者能在頁面執行任意 JavaScript，就能讀取前端儲存，並把資料送走

localStorage / sessionStorage / IndexedDB 中的任何資料，然後用 fetch / Image / navigator.sendBeacon 等方式把資料傳到攻擊者服務器

`JS 在同一 origin 可以自由存取這些儲存，攻擊者插入的 JS 也是同源腳本，於是可讀可寫!`



![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Querystring/4__steal_info.png)


<!-- endtab -->

<!-- tab  Cookie-->

## HttpOnly

HttpOnly 的核心價值不是「防止攻擊發生」，而是把「最值錢的資料」從 JavaScript 能碰到的世界中隔離出去，就算前端被打穿，關鍵資產也不會一起陪葬

1. 若 Server 在設定 cookie 時加上 HttpOnly flag，這是一個「伺服器層級的決定」，前端無法自己加或移除，瀏覽器接收到這個 cookie 後，標記為 HttpOnly
2. 瀏覽器知道這顆 cookie 只能用在 HTTP 層，不是 JS 的玩具
3. 每次對符合 domain / path 的請求，瀏覽器自動帶上 cookie
4. 不需要前端寫任何程式碼，瀏覽器會放進 Cookie header
5. JavaScript 無法用 document.cookie 讀或寫這顆 cookie
6. 不論是正常程式碼，還是被注入的 XSS 腳本，都碰不到
7. 就算 XSS 發生，session id 仍然留在瀏覽器與伺服器之間
8. 攻擊者可以執行 JS，但偷不到最重要的憑證


![cookie](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Querystring/5__http_only.png)


## SameSite

SameSite 是伺服器端在「Set-Cookie」回應裡設定的屬性，瀏覽器根據這個屬性在後續請求時自行判斷「要不要送出 cookie」。前端（JavaScript）不能直接設定或修改 SameSite 屬性

當使用者在網站 A 的頁面上點了一個連到網站 B 的連結、或在網站 A 的 iframe 裡載了網站 B，或由其他網站發起對 B 的請求——這些情境就是跨站。瀏覽器會在某些情況送出或不送 cookie，取決於 cookie 的 SameSite 屬性。

- `SameSite=Strict` 最嚴格，只有在「同站（same-site）導航」時才會送出 cookie
- `SameSite=Lax`（瀏覽器預設，適中）在大多數「安全的頂層導覽」會帶 cookie（例如 `GET` 導航點擊），但在跨站 `POST` 或子資源請求`（iframe、AJAX）`不帶 cookie;
- `SameSite=None; Secure`，表示允許跨站情境下也送 cookie（例如第三方嵌入需要 cookie），但必須同時設 Secure（HTTPS），只有在你确实需要跨站 cookie（例如第三方登入、跨域 SSO）時使用


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Querystring/5__samsite.png)


<!-- endtab -->

<!-- tab 消毒或編碼-->

## 完全不允許 HTML（最安全）

```html
<div id="msg"></div>
<script>
  const params = new URLSearchParams(location.search);
  const raw = params.get('msg') || '';
  // 安全：textContent 會把字串視為純文字，自動 escape
  document.getElementById('msg').textContent = raw;
</script>
```
- 從 URL query string 取得使用者輸入，raw 是不可信資料可能包含 `<script>、onerror、HTML tag`
- 使用 textContent 輸出，瀏覽器會把內容當純文字 `<b>、<script> `，都只會顯示字樣，不會執行

攻擊 payload 無法變成 DOM，無執行機會

<br>
<br>

## 允許部份 HTML（需要 sanitize）

```html
<!-- 使用受信賴的 sanitize lib (DOMPurify) -->
<script src="https://unpkg.com/dompurify@latest/dist/purify.min.js"></script>
<script>
  const raw = new URLSearchParams(location.search).get('msg') || '';
  const clean = DOMPurify.sanitize(raw, {ALLOWED_TAGS: ['b','i','strong','em']});
  document.getElementById('content').innerHTML = clean; // innerHTML 接受的是已清理過的內容
</script>
```

使用者可以放格式（粗體、斜體），代表不能用 `textContent`，但 `innerHTML = raw` 絕對不行



![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Querystring/5__encode.png)


<!-- endtab -->

<!-- tab Razor-->

預設會 encode，直接用 @Model.Field，不要用 Html.Raw
```csharp
using System.Text.Encodings.Web;
var safe = HtmlEncoder.Default.Encode(untrustedInput);
```

<!-- endtab -->

<!-- tab Token-->

## Query String 傳送 Token

把敏感資訊存在伺服器（Memory / Redis / DB），產生一個隨機 Token（例如 GUID 或更安全的隨機字串）對應到該資料，將 Token 放在 Query String，前端新頁面再用這個 Token 向伺服器請求資料並顯示。Token 不含敏感內容，且應有短時效、綁定與可撤銷性，降低被竊用的風險

Token 只是指標，不是資料本身：即使 Token 被記錄，攻擊的損害也受限，因為伺服器可以

- 設短期過期（TTL）
- 設單次使用或綁定使用者（session / user id / IP / UA）
- 隨時撤銷或清掉對應資料


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Querystring/6__how_token.png)


## Token 設計

- Token 必須 不可猜測（GUID 可用，但最好使用加密強隨機字串）。
- Token 要有 TTL（過期時間）。
- 資料儲存位置：Memory Cache（單機）、Redis（分散式）、DB（持久）。選哪個看需求
  - 臨時、短期、單機測試 → IMemoryCache（ASP.NET Core）
  - 分散式或多實例 → Redis（推薦）
  - 要長期保存或審計 → 資料庫（搭配 TTL 欄位）
- 綁定（Binding）：若資料與 user 有關，將 token 與 userId / sessionId 綁定。
- 支援 單次使用（one-time use） 或限制讀取次數。
- 日誌遮罩：不要把資料本體寫入 logs；只記 token、userId、操作事件。
- 刪除/過期回收：定期清理過期 token（Redis 自帶 TTL）。
- 速率限制與 WAF：避免 token 暴力猜測或濫用。
- CSP / HttpOnly / SameSite：配合其他安全控制


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Querystring/5__token.png)


## 以金流為例

最好的作法是，讓 `Payment Provider` 做 `Client-side Tokenization（Hosted Fields / Elements / Iframe）`，卡號/CVV 永不經過後端；前端 `Hosted iframe` 直接送到支付商，回傳不可逆 `token（paymentMethodId）`，後端只是帶著這個 token 去扣款，這樣可以降低 PCI 範圍、讓支付廠商負責安全細節


**PCI 範圍 PCI Scope**

這包含

- 主卡號`（PAN, Primary Account Number）`
- 卡片有效期`（Expiry Date）`
- 持卡人姓名`（Cardholder Name）`
- CVV / CVC 安全碼`（Card Verification Value）`

只要系統的任何一環 「看過這些資料」，那個環節就要遵守 PCI 的安全要求，而這個範圍就叫做 `PCI Scope`，讓越少的系統「碰到卡資料」越好!


![df](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Querystring/7_PCI.png)


<!-- endtab -->


<!-- tab summary-->


![aa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Querystring/query.png)


<!-- endtab -->


{% endtabs %}