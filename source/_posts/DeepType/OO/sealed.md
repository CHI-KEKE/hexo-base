---
title: sealed - 封印
date: 2025-09-13 11:00:03
categories: OO
top_img: https://i.imgur.com/LF7L1Ws.png
cover : https://i.imgur.com/LF7L1Ws.png
tags:
    - OO
toc:
toc_number:
comments :
---

{% tabs seal %}

{% btn 'https://www.reddit.com/r/dotnet/comments/f3hgqm/eli5_why_sealed/?tl=zh-hant',五歲小孩也能懂：為什麼要用 sealed？,far fa-hand-point-right %}

<!-- tab 法律-->

法律是一個經過設計的系統，不能隨意被改寫、繼承或覆蓋。你不能說：「我繼承《交通法》，然後 override 掉超速罰則。」就算你覺得這條法律不合理，也不能自己改動，只能在法律框架下使用它。  

法律是 **sealed** 的。因為它必須保證一致性、可預測性、公平性。若人人都能繼承、改寫，那麼整個社會將陷入混亂



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/sealed/1_law.png)



同樣地，一位資深醫生制定的手術流程極為嚴謹有效。新人不能說：「我繼承這個流程，但不消毒就直接開刀。」這就是為什麼醫療 SOP 是 sealed ，當一個系統至關重要、不能容許錯誤時，它就應該被封印


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/sealed/1_sop.png)



**sealed 的意義在於：**  
在自由與彈性之中，劃下一條「不可更動」的底線，以確保核心價值的穩定與安全。  

- 對系統來說：是一種 **封裝**  
- 對團隊來說：是一種 **約束**  
- 對使用者來說：是一種 **保證**


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/sealed/2_to_different_people.png)


<!-- endtab -->


<!-- tab 對開發者的具體好處-->


如果 sealed 只是「道德感的約束」，對工程師來說可能太抽象。我們更需要理解它對寫程式的人帶來的實際好處。


## 明確表達設計意圖，不怕被「誤用」或「誤繼承」

當你設計一個類別並加上 sealed，你在宣告：  
「這個類別不是設計來被繼承的，也不希望有人 override 其中的邏輯。」  

這是一種 **清楚的溝通**。  
- 對團隊來說，sealed class 代表「只能用，不能改」。  
- 對自己來說，它是一種 **自我保護**：避免別人擴充後造成 bug，最後鍋還要你背。


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/sealed/2_real_communication.png)


## 減少未來的技術債

若類別未封印，別人繼承後可能 override 方法、依賴特定 constructor。當你之後想調整內部邏輯，就會進退兩難：  

- 想優化，卻怕動到別人 override 的方法。  
- 想新增欄位，卻怕破壞別人寫死的繼承結構。  

加上 sealed，你等於保留了 **完整控制權**，不會讓維護責任流失。


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/sealed/3_safe_for_self.png)


## 可被 JIT 做效能最佳化（小 bonus）

效能雖不是 sealed 的主要目的，但它確實能帶來微幅提升：  
- 減少虛擬表查找（vtable lookup）  
- 更容易進行內嵌呼叫（inline method）  
- 降低型別檢查與分派成本  

對於小型、高頻率使用的類別（如 Utility 類別、資料處理器），加 sealed 可說是 **雙贏**


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/sealed/4_JIT.png)



<!-- endtab -->

<!-- tab summary-->

在程式世界裡，**sealed 就像這些被封印的基石**。  它提醒我們：  有些地方需要靈活，有些地方卻必須堅定不動。因為若一切都能被隨意改寫，那麼建築再高再華麗，也可能瞬間崩塌。  所以，每當你在類別上加上 sealed，就像在告訴團隊與未來的自己：「這裡是不可侵犯的基石。在這塊基石上，我們可以放心地往上建構更多自由而多樣的創造。」  

<!-- endtab -->


<!-- tab 工具 / Helper / Utility 類別-->


```csharp
public sealed class StringHelper
{
    public static string Slugify(string input) => ...;
}
```

工具類別通常沒有狀態，也不需被擴充。只需「拿來用」，而非「拿來改」


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/sealed/5_1_utility.png)


<!-- endtab -->

<!-- tab 邏輯封閉、不可更改的業務邏輯類別-->


```csharp
public sealed class TaxCalculator
{
    public decimal Calculate(decimal amount) => amount * 0.05m;
}
```

稅務邏輯屬於核心規則。若有人 override，偷改稅率，後果不堪設想

![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/sealed/5_2_core_business.png)


<!-- endtab -->


<!-- tab 安全性/授權處理類別-->


```csharp
public sealed class TokenGenerator
{
    public string GenerateToken(string userId) => ...;
}
```

安全相關程式若能被 override，極易產生漏洞。token 結構、演算法、有效期必須被固定


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/sealed/5_3_safety.png)



<!-- endtab -->

<!-- tab 小型不可變物件（Immutable Value Object）-->



```csharp
public sealed class Currency
{
    public string Code { get; }
    public Currency(string code) => Code = code;
}
```

像 Currency、CountryCode、Gender 這類 Value Object 本質上應固定不變。若允許繼承，會破壞值相等性比較的邏輯。


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/sealed/5_4_value_obj.png)


<!-- endtab -->

<!-- tab 專案 SDK / Library 對外的「黑盒類別」-->



```csharp
public sealed class Currency
{
    public string Code { get; }
    public Currency(string code) => Code = code;
}
```

像 Currency、CountryCode、Gender 這類 Value Object 本質上應固定不變。若允許繼承，會破壞值相等性比較的邏輯


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/sealed/5_5api_sdk.png)



```csharp
public sealed class ClientConfiguration
{
    public string ApiKey { get; set; }
    public string BaseUrl { get; set; }
}
```

對外提供的套件類別，不希望使用者隨意繼承。它們是「黑盒」：設計給人 使用，而非 擴充


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/sealed/6_bonudary_blurr.png)


技術上，如果你沒加 sealed，有心人還是可以這樣玩：

```csharp
public class MyLogger : Logger
{
    public override void Log(string message)  // ❌ 如果原本是 virtual，就能改
    {
        Console.WriteLine($"[HACKED] {message}");
    }
}
```

或者，用 反射 / IL 修改工具（如 dnSpy），甚至能做這種事：
```csharp
// 動態修改某個 method 行為
typeof(Logger)
    .GetMethod("Log")
    .Invoke(...); // 或動態產生 proxy 取代行為
```

這也是 sealed 存在的價值：在邊界上畫下不可侵犯的底線。

<!-- endtab -->

<!-- tab 專案 SDK / Library 對外的「黑盒類別」-->


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/sealed/7_seal_is_stone.png)

![e](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/sealed/8_final.png)


<!-- endtab -->


{% endtabs %}