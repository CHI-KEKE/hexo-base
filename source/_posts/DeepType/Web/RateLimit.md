---
title: RateLimit
date: 2025-09-23 09:21:34
categories: Web
top_img: 
cover : 
tags:
    - 
toc:
toc_number:
comments :
---



- 超過時回 429 Too Many Requests


## 限流演算法

Fixed Window：最好懂，像「每分鐘 60 次」
適合大多數一般 API

Sliding Window：比固定視窗平滑，邊界比較不會突然爆量
適合你很在意公平性

Token Bucket：比較像流量池，適合有突發流量的情境
很多公開 API 會偏愛這個

Concurrency：保護重 CPU / 重 I/O API
例如報表匯出、影像處理、AI 推論


## 把 RateLimiter 掛進 Program.cs

用 AddRateLimiter 註冊政策

用 OnRejected 統一處理超限時的回應

回 429

有 Retry-After 就寫到 header

記結構化 log，後面好做告警

```csharp
using System.Globalization;
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    options.OnRejected = async (context, token) =>
    {
        var http = context.HttpContext;

        // 官方文件示範：可從 lease metadata 取 RetryAfter
        if (context.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        {
            http.Response.Headers.RetryAfter =
                ((int)retryAfter.TotalSeconds).ToString(CultureInfo.InvariantCulture);
        }

        // 你自己的 header，前端/呼叫方會很好判讀
        http.Response.Headers["X-RateLimit-Policy"] = "api-per-user";
        http.Response.Headers["X-RateLimit-Blocked"] = "true";

        var logger = http.RequestServices
            .GetRequiredService<ILoggerFactory>()
            .CreateLogger("RateLimit");

        logger.LogWarning(
            "Rate limit exceeded. TraceId={TraceId}, Path={Path}, User={User}, IP={IP}",
            http.TraceIdentifier,
            http.Request.Path,
            http.User?.Identity?.Name ?? "anonymous",
            http.Connection.RemoteIpAddress?.ToString());

        http.Response.ContentType = "application/json; charset=utf-8";
        await http.Response.WriteAsJsonAsync(new
        {
            error = "rate_limit_exceeded",
            message = "請求過於頻繁，請稍後再試。",
            traceId = http.TraceIdentifier
        }, cancellationToken: token);
    };

    options.AddSlidingWindowLimiter("api-per-user", limiterOptions =>
    {
        limiterOptions.PermitLimit = 30; // 30 次
        limiterOptions.Window = TimeSpan.FromMinutes(1); // 1 分鐘
        limiterOptions.SegmentsPerWindow = 6;
        limiterOptions.QueueLimit = 0;
        limiterOptions.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
    });
});

var app = builder.Build();

app.UseRouting();

// 若分區要看登入者資訊，要放在驗證之後
app.UseAuthentication();
app.UseAuthorization();

app.UseRateLimiter();

app.MapControllers().RequireRateLimiting("api-per-user");

app.Run();
```

官方文件明確提到 OnRejected 可用來統一處理超限請求，官方範例也示範了從 MetadataName.RetryAfter 取得 retry 時間、設定 Retry-After header、回 429、以及寫 log。官方也指出預設 rejected status 是 503，若要表達「你打太快」，通常應改成 429 Too Many Requests。


## Header 要回哪些，前端和外部串接的人才真的看得懂

最低標配

Retry-After
告訴對方幾秒後再來。官方指出這個 metadata 可用於 Token Bucket、Fixed Window、Sliding Window；Concurrency limiter 因為沒法估計何時釋放，因此不適合期待一定拿得到。

X-RateLimit-Policy
讓你知道是哪條政策擋的

X-RateLimit-Blocked: true
很直白，好查問題

進階常用

X-RateLimit-Limit

X-RateLimit-Remaining

X-RateLimit-Reset


## Middleware 順序不能亂放

若是 endpoint-specific rate limit，UseRateLimiter 要放在 UseRouting 之後

若分區規則需要看認證資訊，應放在 UseAuthentication 之後
官方範例也直接註明因為 limiter 用到了 auth info，所以 UseRateLimiter() 放在 UseAuthentication() 後面。


```csharp
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();
app.MapControllers().RequireRateLimiting("api-per-user");
```


## Response Header

Header 的價值，在於你當場把遊戲規則講清楚，對方才不會一直瞎撞，把你的系統打得更慘。

1. Client 打 API 過來

例如前端、手機 App、合作廠商系統、別的後端服務來呼叫你的 API。

2. 你的 API 判斷這次請求有沒有超過限制

例如：

每分鐘只能打 60 次

同一個使用者只能同時有 2 個匯出報表請求

同一個 IP 每分鐘只能嘗試登入 5 次

3. 你把結果放在 Response Header

例如：

Retry-After: 30

X-RateLimit-Limit: 60

X-RateLimit-Remaining: 0

X-RateLimit-Policy: login-ip-limit

這些不是給你自己內部查帳用的，主要是給「呼叫方」看。

4. 呼叫方根據 Header 調整行為

例如：

前端看到 Retry-After: 30，就 30 秒後再重試

SDK 看到剩餘次數快沒了，就先暫停

第三方串接商看到 policy 名稱，就知道是哪條限制生效

這樣才不會變成：

被擋了還狂打

使用者只看到錯誤卻不知道原因

串接方抱怨 API 不穩，其實只是自己打太快





## 問題
 
我們現在有串接facebook 直播站台，我們是使用 aspnet core webapi，我們想紀錄或加上 ratelimit 的狀況(rps = 100)，串接方式就是我們的站台開出webhook api, facebook 事件(例如留言) 打過來我就可以將資訊透過 webhook 接收並使用 aws apigateway websocket 發送給前端，現在有需求說Header 紀錄 API rate limit Alert, 但我不太懂甚麼意意思，我該做什麼




希望 Webhook API 能知道自己目前有沒有接近流量上限，並且在快要超過時，把這個狀態記錄下來，最好也能放在 Response Header 或 Log 裡面，方便監控、告警、追查。


你的 API 要限制自己最多 100 RPS。

你的 API 每次回應時，要告訴呼叫方目前額度狀態。

當流量接近 100 RPS，例如到 80% 或 90%，要記錄 alert，讓監控系統告警。
這三件事常常會一起做。ASP.NET Core 官方範例也有展示「超限時回應 429」和 OnRejected 的做法。


A. 入口 webhook 要有限流

每秒最多 100 個 request，超過就 429。ASP.NET Core 官方支援。

B. 回應時要帶 rate limit 狀態

即使是 200，也能回：

X-RateLimit-Limit: 100
X-RateLimit-Remaining: 23
X-RateLimit-Alert: WARNING
C. 系統內部要能告警

當 1 秒內請求數超過 80 或 90，就寫 warning log，送 CloudWatch Alarm、Slack、PagerDuty 都可以。


你可以直接問這 4 句，幾乎就能把需求講清楚

- 「這個 rate limit 是要限制我們 webhook 入口，還是只要紀錄狀態？」
- 「Header 是要回給 Facebook 看，還是給我們內部監控/測試使用？」
- 「Alert 的門檻是多少？80%、90%、還是超過 100 才算？」
- 「除了 Header，是否也需要寫入 log 與監控告警？」