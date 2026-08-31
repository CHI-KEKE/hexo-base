---
title: BaseAddress - 那一撇，決定了命運的方向
date: 2024-05-20 22:25:05
categories: 穿越雲層的信使
top_img: https://i.imgur.com/hZWviQz.png
cover : https://i.imgur.com/hZWviQz.png
tags:
    - Http
toc:
toc_number:
comments :
---

![Image](https://i.imgur.com/VlnAZYi.png)

這是一隻蛇蛇。突發奇想想用 / 畫出來當開場，欸還真不錯，看著看著竟有點可愛（？）

## 🐍 寄錯的明信片

有時候，一行程式碼，就像一張寄出的明信片。

你滿懷誠意地寫下問候、地址、郵遞區號，甚至還貼上特別版的郵票，只為讓那封來自心底的訊息，抵達你想念的那個人手上。

但你忘了一件事──地址格式要「符合規格」。

就像 HttpClient 看著那個不完整的 BaseAddress，露出一種：「呃，好啦我自己猜看看好了」的尷尬表情。
於是它猜錯了，訊息雖然出發了，但永遠沒有人收到。


## 🐍 建立實驗

讓我們來實驗一下 BaseAddress 設定錯誤會發生甚麼事吧!

### 1.Autofac 註冊HttpClient

```csharp


public class ServiceModule : Module
{
	protected override void Load(Autofac.ContainerBuilder builder)
	{
		builder.RegisterType<BoredHttpClient>()
			.As<IBoredHttpClient>()
			.SingleInstance()
			.WithParameter((info, _) => info.Name == "httpClient", (_, ctx) =>
			{
				HttpClient client = HttpClientFactory.Create();
				client.BaseAddress = new Uri("https://www.boredapi.com/api");
				client.Timeout = TimeSpan.FromSeconds(30);
				return client;
			});
	}
}

```

### 2.BoredHttpClient

```csharp

public class BoredHttpClient : IBoredHttpClient
{
	private readonly HttpClient _httpClient;

	public BoredHttpClient(HttpClient httpClient)
	{
		_httpClient = httpClient;
	}

	public string GetBored()
	{
		const string url = "/activity";

		return this._httpClient.GetStringAsync(url).GetAwaiter().GetResult();
	}
}

```

### 3.TestController

```csharp

[ApiController]
[Route("[controller]")]
public class TestController : ControllerBase
{
	private readonly  IBoredHttpClient _boredHttpClient;

	private readonly ILogger<TestController> _logger;

	public TestController(ILogger<TestController> logger, IBoredHttpClient boredHttpClient)
	{
		_logger = logger;
		_boredHttpClient = boredHttpClient;
	}

	[HttpGet(Name = "Bored")]
	public string GetBored()
	{
		return _boredHttpClient.GetBored();
	}
}

```

### 4.測試

![Image](https://i.imgur.com/yxcHhbi.png)

嗯...跟想像的不太一樣

仔細研究一番後，發現問題出在這

![Image](https://i.imgur.com/DQ5bayF.png)

立馬實驗看看

![Image](https://i.imgur.com/3QskWoI.png)

aseAddress 後面的 api 最後沒補上 / ! 最後的 "api" 會被切掉!

![Image](https://i.imgur.com/TOhXafJ.png)

## 🐍 修正

1. 補上 forward slash

```csharp

protected override void Load(Autofac.ContainerBuilder builder)
{
	builder.RegisterType<BoredHttpClient>()
		.As<IBoredHttpClient>()
		.SingleInstance()
		.WithParameter((info, _) => info.Name == "httpClient", (_, ctx) =>
		{
			HttpClient client = HttpClientFactory.Create();
			client.BaseAddress = new Uri("https://www.boredapi.com/api/");
			client.Timeout = TimeSpan.FromSeconds(30);
			return client;
		});
}

```

2. 確認調整後的結果
![Image](https://i.imgur.com/VxAW14V.png)


## 🐍 官方文件說了什麼？

根據微軟官方文件，BaseAddress 的結尾如果沒有 /，後續的 path 會覆蓋掉它，而不是接在它後面。這遵循 RFC 3986 的 URI 合併規範。

![Image](https://i.imgur.com/IsKxlCX.png)

## 🐍 思考一下背後的意義

我們來看看為什麼要這樣設計：

1. URI 路徑分段的清晰性
   
當 URI 結尾沒有 "/" 時，代表這是一個具體的資源位置
當 URI 結尾有 "/" 時，代表這是一個目錄或命名空間

```plaintext

https://api.example.com/v1      # 代表 v1 這個具體資源
https://api.example.com/v1/     # 代表 v1 目錄下的資源集合

```

也就是說 BaseAddress + 後綴  可以看成 倉庫 + 哪一個資源 的概念

而如果 BaseAddress 結尾沒有 "/"

系統會認為這不是一個倉庫（目錄），而是一個具體的東西，所以後面的路徑會直接取代它，這就像是把「台北」當成了商品而不是倉庫位置。
這就是為什麼在表示「容器」或「目錄」概念的 BaseAddress 最後要加上 "/"

![Image](https://i.imgur.com/95hZeE2.png)

## 🐍 結語

我們總以為自己已經很細心，但有時候出錯的不是邏輯，而是那個**「你不以為意的小細節」**。

HttpClient 的這個小坑，就像人生裡那些：

沒注意報名表寫上名字

專案簡報沒轉檔結果排版全炸

你不是不夠努力，你只是少了一撇。

下次，記得補上那個 /

那一撇，決定了你 API 的命運。有時候，也決定了你的。

## 其他參考資料

[黑暗執行續](https://blog.darkthread.net/blog/httpclient-baseaddress/#google_vignette)
[MicroSoft Developer Community](https://developercommunity.visualstudio.com/t/httpclient-not-smart-about-combining-baseaddress-w/1592519#T-N1645173-N1647703)
[StackOverFlow](https://stackoverflow.com/questions/23438416/why-is-httpclient-baseaddress-not-working)