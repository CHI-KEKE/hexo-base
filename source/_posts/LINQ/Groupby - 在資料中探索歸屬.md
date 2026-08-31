---
title: Groupby - 在資料中探索歸屬
date: 2024-03-17 13:39:34
categories: LINQ
top_img: https://i.imgur.com/cBfeEDY.png
cover : https://i.imgur.com/cBfeEDY.png
tags:
    - LINQ
toc:
toc_number:
comments :
---

![Image](https://i.imgur.com/Asp1OtT.png)

當旋律悄然劃過時空的軌跡，每一首歌就像一顆星辰，閃耀著創作者獨有的靈魂。  
你是否曾想過，這些音符與樂章之間，是否也隱藏著某種秩序？某種——**相似而未被察覺的連結**？

在現實中，我們以風格定義音樂，以性別分類聲線，以名字尋找熟悉。  
在程式語言的世界中，我們用 `GroupBy` 尋找規律，試圖透過「分組」理解資料之間微妙的關係。這不只是邏輯的操作，更像是一場哲學的提問：

<br/>

<font color=#D3D3D3 style="font-size: 22px;">&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;什麼定義了我們的「相同」？ </font>

<font color=#D3D3D3 style="font-size: 22px;">&emsp;&emsp;&emsp;&emsp;&emsp;在群體之中，我們究竟選擇呈現什麼？又忽略了什麼？
</font>

<br/>

今天，讓我們從一份歌手與歌曲的資料出發，探索 LINQ 中 `GroupBy` 的靈魂。

<br/>
<br/>

---

<br/>
<br/>

## 🐛 MS 文件中的命名解析

[官方文件連結](https://learn.microsoft.com/zh-tw/dotnet/api/system.linq.enumerable.groupby?view=net-8.0)

交響樂需要指揮的手勢，`GroupBy` 也仰賴幾個核心角色來指引分組的方式與結果

- {% label keySelector %}  
  它定義了分組的依據 —— 像是將歌曲按照**曲風**（搖滾、爵士、古典）、**調性**（大調、小調）、或是**演出者**進行分類。每一個 key 就像樂章中的主題，決定了整體的情緒。

- {% label comparer %}  
  當我們決定依照某個欄位分組後，還需要定義「何謂相同」。  
  是 `Ado` 和 `ado` 這兩個名字？還是說它們應被視為不同？這就像辨識旋律中的細微差異——不同演奏者的詮釋是否屬於同一首曲子？

- {% label elementSelector %}  
  分好組之後，我們要選擇的是：**組內要呈現什麼？**  
  是整筆資料、只顯示歌曲名？還是演出者的性別？這像是在決定觀眾到底看哪一段表演。

- {% label resultSelector %}  
  如果說 `elementSelector` 是選擇每段表演的內容，`resultSelector` 則是決定**整場演出的總呈現**。  
  或許我們不只要知道有哪些歌曲，而是想知道每位歌手的代表作、總共幾人參與，或某組的平均年齡——這些都是原始資料中未直接給出的價值。

<br/>
<br/>

---

<br/>
<br/>

## 🐛 讓資料說話

我們有這樣一份資料，來自世界各地的音樂人...
```CSHARP

	var singers = new List<Singer>
	{
		new Singer { Name = "Ado", Genre = "J-Pop", Song = "Lemon", IsFemale = true },
		new Singer { Name = "ado", Genre = "J-Pop", Song = "Usseewa", IsFemale = true },
		new Singer { Name = "Taylor Swift", Genre = "Pop", Song = "Shake It Off", IsFemale = true },
		new Singer { Name = "Ed Sheeran", Genre = "Pop", Song = "Shape of You", IsFemale = false },
		new Singer { Name = "Beethoven", Genre = "Classical", Song = "Für Elise", IsFemale = false },
		new Singer { Name = "Miles Davis", Genre = "Jazz", Song = "Kind of Blue", IsFemale = false }
	};

```

<br>

###　elementSelector / comparer

若我們希望將相同名字的歌手歸為一組，即便大小寫不同也視為相同人，可使用 StringComparer.OrdinalIgnoreCase
```CSHARP
	var groupedBySingerName = singers.GroupBy(
		s => s.Name,
		StringComparer.OrdinalIgnoreCase
	);

    foreach (var group in groupedBySingerName)
    {
        Console.WriteLine($"Group Key: {group.Key}");
        foreach (var singer in group)
        {
            Console.WriteLine($"- {singer.Name} - {singer.Song}");
        }
        Console.WriteLine();
    }


/// Group Key: Ado
///  - Ado - Lemon
///  - ado - Usseewa

/// Group Key: Taylor Swift
///  - Taylor Swift - Shake It Off


```

<br>

### resultSelector
```CSHARP
    // 使用 resultSelector
	var groupedBySingerGender = singers.GroupBy(
		s => s.IsFemale,
		(key,group) => new{
			Gender = key ? "Female" : "Male",
			Count = group.Count(),
			Songs = group.Select(s => s.Song)
		}
	);

	// 輸出每個組別的性別、人數和歌曲列表
	foreach (var group in groupedBySingerGender)
	{
		Console.WriteLine($"Gender: {group.Gender}");
		Console.WriteLine($"Singer Count: {group.Count}");
		Console.WriteLine("Songs:");
		foreach (var song in group.Songs)
		{
			Console.WriteLine($"- {song}");
		}
		Console.WriteLine();
	}

```

<br>
<br>

## 🐛 GroupBy + Dictionary

Map（一對一映射）：Dictionary<TKey, TValue> 的精神是「每個 Key 只有一個 Value」。這是它的不變式（invariant）。ToDictionary 直接從序列建字典時，假設你要的是 Map；

Bucket（分桶/分組）：GroupBy 的精神是「把同 Key 的元素丟進同一個桶」，一個 Key 可以對應很多個元素。GroupBy 先把資料分桶，代表你要的是 Bucket。

```csharp
void Main()
{
	//來源資料如下
	List<Trainer> trainers = new List<Trainer>()
			{
				new Trainer(Team.Valor, "Candela"),
				new Trainer(Team.Valor, "Bob"),
				new Trainer(Team.Mystic, "Blanche"),
				new Trainer(Team.Valor, "Alice"),
				new Trainer(Team.Instinct, "Spark"),
				new Trainer(Team.Mystic, "Tom"),
				new Trainer(Team.Dark, "Jeffrey")
			};
	
	//// 自己組織 Dictionary
	//優點
	//	邏輯可控：你可以在 Add 之前做驗證、過濾、排序等客製化操作。
	//	記憶體彈性：你可以選擇每個 List<Trainer> 的初始化方式（甚至用 HashSet）。
	//	不需要額外遍歷：一次 foreach 就能完成分組，比 GroupBy → ToDictionary 少了一個中介集合。

	//缺點
	//	程式碼冗長：手寫重複模式（檢查 key → 新建 → 加入）。
	//	可讀性差：表達「分組」的意圖不明確，看起來只是單純地操作 Dictionary。
	var dicts = new Dictionary<Team, List<Trainer>>();
	foreach(var trainer in trainers)
	{
		if (dicts.ContainsKey(trainer.Team) == false)
		{
			dicts.Add(trainer.Team, new List<Trainer>());
		}
		
		dicts[trainer.Team].Add(trainer);
	}
	
	//// groupby
	//優點
	//	語意清楚：一眼就能看出「依 Team 分組 → 每組變成 List → 放進 Dictionary」。
	// LINQ 表達力：可以輕鬆插入篩選、排序（例如 OrderBy、Where）。
	// 少手動錯誤：不用寫 ContainsKey 判斷，錯誤機率更低。

	//缺點
	//效能稍低：GroupBy 本身會產生一個分組的 IEnumerable，ToDictionary 又把這些分組轉成 Dictionary，所以比手動版本多一個中間層。（但通常資料量不大時差異可以忽略）
	// 靈活性較差：如果要對「同一組內」做特殊處理（例如：只挑第一個 Trainer，不是全部收集），手動迴圈可能更好。
	trainers.GroupBy(t => t.Team).ToDictionary(g => g.Key, g => g.ToList()).Dump();
}


public enum Team
{
	Valor, Mystic, Instinct, Dark
}

public class Trainer
{
	public Team Team{get;set;}
	public string Name{get;set;}
	public Trainer(Team team, string name)
	{
		Team = team;
		Name = name;
	}
}
```

**Dictionary 撞 key**

值得注意的是，如果集合直接 ToDictionary 會炸出

```plaintext
ArgumentException: An item with the same key has already been added.
```

因為 t => t.Team 是 key。Team.Valor 出現了多次是不被允許的，有違反 Map 的精神

<br>

**比較器一致性**
GroupBy 與 ToDictionary 都能帶 IEqualityComparer<TKey>。如果前者大小寫不敏感、後者用預設大小寫敏感，分桶與建表的規則就會不一致，可能出現你不預期的重複或分裂。

盡量在兩者都傳入同一個 comparer。

<br>

**Key 不能是 null**

Dictionary<TKey, TValue> 對參考型別 Key，Add(null, ...) 會丟 ArgumentNullException。
分組前可先過濾：Where(x => x.Key != null)。

<br>
<br>

## 🐛 用 GroupBy 看見背後的故事

GroupBy 不只幫助我們將資料歸類，它也讓我們重新思考：如何定義差異，如何尋找共通，如何呈現意義。召喚我們看見被忽略的關聯、遺落的故事，以及那個「屬於同一組」的哲學命題。

<br>
<br>

##  🐛 參考文章

{% btn 'https://blog.darkthread.net/blog/linq-groupby-todictionary-grouping/',利用LINQ GroupBy快速分組歸類,far fa-hand-point-right %}
