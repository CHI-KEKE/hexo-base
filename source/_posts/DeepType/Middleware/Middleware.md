---
title: Middleware
date: 2025-11-06 14:07:05
categories: Middleware
top_img: https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
tags: WebAPI
    - 
toc:
toc_number:
comments :
---

{% tabs Middleware%}

![onion](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/landing_onion.png)


<!-- tab Middleware 的運作原理-->


ASP.NET Core 的請求流程是一條「洋蔥模型」


![onion](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/request_is_onion.png)


當一個 Request 進來，他進入 ASP.NET Core 的處理管線（Pipeline），依照註冊順序進入 Middleware，第一個 UseXXX 註冊的 Middleware 最先被執行，每個 Middleware 都「包住」後面所有的 Middleware


![wrap](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/middleware_is_wrapper.png)


![use](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/program.png)


當你呼叫 UseMiddleware() ，Framework 會產生一個 RequestDelegate 並把「下一個 Middleware」當成 next 傳進來，而在 Middleware 執行自己的邏輯時，可以在這裡做任何事（Log、驗證、改 Header…）、決定要不要呼叫 await next(context)，若呼叫表示請求繼續往下一層走，選擇不呼叫，則流程在這一層就被「攔截」


![delegate](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/request_delegate.png)


此時 Response 從最內層往外回傳

![response](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/response.png)


<!-- endtab -->

<!-- tab 如何「交換 Middleware 的順序」-->


## ✅ 方法一：直接調整註冊順序

在 `Program.cs` 或 `Startup.cs` 的 `Configure()` 方法中，Middleware 是 照順序被執行 的
```csharp
app.UseMiddleware<A>();
app.UseMiddleware<B>();
app.UseMiddleware<C>();
```

執行順序：Request → A → B → C → Controller

若改順序
```csharp
app.UseMiddleware<C>();
app.UseMiddleware<A>();
app.UseMiddleware<B>();
```

就變成：Request → C → A → B → Controller

ASP.NET Core 的 Middleware 管線 是固定順序、靜態構建，一旦應用啟動後，順序無法在 runtime 動態交換，所以你必須在啟動階段（Configure）明確控制註冊順序，沒有提供「動態插拔或交換」的機制


![exchange](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/exchange_order.png)


<br>

## ✅ 方法二：動態條件式決定是否呼叫下一層

如果你想在 runtime「有時跳過某層、有時通過」，可以透過條件控制 await next(context) 的呼叫

```csharp
public class MyMiddleware
{
    private readonly RequestDelegate _next;
    public MyMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        if (context.Request.Path.StartsWithSegments("/health"))
        {
            // 不繼續往下傳 → 直接跳過下一層
            await context.Response.WriteAsync("OK");
            return;
        }

        // 繼續執行下一層
        await _next(context);
    }
}
```

/health 請求會被攔截、直接回應（跳過之後的所有 Middleware），其他路徑才繼續往下傳

<!-- endtab -->

<!-- tab 如何「跳過」特定 Middleware（但不跳過整條）-->


有時你想「A → B → C → Controller」，但在某些情況想「A → C → Controller」而略過 B，這種情況可以用「條件判斷 + 不呼叫 next()」的方式達成

`UseWhen()` 是 ASP.NET Core 提供的「分支式 Middleware」，可以依條件決定是否啟用某段管線，如此可以避免條件寫在 Middleware 內，讓 Middleware 不會同時承擔「業務邏輯」與「流程控制」

```csharp
// Startup.cs
app.UseMiddleware<A>();
app.UseWhen(
    context => !context.Request.Path.StartsWithSegments("/skip-b"),
    appBuilder => appBuilder.UseMiddleware<B>()
);
app.UseMiddleware<C>();
```

若 URL 是 /skip-b/... → 會跳過 Middleware B，其他路徑仍會進 B

```csharp
app.UseMiddleware<LoggingMiddleware>();

app.UseWhen(
    ctx => ctx.Request.Path.StartsWithSegments("/api"),
    branch =>
    {
        branch.UseMiddleware<ApiAuthMiddleware>();   // 只在 /api 下執行
        branch.UseMiddleware<ApiRateLimitMiddleware>();
    }
);

app.UseMiddleware<MvcMiddleware>();
```

<!-- endtab -->

<!-- tab 多層 Middleware 之間交換資訊-->


![items](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Items/challenge.png)


Middleware 之間常要傳遞狀態（例如驗證結果、追蹤 ID），可以透過 HttpContext.Items 共享

```csharp
// Middleware A
context.Items["TraceId"] = Guid.NewGuid().ToString();
await _next(context);

// Middleware B
if (context.Items.TryGetValue("TraceId", out var traceId))
{
    _logger.LogInformation($"TraceId: {traceId}");
}
```


![traceId](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Items/traceid_pass.png)


當 Request 進入 Middleware A，系統會建立一個 HttpContext，這個 Context 專屬於這一次請求，Middleware A 把資料放進 context.Items，Items 本質是一個 Dictionary，而 Key / Value 都只在這次 Request 存在，呼叫 _next(context) 時，同一個 HttpContext 被一路往下傳，Middleware B 讀取同一份 Items 因為是同一個記憶體參考，最後 Response 回傳後，HttpContext 被釋放，Items 裡的資料自然消失


![context](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Items/context_lifecycle.png)


## MemberManager - 會員身分相關的一切複雜度集中在一個地方

先來說說這個 service 的作用


#### 1️⃣ 它是「登入狀態的唯一權威來源」

```csharp
public static bool IsLogin(...)
public static MemberEntity Current
```

不管是 Cookie、Redis、未登入狀態、官網 / 商店 / APP，所有地方都透過 MemberManager 判斷，「不要自己判斷登入，一律問 MemberManager」


![manager](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Items/member_source_of_truth.png)


#### 2️⃣ 它處理「登入不是只有 Cookie 那麼簡單」這件事

這套系統的登入其實包含

- auth cookie（Web）
- uAUTH cookie（未登入識別）
- Redis Login Cache（跨 Domain / 官網）
- SessionExpire（主動登出 / 過期）
- SameSite / Secure / HttpOnly 相容處理

這些都不是 Controller 該知道的事


![encap](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Items/Member_Encap.png)


#### 3️⃣ 它同時支援「已登入」與「未登入用戶」

這代表即使使用者還沒登入，系統依然能記購物行為、綁 APP、發簡訊、建立 Redis 狀態，因為「沒登入 ≠ 沒身分」


![nologin](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Items/nologin.png)


## 為什麼 MemberManager 選擇使用 HttpContext.Items 作為狀態暫存機制


因為系統需要的是「只在單一 Request 有效、跨多層共用、又不能污染全域」的資料容器，而 .Items 剛好精準命中這個需求


![choose](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Items/why_member_choose_items.png)


```csharp
HttpContext.Current.Items["isLogin"]
HttpContext.Current.Items["Member.Current"]
```

我們需要同一個 Request 內會被多次使用、不同方法／不同層（Controller、BL、Helper）都要看得到，並且 Request 結束就該自動消失，不能影響其他使用者或其他 Request，這四個條件，同時成立 => 使用 .Items ，在這個專案中，選擇使用 HttpContext.Items，是為了在「不增加架構複雜度」的前提下，提供一個安全、低成本、只限單一 Request 的狀態共享與快取機制


![items](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Items/four_column.png)


<!-- endtab -->


{% endtabs %}