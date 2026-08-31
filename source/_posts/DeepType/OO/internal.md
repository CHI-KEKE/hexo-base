---
title: internal - 不要偷看!
date: 2025-09-14 11:00:03
categories: OO
top_img: https://i.imgur.com/LF7L1Ws.png
cover : https://i.imgur.com/LF7L1Ws.png
tags:
    - OO
toc:
toc_number:
comments :
---

{% tabs internal %}

<!-- tab 組件（Assembly）-->

在 .NET 中，「組件（Assembly）」是一個可以被執行或被載入的最小單位，通常就是一個 **.dll（類別庫）** 或 **.exe（應用程式執行檔）**。  

Dynamic Link Library（動態連結程式庫）。它是一個 二進位檔（binary file），副檔名是 .dll。
裡面存放的是「程式碼 + 資源」，但不能自己執行。要靠其他程式（例如 exe）來「載入」它，才能使用裡面的功能。簡單說，DLL 就像一本工具書，裡面有很多函式與類別，你可以引用它，但它不會自己動


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/Internal/1_dll.png)



在 Windows 下，DLL/EXE 都遵循 PE（Portable Executable）格式。.NET 世界裡，DLL 裡面會多一層 IL（Intermediate Language，中間語言） 和 Metadata（中繼資料）。也就是說，.NET 的 DLL 並不是直接的 CPU 機器碼，而是「IL + Metadata」組合。

而 Visual Studio 的「Project」編譯出來之後，通常就是一個組件（Assembly）📦

例如像這樣

| 專案名稱（Project）        | 編譯後的組件（Assembly）         |
| -------------------- | ------------------------ |
| MyApp                | MyApp.exe                |
| MyApp.Core           | MyApp.Core.dll           |
| MyApp.Infrastructure | MyApp.Infrastructure.dll |


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/Internal/3_proj_assamly.png)


**internal** 的意思是：這個類別或成員**只能在同一個組件（Assembly）裡被使用**，外部組件無法直接存取。  
換句話說，它就是「只給自己家人用的成員，不對外開放」。


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/Internal/4_external_block.png)


**p.s. binary file 指得是裡面的 0/1 沒有直接對應到文字，而是依照某種格式定義，給「程式」來解讀，它不是人類可以直接讀的文字，而是 給作業系統 / CLR 用的結構化資料。**



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/Internal/2_pe.png)


<!-- endtab -->

<!-- tab internal 想解決的問題-->


這其實與「封裝（Encapsulation）」和「模組化（Modularity）」密切相關。本質上 internal 解決的是：「避免內部細節被外部依賴或誤用」。 

- **外部誤用**：其他開發者可能把你只打算內部用的類別，拿來當正式 API。  
- **依賴綁死**：別人對內部實作產生依賴，導致你無法隨意修改。改了可能會壞一堆外部程式，不改又會變成技術債。  
- **測試污染**：你原本只打算給測試用的類別（例如 Mock、Stub），可能被誤以為是正式功能。


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/Internal/6_public_risks.png)


所以 **internal 就像是在說：**「這東西是內部細節，我不保證它的穩定性，也不打算提供給外部使用。」它為你劃出一個「自由調整的區域」，讓你能靈活修改內部設計，而不用擔心牽連外部使用者。


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/Internal/5_encapu.png)



internal 的存在並不是限制，而是一種保護。它讓設計者能清楚表達：「哪些東西是 API（保證穩定），哪些只是內部零件（隨時可能調整）」。當你合理運用 internal，就能保持專案的邏輯清晰、邊界明確，讓程式碼既有彈性又不失秩序。這是一種軟體設計者的自我約束，也是一種給使用者的承諾


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/Internal/7_protect.png)


<!-- endtab -->

<!-- tab Helper 或 Utility 類別-->


```csharp
internal class PasswordHasher
{
    public string Hash(string password) => ...
}
```

這類工具類別通常只是專案內部的實作細節。如果真的需要跨組件共用，就應該另外建立一個獨立的 Utility/Shared 專案，並把需要公開的工具類別設為 public。internal 強調了責任劃分：哪些是內部使用，哪些是跨專案共用


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/Internal/8_1_detail_dodge.png)




<!-- endtab -->

<!-- tab 大型專案結構的流程物件-->


假設你有一個大型專案結構如下

- MyApp.Domain（商業邏輯）
- MyApp.Application（應用邏輯）
- MyApp.Infrastructure（第三方套件實作）
- MyApp.API（Web API）

你可能會在 Application 裡面有一些 handler 類別，只給內部 Command 用：

```csharp
internal class CreateOrderFlowHandler
{
    public void Execute(CreateOrderInput input) { ... }
}
```


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/Internal/8_2_encap.png)


<!-- endtab -->

<!-- tab 測試專用 Stub 或 Mock class-->


有時候為了單元測試，你會做一些模擬類別或資料，這些不應該給正式程式使用：

```csharp
internal class MockPaymentGateway : IPaymentGateway
{
    public void Pay() => Console.WriteLine("Fake payment done.");
}
```

讓測試專案可以存取 internal
```csharp
// AssemblyInfo.cs 加這一行
[assembly: InternalsVisibleTo("MyApp.Tests")]
```


![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/Internal/8_3_test.png)


<!-- endtab -->


<!-- tab Factory 或 Builder 的內部部件-->

```csharp
internal class OrderItemValidator
{
    public bool Validate(OrderItem item) { ... }
}
```

你用這個 Validator 來組成一個更大的處理流程，這個小部件 class 是內部結構之一。


<!-- endtab -->


<!-- tab summary-->

![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/OO/Internal/9_decision.png)

<!-- endtab -->


{% endtabs %}