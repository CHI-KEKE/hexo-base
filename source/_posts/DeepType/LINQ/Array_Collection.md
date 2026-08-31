---
title: Array - Collection
date: 2024-08-15 12:15:00
categories: 
top_img: https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true
tags:
    - 
toc:
toc_number:
comments :
---


{% tabs Array - Collection%}


Array 是「具體資料結構」，而 `IEnumerable<T>`（以及 LINQ）是 **「逐一枚舉資料的抽象」**


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/LINQ/Array_Collection/2_array_concrete.png)


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/LINQ/Array_Collection/2_ienumerable_is_pipe.png)


<!-- tab Array（T[]）是什麼？-->

當我們有一塊連續記憶體的資料結構（大小固定），就會有 Lengthf、values[i] 直接索引（O(1)），這是真正「已經全部在手上」的一批資料，適合需要高速索引、固定長度、確定要把資料一次放好的場景


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/LINQ/Array_Collection/2_array_T.png)


<!-- endtab -->

<!-- tab IEnumerable（IEnumerable<T>）是什麼？-->


它不是資料結構，它是「你可以被 foreach 一個個拿出來」的能力（介面），可能背後是 array / list / db query / 檔案讀取 / generator / yield return…，重點在於，它允許 deferred execution（延遲執行），也就是寫了 Select/Where 不代表立刻跑，是 foreach / ToList() / Count() 的那刻才真正開始取資料

這適合想要串接處理流程、資料可能很大、資料來源不是一次就全部拿到的場景（像 DB、串流、yield）


<!-- endtab -->

<!-- tab 為什麼 Enum.GetValues 回傳的是 Array（而且是非泛型）-->


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/LINQ/Array_Collection/1_why_enumvalures_no_linq.png)


`Enum.GetValues(Type)` 是比較早期的 API，為了能支援「任何 enum 型別」，它回傳 Array（非泛型）。所以才需要


```csharp
values.Cast<PromotionEngineTypeEnum>()
```


`Cast<T>()` 的作用是：把「非泛型可枚舉（`IEnumerable`）」裡面的每個元素，逐一轉成 T，變成 `IEnumerable<T>`，LINQ 才能舒服地用


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/LINQ/Array_Collection/1_cast_to_generic.png)


```CSHARP
[Flags]
public enum PromotionEngineTypeEnum
{
	/// <summary>
	/// 第N件打折
	/// </summary>
	DiscountNthPieceWithRate = 1,

	/// <summary>
	/// 第N件固定價
	/// </summary>
	DiscountNthPieceWithPrice = 2,

	/// <summary>
	/// 第N件折現
	/// </summary>
	DiscountNthPieceWithAmount = 4
}
```

```CSHARP
var values = Enum.GetValues(typeof(PromotionEngineTypeEnum)); //// Array 非泛型集合
values.Cast<PromotionEngineTypeEnum>().Select(x => x.ToString()).Dump(); //// Cast 明確轉化為 IEnumeable<Enum>
//// 當然這裡也可以直接操作 foreach
```

用途例如


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/LINQ/Array_Collection/3_1_check_enum.png)


```csharp
//// 檢查 request
var allowed = new HashSet<string>(
    Enum.GetValues(typeof(PromotionEngineTypeEnum))
        .Cast<PromotionEngineTypeEnum>()
        .Select(e => e.ToString()),
    StringComparer.OrdinalIgnoreCase);

string input = "DiscountNthPieceWithRate"; // 來自設定檔或 request

if (!allowed.Contains(input))
{
    throw new ArgumentException($"Unknown PromotionEngineTypeEnum: {input}");
}


//// 把 enum 值做排序、過濾，再輸出到 log / Dump
var result = Enum.GetValues(typeof(PromotionEngineTypeEnum))
    .Cast<PromotionEngineTypeEnum>()
    .OrderBy(e => (int)e)
    .Where(e => (int)e >= 2)
    .Select(e => $"{(int)e} - {e}")
    .ToList();

//// 後台 UI 下拉選單（value + text）
var items = Enum.GetValues(typeof(PromotionEngineTypeEnum))
    .Cast<PromotionEngineTypeEnum>()
    .Select(e => new
    {
        Value = (int)e,
        Text = e.ToString()
    })
    .ToList();
```


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/LINQ/Array_Collection/3_2_dropdown.png )


<!-- endtab -->

<!-- tab LINQ 不是「不能用在 Array」，是因為遇到「非泛型」問題-->


其實 T[] 本身就能 LINQ

```csharp
int[] a = { 1, 2, 3 };
a.Select(x => x * 2)
```

`Enum.GetValues(typeof(PromotionEngineTypeEnum))`不能直接 Select 的原因是 values 的型別是 Array（非泛型），我們需要先變成 `IEnumerable<PromotionEngineTypeEnum>` 才行


<!-- endtab -->

<!-- tab 選 Array 還是 IEnumerable-->

## 「第 N 個元素」→ 用 `Array/List（索引能力強）`

已經把資料撈完（或快取好）放在記憶體，現在要直接拿第 0、1、2…筆做事
```csharp
var topUsers = new List<string> { "Allen", "Ben", "Cindy", "Dora" };

// 你要「第 3 名」(index=2)
var third = topUsers[2];

// 你要「最後一名」
var last = topUsers[^1];

// 你要「第 10 筆」：如果用 IEnumerable，你得走訪 10 次；List/Array 直接 O(1)
```



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/LINQ/Array_Collection/4_index.png)


## 「串接一堆處理邏輯」→ 用 `IEnumerable（管線思維）`

只要近 7 天、付款成功、金額 > 1000 的訂單，最後只輸出你要的欄位
IEnumerable<Order> orders = GetOrdersFromSomewhere(); // 可能是記憶體、可能是 DB 查詢結果

```csharp
var reportRows = orders
    .Where(o => o.CreatedAt >= DateTime.Today.AddDays(-7))
    .Where(o => o.PayStatus == PayStatus.Paid)
    .Where(o => o.TotalAmount > 1000)
    .OrderByDescending(o => o.TotalAmount)
    .Select(o => new ReportRow(o.Id, o.MemberId, o.TotalAmount));
```



![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/LINQ/Array_Collection/4_2_pipe.png)


## 「馬上 materialize（實體化）成集合」→ 最後 `ToArray() / ToList()`


避免「資料來源很貴」被重複讀（DB / 檔案 / 網路）
```csharp
var expensiveQuery = GetOrdersFromDb()      // 假設這裡會真的打 DB
    .Where(o => o.PayStatus == PayStatus.Paid)
    .ToList(); // 只查一次，結果固定

// 後面可以重複用，不會每次都再打 DB/再枚舉一次
var count = expensiveQuery.Count;
var max = expensiveQuery.Max(o => o.TotalAmount);

```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/LINQ/Array_Collection/4_3_materizl.png)



## 「避免一次載入大量資料」→ `IEnumerable + 延遲執行（例如讀檔、DB query）`
  
DB（EF Core）延遲執行：先組查詢，真的需要時才
```csharp
// IQueryable 也是 IEnumerable 的一種「更強版本」（可以翻譯成 SQL）
var query = db.Orders
    .Where(o => o.CreatedAt >= DateTime.Today.AddDays(-7))
    .Where(o => o.PayStatus == PayStatus.Paid);

// 這裡才真的送到 DB 執行
var page = query
    .OrderByDescending(o => o.CreatedAt)
    .Take(50)
    .ToList();
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/LINQ/Array_Collection/4_4_defferred.png)


<!-- endtab -->


<!-- tab 選 Array 還是 IEnumerable-->


![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/LINQ/Array_Collection/5_final.png)


![b](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/LINQ/Array_Collection/arrr.png )


<!-- endtab -->


{% endtabs %}