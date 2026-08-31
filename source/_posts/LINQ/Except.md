---
title: Except
date: 2025-09-22 08:50:00
categories: LINQ
top_img: https://i.imgur.com/cBfeEDY.png
cover : https://i.imgur.com/cBfeEDY.png
tags:
    - LINQ
toc:
toc_number:
comments :
---

## 🐛 比較問題

在 .NET 裡，Distinct()、Except()、Intersect() 這些方法都依賴物件的 相等性判斷。問題是，自訂類別沒有覆寫 Equals/HashCode 時，預設只會用「參考相等」來判斷。

也就是說，就算兩個 User 物件的內容 (Id, Name) 一模一樣，只要它們是從不同 JSON 反序列化回來的，記憶體位址不同 → 就被視為「不同物件」。結果就是：Except() 無法正確比對出差異。


Dictionary、HashSet、Distinct、Except 這些操作的底層共通點：都要先決定「兩個物件是否相等」、「如何計算 HashCode」。
預設情況：C# 類別若沒覆寫，會使用物件的參考相等 → 兩個內容相同的物件仍被視為不同。
正確做法：告訴系統如何判斷相等：

- IEqualityComparer<T> → 外部策略，適合偶爾需要、不同情境可切換。
- IEquatable<T> + Equals/GetHashCode → 內建策略，適合固定規則，讓所有集合操作一律遵守。

本質就是「給集合一個可靠的相等性規則」，否則系統只能退回到「記憶體位址比較」。

<br>

### 解法 1 實作自定義 IEqualityComparer

最直觀的方式：自己寫一個 UserComparer，明確告訴 LINQ「判斷兩個 User 相等時，要比對 Id 就好」。
```csharp
class UserComparer : IEqualityComparer<User>
{
    public bool Equals(User x, User y)
    {
        if (Object.ReferenceEquals(x, y)) return true;
        if (Object.ReferenceEquals(x, null) || Object.ReferenceEquals(y, null))
            return false;
        return x.Id == y.Id;
    }

    public int GetHashCode(User obj)
    {
        if (object.ReferenceEquals(obj, null)) return 0;
        return obj.Id == null ? 0 : obj.Id.GetHashCode();
    }
}


static void Test3()
{
    var original = new List<User>()
    {
        new User{ Id = "A01", Name = "Jeffrey" },
        new User{ Id = "A02", Name = "Darkthread" }
    };
    var originalJson = JsonConvert.SerializeObject(original);

    var merged = new List<User>();
    merged.AddRange(original);
    merged.Add(new User
    {
        Id = "B01",
        Name = "Ironman"
    });
    var mergedJson = JsonConvert.SerializeObject(merged);

    //模擬情境
    //original 由資料庫欄位 JSON 還原
    //merged 由 WebAPI Request 傳入 JSON 還原
    var fromDb = JsonConvert.DeserializeObject<List<User>>(originalJson);
    var fromReq = JsonConvert.DeserializeObject<List<User>>(mergedJson);

    var diff = fromReq.Except(fromDb, new UserComparer());
    Console.WriteLine(JsonConvert.SerializeObject(diff));
}
```
這樣 Except 就能按照 Id 判斷，得到正確的差異結果。

<br>

### 解法 2  IEquatable<T> 與覆寫 Equals()、GetHashCode()

另一種方式是，直接在 User 類別上宣告「我的相等性定義」，實作 IEquatable<User>，並覆寫 Equals() 與 GetHashCode()。

```csharp
 public class User : IEquatable<User>
{
    public string Id { get; set; }
    public string Name { get; set; }

    public bool Equals(User other)
    {
        if (other == null) return false;
        return this.Id == other.Id;
    }

    public override bool Equals(object obj) => Equals(obj as User);
    public override int GetHashCode() => Id.GetHashCode();
}
	
static void Test4()
{
    var original = new List<User>()
    {
        new User{ Id = "A01", Name = "Jeffrey" },
        new User{ Id = "A02", Name = "Darkthread" }
    };
    var originalJson = JsonConvert.SerializeObject(original);

    var merged = new List<User>();
    merged.AddRange(original);
    merged.Add(new User
    {
        Id = "B01",
        Name = "Ironman"
    });
    var mergedJson = JsonConvert.SerializeObject(merged);

    //模擬情境
    //original 由資料庫欄位 JSON 還原
    //merged 由 WebAPI Request 傳入 JSON 還原
    var fromDb = JsonConvert.DeserializeObject<List<User>>(originalJson);
    var fromReq = JsonConvert.DeserializeObject<List<User>>(mergedJson);

    var diff = fromReq.Except(fromDb);
    Console.WriteLine(JsonConvert.SerializeObject(diff));
}
```
這樣一來，LINQ 的 Except、Distinct、Intersect 等方法就能直接使用，不需要額外傳 comparer

<br>
<br>

## 🐛 產生資料庫增刪指令

假設有兩份資料（舊的 from DB、新的 from Request），要算出哪些領地新增、哪些刪除，就能用 Except() 快速完成。

```csharp
void Main()
{
		string srcOrig =
	@"曹操：冀州、兗州、青州、徐州、梁州、雍州、豫州
	孫權：揚州
	劉備：益州、荊州
	黑大：潮州
	梅西：美州";
		string srcNew =
	@"曹操：冀州、兗州、青州、潮州、梁州、雍州、豫州
	孫權：揚州、荊州、交州
	劉備：益州
	黑大：徐州
	魯本：美州";

    //// 整理有哪些 leaders , 既有以及新的領地有哪些
	var originalLeaderRegions = parseTeritory(srcOrig);
	var newLeaderRegions = parseTeritory(srcNew);
	var allLeaders = originalLeaderRegions.Keys.Union(newLeaderRegions.Keys);
	foreach (var leader in allLeaders)
	{
		var originRegions = originalLeaderRegions.ContainsKey(leader) ? originalLeaderRegions[leader] : new List<string>();
		var newRegions = newLeaderRegions.ContainsKey(leader) ? newLeaderRegions[leader] : new List<string>();
		
        //// 要移除的領地
		var toDelRegions = originRegions.Except(newRegions);

        //// 要新增的領地
		var toAddRegions = newRegions.Except(originRegions);

		Console.WriteLine($"主公 {leader} , 大意失去了 : {string.Join(",", toDelRegions)}, 但搞定了 : {string.Join(",",toAddRegions)}");
	} 
}


public static Dictionary<string,List<string>> parseTeritory(string input)
{
	var finalDicts = new Dictionary<string,List<string>>();
	var lines = input.Replace("\r","").Split("\n");
	foreach(var line in lines)
	{
		var leaderRegionsParis = line.Split("："); //// 全形符號啦!
		var leader = leaderRegionsParis[0];
		var regions = leaderRegionsParis.Last().Split("、").ToList();
		if(finalDicts.TryAdd(leader,regions) == false)
		{
			finalDicts.Add(leader,regions);
		}
	}
	
	return finalDicts;
}
```

執行結果會列出每個人失去與新增的領地，等於幫你把「差異」轉換成可以執行的「資料庫指令」。

<br>
<br>

## 🐛 參考文章

{% btn 'https://blog.darkthread.net/blog/linq-except-for-custom-class/',LINQ Except() 比對自訂類別,far fa-hand-point-right %}
