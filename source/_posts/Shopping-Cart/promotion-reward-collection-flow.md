---
title: 回饋給券活動商品異動的全貌鏈路
date: 2026-09-18 10:00:00
categories: Shopping-Cart
tags:
  - Promotion
  - Collection
toc: true
toc_number: false
comments:
---

{% tabs promotion-reward-collection-flow %}

<!-- tab 全貌鏈路 -->

<style>
.rc-flow{--rc-ink:#243449;--rc-muted:#526379;--rc-line:#dce4ed;--rc-panel:#f5f8fc;--rc-blue:#245eaa;color:var(--rc-ink);line-height:1.85;overflow-wrap:anywhere}
.rc-flow *{box-sizing:border-box}
.rc-flow h2,.rc-flow h3{color:var(--rc-ink)}
.rc-flow h3{margin-top:2em;font-size:1.15em}
.rc-lead{padding:22px 24px;border:1px solid var(--rc-line);border-top:4px solid var(--rc-blue);border-radius:12px;background:var(--rc-panel);margin:16px 0 24px}
.rc-lead p{margin:0}
.rc-figure{border:1px solid var(--rc-line);border-radius:10px;overflow:hidden;margin:24px 0}
.rc-diagram{padding:20px 12px;overflow-x:auto;background:var(--rc-diagram-bg,#fff)}
.rc-diagram .mermaid-wrap{margin:0}
.rc-diagram svg{display:block;margin:auto;width:100%;min-width:600px;max-width:none!important;height:auto}
.rc-diagram.rc-decisions svg{min-width:1000px}
.rc-diagram:focus-visible{outline:2px solid var(--rc-blue);outline-offset:-3px}
.rc-figure figcaption{background:var(--rc-panel);color:var(--rc-muted);padding:12px 16px;font-size:13px;border-top:1px solid var(--rc-line)}
.rc-flow code{overflow-wrap:anywhere}
.rc-flow table{font-size:14px}
.rc-flow th{background:var(--rc-panel);color:var(--rc-ink)}
.rc-flow th,.rc-flow td{white-space:normal;overflow-wrap:anywhere}
.rc-note{border-left:4px solid var(--rc-blue);padding:14px 18px;background:var(--rc-panel);margin:20px 0}
.rc-evidence{border:1px solid var(--rc-line);border-radius:8px;padding:14px 18px;margin-top:24px}
.rc-evidence summary{cursor:pointer;color:var(--rc-blue);font-weight:700}
.rc-evidence li{margin:8px 0}
[data-theme="dark"] .rc-flow{--rc-ink:#e1e9f3;--rc-muted:#b5c2d3;--rc-line:#405066;--rc-panel:#232e3d;--rc-blue:#91bfff;--rc-diagram-bg:#202631}
@media(max-width:600px){.rc-lead{padding:18px 16px}.rc-note{padding:12px}.rc-evidence{padding:12px}}
</style>

<div class="rc-flow">

## 全貌鏈路

<div class="rc-lead">
<p>同樣是修改回饋給券活動的商品，從後台畫面圈選與從批次匯入進入，會走到不同的 API。畫面圈選直接修改原商品集合；批次異動會先比較商品清單，再更新活動引用的集合。</p>
</div>

這裡的商品集合是 `collection`，由 `collectionId` 識別。以下聚焦 `RewardReachPriceWithCoupon` 已有指定／排除商品集合的流程；實際執行仍需通過 API 驗證。

### 呼叫來源：SMS 畫面與 SCM NMQ 批次

<figure class="rc-figure">
<div class="rc-diagram" tabindex="0" role="region" aria-label="SMS 與批次商品異動的呼叫來源">

{% mermaid %}
flowchart TB
    subgraph SmsSource["SMS 後台入口"]
        Screen["給券活動商品圈選"] --> Edit["EditTargetIdList"]
        Upload["回饋商品批次匯入"] --> Task["建立批次任務"]
    end
    Task -.->|"批次交接"| Nmq["SCM NMQ：商品批次處理"]
    Nmq --> Client["SCM API Client"]
    Client --> Scm["SCMAPIV2：ModifyPromotionSalePages"]
    subgraph WebApi["Promotion WebAPI"]
        CollectionApi["collection update"]
        SalePageApi["salepage-update"]
    end
    Edit -->|"HTTP POST"| CollectionApi
    Scm -->|"HTTP POST"| SalePageApi
    classDef service fill:#edf4ff,stroke:#517ab0,color:#203c60
    class Screen,Edit,Upload,Task,Nmq,Client,Scm,CollectionApi,SalePageApi service
{% endmermaid %}

</div>
<figcaption>實線表示呼叫方向，虛線表示批次任務交接。第二條路徑由 SCMAPIV2 直接呼叫 Promotion WebAPI，SCM NMQ 是它的上游批次來源。</figcaption>
</figure>

| 呼叫端 | 接收端與方法 | 完整 API 路徑 |
|---|---|---|
| SMS 前端 | SMS `EditTargetIdList` | `Api/PromotionEngine/EditTargetIdList` |
| SMS 後端 | Promotion WebAPI，`POST` | `/api/salepage-collections/update` |
| SCM API Client | SCMAPIV2，`POST` | `/v2/Promotion/ModifyPromotionSalePages` |
| SCMAPIV2 | Promotion WebAPI，`POST` | `/api/promotion-rules/salepage-update` |

SCM NMQ 由 `BatchModifyPromotionSalePageService.DoProcess` 呼叫 Client 的 `ModifyPromotionSalePages`。給券活動符合 SCMAPIV2 的轉送條件，因此進入 `salepage-update`。

### 處理分支：原集合增刪、重建與刪除

<figure class="rc-figure">
<div class="rc-diagram rc-decisions" tabindex="0" role="region" aria-label="Promotion WebAPI 的 collection 處理分支">

{% mermaid %}
flowchart TB
    Collection["collection update"] --> Content["CollectionService：原集合增刪商品"]
    Content --> Same["collectionId 不變"]
    Content --> Sync["建立 PromotionTag 同步任務"]
    SalePage["salepage-update"] --> Diff{"商品有實際異動？"}
    Diff -->|"否"| Return["直接返回，不重建"]
    Diff -->|"是"| Update["PromotionEngineService.UpdateAsync"]
    Save["活動 update API"] --> Update
    Update --> Scope{"這次處理的範圍"}
    Scope -->|"SalePage，已有 fixed collection"| New["建立新 collection，填入商品全集"]
    New --> Link["更新活動關聯與規則；保留舊 collection"]
    Scope -->|"Shop 或 None，存在舊 collection"| Delete["刪除該範圍目前引用的 collection"]
    classDef service fill:#edf4ff,stroke:#517ab0,color:#203c60
    classDef decision fill:#fff4d6,stroke:#b48824,color:#614500
    classDef result fill:#f3f4f6,stroke:#7d8998,color:#334155
    class Collection,Content,SalePage,Update,Save,New service
    class Diff,Scope decision
    class Same,Sync,Return,Link,Delete result
{% endmermaid %}

</div>
<figcaption>藍色為處理步驟，菱形為判斷，灰色為結果。圖中只展開給券活動的既有 fixed 商品集合，以及切換成全站或取消排除範圍的分支。</figcaption>
</figure>

`CollectionService` 直接增刪原集合，並建立 `SyncCollectionToPromotionTagJob`。這條路徑不經過活動的 `UpdateAsync`。

`salepage-update` 會比較異動前後的商品清單。有實際新增或移除才進入 `UpdateAsync`；重複新增已有商品、移除不存在的商品，都不會在這個差異分支重建集合。

<div class="rc-note">
<strong>活動儲存也能直接進入更新流程。</strong> <code>POST /api/promotion-rules/update</code> 直接呼叫 <code>UpdateAsync</code>，沒有經過上述商品差異判斷。給券活動處理既有 fixed collection 時，即使商品清單相同，也會進入重建分支。<code>salepage-update</code> 呼叫 <code>UpdateAsync</code> 則是內部方法呼叫，不會再發送一次 HTTP update 請求。
</div>

範圍由「全站排除商品」改成「全站活動」時，排除類型從 `SalePage` 變為 `None`，會刪除目前排除範圍引用的集合。這與一般商品異動重建後保留舊集合，是不同分支。

純全站且尚無排除集合時，`salepage-update` 的新增操作會先建立排除範圍，再進入商品比較；這個前置情境不包含在上圖的既有集合主線中。

<details class="rc-evidence">
<summary>鏈路來源對照</summary>

- SMS：`rewardCoupon.config.tsx:187` 的 `onUpdate` → `PromotionEngineService.EditTargetIdList:3641` → `Nine1PromotionWebApiHttpClient.UpdateCollectionSalepages:166`。
- SCM NMQ：`BatchModifyPromotionSalePageService.cs:182` → SCM Client `PromotionClient.cs:65`。
- SCMAPIV2：`PromotionService.ModifyPromotionSalePages:788` → `Nine1PromotionWebApiHttpClient.UpdateSalepageScopes:238`。
- Promotion WebAPI：`CollectionService.cs:76` 處理原集合；`PromotionEngineService.cs:421` 比較商品，`:2819` 分流範圍，`:3049` 重建集合。

來源為 2026-09-16 的「回饋給券活動商品異動與 Collection 生命週期鏈路分析」。以上為本機程式碼所確認的呼叫鏈，未比對線上部署版本。

</details>

</div>

<script>
document.querySelectorAll('.rc-flow .rc-diagram').forEach(function (container) {
const observer = new MutationObserver(function (changes) {
const inserted = changes.flatMap(function (change) { return Array.from(change.addedNodes); }).filter(function (node) { return node.nodeType === 1 && node.tagName.toLowerCase() === 'svg'; });
const newest = inserted[inserted.length - 1];
if (!newest) return;
Array.from(container.children).forEach(function (node) { if (node.tagName.toLowerCase() === 'svg' && node !== newest) node.remove(); });
});
observer.observe(container, { childList: true });
});
</script>

<!-- endtab -->

{% endtabs %}
