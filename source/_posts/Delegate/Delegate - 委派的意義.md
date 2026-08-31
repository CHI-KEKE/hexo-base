---
title: Delegate - 委派的意義
date: 2025-09-23 23:07:34
categories: 風音子
top_img: https://i.imgur.com/zTSkblP.png
cover : https://i.imgur.com/zTSkblP.png
tags:
    - 風音子
toc:
toc_number:
comments :
---

![Image](https://i.imgur.com/zTSkblP.png)

在 .NET 裡，委派 (Delegate) 本質上就是一個「可以存放方法的變數」。像你平常會用 int x = 5; 來存一個數字，委派就是一個「方法變數」，它可以存「方法的位址」，以便之後呼叫。
委派提供了「把方法包起來，交給別人去執行」的能力。

<br>

## 🧚🏻 委派與 Lambda 的關係

Lambda 表示式 (Lambda Expression)，是一種「快速寫法」來建立匿名方法 (Anonymous Method)。這些 Lambda 會自動轉換成委派物件，或轉成 Expression Tree (取決於用在哪裡)。而在 LINQ 裡，我們常常看到 .Where(x => x > 10) 這種 Lambda，它背後就是：Where 方法需要一個委派 (通常是 Func<T, bool>)，Lambda 就是一個「方便產生委派」的語法糖。

<br>

### 傳統委派寫法
```csharp
// 定義一個委派型別
delegate bool MyFilter(int number);

class Program
{
    static void Main()
    {
        // 用一般方法
        MyFilter filter = IsGreaterThanTen;
        Console.WriteLine(filter(15)); // 輸出 True
    }

    static bool IsGreaterThanTen(int n)
    {
        return n > 10;
    }
}
```

<br>

###　用 Lambda 簡化
```csharp
MyFilter filter = n => n > 10;
Console.WriteLine(filter(15)); // 輸出 True
```

### LINQ 搭配 Lambda
```csharp
var numbers = new List<int> { 5, 12, 7, 20 };
var result = numbers.Where(n => n > 10);
foreach (var item in result)
{
    Console.WriteLine(item); // 輸出 12, 20
}
```

這裡的 n => n > 10，就是一個 Lambda，背後轉換成了 Func<int, bool> 這種委派型別，符合 Where 的參數需求。

<br>
<br>

## 🧚🏻 委派在異步操作 (Asynchronous Operation) 的角色

在 C# 早期（還沒有 async/await 之前），委派本身就可以「包裝方法」並支援非同步呼叫。可以用 BeginInvoke / EndInvoke 在背景執行方法，類似「開一條小幫手線程」。
雖然現在已經被 Task-based Async (Task, async/await) 取代，但這個歷史脈絡能幫助理解：委派不只是同步呼叫，也能異步呼叫。

另外，現在即使是 Task.Run(() => DoSomething())，這個 () => DoSomething() 也是一個 Lambda，背後仍然是委派 (Action)。
換句話說，非同步的核心也是靠委派把「要做的事情」包起來，交給 Task 排程去跑。

```csharp
Task.Run(() => LongRunningTask(10));
```

<br>
<br>

## 🧚🏻 委派在事件處理

事件 (Event) 在 C# 裡本質就是 一種特殊的多播委派 (Multicast Delegate)。

多播委派 = 可以同時「訂閱」多個方法。

當事件觸發時，所有訂閱的方法會依序被呼叫。所以事件處理機制其實就是靠委派來完成的

- 事件 (Event) 是一個「公開的入口」，讓別人註冊方法。
- 委派 (Delegate) 是「方法的容器」，存放訂閱者的方法清單。
- 觸發事件 就是「逐一呼叫委派裡的方法」。

```csharp
class Button
{
    // 定義事件 (背後是委派)
    public event Action Click;

    public void SimulateClick()
    {
        Console.WriteLine("Button 被點擊！");
        Click?.Invoke(); // 呼叫委派，通知所有訂閱者
    }
}

class Program
{
    static void Main()
    {
        var button = new Button();

        // 訂閱事件（就是把方法加到委派裡）
        button.Click += () => Console.WriteLine("事件處理器 A");
        button.Click += () => Console.WriteLine("事件處理器 B");

        button.SimulateClick();
        // 輸出：
        // Button 被點擊！
        // 事件處理器 A
        // 事件處理器 B
    }
}
```