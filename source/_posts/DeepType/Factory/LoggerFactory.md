---
title: LoggerFactory
date: 2026-01-11 10:58:05
categories: Factory
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Factory/factory.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Factory/factory.png
tags:
    - Factory
toc:
toc_number:
comments :
---

[Reference](https://github.com/dotnet/runtime/blob/main/src/libraries/Microsoft.Extensions.Logging/src/LoggerFactory.cs#L149)


{% tabs Factory%}

<!-- tab 服務（Service）直接注入 ILogger<T> 來使用-->

##　表層

```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;

    public OrderService(ILogger<OrderService> logger)
    {
        _logger = logger;
    }

    public void PlaceOrder()
    {
        _logger.LogInformation("Place order");
    }
}
```


![1](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Factory/Logger/1__serface.png)



## 🕒 T0：應用程式啟動時

```csharp
var builder = WebApplication.CreateBuilder(args);
```

框架在這一步做了幾件關鍵事情

- 建立 DI Container
- 註冊 Logging 系統 ILoggerFactory、ILogger<T>（泛型 Logger 的「生產規則」）、各種 Logger Provider（Console、File、Cloud…）

注意，這時候還沒有任何 Logger 實例被建立


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Factory/Logger/2__t0t1_di_logger.png)


## 🕒 T1：某個地方需要 OrderService

```csharp
public class OrderController
{
    public OrderController(OrderService orderService)
    {
    }
}
```

DI Container 開始「建立物件樹」

<br>

## 🕒 T2：DI Container 要建立 OrderService

DI 發現 OrderService 需要 ILogger<OrderService>，於是它內部做了這件事
```csharp
var loggerFactory = container.Get<ILoggerFactory>();
var logger = loggerFactory.CreateLogger(
    "YourApp.Services.OrderService"
);
```

Category 就是在這一刻決定的，來源是 typeof(OrderService).FullName，不是我們寫的，是 DI 推導的


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Factory/Logger/2__t2_create_logger.png)


## 🕒 T3：ILogger<OrderService> 被注入完成

```csharp
new OrderService(logger);
```

拿到的是已經綁好 Provider、已經套好 Filter、已經知道 Category 的 Logger

<br>

## 🕒 T4：你呼叫 _logger.LogInformation()

```bash
ILogger
  ↓
LogLevel Filter（這個 Category 能不能寫）
  ↓
Scope（TraceId, RequestId）
  ↓
所有註冊的 Provider
  ↓
Console / File / Cloud
```

這個情境完全沒有「建立 Logger」，只是「被動接收一個已經決定好身份的 Logger」!


![ff](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Factory/Logger/2__t3_t4_execute.png)



<!-- endtab -->

<!-- tab 在 Program.cs 使用 CreateLogger()-->


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Factory/Logger/3__logger_factory_by_self.png)


```csharp
var startupLogger = app.Services
    .GetRequiredService<ILoggerFactory>()
    .CreateLogger("Startup");

startupLogger.LogInformation("App starting");
```

<br>

## 🕒 T0：Host & ServiceContainer 已建立

```csharp
var app = builder.Build();
```

這一刻代表 DI Container 已經完成、Logging Provider 已經註冊，但還沒進入 Request Pipeline

<br>

## 🕒 T1：你手動向 DI 要 Factory

```csharp
var loggerFactory = app.Services.GetRequiredService<ILoggerFactory>();
```

「我現在不在 DI 管理的物件裡，但我想主動參與 Logging 系統」

<br>

## 🕒 T2：你手動決定 Category

```csharp
CreateLogger("Startup");
```

| 項目       | Service 注入 | Program.cs |
| -------- | ---------- | ---------- |
| Category | DI 推導      | 你自己命名      |
| 時機       | 建構物件時      | 立即建立       |
| 語意       | 類別         | 流程 / 階段    |

<br>

## 🕒 T3：Factory 建立 Logger

```bash
Create Logger
  ↓
套用設定檔 Filter
  ↓
組合 Provider
  ↓
回傳 ILogger
```

這次沒有型別、沒有泛型、完全由自己負責命名

<br>

## 🕒 T4：你呼叫 LogInformation

```bash
ILogger
  ↓
LogLevel Filter（Startup 能不能寫）
  ↓
Scope（通常比較少）
  ↓
Provider
```


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Factory/Logger/3__logger_self_naming.png)



![5](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Factory/Logger/4_self_vs_di_passive.png)


<!-- endtab -->

<!-- tab Category-->

Category 是用來「讓你只看你現在想看的東西」

預設情況下，設定檔是這樣
```json
"Logging": {
  "LogLevel": {
    "Default": "Information"
  }
}

```

Logging 系統用 Category 內部做的是

```bash
Log 發生
 ↓
用 Category 找最精確的設定
 ↓
找不到 → 往上用 Prefix
 ↓
最後用 Default
```


![7](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Factory/Logger/5_what_is_category.png)


## 情境 1：只想 Debug 某個 Service

```json
"Logging": {
  "LogLevel": {
    "Default": "Warning",
    "MyApp.Services.OrderService": "Debug"
  }
}
```

OrderService：Debug 全開，其他 Service：只剩 Warning，不用改任何程式碼


![8](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Factory/Logger/5_logger_level.png)


## 情境 2：關掉吵死人的 Framework Log

```json
"Logging": {
  "LogLevel": {
    "Microsoft": "Warning",
    "Microsoft.Hosting.Lifetime": "Information"
  }
}
```

Category 讓你「降噪」，不是加功能


![5](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Factory/Logger/5__lower_noisy.png)


## 情境 3：Production 緊急追問題

不可能這樣做 : "Default": "Debug"，會直接炸掉 log，因此 Category 可以「精準開刀」

<!-- endtab -->

<!-- tab Factory Pattern-->

Factory Pattern 真正在解決的不是「怎麼 new」，而是「你不該知道你拿到的是哪一個實作」

<br>

## 你只表達「我需要 Logger」

```csharp
CreateLogger("Startup");
```

完全沒說用 ConsoleLogge、還是 FileLogger、還是 CloudLogger，只描述需求，不描述實作

<br>

## Factory 依照「當下環境」決定實作

ILoggerFactory 會根據

- appsettings.json
- Environment（Dev / Prod）
- 已註冊的 Provider

組合出一個 Logger（甚至是多個 Logger 的集合）

<br>

## 拿到的是「結果」，不是「過程」

這正是 Factory Pattern 的經典精神 Object creation is delegated elsewhere，沒有 Factory Pattern，程式會長這樣

```csharp
ILogger logger;

if (env.IsDevelopment())
{
    logger = new ConsoleLogger();
}
else
{
    logger = new FileLogger("log.txt");
}
```

❌ 呼叫端知道太多
❌ 每加一種 Logger 都要改業務邏輯
❌ Startup / Controller / Service 全部會複製這段判斷

使用 ILoggerFactory 之後

```csharp
var logger = loggerFactory.CreateLogger("Startup");
```


✅ 呼叫端不關心實作
✅ 新增 Provider 不影響舊程式
✅ 完全符合 Open/Closed Principle

.NET 的 Logger Factory

- 結合 DI
- 結合 Configuration
- 結合 Runtime Context

它不是教學用的小工廠，而是基礎建設的一環


![5](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Factory/Logger/7_factory.png)


## 這和 Dependency Injection 的關係是什麼？

DI 是「物件怎麼來」，Factory 是「物件怎麼生」。DI Container 負責管理生命週期、Factory 負責建立「合適的實例」、Logger 是兩者合作的經典案例


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Factory/Logger/8_factory_strategy.png)


## 為什麼不用直接注入 ILogger？

因為 Logger 需要「延後決定」與「上下文資訊」，Logger name / category 是 runtime 決定的，Factory 能在建立時注入 context，而 new 做不到這件事

❌ Factory Pattern = 把 new 包起來
✅ Factory Pattern = 隔離變化點

Logger 的「變化點」是

- 環境
- Provider
- 設定
- 輸出策略

而 Factory 正好把這些全部擋在外面!


![5](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Factory/Logger/7_decouping_mean.png)

<!-- endtab -->

<!-- tab 為什麼不直接 new Logger -->

因為 Logger 不是單純「印字串」，而是整個「記錄策略與環境設定」的一部分，你不能自己亂生一支不受控的 Logger，這不是在 new 一個物件，而是在跟系統說「請依照目前環境與設定，幫我產生一支屬於 Startup 類別/模組的 Logger」

如果自己 new 一個 Logger

- ❌ 不知道目前是 Development 還是 Production
- ❌ 不知道 Log Level 設定（可能該寫的不寫、不該寫的狂寫）
- ❌ 不會進到 Console / File / Cloud
- ❌ 跟其他 Logger 行為不一致

Logger 是「策略物件」，不是「工具物件」，如果 Logger 可以亂 new，每個人都用自己的規則，Log 格式、Level、輸出位置全亂，Debug 時你會找不到「為什麼這行沒印」，Factory 強迫你「走同一套規則」


![5](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Factory/Logger/6_dont_new_logger.png)

<!-- endtab -->

<!-- tab summary-->


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Factory/Logger/logger.png)


<!-- endtab -->


{% endtabs %}
