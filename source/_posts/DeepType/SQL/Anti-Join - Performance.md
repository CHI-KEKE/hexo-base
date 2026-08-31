---
title: Anti Join
date: 2024-06-08 23:50:34
categories: 情報の存在論
top_img: https://i.imgur.com/NfCnwwU.png
cover : https://i.imgur.com/NfCnwwU.png
tags:
    - 情報の存在論
toc:
toc_number:
comments :
---


{% tabs Anti Join %}

<!-- tab 雞巴人-->

你曾經鼓起勇氣，傳訊息給喜歡的人，卻始終只收到那令人心碎的「已讀不回」？
你也曾在朋友聚會中，主動張羅大小事，結果現場只來了一半，還都最後才現身？

~~你就是個魯蛇~~

這背後有「資料科學」的真相，就是所謂的隱藏多數

<br/>

<font color=#D3D3D3 style="font-size: 22px;">&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;全部的人 - 出現的人 = 雞巴人 </font>

<br>

今天，我不想再假裝一切都沒發生。我要來製作一張黑名單，把這些嘴上說愛你、行動卻放你鴿子的人，通通揪出來！

手上有兩份資料：

- Friends（朋友名單）：那些理論上應該會出現的傢伙
- Attendance（實際到場名單）：那些真的有出現、值得你好好珍惜的朋友



![aa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Anti-Join/1___pigeon_case.png)



接下來，我們要開始用幾個方法，來抓出那些含糊不清的嘴砲王

![rooster](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/rooster.png)

<!-- endtab -->

<!-- tab NOT IN-->

「拿一個主表的鍵，去跟一張子查詢產生的名單比對，只留下『不在名單裡』的那些資料」

```SQL
SELECT FriendID, FriendName
FROM Friends
WHERE FriendID NOT IN (SELECT FriendID FROM Attendance);
```

<br>

1. 執行子查詢，產出「出席名單」
   - SELECT FriendID FROM Attendance
   - 資料庫會把 Attendance.FriendID 抽出來，概念上變成一個集合（set）或清單。
   - 這一坨就是「已經出席過的 FriendID 名單」。

<br>

2. 掃主表 Friends，逐筆拿 FriendID 出來比對
   - 對 Friends 每一列，拿出 Friends.FriendID。
   - 問一句：這個 FriendID 有沒有在剛剛子查詢那張「出席名單」裡？

<br>

3. 用三值邏輯做 NOT IN 判斷
   - FriendID 確定 不在 子查詢名單 → 條件為 TRUE → 這筆朋友保留。
   - FriendID 有在 子查詢名單 → 條件為 FALSE → 這筆被排除。
   - 因為名單裡「混進 NULL」，導致判斷變成 UNKNOWN → WHERE 不會選這筆。

<br>

4. WHERE 只留下條件結果為 TRUE 的列
   - FALSE、UNKNOWN 都被排掉。
   - 所以只要子查詢名單裡出現 NULL，整個 NOT IN 的邏輯就可能被「拉成 UNKNOWN」，結果就是：你預期會被選出來的資料，全部消失。



![v](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Anti-Join/2___not__in__intro.png)


## 🔹 NULL 與 NOT IN 的邏輯陷阱

一個 NULL 就足以讓整個 NOT IN 的判斷從「肯定」變成「我不知道」，最後變成「全部不要」。因為 NOT IN 是在跟一張「可能藏著陷阱的名單」比對，只要那張名單裡混進一個 NULL，整個判斷就會失靈，如果 Attendance.FriendID 裡出現了 NULL，整個查詢結果可能會變「空集合」！因為 NOT IN 碰到 白名單 的 KEY 有 NULL，結果會變成「不確定」，最後導致 所有朋友都不會被選出來

子查詢結果如果長這樣
```bash
Attendance.FriendID
-------------------
101
102
NULL
```

<br>

當我們判斷： WHERE FriendID NOT IN (101, 102, NULL)，對任何一個 FriendID = X，實際邏輯是 X 不等於 101 AND X 不等於 102 AND X 不等於 NULL，X <> NULL 這個判斷本身就是 UNKNOWN，而整個 AND 鏈只要有一個 UNKNOWN，整體就不是 TRUE，只會是 FALSE 或 UNKNOWN。結果是沒有任何一筆 row 能得到 TRUE，整個結果變空集合


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Anti-Join/2___not_in__null_trap.png)


**測試方法**
```SQL
INSERT INTO Attendance VALUES (103, NULL);

-- 再跑剛剛的查詢
SELECT FriendID, FriendName
FROM Friends
WHERE FriendID NOT IN (SELECT FriendID FROM Attendance);
```

**實務技巧**
```SQL
WHERE FriendID NOT IN (
    SELECT FriendID FROM Attendance WHERE FriendID IS NOT NULL
);
```


![k](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Anti-Join/2___not__in__fix.png)



## 🔹 關於資料建模：為什麼會有 NULL？

很多 NOT IN 的災難，不是寫法的錯，而是「這個欄位本來就不應該允許 NULL 卻被放進來了」，要避免這種坑，NOT NULL 的設定決策很重要，一般來說會設定好 FK 到 Friends(FriendID) NOT NULL，甚至直接禁止插入沒有 FriendID 的出席紀錄

<br>

## 🔹 效能

在合理的索引與條件下，會被轉換成 Anti Semi Join，也就是一種『只取左表有、右表沒有』的最佳化操作。當右表（Attendance）有索引時，效能很好

- `Semi Join`：左表只要在右表「有對應」就留下來（`EXISTS`、`IN`）
- `Anti Semi Join`：左表只要在右表「找不到對應」就留下來（`NOT EXISTS`、`NOT IN`）
- `DB Optimizer` 通常會把這些語法轉成類似的執行計畫，效能差距未必想像中那麼大

如果 Attendance.FriendID 有 index，每次比對某個 FriendID，只要在 index 裡快速查「有沒有這個值」。大量資料時會差很多，沒有 index 時，很可能變成「一邊掃主表、一邊全表掃子查詢」，一整個 N × M 效能會炸裂

<!-- endtab -->

<!-- tab NOT EXISTS-->

對於每個朋友，問一次「你有出現在 Attendance 嗎？」如果沒有，就被挑出來

```SQL
SELECT f.FriendID, f.FriendName
FROM Friends f
WHERE NOT EXISTS (
    SELECT 1
    FROM Attendance a
    WHERE a.FriendID = f.FriendID
);
```

- 掃描左表 Friends：逐筆讀取 FriendID。
- 利用索引（如果有的話），快速查 Attendance 是否存在相同 FriendID。
- 找到第一筆匹配就停止檢查（效率比 NOT IN 高）。
- 決策：
  - 如果右表存在 → EXISTS = TRUE → 這筆不選。
  - 如果右表不存在 → NOT EXISTS = TRUE → 這筆選出來。

不需要完整掃描右表，只檢查「是否存在」。有索引時效能最佳。不會受 NULL 影響，結果永遠正確。


![cc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Anti-Join/3___not__exist__intro.png)


## 🔹 不受 NULL 影響

NOT IN 可能因為右表有 NULL 而導致整個查詢「不回任何資料」。但 NOT EXISTS 完全不在乎 NULL，因為它只是檢查「有沒有匹配行」，在 SQL Server、Oracle、PostgreSQL 等主流 DB，NOT EXISTS 通常會被轉換成 Anti Semi Join。當右表有索引（例如 Attendance.FriendID 上有 Index），效能幾乎跟 NOT IN 一樣，甚至更穩定

<br>

## 🔹 效能

在有 Index 的情況下，與 NOT IN 幾乎相同；在無 Index 的情況下，兩者都可能慢

<!-- endtab -->

<!-- tab LEFTJOIN... IS NULL-->

就是找「左表有，但右表沒有」的資料

```SQL
SELECT f.FriendID, f.FriendName
FROM Friends f
LEFT JOIN Attendance a
    ON f.FriendID = a.FriendID
WHERE a.FriendID IS NULL;
```

<br>

- 把 Friends 全部取出，跟 Attendance 嘗試做匹配。
  - 如果有匹配 → 把 Attendance 的欄位帶出來。
  - 如果沒有 → Attendance 欄位設成 NULL。


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Anti-Join/4___left_join.png)


>（這裡可能產生很大的「中間結果集」）


- 過濾條件，保留那些 a.FriendID IS NULL 的紀錄。也就是「左表有但右表沒有對應」，語意直觀，但中間結果集可能龐大。如果右表很大且沒有索引 → 效能可能比 NOT EXISTS 差。在現代 DBMS，最佳化器有時會把它重寫成 Anti Semi Join，效能差異不大。


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Anti-Join/4__left_join__cost.png)


## 🔹 重複值的處理

如果右表 (Attendance) 有重複紀錄，LEFT JOIN 會把左表的資料「複製」多次，雖然最後 IS NULL 可以過濾掉，但中間結果可能會比 NOT EXISTS 更大，導致效能下降，LEFT JOIN … IS NULL 通常會被最佳化器轉換成 Anti Semi Join，效能跟 NOT EXISTS 幾乎一樣，但當 右表很大、且無索引 時，LEFT JOIN 在中間產生的資料量會比 NOT EXISTS 多，效能可能會差一點

<br>

## 🔹 中間結果集問題

先把 左表 (Friends) 和 右表 (Attendance) 做一次 Join。即使最後要的是「右表沒對上的資料」，但在 執行計畫 裡，資料庫還是會產生「合併後的結果集 (Intermediate Result Set)」，如果右表資料很大，這個中間結果集可能會非常龐大，再加上 IS NULL 過濾，效能就會下降，LEFT JOIN 是「先全都比對，再篩掉不要的」

<br>

## 🔹 想要帶出「右表的一些欄位」

當你不只是要排除，而是想找出所有朋友，以及他們的出席紀錄（有來顯示，沒來就 NULL）

```SQL
SELECT f.FriendID, f.FriendName, a.AttendID
FROM Friends f
LEFT JOIN Attendance a
    ON f.FriendID = a.FriendID;
```


![bb](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Anti-Join/4___left_join_risk2.png)


<!-- endtab -->

<!-- tab EXCEPT-->

EXCEPT 在 SQL 裡表示「集合差集」：取左邊查詢的結果集合減去右邊查詢的結果集合


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Anti-Join/5___except__1.png)


```SQL
SELECT FriendID, FriendName
FROM Friends

EXCEPT

SELECT f.FriendID, f.FriendName
FROM Friends f
INNER JOIN Attendance a
    ON f.FriendID = a.FriendID;

-- 或你只需要 Id
SELECT FriendID FROM Friends
EXCEPT
SELECT FriendID FROM Attendance;
```

- 左邊結果集算出來（Friends）。
- 右邊結果集算出來（Attendance）。
- 做一次「去重 (DISTINCT) + 排序 (Sort)」。
- 算出「差集」。

👉 所以兩張表都會被完整掃描一次。





## 🔹 天生去重

EXCEPT 天生會幫你做去重 (DISTINCT)。如果 Friends 或 Attendance 有重複值，EXCEPT 會自動去掉
預設去重可能造成效能問題，EXCEPT 在內部會做 DISTINCT，等於要排序 + 去重。
如果你不需要去重，這個動作就是多餘的。在資料量大的時候，這會比 NOT EXISTS / LEFT JOIN ... IS NULL 多一些效能開銷

<br>

## 🔹 處理 NULL

在 SQL 標準裡，EXCEPT 會正確處理 NULL，不像 NOT IN 那樣踩陷阱


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Anti-Join/5___except__2.png)


<!-- endtab -->

<!-- tab 測試方法-->


![v](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Anti-Join/6___index_is_key_semi_optimize.png)


想要比較 Anti Join 語法的效能時，我們可以開啟 SQL Server 的統計資訊來觀察查詢效能。

```SQL
SET STATISTICS TIME ON;  -- 顯示 CPU Time 與 Elapsed Time
SET STATISTICS IO ON;    -- 顯示 IO 使用情況（邏輯讀取次數）

```

接著在不同的查詢之間用 GO 區隔，這樣每個查詢的統計結果會分開顯示，方便觀察：
```SQL
-- NOT EXISTS
SELECT ...
GO

-- NOT IN
SELECT ...
GO

-- LEFT JOIN ... IS NULL
SELECT ...
GO

-- EXCEPT
SELECT ...
GO
```

<br>

**CPU Time** : 代表 SQL Server 在這次查詢中，真正花費在 CPU 運算 的時間。通常越小越好，表示 CPU 負擔較低。
**Elapsed Time** : 代表查詢從開始到結束，實際花費的總時間。這個值會受到 CPU、IO、網路、記憶體等多種因素影響。
**Logical Reads** :代表 SQL Server 讀取了多少資料頁 (Page)，也就是 查詢需要存取多少次資料。越少越好，因為表示這個查詢需要讀取的資料較少，IO 負擔較輕。
**執行計畫 (Execution Plan)** :　在 SSMS (SQL Server Management Studio) 中，可以點選「Include Actual Execution Plan」(快捷鍵：Ctrl+M)，或是執行前按下工具列的執行計畫按鈕。執行計畫會顯示 SQL Server 實際使用了哪種演算法（例如 Nested Loop Anti Semi Join、Hash Match Anti Semi Join、Merge Join…），這能幫助我們理解資料庫在不同語法下的最佳化差異


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Anti-Join/7___statistics.png)


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Anti-Join/8___decision_table.png)


![as](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Anti-Join/9___hint.png)


<!-- endtab -->

{% endtabs %}