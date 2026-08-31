---
title: OCP
date: 2025-09-12 08:00:03
categories: OO
top_img: https://i.imgur.com/LF7L1Ws.png
cover : https://i.imgur.com/LF7L1Ws.png
tags:
    - OO
toc:
toc_number:
comments :
---

{% tabs OCP%}


<!-- tab OCP 的價值-->


我們先來想一個問題：**軟體服務與實體產品最大的差異是什麼？它的價值來自哪裡？**

一台車、一棟房子、一支手機，當它製造完成、出廠後，要再改動幾乎不可能。  
你或許可以維修或加裝零件，但「核心設計」已經固定。因為製造實體產品的成本主要在 **生產鏈**：模具、材料、工廠產線、物流。只要東西做出來，要大幅修改就得「重新開模 → 重做 → 巨大成本」。因此，實體產品通常必須在設計階段就「盡量定案」，避免後期修改。

而軟體完全不同。  
軟體的「製造成本」幾乎為零，複製一份就能立刻運行。它的價值不在於「做出來」，而在於**是否持續符合需求**。當需求變動，只要重新開發、部署，就能立刻改變行為。軟體的主要成本來自 **設計與維護**，而不是製造。  
這也是為什麼軟體可以一直改，甚至「應該一直改」──因為它的價值就在於能持續貼合不斷變化的需求



![32](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/COP/1__software_essence.png)



軟體服務的本質是 **資訊結構**，而非物理材料。 
你可以隨時推翻邏輯、重組流程，而不用重開工廠或換掉原料。唯一的代價是 **人類理解與協作的成本**（需求確認、程式設計、測試、部署），而不是物理世界的限制


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/COP/1_sofeware_need.png)


<!-- endtab -->

<!-- tab 甚麼? 又要改程式?-->


理解了軟體的本質後，我們可以接受「改程式是必然」，但依舊會覺得頭痛。
為什麼呢？因為改程式有一些本質性的風險。

## 牽一髮動全身的不確定性

程式就像一個複雜的機械系統，零件彼此咬合。當你修改其中一小段程式碼時，它可能影響到：

- **隱藏的依賴**：別的模組偷偷依賴這裡的邏輯  
- **既有的假設**：原本設計時的預期被破壞  
- **測試覆蓋不足**：沒有人驗證的部分可能默默壞掉  

結果就是，你改了一行，看似 OK，卻在另一個模組引發災難。


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/COP/2_afraid_of_modify.png)


## 知識負債與理解成本

要改程式，前提是「先理解它在做什麼」。然而現實中常會遇到：  

- 原本的開發者已經離職，留下 **知識斷層**（需求來自人，人也會流動）  
- 註解不足，或程式邏輯過於複雜，必須花很長時間「reverse engineering」  
- 架構不清楚，程式碼像 **意大利麵條** 一樣難以追蹤資料流  

這些都會導致開發者花大量時間在「摸索」，而不是「解決問題」。  
程式不只是指令，它承載了當時開發者對問題的理解。需求會變，理解也會變。當你回頭改程式，其實是在「重新進入當時的思維模型」。如果命名和設計沒有清楚表達思維脈絡，修改者就像在讀「別人的腦內速記」。


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/COP/2_no_knowledge.png)


## 改壞風險大

程式一旦改動，就可能導致：

- **既有功能失效**（回歸 bug）  
- **效能下降**（新的邏輯更耗資源）  
- **安全漏洞**（意料之外的攻擊點）  

這就是為什麼業界有句老話：**「Don’t touch working code」**



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/COP/3_need__modify_but_afrid.png)


<!-- endtab -->

<!-- tab 回到 OCP 的精神-->


那麼，該怎麼降低改程式的痛苦呢？  
答案就是 **OCP（Open-Closed Principle，開放－封閉原則）**：

- **對擴展開放**：系統要能透過新增程式碼來增加功能  
- **對修改封閉**：不要去動到既有程式碼，因為它已經被驗證過能正常工作  

換句話說，當需求改變時，我們應該是「加東西」而不是「改東西」。  
OCP 不是死規則，而是一種心法。透過抽象化、模組化和設計模式，我們能讓系統在面對變化時更有彈性


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/COP/4_ocp.png)


希望基礎建設、主流程不會隨著業務需求改變，並留好需求變動的插槽在那


<!-- endtab -->

<!-- tab 怎麼做到？-->


要實現 OCP，常見的手段有：

- **抽象化**：在主要邏輯與附加邏輯之間，加上一層「抽象介面」來解耦合  
- **設計模式**：工廠模式（Factory）、策略模式（Strategy），都是 OCP 的實用技巧  
- **Plug-in 架構**：就像 Visual Studio、Chrome 透過「擴充套件」加功能  
- **Dependency Injection**：讓程式在運行時注入不同實作，而不是寫死在程式碼中  

總之，我們要 **建立抽象，封閉(close)修改，開放擴充**


<!-- endtab -->

<!-- tab 組裝電腦-->


主機板上會預留插槽，你可以自由更換顯示卡、記憶體。主機板（抽象）不用改，每次只要插上新的元件（實作）就能擴充功能。


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/COP/5_like_motherboard.png)


<!-- endtab -->


<!-- tab 基因演化-->


生物不是一成不變的，而是會隨環境調整自己。好的程式架構也應該具備「適應變化」的能力。

<!-- endtab -->



<!-- tab 介面-->


建立 `IService`，主流程只依賴介面，行為用不同實作 class 表達。  
常見於：策略模式、服務導向架構、DI 容器


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/COP/6_interface.png)


<!-- endtab -->


<!-- tab 策略模式（Strategy Pattern）-->

將變動行為抽出來注入

```CSHARP
public class Animal
{
    private readonly IVoiceStrategy _voice;
    public Animal(IVoiceStrategy voice) => _voice = voice;
    public void Speak() => _voice.MakeSound();
}
```

新增叫聲時不用改 Animal → 符合 OCP。


<!-- endtab -->


<!-- tab 委派 / 函式注入（Func、Delegate）-->


```CSHARP
public class Executor
{
    private readonly Action _action;
    public Executor(Action action) => _action = action;
    public void Run() => _action();
}
```

新增邏輯只要傳入 lambda，不用修改 class


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/COP/6_2_lambda.png)



<!-- endtab -->


<!-- tab 事件機制 / 觀察者模式（Event / Observer）-->


```CSHARP
public class OrderService
{
    public event Action<Order> OrderCompleted;
    public void Complete(Order o) => OrderCompleted?.Invoke(o);
}
```

新增行為只要 +=，不用改 OrderService


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/COP/6_3_event.png )


<!-- endtab -->

<!-- tab 組態驅動（Config-Driven）-->


```JSON
{
  "DiscountRules": [
    { "Type": "VIP", "Rate": 0.2 },
    { "Type": "Student", "Rate": 0.1 }
  ]
}
```

改設定，不改程式 → 符合 OCP


<!-- endtab -->


<!-- tab 資料驅動（Data-Driven）-->

| Step | ActionType |
| ---- | ---------- |
| 1    | SendEmail  |
| 2    | SaveDB     |

程式讀表即可決定流程，加流程改資料，不改邏輯


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/COP/6_4_datadriven.png)


<!-- endtab -->


<!-- tab 插件架構（Plugin / 模組化 / MEF）-->

功能獨立封裝成 DLL，透過動態載入加功能。新增功能丟 DLL，不動主程式 → OCP。

<!-- endtab -->


<!-- tab 管線（Pipeline） / 中介軟體（Middleware）-->

```CSHARP
app.Use(async (ctx, next) =>
{
    Console.WriteLine("Before");
    await next();
    Console.WriteLine("After");
});
```
新增功能只要加 middleware，不用改現有流程。ASP.NET Core 就是典型實踐


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/COP/6_5_middleware.png)


<!-- endtab -->


<!-- tab Reference-->


{% btn 'https://teddy-chen-tw.blogspot.com/2014/04/solid.html',SOLID：五則皆變,far fa-hand-point-right %}

<br>

{% btn 'https://teddy-chen-tw.blogspot.com/2011/12/2.html',亂談軟體設計（2）：Open-Closed Principle,far fa-hand-point-right %}

<br>

{% btn 'https://www.jyt0532.com/2020/03/19/ocp/',深入淺出開放封閉原則 Open-Closed Principle,far fa-hand-point-right %}

<br>

{% btn 'https://igouist.github.io/post/2020/10/oo-11-open-closed-principle/',菜雞與物件導向 (11): 開放封閉原則,far fa-hand-point-right %}


<!-- endtab -->


{% endtabs %}