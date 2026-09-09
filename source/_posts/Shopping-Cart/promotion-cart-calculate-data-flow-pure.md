---
title: Promotion Cart Calculate
date: 2026-09-07 10:30:56
categories: Shopping-Cart
tags:
toc:
toc_number:
comments:
---

## 架構摘要

這篇文章從購物車促銷計算的系統邊界開始，依序整理請求資料、活動規則來源、Redis 快取分支與實際失敗位置，讓排查路徑可以從入口一路追到程式碼證據。

### 外部呼叫端

- `Cart Service`：組裝購物車與 `SalepageSkuList`，呼叫 `POST /api/cart-calculate`

### Promotion Frontend 邊界

- `CalculateController`：API 入口與 response 包裝
- `CalculateService`：購物車促銷計算協調
- `PromotionEngine`：活動匹配與排序時失敗

失敗位置：`MatchedUserScopes` 為 `null`，`GetUserScopePriority()` 對 null 集合執行 LINQ，拋出 `ArgumentNullException`。

圖例：

- 服務或 API
- 快取或資料取得
- 失敗節點

## C4 容器視圖

以系統邊界區分呼叫者、Promotion Frontend 內部容器、外部 Collection API 與持久化資料源。箭頭標註的是實際 API 或資料用途，不把不同責任混在同一個節點。

### Caller System

- `Cart Service`：HTTP Client，購物車促銷請求

### Promotion Frontend System

- `CalculateController`：`POST /api/cart-calculate`
- `CalculateService`：建立計算流程上下文
- `PromotionRuleRepository`：找候選活動與載入規則
- `Promotion Engine`：規則匹配、`DynamicPriority` 排序

### Supporting Systems

- `Collection API`：`api/salepage-collections:match`，商品頁與活動關聯
- `Redis`：活動與商品匹配快取
- `WebStoreDB`：`PromotionEngine`、`PromotionEngine_Rule`
- `MongoDB PromotionDB`：`PromotionRule`、`PromotionRuleSetting`

此圖採 C4 Container 的責任分層方式，搭配資料流圖的方向箭頭。Collection API 是外部支援系統，活動資料庫是持久化來源，Redis 是可失效的快取，不代表永久資料來源。

## 請求與資料流

### 請求流

1. `Cart Service` 送出購物車商品、會員、付款與配送資料
2. 透過 HTTPS 呼叫 `POST /api/cart-calculate`
3. `Promotion Frontend` 從 `SalepageSkuList` 取得商品頁 ID
4. 透過 HTTP 呼叫 `salepage-collections:match`
5. `Collection API` 依既有商品集合關聯回傳候選活動

### 識別值轉換

1. `SalepageId`：來自購物車商品頁
2. `SalepageCollectionId`：商品所屬集合
3. `PromotionEngineId`：候選活動規則識別值

### 資料流明細

| 步驟 | 輸入 | 輸出 | 責任元件 |
| --- | --- | --- | --- |
| 1 | 購物車商品與 `SalepageId` | `POST /api/cart-calculate` | `Cart Service` |
| 2 | `SalepageIds` | 商品頁的匹配集合 | `Collection API` |
| 3 | `SalepageCollectionId`、`PromotionEngineId` | 完整活動規則 | `PromotionRuleRepository` |
| 4 | 規則物件 | 排序後的促銷結果 | `Promotion Engine` |

## 活動規則來源與快取分支

### 規則載入路徑

1. Redis lookup：以商店與活動 ID 查快取
2. 命中：直接使用快取規則，跳過資料庫查詢
3. 未命中：由 Repository fallback 查 WebStoreDB 或 MongoDB

| 活動 ID 條件 | 資料來源 | 主要內容 |
| --- | --- | --- |
| 傳統活動或低於 Mongo 分界 | `WebStoreDB.PromotionEngine` | `PromotionEngine_Rule` JSON 與活動基本欄位。 |
| 大於等於 `MongoDB.PromotionId.Divide` | `MongoDB PromotionDB` | `PromotionRuleSetting.UserScope`、`ProductScope` 等設定。 |
| 已存在快取 | `Redis` | 快取的活動規則資料，可能保存舊版本。 |

資料責任：Collection API 只提供商品頁與活動的關聯。活動規則內容仍由 Promotion Frontend 的 Repository 從 Redis、WebStoreDB 或 MongoDB 載入。

## Redis Key 與快取分支

本流程使用 Cache Aside。應用程式先查 Redis，未命中時查持久化資料，再把結果寫入 Redis。Key 由模組名稱、資料類型版本、`ShopId` 與業務識別值組成。

| 用途 | 完整格式 | 最後一段識別值 |
| --- | --- | --- |
| 商品頁匹配結果 | `CartCalculate:SalepageCollection-20230801:{shopId}:{salepageId}` | `salepageId` |
| 活動規則資料 | `CartCalculate:PromotionEngine-20260317:{shopId}:{promotionEngineId}` | `promotionEngineId` |

Key 來源：上述格式分別定義在 `PromotionRuleRepository.GetMatchedSalepageCollectionAsync` 與 `GetPromotionEngineDataListAsync` 的 `FormatCacheKey` 區域。日期字串是程式內的 key version，不能視為資料日期。

### Cache Aside 流程

1. 查詢 Redis：以 `ShopId` 加識別值組成 key
2. HIT：直接使用快取，跳過外部查詢或資料庫
3. MISS：查詢資料源，回填規則或商品匹配快取

一致性注意：若活動規則已在 Redis 命中，Promotion Frontend 可能持續使用舊版本內容。排查 `MatchedUserScopes` 時必須同時比對 Redis value 與資料庫原始資料。

## 活動建立與會員範圍

### 建立活動時，會員範圍如何寫進規則？

建立「全體會員適用」「金卡限定」或「當月壽星限定」活動時，WebAPI 會先把會員條件組成清單，再與折扣或回饋條件一起存入 SQL。購物車計算讀取的是這份已保存的規則。

本節聚焦 `POST /api/promotion-rules/create` 的 SQL 建立流程。`MatchedUserScopes` 由 request 建構，最後寫進 `PromotionEngine_Rule` 的 JSON。

#### 兩個相似欄位，負責不同事情

| Request 欄位 | 用途 |
| --- | --- |
| `TargetMemberTypeDef` | 字串，供基本驗證、主檔活動對象設定等邏輯使用。 |
| `TargetMemberType` | Enum，實際決定 `MatchedUserScopes` 的建立分支。 |
| `CrmShopMemberCardIds` | 指定會員等級時，轉成各會員卡的 Tag。 |
| `MemberCollectionId` | 指定客群時，作為客群 Tag。 |
| `IsBirthdayMonthEnabled` | 壽星開關；支援的回饋活動會追加壽星 Tag，並把開關存進規則。 |

**兩個活動對象欄位不會自動同步。**`TargetMemberTypeDef` 與 `TargetMemberType` 是獨立屬性。只傳前者為 `All`，後者仍可能保留預設值 `0`，使會員範圍建構得到 `null`。

#### SQL Create 的建立順序

1. **驗證 request** `PromotionEngineController.Create()` 先執行 `ValidateAndThrowAsync()`。基本驗證檢查 `TargetMemberTypeDef` 非空且為合法 enum 名稱；部分活動另有限制，但沒有全面檢查兩個活動對象欄位一致。

1. **選擇規則服務、整理客群設定** `PromotionEngineService.Create()` 依 `TypeDef` 選出 `ruleService`。`SetMemberCollectionAsync()` 依活動種類與商店開關調整客群 ID，例如部分全體會員活動設成 `-1`；這一步不會將字串欄位同步到 enum 欄位。

1. **建立 SQL 活動資料** 開啟 `TransactionScope`，建立主檔、設定與商品範圍。若對象是 `MemberTier`，另外將活動與會員卡的關聯寫入 `PromotionEngineMemberTier`。

1. **在記憶體建立會員範圍** `SetMatchedUserScopes(entity.TargetMemberType, entity.CrmShopMemberCardIds, entity.MemberCollectionId)` 呼叫 `GetUserScopes()`，將結果放進 `_matchedUserScopes`。這裡直接使用 request，沒有回查剛寫入的會員等級關聯表。

1. **符合壽星設定時追加條件** 支援的回饋活動啟用壽星開關後，會確認或建立壽星客群，再以 `SetBirthdayMonthMatchedUserScopes()` 對既有清單追加 `CurrentBirthdayMonth`。

1. **組合規則並保存** 各活動的 `GetPromotionEngineRule()` 組合折扣或回饋條件。共用的 `GetRuleObject()` 設定 `result.MatchedUserScopes = _matchedUserScopes`，序列化後嘗試 `engine.LoadRules()`，再更新 `PromotionEngine_Rule`。完成其餘建立步驟後提交交易。

#### 會員範圍的建立分支

| `TargetMemberType` | 建立內容 | 結果 |
| --- | --- | --- |
| `All` | 加入一個 `AllUserScope`。 | 非 null，代表全體會員。 |
| `MemberTier` | 每個會員卡 ID 建立 `TagUserScope`，Tag 為 `CrmShopMemberCard:{id}`。 | 有值為 Tag 清單；空清單得到 `[]`。 |
| `MemberCollection`，ID 有值 | 建立一個 `TagUserScope`，Tag 直接使用客群 ID。 | 非 null，不展開個別會員名單。 |
| `MemberCollection`，ID 為 null 或空字串 | 不加入任何元素。 | 空集合 `[]`；部分活動會先被客群驗證拒絕。 |
| `None` | 明確設定 `result = null`。 | null。 |
| 其他值，包含預設值 `0` | 進入 `default`，設定 `result = null`。 | null。 |

`MemberTier` 若收到 null 的會員卡清單，會在 `typeIds.ForEach()` 拋錯。正常且欄位一致的會員等級 request，會先受到驗證器的非空檢查。空集合、null 與全體會員條件是三種不同結果。

#### 案例一至三：全體會員、會員等級與指定客群

以下都是示例設定，並非真實商店紀錄。只列會員相關欄位，其餘建立活動的必要欄位假設正確提供。

| 活動情境 | Request 設定 | 寫入 `MatchedUserScopes` 的內容 |
| --- | --- | --- |
| 全體會員滿 1,000 元折 100 元 | `TargetMemberTypeDef` 與 `TargetMemberType` 都為 `All`。 | 一個 `AllUserScope`。 |
| 金卡會員滿 1,000 元折 100 元 | 兩個對象欄位都為 `MemberTier`；`CrmShopMemberCardIds = [123]`。 | 一個 Tag 為 `CrmShopMemberCard:123` 的 `TagUserScope`。另寫入會員卡關聯資料。 |
| 指定客群限定活動 | 兩個對象欄位都為 `MemberCollection`；`MemberCollectionId = "audience-001"`，假設該 ID 已存在且通過驗證。 | 一個 Tag 為 `audience-001` 的 `TagUserScope`。 |

#### 案例四：全體會員的當月壽星，滿 1,000 元送 100 點

以 `RewardReachPriceWithPoint2` 為例。會員相關 request 如下：

```json
{
  "TargetMemberTypeDef": "All",
  "TargetMemberType": "All",
  "IsBirthdayMonthEnabled": true
}
```

先由 `All` 建立全體會員條件，再追加壽星 Tag。保存的規則片段如下：

```json
{
  "IsBirthdayMonthEnabled": true,
  "MatchedUserScopes": [
    {
      "UserScopeType": "NineYi.Msa.Promotion.Engine.AllUserScope"
    },
    {
      "UserScopeType": "NineYi.Msa.Tagging.TagUserScope",
      "Tag": "CurrentBirthdayMonth"
    }
  ]
}
```

給點條件中的 `RewardPointRuleList.CrmShopMemberCardId` 設為 `0`，會建立以 `AllUserScope` 為 Key 的給點門檻。當會員具備對應 Tag，且金額、商品與其他條件都符合時，一般會員與金卡會員只要是當月壽星都能給點；非當月壽星會被獨立的壽星檢查拒絕。

#### 案例五：金卡的當月壽星，滿 1,000 元送 100 點

假設金卡會員卡 ID 為 `123`，仍以 `RewardReachPriceWithPoint2` 為例：

```json
{
  "TargetMemberTypeDef": "MemberTier",
  "TargetMemberType": "MemberTier",
  "CrmShopMemberCardIds": [123],
  "IsBirthdayMonthEnabled": true
}
```

先建立會員卡條件，再追加壽星 Tag。保存的規則片段如下：

```json
{
  "IsBirthdayMonthEnabled": true,
  "MatchedUserScopes": [
    {
      "UserScopeType": "NineYi.Msa.Tagging.TagUserScope",
      "Tag": "CrmShopMemberCard:123"
    },
    {
      "UserScopeType": "NineYi.Msa.Tagging.TagUserScope",
      "Tag": "CurrentBirthdayMonth"
    }
  ]
}
```

給點條件的 `CrmShopMemberCardId` 也設成 `123`，因此 `Thresholds` 中只有 `CrmShopMemberCard:123` 對應的給點門檻。

**兩個 Scope 本身不是 AND。**`IsUserScopeMatched()` 使用 `Any()`，清單中任一條件符合即可。金卡與壽星的共同限制，還依賴獨立的壽星開關檢查，以及給點門檻的會員 Tag 對應。

1. **會員範圍** `MatchedUserScopes` 中任一條件符合即可。銀卡壽星也可能因壽星 Tag 通過這一關。

1. **壽星資格** `IsBirthdayMonthEnabled = true` 時，會員必須具有 `CurrentBirthdayMonth` Tag。金卡非壽星在這裡被拒絕。

1. **給點門檻** `GetThreshold()` 以會員 Tag 尋找 `Thresholds`。本例只設定金卡門檻，銀卡壽星找不到對應門檻，仍無法給點。找到門檻後還要符合金額等活動條件。

| 會員 | 當月壽星 | 判斷結果（其餘活動條件均符合） |
| --- | --- | --- |
| 金卡 `123` | 是 | 會員範圍、壽星資格與金卡給點門檻都符合，可以給點。 |
| 金卡 `123` | 否 | 未通過獨立的壽星檢查。 |
| 銀卡 `456` | 是 | 可以因壽星 Tag 通過會員範圍，但找不到銀卡給點門檻，不給點。 |
| 銀卡 `456` | 否 | 不符合活動。 |

Create 追加壽星 Tag 的活動類型為 `RewardReachPriceWithPoint2`、`RewardReachPriceWithRatePoint2` 與 `RewardReachPriceWithCoupon`。不能直接把這套建立行為套用到所有折扣活動。

#### 案例六與七：漏傳活動對象，或明確指定 None

| Request 情境 | 建構結果 | 後續影響 |
| --- | --- | --- |
| 只傳 `TargetMemberTypeDef = "All"`，漏傳 `TargetMemberType` | Enum 保留 `0`，走 `default` 得到 null。 | 一般活動若通過其餘驗證與建立步驟，可能保存 null 規則；主檔對象仍可能顯示 `All`。 |
| 兩個活動對象欄位都指定 `None` | 明確回傳 null。 | `None` 並不等於全體會員；部分活動會先被額外驗證拒絕。 |
| 支援壽星的活動漏傳 `TargetMemberType`，並啟用壽星開關 | 基本會員範圍先得到 null。 | 若執行到追加壽星的 `.Add()`，會因 null 拋錯，本次 SQL 建立交易無法正常完成。 |

**壽星追加不會修復 null。**`SetBirthdayMonthMatchedUserScopes()` 只對既有清單執行 `.Add()`，沒有初始化空清單。正常的全體會員或會員等級設定會先建立清單；若前一步已是 null，錯誤會提早在 Create 階段發生。

#### 為什麼建立時能載入，計算時卻出錯？

`SerializeRule()` 的檢查是序列化後嘗試 `engine.LoadRules()`。規則成功載入，不代表已執行活動排序。引擎的 `ProcessPromotion()` 先篩選啟用與日期，再讀取 `DynamicPriority` 排序，之後才進行活動匹配。

當符合排序前置條件的規則包含 `MatchedUserScopes = null`，`GetUserScopePriority()` 對 null 呼叫 `OfType<AllUserScope>()`，便會拋出 `ArgumentNullException`，參數為 `source`。是否適用會員的後續判斷還來不及執行。

這些是可由程式確認的建構與失敗路徑。要判定特定異常活動的原因，仍需比對原始 Create request 的兩個活動對象欄位、SQL 的 `PromotionEngine_Rule`，以及實際命中的快取內容。

#### 程式碼對照

分析依據為 WebAPI 本機版本 `fcba6180` 與引擎版本 `d852c3c`；以下為對應程式位置。

| 階段 | 程式位置 |
| --- | --- |
| Request 與驗證 | `PromotionBaseEntity.cs:62`、`PromotionBaseValidator.cs:182`、`CreatePromotionRequestEntityValidator.cs:243`。 |
| API 入口 | `PromotionEngineController.cs:219`，先驗證再呼叫服務。 |
| SQL Create | `PromotionEngineService.cs:587`；會員範圍設定在 `650`，壽星分支在 `655`，規則組合在 `727`。 |
| 基本範圍與壽星追加 | `PromotionEngineRuleBaseService.cs:226`、`:234`、`:1007`。 |
| 規則 JSON | `PromotionEngineRuleBaseService.cs:370`、`PromotionEngineHelper.cs:77`。 |
| 給點門檻 | `RewardReachPriceWithPoint2RuleService.cs:64`，以會員卡 ID 產生門檻 Key。 |
| 引擎範圍與壽星判斷 | `IBasicUserScope.cs:40`、`PromotionCommonRuleBase.cs:152`。 |
| 給點資格與門檻查找 | `RewardReachPriceWithPoint2.cs:55`、`:175`。 |
| 排序失敗位置 | `PromotionEngine.cs:128`、`IBasicUserScope.cs:32`。 |

## 失敗路徑與程式證據

### Runtime failure path

1. `ProcessPromotion()`：取得活動規則集合
2. `OrderBy(x => x.DynamicPriority)`：開始計算活動排序值
3. `GetUserScopePriority()`：`MatchedUserScopes` 是 null

```csharp
public static int GetUserScopePriority(IBasicUserScope target) =>
    target.MatchedUserScopes.OfType<AllUserScope>().Any()
        ? 9000
        : 1000;
```

| 位置 | 行為 | 判定 |
| --- | --- | --- |
| `CalculateController.CartCalculateAsync` | 接收 Cart request | 請求已進入 Promotion Frontend |
| `PromotionRuleRepository.GetMatchedSalepageCollectionAsync` | 以 `SalepageId` 呼叫 Collection API | 取得候選活動的流程位置 |
| `PromotionEngine.ProcessPromotion` | `OrderBy` 計算 `DynamicPriority` | 例外觸發點 |
| `IBasicUserScope.GetUserScopePriority` | 對 `MatchedUserScopes` 呼叫 `OfType` | 直接拋出 `ArgumentNullException: Parameter 'source'` |

根因方向：至少一筆活動規則的會員範圍資料在原始資料、反序列化、Mapping 或快取版本中成為 `null`。目前堆疊沒有指向 Cart 或 Collection API 查詢失敗。

## 查核與修正方向

### 查核重點

- 鎖定活動：從錯誤 trace 取得 `ShopId` 與 `SalepageId`，再查 Collection mapping 的 `PromotionEngineId`
- 比對資料版本：依實際 key 比對 Redis value，並回查 WebStoreDB 的 `PromotionEngine_Rule` 或 MongoDB 的 `UserScope`
- 避免整體失敗：補齊活動資料，或對 `MatchedUserScopes` 建立明確的 null-safe 預設行為並補單元測試

### 查核步驟

1. 用 `CartCalculate:SalepageCollection-20230801:{shopId}:{salepageId}` 找到商品匹配結果
2. 用 `CartCalculate:PromotionEngine-20260317:{shopId}:{promotionEngineId}` 找到活動規則快取
3. 確認原始活動資料是否缺少會員範圍
4. 確認反序列化後是否由空資料變成 `null`
5. 確認修正後單筆異常活動不會使整個 `Cart Calculate` request 失敗

架構表達參考：

- [C4 Model Container Diagram](https://c4model.com/diagrams/container)
- [C4 Dynamic Diagram](https://c4model.com/diagrams/dynamic)
- [Microsoft Cache Aside Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
- [AWS Architecture Reference](https://aws.amazon.com/architecture/)

本文件採用系統邊界、容器責任、標註協定與資料內容的箭頭、Cache Aside 命中分支，以及獨立錯誤路徑，對應業界常見的 C4 與雲端參考架構畫法。
