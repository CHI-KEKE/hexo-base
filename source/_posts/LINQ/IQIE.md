---
title: IEnumerable<T> and IQueryable<T>
date: 2024-10-21 08:02:11
categories: LINQ
top_img: https://i.imgur.com/cBfeEDY.png
cover : https://i.imgur.com/cBfeEDY.png
tags:
    - LINQ
toc:
toc_number:
comments :
---


## IQueryable

在 IDE 中寫好的 IQueryable 只是 "查詢狀態"，此時還沒執行資料庫的查詢，因此不會有資料載入記憶體的行為。若指派某些會得到 "明確結果" 的 function，如 Count ()、ToList () 等，此時才會執行 SQL 查詢指令，取得查詢結果。
IQueryable 介面會將 Expression 傳遞給 Provider，由 Provider 轉譯成 T-SQL 後，從 DB 中取得資料，得到 "明確結果"

```CSHARP

// 此時的data是"查詢狀態"，資料還未載入記憶體中。
IQueryable<Book> Qdata = dbContext.Book;

// 在定義變數時加入條件式，也不會將"查詢狀態"載入記憶體中。
IQueryable<Book> filterQdata = dbContext.Book.Where(x => x.Id == 1);

//下面兩行程式才會執行SQL指令，並將查詢的資料載入記憶體中。
var book = filterData.ToList();
var book = filterData.Count();

```

![Image](https://i.imgur.com/d1ACAOk.png)

IQueryable<T> 繼承自 IEumerable<T>，一樣具有 Enumerability 特性，也就是他們都具有可以被逐一走訪的能力，IEnumerable 有的功能它都有，一樣都不許少，且都有延遲執行的效果，白話文就是，只是一個 "可以執行的狀態"，想像一下，寫出來會有一個小精靈淚眼汪汪的盯著你看等待著你下一步指令...

![Image](https://i.imgur.com/w8FWb0F.png)

IQueryable 與 IEnumerable 最大的不同點在於，背後要有一個 Query Provider (例如 LINQ to SQL、Oracle EF Data Provider...)、且它能保存 Query Expression，允許稍後繼續加工調整查詢邏輯，直到最後要列舉成具體資料時，再將最後版本的 Query Expression 交由 Query Provider 轉換成實際可在資料庫執行的 SQL 語法，執行後取得資料，產生結果。


## IQueryable<T> 存在的意義

IEnumerable<T> = 「結果已經確定」= 在 C# 記憶體裡操作
IQueryable<T> = 「查詢還沒執行」= 可以一直組條件，最後由 DB 轉 SQL 執行

本質就是用資料庫的 CPU 來幫你過濾、排序、分頁，而不是把所有資料拉回來自己算。


想像你要找「姓王、年紀小於 30 歲的員工」

**IEnumerable<T>全撈再過濾**

你先把公司所有員工資料都搬到你家客廳，然後自己用 Excel 過濾「姓王」+「小於 30 歲」，得到正確結果，但浪費力氣、客廳被塞爆

**IQueryable<T>（下 SQL 指令讓資料庫處理）**

你直接告訴 HR 系統：「我要姓王而且小於 30 歲的員工」，系統只給你符合條件的清單，搬運資料少，速度快，省記憶體


## IEnumerable 案例

```csharp
// 先全撈 (ToList 馬上執行 SQL)
IEnumerable<Product> products = context.Products.ToList();

// 在記憶體篩選 (LINQ to Objects)
var result = products.Where(p => p.UnitPrice < 20 && p.CategoryID == 6).Count();
```

```SQL

exec sp_executesql N'SELECT 
[Extent1].[ProductID] AS [ProductID], 
[Extent1].[ProductName] AS [ProductName], 
[Extent1].[SupplierID] AS [SupplierID], 
[Extent1].[CategoryID] AS [CategoryID], 
[Extent1].[QuantityPerUnit] AS [QuantityPerUnit], 
[Extent1].[UnitPrice] AS [UnitPrice], 
[Extent1].[UnitsInStock] AS [UnitsInStock], 
[Extent1].[UnitsOnOrder] AS [UnitsOnOrder], 
[Extent1].[ReorderLevel] AS [ReorderLevel], 
[Extent1].[Discontinued] AS [Discontinued]
FROM [dbo].[Products] AS [Extent1]
WHERE [Extent1].[UnitPrice] < @p__linq__0',N'@p__linq__0 decimal(2,0)',@p__linq__0=20

```

DB 把一大堆產品都吐回來，C# 再慢慢計算 CategoryID == 6 和 Count。

## IQuerable 案例

```csharp
// IQueryable (延遲執行)
IQueryable<Product> products = context.Products;

// 條件會被轉成 SQL
var result = products.Where(p => p.UnitPrice < 20 && p.CategoryID == 6).Count();
```

EF 會產生的 SQL（Count + 條件全部下到 DB）
```SQL

exec sp_executesql N'SELECT 
[GroupBy1].[A1] AS [C1]
FROM ( SELECT 
    COUNT(1) AS [A1]
    FROM [dbo].[Products] AS [Extent1]
    WHERE ([Extent1].[UnitPrice] < @p__linq__0) AND (6 = [Extent1].[CategoryID])
)  AS [GroupBy1]',N'@p__linq__0 decimal(2,0)',@p__linq__0=20

```

| 特性           | IEnumerable<T>         | IQueryable<T>                       |
| ------------ | ---------------------- | ----------------------------------- |
| 執行時機         | 馬上執行 SQL（通常 ToList 時）  | 延遲執行（等 ToList/First/Count）          |
| 過濾 / 排序 / 分頁 | 在 **記憶體** 做            | 在 **資料庫** 做                         |
| SQL 產生       | 只負責取資料                 | 會根據條件自動生成更精簡的 SQL                   |
| 效能           | 大量資料時浪費網路、記憶體、CPU      | 只取需要的資料，效能最佳                        |
| 適合情境         | 已經在記憶體中的集合（List、Array） | DB 查詢（Entity Framework、LINQ to SQL） |


## 立即執行的動詞

Immediate Execution : foreach, ToArray(), ToList(), Min(), Max(), Count()
Deferred Execution : GroupBy(), OrderBy(), Include(), Skip(), Take()

## 參考文章

{% btn 'https://hackmd.io/@AndyShih/SJOlljYI2',IEnumerable 與 IQueryable,far fa-hand-point-right %}

<br>
<br>

{% btn 'https://blog.darkthread.net/blog/iqueryable-experiment',關於 IQueryable 特性的小實驗,far fa-hand-point-right %}

<br>
<br>

{% btn 'https://www.youtube.com/watch?v=EWB3POlVkGk',Part 3 How to view LINQ to SQL generated SQL queries,far fa-hand-point-right %}
