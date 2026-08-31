---
title: Concurrency-safe
date: 2026-08-27 08:44:00
categories: DB
top_img: 
cover : 
toc:
toc_number:
comments :

---

{% tabs Concurrency-safe%}

<!-- tab 情境切入-->

<style>
.pg{
display:block;
font-family:'Segoe UI',-apple-system,BlinkMacSystemFont,'PingFang TC','Microsoft JhengHei',sans-serif;
background:
radial-gradient(700px 400px at 94% -6%, rgba(51,103,145,.10), transparent 60%),
radial-gradient(900px 500px at -6% 106%, rgba(28,28,28,.06), transparent 55%),
#fdfdfd;
color:#1c1f22!important;
line-height:1.75;
border-radius:20px;
padding:1px 0 28px;
margin:0 0 20px;
overflow:hidden;
--ink:#1c1f22; --sub:#54606b; --card:#ffffff; --line:#e3e6e9;
--pg:#336791; --pg2:#1a3c56; --outline:#1c1c1c; --teal:#0d9488; --violet:#7c6cf7; --danger:#e5484d;
--shadow:0 10px 30px -12px rgba(28,28,28,.16);
}
.pg *{box-sizing:border-box;}
.pg a{color:var(--pg)!important;text-decoration:none;}
.pg a:hover{text-decoration:underline;}
.pg .pg-hero{padding:48px 6vw 34px;text-align:center;position:relative;overflow:hidden;background:linear-gradient(160deg,var(--pg2),var(--pg));}
.pg .pg-hero::before{content:"🐘 · · · 🐘 · · · 🐘 · · · 🐘";position:absolute;top:14px;left:0;right:0;text-align:center;font-size:.9rem;opacity:.25;letter-spacing:.3em;color:#fff!important;pointer-events:none;}
.pg .pg-eyebrow{display:inline-block;font-size:.78rem;letter-spacing:.16em;color:#fff!important;text-transform:uppercase;font-weight:700;margin-bottom:12px;background:rgba(0,0,0,.28);border:1.5px solid rgba(255,255,255,.55);padding:6px 14px;border-radius:999px;}
.pg .pg-hero h1{font-size:1.7rem;margin:0 0 12px;color:#fff!important;font-weight:800;}
.pg .pg-hero p{color:#dbe8f0!important;max-width:700px;margin:0 auto;font-size:1rem;}
.pg .pg-badges{display:flex;gap:10px;justify-content:center;flex-wrap:wrap;margin-top:20px;}
.pg .pg-badge{background:rgba(255,255,255,.12);border:1.5px solid rgba(255,255,255,.4);padding:6px 14px;border-radius:999px;font-size:.8rem;color:#f2f7fa!important;}
.pg .pg-container{padding:0 5vw;}
.pg section{margin-bottom:44px;}
.pg .pg-section-head{display:flex;align-items:center;gap:12px;margin-bottom:18px;}
.pg .pg-section-head .pg-chip{flex:none;width:38px;height:38px;border-radius:11px;display:flex;align-items:center;justify-content:center;font-weight:800;background:linear-gradient(135deg,var(--pg2),var(--pg));color:#fff!important;font-size:1rem;border:2px solid var(--outline);box-shadow:var(--shadow);}
.pg .pg-section-head.teal .pg-chip{background:linear-gradient(135deg,#0d7a70,#5fd4c4);}
.pg .pg-section-head.violet .pg-chip{background:linear-gradient(135deg,#7c6cf7,#a78bfa);}
.pg .pg-section-head h2{margin:0;font-size:1.3rem;font-weight:800;color:var(--ink)!important;}
.pg h3{font-size:1.08rem;margin:26px 0 10px;color:var(--ink)!important;border-left:4px solid var(--pg);padding-left:10px;}
.pg p{color:var(--sub)!important;}
.pg .pg-card{background:var(--card);border:2.5px solid var(--outline);border-radius:18px;box-shadow:var(--shadow);padding:18px 22px;margin:14px 0;}
.pg table{width:auto;max-width:100%;border-collapse:separate;border-spacing:0;background:var(--card)!important;border:2.5px solid var(--outline);border-radius:16px;overflow:hidden;margin:14px 0 22px;font-size:.88rem;box-shadow:var(--shadow);}
.pg thead th{background:#eef4f8!important;color:var(--ink)!important;text-align:left;padding:13px 16px;font-weight:800;border-bottom:2px solid var(--outline);}
.pg tbody td{padding:13px 16px;border-bottom:1px solid var(--line);color:#2b3742!important;vertical-align:top;background:var(--card)!important;}
.pg tbody tr:last-child td{border-bottom:none;}
.pg tbody tr:hover td{background:#f4f8fb!important;}
.pg tbody td strong{color:var(--ink)!important;}
.pg tbody td code{color:#1a3c56!important;}
.pg code{font-family:Consolas,'SFMono-Regular','Liberation Mono',Menlo,monospace;background:#eaf1f6!important;color:#1a3c56!important;padding:2px 7px;border-radius:6px;font-size:.88em;border:1px solid #d7e2ea;}
.pg pre.pg-pre{background:#0f1f2e!important;border:2.5px solid var(--outline);border-radius:16px;padding:20px 22px;overflow-x:auto;margin:16px 0;position:relative;color:#e8f1f7!important;font-size:.85rem!important;line-height:1.7!important;font-family:Consolas,'SFMono-Regular','Liberation Mono',Menlo,monospace!important;white-space:pre!important;box-shadow:var(--shadow);}
.pg pre.pg-pre::before{content:attr(data-lang);position:absolute;top:12px;right:16px;font-size:.65rem;letter-spacing:.08em;text-transform:uppercase;color:#7d9bad!important;font-family:'Segoe UI',sans-serif;}
.pg .kw{color:#7fc4ff!important;} .pg .str{color:#c3e88d!important;} .pg .cm{color:#5f8299!important;font-style:italic;} .pg .fn{color:#ffb454!important;} .pg .type{color:#7fd0ff!important;} .pg .num{color:#c3e88d!important;} .pg .op{color:#ff6b9d!important;}
.pg .pg-callout{border-radius:16px;padding:18px 22px;margin:18px 0;border:1px solid transparent;display:flex;gap:14px;}
.pg .pg-callout .pg-icon{font-size:1.3rem;flex:none;}
.pg .pg-callout.info{background:#eff6ff;border-color:#bfdbfe;}
.pg .pg-callout.warn{background:#fffbeb;border-color:#fde68a;}
.pg .pg-callout.danger{background:#fef2f2;border-color:#fecaca;}
.pg .pg-callout.ok{background:#ecfdf5;border-color:#a7f3d0;}
.pg .pg-callout p{margin:0;font-size:.92rem;color:#2b3742!important;}
.pg .pg-callout strong{color:var(--ink)!important;}
.pg .pg-tag{display:inline-block;font-size:.72rem;font-weight:800;padding:3px 11px;border-radius:999px;letter-spacing:.02em;}
.pg .pg-tag.danger{background:#fee2e2!important;color:#b91c1c!important;}
.pg .pg-tag.ok{background:#dcfce7!important;color:#15803d!important;}
.pg .pg-tag.warn{background:#fef3c7!important;color:#b45309!important;}
.pg .pg-tag.info{background:#e0f2fe!important;color:#0369a1!important;}
.pg .pg-tag.pg{background:#dfeaf1!important;color:#1a3c56!important;}
.pg .pg-summary-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:16px;margin:18px 0;}
.pg .pg-summary-grid .pg-box{background:var(--card);border:2.5px solid var(--outline);border-radius:16px;padding:18px 20px;box-shadow:var(--shadow);border-top:5px solid var(--pg);position:relative;overflow:hidden;}
.pg .pg-summary-grid .pg-box.pg{border-top-color:var(--pg);}
.pg .pg-summary-grid .pg-box.teal{border-top-color:var(--teal);}
.pg .pg-summary-grid .pg-box.violet{border-top-color:var(--violet);}
.pg .pg-summary-grid .pg-box h4{margin:0 0 8px;color:var(--ink)!important;font-size:.85rem;text-transform:uppercase;letter-spacing:.04em;}
.pg .pg-summary-grid .pg-box p{margin:0;font-size:.87rem;color:var(--sub)!important;}
@media (max-width:780px){.pg .pg-summary-grid{grid-template-columns:1fr;}}
.pg .pg-flow{display:flex;align-items:center;gap:10px;flex-wrap:wrap;margin:20px 0;}
.pg .pg-flow .pg-step{background:var(--card);border:2px solid var(--outline);border-radius:12px;padding:10px 16px;font-size:.85rem;font-weight:700;color:var(--ink)!important;box-shadow:var(--shadow);}
.pg .pg-flow .pg-step.cond{background:#fffbeb;border-color:#c99a1f;color:#b45309!important;}
.pg .pg-flow .pg-arrow{color:var(--pg)!important;font-size:1.2rem;font-weight:900;}
.pg .pg-steps{display:flex;flex-direction:column;gap:14px;margin:16px 0 22px;}
.pg .pg-stepcard{background:var(--card);border:2.5px solid var(--outline);border-radius:14px;padding:16px 20px;position:relative;padding-left:60px;box-shadow:var(--shadow);}
.pg .pg-stepcard .pg-stepnum{position:absolute;left:16px;top:16px;width:30px;height:30px;border-radius:50%;background:linear-gradient(135deg,var(--pg2),var(--pg));color:#fff!important;font-weight:800;font-size:.9rem;display:flex;align-items:center;justify-content:center;border:2px solid var(--outline);}
.pg .pg-stepcard.cond .pg-stepnum{background:linear-gradient(135deg,#b45309,#f7b955);}
.pg .pg-stepcard .pg-step-title{font-weight:700;color:var(--ink)!important;font-size:1rem;margin-bottom:4px;display:flex;align-items:center;gap:8px;flex-wrap:wrap;}
.pg .pg-stepcard p{margin:6px 0;font-size:.9rem;color:var(--sub)!important;}
.pg .pg-checklist{list-style:none;padding:0;margin:16px 0;}
.pg .pg-checklist li{display:flex;gap:12px;align-items:flex-start;padding:10px 0;border-bottom:1px dashed var(--line);color:var(--sub)!important;}
.pg .pg-checklist li:last-child{border-bottom:none;}
.pg .pg-checklist li::before{content:"✓";flex:none;width:22px;height:22px;border-radius:7px;background:#dfeaf1;color:#1a3c56!important;font-size:.75rem;display:flex;align-items:center;justify-content:center;margin-top:2px;font-weight:900;border:1.5px solid var(--outline);}
.pg .pg-evidence{border:2.5px solid var(--outline);background:var(--card);border-radius:18px;box-shadow:var(--shadow);padding:18px 20px;margin:16px 0 22px;}
.pg .pg-evidence .pg-evidence-head{display:flex;align-items:center;gap:8px;margin-bottom:10px;}
.pg .pg-evidence .pg-evidence-head .pg-tag{background:var(--pg2)!important;color:#fff!important;}
.pg .pg-trunk-divider{display:flex;align-items:center;gap:10px;justify-content:center;margin:36px 0;color:var(--pg)!important;}
.pg .pg-trunk-divider::before,.pg .pg-trunk-divider::after{content:"";flex:1;height:1px;background:linear-gradient(90deg,transparent,var(--line) 40%,var(--line) 60%,transparent);}
.pg .pg-trunk-divider span{font-size:1.1rem;}
.pg footer{text-align:center;padding:30px 5vw 6px;color:var(--sub)!important;font-size:.82rem;border-top:1px solid var(--line);margin-top:8px;}
</style>

<div class="pg">
<div class="pg-hero">
<span class="pg-eyebrow">Database · Concurrency Control</span>
<h1>🐘 為什麼加了 Transaction，帳戶餘額還是會算錯</h1>
<p>轉帳這種先查詢、再判斷、最後修改的邏輯，就算包在 <code>BEGIN</code> 與 <code>COMMIT</code> 之間，同時有兩筆請求進來時，餘額還是有可能算錯。原因不是 transaction 沒生效，而是 transaction 保證的東西，跟表面上看起來的並不完全一樣。</p>
</div>
<div class="pg-container">

<section>
<div class="pg-section-head">
<div class="pg-chip">01</div>
<h2>情境</h2>
</div>

<p>有三個帳戶</p>

<table>
<thead><tr><th>帳戶</th><th>餘額</th></tr></thead>
<tbody>
<tr><td>A</td><td>100 元</td></tr>
<tr><td>B</td><td>100 元</td></tr>
<tr><td>C</td><td>100 元</td></tr>
</tbody>
</table>

<p>現在幾乎同時來了兩個請求，一個要從 <code>A</code> 轉 80 元給 <code>B</code>，另一個要從 <code>A</code> 轉 80 元給 <code>C</code>。應用程式常見的寫法是這樣：</p>

<pre class="pg-pre" data-lang="sql"><span class="kw">BEGIN</span>;
balance <span class="op">=</span> <span class="kw">SELECT</span> balance <span class="kw">FROM</span> <span class="type">accounts</span> <span class="kw">WHERE</span> id <span class="op">=</span> <span class="str">'A'</span>;
<span class="kw">IF</span> balance <span class="op">>=</span> <span class="num">80</span> <span class="kw">THEN</span>
    <span class="kw">UPDATE</span> <span class="type">accounts</span> <span class="kw">SET</span> balance <span class="op">=</span> balance <span class="op">-</span> <span class="num">80</span> <span class="kw">WHERE</span> id <span class="op">=</span> <span class="str">'A'</span>;
    <span class="kw">UPDATE</span> <span class="type">accounts</span> <span class="kw">SET</span> balance <span class="op">=</span> balance <span class="op">+</span> <span class="num">80</span> <span class="kw">WHERE</span> id <span class="op">=</span> <span class="str">'B'</span>;
<span class="kw">END IF</span>;
<span class="kw">COMMIT</span>;</pre>

<p>另一個處理另一筆轉帳的執行緒，同時做著幾乎一模一樣的事情，只是把最後的收款帳戶換成 <code>C</code>。</p>

<div class="pg-callout warn">
<span class="pg-icon">⚠️</span>
<p>即使整段邏輯用 <code>BEGIN</code> 與 <code>COMMIT</code> 包起來，也不代表這兩筆轉帳彼此看不到對方、不會互相干擾。因為 transaction 其實同時在處理兩件不同的事，一件是這段操作要嘛全部成功、要嘛全部失敗，另一件是多個 transaction 同時執行時，彼此能看到什麼、會不會互相踩到對方的資料。</p>
</div>

<p>這兩件事分別對應 <code>ACID</code> 的</p>

<div class="pg-summary-grid">
<div class="pg-box pg">
<h4>Atomicity（原子性）</h4>
<p>保證一個 transaction 內的多個操作，要嘛全部成功，要嘛全部沒發生，不會卡在做一半的狀態。</p>
</div>
<div class="pg-box teal">
<h4>Isolation（隔離性）</h4>
<p>決定多個 transaction 同時執行時，彼此能看到什麼版本的資料，能不能同時修改同一筆資料。</p>
</div>
</div>

<p>其中又以 Isolation 最關鍵，因為 Atomicity 只保證單一 transaction 不會做一半失敗，並不保證兩個同時執行的 transaction 不會互相干擾。</p>

</section>
</div>
</div>

<!-- endtab -->

<!-- tab Atomicity-->

<div class="pg">
<div class="pg-container">

<section>
<div class="pg-section-head">
<div class="pg-chip">02</div>
<h2>Atomicity</h2>
</div>

<p>假設一筆轉帳需要兩個操作，先扣款再加款：</p>

<div class="pg-flow">
<div class="pg-step">A 扣 80</div>
<div class="pg-arrow">→</div>
<div class="pg-step">B 加 80</div>
</div>

<p>如果沒有 transaction 保護，這兩步之間任何一個環節出狀況，都可能只完成一半。</p>

<div class="pg-steps">
<div class="pg-stepcard">
<div class="pg-stepnum">1</div>
<div class="pg-step-title">A 扣款成功</div>
<p><code>A</code> 的餘額從 100 變成 20，這一步已經寫入資料庫。</p>
</div>
<div class="pg-stepcard cond">
<div class="pg-stepnum">2</div>
<div class="pg-step-title">伺服器在這裡當機</div>
<p><code>B</code> 加款的那一步還沒執行，程式就中斷了。</p>
</div>
</div>

<div class="pg-callout danger">
<span class="pg-icon">💥</span>
<p><strong>結果：</strong><code>A</code> 變成 20，<code>B</code> 還是 100，中間的 80 元從系統裡憑空消失，帳目對不起來。</p>
</div>

<p>Transaction 的 Atomicity 能保證的是，把這兩個操作包進同一個 <code>BEGIN</code> 到 <code>COMMIT</code> 之間之後，最終只會有兩種結果，不會停在半途。</p>

<table>
<thead><tr><th>情境</th><th>A 餘額</th><th>B 餘額</th><th>是否允許</th></tr></thead>
<tbody>
<tr><td>全部成功</td><td>20</td><td>180</td><td><span class="pg-tag ok">允許</span></td></tr>
<tr><td>全部失敗（rollback）</td><td>100</td><td>100</td><td><span class="pg-tag ok">允許</span></td></tr>
<tr><td>只扣款、沒加款</td><td>20</td><td>100</td><td><span class="pg-tag danger">不允許</span></td></tr>
</tbody>
</table>

<p>所以 Atomicity 解決的問題，是一個 transaction 做到一半失敗這種情境，可以理解成整組操作只有全部成功或全部失敗兩種結局，這就是 <code>ACID</code> 裡的 <code>A</code>。不過這件事完全沒有觸及另一個同時執行的 transaction 會不會看到中間狀態、會不會跟這筆轉帳互相干擾，那是 Isolation 要處理的範圍。</p>

</section>
</div>
</div>

<!-- endtab -->

<!-- tab Isolation-->

<div class="pg">
<div class="pg-container">

<section>
<div class="pg-section-head teal">
<div class="pg-chip">03</div>
<h2>Isolation</h2>
</div>

<p>多個 transaction 同時執行的時候，彼此可以看到什麼，會不會互相踩到對方還沒確定的資料。回到最開始的情境，兩筆轉帳同時進行時，實際發生的時序可能像這樣：</p>

```mermaid
sequenceDiagram
    participant T1 as Transaction 1（A 轉給 B）
    participant DB as Database（帳戶 A）
    participant T2 as Transaction 2（A 轉給 C）
    T1->>DB: BEGIN
    T2->>DB: BEGIN
    T1->>DB: SELECT A.balance
    DB-->>T1: 回傳 100
    T2->>DB: SELECT A.balance
    DB-->>T2: 回傳 100
    Note over T1: 判斷 100 >= 80，成立
    Note over T2: 判斷 100 >= 80，成立
    T1->>DB: A 扣 80，B 加 80
    T2->>DB: A 扣 80，C 加 80
    T1->>DB: COMMIT
    T2->>DB: COMMIT
```

<p>兩邊都讀到 <code>A = 100</code>，都判斷餘額足夠，於是都各自完成了自己的轉帳。最後的結果是：</p>

<table>
<thead><tr><th>帳戶</th><th>最終餘額</th></tr></thead>
<tbody>
<tr><td>A</td><td>20</td></tr>
<tr><td>B</td><td>180</td></tr>
<tr><td>C</td><td>180</td></tr>
</tbody>
</table>

<div class="pg-callout danger">
<span class="pg-icon">🚨</span>
<p>系統實際上等於發出去了 160 元（B 收到 80，C 收到 80），但 <code>A</code> 一開始只有 100 元。這種兩個 transaction 各自檢查條件時都成立，合併起來卻超出實際限制的狀況，就是 race condition。</p>
</div>

<p>而最容易被忽略的地方在於，<code>BEGIN</code> 與 <code>COMMIT</code> 本身並不會自動保證這種事不會發生。以 PostgreSQL 為例，預設的 isolation level 是 <code>READ COMMITTED</code>。這個等級能保證的仍然是 Atomicity 那部分，但如果應用程式的邏輯是先 <code>SELECT</code>、在應用程式裡判斷、之後才 <code>UPDATE</code>，這種讀取與判斷之間的空窗期，並不會被 <code>READ COMMITTED</code> 自動堵住，仍然需要額外處理 concurrency，例如接下來提到的三種方法。</p>

</section>
</div>
</div>

<!-- endtab -->

<!-- tab 方法一：鎖住 A-->

<div class="pg">
<div class="pg-container">

<section>
<div class="pg-section-head">
<div class="pg-chip">04</div>
<h2>方法一：用 FOR UPDATE 鎖住那一筆資料</h2>
</div>

<p>以 PostgreSQL 為例，可以在 <code>SELECT</code> 的時候加上 <code>FOR UPDATE</code>：</p>

<pre class="pg-pre" data-lang="sql"><span class="kw">BEGIN</span>;
<span class="kw">SELECT</span> balance
<span class="kw">FROM</span> <span class="type">accounts</span>
<span class="kw">WHERE</span> id <span class="op">=</span> <span class="str">'A'</span>
<span class="kw">FOR UPDATE</span>;</pre>

<p>這裡的 <code>FOR UPDATE</code> 是關鍵，效果可以理解成，先執行的 transaction 等於在跟資料庫說，正在處理帳戶 <code>A</code>，在這筆處理完成之前，其他 transaction 不准修改這一筆資料。實際的執行時序如下：</p>

```mermaid
sequenceDiagram
    participant T1 as Transaction 1
    participant DB as Database（帳戶 A 的 row lock）
    participant T2 as Transaction 2
    T1->>DB: SELECT A FOR UPDATE
    DB-->>T1: 取得 row lock，A = 100
    T2->>DB: SELECT A FOR UPDATE
    Note over T2: 想取得同一把鎖，被阻塞（BLOCKED）
    Note over T1: 判斷 100 >= 80，成立
    T1->>DB: A 扣 80，B 加 80，COMMIT
    DB-->>T2: 釋放鎖，T2 才能繼續
    T2->>DB: SELECT A，讀到最新的 A = 20
    Note over T2: 判斷 20 >= 80，不成立，拒絕轉帳
```

<p>先執行的 <code>T1</code> 完成扣款並提交之後，鎖才釋放，接手的 <code>T2</code> 這時讀到的已經是最新的餘額 20，判斷 20 是否大於等於 80 不成立，於是拒絕這筆轉帳。最終結果會是正確的：</p>

<div class="pg-evidence">
<div class="pg-evidence-head">
<span class="pg-tag">結果</span>
<strong>兩筆轉帳依序處理，沒有超額</strong>
</div>
<p><code>A</code> 剩下 20，<code>B</code> 收到轉帳變成 180，<code>C</code> 因為餘額不足被擋下，維持 100。</p>
</div>

<p>這個做法的核心概念，是先讀資料、再由應用程式根據資料做判斷，所以在 <code>SELECT</code> 的當下就把這筆資料鎖住，屬於悲觀式的做法，先阻止其他 transaction 有機會碰到這筆還在處理中的資料。</p>

</section>
</div>
</div>

<!-- endtab -->

<!-- tab 方法二：Conditional UPDATE-->

<div class="pg">
<div class="pg-container">

<section>
<div class="pg-section-head teal">
<div class="pg-chip">05</div>
<h2>方法二：把檢查條件寫進 UPDATE 本身</h2>
</div>

<p>另一種做法是不要先 <code>SELECT</code>、判斷、再 <code>UPDATE</code>，而是讓 <code>UPDATE</code> 本身就帶有條件：</p>

<pre class="pg-pre" data-lang="sql"><span class="kw">UPDATE</span> <span class="type">accounts</span>
<span class="kw">SET</span> balance <span class="op">=</span> balance <span class="op">-</span> <span class="num">80</span>
<span class="kw">WHERE</span> id <span class="op">=</span> <span class="str">'A'</span>
  <span class="kw">AND</span> balance <span class="op">>=</span> <span class="num">80</span>;</pre>

<p>接著只要檢查這行語句實際影響了幾筆資料，也就是 <code>affected rows</code>，如果是 1 才代表扣款真的成功。</p>

<div class="pg-flow">
<div class="pg-step">T1 執行 UPDATE</div>
<div class="pg-arrow">→</div>
<div class="pg-step">A 扣款成功，變成 20</div>
<div class="pg-arrow">→</div>
<div class="pg-step cond">T2 執行 UPDATE</div>
<div class="pg-arrow">→</div>
<div class="pg-step">balance >= 80 不成立</div>
<div class="pg-arrow">→</div>
<div class="pg-step">affected rows = 0，轉帳失敗</div>
</div>

<p>假設 <code>T1</code> 與 <code>T2</code> 幾乎同時對 <code>A = 100</code> 發出這個條件式 <code>UPDATE</code>，資料庫層級會保證這兩個 <code>UPDATE</code> 不會同時套用在同一筆資料上。先完成的那一個會把 <code>A</code> 改成 20，後面那一個再檢查 <code>balance >= 80</code> 時，因為 20 已經不滿足這個條件，這次 <code>UPDATE</code> 影響的資料筆數就是 0，代表這筆轉帳沒有真的發生。</p>

<p>這個做法之所以重要，是因為原本讀 <code>balance</code>、應用程式判斷、修改 <code>balance</code> 三個分開的步驟，被壓縮成資料庫裡的一個 atomic operation，也就是把檢查與寫入合併成同一個原子動作，不再留下讓兩個 transaction 都通過檢查的空窗期。</p>

</section>
</div>
</div>

<!-- endtab -->

<!-- tab 方法三：Serializable-->

<div class="pg">
<div class="pg-container">

<section>
<div class="pg-section-head violet">
<div class="pg-chip">06</div>
<h2>方法三：使用 Serializable isolation</h2>
</div>

<p><code>SERIALIZABLE</code> 這個 isolation level 想表達的概念是，這些 transaction 就算實際上是同時執行的，最終呈現出來的結果，必須等價於把它們一個接一個依序執行過一次。</p>

<div class="pg-summary-grid">
<div class="pg-box violet">
<h4>實際發生的狀況</h4>
<p><code>T1</code> 與 <code>T2</code> 在時間軸上是真正並行、互相重疊執行的。</p>
</div>
<div class="pg-box pg">
<h4>資料庫保證看到的結果</h4>
<p>結果必須等同於 <code>T1</code> 先跑完、<code>T2</code> 才開始，或是反過來 <code>T2</code> 先跑完、<code>T1</code> 才開始，兩種順序中的其中一種。</p>
</div>
<div class="pg-box teal">
<h4>做不到的時候</h4>
<p>資料庫判斷兩者無法安全地排出一個合法順序，就會讓其中一個 transaction 發生 serialization conflict 而被 <code>ROLLBACK</code>。</p>
</div>
</div>

<p>當發生 serialization conflict 而被回滾時，應用程式必須自行 retry 這個 transaction，這個重試機制是使用 <code>SERIALIZABLE</code> 時很重要的一部分。所以 <code>SERIALIZABLE</code> 並不是資料庫用某種魔法讓所有 transaction 永遠都成功，而是如果沒辦法得到一個合法的序列化執行結果，資料庫寧願讓其中一個 transaction 失敗，交給應用程式重試。這種做法允許多個 transaction 同時執行，但資料庫保證最終結果等價於某一種依序執行的排列方式，至於底層具體怎麼實作這件事，屬於各家資料庫自己的實作細節，可能用鎖、MVCC、predicate 或 range locking、conflict detection 加上 abort 及 retry，甚至混合使用多種手段。</p>

<p>PostgreSQL 的 <code>SERIALIZABLE</code> 是一個很好的例子，它採用的機制叫做 Serializable Snapshot Isolation，簡稱 <code>SSI</code>。它並不是單純看到某個 transaction 讀了 <code>A</code> 就把 <code>A</code> 整個鎖死、不准其他 transaction 再讀取，而是允許兩邊都正常讀取與繼續執行，改由資料庫在背後持續追蹤彼此的依賴關係：</p>

```mermaid
sequenceDiagram
    participant T1 as Transaction 1
    participant DB as Database（SSI 依賴追蹤）
    participant T2 as Transaction 2
    T1->>DB: 讀 A = 100
    T2->>DB: 讀 A = 100
    Note over T1: 繼續執行
    Note over T2: 繼續執行
    DB->>DB: 偵測兩者之間的 dependency
    Note over DB: 發現可能形成不合法的 serialization 結果
    DB-->>T1: 其中一個 transaction 被 ABORT
    Note over T1: 應用程式收到失敗，重新 retry 這個 transaction
```

<p>所以方法三的核心精神，比較接近不一定要一開始就阻止兩個 transaction 同時執行，但保證最後不會產生一個不可能由任何一種依序執行方式得到的結果，跟方法一那種一開始就悲觀鎖住資料的做法，在設計哲學上是不同方向。</p>

</section>
</div>
</div>

<!-- endtab -->

<!-- tab RDBMS-->

<div class="pg">
<div class="pg-container">

<section>
<div class="pg-section-head">
<div class="pg-chip">07</div>
<h2>RDBMS</h2>
</div>

<p>如果把問題範圍限定在多筆資料之間存在複雜關係，而且在 concurrency 之下仍然必須維持 business invariant 這一類情境，關聯式資料庫通常確實有比較成熟、完整、自然的處理方式。這不是說 NoSQL 完全做不到，而是關聯式資料庫從設計核心開始，就把 transaction、concurrency control、constraints、關聯性當成第一級公民在對待。以 PostgreSQL、MySQL 的 InnoDB 引擎、SQL Server 為例，通常可以直接取得一整套現成工具。</p>

<table>
<thead><tr><th>能力分類</th><th>包含項目</th></tr></thead>
<tbody>
<tr><td>Transaction 控制</td><td><code>BEGIN</code> / <code>COMMIT</code> / <code>ROLLBACK</code>，以及 Atomicity 保證</td></tr>
<tr><td>Isolation Level</td><td><code>READ COMMITTED</code>、<code>REPEATABLE READ</code>、<code>SERIALIZABLE</code></td></tr>
<tr><td>Concurrency Control</td><td>MVCC、Row Lock、<code>SELECT ... FOR UPDATE</code></td></tr>
<tr><td>Constraints</td><td><code>PRIMARY KEY</code>、<code>UNIQUE</code>、<code>FOREIGN KEY</code>、<code>CHECK</code>、<code>NOT NULL</code></td></tr>
</tbody>
</table>

<p>這些工具可以互相組合，用來維護資料上的種種前提條件。舉例來說，一套銀行系統裡通常會有帳戶、交易紀錄、客戶、卡片、貸款這些彼此之間存在大量關聯的資料，而且系統通常要同時滿足下面這些前提。</p>

<ul class="pg-checklist">
<li>帳戶不能被提領超過實際餘額</li>
<li>一筆交易不能只完成一半就停在中間狀態</li>
<li>帳戶不存在的情況下，不能產生對應的交易紀錄</li>
<li>同一個 <code>payment_id</code> 不能被扣款兩次</li>
<li>刪除客戶資料時，不能留下沒有對應客戶的孤兒帳戶</li>
<li>多人同時操作同一筆資料時，結果仍然必須正確</li>
</ul>


</section>
</div>
</div>

<!-- endtab -->

<!-- tab DynamoDB 的差異-->

<div class="pg">
<div class="pg-container">

<section>
<div class="pg-section-head violet">
<div class="pg-chip">08</div>
<h2>換成 DynamoDB，思考方式差異更明顯</h2>
</div>

<p>DynamoDB 目前也提供 ACID transaction，可以透過 <code>TransactWriteItems</code> 把多個操作組成一個 all-or-nothing 的 transaction。所以「NoSQL 沒有 transaction」這個說法，已經不成立。不過 DynamoDB 有一個非常關鍵的設計概念，就是 <code>Partition Key</code>，整個資料模型、擴充能力與效能表現，都高度圍繞著 partition 在運作。</p>

<p>因此在設計 DynamoDB 的資料結構時，思考的起點通常跟關聯式資料庫不太一樣：</p>

<div class="pg-summary-grid">
<div class="pg-box violet">
<h4>DynamoDB 的思考起點</h4>
<p>存取模式（access pattern）長什麼樣子。<code>Partition Key</code> 該怎麼設計。資料要不要 denormalize、哪些資料應該放在一起、怎麼避免 hot partition。</p>
</div>
<div class="pg-box pg">
<h4>關聯式資料庫的思考起點</h4>
<p>Entity 之間的關聯是什麼、要 normalize 成哪些資料表、foreign key 怎麼建立、查詢時要怎麼 <code>JOIN</code>。</p>
</div>
</div>

<div class="pg-callout info">
<span class="pg-icon">💡</span>
<p>兩者並不是誰優誰劣的關係，而是在優化不同的問題。用 DynamoDB 硬做複雜關聯查詢，等於是在使用它比較昂貴、比較複雜的能力；相反地，這正是 PostgreSQL 這類關聯式資料庫本來就擅長的 workload。</p>
</div>

</section>
</div>
</div>

<!-- endtab -->

<!-- tab 總覽-->

<div class="pg">
<div class="pg-container">

<section>
<div class="pg-section-head">
<div class="pg-chip">09</div>
<h2>整體回顧</h2>
</div>

<p>轉帳這類先讀取、再判斷、最後修改的邏輯，光靠 <code>BEGIN</code> 與 <code>COMMIT</code> 包起來並不足夠，因為 <code>ACID</code> 裡的 Atomicity 只保證單一 transaction 不會做到一半失敗，真正決定多個 transaction 同時執行會不會互相干擾的，是 Isolation。要解決 Isolation 帶來的 race condition，常見有三種方向，各自的取捨如下。</p>

<div class="pg-summary-grid">
<div class="pg-box pg">
<h4>方法一：FOR UPDATE</h4>
<p>在 <code>SELECT</code> 當下就悲觀鎖住那一筆資料，其他 transaction 必須等待，實作直覺，但等待中的 transaction 會被阻塞。</p>
</div>
<div class="pg-box teal">
<h4>方法二：Conditional UPDATE</h4>
<p>把檢查條件寫進 <code>UPDATE</code> 語句本身，靠 <code>affected rows</code> 判斷是否成功，把檢查與寫入合併成一個原子操作，不需要額外鎖。</p>
</div>
<div class="pg-box violet">
<h4>方法三：Serializable</h4>
<p>允許 transaction 同時執行，資料庫在背後偵測衝突，一旦發現無法排出合法順序就讓其中一方失敗，交由應用程式 retry。</p>
</div>
</div>

<p>再往上一層看，這類多筆資料互有關聯，又要求 concurrency 下仍然正確的問題，關聯式資料庫從設計核心開始就把 transaction、isolation level、row lock、各種 constraint 當成第一級公民，通常能提供比較完整的現成工具。DynamoDB 這類 NoSQL 資料庫雖然也已經有 ACID transaction，但設計思維更圍繞在 <code>Partition Key</code> 與存取模式上，跟關聯式資料庫的正規化、外鍵、<code>JOIN</code> 思維是不同的優化方向，兩者是在解決不同性質的問題，而不是互相取代的關係。</p>

<div class="pg-trunk-divider"><span>🐘</span></div>

</section>

<footer>整理自 Concurrency-safe 系列筆記</footer>

</div>
</div>

<!-- endtab -->

{% endtabs %}
