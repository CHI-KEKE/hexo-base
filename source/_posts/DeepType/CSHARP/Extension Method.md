---
title: Extension Method
date: 2025-09-21 10:25:03
categories: 程思舞想
top_img: https://i.imgur.com/Zly1UDM.png
cover : https://i.imgur.com/Zly1UDM.png
tags:
    - 程思舞想
toc:
toc_number:
comments :
---


{% tabs Extension%}


{% btn 'https://old-oomusou.goodjack.tw/csharp/extension-method/',深入探討 C# 之 Extension Method,far fa-hand-point-right %}

{% btn 'https://www.huanlintalk.com/2009/01/csharp-3-extension-methods.html',C# 筆記：擴充方法,far fa-hand-point-right %}

{% btn 'https://medium.com/@lfilipecosta3/c-extension-methods-with-practical-use-cases-530948a8f8d9',C# — Extension methods with practical use cases,far fa-hand-point-right %}

{% btn 'https://stackoverflow.com/questions/71976566/what-does-the-extension-method-do-in-c-sharp-and-why-do-you-need-it',What does the extension method do in c# and why do you need it?,far fa-hand-point-right %}


<!-- tab Extension-->


## 語言層級的能力


讓我們在不修改原本 source code 的前提下，就能為 class 增加新 method，寫起來像 instance method，C# 編譯器幫你 static method + syntactic sugar


## 開放封閉原則（OCP）

對「不能改、也不該改」的型別開放擴充，例如 .NET Framework、第三方 package、legacy code


![ex](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Extension/new_lvie.png)


<!-- endtab -->

<!-- tab IEnumerable 擴充問題-->


```CSHARP
void Main()
{
	Enumerable.Range(1,3)
			  .Select(num => num*2)
			  .ForEach(num => Console.WriteLine(num));
}


internal static class Extension
{
	internal static void ForEach<T>(this IEnumerable<T> source, Action<T> action)
	{
		foreach(T item in source)
		{
			action(item);
		}
	}
}
```

其實在 IEnumerable<T> 上加 ForEach 是違反他的設計原則的，因為 IEnumerable<T> 的 LINQ 運算子屬於純查詢與延遲評估；ForEach 是副作用且會立刻消費序列，容易誤用在 DB/檔流/無限序列。更安全直覺的寫法是語言級 foreach，或明確 ToList().ForEach(...) 表示「我要物化後再逐一處理」

小抄：純計算的終端運算子（Sum/Count/Aggregate/Any/First）是 OK 的；但副作用請慎用擴充


![ie](https://github.com/CHI-KEKE/pics/blob/main/Code_Design/Extension/IE_Trap.png?raw=true)


<!-- endtab -->

<!-- tab 把「由右而左」改寫成「由左到右」-->


擴充方法可以把一連串轉換串成閱讀順序，像資料流在左到右地流動

```csharp
void Main()
{
	var person = new Person
	{
		LastName = "Lin",
		FirstName = "Allen"
	};
	
	person.LowerFullName().AppendDomain("google").Dump();
}


internal static class Extension
{
	internal static void ForEach<T>(this IEnumerable<T> source, Action<T> action)
	{
		foreach(T item in source)
		{
			action(item);
		}
	}
}


internal static class Email
{
	internal static string LowerFullName(this Person person) => person.FirstName.ToLower() + person.LastName.ToLower();
	internal static string AppendDomain(this string source, string domain) => $"{source}@{domain}.com";
}


public class Person
{
	public string LastName{get;set;}
	public string FirstName {get;set;}
}
```


![flow](https://github.com/CHI-KEKE/pics/blob/main/Code_Design/Extension/flow.png?raw=true)


<!-- endtab -->

<!-- tab 字串反轉-->


```csharp
void Main()
{
	"abcde".Reverse().Dump();
}

internal static class Extension
{
	internal static string Reverse(this string source)
	{
		if (string.IsNullOrWhiteSpace(source))
			return source;

		char[] newChars = new char[source.Length];
		var finalCharPoint = source.Length - 1; //// 真正的 finalPoint
		for(int i = 0; i <= finalCharPoint;i++) //// 從 0 開始
		{
			newChars[i] = source[finalCharPoint-i]; //// 一開始 - 0
		}
		
		return new string(newChars);
	}

    internal static string Reverse2(this string str)
	{
		char[] charArray = str.ToCharArray();
		Array.Reverse(charArray);
		return new string(charArray);
	}
}
```

可能我們會問，直接套用索引反轉不行嗎

```csharp
char[] charArray = str.ToCharArray();
Array.Reverse(charArray);
return new string(charArray);
```


兩個版本「其實犯的是同一個假設」 => 一個字元 = 一個 char，在 .NET 裡，這個假設 不永遠成立，例如 "abcde"
每個字都是 1 個 char，不管怎麼反轉都行，但Emoji 或特殊字元 😄 呢?實際上：😄 = 兩個 char（代理對 / surrogate pair），因此 string.Length == 3，反轉結果會變成 a + 半個 😄 + 半個 😄，編譯不會錯、程式不會 crash，但字壞了

問題在於可能切斷代理對（surrogate pair）或結合字元

![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Extension/string_is_surro.png)



## 最「語意正確」的反轉寫法

```csharp
using System.Globalization;

internal static class StringExtensions
{
    internal static string ReverseByTextElement(this string source)
    {
        if (string.IsNullOrEmpty(source))
            return source;

        var elements = StringInfo.GetTextElementEnumerator(source);
        var list = new List<string>();

        while (elements.MoveNext())
        {
            list.Add((string)elements.Current);
        }

        list.Reverse();
        return string.Concat(list);
    }
}

```

Text Element ≈ 人類認知的一個字，surrogate pair 不會被拆且 combining character 不會亂跑，因此emoji、重音字母都安全，而這是「語言層級」的反轉，不是儲存層級，但這樣寫的 tradeoff 就是效能比 char 反轉差，所以它不是 「知道風險後的選擇」


![re](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Extension/right_reverse.png)


<!-- endtab -->

<!-- tab 字串移除特定字元-->

技術上沒有增加新能力但設計上增加了「語意層」

```csharp
source.Replace(",", "");
```

讀到這行時，你腦中會想「為什麼要 replace？為什麼是逗號？這是在 format？還是在清洗？」

```csharp
void Main()
{
	"fewifjewofijokvoew,,,,,,ff".RemoveChar(",").Dump();
}


internal static class Extension
{
	internal static string RemoveChar(this string source, string intputChar)
	{
		return source.Replace(intputChar,"");
	}
}
```

「喔，他在移除某個字元」

![encap](https://github.com/CHI-KEKE/pics/blob/main/Code_Design/Extension/encap_meaning.png?raw=true)


<!-- endtab -->


<!-- tab DI 的語法糖：建立可讀的註冊 API-->

把一組有意義的註冊行為，包成一個可讀、可組合、可重用的註冊單位，IServiceCollection 的責任本來就是「註冊服務」，沒有幫它塞進奇怪的業務邏輯。Extension 只是補齊它原本就該有、但太瑣碎的操作

```csharp
namespace MySimpleCalculator.Extensions
{
    public static class CalculatorEx
    {
        public static IServiceCollection AddCalculatorService(this IServiceCollection services)
        {
            services.AddSingleton<ICalculator, Calculator>();

            return services;
        }
    }
}
```


![d](https://github.com/CHI-KEKE/pics/blob/main/Code_Design/Extension/DI.png?raw=true)


<!-- endtab -->


<!-- tab AntiPattern 1 - 綁死生命週期物件-->

當我們讓 Domain Model 自己「知道怎麼被存起來」，其實是在把資料庫的生命週期，強行灌進業務語意裡，Order 被迫知道 EF Core、Order 被迫依賴 DbContext 的生命週期、Save 的語意變成「用 EF 存進資料庫」，這已經是基礎設施入侵 Domain

```csharp
public static class OrderExtensions
{
    // ❌ 網域模型 + 基礎設施強耦合
    public static void Save(this Order order, DbContext db)
        => db.Set<Order>().Update(order);
}
```


![infra](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Extension/infra_invade.png)

<!-- endtab -->

<!-- tab AntiPattern 2 - IEnumerable<T> 上做副作用（寄信、寫檔、打 API）-->


```csharp
public static void SendEmail(this IEnumerable<User> users, IEmailSvc svc)
{
    foreach (var u in users) svc.Send(u.Email);
}
```


<!-- endtab -->

<!-- tab AntiPattern 3 - 擴充承載過多業務意義-->


IEnumerable<T> 的語意偏向「查詢/變換」。把副作用綁進去會立刻消費序列、難以推理（尤其來源是 DB/串流/無限序列）

```csharp
public static bool IsTaiwanId(this string id) { /* 一堆規則 */ }
```


DateTime.Age() 暗示的是「年齡是 DateTime 的自然屬性」，但實際上年齡 = 人 + 規則 + 今天

```csharp
void Main()
{
	var bir = new DateTime(1994,10,14);
	bir.Age().Dump();
}


internal static class Extension
{
	internal static int Age(this DateTime birthdate, string gender)
	{
        if (gender == "Female")
        {
            return 18;
        }

		var birDate = birthdate.Date;
		var today = DateTime.Today.Date;
		int years = today.Year - birDate.Year;
		//// 為什麼不用 TotalDays/365？ 閏年與 2/29 生日會有誤差；AddYears 校正是最佳實務。
		if (today < birDate.AddYears(years))
			years--;
			
		return Math.Max(years,0);
	}
}
```


![rise](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Extension/age_is_wrong.png)


<!-- endtab -->

<!-- tab AntiPattern 4 - 在 object 上加「萬能」擴充（例如 ToJson()）-->


把全域序列化策略硬塞進所有型別，還跟某個序列化器（Newtonsoft/System.Text.Json）耦合

```csharp
public static string ToJson(this object o)
    => JsonConvert.SerializeObject(o); // 綁特定實作
```

![tojson](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Extension/json_object.png)


<!-- endtab -->

<!-- tab AntiPattern 5 - 擴充方法包「重策略」：Retry、Timeout、Circuit Breaker-->


重試策略牽涉延遲、退避、可重入、可中止等細節；通用擴充容易語意不清/實作不完備。更好的設計是使用明確的策略元件（Polly 或自定義 Policy）

```csharp
public static async Task Retry(this Task task, int times) { /* 一堆通用重試 */ }
```

![r](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Extension/retry.png)

<!-- endtab -->


<!-- tab Summary-->


![r1](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Extension/decision.png)

![r2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Extension/summ.png)


<!-- endtab -->


{% endtabs %}