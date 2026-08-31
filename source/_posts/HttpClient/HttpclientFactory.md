---
title: HttpClientFactory
date: 2025-08-28 08:35:01
categories: 穿越雲層的信使
top_img: https://i.imgur.com/hZWviQz.png
cover : https://i.imgur.com/hZWviQz.png
tags:
    - Http
toc:
toc_number:
comments :
---

在浩瀚的網路之海，每一次 API 呼叫，就像派出一位無聲的信使。他背著我們的訊息，穿越纜線與雲層，去敲響遠方伺服器的大門。而我們該如何訓練這些信使？如何確保他們既不迷路，也不在途中耗盡力氣？這就是 HttpClientFactory 存在的理由。

![message](https://i.imgur.com/hZWviQz.png)

<br>

## 📨 A. 方法內各自 CreateClient（匿名信使）

每次出征時，我們都臨時徵召一位信使，並告訴他要去的目的地與身份證明。
信使們雖然每次看似「新生」，但其實都共享一條秘密通道（Handler Pool），因此不會因為頻繁占用多個港口。但缺點是每次都要重複交代路線與口令，若忘了，就會讓信使迷途。

```CSHARP
public void ConfigureServices(IServiceCollection services)
{
    services.AddHttpClient();
    services.AddRazorPages();
}

public class LineService : ILineService
{
    private readonly IHttpClientFactory _factory;

    public LineService(IHttpClientFactory factory) => _factory = factory;

    public async Task PushMessageAsync(LinePushMessage payload, string? token = null, CancellationToken ct = default)
    {
        var client = _factory.CreateClient(); // 匿名 client
        client.BaseAddress = new Uri("https://api.line.me");

        if (!string.IsNullOrWhiteSpace(token))
        {
            client.DefaultRequestHeaders.Authorization =
                new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);
        }

        var json = System.Text.Json.JsonSerializer.Serialize(payload, Jsons.Options);
        using var content = new StringContent(json, Encoding.UTF8, "application/json");

        // 統一使用同一路徑
        using var resp = await client.PostAsync("/v2/bot/message/push", content, ct);
        resp.EnsureSuccessStatusCode();
    }
}
```

適合小型/單一端點；設定只在這個方法用到。每次 CreateClient() 都是新 HttpClient 物件，但會共用 Handler（不會耗盡 Socket）。這符合 HttpClientFactory 的預期使用方式，雖然每次 CreateClient()，但其實背後是重用連線

但缺點是每次呼叫都重複設定，設定有可能散落在各處，造成浪費及遺漏或錯誤，如果有其他人也要呼叫 LINE API，就得複製貼上這段設定，共用設定無法集中管理，且後續要 mock HttpClient 的行為、或是想改為使用其他 API，就會發現這段寫死的邏輯不好抽換

<br>
<br>

## 📨 B. Named Client（在建構子取得固定實例）（信使營隊）

為了避免一再重複交代，我們開始建立「專屬營隊」。在 Program.cs 事先訓練好一批信使：告訴他們固定的駐紮地（BaseAddress）、出征規則（Accept、Timeout）。呼叫時只要點名 "LineBot"，就能立刻派出一名訓練有素的信使。

這適合長期跑同一路線的場景。但要小心：如果信使營隊只帶著一張「固定通行證」（DefaultRequestHeaders），那麼當任務需要切換不同身份時，就會出現問題。

```CSHARP
public class LineService : ILineService
{
    private readonly HttpClient _httpClient;

    public LineService(IHttpClientFactory factory)
    {
        // 取用集中設定好的 Named Client
        _httpClient = factory.CreateClient("LineBot");
    }

    public async Task PushMessageAsync(LinePushMessage payload, string? token = null, CancellationToken ct = default)
    {
        using var request = new HttpRequestMessage(HttpMethod.Post, "/v2/bot/message/push");

        if (!string.IsNullOrWhiteSpace(token))
        {
            // 動態 Token 放在「這次請求」的 Header，避免污染共享實例
            request.Headers.Authorization =
                new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);
        }

        var json = System.Text.Json.JsonSerializer.Serialize(payload, Jsons.Options);
        request.Content = new StringContent(json, Encoding.UTF8, "application/json");

        using var resp = await _httpClient.SendAsync(request, ct);
        resp.EnsureSuccessStatusCode();
    }
}
```

適合固定打同一個服務、Header/Token 不隨用戶切換。_httpClient 在建構時取一次；要注意不要把動態 Token 設在 DefaultRequestHeaders。這種做法一樣 HttpClient 已經由工廠管理好生命週期與設定，因此建構子中只是注入一個「受控且設定好」的實例


但缺點是 HttpClient 建在 constructor，若 service 是 Singleton 或 Scoped，會共用整個 client，這行只會執行一次，所以取得的 _httpClient 是 固定實例。要確認 Header 都來自參數，不是寫在 _httpClient.DefaultRequestHeaders 裡，也沒有根據使用者切換 token 或身分驗證的需求。


<br>
<br>

## 📨 C. Named Client + 方法內 CreateClient（動態 Header／最好用）（臨時加註口令）

這裡，我們依舊依賴「信使營隊」，但在每次出征前，會再臨時賦予他一張專屬令牌。如此一來，他既能沿用集中管理的訓練，又能隨任務需求攜帶不同的身份。

很多人會擔心：這樣不是每次都要生一個新的 HttpClient 嗎？事實上，這只是換了個外衣。真正昂貴的通道（Handler）早已共用。因此，這是最靈活、最安全的做法。

正如官方文件所言：

“You can create many HttpClient instances with minimal overhead by using IHttpClientFactory.”

```CSHARP
public class LineService : ILineService
{
    private readonly IHttpClientFactory _factory;

    public LineService(IHttpClientFactory factory) => _factory = factory;

    public async Task PushMessageAsync(LinePushMessage payload, string? token = null, CancellationToken ct = default)
    {
        var client = _factory.CreateClient("LineBot"); // 套用集中 BaseAddress/Accept 等

        if (!string.IsNullOrWhiteSpace(token))
        {
            client.DefaultRequestHeaders.Authorization =
                new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);
        }

        var json = System.Text.Json.JsonSerializer.Serialize(payload, Jsons.Options);
        using var content = new StringContent(json, Encoding.UTF8, "application/json");

        using var resp = await client.PostAsync("/v2/bot/message/push", content, ct);
        resp.EnsureSuccessStatusCode();
    }
}

```

每次呼叫都是同一組基礎設定，但可以安全加入不同 Token/身分。背後是每次從 Named Client 建新 HttpClient 物件，因此也共用 Handler、套用集中設定，再加上呼叫時的專屬 Header。可能我們會想，這樣不就每次 CreateClient() 都會建立一個新的 HttpClient 實例（物件），但這樣做完全沒問題，也不會導致效能問題或資源浪費，因為 HttpClient 本身是一個輕量級的包裝器，真正昂貴的是底層的 HttpMessageHandler，而 HttpClientFactory 就是專門用來共用與重用 handler 的！

✅ 微軟官方說明說明：

"The factory manages the lifetimes of the HttpMessageHandler instances to avoid socket exhaustion issues. You can create many HttpClient instances with minimal overhead by using IHttpClientFactory."


<br>
<br>

## 📨 D. Typed Client（強型別用戶端，封裝成類別）（專屬信使團）

到了這一步，我們不再只是訓練「某一營的信使」，而是打造一支 專屬信使團。這支團隊專門負責 LINE API 的任務，他們有獨立的徽章（LineBotClient 類別），出征口令與路徑都被寫進章程（方法內部），呼叫者不需要知道 /v2/bot/message/push 是什麼，只要下達「推送訊息」的指令，信使團便會替你完成一切。

這就是 Typed Client 的力量。

```CSHARP
// 集中設定：BaseAddress / Accept / Timeout / Polly 等都放這裡
services.AddHttpClient<LineBotClient>(client =>
{
    client.BaseAddress = new Uri("https://api.line.me");
    client.DefaultRequestHeaders.Accept.ParseAdd("application/json");
    // client.Timeout = TimeSpan.FromSeconds(10); // 需要時再設
})
// 可選：加入 Polly、重試/熔斷、或自訂 DelegatingHandler
// .AddPolicyHandler(retryPolicy)
// .AddHttpMessageHandler<LoggingHandler>()
;


public interface ILineBotClient
{
    Task PushMessageAsync(LinePushMessage payload, string token, CancellationToken ct = default);
}

public sealed class LineBotClient : ILineBotClient
{
    private readonly HttpClient _http;

    public LineBotClient(HttpClient http)
    {
        _http = http; // 由 AddHttpClient<LineBotClient> 注入，已套用集中設定
    }

    public async Task PushMessageAsync(LinePushMessage payload, string token, CancellationToken ct = default)
    {
        // 用「這次請求」的 Header 帶入動態 Token，避免污染共享實例
        using var req = new HttpRequestMessage(HttpMethod.Post, "/v2/bot/message/push");

        req.Headers.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);

        var json = System.Text.Json.JsonSerializer.Serialize(payload, Jsons.Options);
        req.Content = new StringContent(json, Encoding.UTF8, "application/json");

        using var resp = await _http.SendAsync(req, ct);
        resp.EnsureSuccessStatusCode();
    }
}

public class LineService : ILineService
{
    private readonly ILineBotClient _client;

    public LineService(ILineBotClient client)
    {
        _client = client;
    }

    public Task PushMessageAsync(LinePushMessage payload, string token, CancellationToken ct = default)
        => _client.PushMessageAsync(payload, token, ct);
}

```


<br>
<br>

## 📨 為什麼 Named Client 做不到 Typed Client？

你或許會問：「既然 Named Client 也能集中設定 BaseAddress，為什麼不直接用它？」

差別在於，Named Client 給你的只是一支「拿著地圖的信使」。路徑、序列化、授權，仍然得由呼叫端口頭交代：

有人想用這個 Named Client 時，他只會給你 HttpClient，後續的「抽象化、可測試性、介面穩定性」都還是你自己拼。

```CSHARP
// Program.cs
services.AddHttpClient("LineBot", c => c.BaseAddress = new Uri("https://api.line.me"));

// 呼叫端（控制器/服務層）
public class OrderService
{
    private readonly IHttpClientFactory _factory;
    public OrderService(IHttpClientFactory factory) => _factory = factory;

    public async Task NotifyAsync(LinePushMessage payload, string token, CancellationToken ct = default)
    {
        var http = _factory.CreateClient("LineBot");        // 只拿到 HttpClient
        http.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Bearer", token); // 呼叫端自己處理 Header

        var json = JsonSerializer.Serialize(payload);       // 呼叫端自己序列化
        using var body = new StringContent(json, Encoding.UTF8, "application/json");

        // 呼叫端必須知道 endpoint（字串分散各處，容易漂移）
        using var resp = await http.PostAsync("/v2/bot/message/push", body, ct);
        resp.EnsureSuccessStatusCode();
    }
}
```

Typed Client（把細節封裝起來，不外洩）
```CSHARP
// Program.cs
services.AddHttpClient<LineBotClient>(c => c.BaseAddress = new Uri("https://api.line.me"));

// 專用 SDK
public interface ILineBotClient { Task PushAsync(LinePushMessage payload, string token, CancellationToken ct); }
public sealed class LineBotClient : ILineBotClient
{
    private readonly HttpClient _http;
    public LineBotClient(HttpClient http) => _http = http;

    public async Task PushAsync(LinePushMessage payload, string token, CancellationToken ct = default)
    {
        using var req = new HttpRequestMessage(HttpMethod.Post, "/v2/bot/message/push");
        req.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
        req.Content = new StringContent(JsonSerializer.Serialize(payload), Encoding.UTF8, "application/json");
        using var resp = await _http.SendAsync(req, ct);
        resp.EnsureSuccessStatusCode();
    }
}

// 呼叫端（乾淨、只有語意）
public class OrderService
{
    private readonly ILineBotClient _line;
    public OrderService(ILineBotClient line) => _line = line;

    public Task NotifyAsync(LinePushMessage payload, string token, CancellationToken ct = default)
        => _line.PushAsync(payload, token, ct);
}
```


<br>
<br>


## 📨 結語

- 匿名信使：單次派遣，快速但零散。
- 信使營隊（Named Client）：集中訓練，但口令仍要自己補。
- 臨時加註口令（Named + CreateClient）：靈活且安全，能隨時換身份。
- 專屬信使團（Typed Client）：徹底抽象，呼叫端再也不必操心細節。

當我們理解這四種模式，就能根據需求選擇最合適的方式。無論是小隊快閃，還是長途征戰，HttpClientFactory 都能替我們養好信使，讓他們在雲層之上穩健飛行，把訊息準確無誤地送到遠方。