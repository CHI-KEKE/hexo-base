---
title: Anonymous type
date: 2025-09-20 22:25:03
categories: 程思舞想
top_img: https://i.imgur.com/Zly1UDM.png
cover : https://i.imgur.com/Zly1UDM.png
tags:
    - 
toc:
toc_number:
comments :
---

{% tabs Anonymous type%}


{% btn 'https://blog.darkthread.net/blog/anonymous-type-to-json/',小技巧－使用匿名型別快速捏出指定JSON格式,far fa-hand-point-right %}

{% btn 'https://blog.darkthread.net/blog/anonymous-type-array-to-csv/',CODE-將匿名型別陣列匯成CSV,far fa-hand-point-right %}



<!-- tab Mob-->


![Mob](https://github.com/CHI-KEKE/pics/blob/main/csharp/mob.png?raw=true)

茂夫 被稱為 Mob，不是因為他弱，而是需要時總在場，不需要時悄悄退場。匿名型別（anonymous type）在 C# 裡就是這種感覺

在方法裡臨時出現、幫你捏 JSON、整理投影（Select）、帶著排序鍵巡邏、裝參數進 Dapper、丟幾個欄位進結構化日誌……做完事就退場，不佔版面、也不必為它立個牌位


![mob](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/mob_landing.png)


<!-- endtab -->

<!-- tab What is anonymous type?-->


匿名型別就像「沒有名字的人」。

```csharp
var person = new { Name = "Allen", Age = 30 };
```

這個 person 裡面確實有 Name 跟 Age 屬性，但它的型別是 編譯器臨時生出來的，當需要方法的回傳結果、宣告屬性/欄位 或當作參數的時候，程式需要一個明確的「型別名稱」來描述它。但匿名型別 沒有名字，所以沒辦法


```csharp
public ??? GetPerson() { ... } // ??? 不知道要寫什麼
```


## 唯讀屬性 

公開且唯讀的屬性(Immutable types)，屬性設定初始值後，無法再修改


## 屬性類型由編譯器推斷

```csharp
var person = new { Name = "Allen", Age = 30 };
// Name → string, Age → int
```


![name](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/anoy_feature.png)



![under](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/under_the_hood.png)


![noname](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/why_no_name.png)


<!-- endtab -->

<!-- tab 快速捏出 JSON-->


![json](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/quick_json_question.png)


有時候我們需要一個指定結構的 JSON，卻不想為此寫一個 class。這時候匿名型別就是好幫手。

```csharp
async void Main()
{
	TestAnoy("Karasuno", "vOLLEY", "Eng").Dump();
}


public static string TestAnoy(string name, string hobby, string job)
{
	var root = new
	{
		rows = new
		{
			row = new[]
			{
				new
				{
					name = name,
					hobby = hobby,
					job = job
				}
			}
		}	
	};

	return JsonSerializer.Serialize(root, new JsonSerializerOptions { WriteIndented = true});
}
```


![json](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/task1_json_shape.png)


<!-- endtab -->

<!-- tab 匯出 CSV-->


匿名型別也能應用在投射 (Select) 場景，快速產生需要的欄位，再匯出成 CSV。

```csharp
void Main()
{
  testtt();
}
public static void testtt()
{
	var arr = "allem,jill,willy".Split(",")
								.Select( (word,index) => new {Id = index, Name = word}).ToArray().Dump();							

	Type elemType = arr.GetType().GetElementType();
	PropertyInfo[] props = elemType.GetProperties();
	StringBuilder sb = new StringBuilder();
	sb.AppendLine(string.Join("\t",props.Select(p => p.Name).ToArray()));
	foreach(object ele in arr)
	{
		sb.AppendLine(string.Join("\t",
			props.Select(o => o.GetValue(ele, null).ToString())));
	}
	sb.ToString().Dump();
}
```


![export](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/task2_data_export_and_projection.png)


<!-- endtab -->

<!-- tab 結構化 Log -->


![log](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/log_question.png)


匿名型別特別適合「只挑幾個重要欄位」來記錄 log，而不需要建立新 DTO。

```csharp
var projection = new
{
    UserId = complexObject.UserId,
    EmailAddress = complexObject.EmailAddress,
    RegisteredAtUtc = complexObject.RegisteredAtUtc
};

logger.LogInformation("New user registered: {@NewUser}", projection);
```

這樣會輸出乾淨的 JSON 格式，對於結構化日誌 (Structured Logging) 特別有用。


![log2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/task3_logger_carrier.png)


<!-- endtab -->

<!-- tab LINQ 資料流 - 排序-->


![order](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/task5_LINQ_OrderKey_save_energy.png)


先計算排序鍵＋原物件包在匿名型別，排序後再取回原物件，避免重複計算。

```CSHARP
var ordered = people
    .Select(p => new { Person = p, Key = ExpensiveNormalize(p.LastName) })
    .OrderBy(x => x.Key)
    .Select(x => x.Person)
    .ToList();
```

排序鍵只算一次，效能更穩。中間匿名型別只是「暫時的載具」


![sequence](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/orderby_flow.png)



<!-- endtab -->

<!-- tab LINQ - GroupBy 的複合鍵（Composite Key）-->


![group](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/group_question.png)


用匿名型別做群組鍵，語法自然且有結構

```CSHARP
var groups = orders
    .GroupBy(o => new { o.Region, o.Category })
    .Select(g => new
    {
        g.Key.Region,
        g.Key.Category,
        Count = g.Count(),
        Total = g.Sum(x => x.Amount)
    });

foreach (var g in groups)
    Console.WriteLine($"{g.Region}/{g.Category}: {g.Count}件，合計{g.Total}");
```

匿名型別自帶成員值相等的相等性（value-based equality），很適合當 GroupBy 鍵。讀起來比手動建比較器好懂


![shape](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/task_6_composite_key.png)


<!-- endtab -->

<!-- tab Join / Left Join 後整形（Shaping）結果-->


把兩邊要的欄位「捏」成你要的形狀，不用為此造型別。

```csharp
var q =
    from o in orders
    join c in customers on o.CustomerId equals c.Id into gj
    from c in gj.DefaultIfEmpty() // Left Join
    select new
    {
        o.Id,
        o.OrderDate,
        CustomerName = c?.Name ?? "(未知客戶)"
    };

foreach (var x in q)
    Console.WriteLine($"{x.Id} {x.CustomerName} {x.OrderDate:d}");
```

Join 後的結果只要幾個欄位，匿名型別是最省力的方式。EF Core 會只抓到需要的欄位（投影）


![shape](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/task_7_result_shape.png)


<!-- endtab -->

<!-- tab 需要索引的中間運算-->


常見於產生行號、比對相鄰元素、分頁編號等。

```csharp
var lines = File.ReadAllLines("data.txt");

var withIndex = lines
    .Select((text, idx) => new { LineNo = idx + 1, Text = text })
    .Where(x => !string.IsNullOrWhiteSpace(x.Text));

foreach (var x in withIndex)
    Console.WriteLine($"{x.LineNo}: {x.Text}");
```


![index](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/task_9_index.png)


<!-- endtab -->

<!-- tab Distinct/Except/GroupBy 需要多欄位相等比較-->


匿名型別的相等性是結構相等（同屬性名稱與值都相等才相等），很好用來做去重或比較。

```csharp
var unique = products
    .Select(p => new { p.Sku, p.Color }) // 決定唯一性的鍵
    .Distinct()
    .ToList();
```


![dis](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/task_8_distinct.png)


<!-- endtab -->

<!-- tab Dapper 參數包（Param Bag）-->



Dapper 常用匿名型別傳參數，寫起來乾淨

```csharp
var rows = connection.Query(
    "SELECT * FROM Users WHERE Email = @Email AND IsActive = @Active",
    new { Email = email, Active = true }
);
```


![dapper](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/task4_dapper.png)


<!-- endtab -->

<!-- tab summary-->


![sum](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/sum_power.png)


![getback](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Anonymous/getback.png)


<!-- endtab -->


{% endtabs %}