---
title: Pagination
date: 2024-12-14 11:49:34
categories: 情報の存在論
top_img: https://i.imgur.com/NfCnwwU.png
cover : https://i.imgur.com/NfCnwwU.png
tags:
    - 情報の存在論
toc:
toc_number:
comments :
---

{% tabs Pagination%}

<!-- tab 照著進度走的人生-->

世界是一整本書，人生可以被清楚地切成第 1 頁、第 2 頁、第 50 頁，我只要照著頁碼翻，就一定能看到我該看到的那一段，像是 30 歲要做到第 30 頁、35 歲要結婚、40 歲要買房，同齡人u3已經在地第 20 頁，我怎麼還在第 15 頁？

因為你的人生座標來自「別人在哪一頁」，這就是 OFFSET FETCH!

![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pagination/1___offset_life.png)


感覺 OFFSET 規規矩矩的，一頁頁走應該沒甚麼問題，然而，OFFSET 有一個隱藏陷阱!

世界不是靜止的書，有人中途離場（刪資料）、有人突然插隊（新增資料），頁碼可能看起來沒有錯，但那一頁的內容已經變了，明明我照順序走，卻發現自己重複經歷同樣的痛苦，又或者莫名其妙的錯過了一些重要的東西，OFFSET 的人生，會迷失在比較與對齊之中。也不是說 OFFSET 是不好，但我們不能一直用 OFFSET 活著!


![jk](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pagination/2___offset___missing.png)


Keyset 的世界觀就完全不同，我不管現在是第幾頁，我只記得我上一次走到哪裡，我知道自己現在的能力邊界，我清楚目前累積到哪個階段，下一步，只要比「現在的我」再前進一點，你不再問：「我是不是落後了？」而是問：「我是不是比昨天更靠前？」

不容易被插隊影響、不怕世界變動、不會因為別人刪掉或新增什麼，就懷疑自己整段旅程，因為人生不是靠「頁碼定位」，而是靠「狀態連續」

![vv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pagination/2____keyset___mind.png)


<!-- endtab -->

<!-- tab OFFSET / FETCH-->

「在已排序的結果集中，用明確的規則只取你想看的那一段資料」，讓資料查詢可以被「分頁」使用

```SQL
SELECT emp_id, emp_name, salary
FROM employee
ORDER BY emp_id
OFFSET 20 ROWS        -- 跳過前 20 筆
FETCH NEXT 10 ROWS ONLY; -- 取 10 筆資料
```
‵
- `FROM employee`，從 employee 這張表把所有資料先拿出來（注意：此時還沒有分頁）
- `SELECT emp_id, emp_name, salar`y，指定你只關心這三個欄位，其他欄位先不理。
- `ORDER BY emp_id`，先把「整個結果集」依 emp_id 排好順序，因為 `OFFSET / FETC` 一定是「對排序後的結果」動手
- `OFFSET 20 ROWS`，把排序後的結果「前 20 筆直接丟掉不看」
- `FETCH NEXT 10 ROWS ONLY`，在跳過前 20 筆之後，再拿接下來的 10 筆資料出來

「我要第 21～30 筆員工資料」


<!-- endtab -->

<!-- tab 程式端 VS Stored Procedure-->

## 複雜但穩定 → SP 很香（效能也好控）

分頁邏輯放在哪裡，取決於「變動成本」要由誰承擔。當分頁查詢「長得都一樣」時交給 Stored Procedure，當使用端地應用也不用太擔心，因為我們不太會變動他

「後台訂單列表」永遠都有狀態、是否刪除、時間區間、固定幾種排序，而且可能 join 訂單、會員、付款、物流很多表，但需求很少變，這種放 SP，DB 可以針對固定查詢做索引/統計/最佳化，多系統共用同一支 SP，不會寫出不同版本

例如使用者清單、商品清單、訂單清單，選項只有那幾樣，固定排序（建立時間 / ID）、固定條件（狀態、是否刪除）、固定分頁（page + pageSize），若直接寫在 sp 優點是

- DB 已最佳化
- 查詢一致、好控管
- 多個系統可共用

## 查詢方式一直變

「前台商品搜尋」join商品、分類、品牌、活動價、庫存、評價，但更麻煩的是一直加規則，本週主打要排前面、不同會員等級價格不同、A/B test 改排序策略、不同入口要插推薦/廣告，這種如果硬放 SP 會得到一支「超長 SP」，裡面塞一堆 if/動態 SQL，改一次很容易影響別的策略，測試成本飆升


![ff](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pagination/7____where_to_put__code.png)


<!-- endtab -->

<!-- tab OFFSET 實現分頁的效能問題-->

OFFSET 請資料庫「先走過不需要的資料，再把你要的那一小段拿出來」，而 Keyset 是讓「查詢條件本身就具有進度狀態」，而不是讓資料庫在結果集裡幫你計數，他用可比較的鍵值，將無狀態查詢轉為具狀態的資料存取

```SQL
SELECT emp_id, emp_name, salary
FROM employee
ORDER BY emp_id
OFFSET 20 ROWS        -- 跳過前 20 筆
FETCH NEXT 10 ROWS ONLY; -- 取 10 筆資料
```

隨著 OFFSET 的增加，速度會越來越慢；因為即使我們只需要返回 10 條記錄，資料庫仍然需要訪問並且過濾掉 N（比如 1000000）行記錄，即使 id 有建立索引，OFFSET 的效能問題仍然存在。索引雖然能讓排序更快，但仍要遍歷前 N 筆索引項目。也就是說，它只是「有順序地走很快」，但...還是要走過去


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pagination/4____offset___loading.p)


![cc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pagination/4____offset___risks.png)


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pagination/4___offset___cost.png)


<!-- endtab -->

<!-- tab 「鍵值分頁（Keyset Pagination）」優化-->

與其讓資料庫「跳過」一堆資料，不如我們自己記住「上一頁最後一筆的 id」，這種方式被稱為 Keyset Pagination 或 Seek Method
```SQL
SELECT TOP 10 *
FROM employee
WHERE emp_id > @last_id
ORDER BY emp_id;
```
如果 id 欄位上存在索引，這種分頁查詢的方式可以基本不受資料量的影響!


![aa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pagination/5___keyset___intro.png)


![gh](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pagination/5___keyset__is__fast.png)


<!-- endtab -->

<!-- tab Keyset Pagination 真的是「萬能解法」嗎-->

有的需求，用 Keyset Pagination 就會變得相當麻煩

- 「跳到第 50 頁」
- 「依 salary DESC 分頁」
- 「排序條件由使用者自由切換」

這些操作更適合用 OFFSET，或其他折衷方式處理

- 後台管理系統 → OFFSET（資料量可控）
- API / 無限滾動 → Keyset
- 排行榜 → 預先計算 / 快取

適合使用 OFFSET 的情境像是

- ✔ 資料量小（幾千筆以內）
- ✔ 後台管理頁（偶爾用）
- ✔ 需要「跳頁」功能（例如直接到第 50 頁）


![cc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pagination/6___keyset___risk.png)


## 問問自己，你到底是想「瀏覽資料」，還是「定位資料」？

- 瀏覽 → Cursor / Keyset
- 定位 → 搜尋條件 + Index

像 Instagram、Twitter 從來不會有「跳到第 1000 頁」這種操作，因為它們的核心體驗是一直滑一直爽

- 社群動態（Facebook / IG / Twitter）
- 電商商品列表
- 訂單紀錄
- 日誌（Log）
- 無限滾動（Infinite Scroll）
- API 分頁（效能關鍵）

「線上系統、高流量系統」需要考慮使用 Keyset Pagination


![Cc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pagination/8___offset__or_keyset.png)


<!-- endtab -->

<!-- tab 如果資料在分頁期間被新增或刪除，OFFSET 與 Keyset 哪個更安全-->

這時候兩者的本質差異就顯現出來了

- OFFSET 分頁是「位置導向」
- Keyset 分頁是「值導向」

當資料變動時 OFFSET：頁面會「漂移」、Keyset：結果仍然連續，OFFSET 的隱性風險不是慢，而是「不一致」

使用者打開第 1 頁，中間有人刪掉一筆資料，接著使用者點第 2 頁 → 結果出現「資料重複」或「資料被跳過」，這種情況在高一致性場景（像交易紀錄）是不可以接受的

<!-- endtab -->

<!-- tab summary-->

![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pagination/9___how_to_choose.png)


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pagination/unnamed.png)

<!-- endtab -->



{% endtabs %}