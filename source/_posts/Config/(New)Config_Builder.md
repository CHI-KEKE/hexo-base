---
title: Config Builder
date: 2025-09-08 08:33:01
categories: Config
top_img: https://github.com/CHI-KEKE/pics/blob/main/Config/gear.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Config/gear.png?raw=true
tags:
    - Config
toc:
toc_number:
comments :
---

還記得第一次接觸大型專案時的場景嗎？
到處散落著不同的設定檔，有人用 JSON，有人用 XML，還有人直接在程式碼裡寫死一堆參數。結果要切換環境時，就像在拼一個缺角的拼圖：少一個值，系統就崩潰。

這就好比一場樂團演奏，每個樂手都拿著自己的譜，卻沒有人統一節拍。開發人員成了指揮，卻只能不停喊：「誰的設定跑掉了！」。

而 .NET Core 的 ConfigurationBuilder 就像是一個樂譜整理師，它把不同來源的設定集中起來，統一成一份總譜。無論是 JSON、環境變數，還是命令列參數，都能被收編進來，讓系統在啟動時就能「合奏」出正確的旋律。

## ⚙️ .NET Core 組態（Configuration）

.NET Core 非常依賴 **Builder 模式**，而「組態（Configuration）」就是其中一個代表性範例。  

Configuration 系統就像一個「設定管理中心」，它把不同來源的設定（JSON 檔、環境變數、命令列參數…）集中起來，讓你用一致的方式讀取與管理。  

<br>
<br>

## ⚙️ ConfigurationBuilder 的角色

在 .NET Core 裡，`ConfigurationBuilder` 就像是組態的 **施工藍圖**。它定義了如何收集設定來源，並最終產生一個可供程式使用的組態物件。

### IConfigurationBuilder 原始碼

```csharp
public interface IConfigurationBuilder
{
    IDictionary<string, object> Properties { get; }
    IList<IConfigurationSource> Sources { get; }
    IConfigurationBuilder Add(IConfigurationSource source);
    IConfigurationRoot Build();
}
```

- Sources：組態的資料來源清單（如 JSON、環境變數、CLI 參數）。
- Add()：把新的資料來源加進來。
- Build()：把所有來源建構成一個完整的 IConfigurationRoot。

ConfigurationBuilder 就像在收集「材料」，而 Build() 則是把這些材料組裝成最終的組態物件。


<br>
<br>

## ⚙️ ConfigurationSource 與 Provider

每一個資料來源（ConfigurationSource），都會對應一 IConfigurationProvider。

- Source：資料的定義，例如 appsettings.json。
- Provider：封裝讀取邏輯，負責把設定值提供給 ConfigurationRoot。

```csharp
public IConfigurationRoot Build()
{
    var providers = new List<IConfigurationProvider>();
    foreach (var source in Sources)
    {
        var provider = source.Build(this);
        providers.Add(provider);
    }
    return new ConfigurationRoot(providers);
}
```

<br>
<br>

## ⚙️ ConfigurationSection

如果只想讀取設定樹狀結構中的某一部分，可以使用 GetSection()。這會產生一個 ConfigurationSection，裡面包含：

- Key：節點名稱
- Path：節點的完整路徑
- Value：對應的值

```csharp
public interface IConfigurationSection : IConfiguration
{
    string Key { get; }
    string Path { get; }
    string? Value { get; set; }
}
```

這樣，就可以針對某一節點建立「小組態系統」。

<br>
<br>

## ⚙️ 建立組態的方式

```POWERSHELL
dotnet add package Microsoft.Extensions.Configuration
```

最簡單的組態建立方式

```csharp
var config = new ConfigurationBuilder()
    .SetBasePath(Directory.GetCurrentDirectory())
    .AddJsonFile("appsettings.json")
    .Build();
```

在 Host 中使用 Configuration

實際的 .NET Core 專案中，組態通常會和 Host Builder 整合
```csharp
static void Main(string[] args)
{
    IHost host = CreateHostBuilder(args).Build();
    host.Run();
}

public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureAppConfiguration((context, config) =>
        {
            var env = context.HostingEnvironment;
            config.AddJsonFile("appsettings.json", optional: true, reloadOnChange: true);
            config.AddJsonFile($"appsettings.{env.EnvironmentName}.json", optional: true, reloadOnChange: true);
        });
```

這樣就能根據不同環境載入不同的設定檔。

<br>
<br>

## ⚙️ 組態的讀取方式

組態主要有兩種讀取方式：

**弱型別讀取（Weakly Typed Reading）**

- 優點：簡單、靈活，可以隨時讀取任何 Key。
- 缺點：缺乏型別安全，容易出現 Runtime Error。

```csharp
public class UserService
{
    private readonly IConfiguration _configuration;

    public UserService(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public int GetMaxUsers()
    {
        return int.Parse(_configuration["UserSettings:MaxUsers"]); // 可能爆炸
    }
}
```

**強型別讀取（Strongly Typed Reading）**

這種方式透過 POCO 類別包裝設定，優點是 型別安全、可讀性高。

有兩種常用方法：

使用 Get<T>()
```csharp
var appSetting = hostContext.Configuration.Get<AppSetting>();
services.AddSingleton(appSetting);
```

使用 Bind()
```csharp
var appSetting = new AppSetting();
hostContext.Configuration.Bind(appSetting);
services.AddSingleton(appSetting);
```

差異：

Get<T>()：直接建立並回傳新物件。
Bind()：將設定填入已存在的物件，可在綁定前後進行額外處理。


<br>
<br>

## ⚙️ 組態的覆蓋特性

Configuration 有「後蓋前」的特性，越晚加入的來源，優先權越高。

例子：

- appsettings.json → 預設設定
- appsettings.Production.json → 覆蓋部分設定

- 環境變數 → 再次覆蓋
- CLI 參數 → 最高優先權

這樣就能針對不同環境，維持統一結構但調整細節。


<br>
<br>

## ⚙️ 組態資料的三態轉換

組態的資料，實際上經歷了三個階段：

**原始態（Raw State）**

來自外部來源：JSON、環境變數、CLI。

**中介態（Intermediate State）**

轉換成鍵/值對，並可進行覆蓋、合併。

**邏輯態（Logical State）**

綁定到 POCO 類別，成為程式可直接使用的邏輯結構。

## ⚙️ 結語

當你理解並善用 ConfigurationBuilder，就會發現：設定不再是專案裡的隱藏炸彈，而是一個可以隨時掌控的基礎建設。它的價值不僅在於「能讀取設定」，而在於

- 讓不同來源的設定能 整合 起來，避免遺漏。
- 透過「後蓋前」機制，讓你輕鬆管理多環境差異。
- 搭配強型別綁定，讓錯誤能在編譯時就被攔下。

如果說程式碼是建築的骨架，那麼設定就是水電管線。ConfigurationBuilder 就像是一張完整的藍圖，保證每根管線都能正確接好，不會因為環境不同而亂流。

有了它，你就能專注在功能開發上，不再被「為什麼 Production 的設定又跑掉了？」這種問題糾纏。換句話說，它就是那個讓專案從「混亂交響」走向「和諧樂章」的關鍵。 