---
title: ValidateAntiForgeryToken
date: 2025-10-05 22:38:11
categories: 
top_img: https://github.com/CHI-KEKE/pics/blob/main/Security/wolf.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Security/wolf.png?raw=true
tags:
    - 
toc:
toc_number:
comments :
---

![Image](https://github.com/CHI-KEKE/pics/blob/main/Security/wolf.png?raw=true)


| 名稱                          | 是什麼？                       | 用來防什麼？              |
| --------------------------- | -------------------------- | ------------------- |
| `ValidateAntiForgeryToken`  | ASP.NET MVC 內建的 CSRF 防護機制  | 防止他站用惡意表單偽造 POST    |
| 自訂資料安全 Token（如 SecureToken） | 自己產生一組防竄改的驗證碼              | 防止表單中的敏感參數（如 ID）被修改 |
| `ISecureTokenEntity`        | 自定義的介面，代表此 Entity 具有安全驗證需求 | 提供加密驗證方法來驗證資料       |




## 🧱 假設場景

你有一個表單要刪除一筆資料：

```HTML
<form method="post" action="/product/delete">
    @Html.HiddenFor(m => m.ProductId)
    @Html.Hidden("SecureToken", Model.SecureToken)
    <button type="submit">刪除</button>
</form>
```

## 🔐 第一步：產生 SecureToken（綁定欄位值）

你在 Controller 的 GET 頁面中，要建立一個「不可竄改」的驗證碼
```CSHARP
public ActionResult Edit(int id)
{
    var product = _productService.GetById(id);

    product.SecureToken = SecureTokenHelper.GenerateToken(product); // ✅ 自定方法

    return View(product);
}
```
SecureTokenHelper.GenerateToken() 做什麼？

- 取出關鍵欄位，例如：ProductId, UserId, TimeStamp
- 組成字串（例如："123|user99|2025-11-16"）
- 加密（例如：HMAC + 密鑰）
- 得到一組 Token → 存入 Model.SecureToken

🔒 這個 Token 只對這組參數有效，一改就驗證失敗


## ✅ 第二步：畫面表單送出時，夾帶 Token

你在 Razor 表單中會這樣
```csharp
@using (Html.BeginForm("Delete", "Product", FormMethod.Post))
{
    @Html.HiddenFor(m => m.ProductId)
    @Html.Hidden("SecureToken", Model.SecureToken)
    @Html.AntiForgeryToken() <!-- ✅ 防止 CSRF -->
    <button type="submit">確定刪除</button>
}
```

## 🔐 第三步：POST 接收端 → 檢查 CSRF + SecureToken

```csharp
[HttpPost]
[ValidateAntiForgeryToken]
public ActionResult Delete(ProductEntity model)
{
    // ✅ 步驟 1：先驗證 SecureToken
    if (!SecureTokenHelper.ValidateToken(model))
    {
        return new HttpStatusCodeResult(400, "資料驗證失敗，疑似被竄改");
    }

    // ✅ 步驟 2：執行刪除
    _productService.Delete(model.ProductId);

    return RedirectToAction("Index");
}
```

## ✅ 第四步：ISecureTokenEntity 與 Token 驗證邏輯

Interface 定義
```csharp
public interface ISecureTokenEntity
{
    string SecureToken { get; set; }
}
```

Entity
```csharp
public class ProductEntity : ISecureTokenEntity
{
    public int ProductId { get; set; }
    public string SecureToken { get; set; }
}
```

Helper 實作
```csharp
public static class SecureTokenHelper
{
    private static readonly string _secretKey = "SuperSecretKey123"; // 建議放設定檔

    public static string GenerateToken(ISecureTokenEntity entity)
    {
        var data = $"{entity.GetType().Name}|{entity.GetId()}"; // 自訂擴充方法
        return Hash(data + _secretKey);
    }

    public static bool ValidateToken(ISecureTokenEntity entity)
    {
        var expected = GenerateToken(entity);
        return expected == entity.SecureToken;
    }

    private static string Hash(string input)
    {
        using (var sha256 = SHA256.Create())
        {
            var bytes = Encoding.UTF8.GetBytes(input);
            var hash = sha256.ComputeHash(bytes);
            return Convert.ToBase64String(hash);
        }
    }
}
```

```plaintext
[GET /Edit/123]
   ↓
取出 ProductId = 123
   ↓
產生 SecureToken: Hash("ProductEntity|123|Secret")
   ↓
傳到畫面（隱藏欄位）
   ↓
[使用者提交 POST /Delete]
   ↓
[後端驗證 AntiForgeryToken（CSRF）]
   ↓
[驗證 SecureToken 是否為原本預期的組合]
   ↓
通過 → 刪除資料
```


CSRF 是什麼？

CSRF 是「利用你已登入某網站」這件事來騙你替駭客送出請求。

駭客不是攻擊系統本身，而是「利用你的瀏覽器」幫他送出惡意請求。

① 方式一：駭客在別的網站放一個惡意表單（最經典）
👉 假設你的網站有一個會刪使用者資料的 API：
POST /user/delete/123
Cookie: sessionId=ABCDEFG


你在你的網站中刪除自己的帳號時，用的是這個 POST。

🔥 駭客在他的網站放這個：
<form action="https://your-site.com/user/delete/123" method="POST" id="attack">
</form>

<script>
    document.getElementById("attack").submit();
</script>

❗ 使用者不小心點進駭客的網站（例如點廣告）
❗ 你的瀏覽器會自動附上 Cookie（因為你登入過）
❗ POST 就自動發送到你網站
❗ 你的網站以為是本人操作 → 直接刪除帳號！

→ 這就是 CSRF：在你不知情的情況下，代替你操作。


② 方式二：偽造圖片請求（GET 攻擊）

如果你的網站有：

GET /money/transfer?to=Hacker&amount=10000


駭客放一張假圖片：

<img src="https://your-site.com/money/transfer?to=Hacker&amount=10000">


只要你瀏覽這個頁面 → 請求自動送出 → 附帶 Cookie → 完成轉帳。

③ 方式三：偽造 AJAX 請求（若 CORS 設太鬆）
fetch("https://your-site.com/pay", {
  method: "POST",
  body: "amount=50000&to=Hacker"
});


如果你的 CORS 錯誤設定為：

Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true


那駭客網站可以直接發 AJAX 攻擊你。

