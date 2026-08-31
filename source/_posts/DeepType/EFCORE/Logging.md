---
title: EF Core - Logging
date: 2025-05-27 07:38:05
categories: 資料疆界的航圖
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Logging/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Logging/1_landing.png
tags:
    - EF CORE
toc:
toc_number:
comments :
---


{% tabs Logging%}

<!-- tab whisper-->

在平穩無波的資料操作背後，EF Core 為我們自動生成了大量 SQL 語法。這些語法就像資料的悄悄話，大部分時間默默運作，沒出錯就好。但一旦效能下滑、查詢遲緩，這些悄悄話就需要被「聽見」。

因此，我們需要打開 EF Core 的日誌紀錄機制，看看背後究竟在跑什麼樣的 SQL


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Logging/1_landing.png)


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Logging/1_underneath.png)


<!-- endtab -->

<!-- tab 為什麼要紀錄 EF Core 自動產生的 SQL 命令?-->

EF Core 是一套功能強大的 ORM 框架，幫助我們用物件導向的方式操作資料。但 ORM 所產生的 SQL 並非總是最有效率的選擇。

透過 Logging，我們可以：

- 檢查 EF Core 實際送出的 SQL 語法  
- 發現不必要的 JOIN 或 WHERE 條件  
- 優化查詢速度（例如加上索引、改用 View 或 Stored Procedure）

這對效能調教與除錯都非常有幫助


![7](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Logging/1_why_need.png)


<!-- endtab -->

<!-- tab 調整 `appsettings.json` 的 LogLevel-->

```JSON
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore.Database.Command": "Information" //// 設定為較低層級就印出來
    }
  },
  "AllowedHosts": "*"
}
```

![EFCoreLogging](https://github.com/CHI-KEKE/pics/blob/main/EF/Logging/EFCoreLogging.png?raw=true)

如此設定後，就能把 EF Core 執行的 SQL 命令印出來



![5](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Logging/2_appsetting.png )



<!-- endtab -->

<!-- tab 在程式中啟用敏感資料紀錄（選配）-->

```CSHARP
builder.Services.AddDbContext<AdventureWorks2022>(options =>
{
    options.UseSqlServer(configurationManager.GetConnectionString("AdventureWorks2022"));
    options.EnableSensitiveDataLogging();
});
```

![EFCoreLoggingSensitive](https://github.com/CHI-KEKE/pics/blob/main/EF/Logging/EFCoreLoggingSensitive.png?raw=true)


可以進一步 Log 出經過的事件

```CSHARP
builder.Services.AddDbContext<AdventureWorks2022>(options =>
{
    options.UseSqlServer(configurationManager.GetConnectionString("AdventureWorks2022"));
    options.EnableSensitiveDataLogging();
    options.LogTo(Console.WriteLine); //// 傳入 Action<string>
});
```

![EFCoreLoggingEvent](https://github.com/CHI-KEKE/pics/blob/main/EF/Logging/EFCoreLoggingEvent.png?raw=true)


![7](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Logging/4_sensitive.png)


這樣能看到查詢中的參數值，對除錯特別有幫助。但要小心不要在 Production 使用，避免洩露敏感資料


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Logging/3_log_toi_console.png)


<!-- endtab -->

<!-- tab 設定 LogTo：印出日誌到 Console 或其他地方-->

```CSHARP
options.LogTo(Console.WriteLine, LogLevel.Information);
```

你也可以指定 LogLevel 或類別：
```CSHARP
options.LogTo(Console.WriteLine, new[] { DbLoggerCategory.Database.Name }, LogLevel.Information);
```

這樣能針對 SQL Command 類別做過濾，更聚焦


![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Logging/5_filter.png)


<!-- endtab -->

<!-- tab 輸出到檔案中：永久保存你的 SQL 日誌-->

可以把日誌寫到檔案中

```CSHARP
private readonly StreamWriter _logStream = new StreamWriter("EFCoreDebug.log", append: true);

protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    => optionsBuilder.LogTo(_logStream.WriteLine);

public override void Dispose()
{
    base.Dispose();
    _logStream.Dispose();
}

public override async ValueTask DisposeAsync()
{
    await base.DisposeAsync();
    await _logStream.DisposeAsync();
}
```
這種方式適合長時間觀察 SQL 執行狀況，或做問題追蹤


![5](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Logging/6_log_to_file.png)


![h](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Logging/7_dispose.png  )


<!-- endtab -->

<!-- tab 閱讀：從官方與社群學更多-->

{% btn 'https://learn.microsoft.com/en-us/archive/msdn-magazine/2018/october/data-points-logging-sql-and-change-tracking-events-in-ef-core?WT.mc_id=DT-MVP-4015686',Logging SQL and Change-Tracking Events in EF Core,far fa-hand-point-right %}

{% btn 'https://learn.microsoft.com/en-us/ef/core/logging-events-diagnostics/?tabs=v3&WT.mc_id=DT-MVP-4015686#other-applications',Overview of Logging and Interception,far fa-hand-point-right %}

{% btn 'https://www.entityframeworktutorial.net/efcore/logging-in-entityframework-core.aspx',Logging in Entity Framework Core,far fa-hand-point-right %}

{% btn 'https://itnext.io/entity-framework-core-show-parameter-values-in-logging-5ac58b6a4929',Entity Framework Core: Show Parameter Values in Logging,far fa-hand-point-right %}


<!-- endtab -->

<!-- tab summary-->

開啟 EF Core Logging，就像戴上一副能聽懂資料庫語言的耳機。它能幫助你更清楚每一筆資料的來去、每一次查詢的代價



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Logging/final.png)


<!-- endtab -->


{% endtabs %}
