---
title: Cache - 從頭開始
date: 2024-08-15 12:15:00
categories: 落葉下的存檔
top_img: https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true
tags:
    - 
toc:
toc_number:
comments :
---

{% tabs Cache - 從頭開始%}

<!-- tab Node-->

村裡，有位小學徒 Yoyo，每天最常被叫去做的工作，就是幫村裡人查詢「住戶資料」。

「Yoyo，去問資料庫大師，這位用戶住哪裡？」
「Yoyo，再去查一次，那筆我剛剛沒記住。」

一開始，Yoyo 也不以為意。畢竟這是學徒的日常


「欸，再去……」


可重複個幾百次後，他終於受不了了：「為什麼我每次查完，都要重跑一樣的路？而且連名字都懶得叫了內」，有天，他開始在口袋裡放一本小筆記本，把查過的資料寫在上面。從此以後，他只要看看自己的筆記本，就能直接回應問題，不必每次都跑一趟，這本小筆記本，就是 Yoyo 發明的「快取系統」

![Cache](https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true)

<!-- endtab -->

<!-- tab 🥥 第一階段：打造屬於自己的「最簡快取機制」-->

在這一階段，我們要實作一個最基礎、卻實用的快取系統

## Lazy Load + Dictionary

現在我們選擇兩樣工具

- `Dictionary` 當作資料容器
- 一個 `Func<T>` 委派來延遲建立資料（Lazy Load）

<br>

## 實作規劃

| 元件                                 | 說明                              |
| ---------------------------------- | ------------------------------- |
| `Dictionary<string, CacheItem<T>>` | 儲存資料的容器，使用 `string` 為 Key |
| `CacheItem<T>`                     | 包裝資料本體與快取時間戳                    |
| `Func<T>`                          | 當資料不存在時，用來產生資料的委派               |
| `static` 類別設計                      | 讓整個系統可以共用同一個快取容器                |

<br>

##　資料包裝類別：CacheItem

```CSHARP
/// <summary>
/// 快取的資料容器，記錄資料與時間戳
/// </summary>
public class CacheItem<T>
{
    public T Value { get; set; }
    public DateTime CreatedAt { get; set; }
	public CacheItem(T value)
	{
		Value = value;
		CreatedAt = DateTime.Now;
	}
}
```

## 快取服務：CacheService
```CSHARP
/// <summary>
/// Yoyo 特製版快取
/// </summary>
public static class CacheService<T>
{
	/// <summary>
	/// 快取資料
	/// </summary>
	private static Dictionary<string, CacheItem<T>> _cacheData = new Dictionary<string, CacheItem<T>>();

	/// <summary>
	/// Gets or creates data
	/// </summary>
	/// <param name="key">cache key</param>
	/// <param name="createItem">delegate to create item</param>
	/// <returns>T</returns>
	public static CacheItem<T> GetOrCreate(string key, Func<T> createItem)
	{
		if (_cacheData.ContainsKey(key) == false)
		{
			//// 快取用[] = 更接近 Create or Update 的狀態不會因為誤判噴 Exception
			//// System.ArgumentException: An item with the same key has already been added.
			_cacheData[key] = new CacheItem<T>(createItem());
		}

		Console.WriteLine($"key : {key}, data : {_cacheData[key]}");
		return _cacheData[key];
	}
}
```

## 快取測試 Endpoint

```CSHARP
public static void MapCacheTestEndpoints(this IEndpointRouteBuilder app)
{
	app.MapGet("/SimpleCacheTest", () =>
	{
		var data = CacheService<string>.GetOrCreate(123, () => YoyoDB.GetUserInfo(123));
		var data2 = CacheService<string>.GetOrCreate(456, () => YoyoDB.GetUserInfo(456));
		return new Tuple<CacheItem<string>, CacheItem<string>>(data,data2);
	});
}
```

**測試畫面（GET 請求）**
![simpleCacheTest](https://github.com/CHI-KEKE/pics/blob/main/Cache/simpletest1.png?raw=true)

<!-- endtab -->

<!-- tab 🥥 關於 Thread-Safety-->

## Thread-Safe？

在 Web API 或多執行緒程式中，可能會有多個請求「同時」存取快取，因此發生同時檢查 `ContainsKey()` 為 false，然後同時執行 `_cacheData[key] = ...`，結果會出現「重複寫入」或 Key already exists 的例外！

<br>

## 試給你看

當「檢查是否存在」與「建立資料」不是同一個不可分割動作時，多個執行緒一定會同時覺得「現在輪到我建立」

```CSHARP
app.MapGet("RunCacheRaceConditionTest", () =>
{
	var tasks = new List<Task>();

	//// 建立 20 個平行 Task，「同一時間」有很多執行緒跑同一段程式
	for (int i = 0; i < 20;i++)
	{
		tasks.Add(Task.Run(() =>
		{
			var data3 = CacheService<string>.GetOrCreate(1, () => YoyoDB.GetUserInfo(1));
			return data3;
		}));
	}

	Task.WaitAll(tasks.ToArray());
});
```

```BASH
System.InvalidOperationException:
Operations that change non-concurrent collections must have exclusive access.
A concurrent update was performed on this collection and corrupted its state.
```

> 「你試圖同時改變一個非執行緒安全的集合（Dictionary），導致內部狀態毀損，Dictionary 已經壞掉了！」


所有 Task 都在做三件事

1. 檢查 _cacheData.ContainsKey(1)
2. 如果沒有，就呼叫 YoyoDB.GetUserInfo(1)
3. 把結果塞進快取

> ⚠️ 這三步沒有任何同步保護，他們彼此不知道這件事情

底層的資料結構已經進入不穩定或不一致的狀態，雜湊表內部的 `bucket index`、`linked list` 結構不一致，導致查詢時可能跳過資料、不準確或找不到值，因此在寫入或查找行為會觸發 Exception 或無法預期的行為

<!-- endtab -->

<!-- tab 🧠 鎖定-->

Thread-Safe 的本質就是「幫這個動作排隊或協調」，而 .NET 提供 Thread-Safe 解法

| 方法                     | 適用情境     | 說明               |
| ---------------------- | -------- | ---------------- |
| `lock`                 | 控制進入區段   | 精準鎖定，但容易產生效能瓶頸   |
| `ConcurrentDictionary` | 原生支援多執行緒 | 高效能內建併發控制機制，推薦使用 |

使用 `lock` 相當於自己控管「誰可以進來」，他把一段程式碼變成「臨界區」其他執行緒只能等，但鎖太大全部人卡住、而鎖太小還是會出事，假設加上 `lock` 之後

```csharp
lock (_lock)
{
    if (!_cache.ContainsKey(key))
    {
        _cache[key] = Create();
    }
}
```

它 鎖住了三件根本不該被鎖的事

- DB I/O（很慢）
- 其他 key 的請求
- 單純讀快取的請求

所以會發生這件事情

1. Thread A（key=1） 進 lock → 查 DB（300ms）
2. Thread B（key=1）卡住
3. Thread C（key=2）卡住（雖然完全不衝突）
4. Thread D（只是想讀）也卡住

因此，若真的要使用 lock，不要在 lock 裡面做「慢、不可控、外部依賴」的事

```csharp
//// 先純讀 cache
if (_cache.TryGetValue(key, out var value))
{
    return value;
}

//// 想查得 cache 沒有再鎖
var data = LoadFromDb(key);

lock (_lock)
{
    //// 再確認一次，避免重複寫入
    if (_cache.ContainsKey(key) == false)
    {
        _cache[key] = data;
    }
}

return _cache[key];
```

✔️ DB 查詢不會卡住別人，多個 thread 可以同時查 DB，不會卡住其他 key
✔️ lock 只保護「共享狀態」，真正需要保護的只有 _cache

不過這版是「效能 OK，但可能做白工」

- Thread A → 查 DB
- Thread B → 也查 DB

最後只會有一個寫進 cache，正確性沒問題，但 DB 被打多次

<!-- endtab -->

<!-- tab ConcurrentDictionary-->

> 它把「全域鎖」變成「key 級別的同步」，而 lock 鎖的是整個 Dictionary

這個選擇相當於框架幫你把「鎖」拆碎、藏起來，它不是單一一把大鎖，而是內部分段鎖定且針對操作語意（Get / Add / Update）設計，他用「分段鎖（Segmented Locking）」的方式來做到

| 機制                 | 說明                                             |
| ------------------ | ---------------------------------------------- |
| 分段桶（bucket）        | 資料分散存放在不同區塊                                    |
| 區段鎖（lock striping） | 每個區段各自鎖住，不影響其他區                                |
| 原子操作（atomic）       | 提供 `TryAdd`、`GetOrAdd`、`TryUpdate` 等具備執行緒安全的操作 |

ConcurrentDictionary 提供的語意是 GetOrAdd，這是一個不可分割的語意「沒有就加，有就拿」，所以我們再也不會寫出先檢查、再建立、再猜會不會被插隊的邏輯

<br>

## 📦 完整實作

改寫後的 Thread-Safe 版本：ThreadSafeCacheService<T>
```CSHARP
public class ThreadSafeCacheService<T>
{
	private static readonly ConcurrentDictionary<string, CacheItem<T>> _cacheData = new();
	public static CacheItem<T> GetOrCreate(string key, Func<T> createItem)
	{
		var cacheItem = _cacheData.GetOrAdd(key, _ => new CacheItem<T>(createItem()));
		Console.WriteLine($"[ThreadSafe] key: {key}, data: {cacheItem.Value}");
		return cacheItem;
	}
}
```

## 快取測試 Endpoint

```CSHARP
app.MapGet("/ThreadSafeCacheTest", () =>
{
	var tasks = new List<Task>();
	for (int i = 0; i < 20; i++)
	{
		tasks.Add(Task.Run(() =>
		{
			var data4 = ThreadSafeCacheService<string>.GetOrCreate(1, () => YoyoDB.GetUserInfo(1));
			return data4;
		}));
	}

	Task.WaitAll(tasks.ToArray());
});
```

這次就不會出現 DictionaryCorrupt 的問題了!

<!-- endtab -->

<!-- tab 🥥 過期機制-->

在現實世界裡，資料是會隨著時間改變的，而「快取」如果永不更新，就會產生過時的資訊（Stale Data）

> 🎯 因此我們應該給每筆快取「設定一個生命週期（Time-To-Live, TTL）」

| 概念                | 說明                     |
| ----------------- | ---------------------- |
| `ExpireAfter`     | 每筆資料儲存時就指定一個有效時間       |
| `DateTime.Now` 檢查 | 每次讀取資料時，檢查是否已過期        |
| `過期就重建`           | 如果過期，則重新執行委派取得新資料並更新快取 |

<br>

## 快取資料 CacheItem：新增 ExpireAfter

```CSHARP
public class CacheItem<T>
{
	public T Value { get; set; }
	public string CreatedAt { get; set; }
	public TimeSpan ExpiredAfter { get; set; } = TimeSpan.FromMinutes(5);
	public bool IsExpired
	{
		get
		{
			DateTime createdAt = DateTime.Parse(CreatedAt);
			return DateTime.Now - createdAt > ExpiredAfter;
		}
	}

	public CacheItem(T value,TimeSpan expireAfter)
	{
		Value = value;
		CreatedAt = DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss");
		ExpiredAfter = expireAfter;
	}
}
```

<br>
<br>

## ThreadSafeCacheService 加入過期邏輯

```CSHARP
public class ThreadSafeCacheService<T>
{
	private static readonly ConcurrentDictionary<string, CacheItem<T>> _cacheData = new();
	public static CacheItem<T> GetOrCreate(object key, Func<T> createItem, TimeSpan expireAfter)
	{
		_cacheData.AddOrUpdate(
			key,
			_ => new CacheItem<T>(createItem(), expireAfter),
			(_, existingItem) =>
			{
				if (existingItem.IsExpired)
				{
					Console.WriteLine($"[Expired] key: {key} 資料已過期，重新建立");
					return new CacheItem<T>(createItem(), expireAfter);
				}

				return existingItem;
			});

		var item = _cacheData[key];
		Console.WriteLine($"[Expirable] key: {key}, data: {item.Value}");
		return item;
	}
}
```

<br>
<br>

## 測試用 API Endpoint

```CSHARP
app.MapGet("/TreadSafeCacheWithTTLTest", () =>
{
	var tasks = new List<Task>();
	for (int i = 0; i < 5; i ++)
	{
		tasks.Add(Task.Run(() =>
		{
			var data5 = ThreadSafeCacheService<string>.GetOrCreate(1, () => YoyoDB.GetUserInfo(1), TimeSpan.FromSeconds(2));
		}));

		Thread.Sleep(1000);
	}

	Task.WaitAll(tasks.ToArray());
});
```

輸出結果顯示，過期後重新建立資料在快取上
```bash
##查詢資料庫: 1
[Expirable] key: 1, data: User-1
[Expirable] key: 1, data: User-1
[Expired] key: 1 資料已過期，重新建立
##查詢資料庫: 1
[Expirable] key: 1, data: User-1
[Expirable] key: 1, data: User-1
[Expired] key: 1 資料已過期，重新建立
##查詢資料庫: 1
[Expirable] key: 1, data: User-1
```

<!-- endtab -->

<!-- tab 🌟 總結-->

在這三個階段，我們已經讓快取具有了「時間的概念」，這是從單純資料存取，進化到資料新鮮度管理的關鍵。

現在已擁有的快取功能
- ✅ 快速查找：用 Dictionary 儲存資料
- ✅ 執行緒安全：使用 ConcurrentDictionary
- ✅ 過期機制：加入 ExpireAfter 自動更新

<!-- endtab -->

{% endtabs %}