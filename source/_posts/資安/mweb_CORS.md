


C:\91APP\NineYi.WebStore.MobileWebMall\WebStore\WebAPI\Extensions\HttpModules\CorsModule.cs


Application_EndRequest

## Access-Control-Allow-Origin

「每次 Request 結束時」，自動決定

- 回應要不要加上 Access-Control-Allow-Origin
- 加上哪一個允許跨域的來源（Allow-Origin）

取得來源網址 (Origin / Referrer) → 判斷是否在允許清單 → 產生 Allow-Origin → 寫入 Response Header

## Step 0：取得 HttpContext

```csharp
var httpApplication = (HttpApplication)sender;
var httpContext = httpApplication.Context;
```

## ✨ Step 1：錯誤狀態不處理 CORS


```csharp
if (httpContext.Response.StatusCode >= 400 || httpContext.Response.HeadersWritten)
{
    return;
}
```

如果回應狀態碼 ≥ 400（錯誤）或 Header 已經送出（無法再加入 Header）

👉 就不處理 CORS

因為錯誤頁通常不需要 CORS，且若 Header 已送出，再加 Header 會噴錯誤：

Server cannot append header after HTTP headers have been sent

## ✨ Step 2：決定來源來源：Origin > Referrer

```csharp
var origin = httpContext.Request.Headers.Get("origin");
if (origin.IsNullOrWhiteSpace() == false && 
    Uri.IsWellFormedUriString(origin, UriKind.Absolute))
{
    sourceUri = new Uri(origin);
}
else if (httpContext.Request.UrlReferrer != null)
{
    sourceUri = httpContext.Request.UrlReferrer;
}
```

決定「請求從哪個網域來」。

有跨域 → 有 Origin, Origin 是標準 CORS 的來源
無跨域 → 只能用 Referrer, Referrer 是普通頁面跳轉、Ajax 或表單的來源


## ✨ Step 3：判斷來源是否合法 → 產生 allowOrigin

```csharp
if (sourceUri != null && 
    sourceUri.DnsSafeHost.IsNullOrWhiteSpace() == false && 
    this.GetIsRequestSourceAllowed(sourceUri))
{
    var host = sourceUri.DnsSafeHost;

    if (sourceUri.Port != 80 && sourceUri.Port != 443)
    {
        host = $"{host}:{sourceUri.Port}";
    }

    allowOrigin = $"{sourceUri.Scheme}://{host}";
}
else
{
    allowOrigin = AppSetting.GetAppSetting("AccessControl.Allow.Domain");
}
```

✔ 「sourceUri 有值」代表 → 這次請求來自某個 domain
✔ GetIsRequestSourceAllowed(sourceUri) = 來源是否在允許清單
✔ 若來源合法 → 依據來源組出 allowOrigin，例如：

https://shop.91app.com
https://shop.91app.com:8001

✔ 若來源不合法 或沒有來源 → 用預設設定的 Allow Domain


## ✨ Step 4：HTTPS 安全性檢查

```csharp
if (httpContext.Request.Url.Scheme == Uri.UriSchemeHttps && 
    (sourceUri == null || sourceUri.Scheme == Uri.UriSchemeHttps))
{
    allowOrigin = allowOrigin.Replace("http:", "https:");
}
```


若 API 是 HTTPS → 必須要求來源也是 HTTPS
避免「從 http 跳到 https」造成：混合內容警告（Mixed Content）

## ✨ Step 5：真正寫入 CORS Header

```csharp
httpContext.Response.AddHeader("Access-Control-Allow-Origin", allowOrigin);
httpContext.Response.AddHeader("Access-Control-Allow-Credentials", "true");
```

## 正常的前端網站呼叫 API（最常見）

來源：https://shop.91app.com/product → 呼叫 API：https://api.91app.com/v1/product

🔍 過程會發生什麼？

前端 JS 發 AJAX / Fetch → 會自動附上 Origin Header：

Origin: https://shop.91app.com

🔎 CORS 模組怎麼做？

取得 Origin

判斷 shop.91app.com 有在允許清單

設定 Allow-Origin：

Access-Control-Allow-Origin: https://shop.91app.com

✔ 結果

前端能成功拿到資料。

❗若沒有這個模組？

你的 API「不會自動回應正確的 Allow-Origin」，瀏覽器會拒絕讀取。


## 🧪 案例 2：惡意網站想盜取資料（攻擊者）
攻擊者做了一個釣魚網頁：
https://evil.com/fake-shop

當使用者登入你的網站後，他開啟攻擊者網站，攻擊網站偷偷呼叫：

fetch("https://api.91app.com/v1/member-info", { credentials: "include" })

🔍 攻擊者會送出 Origin：
Origin: https://evil.com

🔎 CORS 模組怎麼做？

來源不是官方 / 不是允許清單

回傳預設的 Allow-Domain（通常不是 evil.com）

結果：

Access-Control-Allow-Origin: https://xxx (非 evil.com)


瀏覽器偵測「Origin 與 Allow-Origin 不一致」
→ 直接 阻擋讀取回應

✔ 結果

攻擊失敗，盜取不到使用者資料。

❗沒有這段邏輯會怎樣？

如果錯誤設定成：

Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true


攻擊者就能跨站盜用你的 API → 超級嚴重資料外洩。


## 🧪 案例 3：官網有多種店家子網域（官方店清單）

店家 A 的官網：

https://shop123.91app.com


店家 B：

https://shop88.91app.com


這些網域是動態建立的（每個商家都有自己的子網域），無法手動一條一條設定。

🔍 Request Origin：
Origin: https://shop123.91app.com

🔎 CORS 模組怎麼做？

分析 domain → 判斷是官方 Domain

呼叫 HasOfficialAllowOriginDomain() 查詢 DB 或 cache

確定此店家存在 → 允許

將此商家網域塞進 Allow-Origin

✔ 結果

所有 90% 的商家子網域都能正常呼叫 API。

❗若少了這段邏輯？

你必須「手動維護上萬筆子網域」的 CORS Allow 清單 → 不可能做到。


## 🧪 案例 4：本機、測試環境、客戶端自訂埠號（Port）

如前端跑在：

http://localhost:8080


或

https://staging.myshop.com:5001


如果你不處理 port：

🔍 Request Origin：
Origin: http://localhost:8080

🔎 CORS 模組會做：
if (port != 80 && port != 443)
{
    dnsSafeHost = $"{dnsSafeHost}:{port}";
}


所以允許來源會變：

Access-Control-Allow-Origin: http://localhost:8080

✔ 結果

前端測試環境能正常調 API。

❗若不判斷 port？

只會允許：

http://localhost


瀏覽器會直接阻擋，開發者無法調 API。

## 🧪 案例 5：API 回應 500 / 404 錯誤時，避免 CORS 崩掉

系統有錯誤 → 例如程式寫錯 → 回傳 500

以前（.NET Framework）
錯誤頁可能會提前送出 Headers → 如果你再寫 Header → 會崩掉：

Server cannot append header after HTTP headers have been sent

🔎 程式如何避免？
if (httpContext.Response.StatusCode >= 400 || httpContext.Response.HeadersWritten)
{
    return;
}

✔ 結果

避免因錯誤導致的 CORS 邏輯異常或網站直接崩潰。

## 🧪 案例 6：http → https 混合內容安全問題（安全限制）

假設 API 是 HTTPS
但前端是 HTTP：

Origin: http://shop.91app.com


為避免混合內容：

程式會強制：
allowOrigin = allowOrigin.Replace("http:", "https:");

✔ 結果

瀏覽器不會跳 mixed content error。

❗如果不做？

前端可能跳：

Mixed Content：The request was blocked because it is insecure.


## 🧪 案例 7：來自 APP（無 Origin，但有 Referrer）

有些 App WebView 不會附 Origin，但會有 Referrer：

Referrer: https://app.91app.com/home

程式邏輯：
if (no Origin)
    use Referrer


這確保 App WebView 的 API 呼叫也能通過 CORS。

❗沒有這段會怎樣？

App 前端 AJAX 會全部被瀏覽器擋掉 → App 無法運作。

## 🧪 案例 8：惡意使用假 referrer（Referrer Spoof）

攻擊者想偽造 Referrer：

Referrer: https://shop.91app.com.evil.com


你的程式檢查：

dnsSafeHost.EndsWith("." + i)


意思是：

.ok
.evil.com  ❌  → 不符合結尾

✔ 結果

成功擋下假網域。

❗ 若判斷不完整，例如用 Contains → 就會被繞過

| 案例               | 會發生什麼                | 這段程式怎麼保護             |
| ---------------- | -------------------- | -------------------- |
| 1. 正常前端呼叫        | 需要 Allow-Origin      | 正確回應 CORS            |
| 2. 攻擊者盜取資料       | 惡意 Origin            | 擋掉非法來源               |
| 3. 多商家子網域        | 每個商家不同 domain        | 透過 DB + Cache 判斷合法官網 |
| 4. 本機/測試有 port   | localhost:8080       | 正確加入 port，避免錯誤       |
| 5. API 錯誤頁       | Header 已送出           | 避免 CORS 崩掉           |
| 6. 混合內容          | http → https         | 自動強制 https           |
| 7. APP WebView   | 無 Origin 但有 Referrer | 保證可用                 |
| 8. 假 Referrer 攻擊 | 惡意偽造 Host            | 嚴格後綴比對擋攻擊            |


## 要「跨兩個不同網域登入」怎麼辦？

→ 使用 SameSite=None; Secure

這是業界標準 SSO 用法，例如：

Google Login

Facebook Login

LINE Login

Auth0

Azure AD

流程：

登入站（login.siteA.com）設定 Cookie：

Set-Cookie: AuthSession=xxxx;
    SameSite=None;
    Secure;
    HttpOnly;


跳回 shop.com（不同 Site）

Cookie 仍然會被帶上

因為 SameSite=None 代表：

「請允許跨站使用 Cookie 」

這是唯一合法、標準、被瀏覽器接受的做法。




## Google / Facebook / LINE / Auth0 為什麼全部都用 SameSite=None

我們來看「跨站惡意請求」到底怎麼運作。

假設有惡意網站：

https://evil.com


它寫：

fetch("https://shop.com/api/user", { credentials: "include" })

如果你是 SameSite=None，Cookie 會被帶出去

✔ 沒錯，你的 Session 會附在 request 上。

但是接下來發生三件事：

🛡 （1）CORS 防止攻擊者讀到 response

你的 API 只允許：

Access-Control-Allow-Origin: https://你的官網
Access-Control-Allow-Credentials: true


這時瀏覽器會說：

「看你是從 evil.com 來的，不給你看 response。」

攻擊者 JS 會看到：

CORS error: blocked by CORS policy

❌ 惡意網站“無法”讀你的回應資料
❌ 無法竊取會員資料
❌ 無法讀到餘額 / 訂單 / token
🛡 （2）CSRF Token 防止攻擊者偽造操作

即使 Cookie 帶出去了，也：

沒有 CSRF Token

你的 API 不會接受

前端會送：

X-CSRF-Token: abc123


evil.com 的 JS 不可能知道這個 Token（SameSite None 也讀不到）。

所以攻擊者即使：

發了 POST

帶了 Session Cookie

仍會得到：

403 Forbidden (Missing CSRF Token)

🛡 （3）HttpOnly 防止 Cookie 被 JavaScript 讀取

Set-Cookie: SessionId=xxx; HttpOnly;

→ 攻擊者的 JS 完全讀不到 Cookie
→ 無法竊取你的 Session
→ 無法放到自己的 request 裡

🎯 整合起來看：

SameSite=None 並不是弱點，只是讓跨網域流程可行。
真正阻擋攻擊的是：
CORS + CSRF Token + HttpOnly Cookie。

所以標準防禦組合是：

Set-Cookie:
    SessionId=xxx;
    HttpOnly;
    Secure;
    SameSite=None;

Access-Control-Allow-Origin: https://你的官網
Access-Control-Allow-Credentials: true

（API 寫操作必須檢查 CSRF Token）


這就是全世界大型登入系統（Google / Facebook / LINE / Azure）正在用的配置。

