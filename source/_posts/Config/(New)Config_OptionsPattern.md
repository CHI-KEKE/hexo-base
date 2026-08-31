---
title: Options Pattern
date: 2025-09-02 22:33:01
categories: Config
top_img: https://github.com/CHI-KEKE/pics/blob/main/Config/gear.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Config/gear.png?raw=true
tags:
    - Config
toc:
toc_number:
comments :
---

這就像搬家時，把東西胡亂裝在箱子裡，搬到新家後才發現少了零件、找不到工具。
而 Options Pattern，就像是給每個箱子都貼上清楚的標籤，還附上說明書，無論是開發、測試還是上線環境，都能安心打開箱子，拿到正確的東西。

Options Pattern 是把 appsettings.json 裡的設定資料，轉成一個 C# 類別，然後透過「依賴注入（DI）」的方式注入到你的服務中使用。

![Options Pattern](https://github.com/CHI-KEKE/pics/blob/main/Config/gear.png?raw=true)

## ⚙️ 編譯時類型檢查（Compile-time Type Checking）

你把設定對應到一個 C# 類別，這代表你在編譯階段就能知道型別對不對，不是等到跑程式才發現爆炸。

```CSHARP
var smtpServer = Configuration["Email:SmtpServer"]; // 傳回 string，錯了也不會報錯
var port = int.Parse(Configuration["Email:Port"]); // 如果 Port 是 null 或字串，這邊才會爆炸
```

Options Pattern 作法（強型別）：

```CSHARP
public class EmailOptions
{
    public string SmtpServer { get; set; }
    public int Port { get; set; } // 如果 JSON 配錯型別，會在啟動時就報錯
}


builder.Services.Configure<EmailOptions>(builder.Configuration.GetSection("Email"));
```

<br>
<br>

## ⚙️ 智能提示和自動完成（IntelliSense）

當你用強型別類別來包裝設定，IDE（像是 Visual Studio）就會幫你提示成員，不會拼錯、不會猜錯。

```CSHARP
public class EmailOptions
{
    public string SmtpServer { get; set; }
    public int Port { get; set; }
}

public class EmailService
{
    private readonly EmailOptions _options;

    public EmailService(IOptions<EmailOptions> options)
    {
        _options = options.Value;
    }

    public void Send()
    {
        Console.WriteLine(_options.SmtpServer); // 這裡打 . 就會提示 SmtpServer 跟 Port
    }
}
```

<br>
<br>

## ⚙️ 跨環境一致性（Consistency Across Environments）

你在開發、測試、正式環境中都用同一個設定類別，只換內容，保證每個環境都有同樣的結構與邏輯。

```CSHARP
public class EmailOptions
{
    public string SmtpServer { get; set; }
    public int Port { get; set; }
}
```

```json
"Email": {
  "SmtpServer": "dev.smtp.com",
  "Port": 1025
}

"Email": {
  "SmtpServer": "prod.smtp.com",
  "Port": 587
}
```

只要寫好一個 EmailOptions 類別，不管在哪個環境都能用 避免環境不一致造成錯誤


<br>
<br>

##　⚙️ 依賴注入整合（Dependency Injection Integration）

Options Pattern 完全整合進 .NET 的 DI 機制，可以像注入其他服務一樣，注入設定物件。

```CSHARP
builder.Services.Configure<EmailOptions>(builder.Configuration.GetSection("Email"));

public class EmailService
{
    private readonly EmailOptions _options;

    public EmailService(IOptions<EmailOptions> options)
    {
        _options = options.Value;
    }
}
```


## ⚙️ 結語

Options Pattern 不只是「把設定寫進一個類別」這麼簡單，它真正的價值在於：讓你的程式能夠更早發現錯誤、更容易維護，並且 在不同環境中保持一致。

如果說程式是建築，那麼設定就是水電管線。Options Pattern 就像是標準化的藍圖，讓每個環境的管線都接得穩、接得對。這樣你才不會在半夜 Debug 的時候，才發現原來爆水管的原因，是因為你在設定裡打錯了一個字母。

當你的團隊開始使用 Options Pattern，你會發現：設定再也不是隱藏的陷阱，而是清晰可靠的基礎，讓小細節不再成為大災難