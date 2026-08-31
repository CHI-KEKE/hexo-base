---
title: Polymorphism
date: 2025-09-16 08:00:03
categories: OO
top_img: https://i.imgur.com/LF7L1Ws.png
cover : https://i.imgur.com/LF7L1Ws.png
tags:
    - OO
toc:
toc_number:
comments :
---

{% tabs Encapsulation%}

{% btn 'https://www.reddit.com/r/csharp/comments/12jaxve/can_someone_explain_to_me_all_the_different_types/',Can someone explain to me all the different types of Polymorphism in C#?,far fa-hand-point-right %}



<!-- tab 巨人之力-->


![landing](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/Polymorphism/Landing.png)


**尤米爾·弗里茲**是最初的巨人之力擁有者。  
在她死後，國王為了維持這股力量，命令她的三個女兒 ── **瑪莉亞、羅塞、希娜** ── 吃下分屍後的母體。於是，巨人之力被分散成 **九大巨人**：始祖、進擊、超大型、鎧甲、女巨人、獸、戰鎚、顎、車力


![split](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/Polymorphism/split_titan.png)


諫山創實作了**多型（Polymorphism）**

- 同樣都是「巨人」，卻能展現出完全不同的外觀、能力與戰鬥風格。  
- 基底類別 `Titan` 定義了巨人的共通特性，而每個具體巨人（ColossalTitan、ArmoredTitan...）則是子類別，**實作或覆寫不同的行為**  
- 後期的 Eren 同時擁有「進擊的巨人」、「始祖巨人」、「戰鎚巨人」的能力，就像**一個物件同時實作多個介面，或覆寫了不同的行為** 
- 共通限制：因為尤米爾只活了 13 年，她的繼承者也都受「13 年壽命」的約束，這就像**基底類別定義的共同規則，所有子類別都必須遵守**


![split_overtime](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/Polymorphism/split_overtime.png)


<!-- endtab -->

<!-- tab Static (Compile-time) Polymorphism-->

首先是靜態多型（Compile-time Polymorphism）中的「方法多載（Method Overloading）」，方法多載的本質，是讓「同一個行為概念」在不同輸入條件下，有最貼近語意的實作，而不是逼使用者記一堆不同方法名稱

在巨人與人類型態，不同角色都會「攻擊」，但攻擊方式可能帶有不同條件（武器、對象、力量），我們先定義一個共同的行為名稱：Attack 代表「攻擊」這個抽象行為，不管是人、巨人、還是拿武器

為了實現不同的攻擊行為， overload 用參數型別與數量來區分不同情境

- 沒參數 → 最基本、預設的攻擊方式
- string target → 有明確攻擊對象
- int titanStrength → 以力量為核心的攻擊
- string weapon + int strength → 特殊狀態＋額外條件

編譯器在編譯期就完成「要叫哪一個 Attack」的判斷，因此呼叫端一寫出參數，對應的方法版本就已經被決定，執行時不需要再判斷 if / switch，每個方法各自專注處理自己的邏輯，不用在一個方法裡塞一堆條件分支，實現了每個版本語意清楚、責任單一的目的

```csharp
public class Warrior
{
    // 基本攻擊
    public void Attack()
    {
        Console.WriteLine("普通士兵揮刀攻擊！");
    }

    // 攻擊巨人（需要立體機動裝置）
    public void Attack(string target)
    {
        Console.WriteLine($"士兵使用立體機動裝置攻擊 {target} 的後頸！");
    }

    // 巨人攻擊（單純肉搏）
    public void Attack(int titanStrength)
    {
        Console.WriteLine($"巨人用力量 {titanStrength} 的拳頭揮擊！");
    }

    // 巨人 + 特殊武器（戰鎚巨人）
    public void Attack(string weapon, int titanStrength)
    {
        Console.WriteLine($"戰鎚巨人用 {weapon} 發動力量 {titanStrength} 的攻擊！");
    }
}
```

同樣是 Attack()，但編譯器會根據呼叫時的參數，決定實際使用哪個版本

- 普通士兵揮刀（Attack()）
- 士兵 + 立體機動裝置 → 特化對巨人的攻擊（Attack(string target)）
- 巨人肉搏（Attack(int titanStrength)）
- 戰鎚巨人召喚武器（Attack(string weapon, int titanStrength)）

👉 靜態多型的關鍵在於在程式「執行前」就能決定要呼叫哪個方法


## 具體的商業實務情境 - 金流系統的「請款（Charge）」流程

假設你在做一個金流／帳務系統，有一個核心行為叫「向客戶請款（Charge）」

不管前面接的是訂單系統、訂閱系統、還是人工後台，最後都會走到這個動作
但實務上會出現這幾種「資料完整度」

- 已知客戶與金額 → 立即請款
- 已知客戶、金額、付款方式 → 指定管道請款
- 只知道付款 Token（例如信用卡 Token） → 快速請款
- 已有授權（Auth）結果 → Capture 請款

這不是角色問題，也不是會員問題，而是**「現在你手上有多少資訊」的問題**


#### Step 1：先定義真正的商業行為名稱

```csharp
Charge(...)
```

不是：

ChargeByCustomer
ChargeByToken
CaptureAuthorizedPayment

因為在帳務語言裡，它們本質上都是「收錢」


#### Step 2：用參數「明確表達」目前已知的商業狀態

```csharp
public class PaymentService
{
    // 已知客戶與金額
    public void Charge(Guid customerId, decimal amount) { }

    // 指定付款方式
    public void Charge(Guid customerId, decimal amount, PaymentMethod method) { }

    // 只知道付款 Token（例如前端一次性付款）
    public void Charge(string paymentToken, decimal amount) { }

    // 已有授權結果，只做 Capture
    public void Charge(string authorizationId) { }
}
```

這裡的重點是方法簽名就是「現在系統知道什麼」的誠實呈現


#### Step 3：編譯器在這裡扮演「流程守門員」

你不可能寫出這種曖昧不清的呼叫 Charge(null, 1000);


#### Step 4：每一條 Charge 路徑都能對應到乾淨的實作

```csharp
public void Charge(string authorizationId)
{
    // 驗證授權狀態
    // 呼叫金流 Capture API
    // 記帳
}

public void Charge(Guid customerId, decimal amount)
{
    // 找出預設付款方式
    // 進行即時請款
}


#### 為什麼不用「一個 Charge + 很多 optional 參數」

```csharp
public void Charge(
    Guid? customerId = null,
    string? paymentToken = null,
    string? authorizationId = null,
    decimal? amount = null,
    PaymentMethod? method = null)
{
    if (authorizationId != null) { }
    else if (paymentToken != null && amount != null) { }
    else if (customerId != null && amount != null) { }
}
```

多載可以自然區分「外部輸入」與「內部狀態流轉」，而不需要額外的 flag

<!-- endtab -->

<!-- tab Dynamic (Runtime) Polymorphism-->

用同一個「父類別介面」在呼叫方法，但真正的行為要留到執行時，交給「實際物件是誰」來決定，這樣系統才能在不改呼叫端的情況下自由擴充，可以讓「使用方式固定、行為可以替換」

![attack](raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/Polymorphism/dynamic_titan.png)


Method Overriding = 不同巨人繼承相同基底，但行為不同

```csharp
public abstract class Titan
{
    public virtual void Attack()
    {
        Console.WriteLine("巨人攻擊！");
    }
}

public class ColossalTitan : Titan
{
    public override void Attack()
    {
        Console.WriteLine("超大型巨人：巨大爆炸！");
    }
}

public class ArmoredTitan : Titan
{
    public override void Attack()
    {
        Console.WriteLine("鎧甲巨人：硬化衝撞！");
    }
}

Titan titan = new ColossalTitan();
titan.Attack(); // 輸出：超大型巨人：巨大爆炸！
```

![override](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/Polymorphism/dynamic_code_override.png)

創哥定義一個父類別 Titan，它代表一個「抽象概念」，只要是巨人，就一定能攻擊，Attack() 被定義成 virtual，意思是我先給你一個預設版本，但我知道你之後一定會想改，子類別繼承同一個基底類別
`ColossalTitan : Titan`、`ArmoredTitan : Titan`，這一步是在建立「是某一種 Titan」的關係，而不是獨立的類別，子類別用 override 重寫行為，方法名稱一樣、參數一樣，但內容完全不同，這是在明確告訴編譯器我不是新增一個方法，我是在替換父類別的那個行為

假設
```csharp
void DoAttack(Titan titan)
{
    titan.Attack();
}
```
完全不需要 if (titan is ColossalTitan)、也不需要 switch 判斷型別，新增一個 BeastTitan 時，這個方法一行都不用改，這就是多型的實際價值，行為的差異，被包在物件裡，而不是灑在流程控制裡
這個設計其實是在保護呼叫端，不被改動影響，多型是為了減少條件判斷、降低修改風險、提高系統擴充性，如果未來有 20 種巨人，真正穩定的不是巨人的數量，而是你「呼叫 Attack 的方式永遠不變」這件事


#### 商業情境 - 促購規則的「共通判斷骨架」與可覆寫的差異點（Promotion Rule Base）

促購規則不是只在比「折多少」，而是要先通過一組「公司不能漏的基本門檻」，子類別只能在這個門檻之上加條件，促購規則不是只在比「折多少」，而是要先通過一組「公司不能漏的基本門檻」，子類別只能在這個門檻之上加條件。


```csharp
/// <summary>
/// Defines the <see cref="PromotionRuleBase" />
/// </summary>
public abstract class PromotionRuleBase
{
    /// <summary>
    /// 折扣目標列舉
    /// </summary>
    /// <seealso cref="DiscountTargetTypeDefEnum"/>
    public virtual string DiscountTargetTypeDef { get; } = nameof(DiscountTargetTypeDefEnum.Product);

    /// <summary>
    /// Gets the DynamicPriority
    /// </summary>
    public virtual int DynamicPriority => this.Priority;

    /// <summary>
    /// Gets or sets a value indicating whether Enabled
    /// </summary>
    public bool Enabled { get; set; } = true;

    /// <summary>
    /// Gets or sets the Priority
    /// </summary>
    [JsonIgnore]
    public virtual int Priority { get; set; } = int.MaxValue;

    /// <summary>
    /// Gets the TypeFullName
    /// </summary>
    public virtual string TypeFullName { get => this.GetType().FullName; }

    /// <summary>
    /// Gets or sets the Since
    /// </summary>
    public DateTime Since { get; set; } = DateTime.MinValue;

    /// <summary>
    /// The IsApplicable
    /// </summary>
    /// <param name="salesChannel">The salesChannel<see cref="SalesChannelEnum"/></param>
    /// <param name="userContext">The userContext<see cref="UserContext"/></param>
    /// <param name="regionContext">The regionContext<see cref="RegionContext"/></param>
    /// <param name="locationContext">The locationContext<see cref="LocationContext"/></param>
    /// <param name="productItem">The productItem<see cref="ProductItem"/></param>
    /// <returns>The <see cref="bool"/></returns>
    public virtual bool IsApplicable(SalesChannelEnum salesChannel, UserContext userContext, RegionContext regionContext, LocationContext locationContext, ProductItem productItem)
    {
        if (!this.Enabled)
            return false;

        if (!IsMatchedDate(DateTime.Now))
            return false;

        return true;
    }

    /// <summary>
    /// The IsMatch
    /// </summary>
    /// <param name="cart">The cart<see cref="ShoppingCartContext"/></param>
    /// <returns>The <see cref="bool"/></returns>
    public virtual bool IsMatch(ShoppingCartContext cart)
    {
        if (!this.Enabled)
            return false;

        if (!IsMatchedDate(cart.Now))
            return false;

        return true;
    }

    /// <summary>
    /// The IsSkipped
    /// </summary>
    /// <param name="cart">The cart<see cref="ShoppingCartContext"/></param>
    /// <returns>The <see cref="bool"/></returns>
    public bool IsSkipped(ShoppingCartContext cart)
    {
        var appliedRules = cart.PromotionRecords.Select(x => x.PromotionRuleId).ToArray();
        return this.ExclusiveRuleIds?.Any(r => appliedRules.Contains(r)) == true;
    }

...
```

把「促購一定要有的共同狀態」集中在 Base Class像這些屬性

Enabled
Since / Until / MultipleDates
Priority / DynamicPriority
ExclusiveRuleIds / ExclusiveTags
TagsWhenMatched / TagsWhenMismatched
SourceType
Version
TypeFullName

這些是「所有促購都必須被同一套引擎理解」的資料結構，為了建立制度語言，假設有三種促購滿額折、指定商品折、會員等級折，它們全部都要不能在活動外套用、不能在停用狀態套用、能被促購引擎排序、排他、標記，這就是 PromotionRuleBase 在做的事


<!-- endtab -->

<!-- tab Interface Polymorphism-->

介面多型的核心是先約定「你一定做得到什麼」。這讓使用者只在意「能力是否存在」，而不是「具體是哪一個實作類別」


每個巨人實作了 ITitanShifter，所以可以用同一種型別（ITitanShifter）去操作不同的巨人，保證他們一定具備「變身」與「繼承」的能力

![interface](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/Polymorphism/interface_origin.png)

```csharp
public interface ITitanShifter
{
    void Transform();   // 必須能巨人化
    void Inherit();     // 必須能被繼承
}

public class AttackTitan : ITitanShifter
{
    public void Transform() => Console.WriteLine("進擊的巨人變身！");
    public void Inherit() => Console.WriteLine("進擊的巨人之力被繼承。");
}

public class ColossalTitan : ITitanShifter
{
    public void Transform() => Console.WriteLine("超大型巨人變身！");
    public void Inherit() => Console.WriteLine("超大型巨人之力被繼承。");
}
```

![interface](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/Polymorphism/interface_code.png)

另外，不管是 弗里茲王族 還是 雷伊斯王族，只要是王室血脈，就必須遵守「維護始祖之力的秘密」這個契約

```csharp
public interface IRoyalFamily
{
    void ProtectFoundingTitan();
}

public class FritzRoyal : IRoyalFamily
{
    public void ProtectFoundingTitan() => Console.WriteLine("弗里茲家族：守護始祖之力！");
}

public class ReissRoyal : IRoyalFamily
{
    public void ProtectFoundingTitan() => Console.WriteLine("雷伊斯家族：守護始祖之力！");
}
```

如果用抽象類別來做，類別就被「身分」綁死，之後想再加其他能力會卡死， 介面讓你只關心能力，不會過早做身分決定

<!-- endtab -->

<!-- tab Abstract Class Polymorphism-->

抽象類別的核心價值，是把「一定要一樣的規則」先鎖死，再逼子類別只專心處理「一定會不一樣的行為」，避免專案後期每個人亂玩
Abstract Class = 一部分是統一規則，一部分是各自實作

```csharp
public abstract class TitanShifter
{
    // 抽象方法 → 必須實作
    public abstract void SpecialMove();

    // 具體方法 → 所有人共用
    public void CommonWeakness()
    {
        Console.WriteLine("弱點：後頸被切開會死亡！");
    }
}

public class FemaleTitan : TitanShifter
{
    public override void SpecialMove()
    {
        Console.WriteLine("女巨人：吸引純潔巨人！");
    }
}

public class CartTitan : TitanShifter
{
    public override void SpecialMove()
    {
        Console.WriteLine("車力巨人：長時間持續戰鬥！");
    }
}
```

抽象類別是「半成品模板」。有些行為是統一的（所有巨人都有後頸弱點）。有些行為必須交由子類別去具體實作（各自的專屬技能）。這就像九大巨人「同源」於尤米爾，因此有共通的限制（壽命 13 年、後頸弱點），但各自的力量卻不同

<!-- endtab -->

<!-- tab 結語-->


![sum](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/Polymorphism/polymorph_sum_pics.png)


巨人之力實現了各式各樣的多型

- Static Polymorphism：同名方法，不同參數 → 如攻擊方式依條件不同而改變
- Dynamic Polymorphism：同樣呼叫 Attack()，不同巨人展現不同招式
- Interface Polymorphism：不同類別遵守相同契約 → 巨人繼承者、王室血脈
- Abstract Class Polymorphism：共通規則 + 專屬能力 → 九大巨人同源於尤米爾

![sum_table](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/Polymorphism/sum_table.png)

多型的核心精神，就是「在共通之中展現差異」。正如巨人世界中，不論如何分支，最後都能追溯到同一個源頭──尤米爾

<!-- endtab -->

{% endtabs %}