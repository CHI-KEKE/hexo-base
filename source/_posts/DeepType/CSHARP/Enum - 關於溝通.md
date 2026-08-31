---
title: Enum - 關於溝通
date: 2024-04-29 22:45:10
categories: 程思舞想
top_img: https://i.imgur.com/Zly1UDM.png
cover : https://i.imgur.com/Zly1UDM.png
tags:
    - Enum
toc:
toc_number:
comments :
---

{% tabs Enum - 關於溝通%}


<!-- tab Communication-->

阿罵 : 要吃水果嗎
我 : 不用

...

阿罵 : 要吃蘋果嗎
我 : 不用，謝謝阿罵
...

阿罵 : 要吃芭樂嗎
我 : 不要

...

阿罵 : 要吃蘋果還是芭樂
我 : (｡ŏ_ŏ)

...
...
...
...
<br>

我們是否也曾在溝通上感到無力 ? 是否想過為甚麼阿罵總是覺得你要吃水果 ?
其實你知道阿罵只是想跟你聊聊天、想跟你有更多的互動
你的拒絕也不是討厭水果甚至討厭阿罵，你只是當下沒有足夠的耐心與阿罵交流，或是某種程度上是因為溝通成本太高

<br>

溝通，是一門藝術，更是一種選擇。
人生中最痛苦的事之一，莫過於你以為你說得很清楚，對方卻完全接收不到對應的資訊。
這樣的錯頻，在程式世界裡也屢見不鮮。
你明明送出了一個清楚的 StatusEnum.Full，卻被序列化成一個看似陌生又難以理解的 2。
又或是，在情感世界裡，你說「我很好」，但其實你需要的是「你懂我」。
真正的溝通，從來不是你說了什麼而已，而是雙方是否都「懂了」。

![Image](https://i.imgur.com/mEGn5FQ.png)

<!-- endtab -->

<!-- tab JSON 序列化與反序列化-->

假設我們今天定義訂單的結構

```CSHARP

public enum StatusEnum
{
	WaitingToPay,
	Processing,
	Finish,
	Cancelled
}

public class Order
{
	public int Id { get; set; }
	public StatusEnum Status {get;set;}
}

```

接著我們將實體序列化後打 API，發現 API 怎麼都打都不通是怎樣，Debug 時印出 payload 看看..

```CSHARP

var json = JsonSerializer.Serialize(order);
json.Dump();

//// {"Id":1,"Status":2}

```

你傳了 2 過去，你傳了 2 過去! 為什麼?

實際上 Enum 背後仍然是數值，要在序列化與反序列化之間做一些特殊處理才能自然之然的傳出 Member Name


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/1_enum_expect.png)


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/1_enum_serialize_to_2.png)



接著，我們試驗反序列化

```CSHARP

var json = "{\"Id\":1,\"Status\":\"Finish\"}";
var order = JsonSerializer.Deserialize<Order>(json);
order.Dump();

```

結果直接噴掉，因為他找不到對應的型別處理 (Text.Json)
![Image](https://i.imgur.com/nWBKx2M.png)


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/2_fail_enum_case.png)



解決這個問題，Json 有出一些工具，解決了傳資料的困擾

- {% label NewtonSoft.Json %} 提供的 Attribute
```CSHARP
[JsonConverter(typeof(StringEnumConverter))]
```


- {% label Text.Json %} 提供的 Attribute
```CSHARP
[JsonConverter(typeof(JsonStringEnumConverter))]

```
服用方式就是掛在 Property 上 (以 Text.Json 為例)
```CSHARP

public class Order
{
	public int Id { get; set; }

	[JsonConverter(typeof(JsonStringEnumConverter))]
	public StatusEnum Status {get;set;}
}

```

藥效絕佳!
![Image](https://i.imgur.com/dYOpEa9.png)


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/3_jsonenum,convertor.png)


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/3_jsonenumconvertor_effective.png)


<!-- endtab -->

<!-- tab 1️⃣ 反序列化時的大小寫處理-->

```CSHARP

var json = "{\"Id\":1,\"Status\":\"fInish\"}";

```
結果是成功的，也就是大小寫問題預設都會放寬限制
![Image](https://i.imgur.com/XLRfejE.png)


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/4_case_insensitive.png)


<!-- endtab -->

<!-- tab 2️⃣ 字串與數字混用的解析-->


```CSHARP

var json = "{\"Id\":1,\"Status\":\"2\"}";

```
結果是成功的，也就是用數字也沒問題
![Image](https://i.imgur.com/EraXEBO.png)


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/5_number_include.png)


<!-- endtab -->

<!-- tab 3️⃣ 無效值的處理（Invalid Enum Value）-->

```CSHARP

var json = "{\"Id\":1,\"Status\":\"ABC\"}";

```


![Image](https://i.imgur.com/UoJu30s.png)
![Image](https://i.imgur.com/qmLhKvn.png)


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/6_enum_dont_understand.png)


<!-- endtab -->

<!-- tab 4️⃣ 反序列化不指定資料呢-->

```CSHARP

var order2 = new Order()
{
	Id = 2,
};


var json2 = JsonSerializer.Serialize(order2);
json2.Dump(); // {"Id":2,"Status":"WaitingToPay"}

```
結果就是抓預設值，因為它屬於值類型的型別


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/6_enum_dont_understand.png)


<!-- endtab -->

<!-- tab 5️⃣ NewtonSoft 支援 SnakeCase-->

```CSHARP

var json = "{\"Id\":1,\"Status\":\"waiting_to_pay\"}";

public class Order
{
	public int Id { get; set; }

	[Newtonsoft.Json.JsonConverter(typeof(StringEnumConverter),typeof(SnakeCaseNamingStrategy))] //// NewtonSoft 系統
	public StatusEnum Status {get;set;}
}

```

結果如下
![Image](https://i.imgur.com/TE68OiB.png)

看到這你可能會想，乾 他會幫我切怎樣是正確的英文單字嗎?
當然不是，他會認 "大寫單字"，也就是 PascalCase 的概念


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/8_naming_strategy.png)


<!-- endtab -->

<!-- tab wrap-->

你以為自己說得夠清楚：「我不用吃水果」，但阿罵聽到的可能是：StatusEnum = WaitingToBeLoved。
我們回傳的是 "Finish"，結果傳出去的是 2，也不難理解為什麼阿罵期待期待可以再多塞一點

我們努力掛上 [JsonConverter]，希望程式能更懂人話，但我們卻常常忘了，真實生活中沒有人幫我們轉格式，阿罵沒有 NamingStrategy，卻總試圖理解你這個版本的「狀態碼」。人生就像是個沒有 schema 的 API，每個人都用自己的版本發 request，也期待對方能正確 decode。

如果我們願意像寫程式那樣，多花一點時間處理錯誤、多加一些備援邏輯、也許就能讓那些「我不要吃水果」但我還是想來看看你的背後訊息，成功被解析成`阿罵，其實我也想你了`

<!-- endtab -->


{% endtabs %}




