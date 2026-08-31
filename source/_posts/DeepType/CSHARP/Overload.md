---
title: Overload
date: 2025-08-31 09:56:01
categories: 程思舞想
top_img: https://i.imgur.com/Zly1UDM.png
cover : https://i.imgur.com/Zly1UDM.png
tags:
    - C#
toc:
toc_number:
comments :
---

{% tabs Overload%}


<!-- tab Overload-->


![o](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Overload/landing.png)


對使用 API 的開發者來說。當他們呼叫一個方法時，心中想的只是「我要讓它運作」，卻不希望因為情境稍有不同，就被迫學習一整套全新的操作方式。這就是 Overload（方法重載） 存在的理由，它讓同一個「旋律」能被多種方式演奏，保持一致性，同時降低學習與使用的心智負擔


![cc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Overload/lower_mind_workload.png)


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Overload/only_know_what.png)


<!-- endtab -->


<!-- tab Console.WriteLine()-->


我們平常在寫 Console.WriteLine() 的時候，可以丟 string、int、bool、甚至物件，對吧？因為每種型別印出來的方式不一定一樣，比如物件就會呼叫 ToString()，而 string 不需要再轉。

```CSHARP
Console.WriteLine("Hello");      // string
Console.WriteLine(123);          // int
Console.WriteLine(3.14);         // double
Console.WriteLine(true);         // bool
Console.WriteLine(new DateTime(2025, 8, 31));  // DateTime
```

.NET 裡面 Console.WriteLine 方法重載了超過 18 種版本!


![c](https://github.com/CHI-KEKE/pics/blob/main/Code_Design/Overload/consolewriteline.png?raw=true)


<!-- endtab -->

<!-- tab String.Format() 或 $"{value}"-->


```CSHARP
string name = "John";
int age = 30;
string result = string.Format("Name: {0}, Age: {1}", name, age);
```

因為參數數量不固定，而且傳進來後要轉成 string，用泛型沒辦法這麼靈活地處理 params object[]。


<!-- endtab -->

<!-- tab HttpClient.GetAsync(...) 重載-->


你用 HttpClient 發送 GET 請求的時候，可能會有很多種情境：

只有 URL
URL + CancellationToken
URL + HttpCompletionOption + CancellationToken

.NET 內部就有多個 Overload 來支援不同的需求。因為這些重載代表的是不同參數的組合、執行流程可能也不一樣，而不是單純型別替換。

```CSHARP
HttpClient client = new HttpClient();
await client.GetAsync("https://api.example.com");                      // 簡單版本
await client.GetAsync("https://api.example.com", cancellationToken);  // 需要取消操作
await client.GetAsync("https://api.example.com", 
    HttpCompletionOption.ResponseHeadersRead, cancellationToken);     // 更細部的控制
```

![f](https://github.com/CHI-KEKE/pics/blob/main/Code_Design/Overload/httpclient_getasync.png?raw=true)

<!-- endtab -->


<!-- tab Overload 是讓「使用者思維」與「開發者實作」分離-->


➤ 當你設計一個 Library、API、SDK 給別人用時，Overload 幫你隱藏複雜度，讓開發者「感覺只做了一件事」，但其實內部可以有多種處理路徑。

```CSHARP
Save(User user);
Save(UserDto dto);
Save(User user, SaveOption option);
```

在使用者角度來看，他只需要呼叫 Save()，不用記得「哪一個是 SaveUserDto、SaveWithOption」，這讓他在心中建構的 mental model 是：「我就是要 Save 資料」而已。這是設計上的一致性（Consistency）與心智模型對齊（Cognitive Load Reduction）。

![d](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Overload/dis_mentalmodel_and_logic.png)


<!-- endtab -->

<!-- tab Overload 幫助 API 適應成長需求（可擴展性）-->


Overload 是一種「演進友善的設計」，當需求成長、功能增加時，你可以不破壞原有 API，讓原本的使用者繼續用，新的使用者使用新的版本。

```CSHARP
// 初版
Send(string message);

// 新增：指定收件人
Send(string message, string to);

// 再新增：加入標題與附件
Send(string message, string to, string subject, string[] attachments);
```

舊系統不需重構就能繼續使用，新需求也能支援，方法名稱保持一致


![o](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Overload/merge_old.png)


<!-- endtab -->

<!-- tab 用 Overload 做「語意導向」的 API 設計（Intent-Driven Design）-->


有時候不同參數，雖然都可以共用邏輯，但「參數本身代表了不同的使用情境」

```CSHARP
FindById(int id);             // 單筆查詢
FindById(int[] ids);          // 多筆查詢
FindById(string externalKey); // 外部系統查詢
```
雖然這三個方法本質上都叫 FindById，但參數型別讓使用者自然知道：「我現在在查哪種資料」


![g](https://github.com/CHI-KEKE/pics/blob/main/Code_Design/Overload/diff_type.png?raw=true)

<!-- endtab -->

<!-- tab 取代複雜參數物件或旗標（Flag）-->


```CSHARP
Save(user, true, false, null);
```

你應該要開始思考：這是 overload 的場景！


```CSHARP
Save(User user);
Save(User user, bool overwrite);
Save(User user, SaveOption option);
```

overload 可以清楚區分語意，避免程式碼變成猜謎遊戲，讓讀程式的人不用去猜每個參數是做什麼的。

<!-- endtab -->

<!-- tab 用在「動作不變，情境變化」的功能設計-->


動作：登入 Login
情境：帳號密碼、Google、Facebook、手機驗證碼

```CSHARP
Login(string username, string password);
Login(GoogleToken token);
Login(FacebookToken token);
Login(string phoneNumber, string smsCode);
```

這時候 overload 是比 interface、策略模式、泛型都來得更清楚直觀的選擇，因為你只是在提供「登入的不同方式」，不是不同的動作

![t](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Overload/login.png)

<!-- endtab -->


<!-- tab Anti - 語意模糊-->


```CSHARP
public class User
{
    public User(string name) { ... }
    public User(string name, bool isActive) { ... }
    public User(string name, bool isActive, bool isAdmin) { ... }
}

new User("Alice", true, false);
```

🧠 使用者根本不知道 true, false 是什麼意思，容易寫錯。語意模糊 + 容易誤用
✅ 解法是使用 Builder Pattern、Factory、或組裝 DTO。


![p](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Overload/confuse.png)


<!-- endtab -->

<!-- tab Anti - 過度 overload-->


```CSHARP
User(string name)
User(string name, string email)
User(string name, string email, string phone)
User(string name, string email, string phone, int age)
...
```

本質問題：「資料初始化應該抽象化，不該硬寫死在 overload」
解法是使用 Builder Pattern、Factory、或組裝 DTO

![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Overload/too_much_overload_need_dto_bulder.png)


<!-- endtab -->

<!-- tab Anti - 行為不一致，產生不可預期狀態-->


```CSHARP
public class Connection
{
    public Connection(string connString) { Connect(); }

    public Connection(string server, int port) { /* 不會自動連線 */ }
}
```

🧠 這讓使用者「以為兩個建構子只差參數，結果差邏輯」，這會導致誤解與難以除錯，屬於語意與行為不一致的典型設計錯誤

![kk](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Overload/behavior_not_as_expected.png)


<!-- endtab -->


<!-- tab 結語-->

![com](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Overload/composer.png)

Overload 的價值在於它能讓開發者「聽見」同一首旋律，不管樂器如何轉換，都能感覺一致、流暢，但若樂譜過度複雜、或每個版本都暗藏不同邏輯，那麼這首曲子將變得支離破碎、難以演奏，因此，好的 Overload 設計，就像一位懂得取捨的編曲者 —— 它保留必要的變化，卻維持主題的純粹與清晰


![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Overload/summa.png)

<!-- endtab -->


{% endtabs %}