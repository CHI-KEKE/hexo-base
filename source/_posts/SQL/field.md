---
title: Field
date: 2025-10-13 22:52:05
categories: 
top_img: https://i.imgur.com/NfCnwwU.png
cover : https://i.imgur.com/NfCnwwU.png
tags:
    - 
toc:
toc_number:
comments :
---


![SQL](https://i.imgur.com/NfCnwwU.png)

<br>

在 SQL 資料表設計時，「欄位型別的選擇」會直接影響到：

- 儲存空間（Database Size）
- 查詢效能（Index / Join / Sort）
- 可擴充性（是否足夠表示未來的資料量）

## 💼 char / varchar / nchar / nvarchar


「var」表示變長，「n」表示支援 Unicode

<br>

### 🔹 定長：char

想像你開一間餐廳，每張桌子都是固定 10 個座位 (char(10))，不管今天坐 2 個人還是 10 個人，桌子都占同樣的空間
也就是說，你設定 char(10)，即使只放 'Hi'，它還是佔 10 Bytes，系統會幫你在後面補空白（space），讓長度固定

char 因為長度固定，SQL Server 可以「快速計算位置」，所以在大量查詢時效能較好。所以 char 比較「浪費空間但速度快」；varchar 比較「省空間但速度略慢一點」

<br>

### 🔹 Unicode vs. 非Unicode（n 開頭的型別）

Unicode 是一種「萬國通用的文字編碼」，用來同時支援 中、英、日、韓、泰文 等不同語系，如果你用 varchar 來存中文，可能會變亂碼，因為 varchar 會根據語系（collation）去解讀每個字佔幾個 Byte。
但 nvarchar 則是固定每個字都用 2 Bytes 儲存，所以再也不會亂碼


```SQL
CREATE TABLE RocketUnicode (
    rocket1 VARCHAR(50),
    rocket2 NVARCHAR(50)
);

INSERT INTO RocketUnicode VALUES ('我愛TsungHua', N'我愛TsungHua');
```

| 欄位      | 儲存方式                                                   | 說明     |
| ------- | ------------------------------------------------------ | ------ |
| rocket1 | 會把中文「我愛」當成 4 Bytes、英文「TsungHua」當成 8 Bytes → 共 12 Bytes | 容易亂碼   |
| rocket2 | 每個字固定 2 Bytes，共 10 個字 = 20 Bytes                       | 安全、不亂碼 |

<br>

### 🔹 實際使用建議

| 型別            | 長度特性 | 語言特性 | 適用情境              |
| ------------- | ---- | ---- | ----------------- |
| `char(n)`     | 固定長度 | 僅英文  | 會員編號、郵遞區號、固定格式代碼  |
| `nchar(n)`    | 固定長度 | 多語系  | 固定長度但可能包含中文的代碼    |
| `varchar(n)`  | 可變長度 | 僅英文  | Email、網址、帳號等不含中文  |
| `nvarchar(n)` | 可變長度 | 多語系  | 姓名、地址、留言內容等含中文的資料 |

<br>

### 🔹 容量限制與差異

| 型別            | 最大長度   | 儲存大小          | 特性         |
| ------------- | ------ | ------------- | ---------- |
| `char(n)`     | 1～8000 | 固定 n Bytes    | 定長、英文最佳化   |
| `nchar(n)`    | 1～4000 | 固定 2n Bytes   | 定長、Unicode |
| `varchar(n)`  | 1～8000 | 依內容而定         | 變長、英文最佳化   |
| `nvarchar(n)` | 1～4000 | 依內容 * 2 Bytes | 變長、Unicode |
| `text`        | ~2GB   | 可變            | 舊型別（不建議再用） |
| `ntext`       | ~1GB   | 可變            | 舊型別（不建議再用） |


### 🔹 nvarchar(max)

nvarchar(max) 最多可儲存 1,073,741,823 個字元，也就是大約 2GB 的資料量。(n) 這個數字的「單位」代表 字元（characters），
而不是 Bytes。你放中文或英文，都算 1 個「字元」；只是底層儲存時會佔用 2 Bytes。


<br>
<br>

## 💼 bigint、guid、int、tinyint、bit、bytes

<br>

### 🔹 概念總覽

| 型別                           | 儲存大小                                       | 數值範圍                                                   | 是否可排序       | 常用用途            | 備註            |
| ---------------------------- | ------------------------------------------ | ------------------------------------------------------ | ----------- | --------------- | ------------- |
| `bit`                        | 1 bit (但實際上 SQL Server 會以 1 byte 儲存多個 bit) | 0 或 1                                                  | ✅ 是         | 布林值（true/false） | 儲存空間最小        |
| `tinyint`                    | 1 byte                                     | 0 ～ 255                                                | ✅ 是         | 小範圍的代號、狀態       | 最小的整數型別       |
| `int`                        | 4 bytes                                    | -2,147,483,648 ～ 2,147,483,647                         | ✅ 是         | 主鍵 ID、一般數值欄位    | 預設最常用         |
| `bigint`                     | 8 bytes                                    | -9,223,372,036,854,775,808 ～ 9,223,372,036,854,775,807 | ✅ 是         | 超大流水號、Log ID    | 比 `int` 多一倍空間 |
| `uniqueidentifier` (GUID)    | 16 bytes                                   | 不適用                                                    | ✅ 是（但排序成本高） | 全域唯一識別碼（UUID）   | 適用分散式系統       |
| `varbinary(n)` / `binary(n)` | 1～8000 bytes                               | 二進位資料                                                  | ❌ 不排序       | 圖片、檔案、雜湊值       | 不可直接比較內容      |

<br>

### 🔹 bit —— 真或假（Boolean）

📦 儲存大小： 最小單位（1 bit，實際儲存時 SQL Server 會一組打包）
📊 用途： 用來儲存「是/否」、「開啟/關閉」、「有效/無效」等狀態

```sql
CREATE TABLE Users (
    IsActive BIT
);

INSERT INTO Users VALUES (1);  -- true
INSERT INTO Users VALUES (0);  -- false
```

<br>

### 🔹 tinyint —— 小範圍的數字

📦 儲存大小： 1 byte
📊 可儲存範圍： 0 ～ 255

```sql
CREATE TABLE Product (
    Status TINYINT
);

-- 例如：
-- 0 = 下架
-- 1 = 上架
-- 2 = 預購
```

像是「等級」或「狀態代碼」的小盒子，只能裝少量數字。建議例如狀態、等級、分類代號、分數等有限範圍數值

<br>

### 🔹 int —— 標準整數

📦 儲存大小： 4 bytes
📊 範圍： -2,147,483,648 ～ 2,147,483,647

```sql
CREATE TABLE Orders (
    OrderId INT IDENTITY(1,1) PRIMARY KEY
);
```

<br>

### 🔹 int —— 標準整數

📦 儲存大小： 4 bytes
📊 範圍： -2,147,483,648 ～ 2,147,483,647

```sql
CREATE TABLE Orders (
    OrderId INT IDENTITY(1,1) PRIMARY KEY
);
```

就像一個能裝「幾十億」的數字倉庫，容量夠大又效率高。這是大多數 ID 欄位的「黃金型別」。
建議使用情境包含主鍵流水號 (Identity)、數量、年齡、分數等一般整數

<br>

### 🔹 bigint —— 超大號整數

📦 儲存大小： 8 bytes（是 int 的兩倍）
📊 範圍： 超級大（約 ±9 × 10¹⁸）

```sql
CREATE TABLE Logs (
    LogId BIGINT IDENTITY(1,1)
);
```
就像一個「宇宙級倉庫」，能放下全球用戶的 ID。比 int 佔空間兩倍，但能撐更久。
建議極大量資料表（例如 Log、IoT 資料、交易紀錄）、分散式系統 ID，不想重複就選它