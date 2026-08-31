---
title: Deferred Execution - 延遲的智慧
date: 2024-10-23 08:29:11
categories: LINQ
top_img: https://i.imgur.com/bUJj6Dn.png
cover : https://i.imgur.com/bUJj6Dn.png
tags:
    - LINQ
toc:
toc_number:
comments :
---
![Image](https://i.imgur.com/bUJj6Dn.png)

<font color=#D3D3D3 style="font-size: 12px;"> 延遲執行（Deferred Execution）</font>

他是一種耐心的智慧 —— 明明寫下一段 Code，卻不急著執行，彷彿在等待一個更合適的時機 !?

<font color=#D3D3D3 style="font-size: 12px;"> 延遲執行（Deferred Execution）</font>

幫助你在被某句話刺到時，第一反應不是想要立刻回嘴、發怒或做出回應。不選擇這種「立即執行」的行為，防止你在關係上造成傷害或是事後後悔

<font color=#D3D3D3 style="font-size: 12px;"> 延遲執行（Deferred Execution）</font>

讓我們理解，重大決定前需要沉澱期，轉職、搬家、投資、進入一段關係…… 不適合因為他人的壓力就出發立即執行

而創意與靈感的生成過程，很多時候你以為自己沒有靈感，其實是「狀態機還沒執行」。
你已經把表達式建好了，放著沒動，等到靈感來臨時，才會開始真正地「MoveNext()」。

<br/>

<font color=#D3D3D3 style="font-size: 22px;">&emsp;&emsp;&emsp;&emsp;&emsp;我們不是缺靈感，只是活不夠久，讓它進入可被遍歷的狀態
</font>

<br/>


## 靈感的延遲執行

人生就是一段靈感蒐集箱
```CSHARP

public class InspirationCollector : IEnumerable<string>
{
    private readonly List<string> _inspirations = new();

    //// 體驗了甚麼?
    public void Experience(string eventDescription, string insight)
    {
        Console.WriteLine($"經歷：{eventDescription}");
        _inspirations.Add(insight);
    }

    //// 是對 IEnumerable<string> 的實作
    public IEnumerator<string> GetEnumerator()
    {
        foreach (var idea in _inspirations)
        {
            Console.WriteLine("靈感抽出狀態機...");
            yield return idea;
        }
    }

    //// 實作非泛型的版本。
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

```

在 InspirationCollector 中，當我們使用 yield return 把靈感逐一回傳時，其實背後是編譯器自動幫我們建立了一個「狀態機」類別，這個類別會記住當前執行的進度，實作了 IEnumerator<string>，讓我們能像撥放影集一樣，一個一個靈感逐步取出，而不是一次全部灌出來。

因此，當我們寫 GetEnumerator 時，背後會編譯成類似以下形式

```CSHARP

private sealed class InspirationEnumerator : IEnumerator<string>
{
    private int _state = -1;
    private string _current;
    private List<string> _list;
    private int _index;

    public InspirationEnumerator(List<string> source)
    {
        _list = source;
        _index = 0;
    }

    public bool MoveNext()
    {
        switch (_state)
        {
            case -1:
                _state = 0;
                break;
        }

        if (_index < _list.Count)
        {
            Console.WriteLine("靈感抽出狀態機...");
            _current = _list[_index++];
            return true;
        }

        _state = -2;
        return false;
    }

    public string Current => _current;
    object IEnumerator.Current => _current;

    public void Dispose() { }
    public void Reset() => throw new NotSupportedException();
}

```

這個生成的類別有完整的 MoveNext/Current 實作，foreach 就可以使用這個生成的迭代器來 loop 集合

```CSHARP

static void Main(string[] args)
{
    var life = new InspirationCollector();

    // 經歷一些人生片段，產生靈感
    life.Experience("在捷運上聽到一對情侶爭吵", "寫下人與人之間的誤解");
    life.Experience("深夜看了一部老電影", "寫下時間與遺憾的距離");
    life.Experience("朋友的一句話刺痛了我", "寫下藏在笑容後的疲憊");

    // 定義一段查詢，過濾出包含「寫下」但又不是「誤解」的內容，並轉成詩句
    // 查詢已定義，但尚未執行，沒有任何產出
    var filteredIdeas = life
        .Where(x => x.Contains("寫下") && !x.Contains("誤解"))
        .Select(x => $"靈感轉換成詩句：{x}");

    life.Experience("散步時被風喚醒", "寫下呼吸的自由感");
    Console.WriteLine("\n--- 靈感代號 呼吸的自由感, 刺激了創作者開始遍歷過往的靈感，啟動創作 ---");
    foreach (var poem in filteredIdeas)
    {
        Console.WriteLine(poem);
    }
}


/**

經歷：在捷運上聽到一對情侶爭吵
經歷：深夜看了一部老電影
經歷：朋友的一句話刺痛了我
經歷：散步時被風喚醒

--- 靈感代號 呼吸的自由感, 刺激了創作者開始遍歷過往的靈感，啟動創作 ---
靈感抽出狀態機...
靈感抽出狀態機...
靈感轉換成詩句：寫下時間與遺憾的距離
靈感抽出狀態機...
靈感轉換成詩句：寫下藏在笑容後的疲憊
靈感抽出狀態機...
靈感轉換成詩句：寫下呼吸的自由感

**/

```

順帶一提，這個 loop 過程是這樣的

1. 進入 foreach
2. 執行 GetEnumerator () , 得到一個 IEnumerator
3. 執行 MoveNext () , 以判斷 loop 是否結束。若尚未結束則將 Current 屬性移動到下一個元素
4. 回傳 Current 屬性給 item

不論是 IEnumerable 或是 IEnumerable<T> 都提供一個 GetEnumerator () 方法。
再透過所得到的 Enumerator 物件去執行 loop 這個動作，所以重點是 foreach 去找走訪器 "Enumerator" 這個行為，完成了我們常見的 loop

<br>
<br>

## 🦉延遲執行的陷阱小測驗

### 案例 1

以下這一段最後結果會是什麼?

```CSHARP

var testList = new List<int> { 5, 9, 8 };
var query = testList.Where(t => t > 6).Select(t => t + 1);
testList.Remove(9);
foreach (var t in query)
{
    Console.WriteLine(t);
}

```


結果是 9!

因為 Remove 會立即執行，而篩選與投射的過程只是資料的準備

<br>

### 案例 2

參考 https://www.damirscorner.com/blog/posts/20221216-BewareOfLinqDeferredExecution.html

```CSHARP

void Main()
{
	var newList = new List<int>{1,2,3};
	var wrapperList = CreateWrapper<int>(newList);
	var storeList = new List<Wrapper<int>>();
	storeList.AddRange(storeList);

	var result = wrapperList.Any(w => storeList.Contains(w));
	Console.WriteLine($"比較結果 : {result}");
}

public static IEnumerable<Wrapper<T>> CreateWrapper<T>(IEnumerable<T> items)
{
	return items.Select(item => {

		Console.WriteLine($"Create Wrapper for {item}");
		return new Wrapper<T>(item);
	});
}

public class Wrapper<T>
{
	private readonly T _items;
	
	public Wrapper(T item)
	{
		_items = item;
	}
}
```

讓我們想想這個執行會有甚麼樣的問題?


...
...
...
...



結果是 CreateWrapper 內部被執行了兩次!
![Image](https://i.imgur.com/0Uap2Bw.png)


因為 CreateWrapper 實際上還沒有真的執行，只是生成一個狀態機，到了 addRange() 以及 Any()，個別被叫用，因此包含的結果也是 False，因為他們參考的物件不同

如果要修正這個問題，我們會在 CreateWrapper 時就把他實體化，大多是使用 ToList()，以便後續的操作行為

改成
```CSHARP

private static List<Wrapper<T>> CreateWrapper<T>(IEnumerable<T> items)
{
    return items.Select(item =>
    {
        Console.WriteLine($"Create wrapper for {item}");
        return new Wrapper<T>(item);
    }).ToList();
}

```

可以得到
![Image](https://i.imgur.com/wk9VfCP.png)


<br>
<br>

根據 [Deferred execution](https://app.studyraid.com/en/read/1776/26682/deferred-execution) 延遲執行這件事要小心誤用，我們有一些重點需要思考

- {% label Multiple_Enumerations %}
  If you enumerate over a deferred execution query multiple times, the query will be executed multiple times, which can lead to performance issues.

- {% label Side_Effects %}
  If the data source changes between enumerations, the results may differ each time you iterate over the query.

- {% label Captured_Variables %}
  Be cautious when using variables captured by a lambda expression in a deferred query. If the value of the variable changes before the query is executed, the new value will be used.


<br>
<br>

## 📦 執行方法對照表

### 延遲執行方法對照表（Deferred Execution）
![Image](https://i.imgur.com/wPEkpAa.png)

<br>

### 立即執行方法對照表
![Image](https://i.imgur.com/JlgJL4q.png)
![Image](https://i.imgur.com/4dZqQn8.png)


<br>
<br>

## 實際的商業應用情境

<br>

### 📖 訂單查詢條件組合

定義好查詢後，可以根據不同的條件，調整查詢

```CSHARP

public class OrderService
{
    private readonly DataContext _context;

    public async Task<List<Order>> GetOrders(OrderQueryDto queryDto)
    {
        // 定義基礎查詢
        var query = _context.Orders
            .Where(o => o.CreateDate >= queryDto.StartDate 
                    && o.CreateDate <= queryDto.EndDate);

        // 根據用戶選擇動態增加條件
        if (queryDto.MinAmount.HasValue)
        {
            query = query.Where(o => o.Amount >= queryDto.MinAmount.Value);
        }

        if (queryDto.ShopId.HasValue)
        {
            query = query.Where(o => o.ShopId == queryDto.ShopId.Value);
        }

        // 根據權限增加限制
        if (!queryDto.IsAdmin)
        {
            query = query.Where(o => o.ShopIds.Contains(queryDto.CurrentShopId));
        }

        // 最後才執行查詢
        return await query.ToListAsync();
    }
}


```

<br>

### 📁 分批處理大量資料

```CSHARP

public class DataMigrationService
{
    private readonly ILogger<DataMigrationService> _logger;
    private readonly DataContext _context;

    public async Task MigrateUserData(DateTime cutoffDate)
    {
        // 定義要處理的資料範圍
        var query = _context.Users
            .Where(u => u.LastLoginDate < cutoffDate)
            .Select(u => new { u.Id, u.Email });

        // 在執行前先確認資料量
        var totalCount = await query.CountAsync();
        _logger.LogInformation($"Found {totalCount} users to migrate");

        if (totalCount > 10000)
        {
            // 分批處理大量資料
            int processed = 0;
            int batchSize = 1000;

            while (processed < totalCount)
            {
                var batch = await query
                    .Skip(processed)
                    .Take(batchSize)
                    .ToListAsync();

                await ProcessUserBatch(batch);
                processed += batch.Count;
                
                _logger.LogInformation($"Processed {processed}/{totalCount} users");
            }
        }
        else
        {
            // 小量資料直接處理
            var users = await query.ToListAsync();
            await ProcessUserBatch(users);
        }
    }
}

```

<br>

### 🗞️ 動態報表查詢

```CSHARP

public class SalesReportService
{
    private readonly DataContext _context;

    public async Task<SalesReportResult> GenerateReport(ReportCriteria criteria)
    {
        // 定義基礎查詢
        var query = _context.Orders
            .Where(o => o.OrderDate >= criteria.StartDate 
                    && o.OrderDate <= criteria.EndDate);

        // 根據報表類型調整分組
        var groupedQuery = criteria.GroupBy switch
        {
            GroupBy.Date => query
                .GroupBy(o => o.OrderDate.Date)
                .Select(g => new ReportItem
                {
                    Key = g.Key.ToString("yyyy-MM-dd"),
                    Amount = g.Sum(o => o.Amount)
                }),

            GroupBy.Shop => query
                .GroupBy(o => o.ShopId)
                .Select(g => new ReportItem
                {
                    Key = g.Key.ToString(),
                    Amount = g.Sum(o => o.Amount)
                }),

            GroupBy.Product => query
                .SelectMany(o => o.OrderItems)
                .GroupBy(i => i.ProductId)
                .Select(g => new ReportItem
                {
                    Key = g.Key.ToString(),
                    Amount = g.Sum(i => i.Amount)
                }),

            _ => throw new ArgumentException("Invalid group by option")
        };

        // 執行前可以加入其他處理邏輯
        if (criteria.NeedSort)
        {
            groupedQuery = groupedQuery.OrderByDescending(x => x.Amount);
        }

        // 最後才執行查詢
        var result = await groupedQuery.ToListAsync();
        return new SalesReportResult
        {
            Items = result,
            TotalAmount = result.Sum(x => x.Amount)
        };
    }
}

```

<br>
<br>

## IEnumerable<T> 沒有 ForEach 是刻意的設計


List<T>.ForEach 是 List 自己的「便利方法」；IEnumerable<T> 沒有 ForEach 是刻意的設計，為了避免把「查詢」介面變成「執行副作用」的工具。


<br>

### 為什麼 List<T> 有 ForEach？

這個方法是 .NET 2.0（早於 LINQ） 時期就存在的「語法糖」，方便你對 已在記憶體中的可索引集合 做動作：

```csharp
list.ForEach(x => Console.WriteLine(x));
```

它可以用 索引 for 迴圈 走訪，少掉建立列舉器的成本，對 List<T> 來說也有一點效能優勢。那為什麼 IEnumerable<T> 刻意不提供 ForEach？

從LINQ 設計原則角度來說，IEnumerable<T> 是「查詢序列」的最小介面，LINQ 標準運算子（Where/Select/GroupBy...）希望是純函式、可組合、可延遲評估。
而 ForEach 代表「執行副作用」，而且是 void，不可組合，會打破這個模型。LINQ 團隊刻意不把副作用方法放在 Enumerable 擴充裡，維持「查詢 vs. 執行」的分界。

<br>

### 延遲/串流來源的安全性

很多 IEnumerable<T> 不是一個現成的清單，而是延遲查詢（EF Core、Dapper、IQueryable 轉成 IEnumerable 時才執行）
ForEach 一呼叫就 強迫整個序列被消費，副作用很難控；若來源是遠端/昂貴/無限序列，後果更不可預期。
我們需要避免多次列舉陷阱 IEnumerable<T> 可能被多次列舉才會得到資料（而且每次結果還可能不同）。若有 ForEach，開發者很容易先 ForEach 做了副作用
又在別處再列舉一次 → 造成 重複執行/重複打 DB 的隱性 bug
並且語言早就有 foreach 關鍵字，對於「把每個元素做一件事」，C# 已有語言級的 foreach，讀性也很好：

```csharp
foreach (var x in source) Do(x);
```

所以加一個 IEnumerable<T>.ForEach 並沒有本質提升，反而模糊了 LINQ 的查詢語意。


<br>
<br>

## 我們，也在活一段延遲執行的人生

你累積的每一段經歷、每一份感受，都像是寫下但尚未執行的表達式；
你可能覺得這些事情暫時沒用，但它們默默存在，等著被取用、被投影、被過濾，
在某個靈感來臨的時刻，成為你作品裡的詩句、專案裡的靈光，甚至某個深夜的領悟。


<br/>

<font color=#D3D3D3 style="font-size: 22px;">&emsp;&emsp;&emsp;&emsp;&emsp;我們不是缺靈感，只是活不夠久，讓它進入可被遍歷的狀態
</font>

<br/>

別急著執行每一個人生邏輯，
給自己多一點延遲，也許，那正是最貼近當下的一種高效。