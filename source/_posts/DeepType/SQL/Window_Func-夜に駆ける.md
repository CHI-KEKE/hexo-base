---
title: 在夜裡奔跑的資料，不需要回頭 JOIN
date: 2025-10-06 22:52:05
categories: 情報の存在論
top_img: https://i.imgur.com/NfCnwwU.png
cover : https://i.imgur.com/NfCnwwU.png
tags:
    - 情報の存在論
toc:
toc_number:
comments :
---

{% tabs MAXGROUP_TO_WINDOW %}

<!-- tab 夜に駆ける-->

《夜に駆ける》- 不是回頭統計「哪一天最痛」，而是在情緒奔跑的過程中，痛苦自然浮到最前面

MAX + JOIN 的人生像在事後寫失戀報告，假設你用 MAX + GROUP BY + JOIN 來寫這首歌，列出所有相處的日子、快樂指數、痛苦指數、聊天次數，GROUP BY「每一天」並算出「痛苦指數 MAX 的那一天」再回頭 JOIN 那一天的對話、場景、畫面最後下結論「原來是那一天最痛。」

- 痛苦 = MAX
- 回憶 = JOIN
- 情緒 = 被拆成資料再拼回來

這會寫成一篇 Excel 報告，不會是一首歌


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Window/1__song_not_report.png)



而 Window Function 實際在做的是，情緒一路推進，畫面一路疊加，痛苦不斷「排序往前」

「我在跑的時候，就知道逃不掉。」

```sql
ROW_NUMBER() OVER (
  PARTITION BY 這段關係
  ORDER BY 情緒崩壞程度 DESC
)
```

![INTO_THE_NIGHT](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/YOASOBI/yosobi_into_the_night.png)

<!-- endtab -->

<!-- tab 先別急著 GROUP BY-->

## MAX => GROUPBY

```sql
SELECT o.*
FROM Orders o
JOIN (
    SELECT CustomerId, MAX(OrderDate) AS MaxDate
    FROM Orders
    GROUP BY CustomerId
) latest ON o.CustomerId = latest.CustomerId
        AND o.OrderDate = latest.MaxDate
```

先不談效能，只看「這個寫法為什麼會出現」的話，需求本身其實很單純「我要每個 CustomerId 最新的一筆訂單」

這時候考慮使用 GROUPBY 會發現，有一個硬限制，你只能選 GROUP BY 欄位聚合結果（MAX / SUM / COUNT），不能同時拿到那一整列的其他欄位


![kk](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Window/1___groupby_lost_data.png)


所以第一步只能退而求其次：先算答案，每個 CustomerId 的「最大 OrderDate」，這一步只是在回答最新日期是什麼？

但我們要的是 OrderId、Amount、Status，這些在 GROUP BY 後全部消失了，因此被迫做第二步：JOIN 回原始表，用「剛算出的答案」回去找「符合這個答案的那一筆資料」，造成整體流程是先算出正確答案，再用答案去反推資料


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Window/1___groupby__backtodata.png)


所以其實是在用 GROUP BY 做一件它不擅長的事 (╥﹏╥)

![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Window/1__why__getbackdata__slow.png)


<!-- endtab -->

<!-- tab Window Rank-->

Window Function 的本質是「在不破壞資料列的前提下，直接在資料流中計算分組結果」，而不是算完再回頭補資料

```sql
SELECT *
FROM (
    SELECT *,
           RANK() OVER (PARTITION BY CustomerId ORDER BY OrderDate DESC) AS RankNo
    FROM Orders
) ranked
WHERE RankNo = 1
```

![oo](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Window/2__run__not__escape.pn)


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Window/2___define__the__window__scope_partion.png)


資料表只被掃描一次，沒有先縮資料、也沒有產生中間結果表，每一列資料都還「活著」，PARTITION BY 不是 GROUP BY，排序與計算在同一個資料流中完成，ORDER BY 定義分組內的順序，RANK 在排序過程中即時計算，不需要「算完再比對」，最後只做一次條件篩選，沒有任何資料回查或比對行為


![aa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Window/2__window__pipeline.png)


- 少一次掃描
- 少一次 JOIN
- 少一個中間結果集
- 更容易順著索引走

| 面向             | 傳統 GROUP BY + JOIN         | Window Function          |
| -------------- | -------------------------- | ------------------------ |
| 資料掃描次數         | 至少 2 次 (聚合、回連)             | 1 次                      |
| 需排序或 Hash      | Hash Aggregate + Hash Join | 單一排序（可用索引）               |
| 中間結果大小         | 聚合結果 + JOIN 結果             | 直接輸出                     |
| ties 處理        | 需額外 DISTINCT 或 TOP         | 可直接控制（ROW_NUMBER / RANK） |
| Optimizer 可調度性 | 較複雜                        | 更可預測、記憶體使用穩定             |


![z](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Window/2___window___why__better.png)


![aa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Window/2__window__ties.png)


<!-- endtab -->


<!-- tabs Window - AVG-->

```sql
SELECT emp_id, emp_name
FROM (
  SELECT emp_id,
         emp_name,
         salary,
         AVG(salary) OVER (PARTITION BY dept_id) AS dept_average
  FROM employee
) e
WHERE salary > dept_average;
```

「在掃描資料的過程中，順便把部門平均薪資貼在每一列上」通常是一次 Scan、一次 Window Aggregate，沒有額外的 JOIN，沒有中間結果集需要再對齊，你可以把它理解成：每個員工在排隊時 HR 一邊看名單，一邊在他旁邊寫上「你們部門平均是 62,000」

![gg](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Window/3__hr__salary.png)


| 面向           | GROUP BY + JOIN | Window Function |
| ------------ | --------------- | --------------- |
| 掃描次數         | 2 次（或更多）        | 通常 1 次          |
| JOIN 成本      | 有               | ❌ 無             |
| 記憶體壓力        | 較高              | 較低              |
| 可讀性          | 中               | ✅ 高             |
| 延展性（加欄位）     | ❌ 要改 GROUP BY   | ✅ 很容易           |
| 查 Top / Rank | ❌ 很麻煩           | ✅ 天生適合          |



![cc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Window/3__optimizer.png)


<!-- endtab -->


<!-- tabs summ-->


![xx](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Window/unnamed.png)


<!-- endtab -->


{% endtabs %}