---
title: Pagination
date: 2025-09-14 22:00:03
categories: 設計図鑑物語
top_img: https://i.imgur.com/LF7L1Ws.png
cover : https://i.imgur.com/LF7L1Ws.png
tags:
    - 
toc:
toc_number:
comments :
---




## 為什麼需要「分頁」

當資料量很多時，一次全部載入會出現幾個問題：

- 傳輸時間變久（效能差）
- 前端渲染太慢（使用者體驗差）
- 資料可能會更新（你讀到的資料其實過時了）
- Server 壓力大（記憶體吃光、CPU 爆掉）

所以我們會把資料「一頁一頁」傳給前端，這就叫做 分頁（Pagination）。



## Offset/Limit 分頁

用 OFFSET 跳過前幾筆資料，用 LIMIT 限制要拿幾筆。

```sql
-- 假設每頁 10 筆
-- 第1頁
SELECT * FROM Products ORDER BY Id LIMIT 10 OFFSET 0;

-- 第2頁
SELECT * FROM Products ORDER BY Id LIMIT 10 OFFSET 10;

-- 第3頁
SELECT * FROM Products ORDER BY Id LIMIT 10 OFFSET 20;
```

**問題一：效能差（尤其當 OFFSET 很大時）**

就像你去圖書館要看第 10000 頁的書，圖書館員還是要翻過前面 9999 頁才能給你第 10000 頁。資料庫也是一樣，它要先讀取前面 9999 筆資料，再丟掉，只為了給你第 10000 筆開始的資料。所以 OFFSET 越大，查詢越慢。


**問題二：資料會「跳頁」、「漏資料」、「重複資料」**

這個是使用者看不出來，但實際超級致命的問題。

假設你在第 2 頁看到的第 10 筆資料是「商品 A」，但就在你按下一頁之前，有人刪除了第 5 筆資料，導致整個排序往前移。

你跳到第 3 頁時，本來應該出現的新資料被往前推，你就「跳過」了那些資料，永遠看不到！

這就是所謂的：

- 資料跳頁（跳過）
- 漏資料
- 重複資料


## 游標分頁（Cursor-based Pagination）

不是用「第幾頁」來找資料，而是用一個**游標（Cursor）**來「記住目前看到哪裡」。你跟資料庫說：「我上次看到 id = 35 的資料，下一頁請給我 id > 35 的前 10 筆。」

這叫做 Keyset Pagination（或叫 Seek 分頁）

```sql
-- 第一頁
SELECT * FROM Products ORDER BY Id LIMIT 10;

-- 第二頁（記得上一頁最後一筆是 Id = 35）
SELECT * FROM Products WHERE Id > 35 ORDER BY Id LIMIT 10;
```

- 速度快：可以用索引快速查找 WHERE Id > ?
- 穩定性高：資料不會漏、不會重複、不會跳頁


**限制一：只能往後翻，不能直接跳到第 n 頁**

因為你是「記得上一筆」，不是「我想看第幾頁」。要看第 100 頁就得從第一頁慢慢翻過去，沒辦法直接跳。

**限制二：排序欄位必須 唯一、不會變動**

假設你用「價格」來當 cursor，結果價格會改變，可能出現這種情況：

第一次查到 price = 100 的資料
下一頁你說 WHERE price > 100，結果價格變動，資料亂了
所以最好是用主鍵（例如 Id、CreatedAt）來當作游標欄位



## 分頁 API 的回應要不要回傳總筆數


「API 回傳資料時，要不要同時告訴前端：總共有幾筆資料？（totalCount）」
👉 也就是「要不要多查一次 SELECT COUNT(*)？」


| 類型                 | 是否需要總筆數 | UI 特徵                              |
| ------------------ | ------- | ---------------------------------- |
| **可跳頁的分頁**         | ✅ 需要    | 有「第幾頁」、「共幾頁」、「前往第 n 頁」的功能          |
| **只能往前/往後翻**（無總頁數） | ❌ 不需要   | 像 Facebook 或 Instagram 的滑動載入「載入更多」 |


✅ 回傳總筆數的優點
✅ 前端可以顯示「第 3 頁，共 20 頁」
✅ 可以顯示「共找到 1,234 筆資料」
✅ 可支援「跳至第 10 頁」
✅ 適合表格型 UI（DataGrid, Table）

❌ 回傳總筆數的缺點

❗ SELECT COUNT(*) 在資料很大的情況下效能會很差 尤其當有很多 JOIN 或 WHERE 條件時
❗ 查詢兩次資料庫（一次 COUNT(*)，一次查分頁資料）
❗ 資料如果經常變動，這個「總筆數」很快就不準了（瞬間有使用者新增/刪除）



https://blog.darkthread.net/blog/tsql-paging-and-get-totalcount/




https://www.reddit.com/r/csharp/comments/gxwmqc/advanced_pagination_in_aspnet_core_webapi/?tl=zh-hant

https://www.threads.com/@ti399/post/DOSYmz1D2pY?xmt=AQF0xTRiXSn6jiIKvTvr_7zLSgD3Q1ezL6LtAjZP_cczDAQ&slof=1

https://www.threads.com/@chenstraveler/post/DMEnqQUS_xd?xmt=AQF0MmKf7TdhIxglns-F9w__62xcoqAQFZleWAtBsFmzbtw&slof=1


資料量決定一切
我才不相信半年超過千萬筆的資料
每一筆客戶都說要
如果要餵資料分析 那需求一開始就不對了

我還遇過不做分頁的，對於前端效能也是考驗

我的api還是回傳巢狀的，直接設定最多500筆結果😈

就問客戶：其實你想從系統知道什麼？😎

一次撈半年的資料量前端要展示也是要考慮效能 🤣
但如果是 excel 那種要一次幾萬筆的話，使用非同步可能可以 🤔

增加時間範圍的限制吧，既然查詢條件允許，為何不能順利執行？用戶也是依照正常的條件去查詢，沒有必要去質疑用戶

怎麼設計一個高效又正確的「分頁 API」？為什麼 offset/limit 在大資料量下會有問題？游標（keyset/seek）分頁要注意什麼  

這題剛好讀到，如有不夠完整再請脆友補充
offset/limit 是先取出資料，然後再丟棄 offset 之前的資料，只保留 limit 筆資料。在大資料量的情況下，資料庫需要掃描並丟棄大量的資料，導致效能下降
所以可以使用問題提到的Keyset/Seek來解決這個問題，基於特定順序id，紀錄最後一筆，下頁查詢從這筆id後開始，但會因此失去頁數資訊，因此如果有精確頁數需求，還需要討論一下前後端誰來處理

另外，資料在用戶讀取期間變動的話，用 offset/limit會造成漏掉資料或重複



6️⃣ ❓未加索引，導致分頁效能差？

本質問題：分頁排序欄位沒加 index

錯誤做法：ORDER BY CreatedAt 但 CreatedAt 沒索引

建議解法：

所有用於分頁排序的欄位都應建立 index（尤其是游標分頁）

若是 offset 分頁，offset + order by 欄位需配合 index



3. 快速查找過往資訊：

相信大家都有過找尋一些歷史訊息的經驗，可能找了幾頁資料，我們就可能10頁10頁這樣跳，大概會抓說可能10頁就是幾天的訊息，可能可以利用頁數，大概抓到資料會在第幾頁哪個位置。但是infinite scroll就無法做到這件事了，如果要找比較遠的資料，就必須一直scroll到底再等待載入，中間載入了許多我們不需要的資訊，就為了看比較久遠的資料，強迫我們必須經歷那個過程。

另外，有時候我們查找一些資料，需要再跳回去看的時候，我們大概會依稀記得之前看到的資料，大概會是在第幾頁，除了幫助記憶外，可以讓我們可以更快跳到想要的內容上。Infinite scroll因為scroll bar長度及位置的不停變動，無法像pagniation易於定位。

https://medium.com/uxcircles/pagination-vs-infinite-scroll-e1c3a3d682d9