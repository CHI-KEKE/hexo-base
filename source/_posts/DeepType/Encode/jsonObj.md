---
title: 序列化魔法：從記憶體城堡到數位信件
date: 2025-07-14 08:53:11
categories: 暗号者の秘密の部屋
top_img: https://i.imgur.com/ltxqyEt.png
cover : https://i.imgur.com/ltxqyEt.png
tags:
    - Encode
toc:
toc_number:
comments :
---


{% tabs jsonObject%}

<!-- tab Serialization&Deserialization-->

在數位世界中，**序列化（Serialization）** 和 **反序列化（Deserialization）** 就像是一對魔法師，負責在不同的資料形式之間進行轉換。想像一下，您有一個複雜的 C# 物件，裡面包含各種屬性、巢狀結構，甚至是陣列——這些資料存在於程式的記憶體中，就像是一座精美的城堡，但這座城堡只有您的程式看得懂


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Encode/JObj/1_freeze_and_resume.png)


<!-- endtab -->

<!-- tab 從記憶體城堡到數位信件-->

當您需要將這些資料傳送給其他系統，或是儲存到檔案中時，就需要將它們「打包」成一種通用的格式。這就是序列化的魔法：

```csharp
// 原始的 C# 物件（記憶體中的城堡）
var player = new Player 
{
    Name = "勇者艾倫",
    Level = 99,
    Equipment = new List<string> { "神劍", "龍鱗甲", "魔法戒指" }
};

// 序列化：將城堡變成可傳遞的信件
string json = JsonSerializer.Serialize(player);
// 結果："{"Name":"勇者艾倫","Level":99,"Equipment":["神劍","龍鱗甲","魔法戒指"]}"
```


<!-- endtab -->

<!-- tab 狀態的保存與重生-->

序列化的核心概念是 **狀態保存與還原**：

- **狀態保存**：將物件的所有屬性值、結構關係完整地「凍結」在文字格式中
- **狀態還原**：從文字格式中「復活」出一模一樣的物件結構

<!-- endtab -->

<!-- tab 一一對應的映射關係-->

序列化和反序列化的基礎是 **一一對應的映射關係**：

- **物件屬性** ↔ **JSON 鍵值對**
- **陣列集合** ↔ **JSON 陣列**
- **巢狀物件** ↔ **JSON 物件嵌套**

這種對應關係確保了資料在轉換過程中不會遺失，就像是一把精確的鑰匙，能夠完美地開啟和鎖定資料的每一個角落。

<!-- endtab -->

<!-- tab 兩大 JSON 操作流派-->

在 .NET 世界中，有兩大主要的 JSON 操作流派，各自擁有獨特的魔法工具：

## 📜 Newtonsoft.Json 的古典魔法
- **JObject**：操作 JSON 物件的經典工具
- **JArray**：處理 JSON 陣列的傳統方式
- **靈活性高**：動態操作、豐富的 API

## ⚡ System.Text.Json 的現代魔法
- **JsonObject**：現代化的物件操作工具
- **JsonArray**：高效能的陣列處理
- **效能優化**：更快的序列化速度、更少的記憶體使用


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Encode/JObj/2_libraries.png)


## 為什麼會使用這類東西?

1. 欄位要不要加是執行期才知道

```csharp
// 匿名型別做不到，欄位在編譯期就固定了
var body = new JObject();
body["merchant_id"] = merchantId;

if (hasDiscount)
    body["discount_code"] = code;  // 有折扣才加這個欄位
```


2. 迴圈動態加欄位
```csharp
var body = new JObject();
foreach (var kv in userSelectedFilters)
    body[kv.Key] = kv.Value;

```

## 串接第三方 API，但你不知道回傳結構

<!-- endtab -->

<!-- tab Newtonsoft.Json 的古典魔法：JObject 與 JArray-->

在 Newtonsoft.Json 的世界中，`JObject` 和 `JArray` 是兩位經驗豐富的魔法師，它們提供了極為靈活的 JSON 操作能力：

### 🏛️ JObject：物件操作大師

```csharp
// 建立一個 JObject
var playerData = new JObject
{
    ["Name"] = "勇者艾倫",
    ["Level"] = 99,
    ["IsActive"] = true
};

// 動態新增屬性
playerData["Experience"] = 1500000;
playerData["LastLogin"] = DateTime.Now;

// 巢狀物件
playerData["Stats"] = new JObject
{
    ["Strength"] = 85,
    ["Magic"] = 92,
    ["Defense"] = 78
};

// 讀取屬性值
string name = (string)playerData["Name"];
int level = (int)playerData["Level"];
```

### 📚 JArray：陣列操作專家

```csharp
// 建立一個 JArray
var equipment = new JArray("神劍", "龍鱗甲", "魔法戒指");

// 新增元素
equipment.Add("治癒藥水");
equipment.Insert(0, "傳說盾牌");

// 移除元素
equipment.Remove("魔法戒指");

// 結合到 JObject 中
playerData["Equipment"] = equipment;

// 遍歷陣列
foreach (var item in equipment)
{
    Console.WriteLine($"裝備：{item}");
}
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Encode/JObj/2_newton_flex.png)


<!-- endtab -->

<!-- tab System.Text.Json 的現代魔法：JsonObject 與 JsonArray-->

現代的 System.Text.Json 提供了更高效能的操作方式：

## 🔮 JsonObject：現代物件操作

```csharp
// 建立 JsonObject
var jsonObject = new JsonObject()
{
    ["Id"] = "A001",
    ["Name"] = "Jeffrey",
    ["Extra"] = "Prop To Remove",
    ["Equipments"] = new JsonArray("Shield", "Sword", "Bottle")
};

// 動態指定屬性
jsonObject["Score"] = 32767;

// 加入物件屬性
jsonObject["Pet"] = new JsonObject()
{
    ["Name"] = "Spot",
    ["Exp"] = 255
};

// 移除屬性
jsonObject.Remove("Extra");
```

## 🌟 JsonArray：高效陣列處理

```csharp
// 操作 JsonArray
var jsonArray = jsonObject["Equipments"]!.AsArray();

jsonArray.Remove(jsonArray.Single(j => j?.GetValue<string>() == "Bottle"));
jsonArray.Insert(0, "Mojiiii");

// 轉換回 JSON 字串
var jsonString = jsonObject.ToJsonString(new JsonSerializerOptions
{
    WriteIndented = true
});
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Encode/JObj/2_strict.png)


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Encode/JObj/3_textJson_remove_insert.png)



<!-- endtab -->

<!-- tab Newtonsoft.Json：經典範例-->

```csharp
// 複雜型別的反序列化
var payTypeExpressInfo = JsonConvert.DeserializeObject<PayTypeExpressCreditCardEntity<PayTypeExpressInfoForStripeEntity>>(data.PayTypeExpressInfo);

// 泛型類別定義
public class PayTypeExpressCreditCardEntity<T>
{
    /// <summary>
    /// 發卡銀行
    /// </summary>
    public string Issuer { get; set; }

    /// <summary>
    /// 發卡組織
    /// </summary>
    public string Association { get; set; }

    /// <summary>
    /// 卡號
    /// </summary>
    public string No { get; set; }

    /// <summary>
    /// 有效月份
    /// </summary>
    public string Month { get; set; }

    /// <summary>
    /// 有效年份
    /// </summary>
    public string Year { get; set; }

    /// <summary>
    /// 擴充資訊
    /// </summary>
    public T ExtendInfo { get; set; }
}
    
      public class StripeCreditCardInfoEntity
    {
        /// <summary>
        /// 發卡國家
        /// </summary>
        [JsonProperty("country")]
        public string CountryAliasCode { get; set; }

        /// <summary>
        /// 卡別
        /// </summary>
        [JsonProperty("brand")]
        public string Brand { get; set; }
    }
    
```


<!-- endtab -->

<!-- tab Test.Json-->

```csharp
var entity = JsonSerializer.Deserialize<AuditRewardLoyaltyPointsPromotionRuleRecordJobTaskData>(taskData);
```

###  以JsonObject操作物件

```csharp
var jsonObject = new JsonObject()
{
	["Id"] = "A001",
	["Name"] = "Jeffrey",
	["Extra"] = "Prop To Remove",
	["Equipments"] = new JsonArray("Shield", "Sword", "Bottle")
};

// 動態指定屬性
jsonObject["Score"] = 32767;

// 加入物件屬性
jsonObject["Pet"] = new JsonObject()
{
	["Name"] = "Spot",
	["Exp"] = 255
};


// 移除屬性
jsonObject.Remove("Extra");

// 操作JsonArray
var jsonArray = jsonObject["Equipments"]!.AsArray();

jsonArray.Remove(jsonArray.Single(j => j?.GetValue<string>() == "Bottle"));
jsonArray.Insert(0,"Mojiiii");


// 把jsonObject變回jsonString
var jsonString = jsonObject.ToJsonString(new JsonSerializerOptions
{
	WriteIndented = true
});

// 把jsonObject Pase成JsonNodes在轉回Object
var restored = JsonNode.Parse(jsonString)!.AsObject();

foreach(var prop in restored)
{
	var pn = prop.Key;
	if(prop.Value is JsonObject)
		$"{pn} is JsonObject".Dump();
	else if (prop.Value is JsonArray)
		$"{pn} is JsonArray, length={prop.Value.AsArray().Count()}, 內容 {prop.Value}".Dump();
	else if(prop.Value?.AsValue().TryGetValue<int>(out int i) ?? false)
		$"{pn} is Int, VALUE={i}".Dump();
}

var avatar = System.Text.Json.JsonSerializer.Deserialize<Avatar>(jsonString);

if (avatar != null)
{
	Console.WriteLine($"Id: {avatar.Id}, Name: {avatar.Name}, Score: {avatar.Score}");

	if (avatar.Pet != null)
	{
		Console.WriteLine($"Pet Name: {avatar.Pet.Name}, Exp: {avatar.Pet.Exp}");
	}

	if (avatar.Equipments != null)
	{
		Console.WriteLine("Equipments: " + string.Join(", ", avatar.Equipments));
	}
}


public class Avatar
{
	public string Id { get; set; }
	public string Name { get; set; }
	public int Score { get; set; }
	public List<string> Equipments { get; set; }
	public Pet Pet { get; set; }
}

public class Pet
{
	public string Name { get; set; }
	public int Exp { get; set; }
}
```

<!-- endtab -->

<!-- tab NewtonSoft vs Text.Json-->

```csharp
void Main()
{
	var jsonOpt = new JsonSerializerOptions()
	{
		WriteIndented = true
	};
	
	var dict = new Dictionary<string,object>()
	{
		["iii"] = 255,
		["sss"] = "StringTest",
		["ddd"] = DateTime.Today,
		["aaa"] = new int[] { 1, 2, 3 },
		["ooo"] = new { Prop = 123 },
		["g"] = Guid.NewGuid(),
		["n"] = null!
	};
	
	var json = JsonSerializer.Serialize(dict, jsonOpt);
	
	
	"=====NewtonSoft=====".Dump();
	var newTonObject = Newtonsoft.Json.JsonConvert.DeserializeObject<Dictionary<string,object>>(json);
	foreach(var kvp in newTonObject)
	{
		$"Result Value : {kvp.Value?.GetType().Name}, KVP : {kvp.Key} = {kvp.Value}".Dump();
	}
	
	//// System.Text.Json 在反序列化 object 時不像 Json.NET 會試著轉型成 int、string、DateTime 等型別
	//// Json.NET 一律視為 JsonElement 雖然 JsonElement 提供了 GetByte()、GetGuid()、GetInt32()、GetDateTime()... 等方法 但得逐 Key 分別處理
	"=====TextJson=====".Dump();
	var textJsonObject = JsonSerializer.Deserialize<Dictionary<string, object>>(json);
	foreach (var kvp in textJsonObject!)
	{
		$"Result Value : {kvp.Value?.GetType().Name}, KVP : {kvp.Key} = {kvp.Value}".Dump();
	}
	
	"=====TextJson With Extension=====".Dump();
	var dSysTextJson = JsonSerializer.Deserialize<JsonObject>(json)!.ToStringObjectDictionary();
	foreach (var kv in dSysTextJson!)
	{
		Console.WriteLine($"{kv.Value?.GetType().Name} {kv.Key} = {kv.Value ?? "null"}");
	}
}

//// 先尻一個擴充函式 ToStringObjectDictionary() 
public static class JsonDictStringObjExtensions
{
    public static Dictionary<string, object> ToStringObjectDictionary(this JsonObject jsonObject)
    {
        var dict = new Dictionary<string, object>();
        foreach (var prop in jsonObject) 
        {
            object value;
            if (prop.Value == null) value = null!;
            else if (prop.Value is JsonArray) value = prop.Value.AsArray();
            else if (prop.Value is JsonObject) value = prop.Value.AsObject();
            else 
            {
                var v = prop.Value.AsValue();
                var t = prop.Value.ToJsonString();
                if (t.StartsWith('"')) {
				if (v.TryGetValue<DateTime>(out var d)) value = d;
				else if (v.TryGetValue<Guid>(out var g)) value = g;
				else value = v.GetValue<string>();
			}
			else value = v.GetValue<decimal>();
		}
		dict.Add(prop.Key, value);
	}
	return dict;
	}
}
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Encode/JObj/3_textjson_jsonelement.png)


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Encode/JObj/4_tostringobjectdictionary.png)

<!-- endtab -->

<!-- tab 空 list 序列化後還是 []-->

空的 list 在序列化（serialize）之後，為什麼依然會是 []（而不是 null、""、或直接不見）。因為 [] 代表「我有這個欄位，而且它是一個集合，只是目前沒有元素」——這個訊息比「沒有欄位」或「null」清楚，前後端才不會各自腦補。

## 購物車：使用者購物車目前沒有商品

cartItems: [] 很直觀：購物車存在，但是空的，如果是 null：是購物車不存在？還是後端出錯？但如果欄位不見：前端可能以為後端沒回、或版本不相容

## 篩選條件：你傳一個 tags 給 API

tags: [] 表示「不要用 tag 篩選」，欄位不見可能被後端解讀成「用預設 tag 篩選」


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Encode/JObj/5_empty_and_null.png)


<!-- endtab -->

<!-- tab Refit 的 api_response 無法 deserialze 原因探討 -->

Refit 其實不是「不會反序列化」，而是你以為拿到的是 JSON，但實際回來的常常是 空 body、非 JSON、或 JSON 形狀跟你的型別對不上，序列化器只能誠實爆炸

他會看方法回傳型別決定錯誤處理策略

- 回 Task<T>：反序列化或 HTTP 錯誤通常會直接丟例外
- 回 Task<ApiResponse<T>>：Refit 會把 HTTP/反序列化相關例外「包進 ApiResponse.Error」，不一定會直接 throw（所以可能以為只是 Content 是 null）。

接著挑選 ContentSerializer（System.Text.Json 或 Newtonsoft）做 response.Content → T，任何一個點不符合預期就失敗:

- body 根本沒有內容（空字串、204、某些 202）
- Content-Type 不是 JSON（回 HTML 錯誤頁、純文字）
- JSON 結構跟 T 不相容（object/array 搞反、欄位型別不對）
- serializer 設定更嚴格（System.Text.Json 常比 Newtonsoft 嚴）


## API 回 204 No Content，但你宣告 ApiResponse<MyDto>


JsonException: The input does not contain any JSON tokens...
因為 body 是空的，serializer 沒東西可解析

## API 出錯時回 HTML（例如 nginx / IIS 的錯誤頁）

Unexpected character encountered while parsing value: < 這種錯，因為你我們期待 JSON，結果拿到 <html>...

## T 期待 array，但實際回 object

會看到「應該是 JSON array」或型別不合的錯誤


![b](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Encode/JObj/6_refit_api.png)



<!-- endtab -->

<!-- tab summary-->


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Encode/JObj/7_compare.png)


<!-- endtab -->


{% endtabs %}