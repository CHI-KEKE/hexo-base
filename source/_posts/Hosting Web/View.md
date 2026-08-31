---
title: MVC - View
date: 2025-09-22 21:41:34
categories: MVC
top_img: https://github.com/CHI-KEKE/pics/blob/main/MVC/pumpkin.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/MVC/pumpkin.png?raw=true
tags:
    - MVC
toc:
toc_number:
comments :
---

![Image](https://github.com/CHI-KEKE/pics/blob/main/MVC/pumpkin.png?raw=true)


在茂密的森林中，鳥兒會築巢 🐦，蜜蜂會傳花粉 🐝，松鼠會儲存堅果 🐿️。

其實網站開發也一樣，每一個畫面（View）就像是一個生命體，它有自己的「家」（Layout）、需要不同的「資訊傳遞方式」（ViewData / ViewBag），甚至還要有準備完善的「儲藏糧食系統」（ViewModel）。

這篇文章，我們就從自然界的角度，帶你用全新的視角理解：

- 🐦 Layout 是如何像鳥巢一樣包住每個頁面
- 🐝 ViewData / ViewBag 如何像蜜蜂快速傳遞資訊
- 🐿️ ViewModel 為什麼像松鼠預先打包好資料，確保整潔與安全


MVC 的 View，是一個「純粹負責呈現資料的角色」，它不處理邏輯、不處理資料，只專注「怎麼顯示」。

<br>

## MVC vs Razor Pages（.NET 內建）

| MVC View                         | Razor Page                |
| -------------------------------- | ------------------------- |
| View 是獨立的、由 Controller 控制        | View 和處理邏輯寫在一起（PageModel） |
| 較適合大型應用程式分層設計                    | 較適合小型頁面導向開發               |
| 分離更清楚：Controller/Model/View 各有職責 | 合併邏輯與畫面，開發速度快但職責模糊        |
| 像是「三人分工合作」                       | 像是「一人包辦全部」                |


✅ MVC 更有結構與彈性、適合多人開發與模組化系統。

<br>
<br>

## MVC vs SPA 框架（React / Vue / Angular）

| MVC View                | SPA View                         |
| ----------------------- | -------------------------------- |
| 伺服器產生 HTML              | 前端產生 HTML（Client-side rendering） |
| 資料先經 Controller 再進 View | 資料通常從 API 取得直接渲染                 |
| 適合傳統網站架構                | 適合高度互動式應用                        |
| SEO 友好、初始載入快            | 互動性高、體驗滑順                        |

✅ MVC View 適合「頁面導向、穩定、分層清晰」的 Web 系統，而 SPA 則更注重使用者體驗與互動性。

<br>
<br>

## 🎃 _Layout.cshtml

Layout 是一種「共用版型模板」，負責網站頁面中重複出現的外觀與結構，例如：頁首、頁尾、導覽列等等。

<br>
<br>

## 🧩 @RenderBody()

這是一個「插槽（Placeholder）」，用來顯示每個頁面自己的內容（View 內容）。當你開啟某個 View 頁面時，它的 HTML 就會被塞進這裡。

```csharp
<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>@ViewData["Title"] - MyWebsite</title>
    <link rel="stylesheet" href="..." />
</head>
<body>
    <header>網站標題/導覽列</header>

    <main>
        @RenderBody()
    </main>

    <footer>網站版權</footer>
</body>
</html>
```

| 區塊                       | 本質與功能                            |
| ------------------------ | -------------------------------- |
| `<!DOCTYPE html>`        | 告訴瀏覽器「這是一份 HTML5」文件。             |
| `<html lang="zh-Hant">`  | 指定語言為繁體中文，有助於搜尋引擎與語音朗讀器。         |
| `<head>`                 | 儲存「網站的設定資訊」：字元集、樣式表、標題等。         |
| `<meta name="viewport">` | 讓網站能夠在手機上適應不同寬度（響應式設計）。          |
| `<title>`                | 設定瀏覽器標籤的文字，可由 Controller 傳入動態資料。 |

<br>
<br>

## 🎃 ViewData vs ViewBag vs ViewModel：傳資料到 View 的各種方式

這兩個工具都是讓 Controller 把資料「暫時傳給 View 使用」的方式。

| 比喻       | 說明                                    |
| -------- | ------------------------------------- |
| ViewData | 就像一張小紙條（字典），你寫好資料貼上 key，傳給小朋友（View）看。 |
| ViewBag  | 就像你直接跟小朋友講：「這個資料叫什麼」；口語但沒保障。          |

<br>

### 簡易實作
```csharp
public class HomeController : Controller
{
    public IActionResult Index()
    {
        // 用 ViewData 傳值（像字典）
        ViewData["Title"] = "這是首頁";
        ViewData["Message"] = "Hello from ViewData";

        // 用 ViewBag 傳值（像屬性）
        ViewBag.Note = "Hello from ViewBag";

        return View();
    }
}


<h1>@ViewData["Title"]</h1>
<p>@ViewData["Message"]</p>

<hr />

<p>@ViewBag.Note</p>
```

**ViewData**
- Dictionary<string, object> → 一張小紙條，存 key/value。
- 缺點是要轉型，要記得 key。

**ViewBag**
- 本質是 dynamic 動態物件。
- 底層其實就是
```csharp
public dynamic ViewBag => new DynamicViewData(() => ViewData);
```
所以它其實只是 ViewData 的「語法糖」


| 項目   | ViewData                    | ViewBag        |
| ---- | --------------------------- | -------------- |
| 類型   | Dictionary\<string, object> | dynamic        |
| 語法   | `ViewData["Key"]`           | `ViewBag.Key`  |
| 型別安全 | ❌ 需要轉型                      | ❌ 編譯時無檢查       |
| 智慧提示 | ❌ 沒有 IntelliSense           | ❌              |
| 本質   | 明確鍵值對                       | ViewData 的動態包裝 |
| 推薦用法 | 控制精準與強型別使用                  | 快速開發測試時可用      |

<br>

### 🧳 ViewModel

ViewModel 是為 View 客製的資料模型，只裝 View 要的東西，乾淨、精簡、型別安全


- ViewData / ViewBag 是「便條紙」：隨手記、容易搞錯、不好維護
- ViewModel 是「整齊打包的禮盒」：資料乾淨、欄位明確，方便交給畫面顯示


**實作**

Step 1：建立 ViewModel
```csharp
// Models/ViewModels/StudentViewModel.cs
public class StudentViewModel
{
    public string Name { get; set; }
    public string Email { get; set; }
}
```

Step 2：Controller 傳 ViewModel 給 View
```csharp
public class StudentsController : Controller
{
    public IActionResult Index()
    {
        var students = new List<StudentViewModel>
        {
            new StudentViewModel { Name = "小明", Email = "ming@example.com" },
            new StudentViewModel { Name = "小美", Email = "mei@example.com" }
        };

        return View(students);
    }
}
```

Step 3：View 接收並顯示 ViewModel
```csharp
@model List<StudentViewModel>

<h2>學生列表</h2>
<table class="table">
    <thead>
        <tr>
            <th>姓名</th>
            <th>Email</th>
        </tr>
    </thead>
    <tbody>
        @foreach (var student in Model)
        {
            <tr>
                <td>@student.Name</td>
                <td>@student.Email</td>
            </tr>
        }
    </tbody>
</table>

```

| 優點         | 說明                            |
| ---------- | ----------------------------- |
| ✅ 強型別      | 有 IntelliSense、自動補齊           |
| ✅ 型別安全     | 編譯時檢查，減少錯誤                    |
| ✅ 結構清楚     | View 需要什麼資料就提供什麼              |
| ✅ 易於測試     | 可以獨立測試，不綁資料庫                  |
| ✅ MVC 架構清晰 | 維持「分工明確」：資料（Model）與畫面（View）分離 |


為什麼不能直接用 Entity Model？
| 問題      | 原因                   |
| ------- | -------------------- |
| 太多不必要欄位 | Entity 通常會含一堆不會顯示的資料 |
| 效能與安全疑慮 | 若包含關聯資料或敏感欄位，可能外洩    |
| 格式轉換不便  | 像日期、地址等常要轉格式         |