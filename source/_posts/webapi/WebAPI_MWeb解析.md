---
title: WebAPI - MWeb
date: 2025-09-26 17:07:34
categories: WebAPI
top_img: https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
tags:
    - WebAPI
toc:
toc_number:
comments :
---

![Image](https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true)

## 啟動點流程


- IIS 啟動網站應用程式池 (Application Pool)

當有請求進來時，IIS 會把 Request 交給 ASP.NET Runtime (w3wp.exe)。

#### w3wp.exe 是什麼？

w3wp.exe = IIS 的 Worker Process（工作進程）

IIS 6/7/8/10 都會用 Windows Process Activation Service (WAS) 啟動 w3wp.exe

每個 Application Pool 會有自己的 w3wp.exe

網站的程式碼、ASP.NET Runtime、Http Pipeline 都是在 w3wp.exe 裡面運作

👉 w3wp.exe 是容器（Container）──不是 Runtime 本身。


#### ASP.NET Runtime 是什麼

ASP.NET Runtime 指的是：

Page Handler Factory (System.Web)
HTTP Pipeline（Modules + Handlers）
Request Processing Pipeline（BeginRequest、AuthorizeRequest、ResolveRequestCache…）
WebForms / MVC Pipeline
ViewState、Session、Membership Provider 等
Compilation System（ASPX 編譯器）
C# Compiler Provider
Configuration System（Web.config）
Caching（HttpRuntime.Cache）

這些邏輯與類別集合起來，稱為 ASP.NET Runtime（CLR 上的 ASP.NET 執行環境），它是一組 .NET Framework 組件，不是一個 exe

```plaintext
IIS → 啟動 AppPool → 生出 w3wp.exe
           ↓
         w3wp.exe → 載入 CLR
           ↓
         CLR → 載入 ASP.NET Runtime（System.Web.*）
           ↓
         ASP.NET Runtime → 執行你的程式碼
```

- IIS = 節點負責人
- w3wp.exe = 容器（像 K8s Pod）
- CLR = .NET 執行引擎（像 JVM）
- ASP.NET Runtime = 框架（像 Spring MVC）
- Your Code = 實作 API / 網頁的部份


w3wp.exe 是 ASP.NET Runtime 的“宿主”；Runtime 是 System.Web DLL 所組成的執行框架，兩者不是同一個東西

IIS → WAS → AppPool → w3wp.exe → CLR → ASP.NET Pipeline → Handler → Response

<br>
<br>

#### CLR 就是 runtime嗎

👉 CLR（Common Language Runtime）就是 .NET Framework／.NET Core 的 Runtime。
但它並不是整個「應用程式執行環境」的全部，它是 核心執行引擎（Execution Engine）。


✅ CLR = Runtime（執行引擎）

CLR 是 .NET 的執行引擎，負責所有程式運行時的關鍵功能：

- IL → 機器碼（JIT）
- Memory 管理（GC）
- 例外處理（Exception）
- 執行緒（Thread Pool）
- 型別安全（Type Safety）
- 安全性（CAS）
- 呼叫 Managed / Unmanaged 程式碼（P/Invoke）
- AppDomain（舊版 .NET Framework）



這些都是 Runtime 才會做的事。

所以從正式定義來說：

CLR = .NET Runtime（執行時期系統）。

❗但注意：

在 ASP.NET / ASP.NET Core 的語境中，「Runtime」常常不只指 CLR。

也會包含：

- ASP.NET Runtime（System.Web 管線）
- Http Pipeline（Handlers, Modules, Middleware）
- Request Processing Framework
- Page/MVC/Web API 的整個解析、執行模型

因此

✔ CLR = .NET 的底層 Runtime
✘ CLR ≠ ASP.NET Runtime（這是更高一層的 Web Framework）

```plaintext
IIS / Kestrel（宿主 Host）
        ↓
   w3wp.exe / dotnet.exe（Process）
        ↓
        CLR（.NET Runtime / Execution Engine）
        ↓
ASP.NET Runtime（System.Web / Middleware Pipeline）
        ↓
Your Code（Controllers, Razor, Handlers）
```


CLR 是在 Process 裡被載入的 Runtime
ASP.NET Runtime 是建立在 CLR 之上的 Web Framework 功能
w3wp.exe / dotnet.exe 則是 Runtime 的宿主

✔ CLR 是 .NET 的底層執行環境（Execution Engine）。
✔ “Runtime” 不是時間，而是指「程式在執行時所依賴的完整環境」。
✔ 叫 Runtime 是源自 Computer Science 對 Compile-time / Run-time 的分類。
✔ 所以不會叫 RunEngine（太工程師，但不科學標準）。




- ASP.NET Runtime 建立 AppDomain

這個時候，它會去讀 web.config 的設定，載入模組與處理程序。

- 載入 Global.asax

如果你的專案裡有 Global.asax，ASP.NET Runtime 會在啟動時執行它裡面的 Application_Start()。

所以 Global.asax.cs 是開發者能控制的「起始點」。

- 進入 HTTP Request 處理流程

請求會經過 HTTP Pipeline（HttpModules → HttpHandlers）。最後才進到 Web Forms Page、MVC Controller、或 Web API Controller。


真正的程式啟動點，在 IIS / ASP.NET Runtime 裡面（你專案裡看不到 Main()）。
開發者能寫邏輯的啟動點：

Global.asax.cs → Application_Start()（應用程式初始化）
App_Start 資料夾裡的檔案（MVC/Web API 常見）
    RouteConfig.cs（註冊路由）
    FilterConfig.cs（註冊全域 Filter）
    BundleConfig.cs（註冊 Script/CSS Bundling）
    WebApiConfig.cs（Web API 路由設定）


## 沒有管線

在 ASP.NET Framework (舊版) 裡沒有 Middleware 概念
請求管線是由 IIS + ASP.NET Runtime 定義好的固定流程，主要靠 HttpModules 和 HttpHandlers 來處理
你可以在 Global.asax 或 web.config 裡設定哪些模組要參與，順序幾乎是固定的 👉 所以在舊版 ASP.NET 專案裡，你看不到 UseXxx，因為那是 Core 的 Middleware 概念


ASP.NET Framework 的 Request Pipeline

IIS 接收請求，交給 ASP.NET Runtime。
依據 web.config，請求會先經過一系列 HttpModules（像攔截器）。例如：驗證、Session、UrlRoutingModule。
請求會被分派到對應的 HttpHandler（像 Controller/ WebForm Page）。

Web API → HttpControllerHandler
MVC → MvcHandler
WebForm → PageHandler

最後產生 Response 回傳給瀏覽器。


HttpModule = ASP.NET Framework 的 Middleware

在 BeginRequest、AuthenticateRequest、EndRequest 等階段注入邏輯。

但順序主要由 ASP.NET Pipeline 決定，沒有像 Core 那樣自由。

HttpHandler = 負責真正處理請求的人

在 Web API 裡，會把請求交給 Controller。

在 MVC 裡，會交給 ActionResult。

在 WebForm 裡，會交給 .aspx Page。


## ASP.NET Framework (HttpModules + HttpHandlers) 的缺點

管線順序固定、難以客製化

HttpModules 的執行順序是由 ASP.NET Runtime 決定（Authenticate → Authorize → ResolveHandler → AcquireRequestState…）。

你能做的只是「插入」某些事件，不容易控制整個流程。

想要改順序（例如先壓縮再驗證）幾乎不可能。

高度耦合 IIS / System.Web

ASP.NET Framework 跑在 IIS 上，強烈依賴 System.Web.dll。

專案要部署到非 Windows 環境（Linux、Docker、雲端）非常困難。

效能與資源消耗大

System.Web 很龐大，哪怕只是回傳一個簡單的 JSON，也會載入整個 ASP.NET Runtime。

記憶體開銷大、啟動慢。

可測試性差

HttpModules/Handlers 與 Runtime 綁死，單元測試很難模擬，通常只能在 IIS Express 下測。

多技術並存難整合

WebForms、MVC、Web API 各自有自己的處理方式（不同 Handler）。

導致開發者要學不同模型，維護困難。


## ASP.NET Core (Middleware Pipeline) 的優勢

完全可組合、順序自由

你可以決定 UseRouting → UseCors → UseAuthentication → UseAuthorization → UseEndpoints 的順序。

自己寫 Middleware 也只是一個 RequestDelegate，比 Module/Handler 輕量很多。

跨平台、跨伺服器

ASP.NET Core 不再依賴 IIS，可以跑在 Kestrel、Nginx、Docker、Linux。

系統核心精簡，不需要 System.Web。

效能極大提升

Pipeline 是單純的委派串接，沒有一大堆「事件模型」開銷。

官方數據顯示 ASP.NET Core 比 ASP.NET Framework 快好幾倍。

統一的請求處理模型

MVC、Web API、Razor Pages 全部整合成「Endpoint Routing」。

不再需要分 MvcHandler、HttpControllerHandler。

可測試性高

Middleware 只是 C# 方法，完全可以在單元測試裡模擬 Request/Response。

開發流程更符合現代 DevOps / TDD。



## 比較可調換順序的差異


你可以根據：

流量特性（高併發先擋垃圾流量）
金流 API 的特殊需求（前置 authentication）
促購中心特例（解碼 promotion token）
前端需求（CORS 必須前置）
觀測 (OpenTelemetry)
監控（APM）
灰度發布（A/B Test）
分環境差異（UAT/Prod）
來調整 Pipeline
這是舊版 ASP.NET 永遠無法做到的


🔸（1）Correlation ID 無法放在最前面

你只能用 HttpModule 套在 BeginRequest，但你沒辦法

在整個 Pipeline 最前面插入

在下游 Middleware 前插入

包裹後續邏輯（next）

結果：

❌ 异常時抓不到完整 CorrelationId
❌ 無法實現 OpenTelemetry 標準
❌ 無法做到像 Core 那種前後包裝 Log（Request + Response）

🔸（2）Rate Limit 無法前置

舊 Pipeline 無法做到：

RateLimit → Logging → Auth → Routing


你只能掛 Module，順序無法保證

因此 RateLimit 必須等到後面才執行，造成：

不必要的 CPU 浪費
不必要的 Routing 計算
不必要的 Auth 驗證
被 DDoS 打爆

在電商 Checkout 這種 RPS 高達 2000 的場景下，這是致命的

🔸（3）CORS 必須在 Routing 之前

在 Core 正確順序：

UseCors
UseRouting
UseEndpoints


在舊版：

❌ 你只能用 HttpModule
❌ 但不知道 Routing 何時做
❌ 有些狀況（如 MVC Attribute Routing）會讓 CORS 失效
❌ 無法精準比對 OPTIONS request
❌ 無法為整個 API 攔截跨域前置檢查

🔸（4）Exception Handling 無法最外層包住

舊版沒有：

try {
   await next();
} catch...


你永遠無法寫出全域 exception middleware。

結果：

很多 500 是 IIS 回的
很多 Error 重導到 defaultErrorPage
很多錯誤無法 Log 到 APM
很多錯誤無法附上 CorrelationId
對大型 API 系統非常痛苦。



## cookie


