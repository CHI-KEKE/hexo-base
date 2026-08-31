---
title: Deep Clone
date: 2024-11-03 09:15:01
categories: 程思舞想
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/DeepClone/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/DeepClone/1_landing.png
tags:
    - C#
toc:
toc_number:
comments :
---


{% tabs Deep Clone - 1%}


<!-- tab 鏡中的自己-->

還記得小時候，我總覺得「鏡中的自己」很神奇。他會跟著我一起笑、一起生氣、一起模仿我的每個動作。但漸漸長大才明白——那並不是真正的「另一個我」，而只是一個淺薄的影子。

程式世界裡，物件的複製也是這樣。有時候，我們只是得到了「鏡子裡的自己」（Shallow Copy），稍不注意，動到這邊，另一邊也跟著被牽連。而有時候，我們會想要的是一個「能獨立走出鏡子」的自己（Deep Copy），擁有完全不同的記憶體、不同的命運。 


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/DeepClone/1_landing.png)


<!-- endtab -->

<!-- tab Shallow Copy & Deep Copy-->

**Shallow Copy** 只複製外層，裡面參考到的物件還是共用同一份
**Deep Copy** 會連內層參考物件一起複製，整包都變成新的


假設有這個類別


```csharp
public class Person
{
    public string Name { get; set; }
    public Address Address { get; set; }
}

public class Address
{
    public string City { get; set; }
}

var p1 = new Person
{
    Name = "Alex",
    Address = new Address { City = "Taipei" }
};

```

## Shallow Copy

```csharp
var p2 = p1;
p2.Name = "Bob";
p2.Address.City = "Kaohsiung";
```

 

結果 p1.Name 也可能受影響，因為 p2 = p1 根本是同一個物件，p1.Address.City 一定也變成 "Kaohsiung"

如果是用真正的淺拷貝概念，例如 `MemberwiseClone()`

```csharp
var p2 = p1.MemberwiseClone();

```

那會變成

p1 和 p2 是不同外層物件，但 p1.Address 和 p2.Address 還是指向同一個 Address

所以改這個
```csharp
p2.Address.City = "Kaohsiung";
```

p1.Address.City 也會一起變。外殼分開了，裡面還黏在一起


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/DeepClone/3_shallow.png)



## Deep Copy

```csharp
var p2 = new Person
{
    Name = p1.Name,
    Address = new Address
    {
        City = p1.Address.City
    }
};
```


這時 p1 和 p2 是不同物件，p1.Address 和 p2.Address 也是不同物件，所以改 `p2.Address.City = "Kaohsiung";`不會影響 `p1.Address.City`。外層跟內層都徹底拆開了。



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/DeepClone/4_deep_clone.png)


<!-- endtab -->


<!-- tab 透過序列化進行 Deep Copy-->


這招**把原物件整個「拆成資料」再重新組回來，藉此硬切斷原本物件之間共用記憶體的連結**，不用每個 class 自己寫 Clone()、一層一層手動 new、額外維護一大堆 mapping 設定，對「先解決問題」的價值很高。尤其物件結構很深時，手刻 Deep Copy 很容易漏欄位，反而更麻煩



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/DeepClone/5_json.png)


## 缺點：效能通常不漂亮

JSON Deep Copy 本質上是：掃描整個物件、轉成字串、解析字串、再建一次物件，這比直接複製欄位慢很多，也會多吃記憶體。所以若動作發生在

- 高頻率迴圈
- 大量資料處理
- API 熱路徑
- 遊戲或即時系統

那就很容易影響效能


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/DeepClone/5_json.png)


## 只能複製相同型別

DeepCloneByJson<Person>() 就只能還原成 Person，不能變成 PersonDto。不像 AutoMapper 可以 Person -> PersonDto


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/DeepClone/5_2_json_performance.png)


## Private 欄位無法複製

JSON 序列化只會處理「公開 (public) 的屬性」，私有欄位 (private) 不會被帶過去。所以如果物件內部有一些「隱藏的狀態」，Clone 出來的會失真。

多物件就像資料袋，實際上裡面可能有

- private 欄位
- 唯讀狀態
- 快取值
- 驗證旗標
- 建構子初始化出來的內部約束


## 循環參考會出錯

如果物件 A 裡面有 B，B 又有 A（互相參考），JSON.NET 會無限遞迴，造成 StackOverflow 或例外

```CSHARP
class Node {
    public Node Next { get; set; }
}
var node1 = new Node();
var node2 = new Node();
node1.Next = node2;
node2.Next = node1; // 循環參考
var clone = node1.DeepCloneByJson(); // ❌ 出錯

```

[循環參考錯誤解法](https://dotblogs.com.tw/wasichris/2015/12/03/152540)

<br>

## 實作擴充方法

```CSHARP
public static class CommonExtensions
{
	/// <summary>
	/// 深層複製
	/// </summary>
	/// <typeparam name="T">複製對象類別</typeparam>
	/// <param name="source">複製對象
	/// <returns>複製出的物件</returns>
	public static T DeepCloneByJson<T>(this T source)
	{
		if (Object.ReferenceEquals(source, null))
		{
			return default(T);
		}

        //// 確保反序列化時完全使用 JSON 中的資料，而不會與物件原本的預設值混合
		var deserializeSettings = new JsonSerializerSettings { ObjectCreationHandling = ObjectCreationHandling.Replace };
		return JsonConvert.DeserializeObject<T>(JsonConvert.SerializeObject(source), deserializeSettings); //// 這一步會把物件目前的 公開資料狀態 抽出來，變成純文字格式。
	}
}
```



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/DeepClone/5_3_jsonsys.png)




<!-- endtab -->

<!-- tab Automapper 進行 Deep Copy-->

AutoMapper 的強項是把資料搬到另一個物件上，不是保證把整個物件世界原封不動複製一份，AutoMapper 的設計目標本來就偏向 **projection / DTO mapping，不是 clone engine**


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/DeepClone/6_automapper.png)


## DeepClone 實作

```CSHARP
private static void AutoMappingDeepClone(IServiceProvider services)
{
    var source = new Entitys.Person();
    source.Address = "hisn";
    source.Age = 18;
    source.Name = "YL";
    source.Pets = new List<Entitys.Pet>
    {
        new Entitys.Pet { Name = "Party"},
        new Entitys.Pet { Name = "Mochi"}
    };

    var loggerFactory = services.GetRequiredService<ILoggerFactory>();
    var config = new MapperConfiguration(cfg =>
    {
        cfg.CreateMap<Entitys.Person, Entitys.Person>();
        cfg.CreateMap<Entitys.Pet, Entitys.Pet>();
    }, loggerFactory);

    var mapper = config.CreateMapper();
    var target = new Entitys.Person();
    mapper.Map(source, target);
    target.Age = 20;
    target.Address = "chaung";
    target.Name = "pan";

    System.Console.WriteLine($"Source: {source.Name}, {source.Age}, {source.Address}");
    System.Console.WriteLine($"target: {target.Name}, {target.Age}, {target.Address}");
}
```


1. 先建立 source，裡面有基本欄位與 Pets 集合。這代表要測的是一個有巢狀集合的物件，而不是只有平面欄位。

2. 建立 MapperConfiguration，註冊了

```csharp
cfg.CreateMap<Entitys.Person, Entitys.Person>();
cfg.CreateMap<Entitys.Pet, Entitys.Pet>();
```


這表示 AutoMapper 知道

```bash
Person -> Person
Pet -> Pet
```

該怎麼對映。AutoMapper 只能處理它已經知道的型別配對，所以 CreateMap 是必要前提。

3. `config.CreateMapper()` 建出 mapper。這就是實際執行 mapping 的工具

4. 建立一個新的 target

```csharp
var target = new Entitys.Person();
```

這一步很重要，因為現在不是把資料蓋回原物件，而是把資料搬進另一個新物件。

5. 執行 `mapper.Map(source, target);`

這會把 source 的資料按設定填進 target。對於巢狀成員，AutoMapper 會依照已註冊的型別映射繼續往下處理。對巢狀 mapping 與集合 mapping，官方文件都有明確說明。後面改 target 的 Age、Address、Name。這是在驗證基本欄位的修改不會回頭污染 source



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/DeepClone/6_2-automapper_how.png)


## 不須理會屬性的異動

AutoMapper 在設定 cfg.CreateMap<TSource, TDestination>() 時，只要兩個型別屬性名稱、型別相同，它就會自動幫你把值填進去，不需要你手動一個個設定。

<br>

## 可複製不同的型別內容
  
AutoMapper 不只限於相同型別，它也能幫你做「不同型別的對映」，只要你告訴它怎麼對應。這讓它非常靈活，比如你可能有個 ViewModel 或 Dto，要對應到 Entity，這時候用 AutoMapper 就超級好用。

```CSHARP
cfg.CreateMap<PersonDto, Person>()
   .ForMember(dest => dest.Name, opt => opt.MapFrom(src => src.FullName));
```

<br>

## 彈性配置對應方式

AutoMapper 它不是死板的，它可以讓你針對某些屬性，自訂對應邏輯，比如轉換格式、改名字、忽略欄位、加入條件等等。

```CSHARP
cfg.CreateMap<Person, Person>()
   .ForMember(dest => dest.Name, opt => opt.MapFrom(src => src.Name.ToUpper()))
   .ForMember(dest => dest.Address, opt => opt.Ignore());
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/DeepClone/67_3_auto_mapping.png)


## 維護成本

維護成本就會浮出來

- 屬性改名
- 型別改掉
- 巢狀物件新增但沒註冊 map
- 集合元素是另一個型別但沒配置
- 某欄位要忽略或轉格式

而且官方也提供 AssertConfigurationIsValid() 來驗證配置，這本身就代表 mapping 並不是「設一次永遠不用管」


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/DeepClone/6_4_maintainer.png)


<!-- endtab -->


<!-- tab summary-->


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/DeepClone/7_table.png)


回想一開始的故事，鏡中的自己只是一個「影子」。Shallow Copy 就像那個影子，你以為獨立，卻其實共用同一份靈魂。Deep Copy 則像是從鏡子裡真正走出來的「另一個自己」，
能走上屬於自己的路，擁有獨立的選擇。程式設計裡，選擇何時該用「影子」、何時該創造「新生命」，正如人生中，我們也常在模仿與創造之間，尋找真正的自我


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/DeepClone/7_2_final.png)


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Code_Design/DeepClone/deep_clone.png)



<!-- endtab -->



{% endtabs %}