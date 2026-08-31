



## Controller 層 如何取得 Header

註冊服務
```csharp
services.AddHttpContextAccessor();
```


在 ASP.NET Core 裡面，每當一個 HTTP 請求進來（像是有人開你的網站、叫你的 API），系統就會產生一個「HttpContext」物件，裡面記錄了使用者是誰（包含登入帳號），請求的網址、Header、Body，回應的資料 Session、Cookie 等等，但是這個 HttpContext 不是你想用就用的，它只在「有請求進來」的時候才存在，而且它不是自己隨時隨地可以 new 的東西。


所以問題來了，如果你寫的程式碼在「服務（Service）」裡面，例如 Repository 或其他非 Controller 的地方，你就拿不到 HttpContext，因為這些不是直接處理 HTTP 請求的

AddHttpContextAccessor() 的本質就是：

✅ 註冊一個可以讓你在任何地方都「安全地取得目前的 HttpContext」的工具。

當你呼叫這行之後，系統會幫你註冊一個叫做 IHttpContextAccessor 的服務，這個服務有個屬性 .HttpContext 可以幫你拿到目前的請求內容


Controller 可以直接讀取

```csharp
[HttpPost]
[Route("Pay/{payMethod}_{payChannel}/{tgCode}")]
public async Task<object> Pay([FromBody] PaymentRequestEntity<IDictionary<string, object>> request, string payChannel, string payMethod, string tgCode)
{
    string requestBody = JsonSerializer.Serialize(request);
    var headers = HttpHelper.GetHeaders(HttpContext);
    return await this._dispatcher.Pay(requestBody, headers, payChannel, payMethod);
}
```

可以拆解成 ToImmutableDictionary (不想讓資料被改掉，或是我們 想要確保這份資料一旦建立後就不會改變，這時就會使用)
```csharp
return httpContext.Request.Headers
    .ToImmutableDictionary(x => x.Key, x => x.Value.ToString(), StringComparer.OrdinalIgnoreCase);
```



## 把金鑰或 Token 放在 HTTP Header 安全嗎

✔️ 如果你用的是 HTTPS（大部分網站現在都用了），那麼：

Header ✅ 是加密的

Body ✅ 是加密的

URL ❌ 雖然加密，但會被紀錄在：

伺服器 log

瀏覽器歷史記錄

中間的 Proxy

Analytics 工具

所以你如果把 Token 放在 URL 上，就像是把密碼寫在便利貼貼在額頭，走過去的人都會看到。

📦 Header 有幾個優點：

標準做法：API Gateway、防火牆、框架都支援

不會被誤記錄：不像 URL 那樣會進 log

被 HTTPS 保護：傳輸中是加密的

不影響資源請求邏輯：不像 Body 那樣受 HTTP Method 限制（GET 沒有 Body）

