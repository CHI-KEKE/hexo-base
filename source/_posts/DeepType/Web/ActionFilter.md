---
title: WebApi - ActionFilter
date: 2025-09-23 23:07:34
categories: Web
top_img: https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
tags:
    - Web
toc:
toc_number:
comments :
---

{% tabs ActionFilter%}


<!-- tab ActionFilter-->

在 ASP.NET MVC / Web API 裡，Action Filter（動作篩選器）就像是一個「小保全」或「攔截器」。
它可以在 Action 執行前 或 Action 執行後 插入邏輯，幫你做額外的事情

把它想像成一場活動

- 活動開始前檢查門票 → `OnActionExecuting`
- 活動結束後收拾場地 → `OnActionExecuted`
- 結果要交給觀眾前再整理一次 → `OnResultExecuting` / `OnResultExecuted`
- 活動中如果出意外要處理 → `OnException`

好處是不用每個 Controller / Action 都重複寫相同程式，邏輯可以集中管理，讓程式更乾淨


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ActionFilter/1__guard.png)

<!-- endtab -->

<!-- tab 內建 Action Filter-->


```csharp
public class HomeController : Controller
{
    // 使用 OutputCache 篩選器，快取結果 60 秒
    [OutputCache(Duration = 60)]
    public ActionResult Index()
    {
        return View();
    }

    // 使用 HandleError 篩選器，處理動作內的例外
    [HandleError]
    public ActionResult ErrorDemo()
    {
        throw new Exception("這是一個錯誤示範");
    }

    // 使用 Authorize 篩選器，限制只有 Admin 角色能進來
    [Authorize(Roles = "Admin")]
    public ActionResult AdminPage()
    {
        return View();
    }
}
```

<!-- endtab -->

<!-- tab Pipeline-->

## MVC

- Routing → 根據 URL 找到要跑哪個 Controller / Action
- Controller Factory → 建立 Controller 物件
- Action Invoker → 準備執行 Action
- Filters (這裡就插進來！)
- Authorization Filters (檢查權限)
- Action Filters (Action 前後的攔截)
- Result Filters (ViewResult 前後的攔截)
- Exception Filters (錯誤攔截)
- 執行 Action
- 產生 ActionResult (通常是 View 或 JSON)
- 渲染結果 (Render View)
- 送回 Response


Action Filter → 貼在 Action 執行的前後
Result Filter → 貼在 ActionResult 執行的前後
Exception Filter → 如果 Action 或 Result 出錯，就會跳進來

<br>

## Web API

- Routing → 決定哪個 Controller / Action
- Authentication Filters → 確認使用者是誰 (驗證 JWT、API Key 等)
- Authorization Filters → 檢查使用者有沒有權限執行
- Action Filters →
    OnActionExecuting (Action 執行前)
    OnActionExecuted (Action 執行後)
- 執行 Action → 執行 API 方法，得到回傳結果
- Result Filters → 對回傳結果 (IHttpActionResult) 做額外處理
- Exception Filters → Action 或 Result 出錯時攔截
- 格式化 (Formatter) → 把回傳物件轉成 JSON 或 XML
- 送回 Response


Filter 就像是 局部版的 Middleware，負責攔截 Request / Response。差別是：Middleware 影響「整條管線」，Filter 只針對「Controller / Action」。
在 Web API 裡，Filter 更細分，還多了 Authentication / Authorization / Formatter 的位置。本質上就是在 Request 流程的不同階段，插入「攔截點」來控制 Request 或 Response。


AuthorizationFilter – 實作 IAuthorizationFilter 屬性。
ActionFilter  – 實作 IActionFilter 屬性。
ResultFilter – 實作 IResultFilter 屬性。
ExceptionFilter – 實作 IExceptionFilter 屬性。
篩選器按照上面列出的順序執行。 例如，授權篩選器始終在動作篩選器之前執行，例外狀況篩選器始終在所有其他類型的篩選器之後執行。


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ActionFilter/1_lifetime.png)


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ActionFilter/2_pipeline.png)


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ActionFilter/3_different_filters.png)

<!-- endtab -->

<!-- tab 執行時間記錄-->

```csharp
using System.Diagnostics;
using System.Web.Http.Controllers;
using System.Web.Http.Filters;

public class ExecutionTimeFilter : ActionFilterAttribute
{
    private Stopwatch stopwatch;

    public override void OnActionExecuting(HttpActionContext actionContext)
    {
        stopwatch = Stopwatch.StartNew();
    }

    public override void OnActionExecuted(HttpActionExecutedContext actionExecutedContext)
    {
        stopwatch.Stop();
        var elapsed = stopwatch.ElapsedMilliseconds;

        // 這裡可以把執行時間記錄到 Log
        System.Diagnostics.Debug.WriteLine(
            $"Action {actionExecutedContext.ActionContext.ActionDescriptor.ActionName} 執行時間: {elapsed} ms");
    }
}

public class ProductsController : ApiController
{
    [ExecutionTimeFilter]
    public IHttpActionResult Get()
    {
        return Ok(new[] { "Apple", "Banana", "Cherry" });
    }
}
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ActionFilter/4_1_executiontime.png)


<!-- endtab -->

<!-- tab 統一回傳格式-->



例如希望所有 API 的回傳長這樣
```JSON
{
  "success": true,
  "data": [ ... ],
  "errorMessage": null
}
```

```CSHARP
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;

public class ApiResponseFilter : ActionFilterAttribute
{
    public override void OnActionExecuted(ActionExecutedContext context)
    {
        if (context.Exception != null) return;

        if (context.Result is ObjectResult objectResult)
        {
            var wrapped = new
            {
                success = true,
                data = objectResult.Value,
                errorMessage = (string)null
            };

            context.Result = new ObjectResult(wrapped)
            {
                StatusCode = objectResult.StatusCode ?? 200
            };
        }
    }
}

// 套用篩選器
[ApiResponseFilter]
public class ProductsController : ApiController
{
    public IHttpActionResult Get()
    {
        return Ok(new[] { "Apple", "Banana", "Cherry" });
    }
}


//{
//  "success": true,
//  "data": ["Apple", "Banana", "Cherry"],
//  "errorMessage": null
//}
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ActionFilter/4_2_responseunified.png)


<!-- endtab -->


<!-- tab 統一錯誤處理-->



```CSHARP
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;

public class ApiExceptionFilter : IExceptionFilter
{
    public void OnException(ExceptionContext context)
    {
        var response = new
        {
            success = false,
            data = (object)null,
            errorMessage = context.Exception.Message
        };

        context.Result = new ObjectResult(response)
        {
            StatusCode = StatusCodes.Status500InternalServerError
        };
    }
}
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ActionFilter/5_exception_handle.png)


<!-- endtab -->

<!-- tab 輸入驗證 (Model Validation Filter)-->

如果 ModelState 驗證失敗，直接攔截回傳錯誤，不讓 Action 繼續執行
```CSHARP
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;

public class ValidateModelFilter : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
        {
            var response = new
            {
                success = false,
                data = (object)null,
                errorMessage = "輸入資料格式錯誤",
                errors = context.ModelState
            };

            context.Result = new BadRequestObjectResult(response);
        }
    }
}


// 使用方式
[ValidateModelFilter]
public class UsersController : ApiController
{
    public IHttpActionResult Post(UserDto dto)
    {
        return Ok("新增成功");
    }
}
```


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ActionFilter/6_validate_model_filter.png)


<!-- endtab -->

<!-- tab 防止重複送出請求 (Idempotency Filter)-->

避免使用者重複點擊送出造成重複下單。可以根據 Request 的唯一 Token 來攔截。
```CSHARP
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;

public class PreventDuplicateRequestFilter : ActionFilterAttribute
{
    private static readonly HashSet<string> RequestTokens = new HashSet<string>();
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        if (context.HttpContext.Request.Headers.TryGetValue("Request-Id", out var requestId))
        {
            if (!RequestTokens.Add(requestId!))
            {
                context.Result = new ConflictObjectResult(new
                {
                    success = false,
                    errorMessage = "請求已處理，請勿重複送出"
                });
            }
        }
    }
}
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ActionFilter/7_idenpotency.png)


<!-- endtab -->

<!-- tab ASP.NET Core 中的註冊方式-->

## 只用在某個 Controller / Action

在 ASP.NET Core 只要繼承 ActionFilterAttribute，就能像 Attribute 一樣直接套用
```CSHARP
[ApiResponseFilter]
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        return Ok(new[] { "Apple", "Banana", "Cherry" });
    }
}
```

<br>

## 全域註冊 (所有 API 都套用)

在 ASP.NET Core 只要繼承 ActionFilterAttribute，就能像 Attribute 一樣直接套用
```CSHARP
builder.Services.AddControllers(options =>
{
    options.Filters.Add<ApiResponseFilter>();
    options.Filters.Add<ApiExceptionFilter>();
    options.Filters.Add<ValidateModelFilter>();
    options.Filters.Add<PreventDuplicateRequestFilter>();
});
```

<br>

## 依照需求註冊到 DI (可控制作用範圍)
```CSHARP
builder.Services.AddScoped<ApiResponseFilter>();
builder.Services.AddScoped<ApiExceptionFilter>();

[ServiceFilter(typeof(ApiResponseFilter))]
public class UsersController : ControllerBase
{
    [HttpPost]
    public IActionResult Post(UserDto dto) => Ok("新增成功");
}
```


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ActionFilter/8_registration_strategy.png)


<!-- endtab -->

<!-- tab Filter 不要存狀態！-->

在 ASP.NET MVC 3+ 與 ASP.NET Core，Filter 實例可能被快取重用，並不是每次 Request 都 new 一個新的。
👉 這代表如果你把狀態存成 欄位變數，可能被多個 Request 共用，導致資料被覆蓋

✔️ 正確做法是用 HttpContext.Items 保存 Request 專屬資料或使用依賴注入 (Scoped Service) 來保存

```csharp
public class ExecTimeFilter : ActionFilterAttribute
{
    private const string Key = "ExecTimeFilter.Start";

    public override void OnActionExecuting(ActionExecutingContext context)
    {
        context.HttpContext.Items[Key] = DateTime.Now;
    }

    public override void OnActionExecuted(ActionExecutedContext context)
    {
        if (context.HttpContext.Items.ContainsKey(Key))
        {
            var start = (DateTime)context.HttpContext.Items[Key]!;
            var elapsed = DateTime.Now - start;
            context.HttpContext.Response.Headers.Add("X-Elapsed-Time", $"{elapsed.TotalMilliseconds}ms");
        }
    }
}
```
👉 每個 Request 都有自己的 HttpContext.Items 字典，不會互相干擾


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ActionFilter/9_thread_thefty.png)


<!-- endtab -->


<!-- tab summary-->


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ActionFilter/10_takeaway.png)


<!-- endtab -->


{% endtabs %}