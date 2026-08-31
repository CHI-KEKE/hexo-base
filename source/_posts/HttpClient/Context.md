---
title: HttpContext 失效時，我們還能說什麼
date: 2024-06-27 16:56:05
categories: 穿越雲層的信使
top_img: https://i.imgur.com/hZWviQz.png
cover : https://i.imgur.com/hZWviQz.png
tags:
    - Http
toc:
toc_number:
comments :
---

那是一個午後，城市被陽光輕輕包裹著。我走過熟悉的街口，沒有目的地，只是順著記憶走。

咖啡廳的窗戶閃著熟悉的光，我下意識地望進去 —— 她，就坐在靠窗的位置。那個多年不見的身影，仍舊筆挺，肩上落著光，就像記憶裡的那樣。

我愣了一下，彷彿穿越了幾年的時間。當正要推開門的時候，門內走來一名男子，笑著坐在她對面，手中拿著兩杯咖啡。

她抬頭接過咖啡，對他微笑，那笑容熟悉卻已不再屬於你。

片刻後，一個小男孩蹦蹦跳跳地衝進店裡，喊著「爸爸！」男人張開手臂將他抱起，她則伸手輕拍孩子的頭髮。

你靜靜地站在門外，手輕輕從門把上放下。你沒有走進去，也沒有發出聲音。因為你知道，那個 context，早已不在了。

你轉身離開，步伐輕而穩，那封從未送出的請求，就此作廢。

<br/>

<font color=#D3D3D3 style="font-size: 15px;">&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;愛必須在對的 context 中發送。
</font>

<br/>

<font color=#D3D3D3 style="font-size: 15px;">&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;遲到的請求，即使格式正確，也只會收到 410 Gone 的錯誤碼。
</font>

<br/>
<br/>

![Image](https://i.imgur.com/kcz0cre.png)

```CSHARP

public IActionResult ConfessLove(HttpContext context)
{
    if (!context.Request.Query.ContainsKey("love"))
    {
        return NotFound("告白內容遺失");
    }

    if (DateTime.Now > GraduationDay)
    {
        return StatusCode(410, "告白已過時，不再受理");
    }

    return Ok("我也一直在等你說這句話");
}

```

我們以為遲到的請求，還有回應的可能；但在 HTTP 世界中，每個請求都仰賴一個尚存、完整、當下的 HttpContext。這不僅是情感的隱喻，更是技術的事實。

在 ASP.NET Core 中，每一次請求的來臨，都會被包裝成一個 HttpContext 物件，這個物件承載了請求的所有背景資訊，像是一個敘事的載體，讓系統可以「理解」並「對應」來者的需求。

讓我們從這個核心概念出發，進入 HttpContext 在實務中如何被應用的場景 —— 不再是情感，而是真正的程式碼。

「Context」這個詞的核心意涵，是提供一個背景與狀態，使我們能理解並處理眼前的資訊。在程式設計中，「上下文」常指某一時刻與事件相關的所有狀態資料與環境設定。

以 ASP.NET Core 為例，當一個 HTTP 請求送進我們的應用，Kestrel 會接收這個請求，並將其包裝為 HttpContext 物件，接著傳遞給中介軟體（Middleware）處理。

這個 HttpContext 就是「請求的上下文」，它承載著使用者發出的 Request 所有資訊：URL、QueryString、Headers、身分驗證資訊、甚至連線狀態。我們正是透過它，來理解「誰在說話」、「說了什麼」、「我們該如何回應」。


{% note info %}

🌀 Kestrel 是 ASP.NET Core 中的內建 Web 伺服器，負責處理 HTTP 請求並傳遞給應用程式內部。當一個請求進入時，HttpContext 就像是整段互動的封套，紀錄著整個來龍去脈，使程式得以理解這場短暫卻關鍵的對話。

{% endnote %}

<figure>
  <img src="https://i.imgur.com/itGQIdK.png" alt="Kestral 示意圖">
  <figcaption style="text-align: center;">Kestrel 示意圖</figcaption>
</figure>

<br/>

<figure>
  <img src="https://i.imgur.com/ATRZcs2.png" alt="官網 IIS 示意圖">
  <figcaption style="text-align: center;"> 微軟官方的 IIS 示意圖</figcaption>
</figure>


接著，我們來了解它可以讓我們做到那些事情吧!

# 實作情境

## ✔️ 一個來自付款的 Request，我們想遮罩包含信用卡資訊的 Header 並 Log 出來

首先，我們實作遮罩秘密 Header 用的擴充方法
```CSHARP
public static class StringExtension
{
	public static string MaskCenter(this string input, int visibleCharCount)
	{
		if (string.IsNullOrEmpty(input)) return input;

		visibleCharCount = Math.Max(0, visibleCharCount);
		if (input.Length <= visibleCharCount) return input;

		int leftVisible = visibleCharCount / 2;
		int rightVisible = visibleCharCount - leftVisible;
		int maskLength = input.Length - visibleCharCount;

		string left = input.Substring(0, leftVisible);
		string right = input.Substring(input.Length - rightVisible);
		return left + new string('*', maskLength) + right;
	}
}

```

接著實作 RequestLoggingMiddleware
```CSHARP

public class RequestLogMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestLogMiddleware> _logger;

    public RequestLogMiddleware(RequestDelegate next, ILogger<RequestLogMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task Invoke(HttpContext context)
    {
        var request = context.Request;

        _logger.LogInformation($"Request Method: {request.Method}, Request Path: {request.Path}, Request QueryString: {request.QueryString}");

        //// 遮罩特殊 Header
        string headerText = context.Request.Headers
            .Where(headerDic => headerDic.Key.StartsWith("X-", StringComparison.OrdinalIgnoreCase))
            .Select(headerDic => $"{Environment.NewLine}{headerDic.Key} : {headerDic.Value.ToString().MaskCenter(headerDic.Value.ToString().Length / 2)}")
            .Aggregate("", (current, next) => current + next);

        if (string.IsNullOrWhiteSpace(headerText) == false)
        {
            this._logger.LogInformation($"REQUEST HEADERS: {headerText}");
        }

        //// 遮罩特殊 Body 內容
        //// HttpContext.Request.Body 是一个一次性讀取的 Stream，因此我們需要保存它以便在處理完之後恢復它
        var bodyStream = context.Request.Body;
        var memoryStream = new MemoryStream();
        await bodyStream.CopyToAsync(memoryStream);
        memoryStream.Seek(0, SeekOrigin.Begin);

        var requestBodyText = new StreamReader(memoryStream).ReadToEnd();
        if (requestBodyText.ToLower().Contains("card"))
        {
            var bodys = requestBodyText.Split(Environment.NewLine);
            var recordItems = bodys.Select(body => body.ToLower().IndexOf("card", StringComparison.OrdinalIgnoreCase) < 0 ? body : body.MaskRight(body.Length * 2 / 3)).ToList();
            requestBodyText = string.Join(Environment.NewLine, recordItems);
        }

        if (string.IsNullOrWhiteSpace(requestBodyText) == false)
        {
            requestBodyText = $"{Environment.NewLine}{requestBodyText}";
            this._logger.LogInformation($"REQUEST BODY:{requestBodyText}");
        }

        memoryStream.Seek(0, SeekOrigin.Begin);
        context.Request.Body = memoryStream;

        //// 保存，後續可以避免再次讀取 context.Request.Body
        context.Items.Add("Body", requestBodyText);

        await _next(context);

        context.Request.Body = bodyStream;
    }
}

```

```csharp

//// 註冊
app.UseMiddleware<RequestLogMiddleware>();

```

測試
![Image](https://i.imgur.com/jih4J7U.png)


這樣我們就達到了把付款請求的 secret Header 做遮罩的目的

## ✔️ 在不同層級下取得 HttpContext

你可能會問，我想在管線以外的位置取用上下文做得到嗎 ?

答案是可以的!

### WebApi


Controller 本身繼承 ControllerBase，而它有提供 HttpContext

![Image](https://i.imgur.com/omDJSK3.png)

因此我們可以在 Controller 直接取用 HttpContext

![Image](https://i.imgur.com/GMhliKf.png)


### 自訂的 Service

自訂的 Class 中，若需要使用請求的上下文，需注入 HttpContextAccessor，在 ASP.NET Core 中，HttpContextAccessor 會註冊為 Singleton 的服務。整個應用程式的生命週期內，只有一個 HttpContextAccessor 實例被創建並使用。

這邊我們以實作 JWT 驗證機制為例，並且在自定義的 Service 取用 HttpContext.User.Identity.Name

1. 首先，我們先註冊 HttpContextAccessor

```CSHARP

//// 注入 HttpContextAccessor
builder.Services.AddHttpContextAccessor();

```

2. 我們自己定義了一個 UserService

```CSHARP

public class UserService : IUserService
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public UserService(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }
    public string GetUserName()
    {
        return _httpContextAccessor.HttpContext.User?.Identity?.Name;
    }
}


//// 記得注入框架
builder.Services.AddScoped<IUserService, UserService>();

```

3. Name 的值會來自解析 JWT Token 得到，所以我們要在管線中解析出來，首先我們要安裝套件來支援 JWT 的驗證機制

```POWERSHELL

dotnet add package System.IdentityModel.Tokens.Jwt
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer

```

4. 管線設定驗證
   
```CSHARP
var key = Encoding.ASCII.GetBytes("your_secret_key_here_longer_longer_longer_your_secret_key_here_longer_longer_longer");

builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.RequireHttpsMetadata = false;
    options.SaveToken = true;
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ValidIssuer = "your_issuer",
        ValidAudience = "your_audience",
        IssuerSigningKey = new SymmetricSecurityKey(key),
        NameClaimType = ClaimTypes.Name
    };
});



app.UseAuthentication();

```

5. 在 Controller 取得這個 Name 的資料

```CSHARP

[Authorize]
[Route("api/[controller]")]
[ApiController]
public class PaymentController : ControllerBase
{
    private readonly IUserService _userService;

    private readonly ILogger<PaymentController> _logger;

    public PaymentController(IUserService userService, ILogger<PaymentController> logger)
    {
        _userService = userService;
        _logger = logger;
    }
    [HttpPost]
    [Route("Pay/{payMethod}_{payChannel}/{tgCode}")]
    public object Pay(PaymentRequestEntity<IDictionary<string, object>> request, string payChannel, string payMethod, string tgCode)
    {
        var userName = _userService.GetUserName();
        if (userName == null)
        {
            return Unauthorized();
        }

        _logger.LogInformation($"User: {userName} is paying with {payChannel} {payMethod} for {request.Amount} {request.Currency}");

        string requestBody = JsonSerializer.Serialize(request);

        var headers = HttpHelper.GetHeaders(HttpContext);

        return new
        {
            Body = requestBody,
            Headers = headers,
            payChannel = payChannel,
            payMethod = payMethod,
            tgCode = tgCode
        };
    }
}

```

6. 我們寫一個簡單的測試發送帶有 JWT 的請求

```CSHARP

public static string GenerateJwtToken()
{
    var tokenHandler = new JwtSecurityTokenHandler();
    var key = Encoding.ASCII.GetBytes("your_secret_key_here_longer_longer_longer_your_secret_key_here_longer_longer_longer");
    var tokenDescriptor = new SecurityTokenDescriptor
    {
        Subject = new ClaimsIdentity(new[]
        {
            new Claim(ClaimTypes.Name, "今天是UserAllen")
        }),

        Expires = DateTime.UtcNow.AddHours(1),
        Issuer = "your_issuer",
        Audience = "your_audience",
        SigningCredentials = new SigningCredentials(new SymmetricSecurityKey(key), SecurityAlgorithms.HmacSha256Signature)
    };

    var token = tokenHandler.CreateToken(tokenDescriptor);
    return tokenHandler.WriteToken(token);
}

```

7. 發送 Request 測試

![Image](https://i.imgur.com/fwMvZ2O.png)


# ☘️ 結語

當我們談論 HttpContext，不只是談一段程式生命週期的資料結構，也是在提醒我們：程式與人一樣，需要活在「當下」。每一個請求，只有在當下那一刻才能被解析、被理解、被回應。錯過了，就無法重來。

在本文中，我們從一段情感隱喻出發，探索了 HttpContext 在 ASP.NET Core 中的角色與運作流程，從 Middleware 中的遮罩處理，到 Service 層如何透過 IHttpContextAccessor 取得使用者資訊，再到完整實作 JWT 驗證的範例，我們看見了一個 Web 應用如何圍繞這個「上下文容器」來構築對話的能力。

程式世界的請求，也如人生的關係，不只是發送而已，更需要「正確的時機」與「有效的上下文」。下一次你在處理 HttpContext 時，或許可以想起那句話：

<br/>

<font color=#D3D3D3 style="font-size: 15px;">&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;遲到的請求，即使格式正確，也只會收到 410 Gone 的錯誤碼。
</font>

<br/>



