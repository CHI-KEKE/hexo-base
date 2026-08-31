---
title: Interface Segregation Principle
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

{% tabs Interface Segregation Principle%}


<!-- tab 違反介面隔離原則的例子-->

客戶（Client）不應被迫依賴對其而言無用的方法或功能。換句話說：介面設計不應包山包海，而應拆分成精簡、專注的責任


```csharp
interface IPrintFunction
{
    public void print();
 
    public void scan();
 
    public void fax();
 
    public void copy();
 
    public void copyThenSendMail();
}


public class EPSON: IPrintFunction
{
    public void print() 
    {
        Console.WriteLine("我能列印!");
    }
 
    public void scan() 
    {
        Console.WriteLine("我能掃描!");
    }
 
    public void fax() 
    {
        Console.WriteLine("我能傳真!");
    }
 
    public void copy()
    {
        Console.WriteLine("我能影印!");
    }
 
    public void copyThenSendMail()
    {
        Console.WriteLine("我影印完還可以發mail!");
    }
}
```


可看出除了 print 與 scan 這兩個函式有內容外，其他的功能都是空的，只是為了使實作介面能正常不跳錯而已，這就違反了介面隔離原則，用戶端不應該強制相依於特別、沒用到的方法。

```csharp
public class HP : IPrintFunction
{
    public void print()
    {
        Console.WriteLine("我能列印!");
    }
 
    public void scan()
    {
        Console.WriteLine("我能掃描");
    }
 
    public void fax(){}
 
    public void copy(){}
 
    public void copyThenSendMail(){}
}
```

![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/ISP/1_not_used_func.png)

![z](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/ISP/1_god_interface.png)

![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/ISP/1__empty_func.png)


<!-- endtab -->


<!-- tab 符合介面隔離原則的例子-->


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/ISP/2_split_interface.png)


```CSHARP
interface IPrintFunction
{
    public void print();
 
    public void scan();
}
 
interface ISuperPrintFunction
{
    public void fax();
    public void copy();
    public void copyThenSendMail();
}

public class HP : IPrintFunction
{
    public void print()
    {
        Console.WriteLine("我能列印!");
    }
 
    public void scan()
    {
        Console.WriteLine("我能掃描");
    }
}
public class EPSON: IPrintFunction, ISuperPrintFunction
{
    public void print() 
    {
        Console.WriteLine("我能列印!");
    }
 
    public void scan() 
    {
        Console.WriteLine("我能掃描!");
    }
 
    public void fax() 
    {
        Console.WriteLine("我能傳真!");
    }
 
    public void copy()
    {
        Console.WriteLine("我能影印!");
    }
 
    public void copyThenSendMail()
    {
        Console.WriteLine("我影印完還可以發mail!");
    }
}
```


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/ISP/2_epson.png)


HP 只依賴自己需要的功能。EPSON 可以擴充進階功能。系統設計更彈性、避免不必要依賴。把龐大介面切割成多個小介面 → 各個類別只需挑選自己真正需要的介面來實作。結果就是，減少空方法、減少不必要依賴、提升系統的可維護性與擴充性


<!-- endtab -->


<!-- tab 與 SRP 的關係-->

SRP（單一職責原則）強調 「一個類別只應該有一個引起變化的理由」。而 ISP（介面隔離原則）則是將這個觀念延伸到 介面層級

- SRP 關注的是 類別本身的責任是否單一。
- ISP 關注的是 介面是否要求類別去實作過多責任。

👉 換句話說，ISP 可以被視為 SRP 在抽象層面的應用。當一個介面過於龐大（像「大一統介面」），會迫使類別去承擔多種責任，違反了 SRP。將介面切割成多個精簡的契約，就能確保類別專注在自己真正的職責範圍內。


若介面設計違反 ISP（塞了太多責任），就會逼得實作類別同時承擔不相關的行為 → 最後這個類別本身也違反 SRP。這是一個「從上游污染到下游」的情境


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/ISP/3_srp.png)


例如
```CSHARP
public interface IMemberService
{
    void Register(string username, string password);
    void Login(string username, string password);

    // 不屬於會員核心職責，卻硬塞進來
    void SendPromotionalEmail(string email, string message);
    void ExportAllMembersToCsv(string filePath);
}
```

實作類別
```CSHARP
public class MemberService : IMemberService
{
    public void Register(string username, string password)
    {
        Console.WriteLine("註冊會員");
    }

    public void Login(string username, string password)
    {
        Console.WriteLine("會員登入");
    }

    // 其實應該由「行銷系統」負責
    public void SendPromotionalEmail(string email, string message)
    {
        Console.WriteLine($"寄送促銷信給 {email}");
    }

    // 其實應該由「報表系統」負責
    public void ExportAllMembersToCsv(string filePath)
    {
        Console.WriteLine($"匯出會員資料到 {filePath}");
    }
}
```


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/ISP/3_memberservice.png)



連帶被注入的類別也被汙染
```CSHARP
public class AccountController
{
    private readonly IMemberService _memberService;

    // 透過 DI 注入
    public AccountController(IMemberService memberService)
    {
        _memberService = memberService;
    }

    public void RegisterUser(string username, string password)
    {
        _memberService.Register(username, password);
    }

    public void SendCampaignMail(string email, string message)
    {
        // ❌ 嘿嘿 memberService 還可以寄行銷mail 來實作吧
        _memberService.SendPromotionalEmail(email, message);
    }
}
```
因為介面污染，注入 IMemberService 的地方（例如 AccountController）也被暴露到「寄送行銷信」這種不屬於會員服務的功能 → 責任邊界繼續擴散，後續的維護者看到的就是 AccountController 的 memberService 做了不屬於他該做的事!


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/ISP/3_fix_pollution.png)


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/ISP/3_up_to_down_pollution.png)

<!-- endtab -->

<!-- tab 與 OCP 的關係-->

OCP（開放封閉原則）強調「對擴充開放，對修改封閉」。ISP 與 OCP 之間的連結在於，如果介面過於龐大，當需求變動時，許多類別都會被迫修改（因為它們都被迫依賴了不需要的方法）。這就違反了 OCP


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/ISP/4_OCP.png)


ISP 把龐大介面拆分成小介面後，新的需求只需要新增或擴充介面，而不是去修改既有的介面或類別，我們藉由遵循 OCP 留好擴充點，但擴充的時候沒有做好 ISP 仍會發生問題

例如需要新增「傳真功能」時，只要定義新的介面（IFaxFunction），而不用去動既有的 IPrintFunction。這樣一來，既有程式碼不需要修改，卻能支援新的功能，正好符合 OCP 的精神


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/ISP/4_OCP_fix.png)


<!-- endtab -->

<!-- tab summary-->

![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/ISP/5_summ.png)

<!-- endtab -->


{% endtabs %}

