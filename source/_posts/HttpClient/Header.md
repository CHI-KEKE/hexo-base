---
title: HttpClient - Header 都亂了，我還怎麼好好當人
date: 2024-05-21 23:28:05
categories: 穿越雲層的信使
top_img: https://i.imgur.com/hZWviQz.png
cover : https://i.imgur.com/hZWviQz.png
tags:
    - Http
toc:
toc_number:
comments :
---

我們人生中最尷尬的某些時刻，不是沉默，而是訊息傳錯
像是你在聊天室打了一句「我覺得主管頭好禿」，下一秒才發現你傳的是公司群組聊天室。
你懊悔、你重開手機假裝這一切會 Reset、數秒鐘後你開始思考辭職信該不該寫得文藝一點。

又或者你做為一個老師，把生活中的小抱怨傳到學生群組中

[來自 Thread 一篇貼文的情境支援](https://www.threads.net/@__ahshoe__/post/DBiIdLxSuca/media?xmt=AQGzNZ7Urqd1V-B3oDzxy3yBKSrr2QCOVEbkmVYSruG0VQ)


這種「錯頻溝通」其實不只在人生會發生，連程式也會。
我們寫程式時，也會很天真地以為，訊息只會送到我們想送的人那裡。
但如果發送 Request 時，將秘密資訊設定在 HttpClient.DefaultRequestHeaders 上，相當於把「寫給 A 的情書，貼在公共佈告欄」。


## 📲 為什麼會這樣？

HttpClient 是一個設計上建議共用的元件，也就是說，我們應該用一個 HttpClient instance 去處理多個 request，以節省資源、避免連線耗盡（像是 socket exhausted）。

但問題也就來了：既然大家都用同一支手機在傳訊息，那麼貼在手機上的「身份識別貼紙」（也就是 DefaultRequestHeaders 裡的 token）是不是有可能搞混？

是的。這正是為什麼我們不應該把 headers，特別是敏感資訊，設定在 HttpClient.DefaultRequestHeaders 上的原因。

在程式裡，我們常說 race condition 是多個執行緒「搶著做事」，結果誰也沒做好。
HttpClient.DefaultRequestHeaders 正是這樣的情況——它是一個共用資源，但我們卻常在多個 request 裡同時動它。

你想設定 A 的 token，B 那邊卻早一步把自己的 token 貼上去。
結果？A 的 request 被誤認成是 B，身分錯亂、驗證失敗、資安告急。
這時候你會希望你人生裡的錯頻訊息，只是發到錯聊天室那麼單純。


## 📲 用 HttpRequestMessage，讓每則訊息「對頻」

更好的做法，是為每個 request 建立自己的 HttpRequestMessage，在上面設定專屬的 headers。
這就像是每個人都用自己的手機發訊息，帶著自己的身份、語氣與內容，自然不會錯頻，也不會讓 B 收到原本寫給 A 的悄悄話。

使用 SendAsync 方法，可以讓我們完整控制每一個 request 的細節，包括：

- 指定 HTTP method（像是 PATCH、OPTIONS 等少見但必要的動詞）

- 自訂 request body 與 headers

- 設定 HTTP version、timeout、proxy 等進階屬性

這不只是技術上的自由度，更是讓系統溝通更「準確」的關鍵。



## 關於 DefaultHeader

DefaultRequestHeaders 是一個共用的集合，這個集合會應用到之後所有的 request 上。你改了這個集合，會影響 之後的所有 request
所以，如果你這樣寫：

```CSHARP

// 執行緒 A
client.DefaultRequestHeaders.Add("X-User", "Alice");

// 執行緒 B（在幾毫秒後執行）
client.DefaultRequestHeaders.Remove("X-User");
client.DefaultRequestHeaders.Add("X-User", "Bob");

// 接著兩個執行緒各自發出 request
client.GetAsync("https://example.com/api");

```

結果很可能是：

兩個 request 都帶 Bob，或是某個 request 一下帶 Alice、一下帶 Bob，或更糟：Header 被重複加入或變成錯誤的格式（取決於 timing），這就是所謂的 Data Race（資料競爭）：資料本身 thread-safe，但邏輯上你改錯了時間點，造成結果錯亂。


因此設計應用發送請求時可以設定成這樣
```csharp

using (var httpRequestMessage = new HttpRequestMessage(HttpMethod.Get, "http://path/to/wherever"))
{
    httpRequestMessage.Headers.Authorization = new AuthenticationHeaderValue("Bearer", "TheToken");

    using (var httpResponseMessage = httpClient.SendAsync(httpRequestMessage))
    {
        // ...
    }
}

```

## 關於身分驗證 : Bearer ? Basic ? Token-based authentication ?

接著，這邊也想提一點，關於放置在 Header 的驗證機制，常把 Token 帶到 Authorization Header 上

### 🪙 Token 的意義是甚麼呢 ?

Wiki 說

{% note info %}
Tokenization, when applied to data security, is the process of substituting a sensitive data element with a non-sensitive equivalent
{% endnote %}

也就是 用一個「替代物」來保護敏感資料的過程

![Image](https://i.imgur.com/sq34zzN.png)

又或者更容易理解的例子

病歷記錄：
- 真實資料：Allen, A123456789, 中二病重度患者
- 系統顯示：病患 T12345, TOK987654321, Chuunibyou

你拿著病歷一般人是看不懂得，只有院方可以知道

![Image](https://i.imgur.com/bHi3w1b.png)

### 🪙 Tokenize 的方式

而這個保護的方式就有很多種，由於現在許多 API 驗證機制不想存大量的狀態資訊在 Server 端 (可能偏好 stateless)，例如 : Microsoft Graph API、Google Cloud API...
<br>

在串接時常會看到 Bearer Token 或偶爾看到 Basic Token，他們其實就是把敏感資訊，打成看起來像亂碼的東西做傳輸，再從接收方解碼識別

### 🪙 Beaer Token 與 Basic Token 的差異是什麼?

Basic  Token 是透過 base64 做編碼，其實他是不嚴謹的，因為我們任何人，只要在網路上找一個 Base 64 解碼器(屬於公開資訊) 就可以知道真實內容，攻擊者可以輕易的解碼這段資訊，就像語言翻譯機一樣

而 Bearer Token 呢?

它是經過 OAuth 2.0 的流程產出的 Token，流程較複雜且更安全，而且他是會經過 Hash 的!

### 🪙 經過 Hash 是甚麼意思 ?

雜湊函數 ( Hash ) 可以將一組隨機長度的字串轉換成一組固定長度的雜湊值，雜湊函數有兩個重要的特性:

- 不同的字串不會產生相同的雜湊值 (唯一性)
- 雜湊值不可逆向回推原始資料 (不可逆)

所以，假設有心人士取得這段雜湊過後的內容他也不能幹嘛，但自己人只要將相同得機密作一樣的函數處理就可以藉由比對雜湊值來確認資料正確性

[淺談 Authentication 中集：Token-based authentication](https://vicxu.medium.com/%E6%B7%BA%E8%AB%87-authentication-%E4%B8%AD%E9%9B%86-token-based-authentication-90139fbcb897)

### 🪙 實作練習

Basic Token 製作
```csharp

	var byteArray = Encoding.ASCII.GetBytes("allen:username");
	var auth = new AuthenticationHeaderValue("Basic", Convert.ToBase64String(byteArray));
	auth.Dump();

    // Parameter YWxsZW46dXNlcm5hbWU=
    // Scheme Basic

```

Bearer Token
```csharp

var signingKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("allen_is_a_very_long_secret_key_for_hmac_sha256"));
var tokenHandler = new JwtSecurityTokenHandler();
var claims = new[]
{
    new Claim(ClaimTypes.NameIdentifier, "user_id"),
    new Claim(ClaimTypes.Name, "username")
};

var tokenDescriptor = new SecurityTokenDescriptor
{
    Subject = new ClaimsIdentity(claims),
    Expires = DateTime.UtcNow.AddHours(1),
    SigningCredentials = new SigningCredentials(signingKey, SecurityAlgorithms.HmacSha256Signature)
};

var token = tokenHandler.CreateToken(tokenDescriptor);
var bearerToken = tokenHandler.WriteToken(token);
var auth = new AuthenticationHeaderValue("Bearer", bearerToken);
auth.Dump();

// Parameter : eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1laWQiOiJ1c2VyX2lkIiwidW5pcXVlX25hbWUiOiJ1c2VybmFtZSIsIm5iZiI6MTcxNjYxMDY5MCwiZXhwIjoxNzE2NjE0MjkwLCJpYXQiOjE3MTY2MTA2OTB9.FhLuXQ82HuzoK2gAuoS6JQZNzt8FEr41a__RkTEyqU8

// Scheme : Bearer

```


# ☘️ 結語

有些 bug，不是寫錯程式，而是你太相信世界會照你預期的方式運作。
你以為加在 HttpClient 共用 Header 也都可以共用? 這出發點是好的 但建議是最好不要出發
這就像你以為自己講悄悄話，結果喇叭根本開全場，還連到藍芽音響。

所以記住一句話：
共用的東西，就不要放私人資訊。
DefaultRequestHeaders 是開放空間，不是你專屬的小信箱。

要講機密的話，就自己開一封 HttpRequestMessage，
愛誰就寫給誰，不要寄錯、不留誤會。

寫程式像生活，不是你不夠認真，而是你沒看清楚訊息是往哪裡飛。