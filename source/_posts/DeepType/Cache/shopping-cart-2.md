---
title: shopping-cart-2
date: 2026-03-15 08:25:03
categories: 落葉下的存檔
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart-2/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart-2/1_landing.png
tags:
    - Cache
    - Redis
toc:
toc_number:
comments :
---


{% tabs shopping-cart-2%}


<!-- tab 為什麼需要 AntiForgeryToken -->


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart-2/5_csrf_token.png)




**CSRF（Cross-Site Request Forgery，跨站請求偽造）** 是一種攻擊手法：攻擊者讓已登入的使用者，在不知情的狀況下，對目標網站送出偽造的請求。

```bash
① 使用者登入了購物網站，瀏覽器已有登入 Cookie
② 使用者同時開著惡意網頁（例如被誘騙點開的廣告）
③ 惡意網頁偷偷對購物網站發出 POST /api/checkout/create 請求
④ 瀏覽器自動帶上 Cookie，購物網站以為是本人操作
⑤ 攻擊者成功替使用者結帳，或清空購物車
```

**請求必須帶著一個只有真正前端頁面才拿得到的一次性 Token**。惡意網頁無法預先取得這個 Token，因此偽造請求會被擋下

---

## Redis Key 設計

```bash
Core:AntiForgeryToken-20230821:{shopId}:{token}

範例：
Core:AntiForgeryToken-20230821:10416:f47ac10b-58cc-4372-a567-0e02b2c3d479
```

---

## 前端進入結帳頁，取得 Token

```csharp
// AntiForgeryService.cs
public async Task<AntiForgeryTokenResponseEntity> GetAntiForgeryTokenAsync()
{
    //// 每次都產生全新的 Guid，36 碼，保證唯一
    var token = Guid.NewGuid().ToString();
    var antiForgeryTokenEntity = new AntiForgeryTokenEntity { Token = token };
    var isEnableTokenValidation = IsEnableTokenValidation();
    if (isEnableTokenValidation)
    {
        //// 注意：這裡沒有 memberId，只有 shopId + token
        await _coreCacheService.SaveAsync(
            antiForgeryTokenEntity,
            CoreCacheKeyEnum.AntiForgeryToken,
            _requestDataRetriever.ShopId,
            token);  // ← token 本身當作 key 的一部分
    }

    return new AntiForgeryTokenResponseEntity { Token = token };
}
```

Redis 此時執行
```bash
SET Core:AntiForgeryToken-20230821:10416:f47ac10b-...  "{\"Token\":\"f47ac10b-...\"}"
```

前端收到 token 後，存在頁面記憶體（或 sessionStorage），不允許其他網域讀取。

---

## 使用者送出操作時，後端驗證 Token

```csharp
// AntiForgeryService.cs
public async Task ValidateAsync(string? antiForgeryToken)
{
    var isEnableTokenValidation = IsEnableTokenValidation();
    if (isEnableTokenValidation == false)
    {
        return;  // 開關未開啟，跳過驗證
    }

    //// 基本格式驗證：Token 必須是 36 碼的 Guid
    if (antiForgeryToken == null || antiForgeryToken.Length != TokenLength)
    {
        throw new AntiforgeryValidationException(AntiforgeryValidationExceptionTypeEnum.InvalidToken);
    }

    //// 去 Redis 查這個 Token 是否存在
    var antiforgyTokenEntity = await _coreCacheService.LoadAsync<AntiForgeryTokenEntity>(
        CoreCacheKeyEnum.AntiForgeryToken,
        _requestDataRetriever.ShopId,
        antiForgeryToken);  // ← 用 token 本身去查 Redis

    //// 查不到 → Token 無效或已被用過
    if (antiforgyTokenEntity == null)
    {
        throw new AntiforgeryValidationException(AntiforgeryValidationExceptionTypeEnum.InvalidToken);
    }
}
```

Redis 此時執行：
```bash
GET Core:AntiForgeryToken-20230821:10416:f47ac10b-...
→ 有值 = Token 有效
→ nil  = Token 無效（從未存在，或已被刪除）
```

---

## Step 3：驗證成功後，立刻刪除 Token（一次性）

```csharp
// AntiForgeryService.cs
public async Task RemoveAsync(string? antiForgeryToken)
{
    //// ...格式驗證...

    await _coreCacheService.RemoveAsync(
        CoreCacheKeyEnum.AntiForgeryToken,
        _requestDataRetriever.ShopId,
        antiForgeryToken);
}
```

Redis 此時執行：
```bash
DEL Core:AntiForgeryToken-20230821:10416:f47ac10b-...
```

Token 刪除後，就算攻擊者攔截到這個 Token，再次送出請求，`ValidateAsync` 就會查不到，直接拋例外。

---

## 設計決策解析

| 設計決策 | 理由 |
|---------|------|
| **沒有 memberId** | CSRF 防護是「請求維度」的，未登入用戶（訪客）也可能需要送出表單，如果綁定 memberId，訪客就無法被保護 |
| **token 本身是 Key 的一部分** | 驗證時只需帶著 Token 去 Redis 查，不用先找到使用者再比對，時間複雜度 O(1)，極速 |
| **用完就刪（一次性）** | 防止重複提交：同一份表單就算按了兩次送出，第一次刪除 Token 後，第二次就會被擋住 |
| **開關控制（IsEnableTokenValidation）** | 逐步推出，可以針對特定商店或環境開啟，不影響其他商店 |

---

## 實境案例

**情境：惡意機器人嘗試批量結帳**

```bash
① 機器人直接 POST /api/checkouts/create，沒有帶 antiForgeryToken
② AntiForgeryService.ValidateAsync() 收到 null → 拋 InvalidToken 例外
③ 請求被拒絕，購物車結帳流程中止
```

**情境：正常使用者結帳**

```bash
① 使用者點進結帳頁 → GET /api/antiforgery/token
   → 後端 Guid.NewGuid() = "f47ac10b-..."
   → 存入 Redis，回傳給前端

② 使用者點「確認結帳」→ POST /api/checkouts/create，header 帶上 token
   → ValidateAsync("f47ac10b-...") → Redis GET → 有值 → 驗證通過
   → RemoveAsync("f47ac10b-...") → Redis DEL → Token 消失

③ 使用者不小心按了兩次「確認結帳」
   → 第二次 POST 送出同一個 "f47ac10b-..."
   → ValidateAsync() → Redis GET → nil → 拋 InvalidToken
   → 重複送出被擋住
```

<!-- endtab -->


<!-- tab DataCache vs CoreCache -->

## 兩種 Cache Service 的職責分工

系統設計了兩套完全不同用途的 Cache 服務，不能混用：

| | DataCacheService | CoreCacheService |
|-|-----------------|-----------------|
| **用途** | 商店靜態設定（全店共用） | 個人結帳流程（每人獨立） |
| **資料特性** | 改動頻率低，讀取頻率極高 | 每個結帳 session 獨立 |
| **Cache 層數** | 雙層（Memory + Redis） | 單層（Redis 只） |
| **Key 有 memberId？** | ❌ 無（商店維度） | ✅ 有（個人維度） |
| **TTL** | Redis 30 分鐘，Memory 10 秒 | 由設定檔控制（預設數小時） |


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart-2/2_datacache_corecache.png)


---

## DataCacheService：三種存取模式


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart-2/2_2_datacache.png)



### 模式 A：Memory + Redis（雙層快取，最常用）

```csharp
// DataCacheService.cs
public async Task<T?> GetMemoryOrRedisCacheDataAsync<T>(
    long shopId, string cacheName, string cacheType, string key,
    Func<Task<T?>> source,
    int memoryCacheExpirationSecond = 10,    // L1 記憶體 10 秒
    int redisCacheExpirationSecond = 1800,   // L2 Redis 30 分鐘
    ...)
{
    var fullCacheName = $"{cacheName}-{cacheType}-{shopId}-{AppendLocale(key)}";

    //// 第一層：先查 MemoryCache（毫秒級）
    var memoryCacheObject = memoryCache.Get(fullCacheName);

    if (memoryCacheObject == null)
    {
        //// 第二層：Memory 沒有，去查 Redis（幾毫秒）
        memoryCacheObject = await GetRedisCacheDataAsync(...);

        if (memoryCacheObject != null)
        {
            //// 寫回 MemoryCache，下次直接從記憶體拿
            memoryCache.Set(fullCacheName, memoryCacheObject,
                DateTimeOffset.Now.AddSeconds(memoryCacheExpirationSecond));
        }
    }
    // Redis 也沒有的話，就呼叫 source()（通常是打 DB 或外部 API）
}
```

**讀取順序：**
```bash
請求
  │
  ├─ L1 MemoryCache 有嗎？ → 有 → 直接回傳（~0.01ms）
  │
  ├─ L2 Redis 有嗎？ → 有 → 回傳 + 寫回 MemoryCache（~1ms）
  │
  └─ 都沒有 → 呼叫 source()（DB / 外部 API）→ 寫入 Redis + MemoryCache（~10~100ms）
```

**實境案例：商店物流類型設定**

商店 10416 一天可能有 100,000 次結帳請求，每次都需要查「支援哪些配送方式」。
- 若每次都打 DB → 100,000 次 DB 查詢，DB 會被打爆
- 有 Redis（30 分鐘）→ 只有第一次打 DB，之後 29 分鐘全從 Redis 拿
- 有 MemoryCache（10 秒）→ 10 秒內所有請求全從記憶體拿，Redis 也不用打

---

### 模式 B：純 Redis（跨 Pod 共用但無記憶體加速）

```csharp
public async Task<T?> GetRedisCacheDataAsync<T>(...)
{
    //// 直接查 Redis，不走 MemoryCache
    var result = await _cacheService.GetAsync(
        cacheName, cacheType, key,
        TimeSpan.FromSeconds(expirationSecond),
        source,
        CommandFlags.PreferReplica);  // ← 優先讀取副本節點，分散讀取壓力
    return result;
}
```

**為什麼不加 MemoryCache？** 因為系統通常有多個 Pod（水平擴展），MemoryCache 是 Pod 內部的記憶體，各 Pod 之間不共享。若資料需要即時跨 Pod 一致，就只能用 Redis。

**實境案例：`LocationPickup`（取貨門市清單）**
- 門市清單可能在後台即時更新（新增/關閉門市）
- 若用 MemoryCache，Pod A 看到最新資料，Pod B 還是舊的（10 秒內）
- 用純 Redis，所有 Pod 都從同一個 Redis 節點讀，資料一致

---

### 模式 C：純 MemoryCache（僅本次請求，不跨 Pod）

```csharp
public async Task<T?> GetMemoryCacheData<T>(...)
{
    //// 只查 MemoryCache，資料不會存進 Redis
    var memoryCacheObject = memoryCache.Get(fullCacheName);

    if (memoryCacheObject == null)
    {
        memoryCacheObject = await source();  // ← 直接呼叫原始資料來源
        memoryCache.Set(fullCacheName, memoryCacheObject, ...);
    }
    return (T?)memoryCacheObject;
}
```

**使用場景：** 同一個 HTTP 請求內，同一筆資料會被讀取多次，但不需要跨請求保留。

**實境案例：`BlacklistService`（付款黑名單）**
- 一次結帳請求中，可能需要多次確認「這個付款方式是否在黑名單」
- 第一次查 DB，後續幾次查 MemoryCache（同一請求內）
- 請求結束，MemoryCache 自動失效（或設 10 秒就過期）
- 不需要存到 Redis，因為黑名單下一個請求就要重新查（可能被更新）

---

## DataCacheService 的 Key 設計


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart-2/3_datacachekey.png)


```bash
fullCacheName = {cacheName}-{cacheType}-{shopId}-{key}-{locale}

實際範例：
Cache:Prod:Shopping:Shop:GetShopShippingTypes-2020082601-10416:-zh-TW
                    ↑    ↑                    ↑      ↑    ↑    ↑
                  前綴  cacheName            cacheType  shopId key locale
```

**每個片段的設計理由：**

| 片段 | 值範例 | 設計理由 |
|------|--------|---------|
| `cacheName` | `Shop` | 分類管理，在 Redis 工具中一眼看出是哪個服務的資料 |
| `cacheType + 日期版本` | `GetShopShippingTypes-2020082601` | **版本號就是部署日期**；改資料結構時只需改日期，舊 Key 自然失效，無需 FLUSHDB |
| `shopId` | `10416` | 多租戶隔離，商店 A 的物流設定不能被商店 B 讀到 |
| `locale` | `zh-TW` | 多語系隔離；台灣商品名稱是中文，香港版是繁中，新加坡版是英文，同一個 `shopId` 下有多份快取 |

**日期版本設計的核心價值：**

```bash
假設今天要在 ShopShippingTypes 新增一個欄位 "IsExpressDelivery"：

❌ 若沒有版本號：
   舊 Cache 還在 Redis 裡 → 所有 Pod 撈到舊結構 → 新欄位是 null → Bug

✅ 有版本號（改日期）：
   舊 Key：GetShopShippingTypes-2020082601-10416
   新 Key：GetShopShippingTypes-20231201-10416
   → 兩個 Key 完全不同，舊 Key 自然過期（30 分鐘），不影響新程式
   → 不需要 FLUSHDB（會影響所有商店的所有快取）
```

---

## CoreCacheService 的 Key 設計





CoreCache 有兩種 overload，對應「有沒有 memberId」

```csharp
// CoreCacheService.cs GetQueryKey（私有方法）

// 有 memberId 版本（個人資料）
private string GetQueryKey(CoreCacheKeyEnum keyEnum, long shopId, long memberId, params string[] args)
{
    var dataType = keyEnum switch
    {
        CoreCacheKeyEnum.PersonalDataMask    => "PersonalDataMask-20230207",
        CoreCacheKeyEnum.PartialCheckoutData => "PartialCheckoutData-20230410",
        _ => throw new ArgumentOutOfRangeException(...)
    };
    return $"Core:{dataType}:{shopId}:{memberId}:{string.Join(':', args)}";
}

// 沒有 memberId 版本（防偽 Token）
private static string GetQueryKey(CoreCacheKeyEnum keyEnum, long shopId, params string[] args)
{
    var dataType = keyEnum switch
    {
        CoreCacheKeyEnum.AntiForgeryToken => "AntiForgeryToken-20230821",
        _ => throw new ArgumentOutOfRangeException(...)
    };
    return $"Core:{dataType}:{shopId}:{string.Join(':', args)}";
}
```

**三種 CoreCacheKey 的完整 Key 範例：**


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart-2/4_corecache_samples.png)



```bash
① PartialCheckoutData（結帳半狀態）
   Core:PartialCheckoutData-20230410:10416:39326184:abc-def-1234
                                     ↑     ↑         ↑
                                   shopId memberId  checkoutUniqueKey

② PersonalDataMask（個資明碼保險箱）
   Core:PersonalDataMask-20230207:10416:39326184:abc-def-1234
                                  ↑     ↑         ↑
   （同一個 checkoutUniqueKey，同一個人，同一個 session）

③ AntiForgeryToken（無 memberId）
   Core:AntiForgeryToken-20230821:10416:f47ac10b-58cc-4372-a567-0e02b2c3d479
                                  ↑     ↑
                                shopId  token（Guid，36 碼）
```

**為什麼 ① 和 ② 用同一個 checkoutUniqueKey？**

因為這兩個 Cache 記錄的是**同一個結帳 session 的不同面向**：
- `PartialCheckoutData`：存結帳流程的狀態（選了哪個物流、哪張發票）
- `PersonalDataMask`：存個資明碼（送單時後端自己 unmask）

用同一個 Key 讓兩者的生命週期可以對應，不會出現「session 在但個資不見了」的情況。

---

## Redis 節點選擇

```csharp
// DataCacheService 建構子
_cacheService = redisInstanceResolver.GetCacheService(CacheInstanceType.ShoppingCache);

// CoreCacheService 建構子
_redisDataProvider = redisInstanceResolver.GetCacheProvider(CacheInstanceType.ShoppingCache);
```

兩者都使用同一個 Redis 節點（`ShoppingCache`），但用不同的操作方式（`ICacheService` vs `ICacheProvider`），前者是高階封裝（含 Get-or-Set 邏輯），後者是低階封裝（直接 HSET / HGET）。

<!-- endtab -->


<!-- tab 清除快取策略（CleanCacheService）-->

## 為什麼要主動清除快取？


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart-2/6_clean.png)


`DataCacheService` 的快取 TTL 是 **Redis 30 分鐘**。這表示商店後台剛儲存的設定，最多要等 30 分鐘才能反映在前端。

```bash
後台人員 → 修改商店物流設定（新增「順豐速運」）
               ↓
          存入資料庫
               ↓
          但 Redis 還快取著舊的物流清單（還有 28 分鐘才過期）
               ↓
前端使用者 → 看不到「順豐速運」選項（資料不一致）
```

`CleanCacheService` 的責任就是：**資料庫一更新，就立刻清除對應的 Redis 快取**，讓下一次請求強制重新從 DB 載入最新資料。

---

## 三個清除方法與對應情境

### 方法一：`CleanShippingTypesAsync`（清除物流設定快取）

**觸發時機：** 後台人員異動商店的配送方式設定（新增、停用、修改物流類型）

```csharp
// CleanCacheService.cs
public async Task CleanShippingTypesAsync(long shopId)
{
    //// 先查這個商店支援哪些語系（可能是 "zh-TW,en-US"）
    var contractLanguage = await this._shopDefaultService.GetShopDefaultValueAsync(
        shopId, ShopDefaultGroupTypeDefEnum.Language, ShopDefaultKeyEnum.Contract, false);

    //// 每個語系各有一份快取，要逐一清除
    foreach (var locale in contractLanguage!.Split(','))
    {
        //// 對應的快取 Key（由程式碼中的註解得知）：
        //// Cache:Prod:Shopping:Shop:GetShopShippingTypes-2020082601-10416:-zh-TW
        this._dataCacheService.RemoveCacheKey(
            shopId: shopId,
            cacheName: "Shop",
            cacheType: "GetShopShippingTypes-2020082601",
            key: $"-{locale}",
            isAlreadyAppendLocale: true);  // ← key 已含 locale，告訴系統不要再 append 一次
    }
}
```

**實境案例：**
```bash
後台人員為商店 10416 停用「黑貓宅急便」選項（台灣 + 英文版）

CleanShippingTypesAsync(shopId: 10416)
  ↓
contractLanguage = "zh-TW,en-US"
  ↓
迴圈執行兩次：
  DEL Cache:...:Shop:GetShopShippingTypes-2020082601-10416:-zh-TW
  DEL Cache:...:Shop:GetShopShippingTypes-2020082601-10416:-en-US
  ↓
下一個使用者請求物流清單時：
  Redis 查不到 → 重新從 DB 載入 → 看到最新（已無黑貓宅急便）
```

---

### 方法二：`CleanShopDefaultListAsync`（清除商店預設值快取）

**觸發時機：** 後台人員異動商店的各種預設設定（結帳頁顯示選項、功能開關等）

```csharp
public async Task CleanShopDefaultListAsync(long shopId)
{
    //// Cache:Prod:Shopping:ShopDefault:GetShopDefaultListAsync-2021011314-1093:-zh-TW
    foreach (var locale in contractLanguage!.Split(','))
    {
        this._dataCacheService.RemoveCacheKey(
            shopId: shopId,
            cacheName: "ShopDefault",
            cacheType: "GetShopDefaultListAsync-2021011314",
            key: $"-{locale}",
            isAlreadyAppendLocale: true);
    }
}
```

**`ShopDefault` 控制的設定範例：**
- 是否開啟新版購物車結帳（`Nine1ShoppingCheckoutEnabled`）
- 是否顯示發票 Email 欄位
- 預設配送方式

```bash
後台人員幫商店 10416 開啟「新版結帳流程」功能
  ↓
DB 更新 ShopDefault 設定
  ↓
CleanShopDefaultListAsync(10416) 被呼叫
  ↓
Redis 清除 ShopDefault 快取
  ↓
下一次使用者結帳，GetCartAndCalculateProcessor 讀到新設定
  → isCheckoutEnabled = true → 新版結帳頁啟用
```

---

### 方法三：`CleanMemberCardAsync`（清除會員卡等級快取）

**觸發時機：** 後台人員異動會員等級制度，或特定會員的等級變更

```csharp
public async Task CleanMemberCardAsync(long shopId, long memberId)
{
    foreach (var locale in contractLanguage!.Split(','))
    {
        //// 清除「全商店會員卡清單」快取
        //// Cache:...:CrmShopMemberCard:GetCrmShopMemberCardListFromWebStoreDBForShoppingCart-2020090800-11500:11500-zh-TW
        this._dataCacheService.RemoveCacheKey(
            shopId: shopId,
            cacheName: "CrmShopMemberCard",
            cacheType: "GetCrmShopMemberCardListFromWebStoreDBForShoppingCart-2020090800",
            key: $"{shopId}-{locale}",
            isAlreadyAppendLocale: true);

        //// 清除「精簡版會員卡清單」快取
        //// Cache:...:CrmShopMemberCard:GetCrmShopMemberCardListFromWebStoreDB-2017031816-41134:-zh-TW
        this._dataCacheService.RemoveCacheKey(
            shopId: shopId,
            cacheName: "CrmShopMemberCard",
            cacheType: "GetCrmShopMemberCardListFromWebStoreDB-2017031816",
            key: $"-{locale}",
            isAlreadyAppendLocale: true);

        if (memberId > 0)
        {
            //// 清除「特定會員的等級資料」快取
            //// Cache:...:VIPMember:GetCrmShopMemberCardDataAsync-2018092401-10416:39326184-zh-TW
            this._dataCacheService.RemoveCacheKey(
                shopId: shopId,
                cacheName: "VIPMember",
                cacheType: "GetCrmShopMemberCardDataAsync-2018092401",
                key: $"{memberId}-{locale}",
                isAlreadyAppendLocale: true);
        }
    }
}
```

注意這個方法**同時清除三種快取**：

1. 為購物車優化的完整會員卡清單
2. 舊版精簡會員卡清單（相容性）
3. 特定會員的個人等級（僅當 memberId > 0 時）

**實境案例：**

```bash
會員 39326184 消費達標，等級升為「金卡會員」
  ↓
DB 更新會員等級
  ↓
CleanMemberCardAsync(shopId: 10416, memberId: 39326184)
  ↓
清除三份相關快取：
  DEL ...CrmShopMemberCard:GetCrmShopMemberCardListFromWebStoreDBForShoppingCart...-10416-zh-TW
  DEL ...CrmShopMemberCard:GetCrmShopMemberCardListFromWebStoreDB...-zh-TW
  DEL ...VIPMember:GetCrmShopMemberCardDataAsync...-39326184-zh-TW
  ↓
下次使用者打開購物車：重新載入 → 看到金卡折扣和優惠
```

---

## `isAlreadyAppendLocale` 參數的設計意義

`DataCacheService.RemoveCacheKey` 有一個特殊參數：

```csharp
public bool RemoveCacheKey(
    long shopId, string cacheName, string cacheType, string key,
    bool isAlreadyAppendLocale = false)  // ← 預設 false
{
    if (isAlreadyAppendLocale == false)
    {
        key = AppendLocale(key);  // ← 自動在 key 後面加上 "-zh-TW"（從 RequestContext 取）
    }
    // 若 isAlreadyAppendLocale = true，表示 key 已經手動包含了 locale，不再 append
    _cacheService.Remove(cacheName, cacheType, key);
}
```

**為什麼要有這個參數？**

`CleanCacheService` 是在**後台 OPS 操作**時被呼叫的，不是在使用者的 HTTP 請求中，因此 `RequestDataRetriever.Lang` 可能是空的或不正確的。

所以 `CleanCacheService` 自己手動組 locale（從 `contractLanguage.Split(',')` 來），再傳入 `isAlreadyAppendLocale: true` 告訴 `DataCacheService` 不要再 append 一次，避免 key 變成：
```bash
GetShopShippingTypes-...-10416:-zh-TW-zh-TW  ← 重複 locale，永遠刪不到正確的 key
```

---

## 快取清除與 TTL 的搭配策略

```bash
資料更新頻率     →  快取策略
─────────────────────────────────────────────────────────
極少更新          →  只靠 TTL 過期（30 分鐘可接受）
後台人員手動異動  →  更新 DB 後，主動呼叫 CleanCacheService 清除
程式碼/資料結構變更 →  升版日期版本號，所有舊 Key 自然失效
```

| 快取清除方式 | 適用情境 | 範例 |
|------------|---------|------|
| 靠 TTL 自然過期 | 資料異動後，短暫不一致可接受 | 商品價格（30 分鐘誤差可接受） |
| `CleanCacheService` 主動清除 | 後台異動後要立即生效 | 物流設定、會員等級、商店功能開關 |
| 升級日期版本號 | 資料結構變更（部署時） | 新增欄位、改變 Key 格式 |



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart-2/6_2_clean_cache.png)


<!-- endtab -->


<!-- tab 整體架構總結 -->


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/shopping-cart-2/7_all.png)


## 兩個 Redis 節點的職責

系統使用同一個 `CacheInstanceType.ShoppingCache` Redis 節點，但在邏輯上分成兩套 Cache 體系：

```bash
Redis（ShoppingCache）
  ├── DataCacheService 管理的 Key
  │     格式：Cache:Prod:Shopping:{cacheName}:{cacheType}-{日期版本}-{shopId}:{key}-{locale}
  │     用途：商店靜態設定、物流類型、會員卡等級（全店共用）
  │     TTL：30 分鐘
  │     清除方式：CleanCacheService 主動清除 或 TTL 自然過期
  │
  └── CoreCacheService 管理的 Key
        格式：Core:{dataType}-{日期版本}:{shopId}:{memberId}:{uniqueKey}
        用途：個人結帳流程（PartialCheckoutData, PersonalDataMask, AntiForgeryToken）
        TTL：由設定檔控制（PartialCheckoutData）/ 60 分鐘（PersonalDataMask）/ TTL（AntiForgeryToken）
        清除方式：結帳完成自動失效 / 用完刪除（AntiForgeryToken）
```

---

## 三種 CoreCacheKey 快速對照

| Key 類型 | Redis 結構 | TTL | memberId | 清除時機 |
|---------|-----------|-----|---------|---------|
| `PartialCheckoutData` | **Hash**（8 個 field） | 設定檔（數小時） | ✅ | 結帳完成 / 過期 |
| `PersonalDataMask` | **String**（JSON） | 固定 60 分鐘 | ✅ | 過期 |
| `AntiForgeryToken` | **String**（JSON） | 與 PartialCheckoutData 同 | ❌（無） | 驗證成功後立即刪除 |

---

## 設計原則總覽

| 原則 | 體現方式 | 好處 |
|------|---------|------|
| **日期版本號** | Key 後綴部署日期（如 `-20230410`） | 改資料結構免 FLUSHDB，舊 Key 自然失效 |
| **多語系隔離** | Key 後綴 locale（如 `-zh-TW`） | 台灣版與英文版不互相污染 |
| **三層 Key 隔離** | `shopId:memberId:uniqueKey` | 多租戶 × 多用戶 × 多 session，零資料洩漏 |
| **Redis Hash** | PartialCheckoutData 用 Hash 存 8 個 field | 只讀需要的 field，省流量 |
| **一次性 Token** | AntiForgeryToken 驗證後立即刪除 | 防止重複提交、CSRF 防護 |
| **TTL 由設定檔控制** | `CacheExpireSetting:CoreData.ExpireHours` | 不重新部署即可調整 TTL |
| **個資遮罩** | PersonalDataMask 存明碼，前端只看遮罩版 | 個資不曝露在 API Response |
| **主動清除** | `CleanCacheService` 在資料異動後清 Redis | 後台操作即時生效，不需等 TTL |

---

## 完整流程回顧

```bash
使用者進入結帳頁
    │
    ├─ 取 AntiForgeryToken → Redis SET Token（無 memberId）
    │
    ├─ Shopping 呼叫 CartService
    │      ↓
    │   CartService 呼叫 WebStore 取得 k 值（UUID）
    │      ↓
    │   CartService HMSET CheckoutEntity（Hash，含 ShoppingDateTimeUtc）
    │      ↓
    │   回傳 UniqueKey 給 Shopping
    │
    ├─ Shopping CachePartialCheckoutDataProcessor
    │      HMSET PartialCheckoutData（Hash，8 個 field）
    │
    └─ 回傳 UniqueKey 給瀏覽器

使用者操作結帳頁（選物流、選付款）
    │
    ├─ 每次操作前：ValidateAsync(antiForgeryToken) → Redis GET
    ├─ 操作完成後：RemoveAsync(antiForgeryToken) → Redis DEL
    ├─ SetDelivery → HSET IsDeliverySet=true（只更新一個 field）
    └─ GetCheckout → 過期檢查（HMGET ShoppingDateTimeUtc，只讀 1 個 field）

使用者送出訂單
    │
    ├─ UnmaskOrderCompleteRequest → GET PersonalDataMask（還原明碼）
    └─ 打 CartService 成立訂單 → 結帳 session 自然失效
```

<!-- endtab -->


{% endtabs %}