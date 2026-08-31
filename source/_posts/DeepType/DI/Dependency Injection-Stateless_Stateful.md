---
title: Dependency Injection - State
date: 2025-11-06 10:43:05
categories: 注入之森
top_img: https://i.imgur.com/xAbpKgd.png
cover : https://i.imgur.com/xAbpKgd.png
tags:
    - Dependency Injection
toc:
toc_number:
comments :
---

{% tabs Factory%}


![state](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/State/state_landing.png)


<!-- tab .NET DI 中的 Service Lifetime（Transient / Scoped / Singleton）-->


在 .NET DI 裡，我們註冊服務時通常會用三種生命週期

| 生命週期  | 建立時機                                 | 何時被回收         | 常見用途                                     |
| --------- | ---------------------------------------- | ------------------ | -------------------------------------------- |
| Transient | 每次注入時都會 new 一個新的實例          | 呼叫結束後可回收   | 短期任務、**Request 專屬狀態的物件**         |
| Scoped    | 每個 Request (或每個範圍) 共用同一個實例 | Request 結束時回收 | Web Request、**需在同一 Request 內共享狀態** |
| Singleton | 第一次解析時建立一次，全程共用           | 應用程式結束時回收 | 共用服務、**可被全域共用的狀態或無狀態服務** |


![three_lifetime](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/State/three_lifetime.png)


選 Lifetime 的本質是「這個物件裡的狀態，允不允許被誰共用、共用多久」


![state](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/State/key_is_state.png)


| 狀態性質                            | 適合的 Lifetime    | 原因                                                  |
| ----------------------------------- | ------------------ | ----------------------------------------------------- |
| Request 專屬狀態（User、DbContext） | Transient / Scoped | 狀態只屬於單一 Request，不可跨 Request 共用           |
| Request 內需共享狀態                | Scoped             | 同一 Request 內需維持一致狀態（交易、快取、流程資料） |
| 全域可共用狀態或無狀態              | Singleton          | 狀態在整個應用程式期間皆可安全共用                    |



所以在決定服務的聲明周期前，要先確認這個 Service 有沒有「狀態」? 有沒有欄位會被改？有沒有依賴使用者、Request、流程資料？

接著我們可以確認這個狀態屬於誰，只屬於某一次 Request？同一個 Request 裡要不要共用？還是全應用程式都可以共用？

像是全域可共用狀態例子包含 Feature Flag、設定快取、全域計數器、Policy / 規則集，而真正危險的是使用者相關資料、計算中間結果、流程暫存資料


![state](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/State/state_decision.png)


<!-- endtab -->

<!-- tab 為什麼「無 Request 專屬狀態」的服務通常用 Singleton？-->

當一個服務的行為不依賴任何使用者、請求或流程狀態時，多人共用同一個實例並不會改變結果，那就沒有理由幫每個人準備一份，因為「共用不會造成錯誤」

先看服務會不會記住任何 Request 相關資訊，他應該沒有 User、沒有 Request Id、沒有流程中間狀態，每次方法呼叫都是「純行為」，輸入一樣，輸出就一樣，不會因為前一次呼叫留下影響，並且要思考多執行緒同時呼叫會不會互相影響，不能共用可變資料、沒有寫入共享欄位，因此這是在「邏輯安全」的前提下，順勢讓整個應用程式共用同一個實例，效能只是附帶好處


![singleton](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/State/singleton_no_state_best.png)


<!-- endtab -->

<!-- tab Singleton - EntityHelper-->


```csharp
public class EntityHelper : IEntityHelper
{
    private readonly ILogger<EntityHelper> _logger;
    public EntityHelper(ILogger<EntityHelper> logger)
    {
        _logger = logger;
    }


    public SqlParameter[] BuildSqlParameters(MyEntity entity)
    {
        _logger.LogDebug("Building SQL parameters for entity {Id}", entity.Id);
        return new[]
        {
        new SqlParameter("@Id", entity.Id),
        new SqlParameter("@Name", entity.Name ?? (object)DBNull.Value)
        };
    }
}
```

以 EntityHelper 為例，實際在做的事情非常單純，他透過 DI 取得 logger，_logger 只是用來輸出訊息，不存使用者、不存 Request、不存流程狀態，接收一個 entity 當作輸入，所有資料都來自方法參數，不依賴物件內部欄位的歷史值，組合 SQL Parameter，單純把 entity 的欄位轉成 SqlParameter，每次呼叫都是獨立計算，方法結束即結束，沒有任何狀態被留下來，下一次呼叫完全不受影響，從頭到尾，這個 class 只做轉換，不保存任何東西



`ILoggerFactory` 是 Singleton 而 `ILogger<T>` 只是輕量 wrapper，真正的 logging pipeline 是共用且 thread-safe 的，logger 不會記住「這一次是誰呼叫我」，所以把 `ILogger<T>` 注入 Singleton 是合理、安全也是 ASP.NET Core 預期的用法


如果錯用 Transient，就是一直創造功能完全一樣，用完就丟的殼，每次注入都 new 一個、每個實例內部什麼都沒存、下一次呼叫也不會重用，物件建立次數增加、GC 壓力變大卻完全沒有換到任何好處


![helper](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/State/helper_logger.png)


<!-- endtab -->


<!-- tab Transient - RequestMessageLoggingDelegatingHandler-->

它是整個 HttpClient 請求管線 中的一個「攔截器 (Interceptor)」，扮演著「攔截 HTTP Request，記錄請求內容 (Logging)」的角色，`IHttpClientFactory` 會替 `HttpMessageHandler / DelegatingHandler` 建一個自己的 DI scope，而這個 scope 跟 ASP.NET Core 的 incoming request scope 是分開的。官方文件也特別說 **handler scope 可以活得比 application request scope 更久**，所以同一個 handler 與它注入的 scoped dependencies，可能被多個 incoming requests 重用


HttpClient 本身的生命週期，在現代 .NET（尤其是用 `IHttpClientFactory`）中 HttpClient 實例 是被 DI 管理的，它的內部會維護一個 Handler Chain（處理鏈），這條 Handler 鏈在建立 HttpClient 時就會被「組合」起來，也就是說，每次透過 `IHttpClientFactory.CreateClient() `建立新的 HttpClient 時，會根據你的註冊設定：`.AddHttpMessageHandler<RawResponseMessageLoggingDelegatingHandler>()` 來 new 一次對應的 Handler 實例

`HttpMessageHandler` 在 `IHttpClientFactory` 中是以 Transient 註冊，但其生命週期與 HttpClient 綁定，每次建立 HttpClient 時，容器會建立一條全新的 Handler Chain，因此其語意更接近「Client-Scoped」，而非 Request-Scoped，一個 Web Request 內，可能會建立多個 HttpClient、也可能完全沒有 HttpClient，HttpClient 的建立不一定跟 Web Request 同步

```bash
YourService
   ↓
HttpClient
   ↓
RequestMessageLoggingDelegatingHandler  👈（這個 class）
   ↓
HttpClientHandler (真正送出請求)
   ↓
外部 API Server
```


細看一下，其實他就是攔截請求來執行 `_requestBodyLoggingTransformer` 把 Request Body 轉成可記錄的文字後 log 出來
```csharp
protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)

if (this._requestBodyLoggingTransformer != null)
{
    string message = await this._requestBodyLoggingTransformer(request);

    if (string.IsNullOrWhiteSpace(message) == false)
    {
        this._logger.LogInformation($"HTTP Request Body: {message}");
    }
}

///呼叫下一層 Handler
return await base.SendAsync(request, cancellationToken);
```

`base.SendAsync` 會把控制權傳下去給下一層 Handler，`ResponseMessageLoggingDelegatingHandler`或最底層的 HttpClientHandler（真正送出網路請求），如果沒呼叫這行，HTTP 請求就永遠送不出去

使用時
```csharp
services.AddRefitClient<IQFPayHttpClient>()
    .AddHttpMessageHandler<RawResponseMessageLoggingDelegatingHandler>()
    .AddTransientHttpErrorPolicy(builder => builder.WaitAndRetryAsync(retryCount, (_) => TimeSpan.FromMilliseconds(retryInterval)));
```


如果把這個 Handler 設成 Singleton，會造成同一個 Handler 實例可能被多個 HttpClient 共用，甚至被多個 Web Request 同時使用，結果出現 Log 交錯、Trace / Correlation ID 錯亂、Body 讀取衝突、Thread-unsafe 行為


![state](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/State/handler.png)


<!-- endtab -->

<!-- tab PromotionEngineService 為何用 Scoped-->


PromotionEngineService 通常涉及整個 Request 的業務流程（例如計算折扣、檢查資格、更新資料庫），尤其內部會依賴 DbContext（EF Core context）、交易控制（TransactionScope）、狀態快取（例如同一個 Request 內要共用一份會員資料或促銷規則），因此它需要在同一個 Request 的執行過程中「保持資料一致性」，也需要確保所使用的 DbContext、Repository 等 Scoped 物件與它 共用同一個 Scope


![promotion](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/State/promotionengine.png)


<!-- endtab -->

<!-- tab Scoped 存在的目的-->

AddScoped 的設計目的，是為了讓物件能在「同一個 Request」期間維持一致狀態，並與其他 Scoped 相依物件（如 DbContext）共用生命週期，所以 Scoped 是為了「共用狀態」而存在的。如果商業邏輯沒有交易脈絡（例如 DbContext、TransactionScope）、Request 專屬的資料（例如當前會員、Header、TraceId）、需要共用的暫存狀態，那它基本上是「Stateless」（無狀態）服務也就不需要 Scoped，用 Transient 反而更乾淨、更節省資源


![scoped](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/State/scoped.png)


## DbContext 為甚麼 scoped


DbContext 是一個「工作單元 (Unit of Work)」，管理資料追蹤（Tracking）、快取（Change Tracker），並與資料庫保持「開啟連線」與「交易範圍」，因此，它不是單純的工具物件，而是一個 狀態性 (stateful) 的資源，若改為 Transient，則會造成 OrderRepository 和 PaymentRepository 都是使用同 DB 卻用的是不同的 DbContext，各自追蹤不同的物件快取，無法在同一個 Transaction 下完成一致的 SaveChanges。結果可能會交易不一致（Order 可能成功、Payment 失敗）、並且因為重複建立連線造成效能下降，記憶體壓力增加（多個 Context Tracking）變得難以控制交易範圍

在假設使用 Singleton 時，則是災難級錯誤，整個應用程式只有一個 DbContext 實例，所有 Request、所有執行緒都共用這一個，因為 DbContext 不是 thread-safe，這會造成 Thread-Safety 問題，像是多個 Request 同時存取 _context.Orders.Add()，會互相干擾

```BASH
InvalidOperationException: A second operation started on this context before a previous operation completed.
```

其他還有因為 ChangeTracker 永不清除，導致 Entity 追蹤會越來越多，記憶體不斷增加，SaveChanges 越來越慢，Tracking 資料污染（舊資料混進新資料），甚至交易與連線死鎖，不同 Request 嘗試同時使用同一個開啟中的連線，造成 Transaction 錯亂或 Connection Pool 枯竭


![Dbcontext](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/State/dncontext.png)

![Dbcontext](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/State/dbcontext_fail.png)


<!-- endtab -->

<!-- tab Interface 跟 DI 關聯-->

Interface 的目的是讓「使用者只關心能做什麼，不必知道怎麼做到」，而 DI 只是讓這個分工可以被系統化管理


**Interface** 先定義「我需要什麼能力」、有哪些方法、行為的語意是什麼，不包含任何實作細節，而**實作類別**負責「怎麼做到」、用哪個 SDK、呼叫哪個 API、怎麼處理錯誤與例外
**DI / IoC 容器**負責「在什麼時候給你哪一個實作」，他會在啟動時註冊，建立物件時注入並且幫你管理生命週期，使用端只依賴 Interface，不需要知道 class 名稱、不需要自己 new、不需要管實作怎麼換



| 概念              | 說明               | 角色         |
| ----------------- | ------------------ | ------------ |
| **Interface**     | 定義「行為的契約」 | 規格、抽象層 |
| **DI / IoC 容器** | 負責「提供實作」   | 管理者、工廠 |


## 何時抽換

- 單元測試：用 Fake / Mock
- 切換外部依賴：換新郵件供應商、支付廠商
- 系統升級：舊版 API 改新版 SDK


![interface](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/State/interface.png)


<!-- endtab -->

<!-- tab Registering & Resolving-->

Registering 是先把「有哪些角色可以用」定義下來，Resolving 則是在需要時，把正確的人派上場。寫的是「我需要什麼」，容器負責的是「現在給你誰」

![Registering & Resolving](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/State/registration_resolution.png)


## Registering Components (註冊服務)

把介面與實作關係告訴 IoC 容器

Autofac
```csharp
var builder = new ContainerBuilder();
builder.RegisterType<OrderService>().As<IOrderService>();
builder.RegisterType<EmailSender>().As<IEmailSender>().SingleInstance();
var container = builder.Build();
```
<br>

Microsoft DI
```csharp
services.AddScoped<IOrderService, OrderService>();
services.AddTransient<IEmailSender, EmailSender>();
```


這一步的語意是

- IOrderService → 用 OrderService
- IEmailSender → 用 EmailSender

Lifetime 決定「用幾次」


## Resolving Services (解析服務)


當程式需要該服務時，容器自動「注入」實作物件。

```csharp
public class PromotionEngineService
{
    private readonly IEmailSender _emailSender;

    public PromotionEngineService(IEmailSender emailSender)
    {
        _emailSender = emailSender;
    }

    public void NotifyReward() => _emailSender.Send("user@91app.com", "You got a reward!");
}
```

沒有 new EmailSender，只是對容器說「我需要一個 IEmailSender」，容器幫你拿對的實作進來

<!-- endtab -->

<!-- tab 靜態物件的問題-->

Static 特色是全域，但問題是它把「控制權」鎖死在 CLR，讓系統無法管理(沒辦法注入依賴（例如 Logger、HttpClientFactory）)、無法替換、也無法測試(單元測試難以替換)

使用 Singleton 容器會限制生命週期的混用，因為 DI 會阻止我們 Singleton 注入 Scoped、Singleton 注入 Request 狀態，它避免了「把短命狀態塞進長命物件」

| 場景                                 | 適合用 Static | 適合用 Singleton |
| ------------------------------------ | ------------- | ---------------- |
| 只包含純函數、無狀態邏輯             | ✅             | ❌                |
| 需要依賴其他服務 (如 Logger、Config) | ❌             | ✅                |
| 要支援單元測試 / Mock                | ❌             | ✅                |
| 不需釋放資源                         | ✅             | ✅                |
| 涉及資源管理（Db、HttpClient）       | ❌             | ✅                |
| 多執行緒存取                         | ⚠️ 需自行保護  | ✅ 容器可管理     |


| 差異重點   | Static       | Singleton (DI)               |
| ---------- | ------------ | ---------------------------- |
| 控制者     | CLR          | IoC 容器                     |
| 建立時機   | 程式載入     | 容器第一次解析               |
| 可注入依賴 | ❌            | ✅                            |
| 可 Dispose | ❌            | ✅                            |
| 可替換     | ❌            | ✅                            |
| 可測試性   | 差           | 好                           |
| 使用場合   | 無狀態工具類 | 有狀態、需管理資源的共用服務 |


![static](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/State/static_singleton.png)


<!-- endtab -->


{% endtabs %}