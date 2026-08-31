---
title: Shopping-Cart-Auth
date: 2026-08-18 16:35:00
categories: Shopping-Cart
top_img: https://github.com/CHI-KEKE/pics/blob/main/Codesss/Arch/ecom/Shopping-Cart/cart-create-auth-landing.png?raw=true
cover: https://github.com/CHI-KEKE/pics/blob/main/Codesss/Arch/ecom/Shopping-Cart/cart-create-auth-landing.png?raw=true
tags:
toc:
toc_number:
comments:
---

{% tabs shopping-cart-auth %}

<!-- tab 同一支 API，有三種不同的呼叫來源-->

<style>
.mint-hero{position:relative;overflow:hidden;border-radius:22px;padding:36px 40px;margin:24px 0 30px;background:linear-gradient(160deg,#eefaf6 0%,#f6faf8 60%,#fdf6ea 100%);border:1px solid #e3ede9;}
.mint-hero .mh-tag{display:inline-block;font-size:.72rem;letter-spacing:.14em;text-transform:uppercase;background:#3fb99a;color:#fff;padding:5px 16px;border-radius:999px;margin-bottom:14px;font-weight:700;}
.mint-hero .mh-title{margin:0 0 10px;font-size:1.4rem;font-weight:800;color:#26333a;}
.mint-hero .mh-desc{max-width:82%;font-size:.92rem;line-height:1.85;color:#5a6b70;}
.mint-hero .mh-desc code{background:#e9f7f2;padding:1px 6px;border-radius:5px;color:#1f7a63;}
.mint-hero .mh-badges{display:flex;gap:10px;flex-wrap:wrap;margin-top:18px;}
.mint-hero .mh-badge{background:#ffffff;border:1px solid #e3ede9;padding:6px 14px;border-radius:999px;font-size:.78rem;color:#5a6b70;box-shadow:0 6px 14px -8px rgba(60,120,100,.25);}
.mint-token-row{display:flex;flex-wrap:wrap;gap:20px;margin:18px 0 28px;padding:22px 24px;background:#ffffff;border:1px solid #e3ede9;border-radius:20px;box-shadow:0 8px 20px -8px rgba(60,120,100,.14);}
.mint-token-row .mt-item{display:flex;flex-direction:column;align-items:center;gap:8px;flex:0 0 auto;min-width:90px;}
.mint-token-row .mt-swatch{width:52px;height:52px;border-radius:50%;box-shadow:inset 0 0 0 1px rgba(0,0,0,.05);}
.mint-token-row .mt-swatch.mint{background:#a7e8d6;}
.mint-token-row .mt-swatch.amber{background:#f3d9a8;}
.mint-token-row .mt-swatch.blue{background:#bcd7f2;}
.mint-token-row .mt-label{font-size:.8rem;font-weight:700;color:#26333a;}
.mint-token-row .mt-code{font-family:Consolas,'SFMono-Regular',Menlo,monospace;font-size:.68rem;color:#8a9aa0;}
.mint-card-grid{display:flex;flex-wrap:wrap;gap:16px;margin:16px 0 26px;}
.mint-card-grid .mc-card{flex:1 1 220px;background:#ffffff;border:1px solid #e3ede9;border-radius:20px;padding:20px 20px 22px;box-shadow:0 8px 20px -8px rgba(60,120,100,.18);}
.mint-card-grid .mc-badge{display:inline-flex;align-items:center;justify-content:center;width:34px;height:34px;border-radius:50%;background:#eafaf4;color:#1f7a63;font-weight:800;font-size:.85rem;margin-bottom:10px;}
.mint-card-grid .mc-card.amber .mc-badge{background:#fdf3e2;color:#a06a1f;}
.mint-card-grid .mc-card.blue .mc-badge{background:#e9f2fc;color:#2b6099;}
.mint-card-grid .mc-title{font-weight:800;font-size:.96rem;color:#26333a;margin-bottom:6px;}
.mint-card-grid .mc-desc{font-size:.85rem;line-height:1.75;color:#5a6b70;}
.mint-card-grid .mc-desc code{background:#eef4f2;padding:1px 6px;border-radius:5px;color:#1f7a63;font-size:.85em;}
.mint-pill-flow{display:flex;align-items:center;flex-wrap:wrap;gap:12px;margin:20px 0 26px;}
.mint-pill-flow .mp-step{background:#ffffff;border:1.5px solid #3fb99a;color:#1f7a63;border-radius:999px;padding:10px 20px;font-size:.85rem;font-weight:700;box-shadow:0 6px 14px -8px rgba(60,120,100,.25);}
.mint-pill-flow .mp-step.amber{border-color:#e8b563;color:#a06a1f;}
.mint-pill-flow .mp-step.blue{border-color:#7fb0e0;color:#2b6099;}
.mint-pill-flow .mp-step.muted{border-color:#cbd6d2;color:#7c8a86;background:#f4f7f6;}
.mint-pill-flow .mp-arrow{color:#9db3ae;font-size:1.15rem;font-weight:900;}
.mint-callout{border-radius:16px;padding:16px 20px;margin:16px 0;display:flex;gap:12px;align-items:flex-start;border:1px solid transparent;}
.mint-callout .mc-icon{font-size:1.2rem;flex:none;}
.mint-callout p{margin:0;font-size:.9rem;color:#3c4a48;line-height:1.7;}
.mint-callout p strong{color:#26333a;}
.mint-callout.success{background:#eafaf1;border-color:#bdeed7;}
.mint-callout.warn{background:#fdf6e6;border-color:#f3dfa7;}
.mint-callout.danger{background:#fdeeed;border-color:#f3c5c1;}
.mint-callout.info{background:#eaf3fc;border-color:#c3dcf3;}
.mint-checklist{list-style:none;padding:0;margin:16px 0;}
.mint-checklist li{display:flex;gap:12px;align-items:flex-start;padding:9px 0;border-bottom:1px dashed #e3ede9;color:#5a6b70;font-size:.88rem;}
.mint-checklist li:last-child{border-bottom:none;}
.mint-checklist li::before{content:"✓";flex:none;width:22px;height:22px;border-radius:50%;background:#dcf5eb;color:#1f7a63;font-size:.72rem;display:flex;align-items:center;justify-content:center;margin-top:1px;font-weight:900;}
.mint-table-wrap{display:inline-block;max-width:100%;background:#ffffff;border:1px solid #e3ede9;border-radius:16px;overflow:hidden;margin:16px 0;box-shadow:0 8px 20px -8px rgba(60,120,100,.14);}
.mint-table-wrap table{border-collapse:collapse;width:auto;font-size:.85rem;}
.mint-table-wrap th{background:#eafaf4;color:#1f7a63;text-align:left;padding:10px 16px;font-weight:800;}
.mint-table-wrap td{padding:10px 16px;border-top:1px solid #e3ede9;color:#3c4a48;line-height:1.6;}
.mint-pre{background:#1c2b26;border-radius:16px;padding:18px 20px;margin:16px 0;color:#eafaf1;font-size:.82rem;line-height:1.7;font-family:Consolas,'SFMono-Regular',Menlo,monospace;white-space:pre;overflow-x:auto;}
.mint-pre .kw{color:#8ee6c4;font-weight:700;}
.mint-pre .st{color:#f3d9a8;}
.mint-pre .cm{color:#7c8a86;font-style:italic;}
</style>

<div class="mint-hero">
<div class="mh-tag">情境說明</div>
<div class="mh-title">建立購物車這支 API，其實同時服務三種不同的呼叫者</div>
<div class="mh-desc">同一支負責建立購物車的 <code>carts/create</code>，可能是消費者用瀏覽器登入後直接呼叫，也可能是合作夥伴系統代替消費者下單，還可能是門市店員在幫客人代客下單。這三種來源的身份驗證方式完全不一樣，這篇要拆解 Shopping 服務怎麼判斷每一次請求究竟屬於哪一種身份，驗證通過之後又還會做哪些「這個人可不可以做這件事」的檢查。</div>
<div class="mh-badges">
<span class="mh-badge">carts/create</span>
<span class="mh-badge">身份驗證</span>
<span class="mh-badge">業務資格檢查</span>
</div>
</div>

<p>先說結論，整支 API 的驗證其實分成兩個完全不同層級，發生的位置與檢查的問題都不一樣，混在一起看容易誤解成同一套機制。</p>

<div class="mint-token-row">
<div class="mt-item"><div class="mt-swatch mint"></div><div class="mt-label">身份驗證</div><div class="mt-code">Authentication</div></div>
<div class="mt-item"><div class="mt-swatch amber"></div><div class="mt-label">授權判斷</div><div class="mt-code">Authorization</div></div>
<div class="mt-item"><div class="mt-swatch blue"></div><div class="mt-label">業務資格檢查</div><div class="mt-code">CartService</div></div>
</div>

<div class="mint-card-grid">
<div class="mc-card">
<div class="mc-badge">1</div>
<div class="mc-title">第一層：身份驗證與授權</div>
<div class="mc-desc">發生在 <code>CartsController.Create</code> 方法執行之前，由 ASP.NET Core 的驗證管線處理，確認這次請求是誰打的，回答的是身份問題。</div>
</div>
<div class="mc-card amber">
<div class="mc-badge">2</div>
<div class="mc-title">第二層：業務層資格檢查</div>
<div class="mc-desc">發生在 <code>CartService.CartCreateAsync</code> 方法內部，是手動寫在程式碼裡的呼叫，確認這個已登入的會員這次操作允不允許，回答的是資格問題，不是身份問題。</div>
</div>
</div>

<div class="mint-callout info">
<span class="mc-icon">💡</span>
<p><strong>白話理解：</strong>一個會員完全可能通過第一層身份驗證，他確實是這個帳號本人，卻在第二層被業務規則擋下來，例如這個會員在商店黑名單裡，或短時間內呼叫太多次觸發限流。這兩層各自獨立，任何一層擋下都會讓整支 API 失敗，但失敗的原因與回應方式不一樣。</p>
</div>



<!-- endtab -->



<!-- tab 三種驗證方式怎麼被自動選出來-->


<p>Shopping 同時支援三種不同的驗證方式，因為 <code>carts/create</code> 這支 API 會被三種不同來源呼叫。系統用一段動態判斷邏輯決定這次要用哪一種驗證方式，判斷順序如下。</p>

<div class="mint-pill-flow">
<div class="mp-step">有 N1-INTERNAL-MEMBER-ID</div>
<div class="mp-arrow">→</div>
<div class="mp-step amber">PartnerApi 驗證</div>
</div>
<div class="mint-pill-flow">
<div class="mp-step">有 NY-SESSION-TOKEN</div>
<div class="mp-arrow">→</div>
<div class="mp-step blue">N1SessionToken 驗證</div>
</div>
<div class="mint-pill-flow">
<div class="mp-step muted">以上都沒有</div>
<div class="mp-arrow">→</div>
<div class="mp-step muted">NineYiCookies 驗證</div>
</div>

<div class="mint-table-wrap">
<table>
<tr><th>驗證方式</th><th>使用時機</th><th>判斷依據</th></tr>
<tr><td>PartnerApi</td><td>合作夥伴系統代替消費者呼叫</td><td>Header 帶 <code>N1-INTERNAL-MEMBER-ID</code></td></tr>
<tr><td>N1SessionToken</td><td>門市店員代客下單</td><td>Header 帶 <code>NY-SESSION-TOKEN</code></td></tr>
<tr><td>NineYiCookies</td><td>消費者用瀏覽器登入後直接呼叫，mweb 走這一種</td><td>以上兩個 Header 都沒有時的預設值</td></tr>
</table>
</div>

<p>這段判斷邏輯寫在應用程式啟動設定裡，程式碼如下。</p>

<pre class="mint-pre"><span class="kw">options</span>.ForwardDefaultSelector = context =&gt;
{
    <span class="cm">//// 優先判斷 PartnerApi（N1-INTERNAL-MEMBER-ID）</span>
    var memberIdStr = context.Request.Headers[<span class="st">"N1-INTERNAL-MEMBER-ID"</span>].ToString();
    if (string.IsNullOrEmpty(memberIdStr) == <span class="kw">false</span>)
        return AuthenticationSchemeEnum.PartnerApi.ToString();
    <span class="cm">//// 其次判斷代客下單（NY-SESSION-TOKEN）</span>
    var sessionToken = context.Request.Headers[<span class="st">"NY-SESSION-TOKEN"</span>].ToString();
    if (string.IsNullOrEmpty(sessionToken) == <span class="kw">false</span>)
        return AuthenticationSchemeEnum.N1SessionToken.ToString();
    <span class="cm">//// 預設使用 NineYiCookies</span>
    return AuthenticationSchemeEnum.NineYiCookies.ToString();
};</pre>



<!-- endtab -->


<!-- tab 一般消費者走的 Cookie 驗證流程-->



<p>mweb 前端呼叫 <code>carts/create</code> 走的是 <code>NineYiCookies</code> 這條路，實際驗證邏輯集中在專門處理 Cookie 驗證的程式裡，每次請求都會經過這段。</p>

<div class="mint-card-grid">
<div class="mc-card">
<div class="mc-badge">1</div>
<div class="mc-title">白名單先擋一次</div>
<div class="mc-desc">先查一份白名單設定，命中的路徑直接放行，完全不做後面的驗證。</div>
</div>
<div class="mc-card amber">
<div class="mc-badge">2</div>
<div class="mc-title">取出加密的登入憑證</div>
<div class="mc-desc">從 Cookie（或對應的 Header）取出 <code>auth</code>／<code>uauth</code> 加密字串。</div>
</div>
<div class="mc-card blue">
<div class="mc-badge">3</div>
<div class="mc-title">依情境判斷登入狀態</div>
<div class="mc-desc">依有沒有 <code>auth</code> 走不同分支，判斷是已登入、免登，還是允許以訪客身份繼續。</div>
</div>
</div>

<p>核心分支邏輯依 <code>auth</code> 存不存在分成兩條路徑，整理成表格如下。</p>

<div class="mint-table-wrap">
<table>
<tr><th>情境</th><th>判斷依據</th><th>結果</th></tr>
<tr><td>有 <code>auth</code> Cookie</td><td>直接解密 Cookie 內容</td><td>解密成功則拿到會員資料，解密失敗視為登入失敗</td></tr>
<tr><td>沒有 <code>auth</code>，但有 <code>uauth</code></td><td>用 <code>uauth</code> 解出匿名裝置編號，去 Redis 查登入快取</td><td>Redis 裡有登入資料且未過期，視為登入成功並要求 Cookie 續期</td></tr>
<tr><td>Redis 查無登入資料，且允許免登</td><td>這次請求的路徑命中設定檔的免登白名單</td><td>建立一個只有裝置編號的訪客身份，不視為登入失敗</td></tr>
<tr><td>Redis 查無登入資料，允許以訪客身份繼續</td><td>路徑命中另一份訪客白名單，且對應商店有開啟訪客購物車功能</td><td>建立明確標記為訪客的身份，購物車後續流程會跳過部分只對真實會員有意義的檢查</td></tr>
<tr><td>以上皆不成立</td><td>沒有任何登入憑證，也不允許免登</td><td>判定為未登入，驗證流程失敗</td></tr>
</table>
</div>




<!-- endtab -->


<!-- tab 合作夥伴與代客下單怎麼驗證-->



<p>另外兩種驗證方式都不走 Cookie，而是各自實作一套驗證流程，格式類似，都是驗證通過後手動把會員資料寫進系統可以識別的憑證裡。</p>

<div class="mint-card-grid">
<div class="mc-card">
<div class="mc-badge">1</div>
<div class="mc-title">PartnerApi 驗證</div>
<div class="mc-desc">先驗證 Header 裡的服務金鑰是否存在於設定檔的合法金鑰清單，再檢查 Header 是否帶有會員編號與商店編號，兩者皆通過後才查出對應的會員資料，查不到則驗證失敗。</div>
</div>
<div class="mc-card amber">
<div class="mc-badge">2</div>
<div class="mc-title">N1SessionToken 驗證</div>
<div class="mc-desc">從 Header 解析 JWT 取得會員編號，同時需要商店編號才能定位商店，兩者都解析成功後同樣查出會員資料，用於門市店員的代客下單場景。</div>
</div>
</div>

<div class="mint-callout warn">
<span class="mc-icon">⚠️</span>
<p><strong>共通點：</strong>這兩種驗證失敗時的處理方式一致，都會回傳固定格式的錯誤訊息，帶著相同的未登入錯誤碼，讓前端可以依錯誤碼統一處理未登入導轉。</p>
</div>




<!-- endtab -->

<!-- tab 不管走哪一種都要通過同一道共用政策-->



<p>不管走哪一種驗證方式，最後都要通過同一道共用的授權政策才算真的過關。</p>

<pre class="mint-pre">builder.Services.AddAuthorization(options =&gt;
{
    options.DefaultPolicy = new AuthorizationPolicyBuilder(AuthenticationSchemeEnum.MultiAuthSchemes.ToString())
                            .RequireClaim(ClaimTypesEnum.LoginMemberEntity.ToString())
                            .Build();
});</pre>

<p>這段的意思很直白，就算前面的驗證流程本身沒有失敗，只要最後沒有成功把代表登入狀態的 Claim 寫進憑證裡，一樣會判定授權失敗，回傳未登入的結果。這代表三種驗證方式雖然實作邏輯完全不同，但對外呈現的授權合格標準是同一個，都是有沒有這個 Claim。</p>

<div class="mint-callout info">
<span class="mc-icon">ℹ️</span>
<p><strong>但取得會員資料的方式並不一樣：</strong>三種驗證方式雖然最後都要滿足同一道 Claim 檢查，但在管線內「怎麼拿到會員資料」其實是分岔的，不是每一種都會即時查詢會員服務。</p>
</div>

<div class="mint-card-grid">
<div class="mc-card">
<div class="mc-badge">Cookie</div>
<div class="mc-title">一般會員驗證 — 不查會員服務</div>
<div class="mc-desc">直接解密 Cookie 本身內含的資料組出會員資料，或是拿解出的識別碼去 Redis 查登入快取，全程不會在管線內另外呼叫會員資料查詢服務。</div>
</div>
<div class="mc-card amber">
<div class="mc-badge">JWT / 金鑰</div>
<div class="mc-title">PartnerApi、N1SessionToken — 會查會員服務</div>
<div class="mc-desc">這兩種都只是先驗證身份資訊本身合法（金鑰存在、JWT 可解析出會員編號），拿到的只是編號，還必須在管線內即時呼叫會員資料查詢服務，用商店編號＋會員編號查出真正的會員資料；查不到就直接判定驗證失敗。</div>
</div>
</div>

<p>換句話說，Cookie 驗證是「資料自帶」（解密或查快取即可），PartnerApi／N1SessionToken 驗證是「先驗證身份合法性，再即時查詢會員資料」，兩種取得會員資料的成本與時機並不相同，但最終都要落地成同一個 Claim 才能通過共用授權政策。</p>



<!-- endtab -->



<!-- tab 進入業務邏輯後還要過兩關-->


<p>身份驗證通過只代表這個人是誰確認無誤，還不代表這次操作一定會被放行。建立購物車的業務邏輯內部還有兩道手動寫在程式碼裡的資格檢查，執行順序如下。</p>

<div class="mint-pill-flow">
<div class="mp-step">限流檢查</div>
<div class="mp-arrow">→</div>
<div class="mp-step amber">會員黑名單檢查</div>
<div class="mp-arrow">→</div>
<div class="mp-step blue">通過才繼續建立購物車</div>
</div>

<div class="mint-card-grid">
<div class="mc-card">
<div class="mc-badge">1</div>
<div class="mc-title">限流檢查</div>
<div class="mc-desc">判斷這個會員短時間內呼叫次數是否超過上限，超過則直接失敗，這部分的完整限流器設計，在先前的限流器分析文章裡已完整拆解。</div>
</div>
<div class="mc-card amber">
<div class="mc-badge">2</div>
<div class="mc-title">會員黑名單檢查</div>
<div class="mc-desc">判斷這個會員能不能繼續使用購物車，黑名單檢查本身也有分支，不是單純的通過或擋下兩種結果。</div>
</div>
</div>

<div class="mint-table-wrap">
<table>
<tr><th>情境</th><th>判斷依據</th><th>處理方式</th></tr>
<tr><td>訪客身份</td><td>被標記為訪客的請求</td><td>直接跳過黑名單檢查，因為根本沒有會員身份可以查</td></tr>
<tr><td>全站黑名單</td><td>黑名單清單中存在整個商城等級的紀錄</td><td>清空這個會員的購物車，整支 API 直接失敗</td></tr>
<tr><td>商店黑名單，且允許付款方式清單為空</td><td>黑名單存在對應商店的紀錄，且查不到任何允許使用的付款方式</td><td>同樣清空購物車並失敗</td></tr>
<tr><td>商店黑名單，但還有允許的付款方式</td><td>黑名單存在對應商店的紀錄，但至少有一種付款方式允許使用</td><td>放行，後續結帳流程只能挑這些付款方式</td></tr>
<tr><td>完全不在任何黑名單</td><td>黑名單查詢結果為空清單</td><td>正常放行，不做任何額外處理</td></tr>
</table>
</div>



<!-- endtab -->

<!-- tab 免登與訪客身份這兩種放行規格有什麼不同-->



<p>前面第一層提到，Cookie 驗證流程裡有兩種不需要登入也能放行的情境，一種單純叫免登，一種是明確標記為訪客身份，兩者容易被誤會成同一件事，這裡補充說明差異。</p>

<div class="mint-callout danger">
<span class="mc-icon">🛑</span>
<p><strong>常見誤解：</strong>以為只要呼叫端傳入某個參數就能免登，實際上這兩種放行規格都不是呼叫端能傳的參數，而是伺服器依這次請求打的 API 路徑，去比對設定檔白名單自己算出來的，前端完全沒有管道能設定或偽造。</p>
</div>

<div class="mint-card-grid">
<div class="mc-card">
<div class="mc-badge">1</div>
<div class="mc-title">免登</div>
<div class="mc-desc">代表這支 API 連匿名裝置都能直接呼叫，語意上仍然視同一般會員規則處理，例如加購物車這種單純的操作。</div>
</div>
<div class="mc-card amber">
<div class="mc-badge">2</div>
<div class="mc-title">訪客身份（P1 免登）</div>
<div class="mc-desc">代表這支 API 允許用一個明確標記為訪客的身份繼續往下走，<code>carts/create</code> 就屬於這一類，需要對應商店開啟訪客購物車功能。</div>
</div>
</div>

<p>兩者產生的會員資料骨架其實一樣，都只有裝置編號，沒有真正的會員編號，差別在多一個訪客標記旗標。<code>carts/create</code> 這條流程裡，建立購物車時要查詢的會員優惠券、會員點數、會員黑名單這三個步驟，都需要真實存在的會員編號才查得到有意義的結果，如果沒有這個訪客標記讓程式碼提早跳過，這幾支呼叫要嘛查不到資料、要嘛拿一個不存在的編號去打外部服務，浪費一次呼叫甚至可能拋出例外。所以 <code>carts/create</code> 必須明確標記成訪客身份，讓這幾個步驟知道要提早跳過。</p>

<div class="mint-callout success">
<span class="mc-icon">✅</span>
<p><strong>一句話總結：</strong>免登跟訪客身份不是使用者狀態，而是後端替每一支 API 貼的存取政策標籤，同一個訪客呼叫不同 API 時可能落在不同分類，完全取決於當下打的是哪一支 API，也不會因為呼叫過幾次而互相升級。</p>
</div>

<ul class="mint-checklist">
<li>整支 carts/create 的驗證分成身份驗證與業務資格檢查兩層，各自獨立</li>
<li>三種驗證方式依 Header 動態選擇，一般消費者走 Cookie 驗證</li>
<li>不管走哪一種驗證方式，最後都要通過同一道共用授權政策才算過關</li>
<li>身份驗證通過後，還要通過限流檢查與會員黑名單檢查才能真的建立購物車</li>
<li>免登與訪客身份是兩種不同的放行規格，不是呼叫端能控制的參數</li>
</ul>



<!-- endtab -->

{% endtabs %}
