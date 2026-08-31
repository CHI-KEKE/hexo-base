---
title: 讓他代我做：Delegate 從使用到設計，為目的而進化
date: 2024-11-10 07:33:00
categories: 風音子
top_img: https://i.imgur.com/zTSkblP.png
cover : https://i.imgur.com/zTSkblP.png
tags:
    - Delegate
toc:
toc_number:
comments :
---

在人生的旅途中，有些行動，不是為了此刻，而是為了將來某個尚未到來的時刻，悄悄種下伏筆。

就像在一場重要的典禮上，你輕聲叮囑朋友：「等我上台尻校長的光頭時，幫我拍一張照片。」
你清楚地設定了做什麼，但並沒有干涉朋友用哪台相機、站在哪個角度，只是信任對方，讓他自由捕捉那個瞬間。

又像叫了一份外送，你指定了：「請把三商巧福送到我家門口。」
你決定了目標，但至於外送員選擇騎機車、開車，走哪條路線抵達，則完全交給他靈活安排。

<br/>

<font color=#D3D3D3 style="font-size: 20px;">&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;你無需親手掌控每一個細節，只需種下清楚的意圖，並信任交付

</font>


這種「指定意圖，委託執行」的方式，在程式設計的世界裡的 Delegate

在[上一篇](https://allenlin000.github.io/2024/11/07/Delegate/)中，我們認識了 Delegate 的基本結構與意義；
而這一篇，我們將走得更深，去感受它在現實設計中如何展現靈活的力量，又如何結合泛型，成為更強大而多變的工具；
更會看見它化身為 Action、Func、Predicate，滲透進每一段程式邏輯之中。


## 從重複中提煉靈活：泛型的智慧設計

在開發過程中，我們遇過這種情況：

```CSHARP

public delegate int GetNumberReturnNumber(int arg, int arg2);
public delegate double GetNumberReturnNumber(double arg, double arg2);
public delegate string GetNumberReturnNumber(string arg, string arg2);

```

看似不同，其實本質上只是傳入與回傳的型別不同。
這樣的情境，自然而然讓人感受到他不夠靈活而且很麻煩

這時，我們可以抽象出一個更通用的委派：

```CSHARP

public delegate T GetNumberReturnNumber<T>(T arg, T arg2);

```

這個 GetNumberReturnNumber<T> 可以接受任意類型的參數，並返回相同類型的結果。
這就是泛型的力量：在設計時保留彈性，在使用時確定具體型別。

怎麼使用?
```CSHARP

GetNumberReturnNumber<int> addInt = Add;
Console.WriteLine("整數加法: " + addInt(5, 10)); // 15

GetNumberReturnNumber<double> addDouble = Add;
Console.WriteLine("浮點數加法: " + addDouble(5.5, 10.5)); // 16.0

GetNumberReturnNumber<string> concatString = Concat;
Console.WriteLine("字串連接: " + concatString("Hello, ", "World!")); // Hello, World!


static int Add(int a, int b)
{
	return a + b;
}

static double Add(double a, double b)
{
	return a + b;
}

static string Concat(string a, string b)
{
	return a + b;
}

```

泛型（Generic）是 .NET 2.0 引入的重要功能。
它讓我們在設計類別或方法時，暫時不指定具體型別，而是等到使用（宣告或具現化）時才決定。

而在執行階段，泛型會根據型別特性有兩種不同的行為：

值型別（value type）：JIT 編譯器會為每一個具體型別生成一份專屬的機器碼，就像直接拿「東西本體」搬運，記憶體大小/CPU操作不同，必須分開

參考型別（reference type）：JIT 編譯器只生成一次通用的機器碼，後續不同型別共用同一份，就像只搬「箱子標籤」（指標），搬來搬去的永遠都是 4 bytes 或 8 bytes 的指標，所以搬運方法可以共用，只搬指標，大小一致，操作方式一致!


## Func、Action、Predicate

C# 早就觀察到 —— 很多委派其實都在做類似的事。
所以，乾脆幫你直接設計好「常用委派模板」，讓你可以不用每次都自己辛苦寫。
這些模板，就是：Action、Func、Predicate。

聽起來還是有點抽象？
沒關係，讓我們用毛吉的故事來解釋吧！


### 🥬 Action：想吃高麗菜的毛吉

毛吉是一隻超愛吃高麗菜的小生物
但主人為了讓他飲食均衡，不會天天給他高麗菜
於是毛吉下定決心：自己上街買！


一開始，毛吉學會用**委派（Delegate）**來規劃他的「買高麗菜計畫」

```CSHARP

public delegate void BuyCabbage();

```

接著，他實作了去商店買菜的方法：
```CSHARP

public static void BuyCabbageFromEZStore()
{
    "拿到一顆可口的高麗菜".Dump();
}

```

```CSHARP

BuyCabbage buyCabbageFromEZStore = BuyCabbageFromEZStore;

```

隔天一早，自己去買高麗菜
```CSHARP

buyCabbageFromEZStore();

```

毛吉成功買到了高麗菜，開心到在街上原地打滾！🎉

---

過沒幾天，毛吉覺得不對勁了。

> 「奇怪，想買個菜還要自己定義一個 delegate，太麻煩了吧！」🐶💬

於是，他發現了微軟幫忙準備好的東西：Action！ Action 是 .NET 內建的泛型委派，用來封裝沒有回傳值（void）的行為。

於是毛吉重新整理買菜計畫：

```CSHARP

Action buyCabbageFromEZStore = BuyCabbageFromEZStore;
buyCabbageFromEZStore();

```

✅ 少了自己寫 delegate，毛吉覺得世界簡直美好！

---

某天他嘗到地瓜葉，覺得超級好吃！這時他又想：難道每個要買的菜都要寫一個新方法？太笨了！

所以，他改成這樣：

```CSHARP

public static void BuyAnything(string item)
{
    $"拿到一個 {item}".Dump();
}

```

```CSHARP

Action<string> buyAnything = BuyAnything;
buyAnything("地瓜葉");

```

🌟 加一個參數就可以買任何想吃的菜了！

分享一個洨知識，傳入參數最多可以放16個喔
![Image](https://i.imgur.com/ENa3Xi6.png)


### 🍔 Func：懶惰期來臨的毛吉

隨著年紀漸長，毛吉變得越來越懶。
有一天，他躺在床上想：

> 「唉，出門買菜好累喔，有沒有直接送到嘴邊的方法？」🐶💭

這時，他學會了Func！Func 是一種可以回傳結果的委派。

來看看毛吉的新技能 —— 叫外送！
```CSHARP

public static string BuyAnythingFromUber(string item)
{
    return $"甚麼都點得到! {item} 來囉!!";
}

```

```CSHARP

Func<string, string> buyAnythingFromUber = BuyAnythingFromUber;
buyAnythingFromUber("地瓜葉").Dump();

//// 毛吉太得意了，順手點了一樣奇怪的東西：
buyAnythingFromUber("美女照").Dump();

```

![Image](https://i.imgur.com/zRzhhqJ.png)


### 📦 Predicate：查庫存的毛吉

後來，UberEats 推出新功能：下單前可以查庫存！

於是毛吉又學會了新的技能：Predicate！

Predicate<T> 是專門用來**判斷某個條件是否成立，並回傳布林值（true/false）**的委派。

因此

```CSHARP

public static bool CheckItemStock(string item)
{
    bool isAvailible = false;
    //// 確認庫存邏輯...

    return isAvailible;
}

```

```CSHARP

Predicate<string> checkStock = CheckItemStock;
checkStock("高麗菜").Dump();

```

這樣，毛吉就可以在下單前先確認：
「這家店，今天到底還有沒有我最愛的高麗菜呢？」

我們來總結一下

![Image](https://i.imgur.com/jDwzRxK.png)


## ☘️ 結語：從使用到設計，為目的而進化

起初，我們只是單純地想完成某件事——
一個動作、一個回應、一個結果。

Delegate，給了我們這份能力，讓我們可以將行動託付給他人，或是延遲到未來的某個時刻再執行。

但隨著需求越來越多樣、邏輯越來越複雜，
我們開始發現：每一次重複撰寫，都藏著一個可以被抽象化的模式。

於是，泛型委派誕生了，讓不同型別的操作擁有了同一種語言；
Action、Func、Predicate 也應運而生，讓那些日常中反覆出現的需求，化作輕盈而標準的模板。

這不只是語法上的優雅，更是一種思考的進化：

從「如何執行」的使用者，走向「如何設計」的思考者。


<font color=#D3D3D3 style="font-size:18px;">&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;
我們學會了，不只是為了解決當下的問題，而是為了讓未來的自己，可以走得更快、更自由。
</font>
