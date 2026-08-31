---
title: Host
date: 2024-08-10 12:31:00
categories: Web
top_img: https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
tags:
    - Web
toc:
toc_number:
comments :
---

{% tabs Host%}

{% btn 'https://github.com/dotnet/runtime/blob/v5.0.0-rtm.20519.4/src/libraries/Microsoft.Extensions.Hosting/src/Host.cs',Source Code,far fa-hand-point-right %}

<!-- tab ASP.NET Core 的誕生-->

以前的 ASP.NET 只能跑在 Windows + IIS 的組合裡。換句話說，你想開發網站，就必須用 Windows Server，因為 ASP.NET 被「綁死」在那個環境。後來微軟想要讓它更自由，所以從頭開始重寫，這就是 ASP.NET Core 的誕生

它一開始用「ASP.NET vNext」這種暫時的名字登場，後來一度被叫做 ASP.NET 5，最後才定名為 ASP.NET Core。這裡的「Core」代表：

- 全新框架 → 不是 ASP.NET 4.x 的小改版，而是重頭設計。
- 精簡核心 → 把最必要的功能保留下來，刪掉複雜、不必要的包袱。
- 跨平台 → 不再侷限於 Windows，可以跑在 Linux、macOS，甚至容器（Docker）裡。
- 
ASP.NET Core 有 自我寄宿 (Self-Hosting) 的能力，也就是說網站不需要一定依靠 IIS，它本身就能啟動一個伺服器來跑網站。

甚至可以用最簡單的程式就跑一個網站
```csharp
var builder = WebApplication.CreateBuilder(args); // 建立一個網站應用程式
var app = builder.Build();

app.MapGet("/", () => "Hello ASP.NET Core!"); // 設定 Routing，當有人訪問 /，就回傳文字

app.Run(); // 啟動伺服器，直接可以對外服務
```



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Hosting/1_tocore.png)


<!-- endtab -->


<!-- tab ASP.NET  VS ASP.NET CORE-->

## ASP.NET（System.Web） 時代

![Image](https://i.imgur.com/dfs9vvQ.png)

在傳統 ASP.NET（System.Web） 時代，網站程式不是用自己的程式 process 在跑；它會被 IIS 啟動並載入到 IIS 的 working process（常見是 w3wp.exe）裡面執行。你看不到 Main() 或 app.Run()，因為沒有自我啟動；IIS 啟動 w3wp.exe 後，載入 CLR 與 System.Web 管線，再把事件丟給你的程式。

路徑大概是這樣

```bash
傳統 ASP.NET：Client → HTTP.SYS → IIS（w3wp.exe）→ System.Web Pipeline → 程式
```

IIS 負責聽 port（透過 Windows 的 HTTP.SYS），接到 HTTP 要求後才把請求交給載在同一個 w3wp.exe 內的 ASP.NET 管線處理。程式碼、生命週期（如 Application_Start / BeginRequest）、執行緒池、記憶體等，都受 IIS/應用程式集區（Application Pool）管理。

IIS 何時啟動/回收工作進程、是否回收、在什麼身分執行（App Pool Identity），程式只能配合。可以說網站「住」在 IIS 的房子裡，由 IIS 管吃管住；程式並不是自己開一間獨立店面。



![Image](https://i.imgur.com/Ru6rFrB.png)



## ASP.NET Core

ASP.NET Core 帶著自己的「生存工具包」Kestrel Web Server，可以不用依賴 IIS，自己就能對外提供服務。就像是一個年輕人有了自己的小套房不需要再跟父母（IIS）住在一起

![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Hosting/1_parent_and_self.png)


但小套房雖然能住，外面環境複雜，最好還是請「大樓警衛」幫忙把關。這個警衛就是 Reverse Proxy（反向代理伺服器），例如 Nginx、Apache 或 IIS，Kestrel 專注在「效能與基本功能」，但在安全與企業級功能上相對簡單。像是 TLS 憑證管理、Request Filtering、IP 白名單、Load Balancing，通常交給更成熟的 Web Server（Nginx、Apache、IIS）


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Hosting/1_reverse_proxy.png)




<!-- endtab -->


<!-- tab Host-->

在 Console App 裡，程式的進入點就是 Program.Main()，這很好理解：程式從 Main 開始跑。但是當我們進入 .NET 的現代應用程式架構（例如 Web API、Worker Service），事情就更複雜了：我們不只需要「程式從哪裡開始」，還需要一個「總管」來管理整個應用程式的生命週期與資源。這個「總管」就是 Host。


- 啟動/停止應用程式（像是總開關）。
- 依賴注入 (DI) 容器（負責物件之間的關係與建立方式）。
- 載入設定 (Config)（從 JSON、環境變數、命令列參數等等來源讀取設定）。
- 記錄 (Logging)（集中管理程式運行過程中的訊息與錯誤）。
- 環境資訊 (Environment)（知道現在是 Dev、QA、還是正式環境）。
- 生命週期管理（包含平順的關閉，確保資源釋放）。

範例
```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

class Program
{
    static async Task Main(string[] args)
    {
        // 建立 Host
        using IHost host = Host.CreateDefaultBuilder(args)
            // 1. 設定 Config
            .ConfigureAppConfiguration((hostingContext, config) =>
            {
                config.Sources.Clear(); // 先清掉預設來源

                config.AddJsonFile("appsettings.json", optional: true, reloadOnChange: true)
                      .AddJsonFile($"appsettings.{hostingContext.HostingEnvironment.EnvironmentName}.json", optional: true)
                      .AddEnvironmentVariables()  // 支援環境變數
                      .AddCommandLine(args);     // 支援 CLI 參數
            })

            // 2. 設定 DI 容器
            .ConfigureServices((context, services) =>
            {
                services.AddSingleton<MyService>();
                services.AddHostedService<Worker>(); // 背景服務
            })

            // 3. 設定 Logging
            .ConfigureLogging(logging =>
            {
                logging.ClearProviders();
                logging.AddConsole();
                logging.AddDebug();
            })

            .Build();

        // 4. 取出服務測試一下
        var myService = host.Services.GetRequiredService<MyService>();
        myService.Run();

        // 5. 啟動 Host（會啟動 Worker 等背景服務）
        await host.RunAsync();
    }
}

class MyService
{
    private readonly IConfiguration _config;
    private readonly ILogger<MyService> _logger;

    public MyService(IConfiguration config, ILogger<MyService> logger)
    {
        _config = config;
        _logger = logger;
    }

    public void Run()
    {
        string? settingValue = _config["MySetting"];
        _logger.LogInformation("MyService is running. Config: {Setting}", settingValue);
    }
}

// 這是一個背景服務 (類似定時任務)
class Worker : BackgroundService
{
    private readonly ILogger<Worker> _logger;

    public Worker(ILogger<Worker> logger)
    {
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("Worker is running at: {time}", DateTimeOffset.Now);
            await Task.Delay(2000, stoppingToken);
        }
    }
}
```


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Hosting/2_host.png)


<!-- endtab -->

<!-- tab Host.CreateDefaultBuilder-->

至於 `Host` 怎麼建立? **首先需要建立 HostBuilder，再透過 Build 方法建立 Host instance**

當我們在 `Host` 選用 `DefaultBuilder`，背後做了些什麼呢?

`Host.CreateDefaultBuilder(args) `就像是 一個幫你準備好場地的模板，把大部分應用程式常見的基礎建設都幫你處理好，讓你不用從零開始搭
它主要做了三件事

<br>

## Configuration

- 建好 Host Configuration（啟動用的設定，例如環境變數、CLI 參數）。例如要不要 reload、用哪個環境 (Development / Production)。
- 建好 App Configuration（應用程式用的設定，例如 appsettings.json、環境專屬的 appsettings.Development.json、Secrets、環境變數、CLI）。是應用程式邏輯運行需要的設定，例如資料庫連線字串、API key。值得注意的是，我們不用自己去寫 AddJsonFile("appsettings.json")、AddEnvironmentVariables()、AddConsole()，因為 CreateDefaultBuilder() 已經幫你做了!

<br>

## Logging

自動載入 Logging 區段設定。預設加上 Console 與 Debug logger。如果在 Windows，還會加上 EventLog。內建支援 TraceId、SpanId（方便分散式追蹤）。
並且會加上 ActivityTrackingOptions，因為現代應用程式常常分散在很多服務裡，TraceId / SpanId 讓你能夠跨系統追蹤一個請求的完整流程。

<br>

## DI

使用 Microsoft 內建的 ServiceProvider。在 Development 環境下，會開啟 ValidateScopes 和 ValidateOnBuild，幫助你提早發現 DI 配置錯誤。因為在 Dev 環境早點報錯比較好，避免到 Prod 才發現某個 Service 沒有被註冊，或有循環依賴問題


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Hosting/5_validatescope.png)


```CSHARP

public static IHostBuilder CreateDefaultBuilder(string[] args)
{
    // 創建一個新的 HostBuilder instance
    var builder = new HostBuilder();

    // 設置應用程式的內容根目錄為當前目錄
    builder.UseContentRoot(Directory.GetCurrentDirectory());

    // 藉由 ConfigBuilder 配置主機的 Config
    builder.ConfigureHostConfiguration(config =>
    {
        // 添加以 "DOTNET_" 為前綴的環境變量
        config.AddEnvironmentVariables(prefix: "DOTNET_");
        // 如果提供了 CLI 參數，則加到 config 中
        if (args != null)
        {
            config.AddCommandLine(args);
        }
    });

    // 藉由 ConfigBuilder 配置應用程式的 Config
    builder.ConfigureAppConfiguration((hostingContext, config) =>
    {
        IHostEnvironment env = hostingContext.HostingEnvironment;

        // 確定是否在配置更改時重新加載
        bool reloadOnChange = hostingContext.Configuration.GetValue("hostBuilder:reloadConfigOnChange", defaultValue: true);

        // 添加 JSON config，包括不同環境
        config.AddJsonFile("appsettings.json", optional: true, reloadOnChange: reloadOnChange)
              .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true, reloadOnChange: reloadOnChange);

        // 在 Dev 環境中在 condfig 加上Secrets
        if (env.IsDevelopment() && !string.IsNullOrEmpty(env.ApplicationName))
        {
            var appAssembly = Assembly.Load(new AssemblyName(env.ApplicationName));
            if (appAssembly != null)
            {
                config.AddUserSecrets(appAssembly, optional: true);
            }
        }

        // 添加環境變量
        config.AddEnvironmentVariables();

        // 添加 CLI 參數到 Config
        if (args != null)
        {
            config.AddCommandLine(args);
        }
    })

    // logging Provider 配置 Logging
    .ConfigureLogging((hostingContext, logging) =>
    {
        bool isWindows = RuntimeInformation.IsOSPlatform(OSPlatform.Windows);

        // 在 Windows 平台上，設置 EventLog 的默認日誌級別為 Warning 或以上
        if (isWindows)
        {
            logging.AddFilter<EventLogLoggerProvider>(level => level >= LogLevel.Warning);
        }

        // 載入 Logiging Config
        logging.AddConfiguration(hostingContext.Configuration.GetSection("Logging"));
        logging.AddConsole();
        logging.AddDebug();
        logging.AddEventSourceLogger();

        // 在 Windows 平台上添加事件日誌提供程序
        if (isWindows)
        {
            logging.AddEventLog();
        }

        logging.Configure(options =>
        {
            options.ActivityTrackingOptions = ActivityTrackingOptions.SpanId
                                                | ActivityTrackingOptions.TraceId
                                                | ActivityTrackingOptions.ParentId;
        });
    })

    // 配置 Default Service Provider
    .UseDefaultServiceProvider((context, options) =>
    {
        bool isDevelopment = context.HostingEnvironment.IsDevelopment();
        options.ValidateScopes = isDevelopment;
        options.ValidateOnBuild = isDevelopment;
    });

    return builder;
}

```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Hosting/2_createdefaultbuilder.png)


<!-- endtab -->


<!-- tab IHostEnvironment-->

IHostEnvironment 是 Host 提供的一個服務，它讓應用程式可以知道自己「現在身在什麼環境」，以及一些基礎的環境資訊。
就像程式的「自我定位系統」，可以回答：

- 我是誰？（ApplicationName）
- 我在哪裡？（ContentRootPath）
- 我能存取哪些檔案？（ContentRootFileProvider）
- 我現在處在哪個環境？（EnvironmentName，例如 Dev / Test / Prod）

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.DependencyInjection;

class Program
{
    static void Main(string[] args)
    {
        using IHost host = Host.CreateDefaultBuilder(args).Build();

        var env = host.Services.GetRequiredService<IHostEnvironment>();

        Console.WriteLine($"應用程式名稱: {env.ApplicationName}");
        Console.WriteLine($"根目錄路徑: {env.ContentRootPath}");
        Console.WriteLine($"當前環境: {env.EnvironmentName}");
    }
}
```

<br>

## 判斷環境決定要不要載入額外設定

```csharp
.ConfigureServices((context, services) =>
{
    var env = context.HostingEnvironment;

    if (env.IsDevelopment())
    {
        Console.WriteLine("載入 Dev 專用的服務或設定");
    }
    else if (env.IsProduction())
    {
        Console.WriteLine("載入 Prod 專用的服務或設定");
    }
})
```

<br>

## 設定方式

可以透過環境變數設定
```bash
set DOTNET_ENVIRONMENT=Development
```

launchSettings.json
```JSON
"profiles": {
  "MyConsoleApp": {
    "commandName": "Project",
    "environmentVariables": {
      "DOTNET_ENVIRONMENT": "Development"
    }
  }
}
```


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Hosting/3_ihostenv.png)


<!-- endtab -->

<!-- tab IConfiguration-->

我們常常在 Service 直接這樣用
```CSHARP
public class MyService
{
    private readonly IConfiguration _config;

    public MyService(IConfiguration config)
    {
        _config = config;
    }

    public void Run()
    {
        string value = _config["MySetting:SubValue"];
        Console.WriteLine($"Setting Value: {value}");
    }
}
```

但為什麼我們可以這樣「直接注入」？原因就是 Host 在 Build 的過程中，已經把 IConfiguration 做好並且註冊進 DI 容器 (AddSingleton)，過程如以下說明

- **CreateDefaultBuilder** 幫我們準備 **ConfigurationBuilder** → 加入 JSON、環境變數、Secrets、CLI
- **BuildAppConfiguration()** 呼叫 **configurationBuilder.Build()** → 建立出一個真正的 IConfigurationRoot 實例
- **CreateServiceProvider()** → 呼叫 **serviceCollection.AddSingleton((IServiceProvider _) => _appConfiguration);** 把這個 IConfiguration 實例註冊進 DI 容器

例如 appsettings.json
```JSON
{
  "MySetting": {
    "SubValue": "Hello Config!"
  }
}
```

其中 CreateDefaultBuilder 的 ConfigurationBuilder
```CSHARP

   // 藉由 ConfigBuilder 配置應用程式的 Config
    builder.ConfigureAppConfiguration((hostingContext, config) =>
    {
        IHostEnvironment env = hostingContext.HostingEnvironment;

        // 確定是否在配置更改時重新加載
        bool reloadOnChange = hostingContext.Configuration.GetValue("hostBuilder:reloadConfigOnChange", defaultValue: true);

        // 添加 JSON config，包括不同環境
        config.AddJsonFile("appsettings.json", optional: true, reloadOnChange: reloadOnChange)
              .AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true, reloadOnChange: reloadOnChange);

        // 在 Dev 環境中在 condfig 加上Secrets
        if (env.IsDevelopment() && !string.IsNullOrEmpty(env.ApplicationName))
        {
            var appAssembly = Assembly.Load(new AssemblyName(env.ApplicationName));
            if (appAssembly != null)
            {
                config.AddUserSecrets(appAssembly, optional: true);
            }
        }

        // 添加環境變量
        config.AddEnvironmentVariables();

        // 添加 CLI 參數到 Config
        if (args != null)
        {
            config.AddCommandLine(args);
        }
    })

```

Build() 內查看包含 **BuildAppConfiguration**、**CreateServiceProvider**

```CSHARP

public IHost Build()
{
    if (_hostBuilt)
    {
        throw new InvalidOperationException(System.SR.BuildCalled);
    }

    _hostBuilt = true;
    using DiagnosticListener diagnosticListener = new DiagnosticListener("Microsoft.Extensions.Hosting");
    if (diagnosticListener.IsEnabled() && diagnosticListener.IsEnabled("HostBuilding"))
    {
        Write(diagnosticListener, "HostBuilding", this);
    }

    BuildHostConfiguration();
    CreateHostingEnvironment();
    CreateHostBuilderContext();
    BuildAppConfiguration();
    CreateServiceProvider();
    IHost requiredService = ServiceProviderServiceExtensions.GetRequiredService<IHost>(_appServices);
    if (diagnosticListener.IsEnabled() && diagnosticListener.IsEnabled("HostBuilt"))
    {
        Write(diagnosticListener, "HostBuilt", requiredService);
    }

    return requiredService;
}

private void BuildAppConfiguration()
{
    IConfigurationBuilder configurationBuilder = new ConfigurationBuilder().SetBasePath(_hostingEnvironment.ContentRootPath).AddConfiguration(_hostConfiguration, shouldDisposeConfiguration: true);
    foreach (Action<HostBuilderContext, IConfigurationBuilder> configureAppConfigAction in _configureAppConfigActions)
    {
        configureAppConfigAction(_hostBuilderContext, configurationBuilder);
    }

    _appConfiguration = configurationBuilder.Build();
    _hostBuilderContext.Configuration = _appConfiguration;
}

private void CreateServiceProvider()
{
    ServiceCollection serviceCollection = new ServiceCollection();
    ((IServiceCollection)serviceCollection).AddSingleton((IHostingEnvironment)_hostingEnvironment);
    ((IServiceCollection)serviceCollection).AddSingleton((IHostEnvironment)_hostingEnvironment);
    serviceCollection.AddSingleton(_hostBuilderContext);
    serviceCollection.AddSingleton((IServiceProvider _) => _appConfiguration);
    serviceCollection.AddSingleton((IServiceProvider s) => (IApplicationLifetime)ServiceProviderServiceExtensions.GetService<IHostApplicationLifetime>(s));
    serviceCollection.AddSingleton<IHostApplicationLifetime, ApplicationLifetime>();
    AddLifetime(serviceCollection);

    ...
}
```



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Hosting/4-config.png)


<!-- endtab -->


<!-- tab Generic Host 和 WebApplication Host-->

## Generic Host（IHost）

是 .NET 的「通用主機」。它本質上只是一個 管線組裝器（IHostBuilder）＋ 建出來的主機實例（IHost），可以用不同的組裝方式與模組來建它

```csharp

//// 入口變形
Host.CreateDefaultBuilder(args); // 官方預設全餐。
new HostBuilder().ConfigureDefaults(args); // 等價於「自己 new 底盤，再套官方預設」。
new HostBuilder().ConfigureDefaults(args).InitNine1BaseSDKConsoleApp(args) // 在官方預設上，再套公司標準件。

//// 功能變形
// 1. 只拿 DI/Config/Logging（不 Run/Start），做一次性批次
var host = new HostBuilder().ConfigureDefaults(args)
    .ConfigureServices((ctx, services) => { services.AddTransient<Job>(); })
    .Build();

var job = host.Services.GetRequiredService<Job>();
await job.RunAsync(); // 跑完就結束，不呼叫 Run/Start

// 2. 加上 IHostedService/BackgroundService，做長時間服務
var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((ctx, services) => services.AddHostedService<Worker>())
    .Build();

await host.RunAsync(); // 阻塞直到關閉

// 3. 加上 .ConfigureWebHostDefaults(...)，變成 Web

//// 容器/架構變形
// 1. 換 DI 容器（UseServiceProviderFactory，如 Autofac）
using Autofac;
using Autofac.Extensions.DependencyInjection;

var host = new HostBuilder()
    .ConfigureDefaults(args)
    .UseServiceProviderFactory(new AutofacServiceProviderFactory())
    .ConfigureContainer<ContainerBuilder>(builder =>
    {
        builder.RegisterType<MyService>().As<IMyService>().SingleInstance();
    })
    .Build();
```

模組化你的建置邏輯（自訂 IHostBuilder 擴充方法），形成企業級骨架（像 InitNine1BaseSDKConsoleApp）
重點是 IHostBuilder 是可組合、可擴充、可封裝的建置器；誰先加、誰後加，會影響設定來源優先序與Logging/DI 的最終行為



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Hosting/7_generic_host.png)


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Hosting/9_extension.png)


## WebApplication.CreateBuilder()

- 內部仍然是 Generic Host
- 只是多做了 Web 特有的設定（Kestrel、Middleware 管線、Routing 等）
- 外觀上變得更「簡單、直覺」

```csharp
var builder = WebApplication.CreateBuilder(args);

// 註冊服務
builder.Services.AddControllers();

var app = builder.Build();

// 配置 Middleware 管線
app.UseHttpsRedirection();
app.MapControllers();

app.Run();
```

WebApplication.CreateBuilder(args) 背後就是

- 建立一個 Generic Host
- 自動呼叫 ConfigureWebHostDefaults 幫你設定 Web 環境（Kestrel, IIS Integration, Routing…）
- 回傳一個更簡化的 WebApplicationBuilder，讓你可以同時操作 Host 層級 (DI, Logging, Config) 與 Web 層級 (Middleware, Endpoint)


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Hosting/6_webapppng.png)


<!-- endtab -->

<!-- tab host.Run / host.Start / 直接跑-->

| 方式                                   | 行為                                                                                            | 適用情境                           |
| ------------------------------------ | --------------------------------------------------------------------------------------------- | ------------------------------ |
| **`host.Run()` / `host.RunAsync()`** | 啟動 Host，並**阻塞目前執行緒**，直到收到關閉訊號 (Ctrl+C / SIGTERM)。所有註冊的 `IHostedService` 會自動跑，Host 負責生命週期。     | Web API、Worker Service (長時間常駐) |
| **`host.Start()`**                   | 啟動 Host，但**不阻塞**。背景的 `IHostedService` 仍會跑，但主程式可以繼續往下執行。你需要自己決定何時 `StopAsync()` 或 `Dispose()`。 | 混合場景：除了背景服務，還要自己跑額外邏輯          |
| **直接跑服務 (不啟動 Host)**                 | 只把 Host 當 **DI/Config 工廠**，完全不啟動生命週期。沒有 `IHostedService` 會跑。你要自己從容器拿出物件並呼叫方法。                 | 一次性批次任務 (Job/Console App)      |


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Hosting/8_run_and_start.png)


<!-- endtab -->


<!-- tab summary-->


![p](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Hosting/10_final.png)


<!-- endtab -->

{% endtabs %}