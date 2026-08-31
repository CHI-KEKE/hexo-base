
---
title: Refit
date: 2025-08-29 08:35:01
categories: 穿越雲層的信使
top_img: https://i.imgur.com/hZWviQz.png
cover : https://i.imgur.com/hZWviQz.png
tags:
    - Http
toc:
toc_number:
comments :
---


在古老的時代，國王要把詔令送往四方，往往需要派出一位信使。但這位信使總得記住每一段路線、每一條小徑，甚至還要自己翻譯當地的語言。每一次派遣，不僅耗費心力，也總有出錯的風險。

程式世界裡的我們，也常常面臨相同的窘境。每次要呼叫第三方 API，就得重複 HttpClient、手動處理序列化與反序列化，還要加上各種 Retry、Logging。久而久之，這些細節就像一張張散落的地圖，讓信使負擔沈重。

而 Refit 的出現，就像給我們一套「信使專屬的傳令書」。只要定義好任務（Interface），信使便能自動遵循指令，把訊息送達遠方，並且用我們熟悉的語言回報結果。從此，我們不必再煩惱翻譯、路線和格式的細節，只需專注於該傳遞的訊息本身。


他讓我們能夠以 Interface 來定義 怎麼串接第三方 API 。

所以我們不需要直接使用 HTTPClient，而是定義一個 Interface，Refit 會將 Interface 的方法包裝起來，處理 HTTP 請求並將 Response 數據序列化為 Interface 中指定的類型，還可以組上自製的 HttpMessageHandler 以及處理 Retry 機制

![Image](https://i.imgur.com/hkB3vIf.png)

<br>

## 📨 套件

Refit.HttpClientFactory
![Image](https://i.imgur.com/pb0dGwS.png)
DependencyInjection(跟HttpClient無關，純粹Demo Code會用到)
![Image](https://i.imgur.com/GBb5WjM.png)

<br>
<br>

## 📨 設定 Interface
```csharp

public interface ITwoCTwoPHttpClient
{
    [Post("/4.3/paymentToken")]
    Task<CreatePaymentRequestResponseEntity> CreatePaymentMethodAsync(EncryptedCreatePaymentRequestEntity body);
}

```


<br>
<br>

## 📨 註冊、設定
```csharp

public class TwoCTwoPModule : NineYi.Extensions.DependencyInjection.Module.IModule
{
    public void Register(IServiceCollection services)
    {
        services.AddPaymentMiddlewarePlugins<TwoCTwoPPlugin>();

        int retryCount = 3;
        int retryInterval = 2000;

        //// 建立Interface後注入，並且在這裡設定HttpClient
        services.AddRefitClient<ITwoCTwoPHttpClient>()
            .ConfigureHttpClient(client => client.BaseAddress = new Uri("https://sandbox-pgw.2c2p.com/payment"))
            .AddHttpMessageHandler<RawResponseMessageLoggingDelegatingHandler>()
            .AddTransientHttpErrorPolicy(builder => builder.WaitAndRetryAsync(retryCount, (_) => TimeSpan.FromMilliseconds(retryInterval)));
    }
}

```


<br>
<br>

## 📨 調用
```csharp

public class TwoCTwoPPlugin
{
    private readonly ITwoCTwoPHttpClient _twoCtwoPHttpClient;

    public TwoCTwoPPlugin(ITwoCTwoPHttpClient twoCtwoPHttpClient)
    {
        this._twoCtwoPHttpClient = twoCtwoPHttpClient;
    }

    public async Task<PaymentResponseEntity<TwoCTwoPCreatePaymentResponseExtendInfo>> Pay(PaymentRequestEntity<TwoCTwoPCreatePaymentRequestExtendInfo> request, IDictionary<string, string> headers, string payMethod)
    {
        //// 不用手動Deserialize
        var twoCtowPayResponse = await _twoCtwoPHttpClient.CreatePaymentMethodAsync(encryptedPayload);
    }
} 

```


使用 Refit 後，我們不用自己寫序列化 & 反序列化，在 API Interface 定義型別，還確保型別安全!


<br>
<br>

## 📨 結語

當我們擁有了 Refit，就等於給信使一雙更聰慧的眼睛。他不再需要每次出發前都重新背誦長長的路線圖，也不必在歸來時，手動將陌生的符號轉換成我們能懂的語言。我們只需用 Interface 說一句：「去完成這個任務」，他便能穿越雲層，帶著正確的格式與安全的型別，精準送達。

在軟體開發的旅程裡，這樣的工具就像一位可靠的隨行軍師，讓我們把心力放在策略與願景上，而不是無止盡的瑣碎細節。