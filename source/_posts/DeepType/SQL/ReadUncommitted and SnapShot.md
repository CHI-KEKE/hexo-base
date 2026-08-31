---
title: ReadUncommitted and SnapShot
date: 2025-12-06 00:09:34
categories: 情報の存在論
top_img: https://i.imgur.com/NfCnwwU.png
cover : https://i.imgur.com/NfCnwwU.png
tags:
    - 情報の存在論
toc:
toc_number:
comments :
---

{% tabs ReadUncommitted and SnapShot %}

<!-- tab 板著一張臉-->

![man](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/IsolationLevel/1__the_man.p)



他的聲音低沉，又總是板著一張臉，走在路上連大人都會不自覺讓開，更別說孩子了。小孩一靠近他，他眉頭一皺，小朋友就嚇得跑掉。

久而久之，村裡流傳的故事越變越誇張，有人說他脾氣壞、有人說他討厭小孩、有人說他從不關心別人
這些話像一堆未經查證的資訊在村裡亂流，就像大家用著 ReadUncommitted 的方式在認識他，還沒看清就急著下結論。

一個人的真心，不是聽來的，而是他在困境裡做出的選擇，謠言說的都是 Uncommitted 的版本，唯有親眼看見的 Snapshot，才是真實的他


<!-- endtab -->

<!-- tab IsolationLevel-->

> 在同一段交易裡，你接受看到「哪一種樣子的世界」？

在多人同時查資料、改資料的世界裡，不可能做到「既快、又完全正確、又不互相干擾」。
資料庫永遠在做取捨，而 IsolationLevel 本質上就在回答這個問題

你想優先保護什麼？速度？正確性？穩定性？還是資料不被干擾？


| 你想要什麼？         | 你必須接受什麼代價？             | 適合的 IsolationLevel  | 評價                              |
| -------------- | ---------------------- | ------------------- | ------------------------------- |
| ⚡ 最快速度         | ❌ 資料可能不準        | **ReadUncommitted** | 那些流言蜚語都聽進去       |
| ✔ 正確資料         | ❌ 等別人寫完才能讀      | **ReadCommitted**   | 很乖但慢一點可以看     |
| ✔ 查完後資料不能變     | ❌ 鎖多、會卡住 | **RepeatableRead**  | 只鎖住在意的部分，個人主義 |
| ✔ 查詢期間世界一致     | ❌ 舊版本、版本庫大  | **Snapshot**        | 別人偷改資料前叫他揪都馬得我要先拷貝一份舊版   |
| ✔ 整個世界不准動，等我做完 | ❌ 最慢、鎖最多         | **Serializable**    | 被罵有特權      |


![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/IsolationLevel/2__readuncommitted__risk.png)


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/IsolationLevel/2___spectrum.png)


![op](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/IsolationLevel/2___the__table.png)



<font color=#D3D3D3 style="font-size: 22px;">&emsp;&emsp;&emsp;&emsp;&emsp;什麼都無法捨棄的人，就什麼都無法改變 - Armin Arlert </font>

<br/>

![Armin Arlert](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Titan/sacrifice.png)


<!-- endtab -->

<!-- tab IsolationLevel = SnapShot-->


Snapshot 的核心是 『我不保證你讀的是最新，但一定保證你讀的是真實存在的資料』

「在你查資料那一刻世界的樣子，我把正確的資料凍結給你。」


就像按下快門那一瞬間，你看到的畫面，就是接下來整段查詢要依據的真實，哪怕外界已經天翻地覆，哪怕別的交易把資料改了又改，你看到的世界都不會被扭曲或破碎

<br/>

<font color=#D3D3D3 style="font-size: 22px;">&emsp;&emsp;&emsp;&emsp;你只要按下按鈕，其餘的都交給我們 </font>

<br/>

<font color=#D3D3D3 style="font-size: 22px;">&emsp;&emsp;&emsp;&emsp;&emsp;- 柯達創始人喬治·伊士曼（George Eastman）於 1888 年創造的廣告語 </font>


![kfj](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/IsolationLevel/2___snapshot__camera.pn)


<!-- endtab -->

<!-- tab SnapShot - 一步步來-->

我們現在已經知道，在查詢時，使用 SnapShot 的隔離層級查詢資料，資料庫會適時地幫我們記住資料，也就是快照，也就是說，我們需要一個「記住世界的時間點」作為基準，也就是 Timestamp

<br>

## 查詢開始前 - 建立 Snapshot Timestamp（版本時間點）

> ✅ SP 第一次查資料時，SQL Server 會建立一個「Snapshot Timestamp」，稱為 Transaction Version。

這個時間點就像拍照那一瞬間，接下來你看到的資料都只能來自「這一刻之前」的版本


![fk](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/IsolationLevel/3__create_timestamp.png)



## 資料異動被異動!

> ✅ 如果其他交易在這個 snapshot timestamp 之後對某筆資料做了 UPDATE 或 DELETE
> 🔁 SQL Server 會自動把「修改前的版本」備份到 tempdb 的 version store

tips : 不是你查詢就會備份，是 你正在查 + 有人動資料 才會備份


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/IsolationLevel/3__2__version__stor)


####　拿資料的當下：依據快照時間點決定資料來源

| 狀況（資料相對於 Snapshot Timestamp 的狀態）                           | SQL Server 判斷  | 資料來源                      | 查詢結果        | 為什麼會這樣？（本質）               |
| ---------------------------------------------------------- | -------------- | ------------------------- | ----------- | ------------------------- |
| 1️⃣ **資料在快照時間點之前就存在，而且沒被改過**                               | ✔ 版本符合快照       | 主資料表                      | ✔ 看得到       | 它本來就存在於你當時「拍下的世界」裡        |
| 2️⃣ **資料在快照時間點之前存在，但之後被修改過**                               | ❌ 主表太新 → 不能讀主表 | tempdb version store      | ✔ 看得到（舊版本）  | 你不能看到快照後的變化，所以改用舊版本       |
| 3️⃣ **資料在快照後才被新增（INSERT）**                                 | ❌ 不符合快照        | 無                         | ✖ 看不到       | 因為新增的資料不在你的照片裡            |
| 4️⃣ **資料在快照後被刪除（DELETE）**                                  | ❌ 主表沒有該資料      | tempdb version store 有舊版本 | ✔ 看得到（舊版本）  | 在你拍照時它還存在，所以要還給你          |
| 5️⃣ **資料在快照時間點後被更新，但更新前沒有舊版本存下來（極少見，通常因沒有 Snapshot 查詢在場）** | ❌ 沒有可用版本       | 無                         | ✖ 看不到 / 或錯誤 | 沒有 Snapshot 查詢，就不會備份舊版本給你 |
| 6️⃣ **沒有任何修改行為（表很安靜）**                                     | ✔ 所有資料版本都穩定    | 主資料表                      | ✔ 全部可讀      | 沒有版本衝突 → Snapshot 走得最順的情況 |


以上就是使用這個隔離層級去查詢資料，整個過程發生了甚麼事，他與 ReadUnCommitted 相同不會與其他交易互卡，但可以確保資料不會是破碎的髒資料


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/IsolationLevel/3___visibility__by_snapshot.png)



<!-- endtab -->

<!-- tab 資料庫設定-->


要讓 `Snapshot` 正常運作，其實不是只在程式裡 `SET TRANSACTION ISOLATION LEVEL SNAPSHOT` 就好，因為 `Isolation Level` 只是「查詢提出的要求」，資料庫要先「準備好版本機制」，才能滿足這個要求

SQL Server 要能提供 `Snapshot`，就必須先有能力「幫你留住舊版本」。這件事是由兩個資料庫層級的設定決定的

- `ALLOW_SNAPSHOT_ISOLATION`
- `READ_COMMITTED_SNAPSHOT`


只有當資料庫有準備好相機時，SQL Server 才會啟動 `Row Versioning`（行版本控制） 機制，也才會在 tempdb 裡幫你保存「被修改前的舊版本資料」


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/IsolationLevel/4__create__snapshot__)


<!-- endtab -->

<!-- tab 使用 TempDB 儲存「行版本（Row Versions）」-->

SQL Server 不能只留現在的資料，還得額外多記一份「過去版本」，也就是 Row Version（行版本）。而這些「舊版本」是放在 tempdb 的 version store 裡

使用 SNAPSHOT 的本質是「資料庫要額外幫你記住查資料時刻的樣子（也就是所謂的版本）」，這麼做雖然讓查詢的資料變得穩定且一致

- 資料庫要維護更多的資料版本（Row Version）
- 資料庫的 記憶體、磁碟、效能開銷都會變大
- 同時，也會有某些「寫入的衝突問題」

每一次資料更新時，SQL Server 不能直接改資料，而是要把「原本的資料」複製一份到 tempdb、再去修改主資料表中的值，也就是說，你一筆 UPDATE，實際上產生了兩份資料！

- TempDB 負擔變重
- 硬碟 I/O 增加
- 若 tempdb 空間不足，會導致查詢失敗（錯誤訊息：version store full）
- 頻繁寫入下，效能下降嚴重


![vv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/IsolationLevel/4__snapshot__cost.)


<!-- endtab -->

<!-- tab 寫入衝突（Update Conflict）-->


雖然 SNAPSHOT 查資料時不會被鎖，但「寫資料」的時候會做版本比對，發現你讀的版本跟當前版本不一致時，就會直接拋出錯誤 : `Snapshot isolation transaction aborted due to update conflict`，結果可能造成造成交易失敗



![v](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/IsolationLevel/5__update__conflict.png)


<!-- endtab -->

<!-- tab 什麼情況下用 SNAPSHOT 最適合？-->

Generally, 查多、寫少 的系統會比較適合，可以避免 `Dirty Read` 被放大

- 報表、BI
- API（查資料居多）


<!-- endtab -->

<!-- tab 什麼情況下用 SNAPSHOT 不適合？-->

- 電商 Checkout
- POS
- 股價、報價系統

因為 CRUD 可能太頻繁了，導致要存一堆版本 → tempdb 可能會爆掉，遇到 Update Conflict 更高也會讓交易一直失敗

<!-- endtab -->

<!-- tab 一個真實的 DirtyRead 案件-->

系統透過 API 取得線上訂單，訂單資訊會被分成兩個步驟查詢

- getlist：查詢訂單清單
- get：查詢單筆訂單詳細內容

然而，在高 RPS 的環境下，因為使用了 `WITH (NOLOCK)`（也就是 `ReadUncommitted` ），導致系統讀到「不一致的資料」，進一步影響了後續出貨工作

1. 2025-08-06 09:08:00 (UTC+8) 查詢訂單 getlist 回傳了 TG250806K00007 | TM250806K00007 有 2 筆 TS (TS250806K000018 及 TS250806K000019)
2. 2025-08-06 09:08:01 (UTC+8) 取得訂單詳情 get 時 "TM250806K00007"，API 只回傳 1個 TS250806K000019，缺少了 TS250806K000018 ，導致未能完成出貨流程!



![v](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/IsolationLevel/7___phantom__order.pn)


查案後發現，針對 `csp_GetSalesOrderDataForApi` 中，為避免髒讀 (Dirty Read) 與幻讀 (Phantom Data)，建議將隔離層級改為 `SNAPSHOT` 並移除 `WITH (NOLOCK)`


![v](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/IsolationLevel/7__fix__with_snaps)


<!-- endtab -->

<!-- tab 結語-->

一天，一場突如其來的大雨打亂了村子。有個孩子跌倒、哭著回不了家，大家都在忙著躲雨時，只有他撐著一把又舊又大的傘，默然地走向小孩，把傘蓋到孩子頭上，自己則濕了一片。那一刻，所有人都愣住了

那皺眉背後是擔心；那沉默只是不擅言語

村裡的此時意識到隔離層級沒有做適當的設計，過去總是聽來的、揣測的、未經證實的版本，不如透過自己親眼看到的 Snapshot


![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/IsolationLevel/8___the__man.pn)



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/IsolationLevel/9__sum.png)


![k](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/IsolationLevel/unnamed.png)



<!-- endtab -->


{% endtabs %}