---
title: Plugin + Factory + Strategy 遊戲機與卡帶
date: 2025-08-16 11:00:03
categories: 設計図鑑物語
top_img: https://i.imgur.com/LF7L1Ws.png
cover : https://i.imgur.com/LF7L1Ws.png
tags:
    - Plugin Pattern（外掛模式）x Factory Method Pattern（工廠方法）x Strategy Pattern（策略模式）
toc:
toc_number:
comments :
---

{% tabs cartridge %}

![plugin](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Design_Pattern/game_catridge/start.png)


<!-- tab payment-->


在開發金流或第三方整合系統時，一定會預期到金流是無止盡在擴充的，今天要支援 **LinePay**，下個月要加上 **ApplePay**，後天客戶希望有 **GooglePay**、**PayPal**

如果每增加一種支付方式都要打開主程式去修改，就好比一台遊戲機的遊戲被寫死在機器裡，每次想玩新遊戲，就得整個流程要再改一次，想想就覺得很可怕


![surgery](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Design_Pattern/game_catridge/surgery.png)


好的設計主軸是 **遊戲機本身保持穩定，玩家只要插上不同的卡帶，就能享受不同的遊戲**

必須要接受支付方式一定會一直增加，LinePay、ApplePay、GooglePay 不會是最後一個，只會是個開始，因此把「會變的部分」獨立出來，不同支付流程、API、驗簽方式，我們把它**全部抽成「外掛模組（Plugin）」**，而主程式只負責呼叫介面，它不關心甚麼 Pay，只知道「你是一個 Payment Plugin」，並且**透過工廠（Factory）拿對卡帶，根據設定或參數，決定要載入哪一個 Plugin 實作**，最後採**用策略模式確保「同一個入口，不同行為」，每個 Plugin 實作同一組介面，但內部流程完全不同**


![strategy](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Design_Pattern/game_catridge/three_good.png)


<!-- endtab -->

<!-- tab Plugin Pattern（外掛模式）-->


主程式就像一台遊戲機，裡面不必寫死所有的邏輯，只要留好插槽，外部就能依需求插入不同的「遊戲卡帶」（Plugin），關鍵不是「怎麼 new 出物件」，而是讓主程式永遠只認得「規格」，而不是「實作細節」，只要規格不變，裡面換多少種卡帶，主程式都不用緊張

- `IPayable<TRequest, TResponse>`：定義插槽的規格（必須符合這個介面，才能順利插上去）。  
- `LinePayPlugin`, `ApplePayPlugin`：就像是不同的遊戲卡帶，各自帶有不同的內容與玩法。  

這樣設計後，當要新增一個新的支付方式，就像多插入一片新的卡帶，主程式不用改動，仍然可以正常運作。  


```CSHARP
static object ResolvePlugin(string pluginKey)
{
    return pluginKey switch
    {
        "LINEPAY" => new LinePayPlugin(),
        "APPLEPAY" => new ApplePayPlugin(),
        _ => throw new Exception("Unknown plugin")
    };
}
```

## 🪵 介面：插槽的規格

所有卡帶都必須符合這個插槽規格，否則遊戲機無法讀取


![interface](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Design_Pattern/game_catridge/interface.png)


```CSHARP
public interface IPayable<TRequest, TResponse>
{
    Task<TResponse> Pay(TRequest request, Dictionary<string, string> headers, string method);
}
```

先定義插槽規格 `IPayable<TRequest, TResponse>`，這個介面就是「卡帶插槽尺寸」，只要符合這個介面，就保證主程式「插得進去、用得起來」，每個支付方式各自實作這個介面 `LinePayPlugin`、`ApplePayPlugin`，各做各的流程、API 呼叫、簽章驗證，而主程式完全不關心 Plugin 裡面怎麼實作，它只知道「你是一個 IPayable，我就可以用你」，接著透過 ResolvePlugin 決定要用哪一片卡帶，並且根據 pluginKey，回傳對應的 Plugin 實例，未來新增支付方式時，只會動到「外面」

真實應用中，可能使用 IServiceProvider.GetService(pluginType) 來注入 Plugin，但其實不論 switch / dictionary / reflection，只要主程式沒有直接依賴 LinePayPlugin，換成 DI、設定檔、甚至遠端載入都只是實作細節

使用 `IServiceProvider.GetService`的點是，當 Plugin 開始需要「別的服務」，手動 new 就會失控，因為真實 Plugin 會依賴 HttpClient、Logger、Config，而 DI 讓 Plugin 好測試、好 mock、不用自己管理生命週期


另外，這邊用到泛型 <TRequest, TResponse> ，它的價值就是**型別即文件**，錯用時編譯器會先幫你擋下來，不同支付方式請求格式不同，用泛型可以避免大量 casting、提早在編譯期發現錯誤，比起 runtime exception，便宜太多了


![interface2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Design_Pattern/game_catridge/linepay_applepay_catridge.png)


而不同遊戲，自然會有各自的資料格式

```CSHARP
public class LinePayRequest { public string OrderId { get; set; } }
public class LinePayResponse { public string Result { get; set; } }
public class ApplePayRequest { public string Token { get; set; } }
public class ApplePayResponse { public string Status { get; set; } }

```

<!-- endtab -->

<!-- tab 卡帶實作-->


這裡有兩片卡帶，分別是 LinePay 與 ApplePay

```CSHARP
public class LinePayPlugin : IPayable<LinePayRequest, LinePayResponse>
{
    public async Task<LinePayResponse> Pay(LinePayRequest request, Dictionary<string, string> headers, string method)
    {
        Console.WriteLine($"[LinePay] OrderId: {request.OrderId}, Method: {method}");
        await Task.Delay(100);
        return new LinePayResponse { Result = "LinePay Success" };
    }
}

public class ApplePayPlugin : IPayable<ApplePayRequest, ApplePayResponse>
{
    public async Task<ApplePayResponse> Pay(ApplePayRequest request, Dictionary<string, string> headers, string method)
    {
        Console.WriteLine($"[ApplePay] Token: {request.Token}, Method: {method}");
        await Task.Delay(100);
        return new ApplePayResponse { Status = "ApplePay Success" };
    }
}

```


![applepay_linepay](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Design_Pattern/game_catridge/linepay_applepay_catridge.png)


<!-- endtab -->

<!-- tab Factory Method Pattern（卡帶管理員模式）-->


Plugin 很多，但資料進來的時候要怎麼挑選？這時候就交給「工廠」來統一建立與管理


![factory](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Design_Pattern/game_catridge/factory.png)


當「要用哪個物件」常改、而且「怎麼建它」越來越麻煩時，把建立過程收進工廠，才能避免主程式變成一坨分岔到爆的組裝區。我們不想讓主程式知道「怎麼 new 出一個物件」的細節，那就把 "怎麼建立的" 封裝在 Factory 裡，主程式只需給條件。我們「封裝了變化（encapsulate variation）」讓物件的建立方式隱藏起來，這樣如果將來物件建立方式要變，只有工廠改就好，不影響使用端


![factory2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Design_Pattern/game_catridge/factory_encap_variance.png)


遊戲機能插的卡帶越來越多，我們需要一個「卡帶管理員」來幫忙分類、快取。這就是工廠方法的角色。這樣一來，遊戲機只要交代「我要玩哪一片卡帶」，工廠就會把卡帶準備好

```CSHARP
static ConcurrentDictionary<string, PluginMetadata> pluginMetadataCache = new();

static PluginMetadata GetOrCreatePluginMetadata(string key, Func<PluginMetadata> factory)
{
    return pluginMetadataCache.GetOrAdd(key, _ => factory());
}

PluginMetadata metadata = GetOrCreatePluginMetadata(payTypeKey, () =>
{
    Type requestType = payTypeKey == "LINEPAY" ? typeof(LinePayRequest) : typeof(ApplePayRequest);
    Type responseType = payTypeKey == "LINEPAY" ? typeof(LinePayResponse) : typeof(ApplePayResponse);

    Type interfaceType = typeof(IPayable<,>).MakeGenericType(requestType, responseType);
    MethodInfo method = interfaceType.GetMethod("Pay");

    return new PluginMetadata(interfaceType, requestType, method);
});
```

我們準備了一個快取 `pluginMetadataCache`，做了快取可以把反射集中在第一次

- key：payTypeKey（例如 LINEPAY / APPLEPAY）
- value：PluginMetadata（你把「找 Plugin 需要的線索」包成一包）

它提供一個 Get-or-Create 的入口 `GetOrCreatePluginMetadata`，先查快取有沒有，沒有才用 factory() 建一份新的
，之後就直接回傳快取的結果，我們用 payTypeKey 決定 request/response 的 Type

- LINEPAY → LinePayRequest / LinePayResponse
- APPLEPAY → ApplePayRequest / ApplePayResponse
...


![cache](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Design_Pattern/game_catridge/metadata_cache.png)


這一步是在「把條件轉成型別資訊」，並且我們用反射拼出對應的介面型別 `IPayable<request,response>` ( `typeof(IPayable<,>).MakeGenericType(requestType, responseType)` ) 這是在「把插槽規格具體化成某個封閉泛型」，接著從介面上抓出 Pay 方法資訊 ( `interfaceType.GetMethod("Pay")`)，之後可能會用它做動態呼叫或驗證，以上資訊被封裝成  `PluginMetadata` 丟進快取，之後相同 key 進來，就不用再反射、再判斷一次


<!-- endtab -->

<!-- tab Reflection 遊戲機的讀卡頭-->


在我們的設計裡，即使遊戲機不知道「卡帶裡面」具體的程式碼，只要知道它一定有 Pay 方法，就能透過讀卡頭（Reflection）去呼叫正確的內容

`MethodInfo.Invoke(...)` → 啟動遊戲（執行 Pay）
`MakeGenericType(...)` → 動態生成正確型別


這就是「延遲決策（Defer to Runtime）」的價值


![reflection](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Design_Pattern/game_catridge/reflection.png)


<!-- endtab -->


<!-- tab 執行遊戲-->


到此我們已經準備好 Plugin 與實作細節，現在主程式只靠 metadata + key，就能把一筆未知支付請求安全地跑完!

```CSHARP

// 反序列化 request data
object requestData = JsonSerializer.Deserialize(jsonRequest, metadata.RequestEntityType);

// 執行 Plugin 的 Pay 方法
object plugin = ResolvePlugin(payTypeKey); // 拿到 plugin instance
object[] parameters = new object[] { requestData, new Dictionary<string, string>(), "PayMethod" };
Task task = (Task)metadata.MethodInfo.Invoke(plugin, parameters);
await task;

// 取得結果
PropertyInfo resultProp = task.GetType().GetProperty("Result");
object result = resultProp.GetValue(task);
Console.WriteLine($"🔁 Plugin 執行結果: {JsonSerializer.Serialize(result)}");
```

jsonRequest 是外部來的原始資料，真正關鍵的是 metadata.RequestEntityType，主程式不用知道這是 LinePayRequest 還是 ApplePayRequest，metadata 已經幫你決定「該反序列化成誰」，這一步是在：把純 JSON 轉成「這個 Plugin 吃得懂的物件」

接著可以看到 key 可能 = LINEPAY / APPLEPAY，並且回來的是某個實作了 IPayable<,> 的 Plugin 主程式，這裡不 cast、不判斷型別、不知道內部邏輯，這一步在把「我要玩哪片卡帶」交給卡帶管理系統

在準備呼叫 Pay 所需的參數時，參數順序與內容必須符合介面定義，這裡其實是在實作一個隱性的呼叫契約，反射要能跑，參數規格一定要被嚴格控制


在用 `MethodInfo` 動態執行 Pay 時，MethodInfo 來自 Factory 階段，真正執行的是 Plugin 內部自己的 Pay 邏輯，他回傳的是 `Task<TResponse>`

<!-- endtab -->

<!-- tab flow-->


![arch_views](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Design_Pattern/game_catridge/arch_view.png)


![flow](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Design_Pattern/game_catridge/flow.png)


![cycle](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Design_Pattern/game_catridge/three_cycle.png)


<!-- endtab -->

<!-- tab sumary-->


![four](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Design_Pattern/game_catridge/four.png)


![summary](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Design_Pattern/game_catridge/summary.png)



<!-- endtab -->

{% endtabs %}