---
title: MySQL-2
date: 2026-08-14 08:08:00
categories: DB
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/MySQL/mysql-trap3-landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/MySQL/mysql-trap3-landing.png
toc:
toc_number:
comments :

---

{% tabs MySQL%}


<!-- tab 外鍵限制-->

<style>
.msq{
  display:block;
  font-family:'Segoe UI',-apple-system,BlinkMacSystemFont,'PingFang TC','Microsoft JhengHei',sans-serif;
  background:
    radial-gradient(700px 400px at 92% -8%, rgba(228,142,0,.14), transparent 60%),
    radial-gradient(900px 500px at -8% 108%, rgba(0,97,138,.12), transparent 55%),
    #f4f9fc;
  color:#12303e!important;
  line-height:1.75;
  border-radius:20px;
  padding:1px 0 28px;
  margin:0 0 20px;
  overflow:hidden;
  --ink:#12303e; --sub:#547485; --card:#ffffff; --line:#dce7ee;
  --navy:#00618a; --navy2:#003b52; --orange:#e48e00; --orange2:#f7b955; --teal:#0d9488; --violet:#7c6cf7; --danger:#e5484d;
  --shadow:0 10px 30px -12px rgba(0,59,82,.18);
}
.msq *{box-sizing:border-box;}
.msq a{color:var(--orange)!important;text-decoration:none;}
.msq a:hover{text-decoration:underline;}
.msq .msq-hero{padding:48px 6vw 34px;text-align:center;position:relative;overflow:hidden;background:linear-gradient(160deg,var(--navy2),var(--navy));}
.msq .msq-hero::before{content:"🐬 · · · 🐬 · · · 🐬 · · · 🐬";position:absolute;top:14px;left:0;right:0;text-align:center;font-size:.9rem;opacity:.25;letter-spacing:.3em;color:#fff!important;pointer-events:none;}
.msq .msq-eyebrow{display:inline-block;font-size:.78rem;letter-spacing:.16em;color:#fff!important;text-transform:uppercase;font-weight:700;margin-bottom:12px;background:rgba(228,142,0,.35);padding:6px 14px;border-radius:999px;}
.msq .msq-hero h1{font-size:1.7rem;margin:0 0 12px;color:#fff!important;font-weight:800;}
.msq .msq-hero p{color:#cfe6f0!important;max-width:700px;margin:0 auto;font-size:1rem;}
.msq .msq-badges{display:flex;gap:10px;justify-content:center;flex-wrap:wrap;margin-top:20px;}
.msq .msq-badge{background:rgba(255,255,255,.12);border:1px solid rgba(255,255,255,.25);padding:6px 14px;border-radius:999px;font-size:.8rem;color:#eaf4f9!important;}
.msq .msq-container{padding:0 5vw;}
.msq section{margin-bottom:44px;}
.msq .msq-section-head{display:flex;align-items:center;gap:12px;margin-bottom:18px;}
.msq .msq-section-head .msq-chip{flex:none;width:38px;height:38px;border-radius:11px;display:flex;align-items:center;justify-content:center;font-weight:800;background:linear-gradient(135deg,var(--navy2),var(--navy));color:#fff!important;font-size:1rem;box-shadow:var(--shadow);}
.msq .msq-section-head.accent .msq-chip{background:linear-gradient(135deg,#c97400,var(--orange2));}
.msq .msq-section-head.teal .msq-chip{background:linear-gradient(135deg,#0d7a70,#5fd4c4);}
.msq .msq-section-head.violet .msq-chip{background:linear-gradient(135deg,#7c6cf7,#a78bfa);}
.msq .msq-section-head h2{margin:0;font-size:1.3rem;font-weight:800;color:var(--ink)!important;}
.msq h3{font-size:1.08rem;margin:26px 0 10px;color:var(--ink)!important;border-left:4px solid var(--navy);padding-left:10px;}
.msq p{color:var(--sub)!important;}
.msq .msq-card{background:var(--card);border:1px solid var(--line);border-radius:18px;box-shadow:var(--shadow);padding:18px 22px;margin:14px 0;}
.msq .msq-card img{max-width:100%;border-radius:12px;display:block;margin:10px auto 0;}
.msq table{width:auto;max-width:100%;border-collapse:separate;border-spacing:0;background:var(--card)!important;border:1px solid var(--line);border-radius:16px;overflow:hidden;margin:14px 0 22px;font-size:.88rem;box-shadow:var(--shadow);}
.msq thead th{background:#eaf3f8!important;color:var(--ink)!important;text-align:left;padding:13px 16px;font-weight:800;border-bottom:1px solid var(--line);}
.msq tbody td{padding:13px 16px;border-bottom:1px solid var(--line);color:#2b4753!important;vertical-align:top;background:var(--card)!important;}
.msq tbody tr:last-child td{border-bottom:none;}
.msq tbody tr:hover td{background:#f3faff!important;}
.msq tbody td strong{color:var(--ink)!important;}
.msq tbody td code{color:#b4630a!important;}
.msq code{font-family:Consolas,'SFMono-Regular','Liberation Mono',Menlo,monospace;background:#e9f2f7!important;color:#b4630a!important;padding:2px 7px;border-radius:6px;font-size:.88em;border:1px solid #d7e6ee;}
.msq pre.msq-pre{background:#0d2233!important;border:none;border-radius:16px;padding:20px 22px;overflow-x:auto;margin:16px 0;position:relative;color:#e6f2f8!important;font-size:.85rem!important;line-height:1.7!important;font-family:Consolas,'SFMono-Regular','Liberation Mono',Menlo,monospace!important;white-space:pre!important;box-shadow:var(--shadow);}
.msq pre.msq-pre::before{content:attr(data-lang);position:absolute;top:12px;right:16px;font-size:.65rem;letter-spacing:.08em;text-transform:uppercase;color:#7fa3b8!important;font-family:'Segoe UI',sans-serif;}
.msq .kw{color:#7fd0ff!important;} .msq .str{color:#c3e88d!important;} .msq .cm{color:#5f89a0!important;font-style:italic;} .msq .fn{color:#ffb454!important;} .msq .type{color:#f7b955!important;} .msq .num{color:#c3e88d!important;} .msq .op{color:#ff6b9d!important;}
.msq .msq-callout{border-radius:16px;padding:18px 22px;margin:18px 0;border:1px solid transparent;display:flex;gap:14px;}
.msq .msq-callout .msq-icon{font-size:1.3rem;flex:none;}
.msq .msq-callout.info{background:#eff6ff;border-color:#bfdbfe;}
.msq .msq-callout.warn{background:#fffbeb;border-color:#fde68a;}
.msq .msq-callout.danger{background:#fef2f2;border-color:#fecaca;}
.msq .msq-callout.ok{background:#ecfdf5;border-color:#a7f3d0;}
.msq .msq-callout p{margin:0;font-size:.92rem;color:#2b4753!important;}
.msq .msq-callout strong{color:var(--ink)!important;}
.msq .msq-tag{display:inline-block;font-size:.72rem;font-weight:800;padding:3px 11px;border-radius:999px;letter-spacing:.02em;}
.msq .msq-tag.danger{background:#fee2e2!important;color:#b91c1c!important;}
.msq .msq-tag.ok{background:#dcfce7!important;color:#15803d!important;}
.msq .msq-tag.warn{background:#fef3c7!important;color:#b45309!important;}
.msq .msq-tag.info{background:#e0f2fe!important;color:#0369a1!important;}
.msq .msq-tag.navy{background:#dceaf1!important;color:#00435c!important;}
.msq .msq-tag.orange{background:#fdecd2!important;color:#9a5a00!important;}
.msq .msq-summary-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:16px;margin:18px 0;}
.msq .msq-summary-grid .msq-box{background:var(--card);border:1px solid var(--line);border-radius:16px;padding:18px 20px;box-shadow:var(--shadow);border-top:4px solid var(--navy);position:relative;overflow:hidden;}
.msq .msq-summary-grid .msq-box.navy{border-top-color:var(--navy);}
.msq .msq-summary-grid .msq-box.orange{border-top-color:var(--orange);}
.msq .msq-summary-grid .msq-box.teal{border-top-color:var(--teal);}
.msq .msq-summary-grid .msq-box h4{margin:0 0 8px;color:var(--ink)!important;font-size:.85rem;text-transform:uppercase;letter-spacing:.04em;}
.msq .msq-summary-grid .msq-box p{margin:0;font-size:.87rem;color:var(--sub)!important;}
@media (max-width:780px){.msq .msq-summary-grid{grid-template-columns:1fr;}}
.msq .msq-flow{display:flex;align-items:center;gap:10px;flex-wrap:wrap;margin:20px 0;}
.msq .msq-flow .msq-step{background:var(--card);border:1px solid var(--line);border-radius:12px;padding:10px 16px;font-size:.85rem;font-weight:700;color:var(--ink)!important;box-shadow:var(--shadow);}
.msq .msq-flow .msq-step.cond{background:#fffbeb;border-color:#fde68a;color:#b45309!important;}
.msq .msq-flow .msq-arrow{color:var(--orange)!important;font-size:1.2rem;font-weight:900;}
.msq .msq-steps{display:flex;flex-direction:column;gap:14px;margin:16px 0 22px;}
.msq .msq-stepcard{background:var(--card);border:1px solid var(--line);border-radius:14px;padding:16px 20px;position:relative;padding-left:60px;box-shadow:var(--shadow);}
.msq .msq-stepcard .msq-stepnum{position:absolute;left:16px;top:16px;width:30px;height:30px;border-radius:50%;background:linear-gradient(135deg,var(--navy2),var(--navy));color:#fff!important;font-weight:800;font-size:.9rem;display:flex;align-items:center;justify-content:center;}
.msq .msq-stepcard.cond .msq-stepnum{background:linear-gradient(135deg,#c97400,var(--orange2));}
.msq .msq-stepcard .msq-step-title{font-weight:700;color:var(--ink)!important;font-size:1rem;margin-bottom:4px;display:flex;align-items:center;gap:8px;flex-wrap:wrap;}
.msq .msq-stepcard p{margin:6px 0;font-size:.9rem;color:var(--sub)!important;}
.msq .msq-checklist{list-style:none;padding:0;margin:16px 0;}
.msq .msq-checklist li{display:flex;gap:12px;align-items:flex-start;padding:10px 0;border-bottom:1px dashed var(--line);color:var(--sub)!important;}
.msq .msq-checklist li:last-child{border-bottom:none;}
.msq .msq-checklist li::before{content:"✓";flex:none;width:22px;height:22px;border-radius:7px;background:#dceaf1;color:#00618a!important;font-size:.75rem;display:flex;align-items:center;justify-content:center;margin-top:2px;font-weight:900;}
.msq .msq-evidence{border:1px solid var(--line);background:var(--card);border-radius:18px;box-shadow:var(--shadow);padding:18px 20px;margin:16px 0 22px;}
.msq .msq-evidence .msq-evidence-head{display:flex;align-items:center;gap:8px;margin-bottom:10px;}
.msq .msq-evidence .msq-evidence-head .msq-tag{background:var(--navy2)!important;color:#fff!important;}
.msq .msq-dot-divider{display:flex;align-items:center;gap:10px;justify-content:center;margin:36px 0;color:var(--navy)!important;}
.msq .msq-dot-divider::before,.msq .msq-dot-divider::after{content:"";flex:1;height:1px;background:linear-gradient(90deg,transparent,var(--line) 40%,var(--line) 60%,transparent);}
.msq .msq-dot-divider span{font-size:1.1rem;}
.msq footer{text-align:center;padding:30px 5vw 6px;color:var(--sub)!important;font-size:.82rem;border-top:1px solid var(--line);margin-top:8px;}
</style>

<div class="msq">
<div class="msq-hero">
<span class="msq-eyebrow">MySQL 底層機制筆記</span>
<h1>🐬 外鍵、交易與隔離等級的那些意外行為</h1>
<p>資料庫平常安安靜靜地把資料存好，但只要牽涉到外鍵、交易或並行讀寫，MySQL 就會出現一些跟直覺不一樣的行為，甚至讓其他主流資料庫的工程師都嚇一跳。這篇整理四個真實會踩到的情境，從外鍵鎖定的意外連坐、交易遇到 DDL 直接失效，到兩個隔離等級案例揭露的資料錯亂風險。</p>
<div class="msq-badges">
<span class="msq-badge">外鍵限制</span>
<span class="msq-badge">Transaction 與 DDL</span>
<span class="msq-badge">隔離等級案例</span>
<span class="msq-badge">跨資料庫比較</span>
</div>
</div>
<div class="msq-container">

<section>
<div class="msq-section-head">
<div class="msq-chip">01</div>
<h2>外鍵的本意：維持兩張表的資料一致性</h2>
</div>
<p>外鍵（foreign key）的用意是讓使用系統的人知道兩張表是相關的，更重要的是讓資料庫本身也「意識到」這層關係。當其中一張表的資料被改動時，資料庫要負責維持兩者之間該有的一致性，而不是各改各的。</p>

<div class="msq-card">
<p>下圖是一個常見的例子，買家表與訂單表，訂單表裡的買家欄位是指向買家表的外鍵。</p>
<img src="https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/MySQL/fk.png" alt="外鍵關聯示意圖">
</div>

<p>外鍵設定了 <code>ON UPDATE CASCADE</code>，代表系統要保證所謂的參照完整性（referential integrity）。也就是說，一旦外鍵被更改，資料的完整性就必須被守住。實務上的效果是，如果訂單表裡正在修改買家欄位，買家表對應的那筆資料就會被鎖住，不能同時被修改。</p>
</section>

<section>
<div class="msq-section-head accent">
<div class="msq-chip">02</div>
<h2>意外的連坐：改訂單狀態也會鎖到買家表</h2>
</div>

<div class="msq-card">
<p>如果修改的其實是訂單的狀態欄位，理論上跟買家表完全無關，但實際測試會發現買家表依然被鎖住。</p>
<img src="https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/MySQL/fk-protection.png" alt="修改訂單狀態時買家表被鎖定示意圖">
</div>

<div class="msq-callout warn">
<span class="msq-icon">⚠️</span>
<p><strong>問題出在 INDEX：</strong>訂單表裡有一個「買家欄位加訂單狀態」的複合索引，而 MySQL 把訂單狀態也一併當作外鍵的一部分來管制，導致修改訂單狀態時，買家表也被牽連鎖住，這個行為相當令人意外。</p>
</div>

<div class="msq-card">
<img src="https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/MySQL/fk-index.png" alt="複合索引導致外鍵鎖定範圍擴大示意圖">
</div>

<div class="msq-evidence">
<div class="msq-evidence-head">
<span class="msq-tag">官方回報</span>
<strong>Bug #94148</strong>
</div>
<p>已經有人把這個現象提交成官方 Bug 回報：<a href="https://bugs.mysql.com/bug.php?id=94148" target="_blank">Bug #94148</a>。有網友找到的解法是，額外加一個「純買家欄位」的索引，結果就意外地不再被鎖住。索引與外鍵原本是兩個不相干的概念，卻在 MySQL 的實作裡被混在一起，變成解法也帶著一點意外的味道。</p>
</div>
</section>

<div class="msq-dot-divider"><span>🐬</span></div>

</div>
</div>

<!-- endtab -->


<!-- tab Transaction 與 DDL-->

<div class="msq">
<div class="msq-container">

<section>
<div class="msq-section-head teal">
<div class="msq-chip">03</div>
<h2>當交易遇上 DDL：一個會悄悄失效的 Rollback</h2>
</div>

<p>SQL 語法大致分成兩種類型，一種是定義資料結構的 DDL，例如 <code>CREATE TABLE</code>、<code>DROP TABLE</code>；另一種是大家熟悉的 DML，例如 <code>SELECT</code>、<code>UPDATE</code>。在 MySQL 裡，交易（transaction）機制對這兩種語法的保護程度並不一樣，以下用一個常見的資料搬遷情境來說明。</p>

<div class="msq-card">
<img src="https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/DB/MySQL/transaction-1-not-DDL.png" alt="交易中包含 DDL 導致回滾失效示意圖">
</div>

<div class="msq-steps">
<div class="msq-stepcard">
<div class="msq-stepnum">1</div>
<div class="msq-step-title">將資料從 <code>old_data</code> 備份到 <code>new_data</code></div>
<p>這一步是單純的資料搬移，屬於 DML 操作。</p>
</div>
<div class="msq-stepcard cond">
<div class="msq-stepnum">2</div>
<div class="msq-step-title">刪除 <code>old_data</code>（這一步是 DDL）</div>
<p>因為是結構層級的操作，凌駕於交易之上，不受交易的 rollback 保護。</p>
</div>
<div class="msq-stepcard">
<div class="msq-stepnum">3</div>
<div class="msq-step-title">將此操作紀錄在 audit table</div>
<p>如果這一步不小心失敗，理論上整包交易都應該回滾。</p>
</div>
</div>

<div class="msq-callout danger">
<span class="msq-icon">🚨</span>
<p><strong>實際結果：</strong>如果 audit table 那一步寫入失敗，整個交易理應回滾到最初的狀態，但因為第二步刪除 <code>old_data</code> 是 DDL，它凌駕於交易之上，不會被回滾撤銷。最後的結果是舊資料真的不見了，但整個搬遷操作卻沒有真正完整執行到，資料等於是憑空消失的。</p>
</div>
</section>

</div>
</div>

<!-- endtab -->



<!-- tab 隔離等級：意外復活的使用者-->

<div class="msq">
<div class="msq-container">

<section>
<div class="msq-section-head violet">
<div class="msq-chip">04</div>
<h2>案例一：明明查詢是空表，修改卻成功了</h2>
</div>

<p>資料庫裡目前有一張空的使用者資料表，小左和小右兩位工程師幾乎同時開啟了各自的交易（transaction）。以下是實際發生的順序。</p>

<div class="msq-steps">
<div class="msq-stepcard">
<div class="msq-stepnum">1</div>
<div class="msq-step-title">小右寫入並提交</div>
<p>小右在自己的交易裡，往這張表寫入了 3 個新使用者，隨後執行 <code>COMMIT</code>。此時資料庫裡實際上已經存在這 3 筆資料。</p>
</div>
<div class="msq-stepcard cond">
<div class="msq-stepnum">2</div>
<div class="msq-step-title">小左查詢，看到 0 個使用者</div>
<p>小左在自己尚未提交的交易裡，執行 <code>SELECT</code> 查詢這張表，結果看到的使用者數量是 0 個。原因是 MySQL 預設的隔離等級是可重複讀（Repeatable Read），小左的交易一開始就建立了一份資料快照，這份快照停留在交易剛開始時的狀態，因此即使小右已經提交了新資料，小左依然只能看到空表。</p>
</div>
<div class="msq-stepcard cond">
<div class="msq-stepnum">3</div>
<div class="msq-step-title">小左閉眼修改，居然成功了</div>
<p>雖然查出來是空表，小左還是被交辦「幫第一個使用者改名字」的任務，於是直接執行 <code>UPDATE users SET name='新名字' WHERE id=1;</code>，結果系統提示修改成功。原因是純查詢用的是快照讀，讀的是歷史資料，但 <code>UPDATE</code>、<code>DELETE</code>、<code>INSERT</code> 這類寫入操作用的是當前讀，會直接讀取資料庫裡最新、最真實的資料。因為小右已經提交了資料，最新資料裡確實存在 <code>id=1</code> 的使用者，小左的修改指令自然就找到了這筆紀錄並完成改名。</p>
</div>
<div class="msq-stepcard">
<div class="msq-stepnum">4</div>
<div class="msq-step-title">小左再次查詢，看到 1 個使用者</div>
<p>小左不放心，再次查詢整張表，這次看到了 1 個使用者，正是剛剛被自己改過名字的那一筆。根據 MySQL 的多版本併發控制（MVCC）規則，一個交易永遠可以看見自己做出的修改，該筆資料被小左修改後，最新版本的交易編號就變成小左的交易編號，因此小左的快照讀在此時被允許看見這筆被修改的資料。至於另外兩筆使用者，因為從未被小左修改過，對小左的歷史快照而言依然被隔離在外，所以看不到。</p>
</div>
</div>

<div class="msq-callout info">
<span class="msq-icon">💡</span>
<p><strong>最終結局：</strong>直到下班，小左都以為這張表裡只有 1 個使用者，也就是自己改過名字的那一個，完全不知道實際上已經有 3 個使用者存在。這個現象的根源，是 MySQL 在可重複讀等級下，把快照讀與當前讀的機制混用，進而打破了原本應該嚴格的隔離性。</p>
</div>
</section>

</div>
</div>

<!-- endtab -->



<!-- tab 隔離等級：備份表變髒資料-->

<div class="msq">
<div class="msq-container">

<section>
<div class="msq-section-head accent">
<div class="msq-chip">05</div>
<h2>案例二：查出來是乾淨資料，寫進備份表卻變髒了</h2>
</div>

<p>這天剛好是愚人節，小左和小右同樣在差不多時間開啟了各自的交易。小右的任務是整活，要把所有使用者的名字都改成「哈基米」；小左的任務是備份，要把目前的使用者資料備份到另一張表，等愚人節活動結束後用來恢復原始資料。</p>

<div class="msq-steps">
<div class="msq-stepcard cond">
<div class="msq-stepnum">1</div>
<div class="msq-step-title">小右修改並提交</div>
<p>小右很快地在交易中把所有使用者名字都改成「哈基米」，執行 <code>COMMIT</code> 後就下班了。此時資料庫裡最新、最真實的使用者名稱已經全部變成「哈基米」。</p>
</div>
<div class="msq-stepcard">
<div class="msq-stepnum">2</div>
<div class="msq-step-title">小左查詢，看到原始資料</div>
<p>小左在動手備份前，先執行單純的查詢 <code>SELECT * FROM users;</code>，看到的依然是整活前的原始資料。原因跟案例一相同，小左的交易在小右提交前就已經開啟，MySQL 的可重複讀利用快照擋住了小右的修改。小左因此鬆了一口氣，心想查出來的是乾淨的原始資料，備份應該不會有問題。</p>
</div>
<div class="msq-stepcard cond">
<div class="msq-stepnum">3</div>
<div class="msq-step-title">小左執行「查詢並寫入」備份</div>
<p>小左寫下了開發中很常見的一行指令：<code>INSERT INTO backup_table SELECT * FROM users;</code>，隨後安心下班。</p>
</div>
<div class="msq-stepcard cond">
<div class="msq-stepnum">4</div>
<div class="msq-step-title">隔天上班，備份表全部變成「哈基米」</div>
<p>第二天愚人節活動結束，小左準備用備份表恢復資料，打開一看卻發現備份表裡存的全部都是「哈基米」，原始資料徹底消失，根本無法恢復。</p>
</div>
</div>

<div class="msq-callout danger">
<span class="msq-icon">🚨</span>
<p><strong>MySQL 的雙重標準：</strong>小左明明在備份前用 <code>SELECT</code> 親眼查出原始資料，為什麼用 <code>INSERT ... SELECT</code> 寫進去就變成了「哈基米」。原因在於 MySQL 一個很反直覺的底層設計。純查詢語句（如 <code>SELECT</code>）會乖乖遵守可重複讀規則，讀取歷史快照，所以小左看到了原始資料。但查詢加寫入的語句（如 <code>INSERT ... SELECT</code>）規則會突然切換，只要指令裡同時包含查詢與寫入，查詢的部分就會自動降級成讀已提交（Read Committed）規則執行。因為小左的備份指令是 <code>INSERT ... SELECT</code>，裡面的 <code>SELECT</code> 部分不再看歷史快照，而是直接執行當前讀，讀到了小右已經提交的最新資料，也就是「哈基米」，導致真正寫入備份表的全部是整活後的髒資料。</p>
</div>

<div class="msq-callout warn">
<span class="msq-icon">⚠️</span>
<p><strong>為什麼這很危險：</strong>其他主流資料庫如 PostgreSQL、Oracle、SQL Server，都不會因為多寫了一個 <code>INSERT</code> 就偷偷改變 <code>SELECT</code> 的讀取規則，機制的一致性是這些資料庫的基本要求。更麻煩的是，這種因為隔離性混用導致的資料錯亂永遠不會報錯，只會靜悄悄地在背後寫入與預期不符的資料，代表整個業務系統可能已經在用錯誤的資料下錯誤的結論，卻完全沒有察覺。整體來說，小左會被坑，是因為相信了 MySQL 聲稱的可重複讀隔離等級，卻不知道 MySQL 在處理 <code>INSERT ... SELECT</code> 這類語句時會悄悄切換規則。</p>
</div>
</section>

<div class="msq-dot-divider"><span>🐬</span></div>

<section>
<div class="msq-section-head">
<div class="msq-chip">06</div>
<h2>換成 PostgreSQL 或 MS SQL Server，結果會不一樣嗎</h2>
</div>

<p>在 PostgreSQL 和 MS SQL Server 這類主流資料庫上，機制的一致性是設計上的底線，不會像 MySQL 一樣在同一個隔離等級下，因為指令種類不同就偷偷切換底層的讀取規則。以下拆解如果換成這兩個資料庫，前面兩個案例會如何處理。</p>

<h3>先了解預設隔離等級的差異</h3>
<p>有一個根本性的差別，MySQL 預設使用可重複讀（Repeatable Read），而 PostgreSQL 和 MS SQL Server 預設使用的都是讀已提交（Read Committed）。如果用預設的讀已提交執行，小左在做任何操作前，只要小右已經提交，小左的 <code>SELECT</code> 就會直接看到最新提交的資料。也就是說，小左一開始就會看到那 3 個使用者（案例一）或「哈基米」（案例二），根本不會產生資訊落差，邏輯上非常直觀，不會出現 MySQL 那種先看到舊資料、後來又冒出新資料的落差。</p>

<h3>如果都手動切換到可重複讀，公平比較會怎樣</h3>
<p>假設把 PostgreSQL 和 MS SQL Server 也手動切換到可重複讀（快照隔離）等級，兩者的底層機制會表現得相當嚴謹，完美避開 MySQL 的陷阱。</p>

<div class="msq-summary-grid">
<div class="msq-box navy">
<h4>案例一：盲改空表</h4>
<p>PostgreSQL 嚴格遵守「只能修改快照裡看得見的資料」，執行 <code>UPDATE ... WHERE id=1</code> 時，因為快照裡是空表，不存在 <code>id=1</code>，這個 <code>UPDATE</code> 會直接回傳影響 0 行。如果交易試圖修改一筆在交易開始後被其他交易修改並提交的資料，PostgreSQL 甚至會直接拋出併發更新序列化錯誤，強制回滾。MS SQL Server 在傳統悲觀鎖等級下，小左查詢空表時就已經鎖定該範圍，小右的 <code>INSERT</code> 會被阻塞直到小左交易結束；若使用樂觀的快照隔離，行為則與 PostgreSQL 相同，<code>UPDATE</code> 會因為快照中找不到資料而影響 0 行，不會出現 MySQL 那種當前讀的意外行為。</p>
</div>
<div class="msq-box orange">
<h4>案例二：備份表變髒</h4>
<p>PostgreSQL 在可重複讀等級下，所有讀取操作都強制對齊同一份快照，不會因為多寫了 <code>INSERT</code> 就把裡面的 <code>SELECT</code> 偷偷降級成讀已提交。執行 <code>INSERT INTO backup_table SELECT * FROM users;</code> 時，裡面的 <code>SELECT</code> 依然讀取小左一開始建立的歷史快照，寫入備份表的會是百分之百的原始資料，備份成功。MS SQL Server 同樣地，在可重複讀或快照隔離下，<code>INSERT ... SELECT</code> 中的 <code>SELECT</code> 部分會嚴格受到交易歷史快照的保護，寫入備份表的同樣會是原始資料，不受小右後來提交的「哈基米」影響。</p>
</div>
<div class="msq-box teal">
<h4>機制一致性的差異</h4>
<p>MySQL 的機制不一致，純查詢用可重複讀走快照讀，混合寫入時 <code>SELECT</code> 卻悄悄降級成讀已提交走當前讀，導致資料在不知不覺中錯亂。PostgreSQL 與 MS SQL Server 的機制高度一致，是可重複讀就是可重複讀，所有讀取一律看快照，不允許「看著一份資料，卻寫入另一份資料」的情況發生。</p>
</div>
</div>

</section>

</div>
</div>

<!-- endtab -->



{% endtabs %}
