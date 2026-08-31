---
title: Pre-filtering vs. Post-filtering
date: 2025-12-10 08:13:34
categories: 情報の存在論
top_img: https://i.imgur.com/NfCnwwU.png
cover : https://i.imgur.com/NfCnwwU.png
tags:
    - 情報の存在論
toc:
toc_number:
comments :
---


{% tabs Pre-filtering vs. Post-filtering %}

<!-- tab 關於訂餐-->

午休前的辦公室接近午休時間。Team leader 桌子一拍：「各位肥宅！麥當勞開團囉！」

Jacky 抬頭：「請客嗎？」
Leader 停了一秒：「呃…好，可以。」

話才剛落音，幾個原本裝忙的同事像被觸發開關似地站起來，開始熱烈點餐：

- 「大薯 + 可樂，搭配麥脆雞腿堡。」
- 「大麥克加一份生菜，再配玉米濃湯。」
- 「精緻牛排。」
- 「鱈魚堡微糖少冰，搭配雪碧。」

……
……
……

Leader 看著訊息流越滾越誇張，只能扶額：「靠…我是不是講太大聲了？剛剛是不是還混進奇怪的東西？」


![Mac](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/mac.png)


外送到後，大家開始大快朵頤，享受難得的免費午餐，而辦公室的一角，Allen 才剛把耳機拿下，看著眾人吃得開心，因為工作太專注，他完全錯過開團訊息


另一邊，也有同事沒發聲，只因為不喜歡炸物，於是默默提著自己的 Subway 回到座位



![ghj](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pre-Post-Filtering/1____allen_and_lily_is_missing_scenerio.png)



以資料表的角度來看，這裡會有兩張表

- Employee（員工資料）
- Order（訂餐資料）

現在我們想查的是哪些員工有參與「活動」（例如平常都跟團），但沒有參與這次的麥當勞訂購


![A](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pre-Post-Filtering/1____allen_and_lily_is_missing_dats_model.png)


```SQL
FROM Employees e
LEFT JOIN Orders o
  ON  e.Id = o.EmployeeId
  AND o.Brand = 'McDonalds'   -- 放在 ON


-- | 員工    | 品牌        | 餐點   |
-- | ----- | --------- | ---- |
-- | Bill | McDonalds | 麥脆雞  |
-- | Allen    | NULL      | NULL |
-- | Jacky    | McDonalds      | 鱈魚堡 |
-- | Lily    | Null      | Null |
```

我們因此可以藉此保留更完整的資料，但同時達到篩選特定條件的效果，看是要研究有的人可能不吃炸的，有的人可能素食偏好，而作為管理者可能可以多多關心這些人!

![aaa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pre-Post-Filtering/3___prefiltering__fix.png)


<!-- endtab -->


<!-- tab LEFT JOIN 事後過濾-->

```sql
-- A：條件放在 WHERE（事後過濾）
select a.k ak, b.k bk
from tbl1 a
left outer join tbl2 b
  on a.k = b.k
where b.ds = '20210101'; -- 這行會把 b 為 NULL 的列都濾掉 → 變相 inner join
```

沒匹配到的列，b 全部是 NULL，接著 WHERE b.ds = '20210101' → 因為 NULL 不符合條件，被濾掉
結果就會變成 只剩匹配到的列（像 INNER JOIN）


| a.k | b.k |
| --- | --- |
| 1   | 1   |


![aaa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pre-Post-Filtering/2___post__filtering__trap.png)


![kkk](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pre-Post-Filtering/2___allen___is__null__and__throw.png)


<!-- endtab -->

<!-- tab 📜 SQL 執行順序-->

概念上，他涉及 SQL 的執行順序，當我們下語法時，他會依據這個規則來執行

```plaintext
FROM → JOIN (ON) → WHERE → GROUP BY → HAVING → SELECT → ORDER BY
```

而 ON 是 JOIN 的階段 用來「決定怎麼連」，WHERE 是 JOIN 完之後才「過濾結果」


![aaa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pre-Post-Filtering/3___where__after__join.png)


<!-- endtab -->

<!-- tab LEFT JOIN 前置過濾-->


```sql
-- 寫法 1：條件放 ON
select a.k ak, b.k bk
from tbl1 a
left outer join (
  select k from tbl2 where ds = '20210101'
) b
  on a.k = b.k; -- 仍然是 left join 的語意（a 全保留）

-- 寫法 2：條件放右表子查詢
SELECT a.k, b.k
FROM tbl1 a
LEFT JOIN (
  SELECT k FROM tbl2 WHERE ds = '20210101'
) b
  ON a.k = b.k;

-- 寫法 3：但這其實就是在 WHERE 裡手動補回 NULL，語意不如放 ON/子查詢 直觀!
select a.k ak, b.k bk
from tbl1 a
left outer join tbl2 b
  on a.k = b.k
WHERE (b.ds = '20210101' OR b.ds IS NULL)；
```

| a.k | b.k  |
| --- | ---- |
| 1   | 1    |
| 2   | NULL |
| 3   | NULL |


把條件放在 ON 或右表子查詢，等於「先把要 JOIN 的右表縮小」，再做 LEFT JOIN → 能保留左表 a 的所有列，只是 b 只有符合條件的才連上，否則為 NULL


![cc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pre-Post-Filtering/4____subquery.png)


![cc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pre-Post-Filtering/4___why___subquery__might_be_better.png)


<!-- endtab -->



<!-- tab INNER JOIN-->

在 INNER JOIN 時，把條件放在 ON 或最後的 WHERE，結果完全一樣，效能幾乎也一樣，因為 INNER JOIN 本質上會把不符合條件的資料在 JOIN 階段就丟掉，WHERE 再次篩掉的是 JOIN 完後的資料，但因為 INNER JOIN 的語意，兩者的資料集合其實一致

```sql
--🅰 放在 ON
SELECT *
FROM a
JOIN b
  ON a.k = b.k
 AND b.type = 'X';

--🅱 放在 WHERE
SELECT *
FROM a
JOIN b
  ON a.k = b.k
WHERE b.type = 'X';
```

在這個情境下，兩段 SQL 是等價的，因為 INNER JOIN 只會留下成功連上的資料，b.type = 'X' 不管放在哪，都只會留下「符合條件且成功連接」的 b，而效能其實 99% 的情況沒有差因為 Optimizer 會做 Predicate Pushdown、Join Simplification，Filter Reordering，讓執行計畫一致!


![z](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pre-Post-Filtering/5___innerjoin__is_same.png)


<!-- endtab -->


<!-- tab summary-->


![cc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/Pre-Post-Filtering/6___diff__trable.png)


<!-- endtab -->

{% endtabs %}