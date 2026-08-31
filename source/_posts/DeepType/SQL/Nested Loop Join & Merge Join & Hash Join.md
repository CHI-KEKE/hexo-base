---
title: Nested Loop Join & Merge Join & Hash Join
date: 2025-10-07 08:27:05
categories: 情報の存在論
top_img: https://i.imgur.com/NfCnwwU.png
cover : https://i.imgur.com/NfCnwwU.png
tags:
    - 
toc:
toc_number:
comments :
---

{% tabs Join Plan %}

<!-- tab 戀愛類型-->

你是什麼樣的戀愛類型呢？

毛吉是那種聞到喜歡的味道就會湊上去確認的類型，因為她他有一開始就設定好標準，只能靠一次次互動中慢慢觀察、逐一比對。透過 Nested Loop 聞到喜歡的屁股就 Join 下去，通常吃幾個巴掌


![Mochi](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Moji/moji1.png)


阿冠比較理性，屬於先驗條件獵人型。他會先用一套邏輯，把自己「Hash」好，然後就靠這組標準去掃描目標對象。這就是 Hash Join，直接把效率拉到極致

最理想的狀況，當然是兩個人都已經「排序」好自己，生活節奏對齊，價值觀同步，見面時幾乎不需要猜測，搭上就能對得很漂亮，就是 Merge Join



<!-- endtab -->

<!-- tab Nested Loop Join-->


「外層每一筆資料，都去內層找一筆相對應的資料」，**效率完全取決於內層查找是否能快速定位**

有兩份名單

- Outer Table：10 個學生名字
- Inner Table：100 萬筆成績紀錄

要做的事是

「拿出第一個學生 → 去成績表查他的資料」
「再拿第二個學生 → 去查他的資料」
...
...
...

➡️ 所以整個流程非常直覺，有點像是「外層迴圈包內層查詢」


![aaa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Join-Type/1___nested_loop.png)


## 內表能不能快速查到資料？

Nested Loop Join 效能好不好，90% 取決於內表能不能用索引直接 Seek

- 沒有索引時，內層每查一次就要掃全表 → 100 萬筆 × 10 個學生 → 直接爆炸
- 有索引時，每查一次只需 O(log N)，效率天差地遠


![v](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Join-Type/1__nested__depends_on_index.png)


## 小表 JOIN 大表最適合用 Nested Loop

Nested Loop 最理想的狀況，就是外表很小、內表很大，但內表有好索引，外表很小 → 外部迴圈跑很快，內表有索引 → 每次搜尋像子彈一樣快，這是 Nested Loop 的甜蜜點

<br>

## 可替代方案：Hash Join、Merge Join

Nested Loop 不被選擇時，多半是有更適合的 Join 演算法可選，因為當內表無索引、大資料量、或條件不精準時 → Hash Join / Merge Join 會更穩定

<br>

## Nested Loop Join 的效能瓶頸在哪裡？是 CPU 還是 I/O？

通常是 I/O，因為內層每次查找若沒有索引，就會導致磁碟大量隨機讀取，這是極其昂貴的操作。即使有索引，若資料未在記憶體中，也可能造成頻繁 Page Fault。CPU 反而是次要瓶頸，除非是在 in-memory 資料庫或已 Cache 的情況下

<br>

## 如何分析效能

- Loop 次數
- Estimated rows vs actual rows
- 使用的索引與存取方式（Seek vs Scan）
- I/O 成本


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Join-Type/1__nested__anal.png)


<!-- endtab -->

<!-- tab Merge Join-->

利用「兩邊都已排序」的特性，像拉鍊一樣從頭同步往下比，靠排序保證能一次線性掃描完成，他會兩份表都按照 join key 排序好（天然已排序或有 Clustered Index），指標從兩份表的第一筆開始往下比較

- 若相等 → match，輸出結果，兩邊都往下走
- 若左邊比較小 → 左邊指標往下
- 若右邊比較小 → 右邊指標往下

重複直到任一邊走完

➡️ 整體流程像 兩條已排序好的清單做同步比對
➡️ 因為是線性掃描，所以時間複雜度為 O(N + M)，非常穩定


有兩份都照學號排序的名單

學生表（100 筆）
成績表（100 萬筆）

拿著兩份表，從頭開始同步往下比
```bash
A001 vs A001 → match
A002 vs A002 → match
A003 vs A004 → 左邊往前
```


![es](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Join-Type/3_merge__join.png)


## 排序是前提條件：看起來快，但排序成本不能忽略

Merge Join 超快，但前提是：你已經先付出 sorting 的代價，Merge Join 跟 Nested Loop 最大差別是「預備作業」


## 適合大資料量：線性掃描的威力

只要排序問題解決，Merge Join 是處理「超大資料量」等值 JOIN 的王者


## 限制：只能用在「等值比較（=）」

<、>、between、like 等不能用 Merge Join，原因是排序要有固定錨點，才能從小到大有效對齊，一旦比較方式不是等值 → 排序無法建立穩定匹配


![ff](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Join-Type/3__merge__sort.png)


<!-- endtab -->

<!-- tab Hash Join-->

用「把小表變成一張 O(1) 查詢的 Hash Table」來加速比對大表的鍵值，避免排序與索引的成本

steps :

- 選一張小表 A（Build Input），因為 Hash Table 要放在記憶體，小表比較不會爆 RAM
- 把 A 的 Join Key 建成 Hash Table（Build Phase）, 以 key 為 hash
- 每個 key 對應 bucket → 存 A 的資料列，這讓之後查詢可以用 O(1) 平均時間
- 掃描大表 B（Probe Phase），逐筆讀 B 的資料
- 用 B.key 去 Hash Table 檢查是否存在，存在 → JOIN 成功，不存在 → 跳過
- 把匹配的資料輸出成 JOIN 結果

先建立高速查詢結構 → 再用大表去查那個結構



![ff](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Join-Type/2__hash_join.png)


## 哪些情況適合 Hash Join？

適合沒有索引、資料沒排序、查詢像資料倉儲一樣以「大量掃描」為主的場景。因為 Hash Join 的價值本來就是「無需索引也能跑得快」，如果你環境本來沒有索引，Hash Join 反而可以省下大量排序或索引讀取的成本


![vv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Join-Type/2__hash__why.png)


## 什麼情況反而不適合 Hash Join？

當 JOIN key 已高度索引化時，Hash Join 會浪費這些索引的威力，使效能變差，因為可能會以為「Hash Join 很快 → 一定最好」，但其實若小表 A 已經在 JOIN key 上有 Clustered/Non-Clustered Index，那 Nested Loop + Index Seek 的隨機讀非常快，此時建立 Hash Table 反而是沒必要的開銷

## 效能

HASH 是 CPU 密集型運算，在 CPU 嘗試 hash key 時會消耗不少 cycles，這在高併發 OLTP 系統可能造成壓力


![ss](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Join-Type/2__hash__costs.png)


<!-- endtab -->

<!-- tab 引擎怎麼做選擇-->


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Join-Type/5__decide_join.png)


| 條件                                                         | Optimizer 傾向使用                |
| ---------------------------------------------------------- | ----------------------------- |
| JOIN 條件為等號（`=`）<br>而且**兩邊都大、沒索引**                          | ⚙️ **Hash Join**              |
| JOIN 條件為等號（`=`）<br>而且**兩邊都已排序**或有索引（例如 Clustered Index）    | 🔗 **Merge Join**             |
| JOIN 條件為等號或範圍（`=`, `<`, `>`, `BETWEEN`）<br>而且**外表小、內表有索引** | 🌀 **Nested Loop Join**       |
| JOIN 條件是非等值（例如 `<`, `>`）                                   | 🌀 **Nested Loop Join**（唯一可行） |
| 表的統計資料估錯（Row Estimate 太低）                                  | 可能誤選 Nested Loop 而慢爆          |


| Join 類型     | 索引需求   | 排序需求   | 記憶體需求 | 大資料表效能 | 小表 vs 大表效能 |
| ----------- | ------ | ------ | ----- | ------ | ---------- |
| Nested Loop | ✅ 強烈建議 | ❌ 不需   | 🔹 低  | ❌ 差    | ✅ 極佳       |
| Merge Join  | ⚙️ 可用  | ✅ 需要排序 | 🔹 中  | ✅ 優    | ✅ 優        |
| Hash Join   | ❌ 不需   | ❌ 不需   | 🔸 高  | ✅ 優    | ❌ 中等       |


![v](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Join-Type/6__table.png)


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Join-Type/unnamed.png)


![cc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Join-Type/4__compare.png)


<!-- endtab -->

{% endtabs %}