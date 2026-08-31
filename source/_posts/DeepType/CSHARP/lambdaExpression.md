---
title: LambdaExpression
date: 2024-11-24 11:24:09
categories: 程思舞想
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/Lambda/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/Lambda/1_landing.png
tags:
    - C#
toc:
toc_number:
comments :
---


{% btn 'https://blog.darkthread.net/blog/lambda-exp-as-prop-param/',Lambda Expression 應用 - 用強型別動態指定欄位名稱,far fa-hand-point-right %}


{% tabs LambdaExpression%}


<!-- tab 🌿 Lambda 的誕生：從 Delegate 到優雅的表達式-->

Lambda 表達式（Lambda Expression） 並不是從天而降的魔法。它的根基，其實建立在一個更早、更樸實的概念上 : delegate



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/Lambda/1_landing.png)


<!-- endtab -->

<!-- tab Delegate：型別安全的 Callback-->

所謂 Callback（回呼），是一種「把方法當作參數傳遞」的設計方式。舉個例子，假設你想在任務完成後執行一段通知程式
```csharp
void DownloadFile(string url, Action onCompleted)
{
    Console.WriteLine($"Downloading {url}...");
    onCompleted(); // 任務完成後回呼
}
```

這裡的 onCompleted 就是「回呼方法」，而在 C# 中，這種 callback 能夠安全運作的關鍵就是 delegate

而 Lambda 表達式，更是進一步，它是「匿名的 delegate」，讓我們能更簡潔地撰寫任務完成後執行的函式

<!-- endtab -->

<!-- tab 🌸 Expression-bodied 屬性-->

在 C# 6 之後，可以用 Lambda 表達式簡化「只有一行邏輯」的屬性、方法或運算子

```csharp

//// get
public DateTime? BookingDateTime
{
    get
    {
        return this.BookingTimeUTC?.ToLocalTime();
    }
}

//// 簡化成
public DateTime? BookingDateTime => this.BookingTimeUTC?.ToLocalTime();


//// 設定 set 甚至可以
public DateTime? BookingDateTime
{
    get => this.BookingTimeUTC?.ToLocalTime();
    set => this.BookingTimeUTC = value?.ToUniversalTime();
}
```

<!-- endtab -->


<!-- tab 🌸 LINQ-->

| 類型                      | 範例                                          | 是否用 Lambda             |
| ----------------------- | ------------------------------------------- | ---------------------- |
| **查詢語法（Query Syntax）**  | `from x in list where x > 5 select x`       | ❌ 沒有直接寫 Lambda         |
| **方法語法（Method Syntax）** | `list.Where(x => x > 5).Select(x => x * 2)` | ✅ 使用 Lambda Expression |

<br>

## LINQ - Where

```CSHARP
var result = numbers.Where(x => x > 5);
```

`Where()` 的參數是一個 `Func<T, bool>`，而 `x => x > 5` 就是傳入這個委派的 Lambda Expression

編譯器會幫你生成一個匿名方法

```CSHARP
bool FilterFunc(int x) => x > 5;
```

<br>

## Query Syntax

Query Syntax 看起來沒 Lambda，其實編譯器會在背後幫你轉成 Lambda 形式，所以底層其實還是轉成 Lambda Expression


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/Lambda/3_query_method_query.png)



LINQ 有兩種「Lambda 型式」

| 類型                                                  | 例子                             | 用途             |
| --------------------------------------------------- | ------------------------------ | -------------- |
| **Expression Lambda (`Expression<Func<T, bool>>`)** | 在 Entity Framework, IQueryable | 會被轉成 SQL 或表達式樹 |
| **Delegate Lambda (`Func<T, bool>`)**               | 在 LINQ to Objects              | 直接在記憶體中執行      |



```CSHARP
// LINQ to Objects
var result = list.Where(x => x.Age > 18); 
// x => x.Age > 18 是 Func<Person, bool>

// LINQ to Entities (EF Core)
var result = db.People.Where(x => x.Age > 18);
// x => x.Age > 18 是 Expression<Func<Person, bool>>
```

- IEnumerable<T> 的世界，Lambda 是可執行程式邏輯（delegate）
- IQueryable<T> 的世界，Lambda 是一棵「語法樹」（Expression Tree），會被轉譯成 SQL 


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/Lambda/4_ie_iq.png)


<!-- endtab -->

<!-- tab 🌸 用強型別動態指定欄位名稱-->

想像你是一個遊戲數據分析師，手上有一堆玩家資料，老闆突然說：「給我做一個 CSV 匯出功能，但我不確定每次要哪些欄位，可能今天要姓名和等級，明天要分數和註冊日期...」

<br>

## 💭 硬碼派

```CSHARP

void Main()
{
	GetCsv(Players,"Name","RegDate","Level");
}

public class Player
{
	public string Name{get;set;}
	public DateTime RegDate{get;set;}
	public byte Level{get;set;}
	public int Score{get;set;}
	public Player(string name, DateTime regDatetime, byte level, int score)
	{
		Name = name;
		RegDate = regDatetime;
		Level = level;
		Score = score;
	}
}

static Player[] Players = new[]
{
	new Player("Allen", new DateTime(2024,1,1),  1,  255),
	new Player("Mochi", new DateTime(2022, 12, 21), 2, 32767),
	new Player("Co", new DateTime(2012, 1, 1), 99, 65535)
};

public static void GetCsv(IEnumerable<Player> playerList, string col1,string col2, string col3)
{
	Func<Player,string,object> getProps = (p,col) => {
		switch (col)
		{
			case "Name":
				return p.Name;
			case "RegDate":
				return p.RegDate.ToString("yyyy-MM-dd");
			case "Level":
				return p.Level;
			case "Score":
				return p.Score;
			default:
				throw new NotImplementedException();
		};
	};

	foreach (var player in playerList)
		Console.WriteLine($" col1 : {getProps(player, col1)}\n col2 : {getProps(player, col2)}\n col3 : {getProps(player,col3)}");
}


// col1 : Allen
// col2 : 2024-01-01
// col3 : 1
// col1 : Mochi
// col2 : 2022-12-21
// col3 : 2
// col1 : Co
// col2 : 2012-01-01
// col3 : 99

```

這個寫法就像是「用鐵鎚釘螺絲」，雖然能用，但毫無彈性

- 🚫 **只能處理 Player 類型**：如果要處理其他類型（如 `Employee`、`Product`），就得重寫方法
- 🚫 **固定三欄限制**：想要四欄？五欄？抱歉，請重新寫方法
- 🚫 **欄位名稱寫死**：新增一個 `Email` 屬性？要修改 switch case



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/Lambda/5_2_hardcode.png)


## 💭 反射

```CSHARP

public void GetCsv2<T>(IEnumerable<T> playerList, string col1,string col2, string col3)
{
	var type = typeof(T);
	var PropInfo1 = type.GetProperty(col1,BindingFlags.Instance | BindingFlags.Public);
	var PropInfo2 = type.GetProperty(col2,BindingFlags.Instance | BindingFlags.Public);
	var PropInfo3 = type.GetProperty(col3,BindingFlags.Instance | BindingFlags.Public);
	foreach (var player in playerList)
		Console.WriteLine($"col1 : {PropInfo1.GetValue(player)}\ncol2 : {PropInfo2.GetValue(player)}\ncol3 : {PropInfo3.GetValue(player)}");
}


// col1 : Allen
// col2 : 1/1/2024 12:00:00 AM
// col3 : 1
// col1 : Mochi
// col2 : 12/21/2022 12:00:00 AM
// col3 : 2
// col1 : Co
// col2 : 1/1/2012 12:00:00 AM
// col3 : 99
```



用 Reflection 解決了泛型問題，但引發了新的煩惱

| 問題 | 具體情況 | 影響 |
|------|----------|------|
| 🎨 **格式控制失效** | 日期只能輸出 `1/1/2024 12:00:00 AM`，無法自訂為 `2024-01-01` | 使用者體驗差 |
| 🐌 **效能犧牲** | Reflection 需要在執行時期透過字串查找屬性 | 大量資料處理會變慢 |
| 💥 **執行時期才爆炸** | 欄位名稱打錯（`"Naem"` 而不是 `"Name"`），編譯期間不會發現 | 生產環境才發現錯誤 |
| 📏 **仍有數量限制** | 還是被綁死在三欄，要四欄還是得改方法簽章 | 不夠靈活 |



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/Lambda/5_3_reflection.png)


## Lambda Expression

既然前兩種方法都有缺點，我們換個思考方式

**「與其限制使用者怎麼取資料，不如讓使用者自己決定要怎麼取！」**

```CSHARP

public void GetCsv3<T>(IEnumerable<T> playerList, params Func<T,object>[] cols)
{
	foreach(var player in playerList)
		foreach(var col in cols)
			Console.WriteLine(col(player));
}

```

```CSHARP
GetCsv3(Players,
    p => p.Name,                           // 📝 直接取姓名
    p => p.RegDate.ToString("yyyy-MM-dd"), // 📅 自訂日期格式
    p => p.Level,                          // 🆙 取等級
    p => p.Score                           // 🏆 取分數
);

// 輸出結果：
//Allen
//2024-01-01
//1
//255
//Mochi
//2022-12-21
//2
//32767
//Co
//2012-01-01
//99
//65535
```

| 優勢 | 實際效果 | 範例 |
|------|----------|------|
| 🛡️ **強型別保護** | 編譯期間就能發現錯誤，打錯屬性名稱會紅線警告 | `p => p.Naem` ❌ 編譯器直接報錯 |
| 🎨 **完全自訂格式** | 想要什麼格式就寫什麼邏輯 | `p => $"Lv.{p.Level}⭐"` |
| 🚀 **無數量限制** | 用 `params` 可以傳任意數量參數 | 1欄、10欄、100欄都沒問題 |
| ⚡ **零 Reflection** | 直接編譯成 IL 程式碼，效能極佳 | 比 Reflection 快數十倍 |
| 🧩 **完全泛型** | 可以處理任何類型，不只是 Player | `GetCsv3<Employee>()` 立即可用 |

```csharp
GetCsv3(Players,
    p => $"🎮 {p.Name}",                              // 加 emoji
    p => p.RegDate.Year > 2020 ? "新手" : "老鳥",         // 條件判斷  
    p => p.Level >= 50 ? "大神" : $"Lv.{p.Level}",       // 複雜邏輯
    p => string.Join("-", p.Score.ToString().ToCharArray()) // 分數格式化：65535 → 6-5-5-3-5
);
```

**這就是 Lambda Expression 的精髓：把『做什麼』的決定權交給呼叫端，把『怎麼做』的執行權留給方法！**


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/Lambda/5_4_lambda.png)


<!-- endtab -->

<!-- tab 在 Task.Run 放 Lambda（背景執行）-->

```csharp
// 背景執行一段 CPU 工作（示意）：
await Task.Run(() => 
{
    // 這裡可以是昂貴的運算或 I/O
    HeavyWork();
});

// 背景執行「本來就 async」的流程：
await Task.Run(async () =>
{
    var data = await httpClient.GetStringAsync(url);
    Save(data);
});
```

在 ASP.NET Core 的 Request Pipeline 內，一般不要把 I/O 轉去 Task.Run，直接 await I/O API（如 GetStringAsync）即可，避免不必要的 Thread 切換

<!-- endtab -->

<!-- tab 在 LINQ 裡塞 async Lambda-->

```csharp
var tasks = urls.Select(async url => await httpClient.GetStringAsync(url)); // 這裡得到 IEnumerable<Task<string>>
var contents = await Task.WhenAll(tasks);  // 正確等待所有請求
```

`Select(async ...)` 的結果是 `IEnumerable<Task<T>>`，要再用 `Task.WhenAll` 把它們「一起 await」。若是 `IAsyncEnumerable<T>`（C# 8），可以用 `await foreach` 搭配 `SelectAwait`（來自 `System.Linq.Async` 套件）處理真正的「逐項 async」流程


<!-- endtab -->

<!-- tab Parallel.ForEachAsync（.NET 6+）搭配 async Lambda-->

```csharp
await Parallel.ForEachAsync(urls, async (url, ct) =>
{
    var html = await httpClient.GetStringAsync(url, ct);
    await SaveAsync(url, html, ct);
});
```

這是「平行 + 非同步」的常見寫法，可用 `new ParallelOptions { MaxDegreeOfParallelism = 8 }` 控制併發


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/Lambda/7_async.png)


<!-- endtab -->

<!-- tab 🌸 多語系主題推薦頁-->


我們要處理的是賣場的主題推薦頁，推薦內容可能有三種類型

| 類型名稱         | 對應類別                | 說明           |
| ------------ | ------------------- | ------------ |
| 🎬 **影片推薦**  | `InfoModuleVideo`   | 顯示影片標題、說明等資訊 |
| 📰 **文字推薦**  | `InfoModuleArticle` | 顯示文章內容、摘要等   |
| 🖼️ **照片推薦** | `InfoModuleAlbum`   | 顯示相簿標題、張數等   |

<br>

## 🎯 問題背景

在編輯後台時，這三種推薦類型都會被封裝為同一個通用類別 `InfoModuleEditorPickEntity`
這樣做是為了統一管理，方便儲存與顯示，但接下來我們需要開發一個「多語系翻譯功能」，系統必須根據內容的類型（影片、文章、照片）去翻譯對應欄位，例如針對影片類型的資料，就要從多語系資料庫中抓出影片的 Title 與 Subtitle 翻譯


> 我們該如何建立一個共用的方法，能夠針對不同類型的內容拆解、翻譯，再覆蓋回原始資料？




![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/Lambda/8_infomodule.png)



如果用傳統寫法，你可能會寫出滿滿的 if 或 switch

```csharp
if (entity.Type == InfoModuleTypeEnum.Video) { /* ... */ }
else if (entity.Type == InfoModuleTypeEnum.Article) { /* ... */ }
```

## 用 Delegate + Lambda 表達式讓邏輯「可插拔」

我們設計一個通用的方法 GetMultilingualContent()，讓它能夠接收混合多種類型的清單，並透過 Lambda Expression 告訴方法「我要處理哪一種類型、該怎麼轉型、要取哪些欄位」

```csharp
public List<InfoModuleEditorPickEntity> GetMultilingualContent<T>(
	List<InfoModuleEditorPickEntity> infoModuleList,
	InfoModuleTypeEnum infoModuleType,
	MultilingualModuleTypeEnum multilingualModuleTypeEnum,
	Func<InfoModuleEditorPickEntity, T> selector, //// 告訴方法要怎麼轉型
	Func<T, int> idGetter, //// 告訴方法要怎麼取得主鍵 ID
	Func<T, string> titleGetter, //// 告訴方法要怎麼取得標題、副標題
	Func<T, string> subtitleGetter) where T : class
{
	// 1️⃣ 篩選出指定類型的資料
	var infoModuleEntities = infoModuleList
		.Where(entity => entity.Type == infoModuleType)
		.ToList();

	// 2️⃣ 轉成特定類型的資料（例如 InfoModuleVideoEntity）
	var specificEntityList = infoModuleEntities.Select(selector).ToList();

	// 3️⃣ 執行翻譯邏輯，取得翻譯後的內容
	var translations = MultilingualContentHelper.GetReplaceContentWithMultilingualContentCache(specificEntityList, multilingualModuleTypeEnum);

	// 4️⃣ 回填翻譯結果到原始資料清單
	foreach (var entity in infoModuleList)
	{
		if (entity.Type == infoModuleType)
		{
			//// 不能寫 .FirstOrDefault(idGetter(t) == entity.PubContentId); 因為 t 根本還不存在!
			var translation = translations.FirstOrDefault(t => idGetter(t) == entity.PubContentId);

			if (translation != null)
			{
				entity.Title = titleGetter(translation);
				entity.Subtitle = subtitleGetter(translation);
			}
		}
	}

	return infoModuleList;
}
```


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/Lambda/8_2_how_cast.png)


<!-- endtab -->


<!-- tab summary-->


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/Lambda/9_final.png)


<!-- endtab -->



{% endtabs %}