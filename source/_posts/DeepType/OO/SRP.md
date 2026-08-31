---
title: Single Resposibility Principle
date: 2025-09-17 08:00:03
categories: OO
top_img: https://i.imgur.com/LF7L1Ws.png
cover : https://i.imgur.com/LF7L1Ws.png
tags:
    - OO
toc:
toc_number:
comments :
---


{% tabs Single Resposibility Principle%}


{% btn 'https://blog.ndepend.com/solid-design-the-single-responsibility-principle-srp/',SOLID Design in C#: The Single Responsibility Principle (SRP),far fa-hand-point-right %}


<!-- tab SRP 想解決甚麼問題-->

在系統設計裡，最常見也最棘手的問題之一，就是「改 A 怕改到 B」。過度耦合會讓維護變得艱難：一個看似單純的修改，卻可能意外影響其他功能。這也是單一職責原則 (Single Responsibility Principle, SRP) 想要解決的核心：一個組件（class、模組、服務）應該只有一個「變動的原因」


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/SRP/1_waht%20_is_respons.png)



<!-- endtab -->

<!-- tab 職責的本質：來自業務，而非技術-->

那麼什麼是「職責」？不是單純的「資料庫」、「控制器」或「畫面元件」，而是 業務驅動的需求。一個組件的職責，應該來自同一群「對它提出業務需求的人」。

舉例來說：

- 帳戶狀態管理：由風控或會員管理部門提出需求。
- 用戶等級規則：由行銷或會員成長團隊提出需求。

這些需求雖然都跟「用戶」有關，但來自不同的業務關聯方。如果硬是把它們混在同一個組件裡，將來只要一邊業務有變化，另一邊就會被波及，導致「明明沒動它，卻壞了」。


<!-- endtab -->

<!-- tab 規模越大，越容易失控-->

在小型專案裡，把功能寫在一起好像還能撐住。但當系統規模逐漸擴大、業務日益複雜，就會開始顯現問題

- 修改週期變慢：每次改動都要檢查一堆不相關的功能。
- 心智負擔增加：維護的人需要搞懂與需求無關的程式碼。
- Bug 傳染：一個小需求意外導致全系統出錯。

SRP 就像保險帶，讓系統「經得起迭帶」。就算發生劇烈的業務變動，也能確保傷害被侷限在單一職責的範圍內


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/SRP/1_violate_SRP_problem.png)


<!-- endtab -->


<!-- tab API-->

曾經在某一個開發項目中，前端需要在畫面顯示前須藉由某一支 API 取得客製化開關，以判斷該商店是否啟用此功能，後續迭代發現，需要額外增加邏輯排除 "即使開關為開也不能顯示該功能在畫面上" 的需求，當時要求是在同一支 API 新增該邏輯，但後續考量到 API 本身的特性以及後續維護的困難(後續要拔除開關時，裡面多了一些不相關的邏輯，是否要多一些工評估這個額外邏輯在做甚麼?)，決定請前端個別使用兩隻 API，我想這是典型的 SRP 問題


<!-- endtab -->

<!-- tab 節點設計問題-->

在專案中，我們常遇到一個情境：業務方臨時提出需求，要求在現有 API 上快速擴充功能。
舉例來說：我們現在有一個「帳戶狀態 API」，回傳使用者是否 停權/正常。突然業務方提出：需要一個「會員等級 API」來顯示用戶等級。有人提議：「既然停權的帳戶不能使用功能，那我們就直接用『沒被停權 = 基本會員』來判斷吧！」

這種做法表面上快速，但其實 違反了 SRP。原因是：

帳戶狀態 → 由風控/安全需求主導。
會員等級 → 由行銷/會員成長需求主導。

這是兩個完全不同的業務關聯方。如果我們把「停權狀態」直接拿來充當「會員等級」，未來只要任何一邊業務規則變動，另一邊就會被牽連。


分離 API：帳戶狀態的 API 就只回傳帳戶狀態，不要摻雜等級資訊。
獨立會員等級 API：會員等級應該由專門的會員等級邏輯來判斷。

```CSHARP
// 停權狀態 API (風控/會員管理需求)
public class AccountStatusService
{
    public bool IsSuspended(Guid memberId)
    {
        // 判斷帳號是否停權
        return false;
    }
}

// 會員等級 API (行銷/會員成長需求)
public class MemberLevelService
{
    public string GetLevel(Guid memberId)
    {
        // 回傳 Bronze / Silver / Gold
        return "Silver";
    }
}
```

這樣就能避免一個組件因多個業務原因而頻繁變動。SRP 的重點就是讓「業務需求的變動」不會互相傳染


![x](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/SRP/3_api_seperate.png)


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/SRP/3_convenient_is_enemy.png)


<!-- endtab -->


<!-- tab 為什麼要區分 DTO、Entity、EF Core Model？-->

如果資料庫變動會污染業務邏輯，假如 DB 結構改了（Email 拆成 EmailLocal + EmailDomain）

當整個 User class 要大改，連帶業務邏輯與 DTO 都被牽連。而業務邏輯變動會污染 DB/DTO，需求說「折扣邏輯要分會員等級」→ 你必須改 User class，可能同時影響 EF Core Mapping 與 API Response。

又或者前端突然要「DisplayName = Name + '(' + Email + ')'」→ 你加了一個屬性，結果 EF Core 嘗試去 Mapping，它會誤以為這也是資料庫欄位。

**EF Core Entity (資料庫實體)**

對應資料表結構 (User, Order …)。負責「存取資料庫」。會有 EF 特定的設定（Navigation Property、Fluent API、Annotation）。

**Domain Entity (領域實體 / 商業邏輯實體)**

負責業務邏輯（如計算折扣、驗證規則）。應該不依賴 EF Core，否則你的業務邏輯就被資料庫綁死。

**DTO (資料傳輸物件)**

專門用來做資料輸入/輸出（Controller ↔ ViewModel ↔ API）。通常不包含邏輯，只是為了「資料傳遞方便」。

👉 它們來自三個不同「變動原因」：

資料庫改結構 → EF Entity 需要修改。
業務規則改變 → Domain Entity 需要修改。
前端需求改變 → DTO 需要修改。


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/SRP/4_why_dto_entity.png)



SRP 的想法：每個類別只對一種業務關聯方負責


![1](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/SRP/4_change_reason_3.png)


<!-- endtab -->


<!-- tab 業務邏輯-->


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/SRP/2_employee.png)


```CSHARP
public class Employee { 
    public string Name { get; set; }
    public string Address { get; set; } 
    ...
    public void ComputePay() { ... }     // 薪資計算 → 財務部門關心
    public void ReportHours() { ... }    // 工時回報 → 人資/排班系統關心
}
```

資料本體（Employee 資料）Name, Address 屬於員工資訊。這是員工基本屬性，沒有問題。
但薪資計算邏輯，ComputePay() 牽涉到薪資公式、加班費、獎金規則。這是「財務/薪資系統」的職責。而工時回報邏輯，ReportHours() 牽涉到打卡紀錄、工時計算。這是「人資/工時系統」的職責。

結果就是 員工類別同時要滿足多個部門（財務、人資）的需求，它就有多個「變動原因」。
一旦薪資算法改了，你要動 Employee；一旦工時規則改了，你也要動 Employee → 這樣就違反了 SRP


```CSHARP
// 員工資料，只負責「承載資訊」
public class Employee
{
    public string Name { get; set; }
    public string Address { get; set; }
}

// 薪資計算服務
public class PayrollService
{
    public decimal ComputePay(Employee employee)
    {
        // 計算薪資邏輯
        return 1000m;
    }
}

// 工時回報服務
public class TimeReportingService
{
    public void ReportHours(Employee employee, int hours)
    {
        // 工時回報邏輯
        Console.WriteLine($"{employee.Name} 報工時 {hours} 小時");
    }
}
```


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/SRP/2_srp_seperate_fix.png)


<!-- endtab -->


<!-- tab 如何實踐：拆解職責-->

**資料層 (DB / Repository)**

只專注於「存取與儲存」資料，而不是混入業務邏輯。

**Use Case / Flow**

每個 Use Case 對應單一業務流程，例如「會員升級」、「帳號停權」。不要讓多個部門的需求擠在同一個流程裡。

**Controller / ViewModel**

控制輸入輸出，負責跟外部互動。保持輕量，不要讓它背負太多業務規則。

**共用流程**

如果不同的業務流程共享一段邏輯（例如驗證用戶是否存在），那就抽取成「穩定的共用元件」。但不要讓共用邏輯承擔業務的差異，而是讓各自的 Printer、View 去改動


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/SRP/5_shared_logic.png)



單一職責原則不是在追求「程式碼越小越好」，而是在降低耦合度，讓系統能承受業務變化。

一個組件只有一個變動原因。
業務驅動，而不是技術堆疊。
規模越大，越要堅守這條原則。

如此一來，系統才不會被業務拖垮，而能像穩健的建築，經得起風雨與震動。



![d](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/SRP/6_final.png)


<!-- endtab -->

{% endtabs %}