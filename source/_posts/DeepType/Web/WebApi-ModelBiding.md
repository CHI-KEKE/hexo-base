---
title: MODEL Binding
date: 2025-09-23 23:07:34
categories: Web
top_img: https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
tags:
    - Web
toc:
toc_number:
comments :
---

{% tabs MODEL Binding%}


<!-- tab Model Binding-->

這一切其實都圍繞著一個核心：溝通與理解

HTTP 的世界裡，Request 扮演著使用者「開口說話」的角色，而 Model Binding 則像是一位貼心的翻譯員，幫我們把這些話語準確地轉換成程式可以理解的變數或物件


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ModelBinding/1__translator.png)


<!-- endtab -->

<!-- tab [FromQuery] – 查詢字串-->


## 商品搜尋 + 分頁


```bash
GET /products?keyword=phone&page=2&pageSize=20
```

**設定**
```csharp
[HttpGet("products")]
public IActionResult Search(
    [FromQuery] string keyword, 
    [FromQuery] int page = 1, 
    [FromQuery] int pageSize = 10)
{
    return Ok(new { keyword, page, pageSize });
}
```

**結果**
```json
{ "keyword": "phone", "page": 2, "pageSize": 20 }
```

因為這是「篩選條件」，而不是資源唯一識別。用 QueryString 比較直觀、可被書籤儲存


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ModelBinding/2_1_fromquery.png)


<!-- endtab -->

<!-- tab [FromRoute] – 路由參數-->

## 取得某個商品詳細資訊

```bash
GET /products/12345
```

**設定**
```csharp
[HttpGet("products/{id}")]
public IActionResult GetById([FromRoute] int id)
{
    return Ok(new { Id = id, Name = "iPhone 15 Pro" });
}
```

**結果**
```json
{ "Id": 12345, "Name": "iPhone 15 Pro" }
```

因為 id=12345 就是這個資源的唯一，應該放在路由裡，而不是 QueryString


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ModelBinding/2_2_fromRoute.png)



## 注意事項

1) 找不到就回 404，不要回 200 + 空物件

2) 路由參數要有型別約束與格式限制，例如只接受整數可以用 {id:int}；如果是 GUID：用 {id:guid}。這樣亂打 /products/abc 會更乾淨地被 routing 擋掉

3) ID 是「定位資源」，Query 是「調整視角/篩選」，GET /products/12345?fields=name,price、?lang=zh-TW、?include=reviews 這種是合理的：資源還是同一個，只是回傳內容或視角不同；但用 Query 來表達「是哪一個商品」會讓語意變混亂。

4) 不要把可猜的 id 當成授權機制，如果商品是公開沒問題；但如果是「訂單 / 發票 / 個資」，使用者改個 id 就能看到別人的資料就是嚴重漏洞（IDOR）。需要做權限檢查或用不可猜的識別（例如 UUID/隨機 token）。

5) 快取與條件式請求，可以考慮 ETag/Last-Modified，或 CDN cache；路由式的資源 URL 天然比較適合快取（相對於一堆複雜 query 組合）。



![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ModelBinding/2_3_query_and_Route.png)


<!-- endtab -->

<!-- tab [FromForm] – 表單欄位-->

## 註冊會員

```html
<form action="/account/register" method="post">
    <input name="username" value="kevin88" />
    <input name="email" value="kevin@example.com" />
    <input name="password" value="123456" />
    <button type="submit">註冊</button>
</form>
```

**設定**
```csharp
[HttpPost("account/register")]
public IActionResult Register(
    [FromForm] string username, 
    [FromForm] string email, 
    [FromForm] string password)
{
    return Ok(new { username, email });
}
```

**結果**
```json
{ "username": "kevin88", "email": "kevin@example.com" }
```

因為 HTML form 提交（application/x-www-form-urlencoded）就是最典型的來源

## 流程

- 瀏覽器用 <form> 送出 POST /account/register，Content-Type 是 application/x-www-form-urlencoded
- ASP.NET Core 看到這種內容格式 → 從 Body 解析成 key/value
- Model Binding 把 username/email/password 塞進 action 參數（[FromForm]）
- 你的程式做驗證、建立帳號、存資料庫
- 回傳結果（成功/失敗）

## 注意事項

1) 用 DTO 收參數，集中驗證規則， 因為當把欄位散落在參數上會讓驗證規則越寫越亂，用一個 RegisterRequest 才好維護也好加欄位。DTO 可以搭配 DataAnnotations 或 FluentValidation，一次處理完整輸入

2) 錯誤訊息要避免「帳號枚舉」，對外回應用較模糊、統一的訊息（例如「註冊失敗」），細節只記在 server log 或用內部 error code，至少避免一眼就能判斷某 email 是否已被註冊。

4) 速率限制與機器人防護要先想，加上 IP/user-agent 速率限制、CAPTCHA、email 驗證、黑名單/風控，至少確保 DB 不會被灌爆，信箱也不會被濫發驗證信

5) CSRF：只要你用 Cookie 認證就要注意，表單天生容易被「別的網站代替你送出」，Cookie 又會自動跟著送，CSRF 就是這樣來的。純註冊通常不依賴既有登入狀態，但很多系統會把註冊和「自動登入 / 綁定推薦人 / 綁定購物車」混在一起，這時 CSRF 就變得重要；若是 MVC/Razor 常會加 Anti-Forgery token。


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ModelBinding/3_from_Form.png)


<!-- endtab -->

<!-- tab IFormFile – 檔案上傳-->

## 上傳大頭照

```html
<form action="/account/upload-avatar" method="post" enctype="multipart/form-data">
    <input type="file" name="avatar" />
    <button type="submit">上傳</button>
</form>
```

**設定**
```csharp
[HttpPost("account/upload-avatar")]
public IActionResult UploadAvatar([FromForm] IFormFile avatar)
{
    var filePath = Path.Combine("wwwroot/avatars", avatar.FileName);
    using var stream = new FileStream(filePath, FileMode.Create);
    avatar.CopyTo(stream);

    return Ok(new { avatar.FileName, avatar.Length });
}

``` 

因為二進位檔案內容不能用 JSON/Query 表示，必須用 multipart/form-data。ASP.NET Core 會自動把它轉成 IFormFile


## 流程

- 用 multipart 收到檔案 → 綁成 IFormFile avatar
- 從 avatar 取得檔名/大小/內容流
- 決定存到哪（路徑、命名、權限）
- 寫入磁碟（或轉存到 S3/Blob）
- 回傳結果（通常回檔案 id / URL，而不是原始檔名）



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ModelBinding/4_iformfile.png)



## 注意事項

1) 不要用 avatar.FileName 當真正檔名，使用者給的檔名只能拿來顯示，拿來存檔就等於把你的路徑控制權交出去。
因為 FileName 可能含有奇怪字元、路徑片段、甚至刻意碰撞同名覆蓋，建議自己產生安全檔名（GUID/雪花 ID），原檔名只當 metadata

2) 路徑要固定在你允許的資料夾內，我們要做的是「把檔案放進盒子」，不是「讓對方決定你電腦哪裡可以寫」。要避免 directory traversal（../）類型的路徑穿越，使用固定根目錄 + 你產生的檔名，再用 Path.GetFullPath 檢查仍在根目錄下更保險

3) 驗證檔案類型要看內容，不要只看副檔名，因為副檔名跟 Content-Type 都可以偽造，只有檔案內容才比較有參考價值。至少做「白名單」，只允許常見圖片（jpg/png/webp），並檢查 magic bytes / 嘗試用圖片 library decode，解不開就拒絕

4) 一定要限制大小，避免被塞爆或拖垮，如果不設上限，就是在邀請別人用超大檔案把你 IO、記憶體或磁碟打到停機，要限制單檔大小、總配額、頻率（rate limit），回應要清楚（413 Payload Too Large），ASP.NET Core 也有 request body/ multipart 限制可設定。

5) wwwroot 要小心，放進去就可能變成公開檔案，當把檔案存進 wwwroot，等於預設「任何人知道路徑就能直接下載/執行」，這通常不是你想要的。頭像通常可以公開，但你還是要確保只會是「安全的圖片格式」，並避免讓上傳內容被當成可執行腳本；更安全的作法是存到非公開資料夾，再用受控 API 讀出。

6) 用 async 並正確釋放資源，因為上傳是 IO 密集，不用 async 你會白白卡住 thread，流量一上來就變慢。用 CopyToAsync、FileStream 設 useAsync: true，並加上 CancellationToken，讓客戶端中斷時能停止寫入

7) 授權與綁定：誰可以上傳、上傳給誰，如果不做授權與歸屬檢查，上傳端點很容易變成「任何人都能替任何人改頭像」。要驗證使用者登入、把檔案關聯到 userId，並確保只能改自己的資源（或有管理權限）


![32](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ModelBinding/4_2formfile_rule.png)


## MVC ? Web API?


回傳 View 才算「MVC 用途」，而回傳 JSON/狀態碼就是「Web API 用途」，但接檔案這件事兩邊用的是同一套底層能力。不要以為 IFormFile 是 MVC 才有，其實它在 Microsoft.AspNetCore.Http，Web API 端點照收，差別只是你回傳什麼（View vs JSON）

<!-- endtab -->

<!-- tab [FromBody] – Request Body (JSON)-->

## 建立新商品

```json
{
  "name": "Gaming Laptop",
  "price": 35000,
  "stock": 5
}
```

**設定**
```csharp
public class Product {
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int Stock { get; set; }
}

[HttpPost("products")]
public IActionResult Create([FromBody] Product product)
{
    return Ok(new { product.Name, product.Price });
}
``` 

```json
{ "name": "Gaming Laptop", "price": 35000 }
```

Body 只能被讀取一次，所以一個 Action 方法最多只能有一個 [FromBody]


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ModelBinding/5_frombody.png)


## 注意項目

1) 用 DTO，不要直接用 Product 實體，真實專案的 Product 可能有 Id/CreatedAt/IsDeleted/OwnerId/CostPrice 等欄位，你不想讓 client 亂塞；用 CreateProductRequest 可以只開放該給的欄位，避免 over-posting。

2) 避免 over-posting（多塞欄位就被你吃進去），client 多送的欄位你如果默默收下來，等於把系統控制權讓出去一部分。，例如 client 送 {"price":1,"isAdmin":true} 這類莫名其妙的欄位，如果你的模型剛好有對應欄位就會被綁定，後果很難收拾；DTO + 白名單欄位是基本功。

5) 金額欄位用 decimal 是對的，但要注意精度與規則，金額最怕浮點誤差與規則不一致，decimal 只是第一步，你還要定義小數位與四捨五入規則。，例如允許幾位小數、是否允許 0、要不要做貨幣欄位；否則資料會慢慢變髒

6) 重複送出與 idempotency（前端按兩次就新增兩筆），使用者網路卡一下連按兩次，你的系統如果就生兩筆商品，後面只會一直擦屁股。，可用前端禁用按鈕、後端用 request id / idempotency key（或至少用去重策略）避免重複建立

7) [FromBody] 一個 action 只能有一個，怎麼解決「想要多東西」？你不是要多個 [FromBody]，你要的是「把需要的資料包成一個 request DTO」，例如你想同時帶 product + tags + categoryId，就做 CreateProductRequest { Product..., Tags..., CategoryId... }，一次反序列化即可


![5](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ModelBinding/5_2_overposting.png)


<!-- endtab -->

<!-- tab [FromHeader] – HTTP Header-->

案例：API 金鑰驗證
```bash
GET /secure/products
X-Api-Key: abc123xyz
```

**設定**
```csharp
[HttpGet("secure/products")]
public IActionResult GetSecureProducts([FromHeader(Name = "X-Api-Key")] string apiKey)
{
    if (apiKey != "abc123xyz") return Unauthorized();
    return Ok(new[] { "Secret Product A", "Secret Product B" });
}
``` 

因為金鑰、Token 這種敏感資訊不適合放在 URL 或 Body，每個請求都應該帶在 Header


## 注意項目

1) Key 需要「權限範圍」與「最小權限」，一把 key 能做越少事，外流時你要賠的就越少。區分 read-only、write、admin；甚至分到單一資源範圍（某店家只能查自己商品），避免一把 key 看全世界

2) 若跨第三方/高風險場景，API Key 可能不夠，API Key 本質上是「誰拿到誰能用」，遇到可重放、可轉傳的風險時要加簽名或改用 OAuth/JWT，更嚴謹的做法會用時間戳 + nonce + HMAC 簽名（避免重放），或 OAuth 2.0 的 token 機制（可短效、可撤銷、可 scope）


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ModelBinding/6_1_header.png)


<!-- endtab -->

<!-- tab [FromServices] – 依賴注入 (DI)-->

## 取得系統時間

**設定**
```csharp
public interface IClock { DateTime Now { get; } }
public class SystemClock : IClock { public DateTime Now => DateTime.Now; }


builder.Services.AddScoped<IClock, SystemClock>();


[HttpGet("time")]
public IActionResult GetTime([FromServices] IClock clock)
{
    return Ok(new { Now = clock.Now });
}
``` 

因為 IClock 不是從 HTTP Request 來的資料，而是來自應用程式內部的服務


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ModelBinding/7_fromservice.png)


<!-- endtab -->

<!-- tab 案例：複合來源-->

```bash
PUT /products/123?notify=true
Authorization: Bearer abctoken
``` 
```json
{
  "name": "iPhone 15 Pro",
  "price": 45000
}
```
```csharp
[HttpPut("products/{id}")]
public IActionResult Update(
    [FromRoute] int id,
    [FromQuery] bool notify,
    [FromHeader(Name = "Authorization")] string token,
    [FromBody] Product product)
{
    return Ok(new { id, notify, token, product });
}
```

## 注意事項

Authorization 不要用 [FromHeader] string token 自己比字串，正規做法是用 ASP.NET Core Authentication/Authorization（JWT Bearer、Policy、Roles/Scopes），讓框架處理驗證、過期、簽章、challenge；action 裡拿到的應該是 User 身分，不是 raw token


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ModelBinding/8_complete.p)


<!-- endtab -->

<!-- tab summary-->


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ModelBinding/9_final.png)


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/ModelBinding/modelbind.png)


<!-- endtab -->


{% endtabs %}