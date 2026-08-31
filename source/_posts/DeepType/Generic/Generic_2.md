---
title: Generic - 記憶の箱
date: 2025-10-30 17:19:05
categories: 程思舞想
top_img: https://i.imgur.com/Zly1UDM.png
cover : https://i.imgur.com/Zly1UDM.png
tags:
    - 程思舞想
toc:
toc_number:
comments :
---

{% tabs Generic%}

<!-- tab 記憶之箱-->

![Generic2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/Cache/laning_1.png)

我們的大腦，其實就像一個會自動學習的「快取系統」。有些經驗，一次就讓我們記得深刻；有些事情，卻需要多次重複才會留下印象。有些回憶被時間淘汰、過期了，就像舊的快取資料一樣，被默默清除；但也正因為如此，新的經驗才能被寫入，人生才不會停在原地

程式中的 泛型（Generic），讓我們能用同一套邏輯，處理不同型別的資料；人生中，我們也在學習如何用同樣的「思考模式」，面對不同的狀況。無論是人際關係、工作挑戰、或自我成長，我們都在建立一個「屬於自己的泛型快取」——那些經驗的抽象化、情緒的緩存、學習的積累，都讓我們在未來遇到類似情境時，不必從零開始

![Generic2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/Cache/hd_cache_compare.png)

![Generic2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/Cache/life_like_cache_1.png)

<!-- endtab -->

<!-- tab 🌸 泛型與快取-->

泛型不僅在計算和包裝上有用，在快取機制中也能發揮強大的作用。透過泛型快取類別，我們可以建立一個通用的快取解決方案，適用於任何型別的資料

<br>

#### 🔹 SimpleCache

```csharp
void Main()
{
    var cache = new SimpleCache<string>();
    var userIntro = cache.GetOrCreate("123", () => DB.GetUserInfo("123"));
    userIntro.Dump();
}

public class SimpleCache<T>
{
    public MemoryCache cache = new MemoryCache(new MemoryCacheOptions());
    
    public T GetOrCreate(string key, Func<T> createItem)
    {
        T cacheEntry;
        if(cache.TryGetValue(key, out cacheEntry) == false)
        {
            cacheEntry = createItem();
            cache.Set(key, cacheEntry);
        }
        
        return cacheEntry;
    }
}

public static class DB
{
    public static string GetUserInfo(string number)
    {
        return "ya";
    }
}

var userCache = new SimpleCache<UserInfo>();
var user = userCache.GetOrCreate("user_123", () => UserService.GetUserById(123));

var calculationCache = new SimpleCache<decimal>();
var result = calculationCache.GetOrCreate("complex_calc_abc", () => PerformComplexCalculation());


var queryCache = new SimpleCache<List<Product>>();
var products = queryCache.GetOrCreate("category_electronics", () => ProductService.GetByCategory("Electronics"));
```


![Generic2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/Cache/simple_cache.png)

<!-- endtab -->

<!-- tab 🔹 記憶之箱-->

我們結合人類記憶的以及泛型的概念製作記憶之箱，因此還要考慮記憶過期等問題

![Generic2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/Cache/simple_to_safe_cache.png)


我們以 CacheItem 為一個記憶單元

![Generic2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/Cache/unit_of_memory.png)


程式實作
```csharp
public class MemoryBox<T>
{
    public static readonly ConcurrentDictionary<string, CacheItem<T>> memories = new ConcurrentDictionary<string, CacheItem<T>>();

    public T Remember(string key, Func<T> experience, TimeSpan duration)
    {
        if(memories.TryGetValue(key, out CacheItem<T> cacheItem) && DateTime.Now < cacheItem.ExpirationTime)
        {
            Console.WriteLine($"[MemoryBox] Hit cache for key: {key}");
            return cacheItem.Data;
        }

        var data = experience();
        var newCacheItem = new CacheItem<T>
        {
            Data = data,
            ExpirationTime = DateTime.Now.Add(duration)
        };

        memories[key] = newCacheItem;
        return data;
    }
}

public class CacheItem<T>
{
    public T Data { get; set; }
    public DateTime ExpirationTime { get; set; }
}
```

<!-- endtab -->

<!-- tab ✨✨ 登入人生 online-->

![LearnFromLife](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/Cache/logging_life.png)

```csharp
[HttpGet("LearnFromLife")]
public void LearnFromLife()
{
    Console.WriteLine("🌱 登入人生 online...\n");

    var memoryBox = new MemoryBox<string>();

    // 第一次遇人沒打招呼，被爆打，學會教訓記5秒
    var lesson = memoryBox.Remember("失敗經驗", () =>
    {
        Console.WriteLine("💥 第一次：沒打招呼被爆打一頓！");
        return "記得要打招呼，不然會被爆打 😵";
    }, TimeSpan.FromSeconds(5));

    Console.WriteLine($"🧠 記住教訓：{lesson}\n");

    // 3秒後再遇到人，因為快取還在，這次沒被打
    Console.WriteLine("⏳ 過了3秒，又遇到人...");
    Thread.Sleep(3000);

    var lesson2 = memoryBox.Remember("失敗經驗", () =>
    {
        Console.WriteLine("（這段不該出現，如果出現代表忘了）");
        return "又被打了 🤕";
    }, TimeSpan.FromSeconds(5));

    Console.WriteLine($"😎 這次學乖了我好棒：{lesson2}\n");

    // 再過3秒（共6秒），超過快取時限，又遇到人又忘了
    Console.WriteLine("⏳ 又過了3秒（共6秒），記憶消失中...");
    Thread.Sleep(3000);

    var lesson3 = memoryBox.Remember("失敗經驗", () =>
    {
        Console.WriteLine("💥 啊他馬孔鼓勵又忘記打招呼，被更慘地爆打一頓！這次學久一點！記住20秒");
        return "沒打招呼好痛 我知道要打招呼";
    }, TimeSpan.FromSeconds(20));

    Console.WriteLine($"🧠 更新記憶：{lesson3}\n");

    // 過6秒，還在快取內
    Console.WriteLine("⏳ 又過了6秒，再次遇到人...");
    Thread.Sleep(6000);

    var lesson4 = memoryBox.Remember("失敗經驗", () =>
    {
        Console.WriteLine("（這段不該出現，如果出現代表又忘了）");
        return "我沒救了";
    }, TimeSpan.FromSeconds(20));

    Console.WriteLine($"😌 還記得教訓：{lesson4}\n");
    Console.WriteLine("後悔登入人生 online...");
}
```

![Generic2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/Cache/t0.png)
![Generic2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/Cache/t3.png)
![Generic2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/Cache/t6.png)
![Generic2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/Cache/t12.png)

<!-- endtab -->

<!-- tab 總結-->

![life](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/Cache/life_learn.png)

人生就像一個不停運作的應用程式，每一次錯誤、每一次學習，都是被快取起來的「記憶物件」

有時候，我們會忘記一些事，那不是壞事，因為「清除快取」正是系統維持效能的必要過程。
若什麼都不忘，我們就再也裝不進新的自己。

![forget](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/Cache/forget_for_update.png)

而泛型就像是一種智慧的思考模板，讓我們在不同情境中，都能套用那份學到的抽象經驗。不論是愛情失敗的泛型、工作崩潰的泛型，還是「不要邊吃邊講話」的社交泛型，都讓我們在下一次執行 LearnFromLife() 時，能少一點例外（Exception），多一點成功回傳（Return）

<!-- endtab -->

{% endtabs %}