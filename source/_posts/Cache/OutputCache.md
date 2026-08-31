---
title: OutputCache
date: 2024-05-04 12:04:10
categories: 落葉下的存檔
top_img: https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true
tags:
    - Cache
toc:
toc_number:
comments :
---

## 🍂 與其他快取的差異

最大的差異，是他快取的維度是整個 API 回傳的結果，但與 Redis 等獨立快取系統相比，OutputCache 的使用門檻較低，無需額外安裝與維護快取伺服器，只要設定好屬性即可立即套用，對中小型應用特別友善。

<br>
<br>

## 🍂 專案觀察

<br>

### Cache.Config

集中管理設定值

![Image](https://i.imgur.com/eZaMDBM.png)

<br>
<br>

##  🍂 設定

OutputCache 屬性，我們可以指定多項快取參數，其中的 Location 是非常關鍵的設定，它決定了快取資料的儲存位置，包含：

- Client 快取存在用戶端（例如瀏覽器）。
- Server 快取存在伺服器的記憶體中。
- Any 交由系統決定最合適的位置。

在使用 OutputCache 機制時，快取資料的仍受到一定限制，以避免對系統資源造成過大負擔。以下為主要的限制項目：

- SizeLimit
    指快取區域的總容量上限。當快取資料總量達到此上限時，若未進行資料逐出（Eviction），系統將不再快取任何新的回應。預設上限為 100 MB。

- MaximumBodySize
    指單一回應（Response Body）可被快取的最大容量。若回應內容超過此大小，系統將不進行快取。預設值為 64 MB。

- DefaultExpirationTimeSpan
    當快取政策（Policy）中未明確設定快取時間時，系統將自動套用預設的有效期限，預設為 60 秒。

<br>
<br>

##  🍂 實作 ( 以.NET Core 7 WebAPI 為例 )

[ASP.NET OutputCache 快取行為深入觀察](https://blog.darkthread.net/blog/aspnet-outputcache-experiment/)

### 1. 加入 Middleware

這邊設定有 3 種 Policy

- Expire : 快取時間
- Tag : 用在清除快取時，可以用 Tag 來識別與清除


```CSHARP
builder.Services.AddOutputCache(options =>
{
options.AddBasePolicy(builder => builder.Expire(TimeSpan.FromSeconds(10)).Tag("tag-all"));
    options.AddPolicy("Expire20", builder => builder.Expire(TimeSpan.FromSeconds(20)).Tag("tag-short"));
    options.AddPolicy("Expire_Min", builder => builder.Expire(TimeSpan.FromMinutes(1)).Tag("tag-long"));
});

app.UseOutputCache();

```

<br>

### 2. Attribute 參數設定

這邊設定了三個 Action

1. 快取 20 秒，依照 QueryKeys 執行快取
2. 第二個快取 1 分鐘，依照 QueryKeys 執行快取
3. 讓開發者依照 Tag 名稱清除快取

```CSHARP
[HttpGet]
[OutputCache(PolicyName = "Expire20",VaryByQueryKeys = new string[] { "category", "rating" })]
public string Get(string category,string rating)
{
    return $"DateTime :{DateTime.Now}, CacheForCategory :{category}, Rating : {rating}";
}

[HttpGet("GetLong")]
[OutputCache(PolicyName = "Expire_Min", VaryByQueryKeys = new string[] { "category", "rating" })]
public string GetLong(string category, string rating)
{
    return $"DateTime :{DateTime.Now}, CacheForCategory :{category}, Rating : {rating}";
}

[HttpDelete("cache/{tag}")]
public async Task DeleteCache(IOutputCacheStore cache, string tag)
{
    await cache.EvictByTagAsync(tag,default);
}
```


<br>

### 3. Demo

實測結果，成功快取資料
![Image](https://i.imgur.com/sUT2eCS.png)

<br>
<br>

##  🍂 HTTPS 轉址 vs OutputCache

[ASP.NET MVC 開發心得分享 (17)：OutputCache 帶來的問題](https://blog.miniasp.com/post/2010/03/28/ASPNET-MVC-Developer-Note-Part-17-Output-Cache-impact)

作者在做網站效能優化，加入了 SSL（也就是使用 HTTPS），但發現：

一加上 HTTPS，伺服器負擔變大了。原因是加密與解密的計算成本，但重點是，以前做好的 Output Caching 似乎也不再那麼順利了。因為 OutputCache 預設會把 HTTP 和 HTTPS 分開處理！


也就是說：

http://example.com/page
https://example.com/page

雖然看起來是同一頁，快取系統會當成兩個不同的頁面在處理！

以作者的案例而言，問題點在於，那些已經被 OutputCache 過的網頁只要一被快取住，就不會再執行任何程式了，ASP.NET 的 Request pipeline 會先執行 OutputCache 模組，只要一發現有輸出快取的項目就會直接回應給用戶端，也因此造成了以下狀況：

- 使用者第一次訪問：http://example.com/
- 而 HomeController.About Action 上套用了：[RequireHttps] 屬性，因此這個 Action 只能透過 HTTPS 訪問，否則 ASP.NET MVC 會自動 302 轉址到 HTTPS

- 使用者被自動轉址到：https://example.com/
- OutputCache 啟動，把「HTTP 請求的結果」快取起來（注意：此時快取的是 HTTP 回應）

所以下次使用者使用者下次又來訪問：http://example.com/

這時：

ASP.NET 不會再執行 Controller 中的程式碼
🚫 不會跑 RequireHttps 屬性
🚫 不會再自動轉址成 HTTPS

而是直接拿出「前次 HTTP 的快取結果」回應給使用者（因為 OutputCache 告訴它可以這樣做）
，使用者看到的不是 https://example.com/，而是舊的快取內容（可能是未加密的畫面），整個轉址邏輯被快取「繞過」了，所以就出現了「邏輯錯誤」，使用者應該被轉到 HTTPS，但卻沒被轉到

OutputCache 是在 HTTP 層級運作，它會快取整個 Response，但 RequireHttps 是在 ASP.NET MVC 層級才生效的檢查，一旦某次請求被快取下來，下次請求就不會再執行 MVC 檢查邏輯


### 解法

**解法 1：不要快取 HTTP 的回應（只快取 HTTPS）**

只允許 https://example.com/ 被快取，如果是 http://，不要存入快取（或自動轉向）
可使用快取條件判斷，例如：

```csharp
if (Request.IsSecureConnection)
{
    // 允許快取
}
```


<br>

**解法 2：在伺服器或 CDN 層級強制 HTTP 轉 HTTPS**

例如在 IIS 中設定 redirect 規則或使用 HTTPS Rewrite Module，自動將所有 HTTP 請求重導向

甚至可以在 web.config 加入：

<rewrite>
  <rules>
    <rule name="Redirect to HTTPS" stopProcessing="true">
      <match url="(.*)" />
      <conditions>
        <add input="{HTTPS}" pattern="off" ignoreCase="true" />
      </conditions>
      <action type="Redirect" url="https://{HTTP_HOST}/{R:1}" redirectType="Permanent" />
    </rule>
  </rules>
</rewrite>

**解法 3：改用全站 HTTPS**

現在幾乎所有現代網站都建議直接「整站 HTTPS」，搭配 CDN 或 TLS 加速器，讓成本降到最低


<br>
<br>

##  🍂 不適合的快取

### 需要授權的頁面被快取


```csharp
[Authorize]
[OutputCache(Duration = 60)] // ❌ 快取在授權頁面上
public ActionResult AdminDashboard()
{
    return View(AdminService.GetStats());
}
```

風險：

- 使用者 A 登入後快取了頁面
- 使用者 B 未登入也可能透過快取看到該頁
- 或登入不同帳號看到同樣內容，造成權限漏洞！

<br>
<br>

## 設計快取清除

```csharp
<add key="Dev.CacheRootPath" value="C:\Files\Cache\abc" />
<add key="Dev.OutputCacheRootPath" value="~/App_Data/Cache" />

```

這是設定檔（通常是 Web.config）的參數。
- Dev.CacheRootPath: 設定實體快取檔案儲存的資料夾（用於開發環境）。
- Dev.OutputCacheRootPath: 相對路徑，代表網站底下儲存快取的資料夾（通常用在 OutputCache 機制中）。

```csharp
public ActionResult CleanOutputCache(string path)
{
TraceLogManager traceLog = new TraceLogManager();
try
{
    if (!this.IsFromCompany(traceLog))
    {
        return new EmptyResult();
    }

    var urlToRemove = path;
    traceLog.Write("urlToRemove", urlToRemove);
    traceLog.Write("RemoveOutputCacheItem");
    this.Response.RemoveOutputCacheItem(urlToRemove);

    traceLog.End();
    return Content(@"path:" + path);
}
catch (Exception ex)
{
    traceLog.Write(ex);
    ExceptionLogManager.WriteLog(ex);
    throw;
}
}


<add name="WebAPI.NewsList" enabled="false" duration="60" varyByParam="shopId" varyByCustom="isOfficial;languageType" />

```

<br>
<br>

##  🍂 結語：快取的倒影

快取，就像一面湖水的倒影。當你走過湖邊時，看到的不是當下的山川，而是剛剛留下的映照。這面倒影能讓你不用每次都抬頭望向真實，節省力氣，快速得到答案。

但若你過度依賴，它也可能讓你誤以為「倒影就是現實」，忽略了背後世界的變化。這正是 OutputCache 的矛盾：它能帶來極致的速度與效率，卻也潛藏著錯置的風險與漏洞。

在開發者的世界裡，選擇是否使用快取，何時快取，快取多久，正如同在時間與真實之間拉鋸。過度保守，你會失去性能的優勢；過度冒進，你可能讓系統邏輯失效。

因此，快取不只是技術選項，它更像是一種「取捨的藝術」。懂得觀察場景、拿捏時機，才不會被快取反噬。就像湖邊散步的人，既欣賞倒影的美，也不會忘記抬頭看看真實的天空。