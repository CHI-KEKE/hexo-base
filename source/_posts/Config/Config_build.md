---
title: Host Configuration
date: 2025-11-13 15:04:01
categories: Config
top_img: https://github.com/CHI-KEKE/pics/blob/main/Config/gear.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Config/gear.png?raw=true
tags:
    - Config
toc:
toc_number:
comments :
---


## ConsoleApp 的啟動流程

Host - 是整個 App 的根
hostingContext - 目前應用程式主機 (Host) 的執行環境上下文, 帶著關於目前執行環境的所有資訊

- `hostContext.Configuration` - 拿出目前 config
- `hostingContext.HostingEnvironment` - 拿出目前環境


#### ConfigureAppConfiguration - 讀設定檔

- `config.AddJsonFile("appsettings.json", optional: false, reloadOnChange: true);`

#### ConfigureServices - 註冊服務

- `hostContext`, `IServiceCollection`
- `services.AddDbContext<WebStoreDbContext>(options`
  - `options` = `DbContextOptionsBuilder`, EF Core 啟動時用來決定連線方式、行為設定的物件

- `configuration.GetConnectionString`, 這是IConfiguration 的 Extension Method, 只是 `configuration?.GetSection("ConnectionStrings")[name]`

#### ConfigureLogging - 日誌


## ConsoleApp - Config 使用與注入的設定

- `var configuration = services.GetRequiredService<IConfiguration>();`

#### IConfiguration 是怎麼進去的？

在 `Host.CreateDefaultBuilder(args)` 這行裡，框架會自動執行 `services.AddSingleton<IConfiguration>(configuration);`，也就是把設定物件注入進 DI 容器中。因此當你有一個 IServiceProvider（例如在你的 PollyRetryTest() 裡傳進來的 services），你就可以隨時從中取出 IConfiguration。請務必從框架拿 IServiceProvider


## WebApi 的啟動流程

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
var configurationManager = builder.Configuration;
configurationManager.AddJsonFile("settings.json", optional: true, reloadOnChange: true);
```

`WebApplication.CreateBuilder` 這行背後做的事情包含建立一個 HostApplicationBuilder（也就是 Generic Host 的新版包裝），會幫你自動加上

- `IConfiguration`（讀取 `appsettings.json`、環境變數、命令列）
- `IServiceCollection`（註冊服務）
- `ILoggerFactory`（日誌）
- Kestrel Web 伺服器設定
- `IWebHostEnvironment`（告訴你目前是 Development / Production）

建立一個 `WebApplicationBuilder` 物件，內含 builder.Services、builder.Configuration、builder.Logging。


## 建立與執行 Web 應用


```csharp
var app = builder.Build();
app.MapGet("/", () => "Hello World!");
app.Run();
```

- 把 Host 組建起來
- 啟動 HTTP Server
- 進入 Web 事件循環（監聽請求直到結束）

#### ASP.NET Core 中 AddXXX 與 UseXXX 的差異

AddXXX() 是在 設定階段（ConfigureServices） 使用的，意思是「把這個功能或服務註冊進 DI 容器（Service Container），讓以後可以被注入使用」。

UseXXX() 是在 執行階段（Configure / Middleware Pipeline） 使用的，意思是「在 HTTP 請求進入時，要不要經過這個中介層（Middleware）來處理」。


















## 這是「應用的啟動流程」



#### ConsoleApp

Host - 是整個 App 的根
hostingContext - 目前應用程式主機 (Host) 的執行環境上下文, 帶著關於目前執行環境的所有資訊
    - hostContext.Configuration - 拿出目前 config
    - hostingContext.HostingEnvironment - 拿出目前環境
ConfigureAppConfiguration - 讀設定檔
    - config.AddJsonFile("appsettings.json", optional: false, reloadOnChange: true);
ConfigureServices - 註冊服務
    - hostContext, IServiceCollection
    - services.AddDbContext<WebStoreDbContext>(options
      - options = DbContextOptionsBuilder,EF Core 啟動時用來決定連線方式、行為設定的物件
      - configuration.GetConnectionString, 這是IConfiguration 的 Extension Method, 只是 configuration?.GetSection("ConnectionStrings")[name]
ConfigureLogging - 日誌

使用

- var configuration = services.GetRequiredService<IConfiguration>();
2️⃣ IConfiguration 是怎麼進去的？

在 Host.CreateDefaultBuilder(args) 這行裡，
框架會自動執行：

services.AddSingleton<IConfiguration>(configuration);


也就是把設定物件注入進 DI 容器中。

因此當你有一個 IServiceProvider（例如在你的 PollyRetryTest() 裡傳進來的 services），
你就可以隨時從中取出 IConfiguration。

務必從框架拿 IServiceProvider



## WebApi

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
var configurationManager = builder.Configuration;
configurationManager.AddJsonFile("settings.json", optional: true, reloadOnChange: true);


這行背後做的事情包含：

建立一個 HostApplicationBuilder（也就是 Generic Host 的新版包裝）；

幫你自動加上：

IConfiguration（讀取 appsettings.json、環境變數、命令列）

IServiceCollection（註冊服務）

ILoggerFactory（日誌）

Kestrel Web 伺服器設定

IWebHostEnvironment（告訴你目前是 Development / Production）

建立一個 WebApplicationBuilder 物件，內含 builder.Services、builder.Configuration、builder.Logging。

var app = builder.Build();
app.MapGet("/", () => "Hello World!");
app.Run();

它會：

把 Host 組建起來；

啟動 HTTP Server；

進入 Web 事件循環（監聽請求直到結束）。


Console App 通常不需要：

Kestrel Web Server；

Web Host 環境；

中介軟體 (Middleware)；

路由 (Routing)；
它只需要一個「通用主機 (Generic Host)」去管理：

背景服務（Hosted Services）

依賴注入（DI）

組態（Configuration）

日誌（Logging）




在 ASP.NET Core 裡：

AddXXX() 是在 設定階段（ConfigureServices） 使用的。
👉 意思是「把這個功能或服務註冊進 DI 容器（Service Container），讓以後可以被注入使用」。

UseXXX() 是在 執行階段（Configure / Middleware Pipeline） 使用的。
👉 意思是「在 HTTP 請求進入時，要不要經過這個中介層（Middleware）來處理」。


所以簡單說：

🧩 Add = 「準備好零件」
🧱 Use = 「把零件裝上去開始運作」



| 階段   | 方法         | 作用時間    | 用途               |
| ---- | ---------- | ------- | ---------------- |
| 啟動階段 | `AddXXX()` | 應用啟動時   | 把功能、設定、服務註冊進容器   |
| 執行階段 | `UseXXX()` | 每次請求進入時 | 組合中介管線處理 HTTP 請求 |
