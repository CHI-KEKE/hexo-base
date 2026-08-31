---
title: IEnumerable
date: 2024-10-24 15:04:11
categories: LINQ
top_img: https://i.imgur.com/cBfeEDY.png
cover : https://i.imgur.com/cBfeEDY.png
tags:
    - LINQ
toc:
toc_number:
comments :
---

## 🐛 Enumerator & Enumerable

<br>

### Enumerator

**就是一個「游標（cursor）」，可以指向序列中的某一筆資料，並且知道「怎麼移到下一筆」、「現在這一筆是誰」。**

是一個「讀取器」，負責「一筆一筆往前走」，只能唯讀 (read-only)、往前走 (forward-only)

實作的定義是

- MoveNext() - 移到下一筆，回傳是否還有資料
- Current - 取得目前資料
- Reset() - 回到初始狀態（很少用）

介面：IEnumerator、IEnumerator<T>


<br>

### Enumerable

**就是一個「序列的抽象描述」，它本身不是游標，但是它可以「生出一個 Enumerator」來幫你走訪它。**

是一個「集合的抽象表示」，它不是游標，不會自己動，但它要能生出 Enumerator，讓你真的能走訪它

實作的定義是

- GetEnumerator()

介面：IEnumerable、IEnumerable<T>

<br>
<br>

## 🐛 IEnumerable & IEnumerator

若希望資料集合類別具有走訪能力，需要實作 IEnumerable 以及 IEnumerator 兩個介面 (或是它們的泛型版本)

```CSHARP

public interface IEnumerable
{
    IEnumerator GetEnumerator();
}

public interface IEnumerator
{
    bool MoveNext();
    object Current { get; }
    void Reset();
}

```


![Image](https://i.imgur.com/EKBUWbB.png)

- Current 屬性：回傳目前走訪到的成員內容值.
- MoveNext () : 走訪到下一個成員，並回傳 bool 值來告知向下移動是否成功.
- Reset : 重置走訪的位置.

<br>
<br>

## 🐛 實作

foreach 能夠作用的條件是：物件必須 可被列舉 (Enumerable)。
而「可列舉」的定義，就是這個物件要能產生一個 Enumerator (游標)，讓 foreach 能透過它 逐筆往前走。

實作規劃

1. FiveElements (Enumerable) 

- 就像「一個書架」，它自己不會動，但它能產生一個「借書證」(Enumerator)
- 這個「借書證」能一格一格掃過去，拿到「金、木、水、火、土」

2. FiveElementsEnumerator (Enumerator)

- MoveNext() → 游標往下一格
- Current → 現在這格的東西
- Reset() → 游標回到起點

3. foreach
只是語法糖（糖衣），背後會自己呼叫 GetEnumerator()、MoveNext()、Current。

```CSHARP

public class FiveElements : IEnumerable
{
     private string[] fiveElements = { "金", "木", "水", "火", "土" };
     public IEnumerator GetEnumerator() => new FiveElementsEnumerator(fiveElements);
}

```


自訂 Enumerator
```CSHARP

public class FiveElementsEnumerator : IEnumerator
{
     private string[] fiveElements;
     private int index = -1;
     public FiveElementsEnumerator(string[] elements) => fiveElements = elements;
     public bool MoveNext() => ++index < fiveElements.Length;
     public object Current => fiveElements[index];
     public void Reset() => index = -1;
}

```

用 yield return
```csharp
public IEnumerator<string> GetEnumerator()
{
    foreach (var element in fiveElements)
    {
        yield return element; // ✅ 一個一個吐出來
    }
}
```

呼叫
```CSHARP

static void Main(string[] args)
{
     var fiveelements = new FiveElements();
     var enumerator = fiveelements.GetEnumerator();
     while (enumerator.MoveNext())
     {
        Console.Write(enumerator.Current);
     }

     Console.WriteLine();

     foreach (var item in fiveelements)
     {
          Console.Write(item);
     }

     Console.ReadKey();
}

```

![Image](https://i.imgur.com/Ug4uyMR.png)

<br>
<br>

## 🐛 繼承關係

![Image](https://i.imgur.com/CbvaEpS.png)

根據上圖，我們可以知道常用的集合介面 (ICollection、IList、IDictionary) 都有繼承 IEnumerable 或是 IEnumerable<T> , 也因此實作這些集合界面的類別，也都具備 loop 的能力.

<br>
<br>

## 🐛 Multiple Enumeration Issues

若過沒有確實實體化 (Materialize) 成資料集合，可能導致多次重複取資料的情形，可能導致性能問題 & 資料不一致的風險

```CSHARP

public IEnumerable<int> GetNumbers()
{
    // Simulating an expensive operation
    for (int i = 0; i < 5; i++)
    {
        Console.WriteLine("Generating number: " + i);
        yield return i;
    }
}

public void Example()
{
    var numbers = GetNumbers();
    
    // This will enumerate the sequence twice
    Console.WriteLine("Count: " + numbers.Count()); // Expensive operation
    foreach (var num in numbers) // Expensive operation again
    {
        Console.WriteLine(num);
    }
}

```