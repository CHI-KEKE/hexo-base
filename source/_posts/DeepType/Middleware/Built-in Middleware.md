---
title: Built-in Middleware
date: 2025-09-26 17:07:34
categories: Middleware
top_img: https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
tags:
    - Middleware
toc:
toc_number:
comments :
---

{% tabs Built-in Middleware%}

<!-- tab UseCors-->


CORS 的本質是**瀏覽器端的安全機制（Same-Origin Policy 的例外白名單）**。伺服器只透過回應標頭「宣告」哪些 Origin、方法、標頭被允許；最後放不放行是瀏覽器決定，他的角色是偵測跨域請求，依 Policy 加上允許的回應標頭。並且透過預檢且允許的方式，直接回 204/200 給瀏覽器，省去進一步處理。當不允許時，伺服器通常不加標頭、也不主動丟 403；接著的行為取決於你的管線/其他中介


![cors](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Buildin/cors.png)


## 🎀 Simple Request（簡單請求）

**瀏覽器送出真正的請求**

例如 GET https://api.contoso.com/orders，同時自動帶上 Origin: https://app.example.com 標頭。

**CORS Middleware（UseCors）判斷**

如果政策允許這個 Origin，就在回應上加 Access-Control-Allow-Origin: https://app.example.com（必要時還會加 Vary: Origin 等）。如果不允許，它通常不會擋下請求，也不會主動回 403，只是不加任何 CORS 回應標頭。

**瀏覽器決定要不要給前端程式讀資料**

若回應有正確的 Access-Control-Allow-Origin，瀏覽器把資料交給你的 JS。若沒有，瀏覽器就把資料「擋在外面」，Console 會報：No 'Access-Control-Allow-Origin' header...（就算伺服器回的是 200）。
重點是，CORS 是瀏覽器在執行與裁決；伺服器只是「宣告政策」的角色。


## 🎀 Preflight（預檢）

當請求「不算簡單」時（例如 PUT/DELETE、自訂 Header、Content-Type 不是 application/x-www-form-urlencoded|multipart/form-data|text/plain、或要帶憑證），瀏覽器會先發一個 OPTIONS 預檢請求

**瀏覽器先送 OPTIONS**

```bash
OPTIONS /orders
Origin: https://app.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Authorization, Content-Type
```

**UseCors 依政策評估**

允許：回 204 No Content（或 200），並加上 Access-Control-Allow-Origin,Access-Control-Allow-Methods, Access-Control-Allow-Headers（必要時 Access-Control-Allow-Credentials）。
這表示「好，你可以發真正的 PUT/帶這些標頭」。

不允許：通常就不加這些允許標頭（可能讓下一個 Middleware 處理），結果預檢失敗。

**瀏覽器據此決定是否送出「真正的請求」**

預檢通過 → 才會送出實際的 PUT /orders。
預檢未通過 → 真正的請求根本不會送，Console 直接報 CORS 錯。
（你可能在網路面板看到 OPTIONS 200/204 或其他碼，但只要少了那些 Access-Control-Allow-* 標頭，瀏覽器就判為失敗。）


![cors2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Buildin/cors_mechanics.png)



<!-- endtab -->

<!-- tab UseAuthentication()-->


這個 Middleware 會去呼叫 已經在 Program.cs / Startup.cs 註冊的驗證方案 (Authentication Schemes)。驗證方案會去「讀請求裡的東西」，像是

- Header (`Authorization: Bearer xxx`) → JWT Token
- Cookie (`.AspNetCore.Cookies`) → Cookie 驗證
- 第三方登入（Google、Facebook OAuth）

驗證成功後，就會建立一個 `ClaimsPrincipal`（代表使用者），存到 H`ttpContext.User`。
`UseAuthentication` 本身不需要你手動寫邏輯，它只是一個「管線上的鉤子」，實際邏輯來自你註冊的 `Handler`（例如 `AddJwtBearer`、`AddCookie`）。

👉 這一步只負責「知道你是誰」，不會擋請求

![authen](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Buildin/authen_autho.png)


<!-- endtab -->

<!-- tab UseAuthorization()-->


UseAuthorization() 這個 Middleware 的任務就是在你真正執行 Controller 或 Endpoint 之前，先檢查「這個使用者有沒有資格做這件事」。
它不會自己去問你要怎麼判斷，而是看你在程式碼裡有沒有加 [Authorize] 這些規則，例如

```csharp
[Authorize(Roles = "Admin")] // 必須是 Admin 角色
[Authorize(Policy = "CanDeleteOrder")] // 必須符合某個自訂的 Policy
```

判斷結果有兩種常見情況

- 沒登入（沒有帶 Token，或 Token 無效） → 回 401 Unauthorized
- 有登入但不符合規則（例如不是 Admin） → 回 403 Forbidden

假設一個請求帶了 Header
```bash
Authorization: Bearer <JWT Token>
```

- `UseAuthentication()` 會先解析這個 Token，確認它是否合法，並幫你產生一個 `ClaimsPrincipal`（使用者身份資訊）。
- `UseAuthorization()` 接著去檢查 [Authorize] 的需求，看看 `HttpContext.User` 裡面有沒有符合的 `Claims`


![autho](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Buildin/autho_policies.png)


## 🎀 簽章正確（Signature Valid）

JWT Token 的格式是
```bash
header.payload.signature
```
- header：說明用什麼演算法（例如 HMAC-SHA256、RSA）
- payload：使用者的資訊（claims）
- signature：用「伺服器的私鑰」加密前兩段算出來的驗證碼

API（Resource Server）收到 Token 之後，會用「公開的金鑰（public key）」驗證 signature 是否正確。如果有人亂改 payload（例如把 "role": "User" 改成 "role": "Admin"），簽章驗證一定會失敗。


## 🎀 Issuer 符合（正確的發證單位）

iss（Issuer）是 JWT 裡的一個 Claim，代表「誰發的這張 Token」
```JSON
{
  "iss": "https://identity.example.com", 
  "sub": "12345",
  "aud": "myApi"
}
```
如果設定是
```csharp
options.Authority = "https://identity.example.com";
```
系統會去檢查 iss 是否符合這個值。這保證了 Token 真的是由你的認證中心發的，不是別的地方隨便生成的。


## 🎀 Audience 符合（Token 是給誰用的）

`aud（Audience）`代表「這張 Token 的受眾（Audience）」 → 也就是「誰可以接受這張 Token」
```JSON
{
  "aud": "myApi"
}
```

如果在 API 設定
```csharp
options.Audience = "myApi";
```
系統就會檢查 Token 裡的 aud 是否包含 myApi。這保證了 Token 是發給這個 API 用的，而不是給別的服務（例如 Web 前端或其他 API）的

只要 Token 簽章正確、`Issuer` 和 `Audience` 符合，系統會把 JWT Payload 裡的 `Claims` 轉成 `HttpContext.User`

- 如果 Token 裡有 "name": "jerry"，就會變成 HttpContext.User.Identity.Name == "jerry"
- 如果 Token 裡有 "role": "Admin"，就可以用 User.IsInRole("Admin")
- 如果 Token 驗證失敗（過期、簽章錯誤、audience 不對），就不會建立合法的使用者，HttpContext.User 會是一個「匿名使用者」

設定範例
```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
    {
        options.Authority = "https://identity.example.com"; // 驗證中心
        options.Audience = "myApi"; // API 名稱
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
    policy.RequireRole("Admin")); // 自訂授權政策
});

var app = builder.Build();

app.UseRouting();

app.UseCors("MyPolicy");
app.UseAuthentication(); // 先驗證身份
app.UseAuthorization();  // 再判斷權限

app.MapGet("/orders", () => "訂單清單"); // 開放的 API
app.MapGet("/admin", [Authorize(Policy = "AdminOnly")] () => "管理員專區");

app.Run();
```


![jwt](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Buildin/jwt.png)



<!-- endtab -->

<!-- tab UseStaticFiles-->


在 ASP.NET Core 中，`app.UseStaticFiles()` 是一個 Middleware，專門用來處理 靜態檔案 (Static Files) 的請求。

靜態檔案 指的是 HTML、CSS、JavaScript、圖片、字型、PDF… 這些不需要後端程式邏輯處理的檔案。當瀏覽器發送請求（例如 GET /css/site.css），這個 Middleware 會去檔案系統找對應的檔案（預設是 wwwroot 資料夾）。如果找到，就直接把檔案傳回瀏覽器，不會再進到後面的 Controller 或 API。它就是讓 ASP.NET Core 也能像一個「檔案伺服器」一樣，回應靜態內容。


## 🎀 自訂資料夾

檔案實體位置：MyFiles/report.pdf
瀏覽器存取路徑：http://localhost:5000/files/report.pdf
```csharp
app.UseStaticFiles(new StaticFileOptions
{
    FileProvider = new PhysicalFileProvider(
        Path.Combine(Directory.GetCurrentDirectory(), "MyFiles")),
    RequestPath = "/files"
});
```

當伺服器回應一個檔案給瀏覽器時，不只會傳檔案本身，還會告訴瀏覽器「這是什麼檔案」，Middleware 會根據副檔名設定正確的 Content-Type（例如 .css → text/css、.jpg → image/jpeg）
如果副檔名不在已知清單裡，預設會拒絕回應（避免洩漏機敏檔案）。預設只開放 wwwroot，避免洩漏整個伺服器檔案系統。如果需要其他資料夾，必須明確設定 StaticFileOptions


![staticfiles](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Buildin/staticfiles.png)



<!-- endtab -->

<!-- tab UseRateLimiter-->


![rate](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Buildin/rate_limiter.png)


它的任務是在單位時間內，限制某個「Key」可以送出多少請求。

這個「Key」怎麼決定？

預設是「每個 HTTP 請求」都有一個唯一的 Key，可以根據 IP、User Token、甚至 HttpContext.User 來決定。但 Middleware 本身不會自動解析所有資訊，而是透過你給的 Partition Key Factory 來決定。


## 🎀 Partition Key Factory
ASP.NET Core 的 RateLimiter 提供一個 API 讓你指定「怎麼分組（Partition）」
```csharp
options.AddFixedWindowLimiter("fixed", opt =>
{
    opt.Window = TimeSpan.FromSeconds(10);
    opt.PermitLimit = 5;
    opt.QueueLimit = 0;

    opt.AutoReplenishment = true;
});
```

上面這種寫法，如果沒有特別指定，所有請求都共用同一個桶子（全局共用限制）。
但如果你要「依照 IP」或「依照 User 身分」分組


## 🎀 依照 IP 限制
```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("ip", opt =>
        RateLimitPartition.GetFixedWindowLimiter(
            key: context => context.Connection.RemoteIpAddress?.ToString() ?? "unknown",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 5,
                Window = TimeSpan.FromSeconds(10),
                QueueLimit = 0
            }));
});
```


## 🎀 依照使用者（已登入 Token）限制
```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("user", opt =>
        RateLimitPartition.GetFixedWindowLimiter(
            key: context => context.User.Identity?.Name ?? "anonymous",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 10,
                Window = TimeSpan.FromMinutes(1),
                QueueLimit = 0
            }));
});
```

這樣如果使用者有登入（JWT Token → 轉成 ClaimsPrincipal），就依照 HttpContext.User.Identity.Name 當作分組 Key


## 🎀 依照 API Key（自訂 Header）限制
```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("apikey", opt =>
        RateLimitPartition.GetFixedWindowLimiter(
            key: context => context.Request.Headers["X-API-KEY"].ToString() ?? "no-key",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromHours(1),
                QueueLimit = 0
            }));
});
```

UseRateLimiter 並不會自己去解析 Token（不像 UseAuthentication 會自動解析 JWT）。它只是 讀取 HttpContext 的資訊（例如 IP、User、Header），然後套用你定義的分組方式。
所以如果你已經先用 UseAuthentication() 解析過 Token，HttpContext.User 就會有值，RateLimiter 就能依照使用者身分來做限制。


```csharp
app.UseRouting(); // 鎖定 endpoint
app.UseAuthentication(); // 先解析 Token → 產生 User
app.UseRateLimiter();    // 再依照 User / IP 限制
app.UseAuthorization();
```


<!-- endtab -->

<!-- tab UseResponseCaching-->

![cache](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Buildin/response_cache.png)


`UseResponseCaching` 是 ASP.NET Core 內建的 快取 Middleware，它的工作就像一個「小型的快取代理伺服器」，會把 回應（Response）暫存起來，讓相同的請求下次可以直接拿快取，而不用重新跑 Controller → Service → DB。但它的限制很多，所以 只適合簡單、靜態的內容（例如：常見問題頁面、最新公告 API）。


## 🎀 快取範圍太小

它只存在「單一 Web Server 的記憶體」，如果伺服器重啟，快取就消失，如果有多台 Web Server（K8S、雲端），每台都有自己的快取，沒辦法共享

## 🎀 快取條件限制


只能快取 GET / HEAD，預設 Response 不能超過 100MB，帶 Cookie 的 Response 不能快取（避免使用者資料被混淆）

## 🎀 安全性風險

如果開發者不小心在敏感 API（例如 /profile）加了 Cache-Control: public，就可能造成A 使用者快取的 Response → 被誤用到 B 使用者 → 資料外洩
如果有 Proxy / CDN（像 Cloudflare, Nginx），它們也會根據 Header 快取 → 風險更大


<!-- endtab -->

<!-- tab UseHttpsRedirection-->


![gate](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Buildin/outergate.png)


`UseHttpsRedirection` 的工作很單純，如果有人透過 http:// 連到你的網站，就自動把他導向 https://。這樣能避免使用者在「不安全的 HTTP」下傳送資料，因為 HTTP 傳輸內容是明文，任何人（駭客、中間人攻擊）都能攔截到。

當 Middleware 發現請求是 http://example.com，就回應一個「重新導向 (Redirect)」給瀏覽器：
```bash
HTTP/1.1 307 Temporary Redirect
Location: https://example.com
```
瀏覽器收到之後，會再自動發一個新的請求到 https://example.com。最終，所有人都會用 HTTPS 來跟伺服器溝通。

ASP.NET Core 會依照情況決定回應哪一種 Redirect

### 🎀 307 Temporary Redirect

暫時的，意思是「下次還是要試 http://」，適合測試或開發環境

### 🎀 308 Permanent Redirect

永久的，意思是「以後都要用 https://」，適合正式環境


<!-- endtab -->

<!-- tab UseHsts-->


`UseHsts()` 的任務就是告訴瀏覽器：「以後拜託只用 HTTPS 找我！」
`UseHsts()` 讓瀏覽器記住「這個網站只能用 HTTPS」，之後就算用戶輸入 http://example.com，瀏覽器也會自動替換成 https://example.com，根本不會發送不安全的 HTTP 請求。

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

if (!app.Environment.IsDevelopment()) // ⚠️ 建議只在正式環境開啟
{
    app.UseHsts();
}

app.UseHttpsRedirection();

app.MapGet("/", () => "Hello HSTS!");

app.Run();
```

執行後，伺服器會在回應裡加上 Header
```bash
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

<br>

## 🎀 max-age=31536000

意思是「瀏覽器要記住這個規則多久」，31536000 秒 = 365 天。期間內，瀏覽器會「強制 HTTPS」，不管使用者怎麼輸入。

<br>

## 🎀 includeSubDomains

不只 example.com，連 api.example.com、shop.example.com 都會被強制成 HTTPS。避免有人利用子網域開一個 HTTP 的「釣魚網站」。

<br>

## 🎀 preload（選用）

你還可以把網站加入 HSTS Preload List（由 Chrome/Firefox/Edge 維護），這樣即使使用者第一次造訪，也會自動走 HTTPS。但要非常小心，因為一旦加入，就幾乎「永久強制」，除非等瀏覽器更新清單才會改回來。

<br>

## 🎀 HSTS 和 Proxy

如果前面有 Nginx、Cloudflare，可能是 Proxy 在處理 SSL，而你的 ASP.NET Core 只看到 HTTP。這時候需要確認 Proxy 會把 HTTPS 的資訊帶進 X-Forwarded-Proto header，否則 UseHsts() 可能判斷錯誤。


<!-- endtab -->


<!-- tab 結語-->


![seq](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Buildin/sequence.png)

![sum](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Buildin/summa.png)


<!-- endtab -->


{% endtabs %}