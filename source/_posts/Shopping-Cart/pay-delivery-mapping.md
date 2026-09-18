---
title: CalculatePayDeliveryMappingProcessor
date: 2026-09-15 10:00:00
categories: Shopping-Cart
top_img: https://pub-d7e550ef212547d888a6e01348459946.r2.dev/ecom/shoppingcart/pay-delivery.png
cover : https://pub-d7e550ef212547d888a6e01348459946.r2.dev/ecom/shoppingcart/pay-delivery.png
tags:
toc: true
toc_number: false
comments: true
---

{% tabs shopping-cart-pay-delivery-mapping %}


<!-- tab 01．處理什麼問題 -->


<style>
#article-container .pdm-paper{--ink:#1f2430;--sub:#5b6472;--line:#ece7dc;--card:#fff;background:#fbf9f4;color:var(--ink);border:1px solid var(--line);border-radius:20px;padding:30px;font-family:"Segoe UI","Microsoft JhengHei",sans-serif;line-height:1.9;}
#article-container .pdm-paper *{box-sizing:border-box;}
#article-container .pdm-paper p{color:var(--sub);margin:14px 0;}
#article-container .pdm-paper strong{color:var(--ink);}
#article-container .pdm-paper code{color:#92502c;background:#f3ece0;padding:2px 5px;border-radius:4px;overflow-wrap:anywhere;word-break:break-word;}
#article-container .pdm-paper h2,#article-container .pdm-paper h3{color:var(--ink);line-height:1.5;}
#article-container .pdm-paper h2{font-size:1.5rem;margin:14px 0;}
#article-container .pdm-paper h3{font-size:1.15rem;margin:34px 0 14px;padding-left:12px;border-left:4px solid #f08b60;}
#article-container .pdm-hero{padding:26px;border-radius:16px;background:linear-gradient(120deg,#fff0df,#fffaf4 75%);border:1px solid #f3dfc8;}
#article-container .pdm-eyebrow{font-size:.82rem;font-weight:700;letter-spacing:.08em;color:#92502c;}
#article-container .pdm-case{padding:18px 22px;background:var(--card);border:1px solid var(--line);border-radius:16px;margin:18px 0;box-shadow:0 10px 30px -12px rgba(31,36,48,.15);}
#article-container .pdm-table{overflow-x:auto;margin:16px 0;border:1px solid var(--line);border-radius:12px;}
#article-container .pdm-table table{width:100%;margin:0;border-collapse:collapse;font-size:.9rem;}
#article-container .pdm-table th{background:#f3f0e8;color:var(--ink);text-align:left;padding:12px;border:0;}
#article-container .pdm-table td{background:var(--card);color:var(--sub);padding:12px;border:0;border-top:1px solid var(--line);vertical-align:top;}
#article-container .pdm-result{padding:14px 18px;border-left:4px solid #12a594;background:#edf8f5;color:#20554d;border-radius:0 10px 10px 0;margin:16px 0;}
#article-container .pdm-paper ul{padding-left:24px;}
#article-container .pdm-paper li{color:var(--sub);margin:8px 0;}
[data-theme="dark"] #article-container .pdm-paper{--ink:#f2ebe0;--sub:#c6c4bc;--line:#4b4944;--card:#292c30;background:#23262a;}
[data-theme="dark"] #article-container .pdm-hero{background:linear-gradient(120deg,#44352c,#2e2c29);border-color:#655044;}
[data-theme="dark"] #article-container .pdm-eyebrow{color:#ffc592;}
[data-theme="dark"] #article-container .pdm-paper code{color:#ffd1a8;background:#463c32;}
[data-theme="dark"] #article-container .pdm-table th{background:#363733;}
[data-theme="dark"] #article-container .pdm-result{background:#213e37;color:#b8eadb;}
@media(max-width:600px){ #article-container .pdm-paper{padding:15px;border-radius:12px;}#article-container .pdm-hero{padding:18px;}#article-container .pdm-case{padding:14px;}#article-container .pdm-paper h2{font-size:1.25rem;}#article-container .pdm-table td,#article-container .pdm-table th{padding:9px;}}
#article-container .pdm-mapping .pdm-compact svg{width:300px;min-width:240px;max-width:100%!important;} @media(max-width:600px){ #article-container .pdm-mapping .pdm-table table{min-width:500px;}}
</style>
<style>
#article-container .pdm-paper{padding:24px;}
#article-container .pdm-hero{padding:20px 24px;}
#article-container .pdm-paper h3{margin-top:28px;}
#article-container .pdm-diagram{overflow-x:auto;background:#fff;border:1px solid #dce4ed;border-radius:12px;padding:16px 10px;margin:16px 0;}
#article-container .pdm-diagram svg{display:block;width:100%;max-width:none!important;min-width:440px;height:auto;margin:0 auto;}
#article-container .pdm-diagram.pdm-full svg{min-width:780px;}
#article-container .pdm-diagram .mermaid-wrap{margin:0;}
#article-container .pdm-matrix td,#article-container .pdm-matrix th{text-align:center;}
#article-container .pdm-matrix td:first-child,#article-container .pdm-matrix th:first-child{text-align:left;}
#article-container .pdm-matrix tr>:nth-child(2){background:#edf8f5;color:#20554d;font-weight:700;}
#article-container .pdm-detail{padding:14px 18px;border:1px solid var(--line);border-radius:12px;background:var(--card);margin:16px 0;}
#article-container .pdm-detail summary{cursor:pointer;font-weight:700;color:var(--ink);}
#article-container .pdm-detail summary:focus-visible{outline:2px solid #12a594;outline-offset:5px;}
#article-container .pdm-summary{display:grid;grid-template-columns:1fr 1fr;gap:14px;margin:20px 0;}
#article-container .pdm-summary>div{padding:18px;background:var(--card);border:1px solid var(--line);border-top:3px solid #12a594;border-radius:12px;}
#article-container .pdm-caption{font-size:.85rem;color:var(--sub);}
[data-theme="dark"] #article-container .pdm-diagram{background:#202631;border-color:#405066;}
[data-theme="dark"] #article-container .pdm-matrix tr>:nth-child(2){background:#213e37;color:#b8eadb;}
@media(max-width:600px){ .pdm-summary{grid-template-columns:1fr!important;}#article-container .pdm-paper{padding:14px;}#article-container .pdm-hero{padding:16px;}#article-container .pdm-table{font-size:.85rem;} }
</style>
<div class="pdm-paper">
<div class="pdm-hero">
<div class="pdm-eyebrow">商品一起結帳，選項一起檢查</div>
<h2>處理什麼問題？</h2>
<p>多件商品放進同一台購物車，付款與配送方式必須符合每件商品的限制。</p>
<p><code>CalculatePayDeliveryMappingProcessor</code> 找出共同支援的選項，再排除無法合法配對的金物流。</p>
</div>
<h3>金流：每件商品都支援，才能留下</h3>
<p>咖啡豆與餅乾一起結帳。✓ 代表支援，× 代表不支援。</p>
<div class="pdm-table pdm-matrix">
<table>
<thead><tr><th>商品</th><th>信用卡</th><th>ATM</th><th>LINE Pay</th></tr></thead>
<tbody>
<tr><td>咖啡豆</td><td>✓</td><td>✓</td><td>×</td></tr>
<tr><td>餅乾</td><td>✓</td><td>×</td><td>✓</td></tr>
<tr><td>商品交集</td><td>保留</td><td>排除</td><td>排除</td></tr>
</tbody>
</table>
</div>
<p class="pdm-caption">信用卡通過商品交集，仍須繼續套用購物車允許付款方式與配對限制。</p>
<h3>物流：共同選宅配，各溫層保留自己的設定</h3>
<p>假設同溫層商品已完成物流交集，且所有配送地區相同。</p>
<div class="pdm-diagram" tabindex="0" role="region" aria-label="資料關係示意">

{% mermaid %}
flowchart TB
    Normal["常溫：宅配 #101、超取 #201"] -->|共同類型與地區| Shared["宅配"]
    Frozen["冷凍：宅配 #301"] -->|共同類型與地區| Shared
    Shared --> WarmDelivery["保留常溫宅配 #101"]
    Shared --> ColdDelivery["保留冷凍宅配 #301"]
    Normal -.冷凍不支援超取.-> Removed["排除超取 #201"]
    classDef keep fill:#edf9f4,stroke:#32856b,color:#175440
classDef base fill:#edf4ff,stroke:#517ab0,color:#203c60
classDef muted fill:#f3f4f6,stroke:#7d8998,color:#334155
classDef stop fill:#fff1f2,stroke:#b42332,color:#852332
    class Normal,Frozen base
    class Shared,WarmDelivery,ColdDelivery keep
    class Removed muted
{% endmermaid %}

</div>
<p class="pdm-caption">跨溫層比較配送類型與地區，不要求物流 Id 相同，也不表示商品要裝在同一包裹。</p>
<h3>配對：有物流，還要有能搭配的金流</h3>
<p>本例商品交集留下信用卡，以及宅配、超取付款兩種物流。</p>
<div class="pdm-diagram" tabindex="0" role="region" aria-label="資料關係示意">

{% mermaid %}
flowchart LR
    subgraph Payment["付款方式"]
        Card["信用卡：本次可用"]
        StorePay["超取付款金流：本次缺少"]
    end
    subgraph Shipping["配送方式"]
        Home["宅配：保留"]
        Store["超取付款物流：排除"]
    end
    Card -->|設定允許，本次成立| Home
    StorePay -.設定允許，本次無法成立.-> Store
    classDef keep fill:#edf9f4,stroke:#32856b,color:#175440
classDef base fill:#edf4ff,stroke:#517ab0,color:#203c60
classDef muted fill:#f3f4f6,stroke:#7d8998,color:#334155
classDef stop fill:#fff1f2,stroke:#b42332,color:#852332
    class Card,Home keep
    class StorePay,Store muted
{% endmermaid %}

</div>

</div>


<!-- endtab -->

<!-- tab 02．內部資料處理 -->


<div class="pdm-paper">
<h2>內部資料處理</h2>
<p>Processor 呼叫 <code>PayShippingMappingService.CalculatePayDeliveryMapping</code>，讀取資料、計算交集，再寫回購物車。</p>
<div class="pdm-diagram pdm-full" tabindex="0" role="region" aria-label="完整資料處理架構">

{% mermaid %}
%%{init: {'flowchart': {'nodeSpacing': 18, 'rankSpacing': 30, 'curve': 'linear'}}}%%
flowchart TB
    Mapping["全域 Mapping 配對資料"] --> Valid{"Mapping 有資料？"}
    Valid -->|否| Error["拋出 CartCacheExpired 例外"]
    Valid -->|是| Restore{"需要還原商品清單？"}
    Dictionary["商品金流／物流 Dictionary"] -.依商品頁 Id 讀取.-> Assign["重新指派商品金物流清單"]
    Restore -->|是| Assign
    Restore -->|否：沿用商品清單| Delivery
    Assign --> Delivery["同溫層與跨溫層物流交集"]
    Delivery --> Pay["所有商品的金流交集"]
    Limits["允許付款方式、優惠碼與黑名單"] -.物流或金流限制.-> Delivery
    Limits -.金流允許清單與優惠碼.-> Pay
    Pay --> Pair["Mapping 篩選 CheckoutType 選項"]
    Mapping -.合法配對.-> Pair
    Pair --> Union["整理海外配送物流聯集"]
    Products["一般、贈品券與海外無交集商品物流"] -.聯集來源.-> Union
    Union --> Available{"金流與物流皆有選項？"}
    Available -->|是| Ready["保留可用選項，更新提示"]
    Available -->|否| Move["商品移入無交集清單，更新提示"]
    Move --> Cache["保存 Redis，設定中斷與重算旗標"]
    classDef input fill:#f3f4f6,stroke:#7d8998,color:#334155
    classDef work fill:#edf4ff,stroke:#517ab0,color:#203c60
    classDef output fill:#edf9f4,stroke:#32856b,color:#175440
    classDef failure fill:#fff1f2,stroke:#b42332,color:#852332
    class Mapping,Dictionary,Limits,Products input
    class Valid,Restore,Assign,Delivery,Pay,Pair,Union,Available work
    class Ready output
    class Error,Move,Cache failure
{% endmermaid %}

</div>
<p class="pdm-caption">灰色為輸入、藍色為處理、綠色為可用結果、紅色為例外或無交集處理。實線表示順序，虛線表示資料供應。圖中省略排序與主子商品置換細節。</p>
<details class="pdm-detail">
<summary>資料從哪裡來？查看來源與節點</summary>
<p>下表購物車欄位位於 <code>context.Data</code>。</p>
<div class="pdm-table ">
<table>
<thead><tr><th>來源</th><th>讀取節點／用途</th></tr></thead>
<tbody>
<tr><td>全域配對設定</td><td><code>PayShippingMapping</code>，定義金流與物流的合法搭配；查詢沒有以 <code>ShopId</code> 篩選。</td></tr>
<tr><td>商品金物流基準</td><td><code>SalepageIdPayTypesDictionary[SalePageId]</code><br><code>SalepageIdDeliveryTypesDictionary[SalePageId]</code></td></tr>
<tr><td>參與交集的商品</td><td><code>SalepageGroupList[…].SalepageList[…].PayTypeList</code><br><code>SalepageGroupList[…].SalepageList[…].DeliveryTypeList</code>；已存在的贈品券商品也參與。</td></tr>
<tr><td>本次允許付款</td><td><code>EnablePayProfileTypeDef</code></td></tr>
<tr><td>優惠碼與黑名單限制</td><td><code>PromoCodeDispatch.PromoCodeInfo.PaymentTypes</code><br><code>BlacklistMemberAllowedPayProfileList</code></td></tr>
</tbody>
</table>
</div>
<p><code>CartCreateProcessor.AssignSalePagePayShipping</code> 先以 <code>ShopId</code> 與商品頁 Id 查詢 <code>CartRepository.GetSalePagePayTypeAsync</code>、<code>GetSalePageDeliveryType</code>，建立 Dictionary 並首次設定商品金物流。</p>
<p>兩支商品查詢分別呼叫 <code>csp_GetSalePagePayTypeV2</code> 與 <code>csp_GetSalePageDeliveryTypeV2</code>。全域 Mapping 則由 <code>GetPayShippingMappingAsync</code> 透過快取取得，未命中才查 Repository。</p>
</details>
<details class="pdm-detail">
<summary>還原與交集：查看條件、方法與比較鍵</summary>
<div class="pdm-table ">
<table>
<thead><tr><th>處理</th><th>規則</th></tr></thead>
<tbody>
<tr><td>是否還原</td><td><code>context.IsSkipSalepagePayDeliveryTypeInitial = false</code> 才執行還原；為 true 時沿用目前商品清單。</td></tr>
<tr><td>商品還原</td><td><code>ArrangePayDeliveryTypeToSalePage</code> 在 Mapping Service 內從 Dictionary 重新指派商品的 <code>PayTypeList</code>、<code>DeliveryTypeList</code>，不重新查商品設定。</td></tr>
<tr><td>同溫層物流</td><td><code>GetSalePageGroupDeliveryType</code> 先處理合併物流，再比對 <code>Id + ShippingAreaId</code>。</td></tr>
<tr><td>跨溫層物流</td><td><code>CalculateDeliveryTypeAsync</code> 比對 <code>ShippingProfileTypeDef + ShippingAreaId</code>，保留各溫層明細。</td></tr>
<tr><td>商品金流</td><td><code>CalculatePayType</code> 依 <code>PayProfileTypeDef</code> 找共同付款類型，再套用允許清單與優惠碼規則。</td></tr>
<tr><td>合法配對</td><td><code>CalculatePayShippingMapping</code> 依全域設定刪減 <code>CheckoutType</code> 的兩份選項清單。</td></tr>
</tbody>
</table>
</div>
</details>
<details class="pdm-detail">
<summary>海外配送：聯集與交集的差別</summary>
<p><code>CalculateOverseaDeliveryTypeAsync</code> 讀取一般商品、已有贈品券商品，以及無交集清單中標記 <code>OverseaNoIntersection</code> 的商品物流，整理合併物流並去重、排序。</p>
<p>結果寫入 <code>DeliveryTypeUnionList</code>，供後續海外配送使用；它不是本輪共同可結帳的物流清單。</p>
</details>
<details class="pdm-detail">
<summary>流程位置與原始碼對照</summary>
<p>在 <code>InitailPayShippingCalculate</code> 中為第 4 步，在 <code>RePayShippingCalculate</code> 中為第 3 步。實際執行仍受入口條件與中斷影響。</p>
<p>主要來源：<code>CalculatePayDeliveryMappingProcessor.ExecuteAsync</code>、<code>PayShippingMappingService.CalculatePayDeliveryMapping</code>、<code>ProcessorDefinitionCenter</code>。</p>
</details> 
</div>


<!-- endtab -->

<!-- tab 03．全域 Mapping 資料 -->


<div class="pdm-paper pdm-mapping">
<h2>全域 Mapping：金流與物流的合法配對設定</h2>
<p><code>PayShippingMapping</code> 定義付款類型與配送類型之間允許的搭配。這份設定供購物車計算使用，查詢沒有依本次商店的 <code>ShopId</code> 篩選。</p>
<h3>一筆資料代表一組允許的搭配</h3>
<div class="pdm-table">
<table>
<thead><tr><th>付款類型</th><th>配送類型</th><th>這筆設定的意思</th></tr></thead>
<tbody>
<tr><td>信用卡</td><td>宅配</td><td>允許信用卡搭配宅配。</td></tr>
<tr><td>信用卡</td><td>海外配送</td><td>允許同一付款類型搭配另一配送類型。</td></tr>
</tbody>
</table>
</div>

<h3>資料取得：先讀快取，未命中才查資料庫</h3>
<p><code>CalculatePayDeliveryMappingProcessor.ExecuteAsync</code> 呼叫 <code>PayShippingMappingService.CalculatePayDeliveryMapping</code>；Service 一開始便呼叫自己的 <code>GetPayShippingMappingAsync</code> 取得設定。</p>
<div class="pdm-diagram" tabindex="0" role="region" aria-label="全域 Mapping 資料取得路徑">

{% mermaid %}
flowchart TB
    Start["GetPayShippingMappingAsync"] --> Memory{"Memory 有資料？"}
    Memory -->|命中| Result["回傳全域配對清單"]
    Memory -->|未命中| Redis{"Redis 有資料？"}
    Redis -->|命中| Fill["回填 Memory"]
    Redis -->|未命中| DB["Repository 查詢兩表有效設定"]
    DB --> Save["快取結果至 Redis"]
    Save --> Fill
    Fill --> Result
    classDef input fill:#f3f4f6,stroke:#7d8998,color:#334155
    classDef work fill:#edf4ff,stroke:#517ab0,color:#203c60
    classDef output fill:#edf9f4,stroke:#32856b,color:#175440
    class Start,Memory,Redis,Save,Fill work
    class DB input
    class Result output
{% endmermaid %}

</div>
<p class="pdm-caption">正常讀取路徑，<code>cleanCache</code> 預設為 false。快取由 <code>IDataCacheService.GetMemoryOrRedisCacheDataAsync</code> 統一處理。</p>
<div class="pdm-table">
<table>
<thead><tr><th>來源節點</th><th>用途與範圍</th></tr></thead>
<tbody>
<tr><td><code>DataCacheService.GlobalSettingShopId</code></td><td>使用全域設定識別值 <code>-735268</code>，不使用本次購物車的商店 Id。</td></tr>
<tr><td>Memory／Redis</td><td>此查詢分別傳入 600 秒、3600 秒的快取期限。</td></tr>
<tr><td><code>PayShippingMappingRepository.GetPayShippingMappingAsync</code></td><td>快取未命中時執行的資料來源查詢。</td></tr>
<tr><td><code>WebstoreReadOnlyDbContext</code></td><td>讀取 <code>PayShippingMapping</code> 與 <code>PayProfile</code>。</td></tr>
</tbody>
</table>
</div>
<details class="pdm-detail">
<summary>快取識別：ServiceName、TypeName 與 Key</summary>
<p><code>PayShippingMappingService.GetCacheKey</code> 建立 <code>ServiceName = PayShippingMapping</code>、<code>TypeName = GetPayShippingMappingAsync-2016122117</code>、<code>Key = All</code>。</p>
<p>這些是傳給快取服務的組成值。<code>DataCacheService</code> 會再加入全域設定 Id 與 locale，因此 <code>All</code> 不是完整實體快取鍵；全域也不表示跨所有 locale 共用同一鍵。</p>
</details>
<h3>資料庫如何組成回傳欄位？</h3>
<p>Repository 以 <code>PayShippingMapping_PayProfileTypeDef = PayProfile_TypeDef</code> JOIN 兩表，且 <code>PayShippingMapping_ValidFlag</code>、<code>PayProfile_ValidFlag</code> 都必須為 true。</p>
<div class="pdm-table">
<table>
<thead><tr><th>資料庫來源欄位</th><th>PayShippingMappingEntity 欄位</th><th>資料意義</th></tr></thead>
<tbody>
<tr><td><code>PayShippingMapping.PayShippingMapping_PayProfileTypeDef</code></td><td><code>PayProfileTypeDef</code></td><td>配對的付款類型。</td></tr>
<tr><td><code>PayShippingMapping.PayShippingMapping_ShippingProfileTypeDef</code></td><td><code>ShippingProfileTypeDef</code></td><td>配對的配送類型。</td></tr>
<tr><td><code>PayProfile.PayProfile_StatisticsTypeDef</code></td><td><code>StatisticsTypeDef</code></td><td>付款統計類型；本輪配對判斷未使用此欄。</td></tr>
</tbody>
</table>
</div>
<h3>取得後存在哪裡、交給誰？</h3>
<div class="pdm-table">
<table>
<thead><tr><th>節點／條件</th><th>處理</th></tr></thead>
<tbody>
<tr><td><code>CalculatePayDeliveryMapping</code> 的區域變數 <code>payShippingMappingList</code></td><td>接收 <code>List&lt;PayShippingMappingEntity&gt;</code>，並未另外寫入 <code>context.Data</code> 的某個 Mapping 欄位。</td></tr>
<tr><td>清單為空</td><td>立即拋出 <code>CalculateException(CartCacheExpired)</code>，尚未進入商品交集計算。</td></tr>
<tr><td>清單有資料</td><td>Service 保留此變數，待商品交集計算後，傳給 <code>CalculatePayShippingMapping(cartEntity, payShippingMappingList)</code> 使用。</td></tr>
</tbody>
</table>
</div>
</div>


<!-- endtab -->


<!-- tab 04．商品頁金物流資料源 -->


<div class="pdm-paper pdm-mapping">
<h2>商品頁本身支援哪些金流與物流？</h2>
<h3>查詢範圍：商店與商品頁 Id</h3>
<p><code>CartCreateProcessor.ExecuteAsync</code> 整理可結帳商品後，呼叫 <code>AssignSalePagePayShipping</code>。方法取出 <code>salePageList</code> 的 <code>CartSalePageId</code>，以 <code>Distinct</code> 去重，再連同 <code>context.ShopId</code> 傳給 Repository。</p>
<div class="pdm-table">
<table>
<thead><tr><th>資料</th><th>CartRepository 方法</th><th>Stored Procedure</th></tr></thead>
<tbody>
<tr><td>商品金流</td><td><code>GetSalePagePayTypeAsync</code></td><td><code>csp_GetSalePagePayTypeV2</code></td></tr>
<tr><td>商品物流</td><td><code>GetSalePageDeliveryType</code></td><td><code>csp_GetSalePageDeliveryTypeV2</code></td></tr>
</tbody>
</table>
</div>
<p>兩支查詢都透過唯讀 DbContext 執行，參數是 <code>@shopId</code> 與逗號串接的 <code>@salePageIds</code>，查詢結果再以 <code>SalePageId</code> 分組。此處未套用前章全域 Mapping 的兩層快取。</p>
<h3>資料如何進入購物車？</h3>
<div class="pdm-diagram pdm-compact" tabindex="0" role="region" aria-label="商品金物流從查詢到讀取的資料流">

{% mermaid %}
flowchart TB
    Query["商店 Id ＋ 商品頁 Id 清單"] --> SP["金流、物流兩支 SP"]
    SP --> Group["各自按 SalePageId 分組"]
    Group --> Prepare["CartCreate 整理資料"]
    Prepare --> Dict["保存兩份 Dictionary"]
    Dict --> First["首次指派商品金物流"]
    First --> Cart["依溫層分組並建立 CartEntity"]
    Cart --> Later["Mapping Service 依條件重新指派"]
    classDef input fill:#f3f4f6,stroke:#7d8998,color:#334155
    classDef work fill:#edf4ff,stroke:#517ab0,color:#203c60
    classDef output fill:#edf9f4,stroke:#32856b,color:#175440
    class Query,SP input
    class Group,Prepare,First,Cart work
    class Dict,Later output
{% endmermaid %}

</div>
<p class="pdm-caption">Dictionary 留在購物車內供後續依商品頁 Id 取用；重新指派時不在該方法重查 SP。</p>
<div class="pdm-table">
<table>
<thead><tr><th>資料節點</th><th>誰寫入、誰讀取</th></tr></thead>
<tbody>
<tr><td><code>context.Data.SalepageIdPayTypesDictionary[SalePageId]</code></td><td><code>AssignSalePagePayShipping</code> 寫入金流基準清單；<code>ArrangePayDeliveryTypeToSalePage</code> 讀取。</td></tr>
<tr><td><code>context.Data.SalepageIdDeliveryTypesDictionary[SalePageId]</code></td><td>同一寫入與讀取流程，保存物流明細。</td></tr>
<tr><td><code>salePageList[…].PayTypeList</code>、<code>DeliveryTypeList</code></td><td><code>AssignSalePagePayShipping</code> 依 <code>CartSalePageId</code> 首次指派。</td></tr>
<tr><td><code>context.Data.SalepageGroupList[…].SalepageList[…].PayTypeList</code>、<code>DeliveryTypeList</code></td><td><code>GroupCartSalePage</code> 帶入商品群組；之後由交集計算方法直接讀取。</td></tr>
</tbody>
</table>
</div>
<p><code>GroupCartSalePage</code> 將 <code>CartSalePageId</code> 轉成商品的 <code>Id</code>，並帶入兩份清單。建立新的 <code>CartEntity</code> 時，也會將先前的兩份 Dictionary 一併帶入。</p>
<h3>例子：同一商品頁的不同 SKU 使用同一個查詢鍵</h3>
<div class="pdm-table">
<table>
<thead><tr><th>購物車商品</th><th>商品頁 Id</th><th>金流來源</th></tr></thead>
<tbody>
<tr><td>咖啡豆，小包 SKU</td><td>1001</td><td><code>SalepageIdPayTypesDictionary[1001]</code></td></tr>
<tr><td>咖啡豆，大包 SKU</td><td>1001</td><td><code>SalepageIdPayTypesDictionary[1001]</code></td></tr>
<tr><td>馬克杯</td><td>2002</td><td><code>SalepageIdPayTypesDictionary[2002]</code></td></tr>
</tbody>
</table>
</div>

<details class="pdm-detail">
<summary>回傳欄位：金流類型與物流明細</summary>
<div class="pdm-table">
<table>
<thead><tr><th>Entity</th><th>欄位／轉換</th></tr></thead>
<tbody>
<tr><td><code>ShoppingCartClientPayTypeEntity</code></td><td><code>PayProfileTypeDef</code>、<code>StatisticsTypeDef</code> 由 SP 結果同名欄位帶入。</td></tr>
<tr><td><code>ShoppingCartClientDeliveryTypeEntity</code> 的識別與分類</td><td><code>Id = ShopShippingTypeId</code>；另含 <code>ShippingProfileTypeDef</code>、<code>ShippingAreaId</code>、<code>TemperatureTypeDef</code>、<code>MergeShippingTypeId</code>、<code>TypeName</code>。</td></tr>
<tr><td>物流費用與限制</td><td><code>Fee</code>、<code>OriginalFee</code> 均來自 <code>Fee</code>；<code>FeeTypeDef</code>、<code>OriginalFeeTypeDef</code> 均來自 <code>FeeTypeDef</code>。另有 <code>FeeTypeDefDesc</code>、<code>OverPrice</code>、<code>MaxPriceLimit</code>、<code>WeightLimit</code>，以及 <code>Rule = FeeRule</code>。</td></tr>
<tr><td>配送日期與排序</td><td><code>IsEnableBookingPickupDate</code>、<code>ExcludeOfWeek</code>；<code>Sort</code> 取 <code>ShippingAreaSort</code>，無值時為 0。</td></tr>
</tbody>
</table>
</div>
<p>本次核對到 Cart Repository 的 SP 呼叫與結果模型，尚未展開兩支 SP 內部的資料表 JOIN，因此不將未查證的資料表列為底層來源。</p>
</details>
<details class="pdm-detail">
<summary>Dictionary 是否就是未加工的資料庫原始設定？</summary>
<p><code>AssignSalePagePayShipping</code> 在寫入 Dictionary 前，依序執行自訂包裝顯示更新、全家點加金設定處理、代客下單金物流調整，以及物流多語系替換。因此它是 CartCreate 準備好的基準資料，可能已經過情境加工。</p>
<div class="pdm-table">
<table>
<thead><tr><th>代客下單條件</th><th>資料調整</th></tr></thead>
<tbody>
<tr><td><code>IsAssistMode = false</code></td><td>不進入代客下單替換分支。</td></tr>
<tr><td><code>FreeOfChargeAssistOrder</code></td><td>付款清單替換為 <code>FreeOfCharge</code>，物流依零元訂單排除清單過濾。</td></tr>
<tr><td>其他代客下單</td><td>付款清單只保留 <code>CustomOfflinePayment</code>。</td></tr>
</tbody>
</table>
</div>
</details>
<details class="pdm-detail">
<summary>Mapping Service 如何讀取？缺 Key 與子商品如何處理？</summary>
<p><code>IsSkipSalepagePayDeliveryTypeInitial = false</code> 時，<code>CalculatePayDeliveryMapping</code> 呼叫自己的 <code>ArrangePayDeliveryTypeToSalePage</code>。這個方法以商品 <code>Id</code> 對兩份 Dictionary 執行 <code>TryGetValue</code>，成功才以 <code>ToList()</code> 重新指派商品清單；缺 Key 時沒有在這裡查 SP，也不在該分支清空既有清單。</p>
<p>旗標為 true 時跳過重新指派。重新指派使用的 <code>ToList()</code> 只建立新的清單容器，不代表其中每個金物流物件都被深層複製。</p>
<p>一般商品來自 <code>SalepageGroupList</code>，已有贈品券商品則從 <code>SalePageGiftCouponGroupList</code> 讀取。當 <code>IsEnabledAddOnsPayDelivery</code> 未啟用時，一般商品還原範圍會排除加購子商品；最後的 <code>ReplaceSubSalePagePayShipping</code> 再依設定，讓符合條件的加購或組合子商品沿用主商品清單。</p>
<p>後續 <code>CalculatePayType</code> 與 <code>CalculateDeliveryTypeAsync</code> 讀取的是商品上的兩份清單。這些清單描述商品在目前處理階段的支援資料，尚不能直接當成整台購物車的最終選項。</p>
</details>

</div>


<!-- endtab -->


<!-- tab 05．商品間的金流交集 -->



<div class="pdm-paper pdm-mapping">
<h2>所有參與商品共同接受哪些付款方式？</h2>
<p>商品各自有付款清單，<code>CalculatePayType</code> 從中找出所有參與商品共同支援的付款類型，再套用本次購物車的限制。共同的範圍是商品之間，金流不按溫層分開計算。</p>

<h3>案例：每件商品都支援，才是共同金流</h3>
<div class="pdm-table"><table><thead><tr><th>商品頁</th><th>信用卡</th><th>ATM</th><th>LINE Pay</th></tr></thead><tbody><tr><td>咖啡豆 A</td><td>✓</td><td>✓</td><td>✓</td></tr>
<tr><td>馬克杯 B</td><td>✓</td><td>×</td><td>✓</td></tr>
<tr><td>商品交集</td><td><strong>保留</strong></td><td>移除</td><td><strong>保留</strong></td></tr></tbody></table></div>

<div class="pdm-diagram pdm-compact" tabindex="0" role="region" aria-label="金流交集與購物車限制">

{% mermaid %}
flowchart TB
    A["讀取所有參與商品付款清單"] --> B["按付款類型找共同選項"]
    B --> C["排序，套用允許付款清單"]
    C --> D["依條件套用優惠碼指定金流"]
    D --> E["寫入 CheckoutType.PayTypeList"]
    classDef work fill:#edf4ff,stroke:#517ab0,color:#203c60
    classDef result fill:#edf9f4,stroke:#32856b,color:#175440
    class A,B,C,D work
    class E result
{% endmermaid %}

</div>
<p>若本次 <code>EnablePayProfileTypeDef</code> 只允許信用卡與 ATM，上例的 LINE Pay 仍會被移除，輸出只剩信用卡。商品共同支援，是進入選項清單的第一道條件。</p>
<h3>讀取來源與比較鍵</h3>
<div class="pdm-table"><table><thead><tr><th>節點／方法</th><th>用途</th></tr></thead><tbody><tr><td><code>context.Data.SalepageGroupList[…].SalepageList[…].PayTypeList</code></td><td>一般商品目前的付款清單；方法也追加 <code>SalePageGiftCouponGroupList</code> 中已有贈品券商品的付款資料。</td></tr>
<tr><td><code>PayProfileTypeDef</code></td><td>依付款類型分組；保留分組筆數等於參與商品清單筆數的類型。</td></tr>
<tr><td><code>context.Data.EnablePayProfileTypeDef</code></td><td>由前序 <code>GetPayTypeIsAvaliableProcessor</code> 準備，本方法用它再次過濾候選金流。</td></tr>
<tr><td><code>context.Data.PromoCodeDispatch.PromoCodeInfo.PaymentTypes</code></td><td>優惠碼指定付款方式，有符合條件才縮減清單。</td></tr>
<tr><td><code>context.Data.CheckoutType.PayTypeList</code></td><td>本方法最終寫入的付款候選清單，接著仍需經過全域 Mapping 配對。</td></tr></tbody></table></div>
<details class="pdm-detail"><summary>優惠碼指定金流：有交集才縮減，完全不符交由後續處理</summary>
<div class="pdm-table"><table><thead><tr><th>條件</th><th>本方法行為</th></tr></thead><tbody><tr><td>候選金流為空，或無優惠碼資料／未指定付款方式</td><td>跳過優惠碼金流過濾。</td></tr>
<tr><td>指定付款方式與候選金流至少有一種相同</td><td>只保留候選中符合指定的付款方式。</td></tr>
<tr><td>指定付款方式與候選金流完全不符</td><td>保留這一步之前的候選清單；註解指定交由 <code>CalculatePromoCodeProcessor</code> 處理優惠碼無交集錯誤。</td></tr></tbody></table></div><p>例如候選為信用卡、LINE Pay：優惠碼指定信用卡時留下信用卡；若只指定 ATM，此方法不因該條件直接清空候選金流。這不表示優惠碼已通過驗證。</p>
</details>
<details class="pdm-detail"><summary>程式中的計數、贈品券與排序細節</summary>
<p>此實作使用 <code>SelectMany → GroupBy(PayProfileTypeDef) → Count == salePageCount</code>，未先依商品去重。將它理解為集合交集，前提是各商品清單中的同類型沒有重複；若資料重複，單純比較總筆數可能偏離集合交集語意。</p><p>前一個物流方法會把贈品券商品併入一般商品群組，而本方法又追加贈品券清單。因此計數基準是方法實際組出的 <code>salePageList.Count</code>，不能直接當作不重複商品頁 Id 數；這裡沿用程式行為，不另外宣稱已去重。</p><p>每個保留類型建立一筆 <code>ShoppingCartClientPayTypeEntity</code>，<code>StatisticsTypeDef</code> 取該組第一筆。排序使用 <code>PayProfileTypeDefComparer</code> 與全域／商店 <code>EventSort</code> 設定。</p>
</details>

</div>


<!-- endtab -->

<!-- tab 06．商品間的物流交集 -->


<div class="pdm-paper pdm-mapping">
<h2>各溫層都能使用哪些配送類型與地區？</h2>
<p><code>CalculateDeliveryTypeAsync</code> 先找同溫層商品的共同物流，再找各溫層共同具有的配送類型與地區。跨溫層通過後，仍保留各溫層自己的物流明細。</p>
<div class="pdm-diagram pdm-compact" tabindex="0" role="region" aria-label="同溫層及跨溫層物流交集">

{% mermaid %}
flowchart TB
    A["依溫層併入已有贈品券商品"] --> B["各溫層先整理合併物流"]
    B --> C["溫層內：物流 Id ＋ 地區"]
    C --> D["跨溫層：配送類型 ＋ 地區"]
    D --> E["保留各溫層明細，再套用限制"]
    classDef work fill:#edf4ff,stroke:#517ab0,color:#203c60
    classDef result fill:#edf9f4,stroke:#32856b,color:#175440
    class A,B,C,D work
    class E result
{% endmermaid %}

</div>
<h3>第一層：同溫層商品要有相同物流 Id 與地區</h3>
<div class="pdm-table"><table><thead><tr><th>常溫商品／地區 1</th><th>宅配 #101</th><th>超取付款 #201</th></tr></thead><tbody><tr><td>咖啡豆 A</td><td>✓</td><td>✓</td></tr>
<tr><td>馬克杯 B</td><td>✓</td><td>×</td></tr>
<tr><td>常溫交集</td><td><strong>保留 #101</strong></td><td>移除 #201</td></tr></tbody></table></div>
<p><code>GetSalePageGroupDeliveryType</code> 在整理合併物流後，以 <code>Id + ShippingAreaId</code> 分組，保留筆數等於該溫層商品筆數的組合，並取各組第一筆明細。即使配送名稱都叫宅配，Id 或地區不同也不直接視為相同。</p>
<h3>第二層：跨溫層改比配送類型與地區</h3>
<div class="pdm-table"><table><thead><tr><th>溫層內交集結果</th><th>物流 Id</th><th>配送類型</th><th>地區</th><th>跨溫層結果</th></tr></thead><tbody><tr><td>常溫</td><td>#101</td><td>宅配</td><td>1</td><td>保留</td></tr>
<tr><td>冷凍</td><td>#301</td><td>宅配</td><td>1</td><td>保留</td></tr>
<tr><td>冷凍</td><td>#302</td><td>宅配</td><td>2</td><td>移除，常溫沒有地區 2</td></tr></tbody></table></div>
<p>兩個溫層都具有「宅配＋地區 1」，所以 #101、#301 同時保留。這一步比較 <code>ShippingProfileTypeDef + ShippingAreaId</code>，不要求物流 Id 相同，也不表示常溫與冷凍商品裝在同一包裹。</p>

<h3>資料讀寫節點</h3>
<div class="pdm-table"><table><thead><tr><th>節點／方法</th><th>用途</th></tr></thead><tbody><tr><td><code>context.Data.SalepageGroupList</code></td><td>主要商品群組；按溫層合併已有的 <code>SalePageGiftCouponGroupList</code> 後參與計算。</td></tr>
<tr><td><code>SalepageList[…].DeliveryTypeList</code></td><td>每件商品目前的物流明細，是 <code>GetSalePageGroupDeliveryType</code> 的讀取來源。</td></tr>
<tr><td><code>salePageGroupDeliveryTypeList</code></td><td>區域變數，保存各溫層的物流交集結果。</td></tr>
<tr><td><code>distinctSalePageGroupShippingTypeDefList</code></td><td>區域變數，保存跨溫層共同的配送類型與地區組合。</td></tr>
<tr><td><code>context.Data.CheckoutType.DeliveryTypeList</code></td><td>展開各溫層明細，只保留共同組合，排序後寫入，再依優惠碼與黑名單條件刪減。</td></tr></tbody></table></div>
<details class="pdm-detail"><summary>合併物流：比較 Id 前會先整理</summary>
<p>方法先收集具有 <code>MergeShippingTypeId</code>，且其指向 Id 也存在於該溫層物流中的設定，再由 <code>MergeDeliveryType</code> 整理每件商品清單。</p><div class="pdm-table"><table><thead><tr><th>假設設定</th><th>商品原清單</th><th>整理後</th></tr></thead><tbody><tr><td>物流 #3 的 <code>MergeShippingTypeId = 1</code></td><td>#1</td><td>#3</td></tr>
<tr><td>同一設定</td><td>#1、#3</td><td>#3</td></tr>
<tr><td>同一設定</td><td>#1、#2</td><td>#3、#2</td></tr></tbody></table></div><p>以上條件成立時，#1 或 #3 都會被整理成合併設定 #3，再進行同溫層交集。這是本方法的比較用清單，不是單純忽略物流 Id。</p>
</details>
<details class="pdm-detail"><summary>交集後的物流限制：優惠碼與黑名單</summary>
<div class="pdm-table"><table><thead><tr><th>來源／條件</th><th>處理</th></tr></thead><tbody><tr><td><code>PromoCodeDispatch.PromoCodeInfo.PaymentTypes</code> 指定 <code>StoreCredit</code>、<code>GiftCard</code> 或 <code>HamiPoint</code></td><td>從該付款方式對應的 <code>DeferredPaymentShippingProfileTypeDefList</code> 找出不適用配送類型並移除；其他付款類型跳過此分支。</td></tr>
<tr><td><code>BlacklistMemberAllowedPayProfileList</code> 非 null，且包含上述折抵方式</td><td>從對應的不適用配送類型清單，扣除允許清單中也存在的值，再移除剩下的配送類型。不是把所有後付款物流一律移除。</td></tr></tbody></table></div><p>對應限制清單分別定義於 <code>CalculateCheckoutStoreCreditProcessor</code>、<code>GiftCardSelectionHandler</code> 與 <code>CheckoutHamiPointCalculator</code>。判斷逐項執行，符合多項時會連續過濾。配送類型為空白的項目在這兩段過濾中保留。</p>
</details>
<details class="pdm-detail"><summary>群組、計數與輸出邊界</summary>
<p><code>salePageGroupList</code> 直接引用 <code>context.Data.SalepageGroupList</code>。贈品券依溫層新增群組或追加商品，會修改 Context 中的群組，不是獨立複本。</p><p>同溫層使用分組總筆數與商品清單筆數比較，未全面先按商品去重。跨溫層則先在各組內對「配送類型＋地區」執行 <code>Distinct</code>，再保留出現次數等於溫層群組數的組合。</p><p>保留明細依 <code>ShippingProfileTypeDefComparer</code> 排序，再依 <code>ShippingAreaId</code> 排序；Comparer 使用全域與商店 <code>EventSort</code> 設定。</p><p>輸出為 <code>CheckoutType.DeliveryTypeList</code>，後續全域 Mapping 與其他 Processor 仍可能刪減。海外聯集另由 <code>CalculateOverseaDeliveryTypeAsync</code> 寫入 <code>DeliveryTypeUnionList</code>，不屬於本章的物流交集輸出。</p>
</details>

</div>


<!-- endtab -->

<!-- tab 07．交集後的金物流配對 -->


<div class="pdm-paper pdm-mapping">
<h2>共同支援的選項，還要有合法搭配</h2>
<p>商品共同接受某種付款、某種配送，不代表兩者可以搭配。<code>CalculatePayShippingMapping</code> 將交集後的選項套入全域 Mapping，留下有搭配對象的付款與配送選項。</p>
<h3>三份輸入，各自回答不同問題</h3>
<div class="pdm-table"><table><thead><tr><th>輸入節點</th><th>前序來源</th><th>讀取欄位</th></tr></thead><tbody>
<tr><td><code>payShippingMappingList</code></td><td>外層 <code>CalculatePayDeliveryMapping</code> 一開始取得的全域配對清單。</td><td><code>PayProfileTypeDef</code>、<code>ShippingProfileTypeDef</code></td></tr>
<tr><td><code>context.Data.CheckoutType.PayTypeList</code></td><td><code>CalculatePayType</code> 完成商品交集與付款限制後的候選金流。</td><td><code>PayProfileTypeDef</code></td></tr>
<tr><td><code>context.Data.CheckoutType.DeliveryTypeList</code></td><td><code>CalculateDeliveryTypeAsync</code> 完成交集與物流限制後的候選配送明細。</td><td><code>ShippingProfileTypeDef</code></td></tr>
</tbody></table></div>
<p class="pdm-caption">上表以 Processor 的 <code>context.Data</code> 表示購物車根節點；進入私有配對方法後，其 <code>context</code> 參數本身就是 <code>CartEntity</code>。</p>
<h3>案例：兩端都存在，這筆配對才有效</h3>
<p>假設候選金流為「信用卡、ATM」，候選物流為「宅配、超取付款」。下表為教學假設設定，名稱是類型代稱。</p>
<div class="pdm-table"><table><thead><tr><th>全域 Mapping 配對</th><th>付款端存在？</th><th>配送端存在？</th><th>本輪配對</th></tr></thead><tbody>
<tr><td>信用卡 → 宅配</td><td>✓</td><td>✓</td><td>保留</td></tr>
<tr><td>ATM → 宅配</td><td>✓</td><td>✓</td><td>保留</td></tr>
<tr><td>超取付款金流 → 超取付款物流</td><td>×</td><td>✓</td><td>排除</td></tr>
<tr><td>信用卡 → 海外配送</td><td>✓</td><td>×</td><td>排除</td></tr>
</tbody></table></div>
<div class="pdm-diagram" tabindex="0" role="region" aria-label="有效配對與候選選項保留結果">

{% mermaid %}
flowchart LR
    Card["信用卡：保留"] --> Home["宅配：保留"]
    ATM["ATM：保留"] --> Home
    Missing["超取付款金流：候選中不存在"] -.設定有配對，本輪無效.-> CVS["超取付款物流：移除"]
    classDef keep fill:#edf9f4,stroke:#32856b,color:#175440
    classDef muted fill:#f3f4f6,stroke:#7d8998,color:#334155
    class Card,ATM,Home keep
    class Missing,CVS muted
{% endmermaid %}

</div>
<p class="pdm-caption">實線為本輪有效配對，虛線為設定存在但本輪無效的配對。圖中聚焦候選物流的去留，省略候選中不存在的海外配送。</p>
<p>反向套回選項後，信用卡與 ATM 都有宅配可搭配，兩者保留；超取付款物流沒有有效連線，因此即使通過商品交集，仍會被移除。</p>
<h3>實作：先篩配對，再依序刪減兩份清單</h3>
<div class="pdm-table"><table><thead><tr><th>步驟</th><th>操作</th><th>結果位置</th></tr></thead><tbody>
<tr><td>1．篩選配對</td><td>兩個 <code>Where</code> 分別確認付款類型、配送類型存在於目前候選。</td><td>區域變數 <code>avaliblePayShippingMappingList</code>，拼字沿用原始碼。</td></tr>
<tr><td>2．刪減金流</td><td><code>RemoveAll</code> 移除沒有出現在有效配對付款端的類型。</td><td>直接修改 <code>CheckoutType.PayTypeList</code>。</td></tr>
<tr><td>3．刪減物流</td><td><code>RemoveAll</code> 移除沒有出現在有效配對配送端的類型。</td><td>直接修改 <code>CheckoutType.DeliveryTypeList</code>。</td></tr>
</tbody></table></div>
<details class="pdm-detail">
<summary>比對鍵：為什麼不再比較物流 Id、溫層與地區？</summary>
<p>這一步只使用 <code>PayProfileTypeDef</code> 與 <code>ShippingProfileTypeDef</code>。物流 Id、溫層、地區已在前段處理，本方法不以它們重新查詢 Mapping。</p>
<p>若宅配類型有有效配對，前段保留的常溫宅配 #101、冷凍宅配 #301 都能留下；此處不把兩筆明細合成一筆。<code>StatisticsTypeDef</code> 也不是本方法的配對鍵。</p>
<p>方法不重新查商品設定、不新增選項，也不把篩選後的配對表另外存入購物車。<code>Where</code> 結果是延遲列舉，未先以 <code>ToList()</code> 建立獨立配對快照。</p>
</details>
<details class="pdm-detail">
<summary>每個選項有配對，不代表可以任意組合</summary>
<p>另一個假設：只允許「信用卡 → 宅配」與「超取付款金流 → 超取付款物流」，而四個端點都在候選清單。此方法會保留全部四個選項，但不會因此允許「信用卡 → 超取付款物流」。</p>
<p>輸出清單表示每個選項至少存在一個合法搭配，不能將兩份清單直接解讀為所有組合都合法。這一步也沒有替消費者選定付款與配送方式。</p>
</details>
<h3>配對後往哪裡走？</h3>
<p>外層 Service 接著整理海外物流聯集，再檢查付款與配送清單是否皆非空。任一為空，就進入下一章的無交集商品搬移與中斷處理；皆有選項則更新提示並繼續後續計算。</p>
<p>「全域 Mapping 本身為空」在 Service 入口就會拋出 <code>CartCacheExpired</code>；「有全域設定，但本輪沒有有效配對」則會讓兩份候選清單被清空。這是兩個不同的處理情境。</p>

</div>


<!-- endtab -->

<!-- tab 08．DeliveryTypeUnionList 的用意 -->


<div class="pdm-paper pdm-mapping">
<h2>DeliveryTypeUnionList：保留供後續查找的物流資料</h2>
<p><code>CalculateOverseaDeliveryTypeAsync</code> 整理商品物流聯集，寫入 <code>context.Data.DeliveryTypeUnionList</code>，供後續海外配送流程使用。方法名稱描述後續用途；此方法本身沒有只挑海外物流，也沒有判斷配送國家或地區。</p>
<h3>為什麼交集之外，還需要聯集？</h3>
<div class="pdm-table"><table><thead><tr><th>商品／資料集合</th><th>宅配 #101</th><th>海外配送 #401</th></tr></thead><tbody>
<tr><td>商品 A</td><td>✓</td><td>✓</td></tr>
<tr><td>商品 B</td><td>✓</td><td>×</td></tr>
<tr><td>商品共同物流</td><td>保留</td><td>不在共同選項中</td></tr>
<tr><td>商品物流聯集</td><td>保留</td><td>保留，商品 A 有此設定</td></tr>
</tbody></table></div>

<div class="pdm-table"><table><thead><tr><th>節點</th><th>用途</th></tr></thead><tbody>
<tr><td><code>CheckoutType.DeliveryTypeList</code></td><td>商品交集、限制與 Mapping 配對後的本輪配送候選明細。</td></tr>
<tr><td><code>DeliveryTypeUnionList</code></td><td>從商品物流重新收集並整理的聯集，保留供後續查找的物流設定，可能同時包含本地與海外物流。</td></tr>
</tbody></table></div>
<p>聯集中存在 #401，只表示來源商品具有這筆物流資料，不表示商品 B 可以海外配送，也不表示整台購物車已通過海外配送檢查。</p>
<h3>建立聯集與挑選海外物流，是兩個階段</h3>
<div class="pdm-diagram pdm-compact" tabindex="0" role="region" aria-label="物流聯集建立與海外配送使用階段">

{% mermaid %}
flowchart TB
    subgraph Prepare["目前的 Mapping Service"]
        A["讀取商品自己的物流清單"] --> B["整理合併物流、去重與排序"]
        B --> C["寫入 DeliveryTypeUnionList"]
    end
    subgraph Later["後續海外配送流程"]
        D["GetOverseaShippingProcessor"] --> E["從聯集挑出 Oversea 類型"]
        E --> F["依配送設定取得地區"]
        F --> G["設定海外可用狀態與地區清單"]
    end
    C -.提供物流資料.-> D
    classDef work fill:#edf4ff,stroke:#517ab0,color:#203c60
    classDef result fill:#edf9f4,stroke:#32856b,color:#175440
    class A,B,D,E,F work
    class C,G result
{% endmermaid %}

</div>
<p class="pdm-caption">實線表示階段內的處理順序，虛線表示資料供應。只有後續 <code>GetOverseaDeliveryTypeList</code> 才以 <code>ShippingProfileTypeDef = Oversea</code> 篩選。</p>
<h3>聯集從哪些商品節點收集？</h3>
<p>下表節點皆位於 <code>context.Data</code>。收集的是商品自己的 <code>DeliveryTypeList</code>，不是把 <code>CheckoutType.DeliveryTypeList</code> 複製成另一份清單。</p>
<div class="pdm-table"><table><thead><tr><th>來源節點</th><th>納入條件</th></tr></thead><tbody>
<tr><td><code>SalepageGroupList[…].SalepageList[…]</code></td><td>目前一般商品群組內的商品。</td></tr>
<tr><td><code>SalePageGiftCouponGroupList[…].SalepageList[…]</code></td><td>清單中已存在的贈品券商品。</td></tr>
<tr><td><code>UnMappingCheckoutSalepageList</code></td><td>只納入 <code>PayShippingIntersectionStatus = OverseaNoIntersection</code> 的商品；不把所有無交集商品一律加回。</td></tr>
</tbody></table></div>
<details class="pdm-detail">
<summary>不只是塞值：合併、去重與排序規則</summary>
<p>方法先展開來源商品的物流明細。若物流有 <code>MergeShippingTypeId</code>，且指向的 Id 也在收集清單內，便交給 <code>MergeDeliveryType</code> 整理比較用物流。</p>
<p>接著依 <code>Id</code> 分組，每組取第一筆，再以 <code>ShippingProfileTypeDefComparer</code> 與 <code>ShippingAreaId</code> 排序，最後用 <code>ToList()</code> 寫入 <code>DeliveryTypeUnionList</code>。此處去重鍵只有物流 Id，不是「類型＋地區」的交集鍵。</p>
<p><code>ToList()</code> 建立新的清單容器，不代表物流明細物件都被深層複製。這個方法也沒有用全域 Mapping 或付款清單重新篩選聯集。</p>
</details>
<details class="pdm-detail">
<summary>後續誰讀取？寫出哪些結果？</summary>
<p><code>GetOverseaShippingProcessor.ExecuteAsync</code> 呼叫 <code>ShippingAreaService.ArrangeOverseaShippingAreasAsync</code>。其中的 <code>GetOverseaDeliveryTypeList</code> 從 <code>DeliveryTypeUnionList</code> 挑出 <code>Oversea</code> 類型。</p>
<p>取得海外物流後，Service 依物流 Id 查找配送運費表與地區資料，整理配送地區聯集，再更新 <code>CanOverseaShipping</code> 與 <code>ShippingAreaList</code>。沒有海外物流時，會將海外可用狀態設為 false、清空地區清單並返回。</p>
<p>這是聯集資料的後續用途，不是 <code>CalculateOverseaDeliveryTypeAsync</code> 已經完成的工作；是否所有商品能送到所選地區仍須後續流程判斷。</p>
</details>
<h3>在目前 Processor 中的執行位置</h3>
<p><code>CalculatePayDeliveryMapping</code> 先完成商品交集與 <code>CalculatePayShippingMapping</code>，再呼叫此方法建立聯集，之後才檢查候選金流、物流是否為空。因此即使建立了聯集，本輪仍可能在隨後的無交集判斷中要求中斷；不能假設後續海外 Processor 每次都一定執行。</p>

</div>



<!-- endtab -->

<!-- tab 09．結果與例外 -->



<div class="pdm-paper">
<h2>結果與例外</h2>
<p>先區分「配對設定缺失」與「商品計算後沒有共同選項」，兩者走不同處理分支。</p>
<div class="pdm-diagram" tabindex="0" role="region" aria-label="資料關係示意">

{% mermaid %}
flowchart TB
    Start{"全域 Mapping 有資料？"}
    Start -->|否| Error["拋出 CartCacheExpired"]
    Start -->|是| Calculate["完成交集、配對與海外物流聯集"]
    Calculate --> Available{"金流與物流皆非空？"}
    Available -->|是| Ready["更新提示，繼續後續計算"]
    Available -->|否| Move["商品移入無交集清單，清空原群組"]
    Move --> Mark["標記 NoIntersection，更新提示"]
    Mark --> Save["保存 CartEntity 至 Redis"]
    Save -->|保存成功| Flags["設定中斷與重算旗標"]
    classDef keep fill:#edf9f4,stroke:#32856b,color:#175440
classDef base fill:#edf4ff,stroke:#517ab0,color:#203c60
classDef muted fill:#f3f4f6,stroke:#7d8998,color:#334155
classDef stop fill:#fff1f2,stroke:#b42332,color:#852332
    class Start,Calculate,Available base
    class Ready keep
    class Error,Move,Mark,Save,Flags stop
{% endmermaid %}

</div>
<div class="pdm-summary">
<div><strong>付款選項</strong><p><code>CheckoutType.PayTypeList</code></p></div>
<div><strong>配送明細</strong><p><code>CheckoutType.DeliveryTypeList</code></p></div>
</div>
<p>這兩份結果供後續運費、重量、金額門檻與預選計算使用，仍可能被調整。</p>
<details class="pdm-detail">
<summary>無交集時，哪些商品與旗標會改變？</summary>
<p>例如咖啡豆只支援信用卡、餅乾只支援 ATM，即使都能宅配，付款交集仍為空。</p>
<div class="pdm-table ">
<table>
<thead><tr><th>節點</th><th>變化</th></tr></thead>
<tbody>
<tr><td><code>UnMappingCheckoutSalepageList</code></td><td>合併目前一般商品與贈品券商品，不只挑其中一件移除。</td></tr>
<tr><td><code>SalepageGroupList</code>、<code>SalePageGiftCouponGroupList</code></td><td>搬移商品後清空。</td></tr>
<tr><td><code>PayShippingIntersectionStatus</code></td><td>無交集清單商品標記為 <code>NoIntersection</code>。</td></tr>
<tr><td><code>context.IsInterrupted</code>、<code>context.NeedReCalculate</code></td><td>Service 回傳無交集結果；Processor 保存 Redis 成功後，兩者設為 true。</td></tr>
</tbody>
</table>
</div>
<p>這裡只提出重算需求，不直接重跑自己，後續由外層流程接手。</p>
</details>
<details class="pdm-detail">
<summary>選項減少時，提示如何更新？</summary>
<p><code>CheckPayShippingMapping</code> 更新 <code>CheckoutType.DisplayMessage</code>。商品原有的金流種類數或配送類型與地區組合數，若與最後選項不同，就設定 <code>OnlyFollowingPaymentAndShipping</code> 對應提示。</p>
<p>出現提示可能只是選項減少，不表示一定無法結帳。無交集時商品已先搬移，也不能假設必定出現同一提示。</p>
</details>

</div>


<!-- endtab -->

{% endtabs %}
