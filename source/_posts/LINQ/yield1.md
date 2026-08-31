---
title: Yield Return - 本質
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


## 🐛 yield return 的核心本質

將「一次性完成」的方法，轉換成「逐步產生序列」的狀態機。

傳統方法：輸入 → 計算 → 輸出結果，一次性結束。
使用 yield return 方法可以 暫停、保存狀態、按需產生值，直到所有結果都產生完畢。

<br>

### 控制流的「暫停與恢復」

使用 yield return，方法不再是從頭到尾一次性執行完畢，而是允許執行到某個點時「暫停」，並記住執行上下文（局部變量、當前位置等），等待下一次繼續執行。

一般方法
```CSHARP

int[] GenerateNumbers()
{
    return new int[] { 1, 2, 3 };
}

```

使用 yield return
```CSHARP

IEnumerable<int> GenerateNumbers()
{
    yield return 1;
    yield return 2;
    yield return 3;
}

```

執行過程像這樣：

第一次 MoveNext() → 跑到 yield return 1，暫停，回傳 1
第二次 MoveNext() → 從剛停的地方繼續，回傳 2
第三次 MoveNext() → 回傳 3
沒值了 → 序列結束

📌 關鍵點： 方法不是一次跑到底，而是可以「停」下來再「繼續」。

<br>

###　狀態的持久化

一般方法結束後，裡面的變數會消失；但 yield return 方法會把狀態「保存在編譯器生成的狀態機裡」。

```CSHARP

IEnumerable<int> Counter()
{
    int count = 0;
    while (count < 3)
    {
        yield return count;
        count++;
    }
}

```

每次迭代 count 的值會被保留，下次繼續時，能從正確的地方繼續跑

**那什麼是狀態機（State Machine）？**

![Image](https://i.imgur.com/riBlhJn.jpeg)

想像編譯器幫你偷偷寫了一個類別，裡面有：

- state：記住現在程式跑到哪一行
- local variables：保存局部變數的值
- MoveNext()：控制「下一步要做什麼」

等同於你有個「劇本導演」，知道戲演到哪一幕，下次演員上台就能接著演。

<br>

### 懶加載（Lazy Evaluation）

傳統方法：一次生成所有資料。
yield return：需要時才生成資料。

```CSHARP

var orders = _context.Orders
    .Where(o => o.Amount > threshold)  // 還沒執行 背後實作 IEnumerable, yield return
    .Select(o => new Order            // 還沒執行 背後實作 IEnumerable, yield return
    {
        Id = o.Id,
        Amount = o.Amount
    });

    // 直到這裡才執行查詢
    foreach (var order in orders)
    {
        // 處理訂單
    }

```

實際上，下面這個方法在被調用時並不會執行，只是返回一個可以用來 loop 的迭代器
```CSHARP

public IEnumerable<Product> GetProducts()
{
    // 這個方法在被調用時並不會執行
    // 只是返回一個可以用來 loop 的迭代器
    for (int i = 0; i < 5; i++)
    {
        yield return new Product() { ... };
    }
}

```

編譯器會將上述的 Code 轉換為類似這樣的結構

```CSHARP

private sealed class ProductEnumerator : IEnumerator<Product>
{
    private int state = 0;
    private int current = -1;
    private Product currentItem;

    public bool MoveNext()
    {
        switch (state)
        {
            case 0:
                if (current < 4)
                {
                    current++;
                    currentItem = new Product { Id = current, ... };
                    state = 1;
                    return true;
                }
                return false;
            default:
                return false;
        }
    }

    public Product Current => currentItem;
}

```

因此第一次真的沒有去執行，因為他只是給你釣竿，沒有幫你釣魚!!!!!!!!!!!!🐟🐟🐟🐟

![Image](https://i.imgur.com/BBXPwtN.jpeg)


<br>

### 重複迭代會重跑

後面若重複使用他，例如 Count(), foreach..就會執行多次

```CSHARP

void Main()
{
	var store = new DemoStore();
	Console.WriteLine("1. 調用 GetProducts()");
	var productList = store.GetProducts();

	Console.WriteLine("\n2. 什麼都不做，只是持有 IEnumerable");

	Console.WriteLine("\n3. 開始計算 Count");
	var count = productList.Count();

	Console.WriteLine("\n4. 開始 foreach");
	foreach (var p in productList)
	{
		p.Id.Dump();
	}
}

public class DemoStore
{
	public IEnumerable<Product> GetProducts()
	{
		for (int i = 0; i < 3; i++)
		{
			Console.WriteLine($"生成產品 {i}");
			yield return new Product { Id = i };
		}
	}
}

public class Product
{
	public int Id { get; set; }
	public string Name { get; set; }
}

```

結果
![Image](https://i.imgur.com/OjhbR3x.png)

可以看到，我們在 Count、 foreach loop 都做了一組 GetProducts!


<br>

### 現實案例

**日誌檔案讀取**

File.ReadLines(path) 用 yield return → 逐行讀，不會一次載入整個檔案，適合讀 10GB log 檔，省記憶體

**資料庫分頁查詢**

用 yield return 實作「一頁一頁撈」，呼叫端邊取邊用，不必一次塞爆記憶體

**遊戲怪物生成**

yield return 每次生成一隻怪物，玩家移動到新區域時才需要生成 → 適合懶加載