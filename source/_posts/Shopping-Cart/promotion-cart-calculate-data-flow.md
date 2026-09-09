---
title: Promotion Cart Calculate
date: 2026-09-07 10:30:56
categories: Shopping-Cart
tags:
toc:
toc_number:
comments:
---

{% tabs promotion-cart-calculate-data-flow %}

<!-- tab 架構摘要 -->

<style>
.pc-article{--pc-ink:#243449;--pc-muted:#526379;--pc-line:#dce4ed;--pc-panel:#f5f8fc;--pc-blue:#245eaa;--pc-green:#167561;--pc-red:#b42332;color:var(--pc-ink);line-height:1.85;overflow-wrap:anywhere}
.pc-article *{box-sizing:border-box}
.pc-article h2{font-size:1.55em;margin:2.6em 0 1em;padding-bottom:.5em;border-bottom:1px solid var(--pc-line);color:var(--pc-ink)}
.pc-article > h2:first-child{margin-top:0}
.pc-article h3{font-size:1.15em;margin:1.8em 0 .7em;color:var(--pc-ink)}
.pc-hero{padding:26px 28px;border:1px solid var(--pc-line);border-top:4px solid var(--pc-blue);border-radius:12px;background:var(--pc-panel);margin:12px 0 24px}
.pc-eyebrow{color:var(--pc-blue);font-size:12px;font-weight:700;letter-spacing:.12em}
.pc-hero .pc-headline{font-size:clamp(23px,3vw,32px);font-weight:700;line-height:1.4;margin:12px 0;color:var(--pc-ink)}
.pc-hero p{margin:10px 0}
.pc-intro{color:var(--pc-muted)}
.pc-note{padding:14px 18px;margin:20px 0;border-left:4px solid var(--pc-blue);background:var(--pc-panel);border-radius:0 8px 8px 0}
.pc-note.pc-cache{border-left-color:var(--pc-green)}
.pc-note.pc-error{border-left-color:var(--pc-red)}
.pc-note strong{color:var(--pc-ink)}
.pc-figure{margin:24px 0;border:1px solid var(--pc-line);border-radius:10px;overflow:hidden}
.pc-figure figcaption{padding:10px 16px;background:var(--pc-panel);color:var(--pc-muted);font-size:13px;border-top:1px solid var(--pc-line)}
.pc-diagram{overflow-x:auto;padding:20px 12px;background:var(--pc-diagram-bg,#fff);color:#243449}
.pc-diagram:focus-visible{outline:3px solid var(--pc-blue);outline-offset:-3px}
.pc-diagram svg{display:block;margin:0 auto;height:auto;max-width:none!important;width:100%;min-width:660px}
.pc-diagram.pc-sequence svg{min-width:740px}
.pc-diagram .mermaid-wrap{margin:0}

.pc-table-scroll{overflow-x:auto;margin:20px 0}
.pc-table-scroll table{width:100%;margin:0;table-layout:fixed;font-size:14px}
.pc-table-scroll th{background:var(--pc-panel);color:var(--pc-ink)}
.pc-table-scroll th,.pc-table-scroll td{padding:12px;border:1px solid var(--pc-line);vertical-align:top;white-space:normal;overflow-wrap:anywhere}
.pc-key{border:1px solid var(--pc-line);border-radius:8px;padding:16px 18px;margin:16px 0}
.pc-key .pc-key-value{display:block;padding:12px;margin:10px 0;background:var(--pc-panel);color:var(--pc-blue);white-space:normal;overflow-wrap:anywhere;font-size:14px}
.pc-key p{margin:8px 0;font-size:14px;color:var(--pc-muted)}
.pc-article code{overflow-wrap:anywhere}
.pc-evidence{border:1px solid var(--pc-line);border-radius:8px;padding:14px 18px;margin:20px 0}
.pc-evidence summary{cursor:pointer;font-weight:700;color:var(--pc-blue)}
.pc-evidence summary:focus-visible{outline:2px solid var(--pc-blue);outline-offset:4px}
.pc-steps{padding-left:25px}
.pc-steps li{padding:5px 0 10px 5px}
.pc-steps li::marker{color:var(--pc-blue);font-weight:700}
.pc-steps p{margin:5px 0}
.pc-small{font-size:13px;color:var(--pc-muted)}
[data-theme="dark"] .pc-article{--pc-ink:#e1e9f3;--pc-muted:#b5c2d3;--pc-line:#405066;--pc-panel:#232e3d;--pc-blue:#91bfff;--pc-green:#68cfb1;--pc-red:#ff9ca8;--pc-diagram-bg:#202631}
@media(max-width:600px){ .pc-hero{padding:20px 16px}.pc-article h2{font-size:1.35em}.pc-note,.pc-key{padding:12px}.pc-table-scroll table{min-width:520px}}
@media print{ .pc-diagram svg,.pc-diagram.pc-sequence svg{min-width:0;max-width:100%!important}.pc-diagram{overflow:visible}.pc-figure{break-inside:avoid}}
</style>

<div class="pc-article">

<div class="pc-hero">
<div class="pc-eyebrow">PROMOTION / DATA FLOW / INCIDENT TRACE</div>
<p class="pc-headline">Salepage-Promotion-Flow</p>
<p class="pc-intro">沿著一次 <code>POST /api/cart-calculate</code>，理解商品如何找到候選活動、規則從哪裡載入，以及促銷排序為什麼會因會員範圍資料而失敗。</p>
<p><strong>先記住三件事：</strong>Cart 提供購物車資料；Collection API 提供商品與活動的關聯；完整活動規則由 Repository 載入。</p>
</div>

<div class="pc-note pc-error"><strong>本次失敗焦點</strong><br><code>MatchedUserScopes</code> 為 <code>null</code>，<code>GetUserScopePriority()</code> 對它執行 LINQ 時拋出 <code>ArgumentNullException</code>。這是本文記錄的直接失敗位置；資料在哪一層變成 null，仍需比對原始資料、轉換結果與快取。</div>

</div>

<!-- endtab -->

<!-- tab 系統關係 -->

<div class="pc-article">

<h2 id="pc-system">系統關係：誰負責哪一段？</h2>

下圖區分 Promotion Frontend 的內部元件與外部支援系統。箭頭表示呼叫或資料存取方向；實際執行順序在「請求與資料流」分頁用時序圖說明。

<figure class="pc-figure">
<div class="pc-diagram" tabindex="0" role="region" aria-label="系統關係圖，可橫向捲動">

{% mermaid %}
flowchart TB
    Cart["Cart Service"]
    subgraph Frontend["Promotion Frontend · 內部元件"]
        Controller["CalculateController"]
        Service["CalculateService"]
        Repo["PromotionRuleRepository"]
        Engine["Promotion Engine"]
        Controller -->|協調計算| Service
        Service -->|取得候選活動與規則| Repo
        Service -->|匹配與排序| Engine
    end
    Cart -->|POST /api/cart-calculate| Controller
    Repo -->|商品與活動關聯| Collection["Collection API"]
    Repo -->|讀取與回填快取| Redis[("Redis")]
    Repo -->|傳統活動規則| SQL[("WebStoreDB")]
    Repo -->|Mongo 活動規則| Mongo[("MongoDB PromotionDB")]
    classDef service fill:#edf4ff,stroke:#517ab0,color:#203c60
    classDef cache fill:#edf9f4,stroke:#32856b,color:#175440
    classDef store fill:#f3f4f6,stroke:#7d8998,color:#334155
    class Cart,Controller,Service,Repo,Engine,Collection service
    class Redis cache
    class SQL,Mongo store
{% endmermaid %}

</div>
<figcaption>圖 1｜藍色：服務與內部元件；綠色：快取；灰色：持久化資料源。</figcaption>
</figure>

<div class="pc-table-scroll" tabindex="0" role="region" aria-label="元件責任表">

| 元件 | 主要責任 |
| --- | --- |
| Cart Service | 組裝商品、會員、付款與配送資料，送出計算請求。 |
| CalculateController / CalculateService | 接收請求、協調計算流程與包裝回應。 |
| PromotionRuleRepository | 取得商品匹配結果、找候選活動，並載入完整規則。 |
| Collection API | 提供商品頁與集合、活動的關聯。 |
| Promotion Engine | 執行規則匹配與 `DynamicPriority` 排序。 |

</div>

<div class="pc-note"><strong>圖的層級：</strong>這是系統邊界與元件關係示意。Controller、Service、Repository 是應用程式內部元件，因此不將這張圖稱為嚴格的 C4 Container Diagram。</div>

</div>

<!-- endtab -->

<!-- tab 請求與資料流 -->

<div class="pc-article">

<h2 id="pc-request">請求時序：一次計算如何完成？</h2>

先看正常完成的主流程。時序圖將 Controller 與 Service 合併為「Frontend」，並以業務動作標註呼叫；規則載入的快取細節在「活動規則來源」分頁展開。

<figure class="pc-figure">
<div class="pc-diagram pc-sequence" tabindex="0" role="region" aria-label="購物車促銷計算時序圖，可橫向捲動">

{% mermaid %}
%%{init: {'sequence': {'actorMargin': 24, 'width': 120, 'mirrorActors': false, 'diagramMarginX': 16}}}%%
sequenceDiagram
    autonumber
    participant Cart as Cart Service
    participant Front as Frontend
    participant Repo as Repository
    participant Collection as Collection API
    participant Engine as Promotion Engine
    Cart->>Front: POST /api/cart-calculate
    Note over Cart,Front: 商品、會員、付款與配送資料
    Front->>Front: 從 SalepageSkuList 取得 SalepageIds
    Front->>Repo: 取得商品匹配結果
    alt 商品匹配快取命中
        Note over Repo: 使用 Redis 中的匹配結果
    else 商品匹配快取未命中
        Repo->>Collection: api/salepage-collections:match
        Collection-->>Repo: 商品集合與候選活動關聯
        Note over Repo: 回填商品匹配快取
    end
    Repo-->>Front: 候選活動識別值
    Front->>Repo: 載入候選活動規則
    Note over Repo: Redis 或資料庫，詳見圖 3
    Repo-->>Front: 完整規則物件
    Front->>Engine: 執行促銷匹配與排序
    Note over Engine: 本次例外發生在排序期間，詳見「失敗路徑」分頁
    Engine-->>Front: 促銷計算結果（成功時）
    Front-->>Cart: 包裝後的計算回應（成功時）
{% endmermaid %}

</div>
<figcaption>圖 2｜正常完成路徑。實線表示呼叫，虛線表示回傳；alt 區塊中的兩條路徑擇一執行。圖中不推定錯誤時的 HTTP 狀態碼。</figcaption>
</figure>

<h3>把資料識別值串起來</h3>

**購物車商品頁 → 商品集合 → 候選活動 → 完整活動規則**

<div class="pc-table-scroll" tabindex="0" role="region" aria-label="資料識別值對照表">

| 識別值 | 在流程中的意義 |
| --- | --- |
| `SalepageId` | 來自購物車的商品頁 ID，用於查詢匹配集合。 |
| `SalepageCollectionId` | 商品所屬集合，用來串接活動關聯。 |
| `PromotionEngineId` | 候選活動識別值，用於載入活動規則。 |

</div>

<div class="pc-note pc-cache"><strong>關聯與規則分開看：</strong>找到候選活動，不代表已取得完整規則。Collection API 提供關聯；Repository 再向 Redis、WebStoreDB 或 MongoDB 取得規則內容。</div>

</div>

<!-- endtab -->

<!-- tab 活動規則來源 -->

<div class="pc-article">

<h2 id="pc-cache">快取分支：命中就使用，未命中才查來源</h2>

本流程採用 Cache Aside。以下聚焦「活動規則快取」：HIT 與 MISS 是互斥分支，命中後不會接著執行資料庫 fallback。

<figure class="pc-figure">
<div class="pc-diagram pc-sequence" tabindex="0" role="region" aria-label="活動規則快取時序圖，可橫向捲動">

{% mermaid %}
%%{init: {'sequence': {'actorMargin': 24, 'width': 120, 'mirrorActors': false, 'diagramMarginX': 16}}}%%
sequenceDiagram
    autonumber
    participant Repo as Repository
    participant Redis as Redis
    participant SQL as WebStoreDB
    participant Mongo as MongoDB
    Repo->>Redis: 以 ShopId 與 PromotionEngineId 查規則
    Redis-->>Repo: 快取查詢結果
    alt HIT：規則已存在
        Note over Repo: 直接使用快取規則，跳過資料庫
    else MISS：規則不存在
        alt 傳統活動或低於 Mongo 分界
            Repo->>SQL: 讀取 PromotionEngine 與規則 JSON
            SQL-->>Repo: 活動規則資料
        else 活動 ID 大於等於 Mongo 分界
            Repo->>Mongo: 讀取 PromotionRule 與相關設定
            Mongo-->>Repo: 活動規則資料
        end
        Repo->>Redis: 回填活動規則快取
    end
    Note over Repo: 將取得的規則交給後續促銷計算
{% endmermaid %}

</div>
<figcaption>圖 3｜活動規則的 HIT／MISS 與資料庫選擇。以單筆規則示意，省略批次處理細節；Mongo 分界設定為 MongoDB.PromotionId.Divide。</figcaption>
</figure>

<div class="pc-table-scroll" tabindex="0" role="region" aria-label="規則來源對照表">

| 使用條件 | 資料來源 | 查核內容 |
| --- | --- | --- |
| 規則快取命中 | Redis | 快取中的活動規則，可能仍是舊版本。 |
| 未命中，且為傳統活動或低於 Mongo 分界 | WebStoreDB | `PromotionEngine` 的基本欄位與 `PromotionEngine_Rule` JSON。 |
| 未命中，且活動 ID 大於等於 Mongo 分界 | MongoDB PromotionDB | `PromotionRule`、`PromotionRuleSetting`，包含 `UserScope`、`ProductScope` 等設定。 |

</div>

<div class="pc-note pc-cache"><strong>排查時要比對兩份資料：</strong>資料庫已修正，不代表命中的 Redis value 同步更新。應同時比對快取內容與持久化來源，再判斷規則實際使用的版本。</div>

</div>

<!-- endtab -->

<!-- tab Redis Key 與快取分支 -->

<div class="pc-article">

<h2 id="pc-keys">Redis Key：用哪個 ID 找哪份資料？</h2>

兩種 Key 都包含 `ShopId`，但最後一段分別是商品頁 ID 與活動 ID。日期字串是程式內的 **Key 版本**，不是資料產生日期。

<div class="pc-key">
<strong>商品頁匹配結果</strong>
<code class="pc-key-value">CartCalculate:SalepageCollection-20230801:{shopId}:{salepageId}</code>
<p>用購物車的 <code>salepageId</code> 查商品匹配結果，再追到候選活動。</p>
<p>程式位置：<code>PromotionRuleRepository.GetMatchedSalepageCollectionAsync</code> 的 <code>FormatCacheKey</code> 區域。</p>
</div>

<div class="pc-key">
<strong>活動規則資料</strong>
<code class="pc-key-value">CartCalculate:PromotionEngine-20260317:{shopId}:{promotionEngineId}</code>
<p>用候選活動的 <code>promotionEngineId</code> 查規則快取，重點檢查會員範圍相關資料。</p>
<p>程式位置：<code>PromotionRuleRepository.GetPromotionEngineDataListAsync</code> 的 <code>FormatCacheKey</code> 區域。</p>
</div>

</div>

<!-- endtab -->

<!-- tab 活動建立與會員範圍 -->

<div class="pc-article">

<h2 id="pc-create-user-scope">建立活動時，會員範圍如何寫進規則？</h2>

建立「全體會員適用」「金卡限定」或「當月壽星限定」活動時，WebAPI 會先把會員條件組成清單，再與折扣或回饋條件一起存入 SQL。購物車計算讀取的是這份已保存的規則。

本節聚焦 <code>POST /api/promotion-rules/create</code> 的 SQL 建立流程。<code>MatchedUserScopes</code> 由 request 建構，最後寫進 <code>PromotionEngine_Rule</code> 的 JSON。

<h3>兩個相似欄位，負責不同事情</h3>

<div class="pc-table-scroll" tabindex="0" role="region" aria-label="建立活動的會員欄位">

| Request 欄位 | 用途 |
| --- | --- |
| `TargetMemberTypeDef` | 字串，供基本驗證、主檔活動對象設定等邏輯使用。 |
| `TargetMemberType` | Enum，實際決定 `MatchedUserScopes` 的建立分支。 |
| `CrmShopMemberCardIds` | 指定會員等級時，轉成各會員卡的 Tag。 |
| `MemberCollectionId` | 指定客群時，作為客群 Tag。 |
| `IsBirthdayMonthEnabled` | 壽星開關；支援的回饋活動會追加壽星 Tag，並把開關存進規則。 |

</div>

<div class="pc-note pc-error"><strong>兩個活動對象欄位不會自動同步。</strong><code>TargetMemberTypeDef</code> 與 <code>TargetMemberType</code> 是獨立屬性。只傳前者為 <code>All</code>，後者仍可能保留預設值 <code>0</code>，使會員範圍建構得到 <code>null</code>。</div>

<h3>SQL Create 的建立順序</h3>

<ol class="pc-steps">
<li><strong>驗證 request</strong><p><code>PromotionEngineController.Create()</code> 先執行 <code>ValidateAndThrowAsync()</code>。基本驗證檢查 <code>TargetMemberTypeDef</code> 非空且為合法 enum 名稱；部分活動另有限制，但沒有全面檢查兩個活動對象欄位一致。</p></li>
<li><strong>選擇規則服務、整理客群設定</strong><p><code>PromotionEngineService.Create()</code> 依 <code>TypeDef</code> 選出 <code>ruleService</code>。<code>SetMemberCollectionAsync()</code> 依活動種類與商店開關調整客群 ID，例如部分全體會員活動設成 <code>-1</code>；這一步不會將字串欄位同步到 enum 欄位。</p></li>
<li><strong>建立 SQL 活動資料</strong><p>開啟 <code>TransactionScope</code>，建立主檔、設定與商品範圍。若對象是 <code>MemberTier</code>，另外將活動與會員卡的關聯寫入 <code>PromotionEngineMemberTier</code>。</p></li>
<li><strong>在記憶體建立會員範圍</strong><p><code>SetMatchedUserScopes(entity.TargetMemberType, entity.CrmShopMemberCardIds, entity.MemberCollectionId)</code> 呼叫 <code>GetUserScopes()</code>，將結果放進 <code>_matchedUserScopes</code>。這裡直接使用 request，沒有回查剛寫入的會員等級關聯表。</p></li>
<li><strong>符合壽星設定時追加條件</strong><p>支援的回饋活動啟用壽星開關後，會確認或建立壽星客群，再以 <code>SetBirthdayMonthMatchedUserScopes()</code> 對既有清單追加 <code>CurrentBirthdayMonth</code>。</p></li>
<li><strong>組合規則並保存</strong><p>各活動的 <code>GetPromotionEngineRule()</code> 組合折扣或回饋條件。共用的 <code>GetRuleObject()</code> 設定 <code>result.MatchedUserScopes = _matchedUserScopes</code>，序列化後嘗試 <code>engine.LoadRules()</code>，再更新 <code>PromotionEngine_Rule</code>。完成其餘建立步驟後提交交易。</p></li>
</ol>

<h3>會員範圍的建立分支</h3>

<div class="pc-table-scroll" tabindex="0" role="region" aria-label="會員範圍建構分支">

| `TargetMemberType` | 建立內容 | 結果 |
| --- | --- | --- |
| `All` | 加入一個 `AllUserScope`。 | 非 null，代表全體會員。 |
| `MemberTier` | 每個會員卡 ID 建立 `TagUserScope`，Tag 為 `CrmShopMemberCard:{id}`。 | 有值為 Tag 清單；空清單得到 `[]`。 |
| `MemberCollection`，ID 有值 | 建立一個 `TagUserScope`，Tag 直接使用客群 ID。 | 非 null，不展開個別會員名單。 |
| `MemberCollection`，ID 為 null 或空字串 | 不加入任何元素。 | 空集合 `[]`；部分活動會先被客群驗證拒絕。 |
| `None` | 明確設定 `result = null`。 | null。 |
| 其他值，包含預設值 `0` | 進入 `default`，設定 `result = null`。 | null。 |

</div>

<code>MemberTier</code> 若收到 null 的會員卡清單，會在 <code>typeIds.ForEach()</code> 拋錯。正常且欄位一致的會員等級 request，會先受到驗證器的非空檢查。空集合、null 與全體會員條件是三種不同結果。

<h3>案例一至三：全體會員、會員等級與指定客群</h3>

以下都是示例設定，並非真實商店紀錄。只列會員相關欄位，其餘建立活動的必要欄位假設正確提供。

<div class="pc-table-scroll" tabindex="0" role="region" aria-label="基本會員範圍案例">

| 活動情境 | Request 設定 | 寫入 `MatchedUserScopes` 的內容 |
| --- | --- | --- |
| 全體會員滿 1,000 元折 100 元 | `TargetMemberTypeDef` 與 `TargetMemberType` 都為 `All`。 | 一個 `AllUserScope`。 |
| 金卡會員滿 1,000 元折 100 元 | 兩個對象欄位都為 `MemberTier`；`CrmShopMemberCardIds = [123]`。 | 一個 Tag 為 `CrmShopMemberCard:123` 的 `TagUserScope`。另寫入會員卡關聯資料。 |
| 指定客群限定活動 | 兩個對象欄位都為 `MemberCollection`；`MemberCollectionId = "audience-001"`，假設該 ID 已存在且通過驗證。 | 一個 Tag 為 `audience-001` 的 `TagUserScope`。 |

</div>

<h3>案例四：全體會員的當月壽星，滿 1,000 元送 100 點</h3>

以 <code>RewardReachPriceWithPoint2</code> 為例。會員相關 request 如下：

```json
{
  "TargetMemberTypeDef": "All",
  "TargetMemberType": "All",
  "IsBirthdayMonthEnabled": true
}
```

先由 <code>All</code> 建立全體會員條件，再追加壽星 Tag。保存的規則片段如下：

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

給點條件中的 <code>RewardPointRuleList.CrmShopMemberCardId</code> 設為 <code>0</code>，會建立以 <code>AllUserScope</code> 為 Key 的給點門檻。當會員具備對應 Tag，且金額、商品與其他條件都符合時，一般會員與金卡會員只要是當月壽星都能給點；非當月壽星會被獨立的壽星檢查拒絕。

<h3>案例五：金卡的當月壽星，滿 1,000 元送 100 點</h3>

假設金卡會員卡 ID 為 <code>123</code>，仍以 <code>RewardReachPriceWithPoint2</code> 為例：

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

給點條件的 <code>CrmShopMemberCardId</code> 也設成 <code>123</code>，因此 <code>Thresholds</code> 中只有 <code>CrmShopMemberCard:123</code> 對應的給點門檻。

<div class="pc-note"><strong>兩個 Scope 本身不是 AND。</strong><code>IsUserScopeMatched()</code> 使用 <code>Any()</code>，清單中任一條件符合即可。金卡與壽星的共同限制，還依賴獨立的壽星開關檢查，以及給點門檻的會員 Tag 對應。</div>

<ol class="pc-steps">
<li><strong>會員範圍</strong><p><code>MatchedUserScopes</code> 中任一條件符合即可。銀卡壽星也可能因壽星 Tag 通過這一關。</p></li>
<li><strong>壽星資格</strong><p><code>IsBirthdayMonthEnabled = true</code> 時，會員必須具有 <code>CurrentBirthdayMonth</code> Tag。金卡非壽星在這裡被拒絕。</p></li>
<li><strong>給點門檻</strong><p><code>GetThreshold()</code> 以會員 Tag 尋找 <code>Thresholds</code>。本例只設定金卡門檻，銀卡壽星找不到對應門檻，仍無法給點。找到門檻後還要符合金額等活動條件。</p></li>
</ol>

<div class="pc-table-scroll" tabindex="0" role="region" aria-label="金卡壽星活動判斷案例">

| 會員 | 當月壽星 | 判斷結果（其餘活動條件均符合） |
| --- | --- | --- |
| 金卡 `123` | 是 | 會員範圍、壽星資格與金卡給點門檻都符合，可以給點。 |
| 金卡 `123` | 否 | 未通過獨立的壽星檢查。 |
| 銀卡 `456` | 是 | 可以因壽星 Tag 通過會員範圍，但找不到銀卡給點門檻，不給點。 |
| 銀卡 `456` | 否 | 不符合活動。 |

</div>

Create 追加壽星 Tag 的活動類型為 <code>RewardReachPriceWithPoint2</code>、<code>RewardReachPriceWithRatePoint2</code> 與 <code>RewardReachPriceWithCoupon</code>。不能直接把這套建立行為套用到所有折扣活動。

<h3>案例六與七：漏傳活動對象，或明確指定 None</h3>

<div class="pc-table-scroll" tabindex="0" role="region" aria-label="會員範圍 null 的案例">

| Request 情境 | 建構結果 | 後續影響 |
| --- | --- | --- |
| 只傳 `TargetMemberTypeDef = "All"`，漏傳 `TargetMemberType` | Enum 保留 `0`，走 `default` 得到 null。 | 一般活動若通過其餘驗證與建立步驟，可能保存 null 規則；主檔對象仍可能顯示 `All`。 |
| 兩個活動對象欄位都指定 `None` | 明確回傳 null。 | `None` 並不等於全體會員；部分活動會先被額外驗證拒絕。 |
| 支援壽星的活動漏傳 `TargetMemberType`，並啟用壽星開關 | 基本會員範圍先得到 null。 | 若執行到追加壽星的 `.Add()`，會因 null 拋錯，本次 SQL 建立交易無法正常完成。 |

</div>

<div class="pc-note pc-error"><strong>壽星追加不會修復 null。</strong><code>SetBirthdayMonthMatchedUserScopes()</code> 只對既有清單執行 <code>.Add()</code>，沒有初始化空清單。正常的全體會員或會員等級設定會先建立清單；若前一步已是 null，錯誤會提早在 Create 階段發生。</div>

<h3>為什麼建立時能載入，計算時卻出錯？</h3>

<code>SerializeRule()</code> 的檢查是序列化後嘗試 <code>engine.LoadRules()</code>。規則成功載入，不代表已執行活動排序。引擎的 <code>ProcessPromotion()</code> 先篩選啟用與日期，再讀取 <code>DynamicPriority</code> 排序，之後才進行活動匹配。

當符合排序前置條件的規則包含 <code>MatchedUserScopes = null</code>，<code>GetUserScopePriority()</code> 對 null 呼叫 <code>OfType&lt;AllUserScope&gt;()</code>，便會拋出 <code>ArgumentNullException</code>，參數為 <code>source</code>。是否適用會員的後續判斷還來不及執行。

這些是可由程式確認的建構與失敗路徑。要判定特定異常活動的原因，仍需比對原始 Create request 的兩個活動對象欄位、SQL 的 <code>PromotionEngine_Rule</code>，以及實際命中的快取內容。

<details class="pc-evidence">
<summary>程式碼對照</summary>

分析依據為 WebAPI 本機版本 <code>fcba6180</code> 與引擎版本 <code>d852c3c</code>；以下為對應程式位置。

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

</details>

</div>

<!-- endtab -->

<!-- tab 失敗路徑 -->

<div class="pc-article">

<h2 id="pc-failure">失敗位置：取得活動之後，排序期間出錯</h2>

本文記錄的失敗呼叫鏈如下。紅色僅標示直接拋出例外的位置；前面的節點用來交代例外如何被觸發。

<figure class="pc-figure">
<div class="pc-diagram" tabindex="0" role="region" aria-label="促銷排序失敗路徑圖，可橫向捲動">

{% mermaid %}
flowchart TB
    Process["ProcessPromotion()"] --> Sort["OrderBy(x => x.DynamicPriority)"]
    Sort --> Priority["GetUserScopePriority()"]
    Priority --> Null["MatchedUserScopes 為 null"]
    Null --> Error["OfType 呼叫拋出 ArgumentNullException<br/>Parameter: source"]
    classDef normal fill:#edf4ff,stroke:#517ab0,color:#203c60
    classDef failure fill:#fff0f1,stroke:#b42332,color:#8e1b28
    class Process,Sort,Priority,Null normal
    class Error failure
{% endmermaid %}

</div>
<figcaption>圖 4｜從規則排序追到 LINQ 的 null 集合來源。這張圖說明失敗位置，不代表已找到資料變成 null 的原因。</figcaption>
</figure>

<details class="pc-evidence" open>
<summary>程式證據：GetUserScopePriority()</summary>

```csharp
public static int GetUserScopePriority(IBasicUserScope target) =>
    target.MatchedUserScopes.OfType<AllUserScope>().Any()
        ? 9000
        : 1000;
```

`OfType` 的來源集合是 `MatchedUserScopes`。當來源為 `null`，程式會在這裡拋出例外，尚未完成後續的 `Any()` 與優先序判斷。

</details>

<div class="pc-table-scroll" tabindex="0" role="region" aria-label="失敗位置與判定表">

| 位置 | 本文記錄的行為與判定 |
| --- | --- |
| `CalculateController.CartCalculateAsync` | 接收 Cart request；請求已進入 Promotion Frontend。 |
| `PromotionRuleRepository.GetMatchedSalepageCollectionAsync` | 取得商品匹配結果的流程位置；未命中快取時呼叫 Collection API。 |
| `PromotionEngine.ProcessPromotion` | `OrderBy` 求取 `DynamicPriority`，觸發失敗呼叫鏈。 |
| `IBasicUserScope.GetUserScopePriority` | 對 null 的 `MatchedUserScopes` 呼叫 `OfType`，直接拋出例外。 |

</div>

<div class="pc-note pc-error"><strong>已知：</strong>失敗發生在會員範圍優先序計算。<br><strong>待查：</strong>null 來自原始規則、反序列化、Mapping，或舊版快取。本文記錄的堆疊沒有指向 Cart 或 Collection API 查詢失敗，也不能僅憑此排除所有上游資料問題。</div>

</div>

<!-- endtab -->

<!-- tab 查核與修正 -->

<div class="pc-article">

<h2 id="pc-checklist">查核與修正：沿著同一筆活動追到底</h2>

<ol class="pc-steps">
<li><strong>鎖定請求與商店</strong><p>從錯誤 trace 取得 <code>ShopId</code> 與 <code>SalepageId</code>，確保後續查詢對應同一筆請求與商店。</p></li>
<li><strong>找到候選活動</strong><p>使用商品頁匹配 Key，檢查集合關聯與 <code>PromotionEngineId</code>；保留查到的匹配結果。</p></li>
<li><strong>比對快取與原始規則</strong><p>使用活動規則 Key 取得 Redis value，再依活動 ID 分界回查 WebStoreDB 或 MongoDB，確認會員範圍欄位是否缺少或為 null。</p></li>
<li><strong>追蹤資料轉換</strong><p>比對反序列化與 Mapping 前後的物件，找出 <code>MatchedUserScopes</code> 首次成為 null 的位置。</p></li>
<li><strong>決定修正語意並驗證</strong><p>依查核結果補齊資料、處理快取版本，或定義明確的 null 處理策略。確認單筆異常活動的處置符合業務規則，並驗證正常活動仍可完成計算。</p></li>
</ol>

<div class="pc-note"><strong>修正不能只讓例外消失：</strong>會員範圍缺失時要拒絕規則、略過活動，還是採用預設行為，需先確認業務語意；不能直接把 null 視為「適用所有會員」。</div>

<details class="pc-evidence">
<summary>展開修正後的驗證項目</summary>

- Redis HIT：使用快取內容，不查持久化來源。
- Redis MISS：依活動 ID 選擇來源，取得資料後回填快取。
- 同一活動的快取與來源資料不一致：能辨認實際使用的版本。
- `MatchedUserScopes` 分別為 null、空集合及正常集合：結果符合明確的業務規則。
- 單筆異常活動：整筆請求的結果與錯誤處置符合預期，其他正常活動的計算未受到非預期影響。

</details>

<h2 id="pc-references">延伸參考</h2>

- [C4 Container Diagram](https://c4model.com/diagrams/container)
- [C4 Dynamic Diagram](https://c4model.com/diagrams/dynamic)
- [Microsoft Cache Aside Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
- [AWS Architecture Reference](https://aws.amazon.com/architecture/)

<p class="pc-small">本文圖表依原文整理，呈現業務層級的呼叫與資料流，省略未提供的批次、重試與錯誤回應細節。方法名稱、Key 版本與 Mongo 分界應以排查當下的程式版本為準。</p>

</div>

<!-- endtab -->

{% endtabs %}

<script>
(() => {
  const article = document.getElementById('promotion-cart-calculate-data-flow');
  if (!article) return;
  // Butterfly 重新繪圖時保留舊 SVG；移除舊圖，避免重複 ID 影響箭頭標記。
  article.querySelectorAll('.pc-diagram').forEach(diagram => {
    const observer = new MutationObserver(() => {
      const charts = [...diagram.children].filter(child => child.tagName.toLowerCase() === 'svg');
      const latest = charts[charts.length - 1];
      if (latest && !latest.dataset.pcMarkers) {
        // 隱藏分頁中的同名 marker 會使其他圖的箭頭消失，為每張圖加上獨立 ID。
        latest.querySelectorAll('marker[id]').forEach(marker => {
          const oldId = marker.id;
          marker.id = latest.id + '-' + oldId;
          latest.querySelectorAll('[marker-start], [marker-mid], [marker-end]').forEach(line => {
            ['marker-start', 'marker-mid', 'marker-end'].forEach(attribute => {
              const value = line.getAttribute(attribute);
              if (value) line.setAttribute(attribute, value.replace('#' + oldId + ')', '#' + marker.id + ')'));
            });
          });
        });
        latest.dataset.pcMarkers = 'true';
      }
      charts.slice(0, -1).forEach(chart => chart.remove());
    });
    observer.observe(diagram, { childList: true });
  });
})();
</script>
