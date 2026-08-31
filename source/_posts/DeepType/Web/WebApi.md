---
title: WebAPI
date: 2025-09-23 17:07:34
categories: Web
top_img: https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
tags:
    - Web
toc:
toc_number:
comments :
---

{% tabs WebAPI%}

<!-- tab WebAPI-->


在我眼中，Web API 就像是一座小小的「橋樑」，一邊是使用者，他們可能透過手機、網頁，甚至 IoT 裝置提出需求，一邊則是我們的系統，安靜地等待著請求，準備回應。Controller 與 ControllerBase 就像這座橋樑的守門人，決定了資訊要如何被傳遞、如何被呈現

而我們依據「交付介質」決定地基：HTML 畫面需要完整的 View 引擎支援，而純數據傳輸應保持輕量無負擔，有些橋樑華麗，鋪上了燈飾與雕花（Controller，帶著 View 的世界），適合人群來往、熱鬧非凡；有些橋樑則簡單而純粹（ControllerBase，專注於 API），只管讓資料穩穩地走過去，沒有多餘的裝飾

我們現在散步在 WebApi 的世界中。走過這些橋樑，一邊看風景，一邊理解它們的職責與差異。會發現，無論是繁華還是簡約，Web API 的核心，始終是讓兩端能夠順暢而真誠地溝通


<!-- endtab -->

<!-- tab ControllerBase 與 Controller-->

在 ASP.NET Core 裡，Controller 與 ControllerBase 的差別是

- Controller：繼承自 ControllerBase，並且加入「View 支援」。適合用在 MVC，有畫面需要回傳 View 的情境
- ControllerBase：只有處理 Web API 要求 所需的功能，不包含 View。適合純粹的 API 專案

如果同一個控制器需要同時支援 View + API，就用 Controller。如果只寫 API，就用 ControllerBase


![view](https://github.com/CHI-KEKE/pics/blob/main/Web/API/mvc_api______controllerbase.png?raw=true)


## 公司形象官網

使用者點擊「關於我們」，伺服器需要回傳一個包含 CSS 與圖片佈局的 About.cshtml 頁面，因此選擇了 `Controller`

## 手機 App 後端

App 發送請求查詢「最新商品列表」，伺服器只需要回傳一個純文字的 JSON 陣列，不需要任何 HTML 標籤。因此選擇 `ControllerBase`


<!-- endtab -->

<!-- tab ControllerBase Return-->

- CreatedAtAction() → 回傳 201 Created 狀態碼，常用在新增成功後回傳新資源。
- BadRequest() → 回傳 400 Bad Request。
- NotFound() → 回傳 404 Not Found。
- PhysicalFile() → 回傳實體檔案。
- TryUpdateModelAsync() → 嘗試更新模型並做 Model Binding。
- TryValidateModel() → 嘗試做 Model 驗證。

📦 範例
```csharp
return CreatedAtAction(nameof(GetById), new { id = pet.Id }, pet);
```

回傳 201 Created告訴呼叫端「這個資源已建立，你可以用 GetById + id 來取得它」，Response Body 直接帶回剛建立好的 pet，前端拿到後可以直接顯示資料或用回傳的 location 再打一次 API


![v](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/API/returncode____standard.png)



## NotFound

```csharp
if (pet == null)
{
    return NotFound();
}
```

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.4",
  "title": "Not Found",
  "status": 404,
  "traceId": "0HLHLV31KRN83:00000001"
}
```

![v](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/API/returncode______404.png)

<!-- endtab -->

<!-- tab Attributes-->


在 WebAPI 裡，我們常用 Attributes 來修飾 Action 方法，指定它如何處理 HTTP 要求

```CSHARP
[HttpPost]
[ProducesResponseType(StatusCodes.Status201Created)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
public ActionResult<Pet> Create(Pet pet)
{
    pet.Id = _petsInMemoryStore.Any() ? 
             _petsInMemoryStore.Max(p => p.Id) + 1 : 1;
    _petsInMemoryStore.Add(pet);

    return CreatedAtAction(nameof(GetById), new { id = pet.Id }, pet);
}
```

[HttpPost] → 指定這個方法只處理 POST 請求
[ProducesResponseType] → 文件化 API，說明可能回傳的狀態碼與內容。Swagger / OpenAPI 文件就會根據這些資訊，產生更清楚的 API 規格


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/API/attribute____on_endpoint.png)


[ApiController] → 會自動幫你做一些常見處理


## ApiController - 自動驗證 ModelState

如果模型驗證失敗，系統會自動回傳 400 Bad Request，不用自己寫
```CSHARP
if (!ModelState.IsValid)
{
    return BadRequest(ModelState);
}
```

## 自動產生錯誤回應

回傳格式會是 ValidationProblemDetails，例如
```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "traceId": "|7fb5e16a-4c8f23bbfc974667.",
  "errors": {
    "": [
      "A non-empty request body is required."
    ]
  }
}
```

![v](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/API/attribute___apicontroller.png)


<!-- endtab -->

<!-- tab 自訂 Controller 行為 - 加入 Logging-->

自訂 ASP.NET Core Controller 在「Model 驗證失敗」時的回應行為（InvalidModelStateResponseFactory），在不破壞預設行為的前提下，把該留下的線索留下來（例如 logging）

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers()
    .ConfigureApiBehaviorOptions(options =>
    {
        //// 當 Model 驗證失敗（例如 [Required] 沒過），所有 API 都會先進到這裡。
        var builtInFactory = options.InvalidModelStateResponseFactory;

        options.InvalidModelStateResponseFactory = context =>
        {
            var logger = context.HttpContext.RequestServices
                                .GetRequiredService<ILogger<Program>>();

            // Perform logging here.
            // 到這裡你已經能：
            // 記錄錯誤欄位
            // 記錄 Request Path
            // 記錄使用者資訊
            // 串監控或告警系統
            return builtInFactory(context); //// 預設會回傳 ValidationProblemDetails (400)
        };
    });

var app = builder.Build();
```

![c](https://github.com/CHI-KEKE/pics/blob/main/Web/API/invalidmodeulstate______factory.png?raw=true)


<!-- endtab -->

<!-- tab 停用自動 400-->

關掉的是「框架替你做決定的那一刻」，把「什麼是錯、要怎麼回」的主導權，從 ASP.NET Core 手上拿回來

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers()
    .ConfigureApiBehaviorOptions(options =>
    {
        options.SuppressConsumesConstraintForFormFileParameters = true;
        options.SuppressInferBindingSourcesForParameters = true;
        options.SuppressModelStateInvalidFilter = true; //// 關閉 ModelState Invalid 時的自動 Filter
        options.SuppressMapClientErrors = true;
        options.ClientErrorMapping[StatusCodes.Status404NotFound].Link =
            "https://httpstatuses.com/404"; //// 自訂 Client Error Mapping（404 連結）
    });

var app = builder.Build();
```


![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/API/invalidmodulestate_____manual.png)


<!-- endtab -->

<!-- tab Buffering IEnumerable<T>-->

Buffer (緩衝區) 本質上就是一塊記憶體，用來存放「還沒送出的資料」。某些序列化器（Newtonsoft.Json / XML Formatter）需要知道完整集合的大小或結構，才能正確輸出。所以 ASP.NET Core 只能等全部資料都準備好，才能送

- `Controller` 開始執行
- 把所有 `yield return` 的資料收集到一個清單
- JSON 序列化器把清單轉成完整 JSON 字串
- 一次性把完整 JSON 寫進 Response
- 前端才收到資料

➡️ 前端必須等「最後一刻」，才會拿到任何東西


假設你寫了這樣的 API
```csharp
public IEnumerable<int> GetNumbers()
{
    for (int i = 0; i < 5; i++)
    {
        yield return i;
        Thread.Sleep(1000);
    }
}
```

以為會第 1 秒前端收到 0、第 2 秒收到 1看起來像 streaming，而實際前 5 秒：前端完全沒資料、第 5 秒結束後：一次收到 [0,1,2,3,4]，它被強制「跑完一整輪」才准輸出
JSON 不是流水帳，而是有頭有尾的格式，沒看到結尾就沒辦法安心開頭，「正確性優先」，不是「即時性優先」

![s](https://github.com/CHI-KEKE/pics/blob/main/Web/API/response______buffer.png?raw=true)


<!-- endtab -->

<!-- tab Streaming IAsyncEnumerable<T>-->

ASP.NET Core 使用 `IAsyncEnumerable<T>` 進行 `Streaming`（逐筆序列化、逐筆輸出）

- Controller 開始執行
- 第一筆資料產生，馬上被序列化
- 立刻寫進 Response，前端可以收到這筆資料
- 下一筆資料出來，再寫進 Response
- 重複，直到所有資料傳送完畢

➡️ 前端可以「邊等邊收到資料」，不用等全部完成

`System.Text.Json` + `IAsyncEnumerable<T>` 支援逐筆序列化與輸出，所以能邊跑邊送，不需要 Buffering。

- `Buffering`：一定要等電影全拍好、全剪好，打包成一整部電影檔案，才能播放。觀眾要等完整影片檔才看到第一個畫面
- `Streaming`：邊拍邊直播，攝影機一拍馬上送到螢幕，你可以即時看到


![cc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/API/response_____streaming.png)

<!-- endtab -->

<!-- tab IActionResult-->


一個 Action 方法可能有「多種可能回應」

例如：

- 找到資料 → 回傳 200 OK 和資料。
- 找不到資料 → 回傳 404 Not Found。
- 請求內容不正確 → 回傳 400 Bad Request。

因為回傳結果不只一種，所以用 ActionResult/IActionResult 來彈性表示「任何合法的 HTTP 狀態回應」。IActionResult 是介面（interface），代表一個結果。ActionResult 是一個實作類別，它支援更多功能，也能讓你直接回傳模型物件，例如 ActionResult<Product>。習慣上，如果會同時回傳不同型別（像 200 帶物件 + 404 沒東西），就用 ActionResult<T>。如果單純只有狀態碼，就用 IActionResult

IActionResult 比較舊、在 MVC 和 Web API Controller 常用，需要手動補 Swagger metadata
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult GetById(int id)
    {
        var product = _productContext.Products.Find(id);
        return product == null ? NotFound() : Ok(product);
    }
}
```

![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/API/responsetype____iacion_action.png)

<!-- endtab -->

<!-- tab IResult-->


Minimal API 引進的簡化方式，但沒有型別資訊
```csharp
app.MapGet("/products/{id}", (int id, ProductContext db) =>
{
    var product = db.Products.Find(id);
    return product == null ? Results.NotFound() : Results.Ok(product);
});
```

<!-- endtab -->

<!-- tab TypedResults + Results<TResult1, TResultN>-->


新一代做法，結合強型別與 API 文件自動產生，讓 回應更嚴謹、Swagger 更正確
```csharp
[HttpGet("{id}")]
public Results<NotFound, Ok<Product>> GetById(int id)
{
    var product = _productContext.Products.Find(id);
    return product == null ? TypedResults.NotFound() : TypedResults.Ok(product);
}
```

![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/API/responsetype_____imporve.png)


<!-- endtab -->

<!-- tab 格式特定的回傳-->


一些 ActionResult 是「綁定特定格式」的，例如：

- JsonResult → 永遠回傳 JSON 格式
- ContentResult → 永遠回傳純文字（text/plain）

即使用戶端要求不同的格式（透過 Accept header），這些結果也會堅持回傳固定格式

![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/API/returntype____force_type.png)


<!-- endtab -->

<!-- tab summmary-->

![d](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/API/decision___table.png)

![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/API/summ___pic.png)

<!-- endtab -->

{% endtabs %}