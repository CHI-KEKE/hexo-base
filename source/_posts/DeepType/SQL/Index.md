---
title: Index
date: 2025-10-04 20:47:05
categories: 情報の存在論
top_img: https://i.imgur.com/NfCnwwU.png
cover : https://i.imgur.com/NfCnwwU.png
tags:
    - 情報の存在論
toc:
toc_number:
comments :
---

{% tabs Index %}

<!-- tab 太陽拉麵-->

一天，阿冠帶著夥伴腦袋與毛吉，到 LaLaport 尋找夢寐以求的「太陽拉麵」

到了商場門口，三人抬頭一看

哇哩！這地方足足有六層樓，還超級大！

<br>

腦袋皺著眉說：「我們從一樓開始，一層一層慢慢找吧！」


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/Index-1/1__table__scan.png)


毛吉在旁邊激動地汪汪叫：「阿烏旺旺旺旺！」看起來沒有甚麼幫助

<br>

阿冠搖搖頭，露出無奈的表情，仿佛已經習慣這一切

<br>

「Follow me」

<br>

三人走到一樓的樓層導覽地圖前，阿冠熟練地滑動著手指，找到了那行關鍵字：「🍜 太陽拉麵 → 3樓中區 C312」

<br>

「走吧！直奔三樓中區！」沒花幾分鐘，他們就聞到濃濃的豚骨香


![sun](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/ramen.png)


腦袋驚呼：太神了吧！我們沒繞半圈就找到了！」

<br>

阿冠微微一笑，扶著眼鏡：**「這是 Index 的力量!」**


<!-- endtab -->

<!-- tab B Tree Index-->

在多數資料庫中（例如 SQL Server、MySQL、PostgreSQL），預設使用的索引結構就是 B-Tree（Balanced Tree）

假設你建立一個索引
```sql
CREATE INDEX IX_Customers_Email ON Customers(Email);
```

這時資料庫會做幾件事

- 把 Customers 資料表裡所有 Email 欄位的值取出來
- 幫這些值「排序」
- 建立一棵 B-Tree（平衡樹） 結構來儲存它們
- 每個節點會記錄：
  - 一個 Email 值
  - 對應這筆資料在資料表裡的位置（Row Pointer）


你可以想像一棵「有秩序的資料樹」
```plaintext
             [brian@yahoo.com]
             /                \
 [andy@gmail.com]       [charlie@outlook.com]
```

這棵樹有個簡單但超關鍵的規則

- 左邊的值比節點小
- 右邊的值比節點大

當你要查某個 Email，比如 'charlie@outlook.com'，資料庫就會這樣走：

- 從樹的「根節點」開始比對
- 發現 charlie > brian → 往右邊走
- 到右邊節點一比對 → 找到了 ✅ 耶比

索引的排序方式，完全取決於你在建立索引時指定的欄位，每次比較都能「排除掉一半的可能範圍」，也就是 log(n)，所以查詢起來很快


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/Index-1/1__build__index.png)


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/Index-1/2__b_tree_log.png)


<!-- endtab -->

<!-- tab 經常變動的欄位上建立索引，有什麼風險？-->

更新與插入成本會提高，可能導致效能反而下降，因為索引會隨著資料的變動而更新。如果某欄位更新非常頻繁，會導致

- 索引更新成本高（尤其是在高併發下）
- 索引碎片增加，查詢效能下降

所以不建議在以下欄位加索引

- 使用率低且經常變動的欄位（如「登入次數」）
- 隨時更新的狀態欄位（如「IsOnline」）

所以建議是儘量只在查詢頻繁且變動不大的欄位上加索引



![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/Index-1/2__index_cost.png)


<!-- endtab -->

<!-- tab Clustered Index 與 Non-Clustered Index-->

| 類型                      | 本質是什麼        | 資料放哪裡     |
| ----------------------- | ------------ | --------- |
| **Clustered Index**     | 資料本身就照索引排序儲存 | 資料本體就是索引  |
| **Non-Clustered Index** | 索引是額外的導覽表    | 索引指向資料的位置 |

- Clustered Index 資料本身就按照索引的順序儲存，沒有「再指過去」的過程
- Non-Clustered Index 索引表是獨立的，它只記錄「欄位值」 + 「資料所在位置（Row Pointer）」，查完索引後，還要「再跳一次」才能拿到資料


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/Index-1/3___cluster__noncluster.png)

<br>
<br>
<br>


今天我們要找太陽拉麵在哪...

<br>
<br>
<br>

## 方法一：Clustered Index 模式

「商場的店就是照樓層與店號排列，所以我們只要看店號 301，就直接走到那裡就好。」不用再查地圖，因為商場本身就是按照順序排好的


![aaa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/Index-1/4__clustered_index.png)


## 方法二：Non-Clustered Index 模式

「但如果你只記得店名太陽拉麵，那就得先去一樓查地圖（索引表），看太陽拉麵在哪個店號 → 然後再走過去。」多了一個中間步驟，但仍比亂找快得多


![aaa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/Index-1/5__nonclustered_index.png)




<!-- endtab -->


<!-- tab Index Scan-->


Index Scan 就是，將某個索引的葉節點（資料入口）從頭到尾一路讀，中間可能還是會套用 filter，它不是「完全不走索引」，但「走索引但整本翻完」

假設我們有個 Non-Clustered Index
```sql
CREATE INDEX IX_Customers_Email ON Customers(Email);
```
```sql
SELECT Name, City FROM Customers WHERE Email LIKE '%gmail.com';
```

查詢條件（Email）在索引中 ✅
但要回傳的欄位 Name、City 不在索引裡 ❌

所以引擎會

- 掃完整個索引（從第一筆 Email 到最後一筆）
- 找到符合條件的 Email
- 透過索引裡記錄的 Row Pointer（或 Clustered Key）

再去主表取回 Name、City → 這動作就是 Key Lookup

因為索引裡沒有完整資料，要「回表」補資料，雖然也是全掃，但是掃「索引表」，再連回原表

| 面向     | Index Scan             | Clustered Index Scan |
| ------ | ---------------------- | -------------------- |
| 掃描對象   | Non-Clustered Index（小） | Clustered Index（大）   |
| 讀取欄位   | 只有索引欄位                 | 所有欄位                 |
| 是否需回表  | 可能需要（Key Lookup）       | 不需要                  |
| 掃描範圍   | 全索引                    | 全表（因索引=表）            |
| I/O 成本 | 較低（索引較小）               | 較高（資料列完整）            |
| 常見原因   | 條件使用索引欄位，但篩選範圍大        | 條件用不到任何索引            |

<!-- endtab -->

<!-- tab Clustered Index Scan-->

資料以「頁（Page）」為基本單位（SQL Server 8KB/頁，8頁 = 1個 Extent）。

- Sequential I/O：一次連續讀很多頁（配合 Read-Ahead），吞吐量高
- Random I/O：不停跳位置取頁，IO 數多、延遲高

Clustered Index Scan 也是「全表掃描」，但因為資料在葉節點按 Clustered Key 連續排列，引擎能用 Sequential I/O（連續讀）＋ Read-Ahead（預先抓一大批頁），整體吞吐量高

<br>

如果有 Clustered Index（資料在葉節點照鍵值有序），走 Clustered Index Scan 步驟會是這樣

- 定位起點：從 Clustered Index 的最左葉節點開始（最小鍵值那頁）
- Read-Ahead：引擎看見葉節點間有「雙向鏈結」且鍵值連續，直接向儲存層一次預取一大批連續頁（例如一次抓數十～數百頁；實際大小由引擎決定）
- Sequential I/O：儲存層順順地把 Extent A1、A2、A3… 連續送進 Buffer Pool
- 逐頁輸出：每頁的資料行已在同一頁，要的欄位也都在葉節點（Clustered 的葉節點存整列），不必 Key Lookup
- 持續直線前進：下一批 Read-Ahead 繼續取 A4～A8、A9～A16… 幾乎不需要磁碟頭（或邏輯位址）大跳
- 完成掃描：整體 I/O 成本接近「線性大吞吐量模式」

👉 連續、可預測、少跳轉 → 吞吐量高 → 大表更有感



<!-- endtab -->


<!-- tab Seek vs Scan 的「取捨邏輯」-->


![aa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/Index-1/6___seek_and_scan.png)


## 1️⃣ Index Seek 是「精準找幾筆」

資料庫利用索引快速定位 → 再用 Key Lookup 取資料。查少量資料時快。每筆都要回表（多次 I/O）若結果太多，回表次數暴增 → 效能反而下降
所以當結果很多時，Seek 的「多次跳來跳去」會拖慢速度。

但不是「Seek」一定慢，也不是「比數多」一定慢，而是要看「要的資料是不是都在索引裡」！

| 類型                             | 說明           | 是否要回表 (Key Lookup) |
| ------------------------------ | ------------ | ------------------ |
| ✅ **Covering Index Seek**      | 查詢要的欄位都在索引裡  | ❌ 不用回表             |
| ⚠️ **Non-Covering Index Seek** | 查詢要的欄位不全在索引裡 | ✅ 需要 Key Lookup    |


![aa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/Index-1/7__convering_index.png)



## 2️⃣ Index Scan 是「整份掃一遍」

資料庫直接從索引的第一筆掃到最後一筆。邏輯簡單、連續讀（Sequential I/O）、不需回表太多次，但一定會掃完整份索引，即使結果只要少數幾筆。所以當結果筆數很多（例如整張表的 30~50% 以上），掃一次反而比「跳幾萬次 Seek」快


![aa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/Index-1/6___preffered_scan.png)




![aa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/Index-1/6__why_to_scan.png)


<!-- endtab -->

<!-- tab 典型 Index Seek（少量、可定位）-->

表：Orders(OrderDate, CustomerId, Total, ... )
索引：IX_Orders_OrderDate 在 OrderDate


```sql
SELECT *
FROM Orders
WHERE OrderDate >= '2026-02-01' AND OrderDate < '2026-02-02';
```

為什麼是 Seek

- 條件是對索引 key（OrderDate）的「連續範圍」
- SQL Server 可以直接跳到 2026-02-01 那段，抓到 < 2026-02-02 就停
- 很像「翻到日曆 2/1 那頁開始抄，抄到 2/2 前就收工」


<!-- endtab -->


<!-- tab 什麼狀況特別容易 Scan（而不是 Seek）-->


## 查詢條件不是索引的 leading key（最左欄位）

索引是 (A, B)，你只用 B=... → 常見 Scan


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/Index-1/8__turn_to_scan_leading.png)


<br>


## 對索引欄位做函數/運算


```sql
SELECT *
FROM Orders
WHERE YEAR(OrderDate) = 2026;

```

為什麼容易變 Scan，因為 YEAR(OrderDate) 是對欄位做函數加工，而索引是按 OrderDate 排序，但你現在問的是「加工後的值」，SQL Server 很難直接跳到「YEAR=2026 的區段」，結果就是把整個 IX_Orders_OrderDate 從頭掃到尾，一筆筆算 YEAR 再判斷，因為這是問「所有年份是 2026 的頁面」，這只好從頭翻，翻到每一頁都看年份是不是 2026


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/Index-1/8__turn_to_scan_refined.png)


<br>

## LIKE 前面有萬用字元

LIKE '%abc' 通常不容易 Seek


<br>


## 命中比例太高

估算會撈很多筆 → Scan 比 Seek + Lookup 便宜


<br>


## 統計資訊判斷會撈很多


就算你以為會少，統計覺得會多，也可能 Scan

```sql
SELECT OrderId, OrderDate, CustomerId, Total, ShippingAddress
FROM Orders
WHERE OrderDate >= '2026-01-01';
```

假設這會命中 60% 的資料，而且 ShippingAddress 不在那個索引裡，如果走 Seek 會先抓到一堆符合的 key（可能幾千萬筆），每一筆都要做 Key Lookup 回到 clustered index / heap 拿 ShippingAddress，變成「找到名單很快，但要一筆筆去倉庫搬貨」造成超多隨機 I/O，SQL Server 可能改選 Index Scan / Clustered Index Scan 直接順著讀（比較連續）或掃 clustered index（因為反正要讀很多行）


<!-- endtab -->

<!-- tab summary-->

![r](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/Index-1/9__final.png)


![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Index/Index-1/index-1.png)


<!-- endtab -->

{% endtabs %}


