---
title: Middleware - Relay
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


{% tabs Relay%}


<!-- tab Middleware 的運作原理-->


![relay](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Relay/landing.png)


想像一個 HTTP 請求進到伺服器，就像一場 **接力賽跑**

- **接力棒** 就是 `HttpContext`，裡面裝著這次請求與回應的所有資訊
- **選手** 就是一個 Middleware
- **接棒的手勢** 就是 `RequestDelegate`
- **比賽終點** 就是 Controller 或 Razor Page，產生最後的回應

ASP.NET Core 的 Middleware 機制，就是靠這樣一場「人人都要接棒」的比賽，把請求一路傳遞下去，直到完成回應


![relay](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Relay/cast.png)


<!-- endtab -->

<!-- tab pipeline-->


在 ASP.NET Core 中，整個 HTTP 請求(Request) 和 回應(Response) 的處理，就像一條 管線 (Pipeline)。這條管線是由一個個的小元件（我們叫它 Middleware）組合起來的。決定要不要把請求交給下一個中介軟體。在下一個中介軟體之前做事（例如：紀錄 log、驗證身分）。在下一個中介軟體之後做事（例如：修改回應、壓縮資料）。如果一個中介軟體選擇不把請求往下傳，它就會 「短路 (Short-circuit)」 管線。這時它就是終端中介軟體 (Terminal Middleware)


![pipeline](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Relay/compose_and_shortcircuit.png)


**Use** → 可以在 前後 做事（像一個攔截器 Interceptor）
**Run** → 是一個 終點，到這裡管線就結束了
**Map** → 可以做條件式分支，像是「如果路徑是 /api，就走另一條管線」


![use_run](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Relay/use_and_run.png)
![map](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Relay/map.png)


中介軟體是一種 組合模式 (Composable Pattern)，把一堆小元件串起來，最終形成整個請求處理流程。
```csharp
var app = builder.Build();

app.Use(async (context, next) =>
{
    Console.WriteLine($"接收到請求 : {context.Request.Path}");
    await next();
});

app.Use(async (context, next) =>
{
    Console.WriteLine($"接收到請求 : {context.Request.Path}");
    await next();
});

app.Use(async (context, next) =>
{
    await next(); //// 這邊如果反過來 等於是 response 後又加上 Header, 是錯誤的操作!!
    context.Response.Headers.Add("X-Custom-Header", "MyCustomHeaderValue");
});

app.Run(async context =>
{
    await context.Response.WriteAsync("Hello from the terminal middleware!");
});
```


![response](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Relay/response_header_missing.png)


<!-- endtab -->

<!-- tab 為什麼 Middleware 裡要寫 async？直接呼叫 next() 不行嗎？-->


在 ASP.NET Core 裡，Middleware 是一個「請求管線的節點」。請求從瀏覽器來到伺服器後，會一層一層地通過 Middleware，最後才送出回應。next() 就是「把請求交給下一個 Middleware」，但這個過程大部分會牽涉到 非同步 I/O

- 讀取 Request.Body → 非同步
- 呼叫資料庫 → 非同步
- 呼叫外部 API → 非同步
- 寫 Response → 非同步


![async](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Relay/async_next.png)


所以 ASP.NET Core 設計上，next() 幾乎一定會回傳一個 Task。如果你沒有 await next()，那就等於「你只是叫下一個人開始工作，但不等他做完，就自己繼續做事。」這樣會導致工作流程亂掉

![right](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Relay/await_right.png)

假設我們寫一個簡單的 Middleware，紀錄請求與回應時間
```csharp
public class TimingMiddleware
{
    private readonly RequestDelegate _next;

    public TimingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var start = DateTime.Now;

        // ✅ 正確：等下一個 Middleware 跑完再繼續
        await _next(context);

        var end = DateTime.Now;
        var duration = end - start;
        Console.WriteLine($"請求耗時: {duration.TotalMilliseconds} ms");
    }
}
```

![zero](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Relay/await_miss.png)

如果你 不用 await，改成這樣
```csharp
public async Task InvokeAsync(HttpContext context)
{
    var start = DateTime.Now;

    // ❌ 錯誤：只是呼叫，但不等待
    _next(context);

    var end = DateTime.Now;
    Console.WriteLine($"請求耗時: {(end - start).TotalMilliseconds} ms");
}
```

執行結果會幾乎是 0 ms，因為程式馬上就跑到 Console.WriteLine，完全沒等下一個 Middleware 或 Controller 處理完。

<!-- endtab -->

<!-- tab 🌿 Map 分支-->


一般 UseXxx 中介軟體會形成一條直線式管線
```bash
Request 進來 → Middleware1 → Middleware2 → Middleware3 → Response
```

但實務上我們可能需要依條件走不同的管線

- 依照路徑（例如 /api 一條管線，/admin 一條管線）
- 依照 QueryString 或 Header（例如有特殊參數才走另一個邏輯）
- 依照狀態（例如測試用戶走不同流程）


這時候就會用到 Map、MapWhen、UseWhen


![usewhen](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Relay/use_when.png)


### Map (依路徑分支)

根據 URL Path 來決定要不要分支。一旦路徑符合 → 會把匹配的部分從 HttpContext.Request.Path 移除，存到 PathBase。

```csharp
app.Map("/map1", branchApp =>
{
    branchApp.Run(async context =>
    {
        await context.Response.WriteAsync("Map Test 1");
    });
});

app.Map("/map2", branchApp =>
{
    branchApp.Run(async context =>
    {
        await context.Response.WriteAsync("Map Test 2");
    });
});
```

結果是
```bash
/map1 → Map Test 1
/map2 → Map Test 2
/map3 → 落到主管線 (non-Map delegate)
```

這邊只看「開頭是否匹配」。一旦進入分支 → 不會再回到主管線。

<br>

### MapWhen (依條件分支)

不是依 Path，而是依 predicate 判斷要不要進分支

```csharp
app.MapWhen(
    context => context.Request.Query.ContainsKey("branch"),
    branchApp =>
    {
        branchApp.Run(async context =>
        {
            var branchVer = context.Request.Query["branch"];
            await context.Response.WriteAsync($"Branch used = {branchVer}");
        });
    });
```

結果

```bash
/ → Hello from non-Map delegate.
/?branch=main → Branch used = main
```

一旦進入分支 → 也不會回到主管線。適合「完全不同處理邏輯」的情境。

<!-- endtab -->

<!-- tab 🌿 RequestDelegate-->


在 ASP.NET Core 裡，每個 HTTP 請求 進來後，都會經過一條「Middleware 處理管線」。這條管線上的每一個節點，都是一個 函式，它的格式統一就是
```csharp
Task SomeMiddleware(HttpContext context);
```
它一定要能接收 HttpContext（代表這次 HTTP 請求與回應的全部資訊）。還會回傳一個 Task（因為處理請求通常是非同步的）。為了讓整個框架可以用一個「型別」來代表這種函式，ASP.NET Core 定義了
```csharp
public delegate Task RequestDelegate(HttpContext context);
```


![delegate](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Relay/requestdelegate_design.png)


而 RequestDelegate 本質上就是一個 委派 (delegate)，跟 `Func<T>` 或 `Action<T>` 的概念很像，只是它專門用來代表 ASP.NET Core 的一個 HTTP 請求處理函式，RequestDelegate 就像一個 接力棒，把 HttpContext 傳下去，直到管線結束（通常是 Controller 或 Razor Page）

```csharp
public class HelloMiddleware
{
    private readonly RequestDelegate _next;

    public HelloMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        await context.Response.WriteAsync("Hello, Middleware!\n");
        await _next(context); // 呼叫下一個 Middleware
    }
}
```

- _next 就是 RequestDelegate
- 當你 await _next(context) 時，其實就是「把這個 HttpContext 交給下一個中介軟體去處理」


![next](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Relay/next_requestdelegate.png)


如果在 Startup 註冊這個 Middleware，ASP.NET Core 在背後會自動生成一個 RequestDelegate，並把它傳給你的建構子
```csharp
app.UseMiddleware<HelloMiddleware>();
```

而且也可以用 lambda 直接寫
```csharp
app.Use(async (context, next) =>
{
    await context.Response.WriteAsync("Before next()\n");
    await next(context); // 這裡的 next 也是 RequestDelegate
    await context.Response.WriteAsync("After next()\n");
});
```

<!-- endtab -->

<!-- tab 🌿 結語-->


![sum](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Relay/summary_aa.png)


Middleware 的世界其實不複雜，只要你把它想成一場「接力比賽」

- 每個人都會拿到同一根接力棒（HttpContext）
- 過程中可以加料：寫 Log、驗證、加 Header
- 只要記得 **要把棒子交下去（await next(context)）**，比賽才能繼續
- 如果有人沒交棒，比賽就會在他手中結束（Terminal Middleware）

而 `RequestDelegate` 就是那個「交接的標準動作」，確保大家能正確傳遞接力棒，讓整場比賽順利完成


![sum2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Relay/summary.png)


<!-- endtab -->


{% endtabs %}