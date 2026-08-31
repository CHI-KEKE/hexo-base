---
title: 從克德族說起：繼承為何讓人生卡關？
date: 2024-07-16 14:08:05
categories: 設計図鑑物語
top_img: https://i.imgur.com/LF7L1Ws.png
cover : https://i.imgur.com/LF7L1Ws.png
tags:
    - Design
toc:
toc_number:
comments :
---

![design](https://i.imgur.com/LF7L1Ws.png)

好久好久以前，有兩個著名的家族——

克德族，是一個信奉血統主義的家族。他們深信純正血脈的力量，嚴格遵循世代傳承的家族傳統，認為唯有優良基因的延續，才能孕育出最優秀的後代。起初，這種方式的確造就了不少傑出人才。克德家的子女大多具備出色的音感，未來幾乎都走上了音樂的道路。

但隨著時間流逝，問題逐漸浮現。某一天，家族中一位孩子心懷成為機械技師的夢想，卻發現家族的傳統成了束縛。他無法脫離音樂的陰影，也難以獲得從事機械所需的技能，陷入深深的痛苦與迷惘。

![Image](https://i.imgur.com/ZHYDbPy.jpeg)

海的另一端，是萊雅族。這是一個秉持開放思想的家族。他們相信卓越不僅來自血統，更來自個人的選擇與學習。他們鼓勵每位成員自由探索，組合各種不同的技能與知識。

這樣的理念帶來了極大的彈性。萊雅族的子女可以依興趣學習，不論是音樂、機械或其他領域皆不受限制。他們甚至可以同時成為音樂家與技師。

然而，自由也帶來困惑。因選擇過多，部分成員在成長過程中感到迷惘，沒有明確的方向。不僅如此，因為每個人都走著截然不同的路，外人甚至難以辨別他們是否屬於同一族群。

![Image](https://i.imgur.com/bMVHuuv.jpeg)

克德族的困境，就像物件導向中的繼承（Inheritance）。子類別（Subclass）往往難以擺脫父類別（Parent Class）賦予的行為與限制。明明想走自己的路，卻被迫實作那些與自身不相干、甚至不合時宜的方法，最後只能無奈地回傳 null、false，或乾脆拋出 NotImplementedException —— 彷彿也在拋下自己的夢想與初衷。

而萊雅族的自由，則象徵著組合（Composition）的靈活與挑戰。你可以自由注入不同的依賴、拼湊出多元的能力，但若缺乏方向與設計哲學，最終可能落得類別之間的耦合雜亂無章，像是一場技能拼貼的狂歡，卻少了意義與核心。

# 🌸 Inheritance 之美

Inheritance 為我們帶來的好處就是可以收攏相關的 Class，並且減少重複的 Code，實現多型 (polymorphism)
(clearly related reusable pieces of code that fit under a single common concept)

例子有 偵聽者（listner）、工廠（factory）模式

# ☠️ Inheritance 的問題與陷阱

### Diamond Problem（菱形繼承問題）

當專案規模逐漸擴大、需求越來越多樣時，如果一味依賴繼承來解決功能擴展，最終可能會陷入「多重繼承」的泥淖，導致系統變得複雜且難以預測——這就是所謂的 Inheritance Hell。

其中一個經典的例子，就是 Diamond Problem（菱形繼承問題）：
```csharp

       A
      / \
     /   \
    B     C
     \   /
      \ /
       D

```

假設：

- A 是基底類別，定義了一個方法 DoSomething()

- B 和 C 都繼承自 A，並各自改寫了 DoSomething()

- D 同時繼承 B 和 C

問題來了：

當我們呼叫 D.DoSomething()，應該執行的是 B 的版本，還是 C 的？

當 D 被建立時，A 的建構函數會被呼叫一次，還是兩次？

這些行為若沒有明確定義，很容易導致錯誤、歧義與維護上的困難。


讓我們用遊戲角色的例子來具體感受這個問題：

假設你正在開發一款 RPG 戰鬥系統：

- Character 是所有角色的基底類別，定義了通用屬性，例如 Name 和 Health

- Warrior 是近戰型角色，擁有 Attack() 方法，使用劍進行攻擊

- Mage 是魔法型角色，擁有 Attack() 方法，使用魔法進行攻擊

- Battlemage 是一種同時擁有戰士與法師能力的角色，看起來理應同時繼承 Warrior 與 Mage

感覺很像楓之谷之類的設計欸

![Image](https://i.imgur.com/1nrGlNV.jpeg)

```csharp

public class Battlemage : Warrior, Mage { }

```

但這樣會產生幾個問題：

🚫 行為歧義
Character 定義了 Attack() 方法，Warrior 和 Mage 都改寫了它。那麼當我們呼叫 Battlemage.Attack() 時，會執行哪一個版本？這會造成邏輯不明、行為不確定。

🚫 屬性重複
Health 屬性定義在 Character 中，但因為 Warrior 和 Mage 都繼承了它，Battlemage 會同時擁有兩份 Health。這會導致狀態混亂、錯誤難以追蹤。

🚫 建構順序問題
當建立 Battlemage 的實例時，系統會對 Character 的建構函數進行呼叫兩次嗎？還是只呼叫一次？這點若沒有清楚定義，會嚴重影響穩定性與行為預測。

這些問題的根源在於——

雖然 Battlemage 看似「同時是戰士也是法師」，但這並不代表它應該「同時繼承兩者」。
它不是典型的 is-a 關係，而更像是「擁有（has-a）」戰士能力與法師能力。

這樣的情境，更適合使用 Composition（組合） 來建構角色的能力，而非多重繼承。



### 多重責任導致類別設計混亂

在實務開發中，當一個類別需要同時具備多個功能時，開發者常會直覺地想用「多重繼承」來整合各種基礎功能。舉例來說：

假設我們在設計一個文件系統：

Storable：可儲存的功能，提供 Save() 和 Load() 方法

Compressible：可壓縮的功能，提供 Compress() 和 Decompress() 方法

CompressedFile：這是一種既需要儲存也需要壓縮功能的類別

於是開發者便想寫個 CompressedFile : Storable, Compressible。看似合理，實際上卻暗藏陷阱：

❗ 問題一：功能耦合過緊
假如 CompressedFile 之後又需要新增「加密」功能（例如 Encryptable），是不是又要再找一個新的 Base Class 來繼承？不知不覺，這個類別會越繼承越多，導致功能堆疊、耦合過深。

❗ 問題二：方法衝突與歧義
若 Storable 和 Compressible 都定義了 Save() 方法，請問 CompressedFile.Save() 應該執行哪一個？
這種情況會讓程式邏輯變得模糊，埋下難以預測的錯誤。

❗ 問題三：測試與維護困難
多重繼承不僅會造成類別責任混亂，還會讓測試變得困難。因為 SubClass 不只耦合於父類別的行為，還得處理父類別彼此間的互動關係，維護成本大幅提升。

### 無法將 parent class 轉為 subclass 使用

讓我們來看一個公司人事系統的例子：

```csharp

public class Employee
{
    public string Name { get; set; }
    public int Salary { get; set; }
}

public class Manager : Employee
{
    public List<Employee> Subordinates { get; set; }
}

// 使用
Employee allen = new Employee { Name = "Allen", Salary = 10000000 };


```

某天 Allen 升職了，你可能直覺地想這樣寫：

```csharp

// 無法直接將 allen 晉升為 Manager
Manager managerAllen = (Manager)allen; // ❌ 這會拋出異常

```

為什麼不能直接轉型？因為繼承是一種靜態關係，代表物件在建立的那一刻就已經決定了它的類型。而升職這種行為，是動態狀態的轉換，這並不符合繼承的邏輯。

換句話說，Allen 一開始就不是 Manager，你不能只是說「他現在是經理」就強制讓他變成 Manager；你得給他一個新的身份、角色與能力。

那該怎麼做？
正確做法是：建立一個新的 Manager 實例，然後把 Allen 的資料轉移過去。我們可以這樣寫，把升職的邏輯封裝在 PromoteToManager 方法中
```CSHARP

public class Employee
{
    public string Name { get; set; }
    public int Salary { get; set; }

    // 顯式轉換為 Manager
    public Manager PromoteToManager()
    {
        return new Manager
        {
            Name = this.Name,
            Salary = this.Salary,
            Subordinates = new List<Employee>()
        };
    }
}

public class Manager : Employee
{
    public List<Employee> Subordinates { get; set; }
}

// 使用
Employee allen = new Employee { Name = "Allen", Salary = 10000000 };
Manager managerAllen = allen.PromoteToManager();

```

但 Manager 可以安全地轉型為 Employee。

因為 Manager 是 Employee 的子類別（SubClass），它自然也符合「是一個員工（is-a Employee）」的邏輯。這就是典型的 多型（polymorphism） 行為。但一旦你這樣做，就只能使用 Employee 暴露出來的功能，無法再直接存取 Manager 專屬的屬性（例如 Subordinates）

### 子類別被迫實作不適用的方法

假設我們有一個基底類別 HomeAppliance，代表「家用電器」，並定義了一些通用功能，例如：

- 開關機（PowerOn / PowerOff）

- 設定計時器（SetTimer）

- 連接 WiFi（ConnectToWifi）

```csharp

public abstract class HomeAppliance
{
    public string Brand { get; set; }
    public bool IsPoweredOn { get; private set; }

    public virtual void PowerOn()
    {
        IsPoweredOn = true;
        Console.WriteLine($"{GetType().Name} is powered on.");
    }

    public virtual void PowerOff()
    {
        IsPoweredOn = false;
        Console.WriteLine($"{GetType().Name} is powered off.");
    }

    public abstract void SetTimer(int minutes);

    public virtual void ConnectToWifi()
    {
        Console.WriteLine($"{GetType().Name} is connected to WiFi.");
    }
}

```

這樣看起來好像很合理，但問題出在並不是所有家電都需要這些功能。


Microwave（微波爐）：需要設定計時器功能，所以合理地實作了 SetTimer。

SmartTV（智慧電視）：支援計時與 WiFi 功能，照單全收沒問題。

Refrigerator（冰箱）：問題來了——不是每台冰箱都需要 WiFi 或計時器功能。

結果 Refrigerator 必須這樣處理：
```csharp

public class Refrigerator : HomeAppliance
{
    public override void SetTimer(int minutes)
    {
        throw new NotSupportedException("Refrigerator does not support timer functionality.");
    }

    public override void ConnectToWifi()
    {
        throw new NotSupportedException("This refrigerator model does not support WiFi connection.");
    }
}

public class Microwave : HomeAppliance
{
    public override void SetTimer(int minutes)
    {
        Console.WriteLine($"Microwave timer set for {minutes} minutes.");
    }
}

public class SmartTV : HomeAppliance
{
    public override void SetTimer(int minutes)
    {
        Console.WriteLine($"TV will shut down after {minutes} minutes.");
    }
}

```

這樣的設計看起來像是冰箱被逼著學會用不到的技能，只好拋出 NotSupportedException 來硬擋。

這個例子凸顯了繼承的一個常見問題：子類別被迫實作它根本不需要、甚至不該有的功能，它違反了 里氏替換原則（Liskov Substitution Principle）。
理論上，當你用一個 HomeAppliance 實體時，不管具體是 Microwave、SmartTV 還是 Refrigerator，行為都應該是預期之內的。但如果你呼叫 SetTimer()，冰箱卻直接拋錯，代表子類的行為已經偏離父類的設計預期。

## ☘️ 結語：繼承，是承接還是束縛？

在物件導向的世界裡，繼承（Inheritance）本意是為了讓後代共享智慧、延續能力，就如同克德族對純血傳統的執著。然而，當這份傳承變成了桎梏，當子類別被迫背負不屬於自己的責任、實作不適合的功能，那麼這樣的繼承，便已悄然違背了初衷。

正如我們在生活中所見，不是所有孩子都該成為音樂家，不是每位員工升職後仍是原來的樣子。設計也是如此——我們應當在結構與靈魂之間尋找平衡，讓系統能夠承接過往的經驗，同時擁有選擇的自由。

繼承，不應該是一條只能直走的道路；它應該是一座橋，讓你能看見過去、理解現在，卻仍能自由地走向未來。