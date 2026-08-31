---
title: IEnumerable, ICollection, IList and List – Which One To Use?
date: 2024-10-22 16:33:11
categories: LINQ
top_img: https://i.imgur.com/cBfeEDY.png
cover : https://i.imgur.com/cBfeEDY.png
tags:
    - LINQ
toc: true
toc_number: true
comments: true
---

## 🐛 IEnumerable

IEnumerable 是所有集合的基礎款，ICollection 繼承 IEnumerable，IList 繼承 ICollection，而我們常用的 List 就是 IList 的實作。  
所以 IEnumerable 相當於阿嬤，要敬老尊賢，或者也可以想像你訂閱了微軟的集合使用基礎方案 : IEnumerable。

![Image](https://i.imgur.com/mjU8Qn8.png)

基於這個基礎方案我們獲得這些功能：

<br>

### 可以用 foreach 逐筆處理（走訪抽象）。

```csharp
IEnumerable<string> names = new[] { "Ann", "Bob", "Cindy" };
foreach (var n in names)
    Console.WriteLine(n);
```

<br>

### 支援延遲執行（LINQ-to-Objects 的 Where/Select/Take）
```csharp
var data = Enumerable.Range(1, 1_000_000); // 還沒算
var firstFiveEvens = data.Where(x => x % 2 == 0) // 還沒算
                         .Take(5)                // 還沒算
                         .ToList();              // 這裡才真的跑
// 結果：2,4,6,8,10
```

<br>

### 用 yield return 自訂走訪
```csharp
public static IEnumerable<int> EvenNumbers() // 無限序列（懶產生）
{
    int n = 0;
    while (true)
    {
        yield return n;
        n += 2;
    }
}

// 使用：只取前 5 筆
foreach (var x in EvenNumbers().Take(5))
    Console.WriteLine(x); // 0,2,4,6,8
```

<br>

### Count() / Contains() / Any()… 是 Enumerable 的延伸方法。
```csharp
IEnumerable<int> seq = Enumerable.Range(1, 10);
int c = seq.Count(); // O(n)
```

<br>

### IEnumerable 的限制

只讀、前進式（無索引、不可改）。
每次枚舉都要重新執行，若來源昂貴會付出重複成本。

✅ 解法：先 .ToList() 快取。

```csharp
IEnumerable<string> FromApi() => CallSlowApiAsEnumerable();

var first = FromApi().First();  // 叫一次
var count = FromApi().Count();  // 又叫一次（慢）

// ✅ 快取
var cache = FromApi().ToList();
var first2 = cache.First();
var count2 = cache.Count;       // O(1)
```

<br>
<br>

## 🐛 ICollection

升級訂閱方案後就可以使用 ICollection!
ICollection<T>（在 IEnumerable 之上）：能被修改，且有 Count 屬性。
```csharp
public int CountSpecialCharacters(ICollection<char> specialChars)
{
    if (!specialChars.IsReadOnly)
    {
        specialChars.Add('~');
        specialChars.Add('!');
    }

    return specialChars.Count;
}

// 呼叫端
var s1 = new List<char> { '$', '%', '^' };
var s2 = new HashSet<char> { '$', '%', '^' };
Console.WriteLine(CountSpecialCharacters(s1));
Console.WriteLine(CountSpecialCharacters(s2));
```

<br>

### MatchCollection

Regex.Matches(...) 會回傳 MatchCollection，可以：

foreach 枚舉

使用 LINQ 運算子
```csharp
var text = "A12+A34+B99-C56+A87";
var matches = Regex.Matches(text, "A(?<n>\\d+)");

foreach (Match m in matches)
    Console.WriteLine($"完整: {m.Value}, 數字: {m.Groups["n"].Value}");

var numbers = matches.Select(m => int.Parse(m.Groups["n"].Value)).ToList();
// numbers: 12, 34, 87
```

<br>

### NameValueCollection：為什麼不能直接 Where？

因為 NameValueCollection 沒有實作 IEnumerable<T>，只能轉成 AllKeys 來 LINQ。
```csharp
var setting = ConfigurationManager.AppSettings
    .AllKeys
    .Select(o => new { Key = o, Value = ConfigurationManager.AppSettings[o] })
    .Where(o => o.Key.StartsWith("J"))
    .ToDictionary(k => k.Key, v => v.Value);
```

另一個方法是用 Cast<string>()
```csharp
var app = ConfigurationManager.AppSettings;
var pairs = app.AllKeys.Cast<string>()
                       .Select(k => KeyValuePair.Create(k, app[k]));
```

<br>
<br>

## 🐛 IList

IList 在 ICollection 之上，新增了 索引操作 (Insert/RemoveAt)。
```csharp
public int CountSpecialCharacters(IList<char> specialCharacters)
{
    specialCharacters.Add('~');
    specialCharacters.Add('!');
    specialCharacters.Insert(0, '$');
    return specialCharacters.IndexOf('~');
}

IList<char> charList = new List<char>(){'$', '%', '^'};
CountSpecialCharacters(charList).Dump();
```

典型應用：

待辦清單 App（新增/刪除第 n 項）
購物車（修改數量、刪除第 n 個商品）
播放清單（移動歌曲）

<br>
<br>

## 🐛 使用甚麼集合

<br>

### 方法回傳值
```csharp
public List<Order> GetOrders() // ❌ 太多：呼叫端能修改
    => _context.Orders.ToList();

public IEnumerable<Order> GetOrders() // ✅ 只保證可走訪
    => _context.Orders.ToList();

public IReadOnlyList<Order> GetOrders() // ✅ 明確，不能修改但可索引
    => _context.Orders.ToList();
```

查詢 → IEnumerable<T> 或 IReadOnlyCollection<T>

索引 → IReadOnlyList<T>

真的要修改才回傳 List<T>

<br>

### 方法參數
```csharp
public void ProcessOrders(IEnumerable<Order> orders) // ✅ 最寬鬆
{
    foreach (var o in orders)
        Console.WriteLine(o.Id);
}

public void NormalizeTags(ICollection<string> tags) // ✅ 因為會 Add/Remove
{
    tags.Add("Default");
    tags.Remove("Deprecated");
}

public void Swap<T>(IList<T> list, int i, int j) // ✅ 因為要用索引
{
    (list[i], list[j]) = (list[j], list[i]);
}
```

<br>

### 類別成員
```csharp
private List<string> _tags = new(); // ✅ 私有欄位用具體型別
public IReadOnlyCollection<string> Tags => _tags; // ✅ 公開只讀
```

<br>

### LINQ 與延遲執行
```csharp
public List<User> GetUsers() => _context.Users.ToList(); // ✅ 立即執行

public IQueryable<User> QueryUsers() => _context.Users; // ✅ 保留查詢
```