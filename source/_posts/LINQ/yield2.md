---
title: Yield Return - 實驗
date: 2024-10-16 08:21:11
categories: LINQ
top_img: https://i.imgur.com/cBfeEDY.png
cover : https://i.imgur.com/cBfeEDY.png
tags:
    - LINQ
toc:
toc_number:
comments :
---

## 🐛 IEnumerable + yield 的運作機制詳解

<br>

### 一般集合的回傳方式
```CSHARP

void Main()
{
    var store = new DemoStore();
    var productList = store.GetProducts();
    foreach (var product in productList)
    {
        $"{product.Id}，{product.Name}".Dump();
    }
}
public class DemoStore
{
    public List<Product> GetProducts()
    {
        List<Product> result = new List<Product>();
        for (int i = 0; i < 5; i++)
        {
            $"{i}產生Product中".Dump();
            result.Add(new Product()
            {
                Id = i,
                Name = $"Pro {i}"
            });
        }
        return result;
    }
}
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
}

```

方法是「一次性跑完」，建立好完整集合，再交給呼叫端使用。呼叫 GetProducts() 的那一刻，就已經完成所有運算。
![Image](https://i.imgur.com/ffzYzSo.png)

<br>

### yield return 的回傳方式

```CSHARP

void Main()
{
	var store = new DemoStore();
	var productList = store.GetProducts();
	foreach (var product in productList)
	{
		$"{product.Id}，{product.Name}".Dump();
	}
}

public class DemoStore
{
	public IEnumerable<Product> GetProducts()
	{
		int count = 5;
		for (int i = 0; i < count; i++)
		{
			$"{i}產生Product中".Dump();
			yield return new Product()
			{
				Id = i,
				Name = $"Pro {i}"
			};
		}
	}
}
public class Product
{
	public int Id { get; set; }
	public string Name { get; set; }
}

```

![Image](https://i.imgur.com/wOw48wV.png)

方法並沒有「馬上跑完」，而是被編譯器轉換成「狀態機」

- foreach 的 MoveNext() 第一次呼叫 → 跑到 yield return，吐出第一個值，暫停
- 第二次呼叫 → 從暫停處繼續，吐出下一個值
- 重複，直到迴圈結束

換句話說，不是「先造好整個清單 → 再走訪」，而是**「一邊造、一邊走」**。

<br>

### 為什麼 foreach 能跑？

因為 foreach 的本質是這樣
```CSHARP
var enumerator = products.GetEnumerator();
while (enumerator.MoveNext())
{
    var current = enumerator.Current;
    // loop body
}
```

只要一個型別實作了 IEnumerable，就能用 foreach。List<T>、Array：實作自己的 Enumerator（固定集合）
而 yield return 是由編譯器幫你產生 Enumerator（狀態機，動態生成），所以，foreach 不是專門為 yield return 設計的，而是為所有 IEnumerable 設計的。

<br>
<br>


## 🐛 IEnumerable + yield 的效能實驗

傳統方法回傳「完整集合」 → 先做完，再交出去。yield return 方法回傳「逐步產生的序列」 → 邊做邊交，按需生成。
這兩者差別，直接影響到 時間 和 記憶體。

<br>

### 🧪 實驗一：時間成本（提早停止）

分別有兩個方法，FilterRandomDataset、FilterRandomDatasetYield

FilterRandomDataset 讀取所有檔案，把尾巴是 000 的檔案抓出來
FilterRandomDatasetYield 掃到 000 就先回傳了，不需要看完全部的檔案

所以當我們今天的需求是，只要拉到三筆 000 結尾的資料時，更適合使用 yield return 來處理
```CSHARP

static void Main(string[] args)
{
    var stTime = DateTime.Now;
    var path = @"C:\Users\Allen Lin\Desktop\AllenLab\PowerShellV2\RandomFile";
    Log("Dump All Start", stTime);

    //// 用 foreach 列出所有結果
    foreach (var file in FilterRandomDataset(path))
    {
        Log(file, stTime);
    }

    Log("End", stTime);

    stTime = DateTime.Now;
    Log("Show First 3 Start", stTime);
    //// 只讀前三筆，這次用 LINQ 寫
    FilterRandomDataset(path).Take(3).ToList().ForEach(file => Log(file, stTime));
    Log("End", stTime);
}

//// 傳統寫法
private static string[] FilterRandomDataset(string path)
{
    return Directory.GetFiles(path)
        .Where(file => File.ReadAllText(file).TrimEnd(Environment.NewLine.ToCharArray()).EndsWith("000")).ToArray();
}

//// yield 寫法
private static IEnumerable<string> FilterRandomDatasetYield(string path)
{
    foreach (var file in Directory.GetFiles(path))
    {
        if (File.ReadAllText(file)
                .TrimEnd(Environment.NewLine.ToCharArray()).EndsWith("000"))
            yield return file;
    }
}

private static void Log(string message, DateTime stTime)
{
    Console.WriteLine($"{(DateTime.Now - stTime).TotalMilliseconds / 1000:000.00}s {message}");
}

```

用 FilterRandomDataset 跑結果
![Image](https://i.imgur.com/NyHWO5l.png)

用 FilterRandomDatasetYield 跑結果
![Image](https://i.imgur.com/h6va8Ow.png)

重複 3 次結果都差不多 0.7 --> 0.2 秒，明顯呈現出 yield return 免除了因為多撈資料造成的時間浪費

<br>

### 🧪 實驗二：記憶體成本（避免多餘分配）

關於程式用了多少記憶體這件事，有個簡單測量方法叫做 GC.GetTotalMemory

### GC.GetTotalMemory

GC.GetTotalMemory 方法是一個用於測量這個程式 Managed Heap 記憶體使用情況的工具。
它回傳應用程式中所有存活（live）的 Managed Object 所佔用的記憶體總量，也就是存活物件的記憶體大小
測量的數值表示「目前程式裡還在用的東西」所佔用的記憶體空間。這些東西（物件）可能包括：已分配的變數、new 出來的 instances、字串、集合等。
即使有些東西程式已經用不到，但垃圾回收器（GC）還沒來得及清掉，它們暫時也會算在這個數字裡。
測量的範圍只管 .NET 自己管理的記憶體，.NET 有一個「託管的記憶體區域」（Managed Heap），專門用來存放程式裡分配的物件。
GC.GetTotalMemory 只會測量這部分的大小，不會管那些由其他地方管理的記憶體，比如你自己用外部函式庫分配的內存、未託管的資源（例如文件指標、資料庫連線等）。


GetTotalMemory 主要有兩種方式

1. 直接返回當前已分配的記憶體快照，無需進行垃圾收集， Managed Heap 中的所有已分配物件，"包括可能不再使用但尚未被垃圾收集的物件"，適用於需要快速了解程式當前的記憶體使用情況，但不要求完全準確，就是還沒被丟掉的都算進來啦!

特點

- 快速，不會因執行垃圾收集而影響應用的性能。
- 數值可能偏大，因為它可能包含一些已成為垃圾（但尚未被回收）的記憶體。

2. 在測量前，強制執行一次完整的垃圾收集，清理掉所有可回收的物件後，再測量記憶體。

只包含仍然被引用且活躍的物件，過時或不再需要的記憶體已被釋放。

特點

- 更準確，因為結果反映了程式實際需要的記憶體大小。
- 花費時間更長，可能對應用性能產生短暫影響。

簡單類比一下就是

只是掃一眼房間，看看房間裡所有東西的總體積（包括垃圾和有用的物品）。
像是先把房間的垃圾清理掉（做一次徹底的打掃），然後再計算房間裡有用物品的總體積。

而決定哪個模式就是帶入參數，簡單的快照帶入 false, 精準的資訊為 true

<br>

### 回到實驗本身

情境是生成 10000個 Guid，將他們轉換成字串，印出來，我需要的資訊分別是最終集合會得到的數量、第一筆 GUID

GetGuidStrings 全部轉換成字串後，生成一個 Array

GetGuidStringsYield 一個個 loop

以 Count() 來說，他沒有生成一個真正的集合出來占用一波記憶體，而是單純計數
以 First() 而言，更容易理解了，只是單純的想取得第一筆資料

```CSHARP

static void Main(string[] args)
{
    TestMemory();
}

private static void TestMemory()
{
    List<Guid> guidPool = Enumerable.Range(0, 10000).Select(i => Guid.NewGuid()).ToList();
    GetMemorySize();
    Console.WriteLine(GetGuidStrings(guidPool).Count());
    GetMemorySize();
    Console.WriteLine(GetGuidStrings(guidPool).First());
    GetMemorySize();
    Console.ReadLine();
}

private static void GetMemorySize()
{
    var size = GC.GetTotalMemory(false) / 1024 / 1024;
    Console.WriteLine($"Memory Size {size} MB");
}

//// 傳統寫法
private static string[] GetGuidStrings(List<Guid> guids)
{
    return guids.Select(guid => guid.ToString()).ToArray();
}

//// yield
private static IEnumerable<string> GetGuidStringsYield(List<Guid> guids)
{
    foreach(var guid in guids)
    {
        yield return guid.ToString();
    }
}


```

傳統寫法結果
![Image](https://i.imgur.com/lVt5HQV.png)

yield 結果
![Image](https://i.imgur.com/0ihhXBt.png)


用 Bytes 為單位觀察，最後結果記憶體占用是有差異的，這件事與原實驗結果相符

1. 有結果立即回傳，提供更好的即時性 (這是要串接成生產線模式的必要條件)
2. 只需部分結果時，省去處理無用資料的成本
3. 不需耗用記憶體儲存全部結果

<br>
<br>

## 🐛 參考文章


{% btn 'https://blog.darkthread.net/blog/yield-return/',善用 yield return 省時省 CPU 省 RAM，打造高效率程式,far fa-hand-point-right %}

<br>

{% btn 'https://learn.microsoft.com/en-us/archive/msdn-magazine/2017/april/essential-net-understanding-csharp-foreach-internals-and-custom-iterators-with-yield?WT.mc_id=DOP-MVP-37580',Understanding C# foreach Internals and Custom Iterators with yield,far fa-hand-point-right %}

<br>

{% btn 'https://www.cnblogs.com/wucy/p/17443749.html',由 C# yield return 引发的思考,far fa-hand-point-right %}

<br>

{% btn 'https://ithelp.ithome.com.tw/articles/10298185?sc=iThelpR',IEnumerable + yield 的坑踩過了嗎,far fa-hand-point-right %}