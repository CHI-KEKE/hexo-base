---
title: Swagger
date: 2025-09-26 08:28:34
categories: Web
top_img: https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
tags:
    - Web
toc:
toc_number:
comments :
---

![Image](https://github.com/CHI-KEKE/pics/blob/main/WebAPI/swagger.png?raw=true)


後端 API 已經改好了，但前端卻還在用舊的呼叫方式，文件也沒有更新，這就像是兩個人在對話，一個講新語言，一個還在講舊語言，結果就會雞同鴨講。

這時候，Swagger 就像是一個貼心的小幫手 🐥。它會自動幫你把 API 說明整理好，還能變成互動式的測試工具。更重要的是，它跟 OpenAPI 規範搭配之後，文件就不再只是「寫給人看的文字」，而是一份「電腦也能讀懂的契約」。
這意味著，前後端、測試工具、甚至 SDK 產生器，都能直接根據這份文件來溝通。

<br>

## 🌿 套件

Swashbuckle.AspNetCore.SwaggerUi

<br>
<br>

## 🌿 文件

https://swagger.io/api-hub/

<br>
<br>

## 🌿 Swagger 與 OpenAPI 的關係

<br>

### Swagger 是一個「專案 / 工具集」

最早由 Wordnik 在 2010 左右提出的 API 文件產生工具，叫做 Swagger。它的核心想法：用一份 規格文件 (JSON / YAML) 來描述 API，然後可以自動產生「文件頁面」和「測試工具」。
這份規格文件，後來被業界廣泛接受。

<br>

### OpenAPI 是「規範 (specification)」

到了 2016 年，Swagger 規格捐給了 Linux Foundation，成立了 OpenAPI Initiative (OAI)。從那之後，Swagger 規格改名為 OpenAPI Specification (OAS)。
也就是說

一開始 (Swagger 1.x / 2.0)，大家習慣把「規格」和「工具」都叫 Swagger。後來 (2016 年)，這個規格捐給 Linux Foundation，改名叫 OpenAPI Specification (OAS)，成為一個獨立的標準。
Swagger 這個名字就保留下來，專指那一組工具（Swagger UI、Swagger Editor、Swagger Codegen...）。

常見的 Swagger 工具

- Swagger UI → 把 OpenAPI 文件轉成可互動的網頁
- Swagger Editor → 線上編輯 OpenAPI 文件
- Swagger Codegen → 依據 OpenAPI 產生客戶端 / 伺服端程式碼
- Swashbuckle (ASP.NET Core 套件) → 自動把 Controller 轉成 OpenAPI

<br>
<br>

## 🌿 註冊

是在 ASP.NET Core 專案裡，啟用 Swagger (OpenAPI) 文件產生器的註冊。
```csharp
using Microsoft.OpenApi.Models;

var builder = WebApplication.CreateBuilder(args);

// 註冊 Swagger 服務
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "Todo API",
        Version = "v1",
        Description = "A simple example ASP.NET Core Web API with Swagger"
    });
});

var app = builder.Build();

// 在開發環境啟用 Swagger
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(options =>
    {
        options.SwaggerEndpoint("/swagger/v1/swagger.json", "Todo API v1");
        options.RoutePrefix = string.Empty; // 讓 Swagger UI 顯示在根目錄 http://localhost:5000/
    });
}

app.MapControllers();
app.Run();
```
AddSwaggerGen() 來自 Swashbuckle.AspNetCore 套件。它會把你的 API Controllers / Actions 掃描起來，自動產生符合 OpenAPI 規範 (Swagger) 的 API 文件。這份文件通常位於 /swagger/v1/swagger.json。

<br>
<br>

### .NET 8 的新方式

在 .NET 8，微軟直接內建了 AddOpenApi() 方法，比 AddSwaggerGen() 更簡單。
```csharp
var builder = WebApplication.CreateBuilder(args);

// 使用 .NET 8 內建的 OpenAPI 支援
builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi(); // 自動掛載 OpenAPI 文件
}

app.MapControllers();
app.Run();
```

差別在於 AddSwaggerGen() 是用 Swashbuckle (第三方套件)，而 AddOpenApi() → .NET 8 官方內建，預設就支援 OpenAPI，本質上都是產生 OpenAPI 文件，只是方式不同。

⚠️ 如果你同時加了 AddSwaggerGen() 和 AddOpenApi()，那麼系統就會輸出兩份 OpenAPI 文件

- /swagger/v1/swagger.json
- /openapi/v1.json

其實技術上是「可以並存」，但通常沒必要。

<br>
<br>

## 🌿 launchsettings.json

設定檔
```json
"profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "http://localhost:5188",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "https": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "https://localhost:7258;http://localhost:5188",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "IIS Express": {
      "commandName": "IISExpress",
      "launchBrowser": true,
      "launchUrl": "swagger",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
```

<br>
<br>

## 🌿 為什麼需要 OpenAPI？

它是「API 的標準格式」。不管你用什麼語言（C#、Java、Python、Node.js…），只要輸出一份 OpenAPI 文件，其他工具或團隊就能直接使用，前端框架 (TypeScript, Angular, React)、測試工具 (Postman, Insomnia)、自動化 SDK 產生器 (C#, TypeScript, Java)

有了 OpenAPI，就能讓很多流程自動化

- 客戶端 SDK → 自動產生 C#、TypeScript、Java 的 API 呼叫程式碼
- 伺服端樣板 → 自動產生 Controller / Service 骨架
- 文件網站 → 自動產生互動式 API 文件頁面 (Swagger UI)

這樣的好處是，後端修改 API 文件自動更新、前端重新產生 SDK 自動同步

<br>
<br>

## 🌿 Swashbuckle 和 NSwag

<br>

### 🔹 Swashbuckle

來源是社群套件，ASP.NET Core 官方文件推薦使用，NuGet 套件：Swashbuckle.AspNetCore
主要用途為自動產生 OpenAPI (/swagger/v1/swagger.json)、自動掛 Swagger UI (/swagger)、支援自訂 Schema、OperationFilter、文件版本化

特色：

- 專注於 API → 文件（從程式碼產生文件）
- ASP.NET Core 社群使用最廣，教學/範例最多
- 功能完整，支援多種安全性（JWT、OAuth2）

👉 最常見的選擇，如果只需要產生文件與 UI，Swashbuckle 就足夠。


**自訂 Schema : 改模型描述（例：隱藏欄位、修改格式）**
用途：修改 Swagger 產生的模型描述（Schema）。
情境：你有一個敏感欄位，不想出現在 API 文件裡。

```csharp
public class User
{
    public int Id { get; set; }
    public string UserName { get; set; }
    public string PasswordHash { get; set; } // 不想暴露
}
```

**SchemaFilter**
```csharp
using Microsoft.OpenApi.Models;
using Swashbuckle.AspNetCore.SwaggerGen;

public class HidePasswordSchemaFilter : ISchemaFilter
{
    public void Apply(OpenApiSchema schema, SchemaFilterContext context)
    {
        if (context.Type == typeof(User))
        {
            schema.Properties.Remove("passwordHash");
        }
    }
}

builder.Services.AddSwaggerGen(c =>
{
    c.SchemaFilter<HidePasswordSchemaFilter>();
});
```

**OperationFilter 改 API 操作描述（例：全域加 Header、修改參數描述）**

用途：修改 API 動作 (Operation) 的 Swagger 描述。
情境：幫 API 加上 全域的 Header 說明，例如「必須帶 X-Request-Id」。
```csharp
public class AddRequestIdHeaderFilter : IOperationFilter
{
    public void Apply(OpenApiOperation operation, OperationFilterContext context)
    {
        operation.Parameters.Add(new OpenApiParameter
        {
            Name = "X-Request-Id",
            In = ParameterLocation.Header,
            Required = true,
            Description = "每個請求必須帶唯一 ID",
            Schema = new OpenApiSchema { Type = "string" }
        });
    }
}
builder.Services.AddSwaggerGen(c =>
{
    c.OperationFilter<AddRequestIdHeaderFilter>();
});
```

**文件版本化 (API Versioning) 支援多 API 版本，Swagger UI 分組顯示**
用途：讓同一個 API 系統支援多個版本（例如 v1, v2），並在 Swagger 裡分開顯示。
情境：Todo API v1 只回傳標題，Todo API v2 回傳標題 + 完成狀態。

```csharp
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
[ApiVersion("1.0")]
[ApiVersion("2.0")]
public class TodoController : ControllerBase
{
    [HttpGet, MapToApiVersion("1.0")]
    public IActionResult GetV1() => Ok(new[] { new { Title = "Buy milk" } });

    [HttpGet, MapToApiVersion("2.0")]
    public IActionResult GetV2() => Ok(new[] { new { Title = "Buy milk", IsDone = false } });
}
```
註冊 API 版本化 + Swagger
```csharp
builder.Services.AddApiVersioning();
builder.Services.AddVersionedApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV"; // v1, v2
    options.SubstituteApiVersionInUrl = true;
});

builder.Services.AddSwaggerGen(options =>
{
    var provider = builder.Services.BuildServiceProvider()
        .GetRequiredService<IApiVersionDescriptionProvider>();

    foreach (var desc in provider.ApiVersionDescriptions)
    {
        options.SwaggerDoc(desc.GroupName, new OpenApiInfo
        {
            Title = $"Todo API {desc.ApiVersion}",
            Version = desc.ApiVersion.ToString()
        });
    }
});
```

Swagger UI 會有兩個文件：v1 和 v2。

GET /api/v1/todo → 只回傳 Title
GET /api/v2/todo → 回傳 Title + IsDone

<br>

### 🔹 NSwag

來源：由 Rico Suter 開發，功能比 Swashbuckle 更廣。NuGet 套件：NSwag.AspNetCore

主要用途：

- 跟 Swashbuckle 一樣能產生 OpenAPI 文件和 Swagger UI
- 額外支援 API Client 產生器：可以根據 OpenAPI 產生 C# Client 或 TypeScript Client
- 可整合到 MSBuild / CLI / NPM，適合 CI/CD

特色：

不只是文件，還能做 API SDK 生成，適合前後端分離專案，前端可直接用 TypeScript Client SDK 呼叫 API，有 GUI 工具 NSwagStudio，方便點選操作
如果專案需要「自動產生 API 客戶端」(C# 或 TypeScript)，NSwag 更適合。


**API SDK 生成的好處**

- 不用手刻 API 呼叫程式碼 → 減少重複工作。
- 型別安全 (Type-safe) → DTO 定義自動同步，避免「後端改了，前端還在用舊格式」。
- 開發更快 → API 文件就是程式碼的來源，契約一致。
- 支援多語言 → 一份 OpenAPI，可以生成 TypeScript、C#、Java、Python SDK。

如果你有 API 的 OpenAPI 文件 (swagger.json)，就能用 NSwag 或 OpenAPI Generator 自動產生 SDK。
✅ 生成後的 TypeScript Client 範例
```typescript
// 自動生成的 TodoClient (TypeScript)
export class TodoClient {
  constructor(private baseUrl: string) {}

  async getTodos(): Promise<TodoItem[]> {
    const response = await fetch(`${this.baseUrl}/api/todos`);
    return response.json();
  }

  async addTodo(item: CreateTodoDto): Promise<TodoItem> {
    const response = await fetch(`${this.baseUrl}/api/todos`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(item),
    });
    return response.json();
  }
}
```

前端呼叫就很乾淨：
```csharp
const client = new TodoClient("https://api.example.com");
const todos = await client.getTodos();
✅ 生成後的 C# Client 範例
public class TodoClient
{
    private readonly HttpClient _http;

    public TodoClient(HttpClient http)
    {
        _http = http;
    }

    public async Task<ICollection<TodoItem>> GetTodosAsync()
    {
        var response = await _http.GetAsync("/api/todos");
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<ICollection<TodoItem>>();
    }

    public async Task<TodoItem> AddTodoAsync(CreateTodoDto dto)
    {
        var response = await _http.PostAsJsonAsync("/api/todos", dto);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<TodoItem>();
    }
}
var client = new TodoClient(httpClient);
var todos = await client.GetTodosAsync();
```

<br>

## 🌿 保護 Swagger UI

呼叫 MapSwagger().RequireAuthorization 來保護 Swagger UI
```csharp
using System.Security.Claims;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

builder.Services.AddAuthorization();
builder.Services.AddAuthentication("Bearer").AddJwtBearer();

var app = builder.Build();

  if (app.Environment.IsDevelopment())
  {
    app.UseSwagger();
    app.UseSwaggerUI();
  }

app.UseHttpsRedirection();

var summaries = new[]
{
    "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
};

app.MapSwagger().RequireAuthorization();

app.MapGet("/", () => "Hello, World!");
app.MapGet("/secret", (ClaimsPrincipal user) => $"Hello {user.Identity?.Name}. My secret")
    .RequireAuthorization();

app.MapGet("/weatherforecast", () =>
{
    var forecast = Enumerable.Range(1, 5).Select(index =>
        new WeatherForecast
        (
            DateOnly.FromDateTime(DateTime.Now.AddDays(index)),
            Random.Shared.Next(-20, 55),
            summaries[Random.Shared.Next(summaries.Length)]
        ))
        .ToArray();
    return forecast;
})
.WithName("GetWeatherForecast")
.WithOpenApi();

app.Run();

internal record WeatherForecast(DateOnly Date, int TemperatureC, string? Summary)
{
    public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);
}
```

在上述程式碼中，/weatherforecast 端點不需要授權，但 Swagger 端點需要。

```bash
curl -i -H "Authorization: Bearer {token}" https://localhost:{port}/swagger/v1/swagger.json
```

<br>
<br>

## 🌿 結語

把 Swagger 想像成一個在你身邊的「API 小精靈」✨。它會默默觀察你的 Controllers，幫你生成一份 OpenAPI 文件；它會把這份文件變成一個漂亮的 UI，讓前後端、QA、PM 甚至客戶，都能一眼看懂；如果你需要，它還能幫你生出前端或後端的 SDK，讓大家不用再重複造輪子。