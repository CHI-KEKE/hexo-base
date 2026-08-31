---
title: shopping-cart
date: 2026-03-15 08:25:03
categories: 落葉下的存檔
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart/1_landing.png
tags:
    - Cache
    - Redis
toc:
toc_number:
comments :
---


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart/1_landing.png)


{% tabs shopping-cart%}


<!-- tab 三層 Key（shopId / memberId / uniqueKey）-->


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart/3_shop_member_uni.png)


## 第一層：shopId

**Key 格式：** `Core:PartialCheckoutData-20230410:{shopId}:...`

**防護：多租戶（Multi-tenant）**

91APP 服務數千個商店，每個商店共用同一套程式和同一個 Redis。
若沒有 `shopId` 隔離，商店 A（`ShopId=100`）的結帳資料，理論上可以被商店 B（`ShopId=200`）的程式讀到。

---

## 第二層：memberId

**Key 格式：** `Core:PartialCheckoutData-20230410:{shopId}:{memberId}:...`

**防護：跨用戶資料洩漏**

同一個商店的不同會員，結帳資料要絕對隔離。
會員 A（`MemberId=12345`）的收件人、信用卡資料，絕不能被會員 B（`MemberId=67890`）讀到。

---

## 第三層：checkoutUniqueKey

**Key 格式：** `Core:PartialCheckoutData-20230410:{shopId}:{memberId}:{UUID}`

**防護：同一用戶的多個並行結帳 Session**

同一個人同時開兩個結帳分頁，各自有不同的 UUID，操作互不影響。

---

## ICoreCacheService 的介面設計也體現了這個分層

```csharp
// 有 memberId 維度（個人資料）
Task SaveAsync<T>(T data, CoreCacheKeyEnum key, long shopId, long memberId, params string[] args);

// 沒有 memberId 維度（商店維度，如 AntiForgeryToken）
Task SaveAsync<T>(T data, CoreCacheKeyEnum key, long shopId, params string[] args);
// AntiForgeryToken 是防 CSRF，屬商店+請求維度，不需要 memberId
```

---

## 實際 Key 對照

| 情境 | Redis Key |
|------|-----------|
| 商店 10416，會員 39326184，第一個結帳分頁 | `Core:PartialCheckoutData-20230410:10416:39326184:a1b2-c3d4-...` |
| 商店 10416，會員 39326184，第二個結帳分頁（同時開著） | `Core:PartialCheckoutData-20230410:10416:39326184:e5f6-g7h8-...` |
| 商店 10416，不同會員 99999999 | `Core:PartialCheckoutData-20230410:10416:99999999:x9y0-z1a2-...` |
| AntiForgeryToken（無 memberId） | `Core:AntiForgeryToken-20230821:10416:{Guid-Token-Value}` |

三層都必須正確才能命中 Cache，單一層錯誤就找不到資料。這個設計確保了電商系統中最敏感的結帳個資，絕對不會洩漏給其他使用者或商店。


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart/3_why_need_3_layers.png)



<!-- endtab -->


<!-- tab Transformer -->

## 存入 Redis（C# → Redis）

由 `PartialCheckoutDataHashTransformer.ToHash()` 負責把 C# 物件拆開成一個 Dictionary：

```csharp
// PartialCheckoutDataHashTransformer.cs
public IDictionary<string, object> ToHash(PartialCheckoutEntity data)
{
    var dictionary = new Dictionary<string, object?>
    {
        // nameof() 取屬性名稱當作 field 名，型別安全、改名時編譯器報錯
        [nameof(data.Member)]                              = data.Member,
        [nameof(data.InvoiceIssuanceList)]                 = data.InvoiceIssuanceList,
        [nameof(data.UserClientTrack)]                     = data.UserClientTrack,
        [nameof(data.TemperatureResponseItems)]            = data.TemperatureResponseItems,
        [nameof(data.TemperatureSalepageList)]             = data.TemperatureSalepageList,
        [nameof(data.SelectedShippingAreaId)]              = data.SelectedShippingAreaId,
        [nameof(data.SelectedDesignatePaymentPromotionId)] = data.SelectedDesignatePaymentPromotionId,
        [nameof(data.ShoppingDateTimeUtc)]                 = data.ShoppingDateTimeUtc.ToString(...)
    };
    // ...
}
```

`nameof(data.Member)` 就是字串 `"Member"`。這個 Dictionary 送進 Redis 後，實際樣貌如下（**所有 value 都是字串**）：

```bash
HGETALL Core:PartialCheckoutData-...:

"Member"                              → "{\"CellPhone\":\"0912345678\",...}"
"InvoiceIssuanceList"                 → "[{\"CarrierTypeDef\":\"ER0027\",...}]"
"UserClientTrack"                     → "{\"Channel\":\"Web\",...}"
"TemperatureSalepageList"             → "[{\"EshopId\":\"...\",\"Qty\":1,...}]"
"TemperatureResponseItems"            → "[]"
"SelectedShippingAreaId"              → "1"
"SelectedDesignatePaymentPromotionId" → "0"
"ShoppingDateTimeUtc"                 → "03/15/2026 01:00:00"
```

> Redis 不知道 `"Member"` 是什麼型別，它只知道那是一串 JSON 字串，**型別由程式碼自己負責。**

---

## 從 Redis 讀回來（Redis → C#）

由 `PartialCheckoutDataHashTransformer.ToType()` 負責把 Redis 回傳的字串 Dictionary 組裝回 C# 物件：

```csharp
// PartialCheckoutDataHashTransformer.cs
public PartialCheckoutEntity ToType(Dictionary<string, string> redisHash)
{
    var partialCheckoutEntity = new PartialCheckoutEntity
    {
        // ExtractHashValue 做兩件事：
        // 1. 用 Expression 取得欄位名 "Member"
        // 2. 把那個 JSON 字串反序列化成 CheckoutMemberEntity
        Member = redisHash.ExtractHashValue((PartialCheckoutEntity x) => x.Member)!,

        InvoiceIssuanceList = redisHash.ExtractHashValue((PartialCheckoutEntity x) => x.InvoiceIssuanceList)!,
        // ... 其他欄位同理

        // DateTime 特別處理（不是物件，直接 Convert）
        ShoppingDateTimeUtc = Convert.ToDateTime(
            redisHash.ExtractHashValue(nameof(CartEntity.ShoppingDateTimeUtc)))
    };

    return partialCheckoutEntity;
}
```

---

## 整體流程

```bash
C# 強型別物件（PartialCheckoutEntity）
    │
    │  ToHash()：把每個屬性序列化成 JSON 字串，放進 Dictionary
    ▼
Dictionary<string, string>
  "Member"              → "{\"CellPhone\":\"0912...\"}"
  "InvoiceIssuanceList" → "[{...},{...}]"
  "ShoppingDateTimeUtc" → "03/15/2026 01:00:00"
    │
    │  Redis HMSET
    ▼
Redis Hash（所有 value 都是字串，Redis 不知道型別）
    │
    │  Redis HMGET / HGET / HGETALL
    ▼
Dictionary<string, string>（從 Redis 讀回來）
    │
    │  ToType()：把每個字串 JSON 反序列化回對應的 C# 型別
    ▼
C# 強型別物件（PartialCheckoutEntity）重新組裝完成
```

---

## 總結

| 問題 | 答案 |
|------|------|
| Redis 本身有型別嗎？ | 沒有，Redis Hash 的每個 field value 都只是字串 |
| 型別由誰負責？ | `PartialCheckoutDataHashTransformer`，存的時候序列化，取的時候反序列化 |
| field 名稱是怎麼來的？ | 用 C# 的 `nameof(data.Member)` 取屬性名，存為 field 名 |
| 為什麼可以只讀一個欄位？ | Redis 的 `HGET` / `HMGET` 命令本來就支援只讀指定 field，這是 Redis Hash 資料結構的原生特性 |



![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart/5_transformer.png)


<!-- endtab -->


<!-- tab 只更新特定 Redis Field -->

## 問題：字串寫死 field 名的風險

假設要「只更新 `IsDeliverySet` 這個 field」，最直覺的寫法是：

```csharp
// ❌ 用字串寫死 field 名
await _redisDataProvider.HashSetAsync(key, "IsDeliverySet", true);
```

這樣有一個致命問題：如果有人把屬性改名叫 `IsDeliverySetting`，**程式不會報錯**，但 Redis 裡會多出一個新的 field，舊的 field 還在，資料就亂掉了。字串是死的，編譯器看不到。

---

## 解法：用 Lambda Expression 傳入屬性

```csharp
// ✅ 用 Lambda 傳入屬性
await _coreCacheService.HashSetAsync(
    (CheckoutEntity x) => x.IsDeliverySet,  // ← 這個 lambda 就是關鍵
    true, ...);
```

這樣如果有人把 `IsDeliverySet` 改名，**編譯器會直接報錯**，不會等到 runtime 才出問題。

---

## 這個 Lambda 怎麼變成 Redis 的 field 名？

**第一步：方法簽章接收的是 `Expression`，不是普通 `Func`**

```csharp
// CoreCacheService.cs
public Task HashSetAsync<TSourceClass, TProperty>(
    Expression<Func<TSourceClass, TProperty>> expr,  // ← Expression，不是普通 lambda
    TProperty data, ...)
{
    var memberInfo = GetPropertyMemberInfo(expr);  // ← 從 Expression 取出屬性名
    return _redisDataProvider.HashSetAsync(key, memberInfo.Name, data, expireTime);
    //                                          ↑
    //                          memberInfo.Name = "IsDeliverySet"（字串）
}
```

**第二步：`GetPropertyMemberInfo` 走訪語法樹取屬性名**

```csharp
private MemberInfo GetPropertyMemberInfo<TDelegate>(Expression<TDelegate> expr)
{
    var bodyExpr = expr.Body;  // 取出 lambda 的「主體」部分

    if (bodyExpr is MemberExpression memberExpr)
    {
        return memberExpr.Member;  // Member.Name 就是 "IsDeliverySet"
    }

    throw new NotSupportedException("只支援 x => x.屬性 這種寫法");
}
```

**第三步：`Expression` 和普通 `Func` 的差別**

```csharp
// 普通 Func：只是一個「可以執行的函數」
Func<CheckoutEntity, bool> f = x => x.IsDeliverySet;
// 只能呼叫 f(someEntity) 得到 true/false
// 無法知道它「讀取了哪個屬性」

// Expression<Func<>>：描述這個函數結構的語法樹物件
Expression<Func<CheckoutEntity, bool>> expr = x => x.IsDeliverySet;
// 編譯器不執行它，而是把它轉成一棵語法樹
// 可以檢查這棵樹，發現它是 MemberExpression
// 並且 Member.Name = "IsDeliverySet"
```

**語法樹結構示意：**

```bash
x => x.IsDeliverySet

        LambdaExpression
               │
          Body（主體）
               │
        MemberExpression        ← bodyExpr is MemberExpression → true
        ├── Expression: x       （參數）
        └── Member: IsDeliverySet  ← memberExpr.Member.Name = "IsDeliverySet"
```


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart/5_2_hardcode_lambda.png)


---

## 完整呼叫追蹤：只寫單一 field

以結帳設定收件人完成後，更新 `IsDeliverySet = true` 為例：

```csharp
// ① 呼叫端（某個 Processor）
await _coreCacheService.HashSetAsync(
    (CheckoutEntity x) => x.IsDeliverySet,  // Expression
    true,                                    // 值
    CoreCacheKeyEnum.CheckoutEntity,
    shopId,
    memberId,
    expireTime: null,
    args: checkoutUniqueKey);

// ② CoreCacheService.HashSetAsync() 內部
var memberInfo = GetPropertyMemberInfo(expr);
//   memberInfo.Name = "IsDeliverySet"

var queryKey = GetQueryKey(CoreCacheKeyEnum.CheckoutEntity, shopId, memberId, checkoutUniqueKey);
//   queryKey = "Core:CheckoutEntity-20230207:1001:888:a7f3bc91-..."

return _redisDataProvider.HashSetAsync(
    queryKey,        // Redis key
    "IsDeliverySet", // field 名（從 Expression 取出）
    true,            // field 值
    expireTime);

// ③ 最終打到 Redis 的指令
// HSET Core:CheckoutEntity-20230207:1001:888:a7f3bc91-...  IsDeliverySet  true
// 其他 19 個 field 完全不動
```

---

## 完整呼叫追蹤：只讀單一 field

以過期檢查只讀 `ShoppingDateTimeUtc` 為例：

```csharp
// ① 呼叫端（CacheLifeCycleService）
var shoppingDateTimeUtc = await _coreCacheService.HashGetAsync<CheckoutEntity, DateTime>(
    (CheckoutEntity x) => x.ShoppingDateTimeUtc,  // Expression
    CoreCacheKeyEnum.CheckoutEntity,
    shopId,
    memberId,
    args: checkoutUniqueKey);

// ② CoreCacheService.HashGetAsync() 內部
var memberInfo = GetPropertyMemberInfo(expr);
//   memberInfo.Name = "ShoppingDateTimeUtc"

var queryKey = GetQueryKey(...);
//   "Core:CheckoutEntity-20230207:1001:888:a7f3bc91-..."

return _redisDataProvider.HashGetAsync<string, DateTime>(
    queryKey,
    "ShoppingDateTimeUtc");  // 只傳這一個 field 名

// ③ 最終打到 Redis 的指令
// HGET Core:CheckoutEntity-20230207:1001:888:a7f3bc91-...  ShoppingDateTimeUtc
// → 只回傳 "2026-03-15 01:00:00"（幾十 bytes）
// → 不回傳其他 19 個 field
```

---

## 整體機制總結

```
呼叫端寫：
  x => x.IsDeliverySet
       ↓
  C# 編譯器把 lambda 轉成「語法樹物件」（Expression）
       ↓
  GetPropertyMemberInfo() 走訪語法樹，找到 MemberExpression
       ↓
  memberExpr.Member.Name = "IsDeliverySet"（取出屬性名字串）
       ↓
  用這個字串當 Redis HSET / HGET 的 field 名
       ↓
  Redis 只讀/寫這一個 field，其他 field 完全不動
```

用 Lambda 取代硬寫字串，讓「field 名」在編譯期就被型別系統保護。改名時編譯器立刻報錯，不怕打錯字，也不怕無聲地寫進錯誤的 field。



![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart/8_hget.png)


<!-- endtab -->


<!-- tab Checkout UniqueKey -->

## Shopping 如何拿到 UniqueKey

Shopping 每次呼叫 `api/checkouts/create`，都會從 response 拿 `UniqueKey`，然後用這個 Key 存 Redis：

```csharp
// CheckoutCreateApiProcessor.cs
var result = await _domainApiHelper.PostAsJsonAsync<CheckoutCreateResponse, CartCheckoutCreateRequest>(
    new Uri("api/checkouts/create", UriKind.Relative),
    context.Request, ...);

context.CheckoutUniqueKey = context.Response.Data!.UniqueKey;  // ← 從 CartService 的 response 取

// CheckoutCreateResponse.cs（CartService 回傳的結構）
public class CheckoutCreateResponse
{
    public string Url { get; set; }        // 付費網址
    public string UniqueKey { get; set; }  // ← 這個 K 值
    public string ReturnCode { get; set; }
}
```

---

## UUID 不是 CartService 產生的，是 WebStore（舊系統）給的

從 `DoPayProcessFlowProcessor.cs` 找到鐵證：

```csharp
// DoPayProcessFlowProcessor.cs
// CartService 呼叫 WebStore 舊系統的 API
var (statusCode, result) = await _webStoreWebApiClient.PostAsJsonAsync<ApiResultEntity<string>>(
    "PayLite/RequestPayProcessUrlV3", ...);

// WebStore 回傳的是一個 URL，例如：
// https://xxx.91app.com/ShoppingCart/Checkout?k=a7f3bc91-1234-5678-abcd-...
var returnUrl = new Uri(result.Data);

// CartService 從這個 URL 的 query string 抓出 k 值
var checkoutUniqueKey = HttpUtility.ParseQueryString(returnUrl.Query).Get("k")!;
```

---

## 完整流程

```bash
CartService → 呼叫 WebStore PayLite/RequestPayProcessUrlV3
                    ↓
WebStore 建立結帳 session，產生一個新的 k 值，回傳 URL
                    ↓
CartService 從 URL 的 ?k= 參數取出這個 k 值
                    ↓
CartService 用這個 k 當作 checkoutUniqueKey，回傳給 Shopping
```

「每次不同」的根源在於：WebStore 每次呼叫 `RequestPayProcessUrlV3` 都會建立一個全新的 session，k 值自然每次不同。



![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart/4_where_uuid_checkout.png)


<!-- endtab -->


<!-- tab 個資兩次存取（存原始、傳遮罩）-->

## 問題背景

結帳頁需要顯示收件人資料，但手機號碼、地址是敏感個資，**不應完整曝露在前端（browser）**。然而送出訂單時，後端又需要完整的原始個資才能打 CartService API。


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart/6_masks.png)



---

## 系統解法

### Step 1：GetCheckout 流程末端 — `MaskProcessDataProcessor`

```csharp
// MaskProcessDataProcessor.cs
public async Task ExecuteAsync(CheckoutContext context)
{
    // ① 先把原始明碼個資打包
    var personalData = new PersonalDataMaskEntity()
    {
        CheckoutUniqueKey = context.CheckoutUniqueKey,
        DisplayReceiver   = context.Data.DisplayReceiver, // 完整手機、地址
        Email             = context.Data.Member.Email,    // 完整 Email
        MemberInvoice     = context.Data.MemberInvoice,
        CellPhone         = context.Data.Member.CellPhone // 完整手機
    };

    // ② 存入 Redis PersonalDataMask（明碼保險箱）
    await this._personalDataMaskService.SetPersonalDataAsync(personalData);

    // ③ 個資遮罩開關檢查
    if (await this._personalDataMaskService.IsPersonalDataMaskEnabledAsync(...) == false)
    {
        return;  // 功能關閉，直接回傳明碼
    }

    // ④ 替換成遮罩版本回傳給前端
    var maskedData = this._personalDataMaskService.GetMaskedPersonalData(personalData);
    context.Data.DisplayReceiver          = maskedData.DisplayReceiver;           // 0912***678
    context.Data.MemberInvoice.Email      = maskedData.MemberInvoice.Email;       // a***@g***.com
    context.Data.Member.Email             = maskedData.Email ?? string.Empty;
    context.Data.Member.CellPhone         = maskedData.CellPhone ?? string.Empty;
}
```

### Step 2：SetDelivery（設定物流）時還原個資

```csharp
// CheckoutService.cs UnmaskSetDeliveryRequest()
private async Task<Internal.DeliverySetRequest> UnmaskSetDeliveryRequest(...)
{
    // 從 Redis 保險箱讀回原始明碼
    var redisData  = (await _personalDataMaskService.GetPersonalDataAsync(request.CheckoutUniqueKey))!;
    var maskedData = _personalDataMaskService.GetMaskedPersonalData(redisData);

    // 比對前端送來的是否是遮罩版，若是則換回明碼
    request.Receiver.CellPhone = RestoreMakedData(
        request.Receiver.CellPhone,      // 前端送來的（可能是 0912***678）
        maskedReceiverData?.CellPhone,   // 系統產生的遮罩版
        unmaskedReceiverData?.CellPhone, // Redis 保存的明碼
        "DeliverySet");

    // 更新 Redis 個資（如果使用者修改了地址/電話）
    await this._personalDataMaskService.SetPersonalDataAsync(redisData);
}
```

### Step 3：SendCompleteRequest（送出訂單）也還原個資

```csharp
// CheckoutService.cs
entity = await this.UnmaskOrderCompleteRequest(entity);
// 還原後用明碼打 CartService 成立訂單
```

---

## 遮罩規則

| 資料 | 原始 | 遮罩後 |
|------|------|--------|
| 手機 | `0912345678` | `0912***678` |
| Email | `user@gmail.com` | `u***@g***.com` |
| 地址 | `台北市信義區信義路五段7號` | `台北市信義區***` |
| 姓名 | `王小明` | `王**` |

---

## 遮罩開關：兩層控制

```csharp
// PersonalDataMaskService.IsPersonalDataMaskEnabledAsync()

// 第一層：全市場開關（ShopStaticSetting GlobalSettingShopId）
var marketSwitchEnabled = await IsMarketSwitchEnabledAsync();
if (marketSwitchEnabled == false) return false;

// 第二層：商店獨立開關
var shopSwitchEnabled = await IsShopSwitchEnabledAsync(shopId);
return shopSwitchEnabled;
```

> 即使遮罩功能關閉，原始個資仍然會存入 Redis，因為 `SetPersonalDataAsync` 在遮罩判斷之前就執行了。這確保了 Unmask 邏輯無論開不開都能正常運作。


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart/6_2_maskrule.png)


<!-- endtab -->


<!-- tab PartialCheckoutData TTL 由設定檔控制-->

## 程式碼

```csharp
// CoreCacheService.cs 建構子
public CoreCacheService(
    IRedisInstanceResolver redisInstanceResolver,
    IConfigurationService configurationService)
{
    _redisDataProvider = redisInstanceResolver.GetCacheProvider(CacheInstanceType.ShoppingCache);

    // TTL 從設定檔讀取，不硬寫在程式碼裡
    _defaultCacheExpireTime = TimeSpan.FromHours(
        configurationService.GetConfiguration<int>(
            ConfigTypeEnum.Normal,
            "CacheExpireSetting:CoreData.ExpireHours"));  // ← 設定檔 key
}
```

設定檔（`appsettings.json` 或環境設定）：

```json
{
  "CacheExpireSetting": {
    "CoreData.ExpireHours": 2
  }
}
```


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart/7_cachetime_config.png)


---

## 好處

| 情境 | 不用設定檔控制 | 用設定檔控制 |
|------|---------------|-------------|
| 促銷活動期間，希望延長結帳保留時間 | 需要改程式碼 + 重新部署 | 改設定，即時生效 |
| 發現 TTL 太長，Redis 記憶體壓力大 | 需要改程式碼 + 重新部署 | 改設定，即時生效 |
| QA / Prod 環境希望不同 TTL | 無法區分 | 各環境獨立設定 |





<!-- endtab -->


<!-- tab PersonalDataMask TTL 60 分鐘-->

## 兩個 TTL 的不同

```csharp
// ① PartialCheckoutData：由設定檔控制（預設 CoreData.ExpireHours）
// ② PersonalDataMask：明確指定 60 分鐘

await _coreCacheService.SaveAsync(
    entity,
    CoreCacheKeyEnum.PersonalDataMask,
    shopId, memberId,
    TimeSpan.FromMinutes(60),  // ← 明確指定 60 分鐘
    entity.CheckoutUniqueKey);
```

購物車過期判斷是 30 分鐘（`CacheLifeCycleService`）：

```csharp
// CacheLifeCycleService.cs
private const int CartExpireMinutes = 30;

return dateTime.ToLocalTime().AddMinutes(CartExpireMinutes) < DateTime.Now;
```

---

## 為什麼 PersonalDataMask 要比購物車過期時間長？

```bash
使用者流程：
  t=0min   進入結帳頁（建立 PartialCheckoutData）
  t=25min  選好配送方式、付款方式（SetDelivery 需要 Unmask）
  t=30min  PartialCheckoutData 可能過期（系統判斷為過期）
  t=31min  使用者點「送出訂單」（SendCompleteRequest 需要 Unmask）
```

如果 `PersonalDataMask` 也是 30 分鐘，`t=31min` 就讀不到原始個資，訂單無法成立。
**60 分鐘確保即使購物車已標記過期，個資仍可被使用於最後的送單動作。**



![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart/7_2_double_expire.png)


<!-- endtab -->


<!-- tab 只讀 ShoppingDateTimeUtc 判斷過期-->

## 問題背景

要判斷購物車是否過期，最直觀的做法是：讀回整份 `PartialCheckoutEntity`，再取其中的 `ShoppingDateTimeUtc`。但整份 Entity 可能很大（包含 `TemperatureSalepageList` 等 List），每次進入結帳頁都要判斷，代價太高。

---

## 系統做法：只讀一個 field

```csharp
// CacheLifeCycleService.cs
public async Task<bool> IsCheckoutExpireAsync(string uniqueKey, long memberId)
{
    var nameOfShoppingDateTimeField = nameof(PartialCheckoutEntity.ShoppingDateTimeUtc);

    // 只請求 ShoppingDateTimeUtc 這一個欄位
    var redisHash = await _coreCacheService.HashBatchGetAsync<string>(
        new List<string> { nameOfShoppingDateTimeField },  // ← 只要這 1 個欄位
        CoreCacheKeyEnum.PartialCheckoutData,
        _requestDataRetriever.ShopId,
        memberId,
        uniqueKey);

    var shoppingDateTimeUtc = redisHash.ExtractHashValue(nameOfShoppingDateTimeField);
    var dateTime = Convert.ToDateTime(shoppingDateTimeUtc);

    return dateTime.ToLocalTime().AddMinutes(CartExpireMinutes) < DateTime.Now;
}
```

底層 Redis 命令：

```bash
# 只讀一個欄位，傳輸量極小（幾十 bytes）
HMGET Core:PartialCheckoutData-20230410:10416:39326184:uuid  ShoppingDateTimeUtc
```

---

## 觸發時機：每次 GetCartFromP2Async 都先檢查

```csharp
// CheckoutService.cs
public async Task<CheckoutEntity> GetCartFromP2Async(string checkoutUniqueKey)
{
    // 過期保護（只讀 1 個欄位）
    if (await _cacheLifyCycleService.IsCheckoutExpireAsync(checkoutUniqueKey, ...))
    {
        throw new CheckoutGetException(CheckoutGetExceptionTypeEnum.CheckoutCacheExpired);
    }
    // ... 後續流程
}
```

---

> **設計精髓：** 用最小的 Redis 查詢代價，做最快速的「是否過期」判斷。每次進入結帳頁都要檢查，因此要盡可能輕量。

<!-- endtab -->


<!-- tab summmary-->


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart/9_table_all.png)



![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart/shopping-cart-cache.png)


<!-- endtab -->


{% endtabs %}

