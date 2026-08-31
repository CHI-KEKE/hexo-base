---
title: STRING_AGG
date: 2024-06-08 11:49:34
categories: 情報の存在論
top_img: https://i.imgur.com/NfCnwwU.png
cover : https://i.imgur.com/NfCnwwU.png
tags:
    - 情報の存在論
toc:
toc_number:
comments :
---

{% tabs STRING_AGG %}


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/STRING_AGG/1___landing__image.png)


<!-- tab 一生中有三大難題-->

☘️ 人的一生中有三大難題：

- 中午吃什麼?
- 明天要不要請假？
- 怎麼把一堆資料拼成一句話？

前兩個我還在想，第三個已經找到解答了—— STRING_AGG

<br>
<br>

他常用來作為文字串接的聚合函數，官方定義是這樣子的 :
用於把 <b>"指定的欄位"</b> 串成一個 <b>"以指定分隔符分隔的字符串"</b>，不用寫一堆複雜的迴圈或自製拼接邏輯


<br>

```SQL

STRING_AGG ( expression, separator ) [ <order_clause> ]

<order_clause> ::=   
    WITHIN GROUP ( ORDER BY <order_by_expression_list> [ ASC | DESC ] )   

```


![aa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/STRING_AGG/2___basic.png)


每天上班打開公司群組，就像打開一個沒人整理過的資料表，一堆廢話、貼圖、+1、讚、下午茶，語焉不詳的訊息而零散像剛剛爆炸過。有時真的很想把他們 STRING_AGG 起來丟到任何看不見的地方


<!-- endtab -->

<!-- tab 🧪 實戰-->

「那個誰，幫我把這個資料表裡的這些串在一起」光頭手上揮舞著資料像在招魂一樣，語音剛落

![Image](https://i.imgur.com/eI4dKBJ.png)

```SQL
SELECT 
    OrderID,
    STRING_AGG(ProductName, ', ') AS ProductList
FROM 
    OrderDetails
GROUP BY 
    OrderID;
```

![ff](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/STRING_AGG/3__puzzle__solution.png)


ok 感覺不戳，資料寄出去趕快下班!


ﾍ( ﾟ∀ﾟ;)ﾉ
ﾍ( ﾟ∀ﾟ;)ﾉ
ﾍ( ﾟ∀ﾟ;)ﾉ

隔天...

ﾚ(ﾟ∀ﾟ;)ﾍ
ﾚ(ﾟ∀ﾟ;)ﾍ
ﾚ(ﾟ∀ﾟ;)ﾍ


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/STRING_AGG/4___puzzle_order.png)


```SQL

SELECT 
    OrderID,
    STRING_AGG(ProductName, ', ')  WITHIN GROUP (ORDER BY ProductId) AS ProductList
FROM 
    OrderDetails
GROUP BY 
    OrderID;

```
WITHIN GROUP 派上用場，有點像 Window Function


![cc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/STRING_AGG/4__puzzle__solution.png)



老闆 : 「欸..有一件事要順便一下」

(喔? 順便什麼? 發獎金給我嗎?)

老闆打開另一個視窗，秀出另外一組資料

老闆 : 「你看... 這些簡訊發出去的號碼都是哪些人啊?」


![cc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/STRING_AGG/5___puzzle__number_agg.png)


```SQL

DECLARE @SmsMessageCellphones NVARCHAR(MAX);
DROP TABLE IF EXISTS #MemberData_0606;

-- 需要Cellphone List 撈取跨伺服器 Member Data
SELECT @SmsMessageCellphones = STRING_AGG(CAST(JSON_VALUE(Task_Data, '$.PhoneNumber') AS NVARCHAR(MAX)), ',')
FROM Task
WHERE Task_ValidFlag = 1
AND Task_JobId = 250
AND Task_BookingTime > '2024-05-05 00:00:00'
AND Task_BookingTime < '2024-06-07 00:00:00'
AND Task_Status = 'Switched'
AND JSON_VALUE(Task_Data, '$.SMSType') = N'會員註冊';

-- 跨伺服器撈取 Member 資料
INSERT INTO #MemberData_0606
EXECUTE WEBSTOREROLS.WebStoreDB.dbo.sp_executesql N'
SELECT MemberRegister_ShopId,
       MemberRegister_CellPhone
FROM dbo.MemberRegister(NOLOCK)
WHERE MemberRegister_CellPhone IN (SELECT CAST(value AS nvarchar(MAX)) FROM STRING_SPLIT(@SmsMessageCellphones, '',''))
AND MemberRegister_ValidFlag = 1
',N'@SmsMessageCellphones NVARCHAR(MAX)', @SmsMessageCellphones = @SmsMessageCellphones;

```


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/STRING_AGG/5___solution__step1.png)


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/STRING_AGG/5___solution__step2.png)


<!-- endtab -->

<!-- tab 💩-->

工作中，我們總是在採坑，不如說，工作本身就是一個坑，工作我們實在避不掉，但 SQL 的坑還是能少踩一點是一點

<br>

## ☑️ 空值處理

如果欄位可能會是 NULL，那這些值會被自動忽略。想控制一下，可以這樣做

```SQL

SELECT STRING_AGG(ISNULL(Department, 'N/A'), ', ') AS AllDepartments
FROM Employees;

```


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/STRING_AGG/6___trap2__null.png)



## ☑️ 長度限制

拼接起來的字串可能超過欄位或變數最大容量，建議加上 LEN() 檢查，或依需求拆批處理，避免過長爆掉


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/STRING_AGG/6___trap__length.png)



## ☑️ 應用到子查詢或 CTE

可以把 STRING_AGG 包進 CTE 或子查詢裡做更進一步邏輯處理。例如，在一個產品銷售資料表中，將每個地區的銷售員名字串接為一個字串，然後再分成串上地區資訊

```SQL

WITH RegionAggregates AS (
    SELECT 
        Region, 
        STRING_AGG(Salesperson, ', ') AS Salespersons
    FROM Sales
    GROUP BY Region
)
SELECT STRING_AGG(Region + ': ' + Salespersons, '; ') AS Summary
FROM RegionAggregates;

```


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/STRING_AGG/7___tech_cte.png)


<!-- endtab -->

<!-- tab 🐛 結語-->


![ccc](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/SQL/STRING_AGG/unnamed%20(1).png)


<font color=#D3D3D3 style="font-size: 20px;">&emsp;&emsp;&emsp;&emsp;&emsp;如果人生也能用 `STRING_AGG` 把快樂的記憶串起來
 </font>

<font color=#D3D3D3 style="font-size: 20px;">&emsp;&emsp;&emsp;&emsp;&emsp;那請記得在最後加上：`WITHIN GROUP (ORDER BY 心情 DESC)`
 </font>

<!-- endtab -->

{% endtabs %}