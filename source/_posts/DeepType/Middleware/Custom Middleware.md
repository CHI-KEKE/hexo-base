---
title: Custom Middleware
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

{% tabs Custom%}

<!-- tab Guards-->


想像你走進一片森林，每個訪問都是一個 HTTP 請求 🌳。森林裡的每個樹屋（Service / Middleware）都會依序經過，有的樹屋會幫你掛上名牌（CorrelationId），有的會在入口拉下柵欄（Maintenance Mode），還有的會檢查你有沒有森林通行證（API Key）。這些小樹屋站點就是 Middleware，它們靜靜地守護著這片森林，讓森林安全、有序


![forest](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Custom/forest_3_gurads.png)


<!-- endtab -->

<!-- tab CorrelationIdMiddleware-->


![trace](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Custom/trace_question.png)


當系統有很多請求同時進來時，Log 會變得很亂。如果我們能幫每一條請求配一個 唯一編號 (Correlation Id)，那麼不論請求經過多少 Service，都能用這個編號把 Log 串起來。就好比 📦 快遞公司給每個包裹一個「物流追蹤碼」，你就能隨時查到它走過的所有路徑


![trace](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Custom/traceid.png)


1. 嘗試從 Request Header 讀取 X-Correlation-ID（若前端傳進來就沿用）
2. 若沒有，就自己產生一個 Guid
3. 存到 HttpContext.Items（只活在這次請求週期，方便後面程式讀取）
4. 同時放進 Response Header（讓前後端對齊）
5. 用 ILogger.BeginScope() 把 CorrelationId 串進所有 Log


![trace](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Custom/trace_use.png)



```CSHARP
public class CorrelationIdMiddleware
{
    private const string CorrelationIdHeader = "X-Correlation-ID";
    private RequestDelegate _next;
    private readonly ILogger<CorrelationIdMiddleware> _logger;
    public CorrelationIdMiddleware(RequestDelegate next, ILogger<CorrelationIdMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public void Invoke(HttpContext context)
    {
        // 自 Header 取得 correlationId
        var correlationId = context.Request.Headers.TryGetValue(CorrelationIdHeader, out StringValues existing) ?
            existing.ToString() : Guid.NewGuid().ToString("N");

        // 裝進 Items, 其生命週期 by Request
        // 讓後續 Middleware/Service 可以取用, 假設某個 Service 想存 Audit Log，不需要再自己去解析 
        // Header，可以直接讀 HttpContext.Items["X-Correlation-Id"]。
        context.Items.Add(CorrelationIdHeader, correlationId);

        // 裝進 Response Header, 利於追蹤
        context.Response.OnStarting(() =>
        {
            if (!context.Response.Headers.ContainsKey(CorrelationIdHeader))
            {
                context.Response.Headers.Add(CorrelationIdHeader, correlationId);
            }
            return Task.CompletedTask;
        });

        var sw = Stopwatch.StartNew();

        // 開啟 logger 的 scope
        // 👉 一旦開了 Scope，這個範圍內的所有 Log 都會自動帶上 CorrelationId。
        using (_logger.BeginScope(new Dictionary<string, object> { ["CorrelationId"] = correlationId }))
        {
            _logger.LogInformation("Incoming {Method} {Path}", context.Request.Method, context.Request.Path);
            _next(context);
            sw.Stop();
            _logger.LogInformation("Completed {StatusCode} in {Elapsed} ms",
                context.Response.StatusCode, sw.ElapsedMilliseconds);
        }
    }
}
```

![logger](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Custom/automate_logger.png)


## 注入管線

```CSHARP
app.UseMiddleware<CorrelationIdMiddleware>();


app.MapGet("/debug", (HttpContext ctx) =>
{
    var cid = ctx.Items["X-Correlation-Id"]?.ToString();
    return $"Your Correlation Id = {cid}";
});
```


![trace](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Custom/id_flow.png)


<!-- endtab -->

<!-- tab 一次性攔截所有請求，回傳 503 Service Unavailable-->


![internal](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Custom/internal_test_question.png)


有時候系統要維護（更新、資料庫升級），就需要一種「一鍵開關」，大部分人進來 → 統一回 503 Service Unavailable。但特殊路徑（例如 /health，給 Kubernetes 偵測） → 照常回 200
特殊 IP（例如工程師的電腦） → 照常可用，方便內部測試。就像商場大門口貼「暫停營業」，但員工憑識別證還能進


![trace](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Custom/maintaince.png)


1. 檢查 Enabled 是否開啟
2. 如果符合 白名單路徑 / 白名單 IP，則直接放行
3. 其他請求 → 回 503，並加上 Retry-After

```json
"Maintenance": {
  "Enabled": true,
  "RetryAfter": "00:05:00",
  "BypassPaths": [ "/health" ],
  "AllowIps": [ "127.0.0.1" ]
}
```

![build](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Custom/maintiner_no_need_build.png)


一般請求
```bash
GET /products
→ 回 503 Service Unavailable
→ Header: Retry-After: 300 (秒)
→ Body: "Service under maintenance. Please try again later."
```

Health Check（例如 /health）
```bash
GET /health
→ 仍然回 200 OK （方便 Kubernetes / 負載平衡器檢查）
```

白名單 IP（例如 127.0.0.1）
```bash
GET /orders
→ 照常執行，不受影響（方便工程師自己測）
```

## MaintenanceModeMiddleware.cs

```csharp
using Microsoft.Extensions.Options;
using System.Globalization;
using System.Net;

public sealed class MaintenanceModeMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IOptionsMonitor<MaintenanceOptions> _options;

    public MaintenanceModeMiddleware(RequestDelegate next, IOptionsMonitor<MaintenanceOptions> options)
    {
        _next = next;
        _options = options;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var opt = _options.CurrentValue;

        if (!opt.Enabled || ShouldBypass(context, opt))
        {
            await _next(context);
            return;
        }

        //// 開始執行 "系統維護中" 維護性設定
        context.Response.StatusCode = StatusCodes.Status503ServiceUnavailable;

        // { } 代表「任何不是 null 的物件」。ra 就是把結果存起來的變數。
        if (opt.RetryAfter is { } ra)
            context.Response.Headers["Retry-fAter"] = ((int)ra.TotalSeconds).ToString(CultureInfo.InvariantCulture);

        await context.Response.WriteAsync("Service under maintenance. Please try again later.");
        // ⬆ 這裡短路，不呼叫 next()，避免後面還在處理。
    }

    private static bool ShouldBypass(HttpContext ctx, MaintenanceOptions opt)
    {
        var path = ctx.Request.Path;

        //// path 白名單
        if (opt.BypassPaths.Any(p => path.StartsWithSegments(p, StringComparison.OrdinalIgnoreCase)))
            return true;

        //// IP 白名單
        var ip = ctx.Connection.RemoteIpAddress;
        if (ip is not null && opt.AllowIps.Contains(ip.ToString()))
            return true;

        return false;
    }
}

public static class MaintenanceExtensions
{
    public static IServiceCollection AddMaintenance(this IServiceCollection services, IConfiguration cfg)
        => services.Configure<MaintenanceOptions>(cfg.GetSection("Maintenance"));

    public static IApplicationBuilder UseMaintenanceMode(this IApplicationBuilder app)
        => app.UseMiddleware<MaintenanceModeMiddleware>();
}
```


![maintainer](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Custom/shortcircuit.png)


<!-- endtab -->

<!-- tab ApiKeyMiddleware-->


![key](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Custom/apikey_question.png)


這個是給「合作夥伴 / 內部系統」的簡單驗證方式，不需要完整的 JWT 或 OAuth。只要在 Header 或 Query 帶正確的 API Key 才能進來。就像去遊樂園，要有門票才能進，不然直接擋在大門口，不讓你排隊


![key](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Custom/apiKey.png)


1. 檢查是否在 白名單路徑（例如 /health） → 直接放行
2. 嘗試從 Header / Query 抓 API Key
3. 如果 Key 不存在或錯誤 → 直接回 401 Unauthorized，不用浪費後面昂貴的資源（DB、Auth）

```CSHARP
public sealed class ApiKeyOptions
{
    public string HeaderName { get; set; } = "X-API-KEY";
    public string QueryName  { get; set; } = "api_key";
    public HashSet<string> Keys { get; set; } = new(StringComparer.Ordinal);
    public string[] BypassPaths { get; set; } = new[] { "/health" }; // 不檢查的路徑
}
```

```CSHARP
using Microsoft.Extensions.Options;

public sealed class ApiKeyMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IOptions<ApiKeyOptions> _options;

    public ApiKeyMiddleware(RequestDelegate next, IOptions<ApiKeyOptions> options)
    {
        _next = next;
        _options = options;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var opt = _options.Value;
        var path = context.Request.Path;

        if (opt.BypassPaths.Any(p => path.StartsWithSegments(p, StringComparison.OrdinalIgnoreCase)))
        {
            await _next(context);
            return;
        }

        var key = context.Request.Headers[opt.HeaderName].FirstOrDefault()
               ?? context.Request.Query[opt.QueryName].FirstOrDefault();

        if (string.IsNullOrEmpty(key) || !opt.Keys.Contains(key))
        {
            context.Response.StatusCode = StatusCodes.Status401Unauthorized;
            context.Response.Headers["WWW-Authenticate"] = "ApiKey";
            await context.Response.WriteAsync("API key is missing or invalid.");
            return; // ⬅ 終端中介軟體
        }

        await _next(context);
    }
}

public static class ApiKeyExtensions
{
    public static IServiceCollection AddApiKeyAuth(this IServiceCollection services, IConfiguration cfg)
    {
        var section = cfg.GetSection("ApiKeyAuth");
        services.Configure<ApiKeyOptions>(o =>
        {
            o.HeaderName  = section["HeaderName"] ?? o.HeaderName;
            o.QueryName   = section["QueryName"]  ?? o.QueryName;
            var keys = section.GetSection("Keys").Get<string[]>() ?? Array.Empty<string>();
            o.Keys = new HashSet<string>(keys, StringComparer.Ordinal);
            o.BypassPaths = section.GetSection("BypassPaths").Get<string[]>() ?? o.BypassPaths;
        });
        return services;
    }

    public static IApplicationBuilder UseApiKeyAuth(this IApplicationBuilder app)
        => app.UseMiddleware<ApiKeyMiddleware>();
}
```


![k](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Custom/apikey_logic.png)



```JSON
{
  "ApiKeyAuth": {
    "HeaderName": "X-API-KEY",
    "QueryName": "api_key",
    "Keys": [ "dev-key-123", "prod-key-456" ],
    "BypassPaths": [ "/health" ]
  }
}
```

## Program.cs

```CSHARP
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddApiKeyAuth(builder.Configuration);

var app = builder.Build();

app.UseRouting();
app.UseApiKeyAuth();                 // ⬅ 放在昂貴作業（Auth/DB）之前，可快速擋無效流量
app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/health", () => "OK");
app.MapGet("/products", () => new[] { "A", "B", "C" });

app.Run();
```

```bash
curl -i http://localhost:5000/products                          # 401
curl -i -H "X-API-KEY: dev-key-123" http://localhost:5000/products  # 200
curl -i "http://localhost:5000/products?api_key=dev-key-123"       # 200
```


![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Custom/test.png)


<!-- endtab -->

<!-- tab 結語-->


![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Custom/sequence.png)


到這裡，我們已經在森林裡走完三個重要的哨站

- `CorrelationIdMiddleware` 🌿 → 幫每位訪客掛上專屬號碼牌。
- `MaintenanceModeMiddleware` 🍂 → 維護時暫時拉下柵欄，但留後門給森林守衛。
- `ApiKeyMiddleware` 🌸 → 只有拿到通行證的人才能繼續前行。

就像一片健康的森林需要規則與守護，WebAPI 也需要 Middleware 來維持秩序與安全


![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Custom/summ.png)


<!-- endtab -->


{% endtabs %}