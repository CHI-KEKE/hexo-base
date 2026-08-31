---
title: MVC - CRUD
date: 2025-09-26 09:21:34
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

<br>

## 🎃 Detail

- 要顯示哪些資料 (Projection)
- Routing + Model Binding 怎麼把 id 傳進去
- 找不到資料要怎麼回應

<br>

### 顯示哪些資料 (Detail)

以學生（Student）為例，Detail 頁面通常只需要基本資料

- 姓名
- 入學日期
- 學號

不需要顯示「選修課程清單」（因為是一個集合，會放在「相關資料」頁面或 API）。

這樣做包含效能考量，如果每次打 Detail API 都把所有課程也撈出來，會很重。並且也讓職責清楚 Detail = 單一物件的資料，不是「關聯清單」。

<br>

### Routing + Model Binding

```plaintext
http://localhost:1230/Student/Details/1
```

對應 Routing 規則
```plaintext
url: "{controller}/{action}/{id}"
```

Controller
```csharp
public async Task<IActionResult> Details(int? id)
```

<br>

### 為什麼要設計成 int? id？

如果 URL 根本沒帶 id，例如 /Student/Details/ → Model Binding 會傳 null，而不是直接拋 Exception。

這樣你就可以自己判斷
```csharp
if (id == null) return BadRequest();
```

### 找不到資料時該怎麼處理？
```csharp
public async Task<IActionResult> Details(int? id)
{
    if (id == null)
    {
        return BadRequest();
    }

    var student = await _context.Students
        .Include(s => s.Enrollments)
            .ThenInclude(e => e.Course)
        .FirstOrDefaultAsync(s => s.Id == id);

    if (student == null)
    {
        return NotFound();
    }

    return View(student);
}
```

這裡有兩種設計選擇：

- 回 404 Not Found 代表資源不存在。適合 RESTful API 設計。
- 回 200 OK + 空結果 (或 API Code), 代表請求成功，但查無資料。適合「前端一定要收到固定格式」的情境。在商業系統或內部 API，有時候會選 200 + 自訂錯誤碼，方便前端統一處理。

<br>
<br>

## 🎃 Create

在「建立」功能 (Create) 裡，會遇到兩個很重要的安全議題

- 防止 Overposting (過度繫結)
- 防止 CSRF (跨站請求偽造)


### 防止 Overposting

使用者提交的表單或 JSON，如果直接綁定到 Entity (資料庫模型)，就可能讓駭客偷偷多送一些欄位（例如 IsAdmin、Price、Role…），導致非預期的修改。我們應該使用 ViewModel / DTO，只包含允許填寫的欄位。Controller 收到 ViewModel 後，再手動轉換成 Entity 存進資料庫。
```CSHARP
public class ProductCreateDto
{
    public string Name { get; set; }
    public decimal Price { get; set; }
}

[HttpPost]
public IActionResult Create(ProductCreateDto dto)
{
    var product = new Product
    {
        Name = dto.Name,
        Price = dto.Price
    };

    _context.Products.Add(product);
    _context.SaveChanges();

    return Ok("新增成功");
}
```

<br>

### 防止 CSRF (Cross-Site Request Forgery)

**問題情境**

使用者登入你的網站後，瀏覽器會保存 Cookie（Session ID 或認證票證）。駭客引誘使用者點一個連結或假表單 → 瀏覽器自動帶上合法 Cookie。系統誤以為這是使用者的合法操作，結果就執行了敏感動作（轉帳、刪資料…）。

**防禦方法**

- 在 View 表單加上 @Html.AntiForgeryToken()。
- Controller Action 加上 [ValidateAntiForgeryToken]。


View 範例 (Razor)
```csharp
@using (Html.BeginForm("TransferMoney", "Account", FormMethod.Post))
{
    @Html.AntiForgeryToken()
    <input type="text" name="toAccount" placeholder="轉帳帳號" />
    <input type="number" name="amount" placeholder="金額" />
    <button type="submit">確認轉帳</button>
}
```
產生結果 (隱藏欄位)
```csharp
<input name="__RequestVerificationToken" type="hidden" value="隨機加密字串" />
```

Controller 範例
```csharp
@using (Html.BeginForm("TransferMoney", "Account", FormMethod.Post))
{
    @Html.AntiForgeryToken()
    <input type="text" name="toAccount" placeholder="轉帳帳號" />
    <input type="number" name="amount" placeholder="金額" />
    <button type="submit">確認轉帳</button>
}
```
產生結果 (隱藏欄位)
```csharp
[HttpPost]
[ValidateAntiForgeryToken]
public IActionResult TransferMoney(string toAccount, decimal amount)
{
    // 只有驗證成功的 Token 才能進來
    return Content("轉帳成功！");
}
```

因為 Token 具有以下性質

- 隨機性：每次頁面載入都不同。
- 伺服器簽章：不能偽造。
- 綁定使用者 Cookie：Token 只對應到該使用者。

所以駭客網站就算模仿表單，也無法產生合法 Token → 驗證失敗，請求會被擋掉。
```html
<form action="https://your-site.com/Product/Create" method="post">
  <input type="hidden" name="Name" value="惡意商品" />
  <input type="hidden" name="Price" value="999999" />
  <button type="submit">點我領獎金！</button>
</form>
```

**什麼情境需要 CSRF 防護？**

- API / Controller 使用 Cookie 驗證（例如 ASP.NET Identity）。
- 因為 Cookie 會自動附帶，駭客網站可以利用。


不需要 (或較少風險) 情境可能是

- API 使用 JWT、Bearer Token、API Key。
- Token 不會自動加在 Cookie 裡，而是前端自己放到 Header。
- 駭客網站受 CORS 限制，拿不到 Token。
- 這種情境下，比較要小心 XSS，因為如果有人在你的網站插惡意 script，就能偷走 Token。

<br>
<br>

## update

- 防止 Overposting（過度繫結）
- 處理 Null 與 NotFound
- 異常處理（DbUpdateException）

資料庫更新可能會失敗，例如 資料格式錯誤、資料庫連線問題、FK/Constraint 違反，要捕捉 DbUpdateException，並給使用者清楚訊息（不要直接把 Exception 訊息丟給前端，避免洩漏細節）。

- Concurrency（並發衝突）
在 Model 加一個 Concurrency Token（例如 RowVersion）。Update 時帶上 RowVersion，EF 會自動檢查版本是否一致，不一致會丟 DbUpdateConcurrencyException。

- 驗證 ModelState
即使更新，只允許特定欄位，也要確保資料符合驗證規則（例如長度、必填）。如果驗證失敗，要回傳原本的 studentToUpdate 回 View，並顯示錯誤訊息。

- 安全性與授權
確保只有授權的使用者可以修改該筆資料（例如學生只能改自己的資料，管理員才能改全部）。在 Controller 加上 [Authorize]，必要時在程式裡再檢查「這筆資料是否屬於當前登入的使用者」。

例子
```csharp
[HttpPost, ActionName("Edit")]
[ValidateAntiForgeryToken]
public async Task<IActionResult> EditPost(int? id)
{
    if (id == null)
    {
        return BadRequest();
    }

    var studentToUpdate = await _context.Students.FindAsync(id);
    if (studentToUpdate == null)
    {
        return NotFound();
    }

    // 只允許指定欄位更新
    if (await TryUpdateModelAsync<Student>(
        studentToUpdate,
        "",
        s => s.LastName, s => s.FirstMidName, s => s.EnrollmentDate))
    {
        try
        {
            await _context.SaveChangesAsync();
            return RedirectToAction(nameof(Index));
        }
        catch (DbUpdateException /* ex */)
        {
            ModelState.AddModelError("", "Unable to save changes. Try again, and if the problem persists, see your system administrator.");
        }
    }
    return View(studentToUpdate);
}
```

<br>
<br>

##  🎃 結語

走完 CRUD 的旅程，我們會發現：這些 API 就像是一群老師傅。

- Detail 是那位眼光精準的師傅，他只挑出需要展示的部分，不多也不少，讓人一眼就明白重點。
- Create 是那位嚴謹的工匠，懂得小心防範，不讓不該出現的東西混進來，還要守護整個工坊不被外人惡意利用。
- Update 是那位細心的維修師，他懂得留意小地方的風險，避免別人同時動手造成衝突，也會在意工件的完整性與正確性。
- Delete則像是穩重的清道夫，知道該怎麼安全地移除東西，不留後患。

這些 CRUD API 不是花俏的建築師，而是一群務實、細膩、默契十足的老師傅。他們在背後默默維護著系統的秩序，讓資料能穩定地被建立、查詢、修改與刪除。
也因此，當我們在設計這些 API 時，不只是單純地完成功能，而是在傳承一種工藝