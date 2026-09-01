---
title: 購物車資料結構解密：ShoppingCart 與 ShoppingCartExtendInfo
date: 2026-08-31 11:30:00
categories: Shopping-Cart
top_img: https://pub-d7e550ef212547d888a6e01348459946.r2.dev/ecom/shoppingcart/extendinfo-landing.png
cover : https://pub-d7e550ef212547d888a6e01348459946.r2.dev/ecom/shoppingcart/extendinfo-landing.png
tags:
toc:
toc_number:
comments :
---


{% tabs shoppingcart-extendinfo %}


<!-- tab 購物車延伸-->


<style>
.hl-hero{position:relative;overflow:hidden;border-radius:18px;padding:36px 40px;margin:0 0 30px;background:linear-gradient(160deg,#f4f2ec 0%,#eae8dc 65%,#e3dfcd 100%);border:1.5px solid #d7cfba;}
.hl-hero::after{content:"";position:absolute;top:-30px;right:-30px;width:120px;height:120px;border-radius:50%;background:radial-gradient(circle at 35% 35%,#c9403c,#8f1c1a);opacity:.16;}
.hl-hero .hh-tag{display:inline-block;font-size:.72rem;letter-spacing:.14em;text-transform:uppercase;background:#0e3746;color:#f4f2ec;padding:5px 16px;border-radius:3px;margin-bottom:14px;font-weight:700;}
.hl-hero .hh-title{margin:0 0 10px;font-size:1.4rem;font-weight:800;color:#182228;}
.hl-hero .hh-desc{max-width:82%;font-size:.92rem;line-height:1.85;color:#3d4d52;position:relative;}
.hl-hero .hh-desc code{background:#e3ddc9;padding:1px 6px;border-radius:4px;color:#0e3746;font-weight:700;}
.hl-hero .hh-badges{display:flex;gap:10px;flex-wrap:wrap;margin-top:18px;}
.hl-hero .hh-badge{background:#f4f2ec;border:1.5px solid #b9c3c6;padding:6px 14px;border-radius:3px;font-size:.78rem;font-weight:600;color:#233238;box-shadow:0 6px 14px -8px rgba(14,55,70,.3);}
.hl-card-grid{display:flex;flex-wrap:wrap;gap:16px;margin:16px 0 26px;}
.hl-card-grid .hc-card{flex:1 1 220px;background:#f4f2ec;border:1.5px solid #d7cfba;border-radius:14px;padding:20px 20px 22px;box-shadow:0 8px 20px -8px rgba(14,55,70,.22);border-top:4px solid #0e3746;}
.hl-card-grid .hc-badge{display:inline-flex;align-items:center;justify-content:center;width:34px;height:34px;border-radius:6px;background:#0e3746;color:#f4f2ec;font-weight:800;font-size:.85rem;margin-bottom:10px;}
.hl-card-grid .hc-card.red{border-top-color:#be2623;}
.hl-card-grid .hc-card.red .hc-badge{background:#be2623;color:#fff;}
.hl-card-grid .hc-card.tan{border-top-color:#a9946a;}
.hl-card-grid .hc-card.tan .hc-badge{background:#a9946a;color:#fff;}
.hl-card-grid .hc-title{font-weight:800;font-size:.96rem;color:#182228;margin-bottom:6px;}
.hl-card-grid .hc-desc{font-size:.85rem;line-height:1.75;color:#3d4d52;}
.hl-card-grid .hc-desc code{background:#e3ddc9;padding:1px 6px;border-radius:4px;color:#0e3746;font-size:.85em;font-weight:600;}
.hl-pill-flow{display:flex;align-items:center;flex-wrap:wrap;gap:12px;margin:20px 0 26px;}
.hl-pill-flow .hp-step{background:#f4f2ec;border:2px solid #0e3746;color:#0e3746;border-radius:4px;padding:10px 20px;font-size:.85rem;font-weight:700;box-shadow:0 6px 14px -8px rgba(14,55,70,.3);}
.hl-pill-flow .hp-step.red{border-color:#be2623;color:#8f1c1a;}
.hl-pill-flow .hp-step.tan{border-color:#a9946a;color:#6b5a37;}
.hl-pill-flow .hp-step.muted{border-color:#b9c3c6;color:#48585d;background:#eceae1;}
.hl-pill-flow .hp-arrow{color:#0e3746;font-size:1.15rem;font-weight:900;}
.hl-callout{border-radius:8px;padding:16px 20px;margin:16px 0;display:flex;gap:12px;align-items:flex-start;border:1.5px solid transparent;border-left-width:5px;}
.hl-callout .hc-icon{font-size:1.2rem;flex:none;}
.hl-callout .hc-body{flex:1;min-width:0;}
.hl-callout p{margin:0;font-size:.9rem;color:#233238;line-height:1.7;}
.hl-callout p+p,.hl-callout p+ul,.hl-callout ul+p{margin-top:10px;}
.hl-callout p strong{color:#182228;}
.hl-callout.success{background:#e3edea;border-color:#9db8b0;border-left-color:#0e3746;}
.hl-callout.warn{background:#f3ecd8;border-color:#d9c48f;border-left-color:#a9946a;}
.hl-callout.danger{background:#f8e2e0;border-color:#e29c98;border-left-color:#be2623;}
.hl-callout.info{background:#e6edef;border-color:#a9bcc2;border-left-color:#37606b;}
.hl-table{width:100%;border-collapse:collapse;margin:16px 0 26px;background:#f4f2ec;border:1.5px solid #d7cfba;border-radius:10px;overflow:hidden;box-shadow:0 8px 20px -8px rgba(14,55,70,.2);}
.hl-table th{background:#0e3746;color:#f4f2ec;font-size:.8rem;text-align:left;padding:12px 16px;border-bottom:1.5px solid #d7cfba;font-weight:700;}
.hl-table td{padding:12px 16px;font-size:.85rem;color:#3d4d52;border-bottom:1px solid #e3ddc9;line-height:1.6;}
.hl-table tr:last-child td{border-bottom:none;}
.hl-table code{background:#e3ddc9;padding:1px 6px;border-radius:4px;color:#0e3746;font-size:.88em;font-weight:600;}
.hl-layer-flow{display:flex;flex-direction:column;gap:10px;margin:20px 0 30px;}
.hl-layer-flow .ml-row{display:flex;align-items:stretch;gap:14px;}
.hl-layer-flow .ml-badge{flex:none;width:40px;height:40px;border-radius:6px;background:#e3ddc9;color:#0e3746;font-weight:800;font-size:.9rem;display:flex;align-items:center;justify-content:center;box-shadow:0 6px 14px -8px rgba(14,55,70,.35);}
.hl-layer-flow .ml-row.core .ml-badge{background:#be2623;color:#fff;}
.hl-layer-flow .ml-body{flex:1;background:#f4f2ec;border:1.5px solid #d7cfba;border-radius:10px;padding:12px 18px;box-shadow:0 6px 16px -10px rgba(14,55,70,.2);}
.hl-layer-flow .ml-title{font-weight:800;font-size:.9rem;color:#182228;margin-bottom:2px;}
.hl-layer-flow .ml-title code{background:#e3ddc9;padding:1px 6px;border-radius:4px;color:#0e3746;font-size:.85em;font-weight:600;}
.hl-layer-flow .ml-desc{font-size:.8rem;color:#3d4d52;line-height:1.6;}
.hl-layer-flow .ml-io-badge{display:inline-flex;align-items:center;gap:4px;font-size:.68rem;font-weight:700;padding:2px 8px;border-radius:20px;margin-left:8px;vertical-align:middle;}
.hl-layer-flow .ml-io-badge.io{background:#f8e2e0;color:#8f1c1a;}
.hl-layer-flow .ml-io-badge.cpu{background:#e6edef;color:#274952;}
.hl-layer-flow .ml-connector{width:40px;display:flex;justify-content:center;}
.hl-layer-flow .ml-connector .ml-line{width:3px;flex:none;background:#0e3746;opacity:.35;height:20px;margin:0 auto;}
.hl-erd{margin:22px 0 30px;padding:26px 24px;background:#f4f2ec;border:1.5px solid #d7cfba;border-radius:14px;box-shadow:0 10px 28px -12px rgba(14,55,70,.28);overflow-x:auto;}
.hl-erd .erd-main{display:flex;align-items:center;gap:0;min-width:820px;}
.hl-erd .erd-node{flex:none;border-radius:8px;padding:12px 14px;text-align:center;font-weight:800;font-size:.82rem;box-shadow:0 6px 14px -8px rgba(14,55,70,.3);min-width:110px;}
.hl-erd .erd-node em{display:block;font-style:italic;font-weight:600;font-size:.7rem;margin-top:3px;opacity:.85;}
.hl-erd .erd-node.gold{background:#f0c419;color:#0e3746;}
.hl-erd .erd-node.lav{background:#dcd9f5;color:#2c2a4a;}
.hl-erd .erd-edge{flex:none;display:flex;flex-direction:column;align-items:center;justify-content:center;padding:0 6px;min-width:46px;}
.hl-erd .erd-edge .erd-line{width:100%;height:2px;background:#0e3746;opacity:.4;}
.hl-erd .erd-edge .erd-card{font-size:.68rem;font-weight:800;color:#8a7a52;background:#f3ecd8;border:1px solid #d9c48f;border-radius:10px;padding:1px 8px;margin-bottom:3px;}
.hl-erd .erd-branch{display:flex;align-items:center;margin-left:150px;margin-top:14px;gap:0;}
.hl-erd .erd-branch-stack{display:flex;flex-direction:column;gap:10px;}
.hl-steps{display:flex;flex-direction:column;gap:12px;margin:16px 0 22px;}
.hl-steps .hs-card{background:#f4f2ec;border:1px solid #d7cfba;border-radius:8px;padding:14px 18px 14px 58px;position:relative;box-shadow:0 6px 16px -10px rgba(14,55,70,.2);}
.hl-steps .hs-num{position:absolute;left:14px;top:14px;width:28px;height:28px;border-radius:6px;background:linear-gradient(135deg,#154a5c,#0e3746);color:#f4f2ec;font-weight:800;font-size:.85rem;display:flex;align-items:center;justify-content:center;}
.hl-steps .hs-card.warn .hs-num{background:linear-gradient(135deg,#c9bc98,#a9946a);}
.hl-steps .hs-title{font-weight:800;font-size:.9rem;color:#182228;margin-bottom:4px;display:flex;align-items:center;gap:8px;flex-wrap:wrap;}
.hl-steps .hs-desc{font-size:.84rem;color:#3d4d52;line-height:1.65;margin:0;}
.hl-steps .hs-desc code{background:#e3ddc9;padding:1px 6px;border-radius:4px;color:#0e3746;font-size:.9em;font-weight:600;}
.hl-checklist{list-style:none;padding:0;margin:16px 0;}
.hl-checklist li{position:relative;padding:10px 0 10px 32px;border-bottom:1px dashed #d7cfba;color:#3d4d52;}
.hl-checklist li:last-child{border-bottom:none;}
.hl-checklist li::before{content:"✓";position:absolute;left:0;top:12px;width:20px;height:20px;border-radius:4px;background:#0e3746;color:#f4f2ec;font-size:.72rem;display:flex;align-items:center;justify-content:center;font-weight:900;}
.hl-checklist li code{background:#e3ddc9;padding:1px 6px;border-radius:4px;color:#0e3746;font-weight:600;}
details.ml-detail{margin-top:8px;background:#f4f2ec;border:1.5px solid #d7cfba;border-radius:10px;padding:12px 18px;}
details.ml-detail summary{cursor:pointer;font-size:.86rem;font-weight:700;color:#37606b;list-style:none;user-select:none;}
details.ml-detail summary::before{content:"▸ ";}
details.ml-detail[open] summary::before{content:"▾ ";}
details.ml-detail summary::-webkit-details-marker{display:none;}
details.ml-detail p{font-size:.85rem;color:#3d4d52;line-height:1.7;margin:10px 0 0;}
details.ml-detail p code{background:#e3ddc9;padding:1px 6px;border-radius:4px;color:#0e3746;font-weight:600;}
.hl-seal-divider{display:flex;align-items:center;justify-content:center;gap:14px;margin:34px 0;color:#a9946a;}
.hl-seal-divider::before,.hl-seal-divider::after{content:"";flex:1;height:1px;background:linear-gradient(90deg,transparent,#a9946a,transparent);}
.hl-seal-divider span{font-size:1.3rem;}
</style>


<div class="hl-hero">
<div class="hh-tag">情境導覽</div>
<div class="hh-title">購物車裡的商品，為什麼有些會綁在一起</div>
<div class="hh-desc">電商購物車常有「主商品綁定子商品」的業務玩法，子商品的金流、物流與活動資格都必須跟著主商品連動。然而負責記錄購物車的 <code>ShoppingCart</code> 天生是一列只記一個 SKU 的單品項明細表，兩者產生了結構矛盾。這篇文章將拆解系統如何透過輔助表 <code>ShoppingCartExtendInfo</code> 補足這層分組關聯。</div>
<div class="hh-badges">
<span class="hh-badge">資料表關聯</span>
<span class="hh-badge">組合商品</span>
<span class="hh-badge">加價購</span>
<span class="hh-badge">ShoppingCartExtendInfo</span>
</div>
</div>

<div class="hl-card-grid">
<div class="hc-card tan">
<div class="hc-badge">組</div>
<div class="hc-title">玩法一：組合商品</div>
<div class="hc-desc">例如相機加電池加腳架以套裝價一起銷售，缺一不可，整組享有統一的套裝優惠與規則。</div>
</div>
<div class="hc-card red">
<div class="hc-badge">加</div>
<div class="hc-title">玩法二：加價購</div>
<div class="hc-desc">例如購買洗髮精（主商品）後，獲得資格以優惠折扣加購潤髮乳（子商品）。</div>
</div>
</div>

<div class="hl-callout warn">
<span class="hc-icon">⚡</span>
<div class="hc-body">
<p><strong>業務需求與資料庫設計的矛盾：</strong></p>
<ul class="hl-checklist">
<li><strong>業務共同特徵：</strong>購物車內必須明確表達「一個主商品搭配多個子商品」的主從關係，且子商品不可脫離主商品獨立存在。</li>
<li><strong>底層結構限制：</strong><code>ShoppingCart</code> 資料表設計為單列單品項明細，每一列只有自身欄位，完全沒有空間記錄群組關係。</li>
<li><strong>解法核心：</strong>引入周邊輔助表 <code>ShoppingCartExtendInfo</code> 作為分組貼紙，透過共用 <code>ItemGroup</code> 串接主子列。</li>
</ul>
</div>
</div>

<!-- endtab -->


<!-- tab ShoppingCart：單列單品項明細表-->

<div class="hl-hero">
<div class="hh-tag">資料表結構</div>
<div class="hh-title">ShoppingCart：每一列就是一個 SKU 項目</div>
<div class="hh-desc"><code>ShoppingCart</code> 表的主鍵是單一自增序號 <code>ShoppingCart_Id</code>，不是複合鍵，代表每一列本身就是一筆可以獨立定址的商品明細，而不是某種容器或群組的代表列。</div>
</div>

<table class="hl-table">
<tr><th>欄位</th><th>用途</th></tr>
<tr><td><code>ShoppingCart_Id</code></td><td>主鍵，單一自增序號，一列代表一個商品項目</td></tr>
<tr><td><code>ShoppingCart_SalePageId</code></td><td>商品頁序號</td></tr>
<tr><td><code>ShoppingCart_SaleProductId</code></td><td>商品序號</td></tr>
<tr><td><code>ShoppingCart_SaleProductSKUId</code></td><td>商品 SKU 序號</td></tr>
<tr><td><code>ShoppingCart_Qty</code></td><td>購買數量，單一整數欄位</td></tr>
<tr><td><code>ShoppingCart_IsMajor</code></td><td>是否為主件</td></tr>
<tr><td><code>ShoppingCart_IsExtra</code></td><td>是否為加購商品</td></tr>
<tr><td><code>ShoppingCart_IsGift</code></td><td>是否為買就送的贈品</td></tr>
<tr><td><code>ShoppingCart_OptionalTypeId</code></td><td>選項類型序號，組合商品用來對應選配區塊</td></tr>
<tr><td><code>ShoppingCart_OptionalTypeDef</code></td><td>選項類型定義，例如 <code>SalepageBundle</code>、<code>AddOnsSalepageExtraPurchase</code></td></tr>
</table>

<div class="hl-callout warn">
<span class="hc-icon">🔍</span>
<div class="hc-body">
<p><strong>判斷依據：</strong>從欄位設計可以看出這張表是典型的單列單品項明細表，理由如下：</p>
<ul class="hl-checklist">
<li><code>Qty</code> 是單一整數：如果一列能代表多個商品，數量欄位就無法用一個整數表示，因為不同商品的數量可能不一樣。</li>
<li><code>IsMajor</code>、<code>IsExtra</code>、<code>IsGift</code> 是自身角色旗標：這些欄位都是描述這一列自己在購物車裡扮演的角色，而不是描述一整組商品的狀態。</li>
</ul>
<p>這些欄位設計方式共同證實了這張表類似訂單明細常見的單列單品項設計模式。</p>
</div>
</div>

<div class="hl-callout info">
<span class="hc-icon">🛒</span>
<div class="hc-body">
<p><strong>購物車與資料列的對應關係：一台購物車對應的是一批列，不是一列。</strong><code>ShoppingCart</code> 表裡沒有任何欄位叫做購物車序號，查詢購物車靠的是以下條件與行為：</p>
<ul class="hl-checklist">
<li>查詢鍵：用 <code>ShoppingCart_MemberOrUnloginId</code> 加上商店序號去篩選，而不是靠某一個固定 ID。</li>
<li>查詢條件：<code>CartRepository.GetCartAsync</code> 的 <code>WHERE</code> 條件只鎖定會員或未登入序號、商店、門市這三個維度。</li>
<li>回傳型別：回傳的是一個清單，而不是單一物件。</li>
</ul>
<p>也就是說，同一位會員的購物車，通常會對應到好幾筆各自獨立的 <code>ShoppingCart_Id</code>，每加入一種商品就多一列，購物車本身是這一批列在程式裡被撈出來後組成的邏輯集合，而不是資料庫裡真實存在的某一列或某一張主表。</p>
</div>
</div>

<!-- endtab -->


<!-- tab ShoppingCartExtendInfo：分組貼紙表-->

<div class="hl-hero">
<div class="hh-tag">資料表結構</div>
<div class="hh-title">ShoppingCartExtendInfo：分組貼紙表</div>
<div class="hh-desc">既然 <code>ShoppingCart</code> 沒辦法表達這幾列是同一組，系統另外開了一張輔助表 <code>ShoppingCartExtendInfo</code>，專門記錄誰跟誰是一組、遵循什麼規則、誰是主誰是從。它本身不存商品內容，商品的價格、名稱、庫存都還是查 <code>ShoppingCart</code> 或商品頁資料，這張表純粹扮演分組索引的角色。</div>
</div>

<table class="hl-table">
<tr><th>欄位</th><th>用途</th></tr>
<tr><td><code>ShoppingCartExtendInfo_Id</code></td><td>主鍵，單一自增序號</td></tr>
<tr><td><code>ShoppingCartExtendInfo_ShoppingCartId</code></td><td>外鍵，對應 <code>ShoppingCart_Id</code>，標記這筆延伸資料屬於哪一個商品項目</td></tr>
<tr><td><code>ShoppingCartExtendInfo_ItemGroup</code></td><td>分組識別碼，同一組主子商品共用同一個值</td></tr>
<tr><td><code>ShoppingCartExtendInfo_ItemType</code></td><td>1 代表主商品 <code>Major</code>，2 代表子商品 <code>Sub</code></td></tr>
<tr><td><code>ShoppingCartExtendInfo_RuleTypeDef</code></td><td>規則類型，<code>SalepageBundle</code> 為組合商品，<code>AddOnsSalepageExtraPurchase</code> 為加價購</td></tr>
<tr><td><code>ShoppingCartExtendInfo_ValidFlag</code></td><td>生效標誌，軟刪除機制</td></tr>
</table>

<div class="hl-callout success">
<span class="hc-icon">🏷️</span>
<p><strong>貼紙比喻：</strong>可以把這張表想像成一疊分組貼紙。<code>ShoppingCart</code> 裡原本互不相干的幾筆獨立商品列，被貼上同一個 <code>ItemGroup</code> 編號後，就變成程式眼中同一組，再搭配 <code>ItemType</code> 標明貼紙上寫的是老大還是跟班。</p>
</div>

<div class="hl-callout warn">
<span class="hc-icon">🧷</span>
<div class="hc-body">
<p><strong>為什麼不是直接在 ShoppingCart 加欄位：</strong>分組資訊描述的是這一列跟別的列有什麼關係，而不是這一列自己的屬性，拆成獨立表有以下幾個理由：</p>
<ul class="hl-checklist">
<li><strong>關聯而非屬性：</strong>主商品那一列需要表達的其實是有哪些子商品跟著我，這是一份長度不固定的清單，無法用單一欄位存下來，只能靠 <code>ItemGroup</code> 這種共用值去關聯多筆資料。</li>
<li><strong>避免稀疏資料：</strong>只有少數商品項目會用到分組，大多數還是一般商品，如果把 <code>ItemGroup</code>、<code>ItemType</code>、<code>RuleTypeDef</code> 硬塞進 <code>ShoppingCart</code>，會讓九成以上的列白白多出幾乎用不到的欄位，而 <code>ShoppingCart</code> 又是加入購物車、結帳都會查詢的高頻表，欄位越寬對效能越不利。</li>
<li><strong>因應規則演進：</strong>拆成獨立表之後，未來如果要新增第三種、第四種分組規則，只需要擴充這張周邊表，完全不影響 <code>ShoppingCart</code> 原本的結構與既有查詢邏輯。</li>
</ul>
</div>
</div>

<!-- endtab -->


<!-- tab 兩表如何用外鍵串起來-->

<div class="hl-hero">
<div class="hh-tag">關聯方式</div>
<div class="hh-title">兩表如何用外鍵串起來</div>
<div class="hh-desc">兩張表之間沒有正式的資料庫外鍵約束，關聯完全靠程式查詢時的 <code>WHERE</code> 條件比對達成。</div>
</div>

<div class="hl-erd">
<div class="erd-main">
<div class="erd-node gold">商品明細<em>ShoppingCart</em></div>
<div class="erd-edge"><div class="erd-card">1 對 0～1</div><div class="erd-line"></div></div>
<div class="erd-node lav">分組貼紙<em>ShoppingCartExtendInfo</em></div>
</div>
</div>

<table class="hl-table">
<tr><th>ShoppingCart 端</th><th>對應方向</th><th>ShoppingCartExtendInfo 端</th></tr>
<tr><td><code>ShoppingCart_Id</code>（主鍵）</td><td>比對</td><td><code>ShoppingCartExtendInfo_ShoppingCartId</code>（外鍵）</td></tr>
<tr><td>商品明細：<code>SalePageId</code>／<code>SKU</code>／<code>Qty</code></td><td>不重複儲存</td><td>分組資訊：<code>ItemGroup</code>／<code>ItemType</code>／<code>RuleTypeDef</code></td></tr>
</table>

<div class="hl-callout info">
<span class="hc-icon">🧩</span>
<p><strong>沒有記錄就是一般商品：</strong>如果某個 <code>ShoppingCart_Id</code> 在 <code>ShoppingCartExtendInfo</code> 裡完全查不到對應記錄，代表這個商品項目是單純的一般商品，不屬於任何組合商品或加價購分組。這也是程式判斷要不要進入分組邏輯的依據。</p>
</div>

<div class="hl-callout warn">
<span class="hc-icon">⚙️</span>
<p><strong>為什麼一個 ShoppingCartId 對應到 0 或 1 筆延伸記錄：</strong>每個商品項目在同一時間只會屬於一組關聯，要嘛是某個組合商品的一員，要嘛是某個加價購的一員，要嘛完全不屬於任何分組，所以對應關係最多只有一筆。真正把多筆商品串成一組的關鍵，是多個不同 <code>ShoppingCartId</code> 的延伸記錄共用同一個 <code>ItemGroup</code> 值。</p>
</div>

<!-- endtab -->


<!-- tab 實際案例：三種商品型態-->

<div class="hl-hero">
<div class="hh-tag">實際案例</div>
<div class="hh-title">一般商品、組合商品、加購品同時出現在購物車</div>
<div class="hh-desc">假設某位會員的購物車裡同時放了三種型態的商品：一件單純的 <code>T-shirt</code>，一組相機加電池加腳架的組合商品，以及洗髮精加購潤髮乳的加價購商品。以下用實際欄位示範這六筆商品在兩張表裡分別長什麼樣子。</div>
</div>

<table class="hl-table">
<tr><th>CartId</th><th>商品</th><th>Qty</th><th>IsMajor</th><th>IsExtra</th><th>OptionalTypeId</th><th>OptionalTypeDef</th></tr>
<tr><td><code>1001</code></td><td>T-shirt</td><td>2</td><td>true</td><td>false</td><td>0</td><td>（空白）</td></tr>
<tr><td><code>1002</code></td><td>相機（主商品）</td><td>1</td><td>true</td><td>false</td><td>0</td><td>（空白）</td></tr>
<tr><td><code>1003</code></td><td>電池（子商品）</td><td>2</td><td>false</td><td>false</td><td>10（電池選配區）</td><td><code>SalepageBundle</code></td></tr>
<tr><td><code>1004</code></td><td>腳架（子商品）</td><td>1</td><td>false</td><td>false</td><td>11（腳架選配區）</td><td><code>SalepageBundle</code></td></tr>
<tr><td><code>1005</code></td><td>洗髮精（主商品）</td><td>1</td><td>true</td><td>false</td><td>0</td><td>（空白）</td></tr>
<tr><td><code>1006</code></td><td>潤髮乳（子商品）</td><td>1</td><td>false</td><td>true</td><td>0</td><td><code>AddOnsSalepageExtraPurchase</code></td></tr>
</table>

<table class="hl-table">
<tr><th>ShoppingCartId</th><th>ItemGroup</th><th>ItemType</th><th>RuleTypeDef</th></tr>
<tr><td><code>1002</code>（相機）</td><td>9001</td><td>Major</td><td>SalepageBundle</td></tr>
<tr><td><code>1003</code>（電池）</td><td>9001</td><td>Sub</td><td>SalepageBundle</td></tr>
<tr><td><code>1004</code>（腳架）</td><td>9001</td><td>Sub</td><td>SalepageBundle</td></tr>
<tr><td><code>1005</code>（洗髮精）</td><td>9002</td><td>Major</td><td>AddOnsSalepageExtraPurchase</td></tr>
<tr><td><code>1006</code>（潤髮乳）</td><td>9002</td><td>Sub</td><td>AddOnsSalepageExtraPurchase</td></tr>
</table>

<div class="hl-callout success">
<span class="hc-icon">🔗</span>
<p><strong>注意：</strong><code>1001</code>（T-shirt）完全沒有出現在延伸表裡，這正是一般商品的特徵。<code>ItemGroup</code> 為 9001 的一組把相機、電池、腳架三筆原本互不相干的獨立列串成組合商品，<code>ItemGroup</code> 為 9002 的一組則把洗髮精、潤髮乳串成加價購，兩組互不干擾。</p>
</div>

<div class="hl-card-grid">
<div class="hc-card">
<div class="hc-badge">一</div>
<div class="hc-title">一般商品：T-shirt</div>
<div class="hc-desc">只出現在 <code>ShoppingCart</code>，沒有任何延伸記錄，程式讀到後直接當獨立商品處理，不會進入分組邏輯。</div>
</div>
<div class="hc-card tan">
<div class="hc-badge">組</div>
<div class="hc-title">組合商品：相機組</div>
<div class="hc-desc">相機是主商品，電池與腳架是子商品，三者共用 <code>ItemGroup 9001</code>，子商品各自的 <code>OptionalTypeId</code> 對應不同選配區塊，用來套用各自的購買數量規則。</div>
</div>
<div class="hc-card red">
<div class="hc-badge">加</div>
<div class="hc-title">加價購：洗髮精組</div>
<div class="hc-desc">洗髮精是主商品，潤髮乳是加購子商品，共用 <code>ItemGroup 9002</code>，子商品的 <code>IsExtra</code> 標記為 true，代表這是靠加購資格才能買到的商品。</div>
</div>
</div>

<!-- endtab -->


<!-- tab 讀取流程：程式怎麼拼回關聯-->

<div class="hl-hero">
<div class="hh-tag">讀取流程</div>
<div class="hh-title">建立購物車時，程式怎麼把兩張表拼回關聯</div>
<div class="hh-desc">建立購物車時，程式會分兩步驟先各自查詢兩張表，再用 <code>ItemGroup</code> 把資料重新拼接回一個帶有主子關係的物件結構。</div>
</div>

<div class="hl-pill-flow">
<div class="hp-step">查詢 ShoppingCart 取得商品明細清單</div>
<span class="hp-arrow">→</span>
<div class="hp-step">查詢 ShoppingCartExtendInfo 取得分組記錄</div>
<span class="hp-arrow">→</span>
<div class="hp-step tan">依 ItemGroup 把分組資訊補回商品物件</div>
<span class="hp-arrow">→</span>
<div class="hp-step red">後續金物流、活動、數量規則才能運作</div>
</div>

<table class="hl-table">
<tr><th>階段</th><th>ShoppingCart 查詢</th><th>ShoppingCartExtendInfo 查詢</th><th>記憶體物件</th></tr>
<tr><td>1</td><td>依會員或未登入序號查詢，回傳六筆商品明細</td><td>尚未執行</td><td>尚未處理</td></tr>
<tr><td>2</td><td>已完成</td><td>依 CartId 清單查出分組記錄</td><td>尚未處理</td></tr>
<tr><td>3</td><td>已完成</td><td>已完成</td><td>補上 ItemGroup／ItemType／CartExtendInfos</td></tr>
</table>

<div class="hl-callout info">
<span class="hc-icon">🧭</span>
<p><strong>一般商品的處理方式：</strong>一般商品直接沿用 <code>ShoppingCart</code> 查詢結果，不需要額外處理，因為在 <code>ShoppingCartExtendInfo</code> 裡完全查不到對應的分組記錄。</p>
</div>

<details class="ml-detail">
<summary>展開：程式怎麼判斷一組資料有沒有問題</summary>
<ol>
<li>把 <code>ShoppingCartExtendInfo</code> 的查詢結果依 <code>ItemGroup</code> 分組，逐一比對每一組裡有沒有主商品。</li>
<li>如果一組裡找不到主商品，代表資料異常，整組會直接從購物車移除。</li>
<li>如果分組規則對應到已關閉的功能開關，例如組合商品開關被關閉，也會把整組從購物車移除。</li>
<li>確認分組有效後，把 <code>CartExtendInfoItemGroup</code>、<code>CartExtendInfoItemTypeDef</code> 這些欄位寫回對應的商品物件，並且在主商品身上記錄一份子商品的 <code>CartId</code> 清單，方便後續步驟快速找到同一組裡的其他成員。</li>
</ol>
</details>

<ul class="hl-checklist">
<li>組合商品的購買數量上限計算，需要依 <code>ItemGroup</code> 找出同一組的所有子商品，取各子商品可賣量除以必購量的最小值，決定主商品最多能賣幾組。</li>
<li>加價購活動有效性驗證，需要先知道哪個商品是加購子商品，才能查詢對應的加購檔期是否還在有效期間內，沒中活動就整組拆解。</li>
<li>金物流繼承，子商品的付款方式與配送方式不會各自查詢，而是直接複製主商品的結果，避免同一組商品出現金物流互相矛盾的情況。</li>
</ul>

<!-- endtab -->


<!-- tab 重點回顧-->

<div class="hl-hero">
<div class="hh-tag">總結</div>
<div class="hh-title">重點回顧</div>
</div>

<table class="hl-table">
<tr><th>概念</th><th>定位</th></tr>
<tr><td><code>ShoppingCart</code></td><td>單列單品項明細表</td></tr>
<tr><td><code>ShoppingCartExtendInfo</code></td><td>分組貼紙表</td></tr>
<tr><td><code>ItemGroup</code></td><td>串接分組的關鍵值</td></tr>
</table>

<div class="hl-callout success">
<span class="hc-icon">🧾</span>
<div class="hc-body">
<p><strong>核心結論：</strong></p>
<ul class="hl-checklist">
<li><code>ShoppingCart</code> 存的是每個商品項目各自獨立的明細，價格、數量、選項都齊全，但完全不知道商品之間的主從關係。</li>
<li><code>ShoppingCartExtendInfo</code> 不存商品內容，純粹記錄哪幾筆商品共用同一個 <code>ItemGroup</code>、誰是主誰是從、遵循哪一種業務規則。</li>
</ul>
<p>程式在建立購物車時必須把兩張表的查詢結果重新拼接，才能重建出相機加電池加腳架是一組、洗髮精加潤髮乳是一組、T-shirt 自己一國的真實購物車樣貌，這也是後續組合商品數量計算、加購活動驗證、金物流繼承等規則的共同基礎。</p>
</div>
</div>

<div class="hl-seal-divider"><span>⚓</span></div>

<!-- endtab -->


{% endtabs %}
