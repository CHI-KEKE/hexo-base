---
title: Liskov Substitution Principle
date: 2025-09-17 09:00:03
categories: OO
top_img: https://i.imgur.com/LF7L1Ws.png
cover : https://i.imgur.com/LF7L1Ws.png
tags:
    - OO
toc:
toc_number:
comments :
---

{% tabs LSP%}


{% btn 'https://igouist.github.io/post/2020/11/oo-12-liskov-substitution-principle/',菜雞與物件導向 (12): 里氏替換原則,far fa-hand-point-right %}

<br>

{% btn 'https://code-maze.com/liskov-substitution-principle/',SOLID Principles in C# – Liskov Substitution Principle,far fa-hand-point-right %}


<!-- tab Inheritance-->

![w](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/LSP/landing.png)

在軟體設計中，繼承 (Inheritance) 的發生是再正常不過的事了，作為結構化的系統，我們可以視其為多個組件裝載而成的，當結構益龐大起來，就會想將行為類似的程式碼重用，而有了繼承的概念，但這也造成父類與子類使用上耦合造成的問題，繼承不應該只是為了「重複使用程式碼」而存在，而是應該基於行為上的一致性

因此里氏替換原則 (LSP, Liskov Substitution Principle) 就是在提醒我們

👉 子類別必須能夠替代父類別而不影響系統的正確性

換句話說，當一個地方期待父類別時，若換成子類別，程式應該還是能正常運作、不會出現違反預期的行為


![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/LSP/what_is_lsp.png)

![S](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/LSP/rule.png)

<!-- endtab -->

<!-- tab 先驗條件（Preconditions）不可加強-->


子類別不能比父類別更挑剔輸入。想像你去餐廳，主廚說：「你隨便帶食材來，我都能幫你做菜。」結果徒弟廚師說：「不行！你只能帶牛肉來，不然我不煮。」這就是把條件變得更嚴格，使用者本來以為能帶什麼都行，結果換成子類卻被限制了


![A](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/LSP/prepredonciditon.png)


<!-- endtab -->

<!-- tab 後驗條件（Postconditions）不可削弱-->


子類別的輸出不能比父類別更弱，至少要保證一樣的結果。父類（物流公司）承諾：「只要你下單，我一定送達貨物，而且保證完好無損。」子類（加盟快遞）卻說：「嗯… 我只能保證幫你送，東西壞掉不關我的事。」這樣就削弱了原本的承諾，使用者會覺得契約被打折


![A](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/LSP/postconditions.png)

<!-- endtab -->

<!-- tab 不變條件（Invariants）必須保持-->


父類別所承諾的規則，子類別不能破壞。父類（紅綠燈）規則是「綠燈才能走，紅燈一定停」。子類（某路口的燈號系統）卻偷偷改成「紅燈也可以偶爾走一下」。這樣會讓所有使用者混亂，因為他們以為規則始終一致，但事實上卻被打破了

![GG](https://github.com/CHI-KEKE/pics/blob/main/OO/LSP/invariant.png?raw=true)

<!-- endtab -->

<!-- tab 錯誤的繼承方式-->


假設我們有一個 SumCalculator，專門計算整數陣列的總和


```csharp
public class SumCalculator
{
    protected readonly int[] _numbers;

    public SumCalculator(int[] numbers)
    {
        _numbers = numbers;
    }

    public int Calculate() => _numbers.Sum(); //// 40
}
```

現在我想做一個「只計算偶數總和」的版本：
```csharp
public class EvenNumbersSumCalculator : SumCalculator
{
    public EvenNumbersSumCalculator(int[] numbers)
        : base(numbers)
    {
    }

    public new int Calculate() => _numbers.Where(x => x % 2 == 0).Sum(); //// 18
}
```

問題來了
```csharp
SumCalculator evenSum = new EvenNumbersSumCalculator(numbers); //// 呼叫父類別版本，結果卻是 40
Console.WriteLine(evenSum.Calculate());
```

當我把子類別當成父類別使用時，得到的卻是錯誤的行為。這違反了 LSP，因為 子類別無法取代父類別而保持一致性

![C](https://github.com/CHI-KEKE/pics/blob/main/OO/LSP/child_new_noexpected.png?raw=true)


<!-- endtab -->

<!-- tab 正確的繼承方式-->


為了避免這種問題，我們應該 使用多型 (Polymorphism)，讓父類別行為能被覆寫 (override)
```csharp
public class SumCalculator
{
    protected readonly int[] _numbers;

    public SumCalculator(int[] numbers)
    {
        _numbers = numbers;
    }

    public virtual int Calculate() => _numbers.Sum();
}

public class EvenNumbersSumCalculator : SumCalculator
{
    public EvenNumbersSumCalculator(int[] numbers)
        : base(numbers)
    {
    }

    public override int Calculate() => _numbers.Where(x => x % 2 == 0).Sum();
}
```

現在，即使我們用父類別的參考，依然能得到正確的結果

![C](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/LSP/override_fix.png)

<!-- endtab -->

<!-- tab 更好的抽象：設計為抽象類別-->


其實更合理的設計是從一開始就抽象化，把「計算」這件事定義成契約
```csharp
public abstract class Calculator
{
    protected readonly int[] _numbers;

    public Calculator(int[] numbers)
    {
        _numbers = numbers;
    }

    public abstract int Calculate();
}
```

然後不同的子類別各自去實作自己的邏輯
```csharp
public class SumCalculator : Calculator
{
    public SumCalculator(int[] numbers) : base(numbers) { }

    public override int Calculate() => _numbers.Sum();
}

public class EvenNumbersSumCalculator : Calculator
{
    public EvenNumbersSumCalculator(int[] numbers) : base(numbers) { }

    public override int Calculate() => _numbers.Where(x => x % 2 == 0).Sum();
}
```

LSP 關注的是「行為」而不是「程式碼重用」子類別繼承父類別時，最重要的是「能否保持契約一致」





<!-- endtab -->

<!-- tab DI 與 LSP-->


假設我們有一個 付款系統
```CSHARP
public interface IPaymentProcessor
{
    void Pay(decimal amount);
}
```

我們希望未來可以新增不同付款方式（信用卡、Line Pay、Apple Pay），而不用修改核心邏輯 → OCP


注入依賴（DI）
```CSHARP
public class CheckoutService
{
    private readonly IPaymentProcessor _paymentProcessor;

    public CheckoutService(IPaymentProcessor paymentProcessor)
    {
        _paymentProcessor = paymentProcessor;
    }

    public void Checkout(decimal amount)
    {
        _paymentProcessor.Pay(amount);
    }
}
```
CheckoutService 不關心是哪一種付款，只透過介面操作 → 依賴注入（DI）。

建立實作（LSP 是否被遵守）
```CSHARP
public class CreditCardPayment : IPaymentProcessor
{
    public void Pay(decimal amount)
    {
        Console.WriteLine($"信用卡支付 {amount}");
    }
}

public class ApplePayPayment : IPaymentProcessor
{
    public void Pay(decimal amount)
    {
        Console.WriteLine($"Apple Pay 支付 {amount}");
    }
}
```
這些類別都可以被 CheckoutService 注入，彼此可替代 → LSP 成立，系統保持穩定


![F](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/LSP/di_implementation.png)



當 LSP 被破壞
```CSHARP
public class FraudulentPayment : IPaymentProcessor
{
    public void Pay(decimal amount)
    {
        throw new Exception("我不接受大於 100 元的交易");
    }
}
```
現在如果有人透過 DI 注入 FraudulentPayment
```CSHARP
var checkout = new CheckoutService(new FraudulentPayment());
checkout.Checkout(500); // Boom! Exception!
```

FraudulentPayment 雖然「語法上」符合介面，但「行為上」卻無法替代其他實作。這破壞了 LSP。一旦 LSP 被破壞，DI + OCP 的組合就失效，因為我們沒辦法保證「注入什麼實作都能正常工作」

![C](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/LSP/payment_lsp_exception.png)


<!-- endtab -->

<!-- tab SUM-->


![S](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/LSP/decision_table_for_inheritance.png)



<!-- endtab -->

{% endtabs %}