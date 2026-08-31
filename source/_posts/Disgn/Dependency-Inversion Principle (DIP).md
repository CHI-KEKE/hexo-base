---
title: Dependency-Inversion Principle
date: 2025-09-17 09:00:03
categories: 設計図鑑物語
top_img: https://i.imgur.com/LF7L1Ws.png
cover : https://i.imgur.com/LF7L1Ws.png
tags:
    - 
toc:
toc_number:
comments :
---


![Design](https://i.imgur.com/LF7L1Ws.png)



## 🪵 我受夠了依賴

「依賴」就是一種受到某個東西牽制、影響 的狀態。

一個大叔不抽菸就全身不舒服，這就是對香菸的依賴。手機一沒電就開始焦慮（好像是我），這也是一種依賴。

在系統裡，如果 A 模組必須靠 B 模組才能運作，那我們就說 A 依賴了 B。最典型的情況是：A 模組需要直接建立或呼叫 B 模組的實例，才能完成某個功能。

問題來了 —— 假設「DB 的連線方法」被修改了，那所有用到這個方法的「會員查詢」功能就得跟著改，甚至其他關聯模組也會被迫一起修改。 改動會一路延燒，越燒越大。

事實上，我們設計功能的順序應該是反過來的。不是因為「手邊剛好有鍋子和刀子」才說：「那就來煮菜吧！」而是因為「我們想做一道番茄炒蛋」，所以才需要鍋子來炒，刀子來切。

這正是 DIP（依賴反轉原則） 所強調的：

👉 高階模組（業務邏輯）不應該依賴低階模組（細節實作），兩者都應該依賴抽象。

<br>
<br>

## 🪵 IoC：控制反轉

就算我們把依賴反轉了，還是得有人「建立實例」吧？ 光是改成依賴抽象還不夠，因為在使用的那一刻，我們依然需要一個具體的物件。這時候，「控制反轉（Inversion of Control, IoC）」就派上用場了!

思路很簡單：

由一個控制反轉中心負責建立實例。高階模組只專心使用，不再自己去建立。低階模組則等待中心分發，拿到任務後專心把事情做好。
這樣一來，高階模組和低階模組之間就解除了緊密的依賴關係。所謂的依賴注入（Dependency Injection, DI），說白了就是用各種方式（建構子、屬性、方法參數）把需要的物件「丟進去」給類別使用。

<br>
<br>

## 🪵 未使用相依反轉原則

以下是「錯誤範例」
```CSHARP
public class Client
{
    Service _Service;
    public Client()
    {
        Service fooObj = new Service();
                fooObj.DoSomething();
    }
}
public class Service 
{
    public void DoSomething() { }
}
```

這裡 Client（高階模組）直接依賴 Service（低階模組）。如果哪天要換成另一個 Service1，因為緊密耦合，我們勢必得修改 Client 類別的程式碼，造成維護上的困難。

<br>
<br>

## 🪵 使用相依反轉原則

改用介面來解耦

```CSHARP
//各 Service 類別與 Client 並不會建立直接關係，而是透過介面來聯繫，即符合了相依反轉原則。

public interface IService{
    void dosomething();
}
public class  Service1 :IService
{
    public void dosomething()
    {
        Console.WriteLine(123);
    }
}
public class Service2 : IService
{
    public void dosomething()
    {
        Console.WriteLine(456);
    }
}

public class Client
{
    IService _service;
    public Client1(IService service)
    {
        _service = service;
    }
            
    public void DoSomething()
    {
        _service.dosomething();
    }
}
```
現在，Client 不再依賴具體的 Service1 或 Service2，而是依賴抽象的 IService。要切換不同的服務，只需要在建立 Client 的時候注入不同的實例即可，完全不需要改動 Client 本身的程式碼。


總結來說，就是讓高階模組專心描述「我要做什麼」，而不是被迫去關心「要怎麼做」。