---
title: Enumerator
date: 2024-06-20 23:50:10
categories: LINQ
top_img: https://i.imgur.com/cBfeEDY.png
cover : https://i.imgur.com/cBfeEDY.png
tags:
    - LINQ
toc:
toc_number:
comments :
---

## 🍔 從 foreach 官方文件開始

想要使用 foreach 的集合需要實作 System.Collections.IEnumerable or System.Collections.Generic.IEnumerable<T> interface

foreach 的真正實現，是一個迴圈控制結構，必須知道：

- 怎麼取得下一筆資料 (MoveNext)
- 怎麼拿到目前這筆資料 (Current)
- 怎麼從頭開始 (GetEnumerator)

這一切都來自於 Iterator 模式，也就是 .NET 中 IEnumerable<T> / IEnumerator<T> 的設計核心。

<br>
<br>

## 🍔 有一天我在摩斯排隊點餐，但前面點餐的大叔點有點久，於是我開始發白日夢...

一位客人
```csharp

public class FatLaw
{
	public int Number {get;set;}
	public string [] Menu {get;set;}
}

```

排隊點餐的一群客人
```csharp

// FatLawQueue 實作了 IEnumerable<FatLaw>，這是使用 foreach 的要求。
public class FatLawQueue : IEnumerable<FatLaw>
{
	private List<FatLaw> fatlaws = new List<FatLaw>();
	
	public void Enqueue(FatLaw fatlaw)
	{
		fatlaws.Add(fatlaw);
	}
	

	public IEnumerator<FatLaw> GetEnumerator()
	{
		return new FatLawEnumerator(fatlaws);
	}
	
	// 實作IEnumerable, 必須實作GetEnumerator		
	// foreach時，編譯器會自動調用 GetEnumerator 來獲取 FatLawEnumerator，才知道怎麼控制依序處理的過程
	IEnumerator IEnumerable.GetEnumerator()
	{
		return GetEnumerator();
	}

	private class FatLawEnumerator : IEnumerator<FatLaw>
	{
		private List<FatLaw> _fatlaws;
		private int _currentIndex = -1;
		
		public FatLawEnumerator(List<FatLaw> fatlaws)
		{
			_fatlaws = fatlaws;
		}

        // 確認自己在哪裡
		public FatLaw Current => _fatlaws[_currentIndex];

		object IEnumerator.Current => Current;

        // 往下一個 Item 處理，已經沒有下一個返回 false
		public bool MoveNext()
		{
			_currentIndex ++;
			return _currentIndex < _fatlaws.Count;
		}

		public void Reset()
		{
			_currentIndex = -1;
		}
		
		public void Dispose() => _fatlaws = null;
	}
}
```

在沒有 foreach 之前，開發者會這樣寫
```csharp
var enumerator = fatLawQueue.GetEnumerator();
while (enumerator.MoveNext())
{
    var item = enumerator.Current;
    // 處理 item
}
```

現在集合只要能「一個一個地走訪」，就能用 foreach。
```csharp

var fatLawQueue = new FatLawQueue();
fatLawQueue.Enqueue(new FatLaw() { Number = 1, Menu = new string[] {"雙人分享餐","雪碧"}});
fatLawQueue.Enqueue(new FatLaw() { Number = 2, Menu = new string[] {"摩斯雞塊豪雪套餐","可樂"}});
fatLawQueue.Enqueue(new FatLaw() { Number = 3, Menu = new string[] {"黃金炸蝦堡","可可"}});
fatLawQueue.Enqueue(new FatLaw() { Number = 4, Menu = new string[] {"藜麥杏鮑菇珍珠堡","摩斯咖啡"}});

foreach(var fatLaw in fatLawQueue)
{
    "♬ ♬ ♩  ♪ ♪ ♫ ♭ ♫  ♪ ♪ ♫ ♭  ♫ ♭ ♭ ♫  ♪ ♪  \n".Dump();
    $"歡迎光臨，肥佬 {fatLaw.Number} 號，請問今天吃甚麼 ? ".Dump();
    $"{string.Join(" ,", fatLaw.Menu)} 是吧，客人吃有點多喔 ".Dump();
    $"收您 {fatLaw.Money}，謝謝光臨\n".Dump();
}

/**

♬ ♬ ♩  ♪ ♪ ♫ ♭ ♫  ♪ ♪ ♫ ♭  ♫ ♭ ♭ ♫  ♪ ♪ 

歡迎光臨，肥佬 1 號，請問今天吃甚麼 ? 
雙人分享餐 ,雪碧 是吧，客人吃有點多喔 
收您 1000，謝謝光臨

♬ ♬ ♩  ♪ ♪ ♫ ♭ ♫  ♪ ♪ ♫ ♭  ♫ ♭ ♭ ♫  ♪ ♪ 

歡迎光臨，肥佬 2 號，請問今天吃甚麼 ? 
摩斯雞塊豪雪套餐 ,可樂 是吧，客人吃有點多喔 
收您 1000，謝謝光臨

♬ ♬ ♩  ♪ ♪ ♫ ♭ ♫  ♪ ♪ ♫ ♭  ♫ ♭ ♭ ♫  ♪ ♪ 

歡迎光臨，肥佬 3 號，請問今天吃甚麼 ? 
黃金炸蝦堡 ,熱可可 是吧，客人吃有點多喔 
收您 1000，謝謝光臨

♬ ♬ ♩  ♪ ♪ ♫ ♭ ♫  ♪ ♪ ♫ ♭  ♫ ♭ ♭ ♫  ♪ ♪ 

歡迎光臨，肥佬 4 號，請問今天吃甚麼 ? 
藜麥杏鮑菇珍珠堡 ,摩斯咖啡 是吧，客人吃有點多喔 
收您 1000，謝謝光臨

**/
```

![Image](https://i.imgur.com/LKsAWAL.png)

<br>
<br>

## 🍔 改用 yield

在 MoveNext & Current 遍歷處理上，也可以用 yield return 來取代，它可以保留位置 & 狀態，下一次呼叫 MoveNext () 時，會從當前位置 & 狀態繼續執行。如此一來，遍歷的作法進一步被封裝起來，且省去自訂義 Enumerator的時間

yield return 是 C# 編譯器幫你自動產生一個類似你自己手寫的 IEnumerator<T> 類別，它自動幫你記住「目前位置、狀態」，所以你不需要自己實作 MoveNext()、Current 等細節。因為 yield return 會把你的方法變成一個「可中斷又可繼續的流程（狀態機）」，這種設計叫做 Compiler-generated State Machine（編譯器產生的狀態機），可以把它想成：每遇到一個 yield return，編譯器都幫你標記一個中斷點（checkpoint），而在下一次呼叫 MoveNext() 時，它就從上一個中斷點「繼續往下執行」
```csharp

public IEnumerator<FatLaw> GetEnumerator()
{
    foreach (var fatLaw in _fatLaws)
    {
        yield return fatLaw;
    }
}

```

## 🍔 參考文章

{% btn 'https://learn.microsoft.com/zh-tw/dotnet/api/system.collections.generic.ienumerable-1?view=net-8.0',IEnumerable<T> 介面,far fa-hand-point-right %}

<br>

{% btn 'https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/iteration-statements?redirectedfrom=MSDN',Iteration statements - for, foreach, do, and while,far fa-hand-point-right %}

<br>

{% btn 'https://en.wikipedia.org/wiki/Iterator_pattern',Iterator pattern,far fa-hand-point-right %}