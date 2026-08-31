---
title: 讓他代我做：Delegate 應用實踐
date: 2024-11-10 07:33:00
categories: 風音子
top_img: https://i.imgur.com/zTSkblP.png
cover : https://i.imgur.com/zTSkblP.png
tags:
    - Delegate
toc:
toc_number:
comments :
---

在 [第一篇](https://allenlin000.github.io/2024/11/07/NEW-Delegate-1/)

我們從「什麼是委派（Delegate）」開始，理解它的誕生是為了解決什麼樣的需求：將行為作為參數傳遞。

接著[第二篇](https://allenlin000.github.io/2024/11/10/NEW-Delegate-2/)

我們進一步探討了實務上會遇到的挑戰，以及 .NET 框架提供的標準解法（像是泛型委派 Action、Func、Predicate）。

這次，我們要來真正動手，體會 Delegate 如何在日常開發中，帶來更高的彈性與更好的設計感。

## 🏪 動態選擇條件總結訂單資訊

**💬說明**

你有一組訂單資料 Order，客戶希望可以動態選擇條件，例如：

- 只加總金額超過 500 的訂單

- 只加總客戶名稱是 "Allen" 的訂單

- 加總指定客戶清單裡的人（比如 VIP 客戶）

請設計一個方法，讓使用者自訂篩選條件（以委派的方式傳入），然後針對符合條件的訂單做加總。


**解析**

先定義甚麼是 Order
```CSHARP

public class Order
{
	public int Id { get; set; }

	public string OrderId { get; set; }
	public string CustomerName { get; set; }
	public decimal Amount { get; set; }
	public string CustomerId { get; set; }
	public decimal TotalAmount { get; set; }
}

```

接著定義方法
```CSHARP

public class OrderFunc
{
	public static decimal SumOrder(IEnumerable<Order> orders, Predicate<Order> filter, Func<Order,decimal> selector)
	{
		return orders.Where(order => filter(order)).Sum(order => selector(order));
	}
}

```


使用時
```CSHARP

var orders = new List<Order>
{
	new Order { CustomerName = "John", Amount = 100, TotalAmount = 150 },
	new Order { CustomerName = "Mary", Amount = 200, TotalAmount = 250 },
	new Order { CustomerName = "Tom", Amount = 600, TotalAmount = 650 },
	new Order { CustomerName = "Allen", Amount = 800, TotalAmount = 900 },
};

OrderFunc.SumOrder(orders,order => order.Amount > 100, order => order.TotalAmount).Dump();


//// 1800

```

✅ 透過 Predicate 自訂篩選條件，Func 指定加總欄位，實現了「動態條件加總」！


## 🐎設計一個 Retry 的機制

**💬 說明**

Retry 是一個很常見的功能，舉凡

- Db 連線的 Retry
- Cache
- 三方 API 重試設計

今天我們簡單實作一個 RetryHelper，我們可以隨意傳入我們想要重試的方法

**🛠設計重點**

- 重試的框架是甚麼 ?
  - 怎麼重複執行? while loop
  - 怎麼正常結束？ break
  - Exception 後怎麼再來一次? try ... catch 接住, 繼續下一個 loop
  - 超過次數仍失敗後怎麼結束? 當 次數 > 最大重試次數, 拋出 Exception
- 重試幾次如何設計 ? 使用者傳入參數, 在 catch 中判斷是否繼續 while loop
- 重試過程要等待嗎 ? 使用這傳入參數, 在 catch 中判斷可重試後，塞入等待時間
- 在可接受次數的重試過程中，需要做什麼處理 (例如 : 記 log)** 在 catch 中判斷可重試後，記 log 標明重試次數與失敗原因

這是我們設計的 Retry 機制
```CSHARP

public static class FuncExtension
{
	public static async Task RetryAsync<TSource>(this Func<TSource,Task> doSomething, TSource arg,int maxRetries = 3, int delayMilliseconds = 300, Action<System.Exception,int> onRetry = null)
	{
		int attempt = 0;
		List<Exception> exceptions = new List<Exception>();
		while(true)
		{
			try{
				await doSomething(arg);
				break;
			}
			catch(Exception ex)
			{
				exceptions.Add(ex);
				if (++attempt >= maxRetries)
				{
					throw new AggregateException($"Operation failed after {maxRetries} attempts.", exceptions);
				}
				
				onRetry.Invoke(ex,attempt);
				await Task.Delay(delayMilliseconds);
			}
		}
	}
}

```

我們可以這樣來使用他
```CSHARP

Func<string, Task> func = async (string something) =>
{
	throw new Exception($"~~~~~{something}~~~");
};

await func.RetryAsync(
		arg: "Something",
		maxRetries: 3,
		delayMilliseconds: 500,
		onRetry: (ex, count) => Console.WriteLine($"Retry 第 {count} 次失敗：{ex.Message}")
	);

```

✅ 使用者只需要專注在要執行的操作，Retry 的邏輯完全抽離！

## 🛒 商品篩選系統


**💬 說明**
設計一個商品篩選系統，可以：

- 過濾商品（如價格篩選，小於某價格的商品）。
- 對符合篩選條件的商品執行特定操作（如列印商品資訊）。
- 計算每個商品的折扣後價格。


我們先定義 Product 有什麼性質
```CSHARP

public class Product
{
	public string Name { get; set; }
	public decimal Price { get; set; }
}

```

接著，我們想建立一個商品服務，本身帶有計算機、做某種行為、篩選行為，並設計一個方法應用他們
```CSHARP

public class ProductService
{
	private readonly Predicate<Product> _productFilter;
	private readonly Action<Product> _productAction;
	private readonly Func<Product,decimal> _productDiscountCalculator;
	private Dictionary<Product,decimal> _productMoneySaveDictionary = new Dictionary<Product,decimal>();

	public ProductService(Predicate<Product> productFilter, Action<Product> productAction, Func<Product, decimal> productDiscountCalculator)
	{
		_productFilter = productFilter;
		_productAction = productAction;
		_productDiscountCalculator = productDiscountCalculator;
	}
	
	public void CalculateAndTellMe(IEnumerable<Product> products)
	{
		foreach (var product in products)
		{
			// 先判斷篩選條件
			if (_productFilter(product))
			{
				// 如果還沒計算過折扣，計算並儲存
				if (!_productMoneySaveDictionary.ContainsKey(product))
				{
					_productMoneySaveDictionary[product] = _productDiscountCalculator(product);
				}

				// 對這個商品執行動作
				_productAction(product);
			}
		}
	}
}

```

應用方式如下
```CSHARP

	var products = new List<Product>
	{
		new Product { Name = "商品A", Price = 100 },
		new Product { Name = "商品B", Price = 200 },
		new Product { Name = "商品C", Price = 300 }
	};

	var service = new ProductService(
		p => p.Price > 100,
		p => Console.WriteLine($"買這個: {p.Name}, 折扣後價格: {p.Price * 0.8m}"),
		p => p.Price * 0.8m
	);

	service.CalculateAndTellMe(products);


//// 買這個: 商品B, 折扣後價格: 160.0
//// 買這個: 商品C, 折扣後價格: 240.0

```

✅ 資料流順暢地從篩選到計算再到執行行為，呼叫者只需定義細節，不必每次手刻流程！

這時你可能會問，這樣全部給呼叫端定義就好了啊，ProductService 不就是一個空殼嗎 ?

以這個例子來說，確實還不夠 "商品服務" 的感覺，主要只有以 CalculateAndTellMe 的方法來表現這件事情，因為
ProductService（內部流程框架）至少保證了一個商品處理流程「先篩選 → 計算折扣 → 再做行為」這個流程，外部人不需要每次都手寫篩選、計算、行為，流程混在一起。
當然我們可以擴充拆分成更多的商品服務功能

ProductService
│
├── FilterProducts()
├── CalculateDiscounts()
├── ExecuteAction()
├── BulkProcess()
├── GenerateReport()
├── SaveProcessedProducts()
├── ValidateProduct()
├── SortProducts()
└── ApplyPromotions()


所以有這個疑問是對的，我們再想一想 Delegate 的本質

✅ Delegate 本身不是為了省寫程式碼而存在的。
✅ Delegate 是為了讓你「保留流程控制權」，但又「開放細節彈性」而存在的。

這種設計特別適合，流程有標準節奏，但細節千變萬化的場景（像商品流程、訂單流程、外掛式設計），希望不同情境下注入不同邏輯，但核心框架穩定不動的場景


## ☘️ 結語

Delegate 的力量，不在於讓程式碼更短，而是讓變化可以被掌握，流程可以被守住。
它讓我們在設計時，清楚劃出界線：哪些邏輯需要彈性，哪些節奏必須穩定。在這個節奏之下，我們不只是在組裝功能，更是在建立一個能應對變化的系統骨架。
這就是 Delegate 真正的價值。