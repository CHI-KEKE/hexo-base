---
title: IsolationLevel
date: 2025-10-11 08:50:05
categories: 
top_img: https://i.imgur.com/NfCnwwU.png
cover : https://i.imgur.com/NfCnwwU.png
tags:
    - 
toc:
toc_number:
comments :
---


![ocean](https://i.imgur.com/NfCnwwU.png)

交易隔離層級是用來控制「當多個交易同時讀寫資料時，要不要讓他們互相看到對方的變更。要資料「越準確」，就得「越鎖」，要系統「越快」，就得「越鬆」，是在討論競爭條件(Race condition)的議題

<br>

## 💼 READ UNCOMMITTED

允許交易去「偷看別人還沒寫完的資料」，這樣雖然速度快，但有風險，因為你讀到的資料可能會「被改回去」或「消失」

- 不會加共用鎖 (Shared Lock)，所以別人正在改資料時，你一樣能讀它。
- 不會被排他鎖 (Exclusive Lock) 阻擋，所以就算別人正在修改資料、還沒提交，你也能去讀它。

<br>

### 🟢 Shared Lock（共用鎖）

是 SQL Server 在「讀資料」時加上的一種鎖，它的作用是我正在看資料，別人在我看完之前不能改它
觸發時機：SELECT 查詢（讀資料），可以多個共存 Shared Lock 但會阻擋 Exclusive Lock（別人不能改）

用途：確保你讀的資料在讀取過程中不會被修改

```sql
BEGIN TRAN;
SELECT * FROM Products WHERE Id = 1;  -- Shared Lock
-- 不結束交易，鎖會持續存在

UPDATE Products SET Price = 200 WHERE Id = 1; -- 會被卡住
```

預設情況下，EF 查詢會有 Shared Lock。因為 SQL Server 的預設隔離層級就是 READ COMMITTED。而 READ COMMITTED 的行為 = 查資料時會加 Shared Lock（直到讀完那筆資料）

<br>

### 🔴 Exclusive Lock（排他鎖）

用在「改資料」。別人不能看、也不能改，觸發時機是 INSERT、UPDATE、DELETE，完全不能共存（別人連讀都不行），會阻擋任何 Shared 或 Exclusive Lock，用途是確保修改時不會被別人讀或改，避免資料不一致
```sql
BEGIN TRAN;
UPDATE Products SET Price = 300 WHERE Id = 1;  -- Exclusive Lock
SELECT * FROM Products WHERE Id = 1; -- 就會被卡住，因為資料正在被修改中。
```

<br>
<br>

## 💼 READ COMMITTED（已提交讀取）—SQL Server 預設

我只能讀「已經被提交」的資料。每次讀資料時會加 Shared Lock，讀完該筆資料就釋放（不是等整個交易結束），不會讀到未提交資料（避免髒讀），但因為釋放太快，可能再次查同筆資料會變了（不可重複讀）。
你只能看「圖書館正式版的書」，別人還沒交稿的不能看。但別人可能過 1 秒就更新再交稿 → 你再看一次內容就變了。

```sql
BEGIN TRAN;

SELECT Price FROM Products WHERE Id = 1;  
-- ✅ 查到資料
-- 🔒 加 Shared Lock → 查完就立刻釋放

-- 🕒 此時別人可以更新這筆資料
-- (例如另個交易 UPDATE Products SET Price = 200)

-- 你這時再更新：
UPDATE Products SET Price = Price + 10 WHERE Id = 1;
-- ⚠️ 可能成功（若別人先提交）
-- ⚠️ 或失敗（若別人鎖著還沒放）
COMMIT;
```

一般瀏覽商品清單
```sql
SELECT * FROM Products WHERE Category = 'Laptop';
```

使用者只是想看有哪些筆電、有多少庫存。資料就算在查詢過程中稍微變動（例如庫存少一台），也沒關係。
適合用 READ COMMITTED 因為不需要強一致性；查完就釋放 Shared Lock；效能高，並行性好。
像在電商網站上看商品頁面時，有人剛好下單一台筆電 → 你的畫面數字可能延遲幾秒更新，但不影響你購物體驗。

<br>
<br>

## 💼 REPEATABLE READ（可重複讀取）

希望整個交易期間，再次查相同條件的資料，內容都一樣

每次讀的資料都加 Shared Lock；鎖會一直保留到交易結束；別人不能修改這些資料；但別人可以插入新的資料（造成幻讀 Phantom Read）。

你在圖書館查資料時，書被你暫時「借出」，別人不能改內容。但圖書館可以新增新書（你再查一次時看到新資料）
在「查詢的當下」會加上 Shared Lock，查完就釋放，所以只會暫時卡住別人。
```sql
BEGIN TRAN;

SELECT Price FROM Products WHERE Id = 1;
-- 🔒 加 Shared Lock，並且直到 COMMIT 都不釋放！

-- 🕒 此時其他交易想改這筆資料：
UPDATE Products SET Price = 200 WHERE Id = 1;
-- ❌ 被卡住 (blocked)，因為你還沒 COMMIT

-- 你後續再查：
SELECT Price FROM Products WHERE Id = 1;
-- ✅ 絕對會看到一樣的值（可重複讀）

COMMIT;
-- Shared Lock 釋放
```

在「整個交易期間」都保留 Shared Lock，直到 COMMIT 或 ROLLBACK 才釋放，所以會卡住別人整段時間

<br>

### 🛒 顧客結帳下單（扣庫存）
```sql
BEGIN TRANSACTION;

SELECT Quantity FROM Products WHERE Id = @ProductId;
-- 顯示目前庫存

UPDATE Products SET Quantity = Quantity - 1 WHERE Id = @ProductId;
COMMIT;
```

這裡如果用 READ COMMITTED，查庫存的那一瞬間加 Shared Lock，查完立刻放，結果在你 UPDATE 之前，另一個交易可能也剛好扣庫存，就可能發生「超賣」情況（overselling）。

👉 為了避免這種情況，會選擇 REPEATABLE READ 或 SERIALIZABLE，在整個交易中鎖住那筆資料，其他人不能修改庫存，直到你這邊 COMMIT，保證你查到的庫存值，不會被別人改。

<br>

### 🛒 錢包轉帳或支付交易

```sql
BEGIN TRANSACTION;

SELECT Balance FROM Accounts WHERE Id = @UserA;
-- 假設餘額 100

UPDATE Accounts SET Balance = Balance - 50 WHERE Id = @UserA;
UPDATE Accounts SET Balance = Balance + 50 WHERE Id = @UserB;

COMMIT;
```

這時候千萬不能讓別人同時修改 UserA 的餘額，否則就會有「重複扣款」、「餘額錯亂」的風險。必須使用 REPEATABLE READ 或 SERIALIZABLE，保證這整個交易期間，查過的餘額不會被改；整個交易視為「一個原子操作」。就像銀行轉帳時會鎖定帳戶，確保你在扣款和入帳中間不會被別的交易干擾

<br>
<br>

## 💼 SERIALIZABLE（可序列化）

我要最嚴格的一致性。我讀到的資料範圍，別人不能改，也不能插入新的在這範圍內
REPEATABLE READ 會為你查過的每一筆資料加上 Shared Lock，且這些鎖會一直保留到交易結束

❌ 但不會鎖住整個查詢範圍 → 所以別人還是可以插入新的資料讓你下一次查多出結果（幻讀）。

鎖整個查詢範圍 (Range Lock)；直到交易結束才釋放；沒有髒讀、沒有不可重複讀、沒有幻讀；但效能最低、最容易造成 blocking，因為他除了保留 Shared Lock 外，還會加上 Range Lock（範圍鎖）

<br>

### 🔹 檢查優惠券是否重複編號

```sql
BEGIN TRAN;
SELECT * FROM Coupons WHERE Code = 'ABC123';
-- 查目前是否已存在

INSERT INTO Coupons (Code) VALUES ('ABC123');
COMMIT;
```

如果用 REPEATABLE READ，查詢沒鎖範圍，另一個交易在你插入前也查一次 → 看不到你的那筆；
結果兩人同時 INSERT 成功 → ❌ 重複代碼。


如果用 SERIALIZABLE，鎖住「Code = 'ABC123'」這個索引範圍，第二個交易在你 COMMIT 前會被卡住；

✅ 保證不會有重複編號。

適合用於「要保證查詢條件的範圍不變」的場景，例如

- 檢查唯一條件（會員代碼、優惠券代碼、帳號註冊重複性）
- 統計或產生編號時避免重複
- 批次處理需要確保範圍不變（例如 select + insert）

<br>
<br>

## 💼 Lost Update（遺失更新）

Lost Update（遺失更新） 指的是兩個交易同時去修改同一筆資料，最後一個 Commit 的人，把前一個人的修改結果覆蓋掉，導致第一個人的更新「被吃掉」，沒有被保留!

想像 Ben 的帳戶餘額是 200 元。此時同時發生兩件事

- A：要扣 100 元（買東西）
- B：也要扣 100 元（轉帳）

A 的更新結果被 B 覆蓋，最終只保留了最後一個 Commit 的資料，這就是「遺失更新」

<br>

### 🔹 用「鎖」防止同時修改

設定隔離層級讓「讀取資料的人鎖住它」：方法 A：使用 REPEATABLE READ 或 SERIALIZABLE
讓當前交易查過的那筆資料，在交易結束前不能被別人修改


- A 查了 Ben 的資料 → Shared Lock 掛上
- B 想改 Ben 的資料 → 🚫 被卡住，要等 A COMMIT 才能動

→ 所以兩人不可能同時改，Lost Update 就不會發生。

「庫存扣減」或「帳戶扣款」都常這樣做

<br>

### 🔹 用「悲觀鎖 (Pessimistic Lock)」手動鎖資料

在程式層明確說：「我這筆要先鎖起來，別人別動。」
我要查這筆資料，順便先拿更新鎖（UPDLOCK），其他交易想改這筆要等我做完

```csharp
using var tran = await context.Database.BeginTransactionAsync(IsolationLevel.ReadCommitted);

var account = await context.Accounts
    .FromSqlRaw("SELECT * FROM Accounts WITH (UPDLOCK) WHERE Id = {0}", "Ben")
    .SingleAsync();

account.Balance -= 100;
await context.SaveChangesAsync();

await tran.CommitAsync();
```
這樣可以確保同時只有一個交易能改這筆資料

<br>

### 🔹 用「樂觀鎖 (Optimistic Concurrency)」

這是 EF Core 推薦 的方式，它不鎖資料，而是透過「版本號 (RowVersion / Timestamp)」檢查資料有沒有被別人改過

在資料表加一個 RowVersion 欄位：
```sql
ALTER TABLE Accounts ADD RowVersion rowversion;
```
在 EF 模型裡
```csharp
public class Account
{
    public int Id { get; set; }
    public decimal Balance { get; set; }
    [Timestamp]
    public byte[] RowVersion { get; set; }
}
```

當兩個交易同時修改時

- A 先查 → RowVersion = 0x0001
- B 也查 → RowVersion = 0x0001
- A 存檔 → 成功，RowVersion 更新為 0x0002
- B 存檔 → EF 比對時發現 RowVersion 不同 → 丟出 DbUpdateConcurrencyException

你就能攔下這種情況，提示使用者「資料已被修改，請重新嘗試」。這就是「樂觀並行控制 (Optimistic Concurrency Control)」的典型作法

<br>
<br>

## 💼 Phantom

在同一個交易中，你用「同樣的查詢條件」查兩次，第二次結果多了或少了幾筆資料（通常是別人插入或刪除的），
感覺像資料「憑空多了一筆」或「少了一筆」——像幻覺一樣

<br>

### 🔹 使用 SERIALIZABLE 隔離層級

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRAN;

SELECT * FROM Orders WHERE Status = 'Pending';
-- 🔒 SQL Server 會鎖整個 "Status = 'Pending'" 範圍
-- 別人不能插入符合這條件的新資料

COMMIT;
```

SQL Server 在這裡會使用 Range Lock（範圍鎖），把「Status = 'Pending'」這個索引範圍整個鎖住。
其他交易若想新增符合這條件的資料，會被卡住直到你 COMMIT。效能最差（鎖的範圍大，並行性低）

<br>

### 🔹 使用 SNAPSHOT 隔離層級

```sql
ALTER DATABASE MyDB SET ALLOW_SNAPSHOT_ISOLATION ON;
GO

SET TRANSACTION ISOLATION LEVEL SNAPSHOT;
BEGIN TRAN;

SELECT * FROM Orders WHERE Status = 'Pending';
-- 🧊 讀的是「交易開始時」的資料快照

-- 其他人插入新資料不影響這次快照
-- 所以不會看到幻讀

COMMIT;
```

我不鎖範圍，而是讀我當下開始交易那一刻的「資料快照」

✅ 不會被幻讀影響
✅ 不會鎖別人
⚙️ 效能高、並行性好

⚠️ 但需要資料庫開啟 ALLOW_SNAPSHOT_ISOLATION


| 情境            | 現象           | 影響          | 適用解法                    |
| ------------- | ------------ | ----------- | ----------------------- |
| 查詢「待出貨」訂單清單   | 同時有人新增了新訂單   | 查第二次多一筆（幻讀） | SERIALIZABLE / SNAPSHOT |
| 查詢「庫存 < 10」商品 | 同時有商品新補貨或新上架 | 清單筆數變       | SERIALIZABLE / SNAPSHOT |
| 查詢「今日下單數量」    | 有人剛下單        | 統計數字變       | SNAPSHOT（用快照確保一致）       |

<br>
<br>

## 💼 Write Skew

兩個交易同時「讀取相同條件下不同資料」，都判斷「目前狀況合法」，於是各自修改不同資料，結果整體狀態卻不再符合業務規則。

假設有一個規則：每個時段至少要有兩位醫生 on call

🧍‍♂️ Transaction A（Leo 請假）

1️⃣ 查目前 on_call 的醫生數量 → 2
2️⃣ 覺得符合規範（>= 2）→ 可以請假
3️⃣ UPDATE Leo 的狀態為 off_call

🧍‍♀️ Transaction B（Mark 請假）

1️⃣ 同時間查 on_call 的醫生數量 → 2
2️⃣ 覺得符合規範（>= 2）→ 也可以請假
3️⃣ UPDATE Mark 的狀態為 off_call

<br>

### 🔹 使用 SERIALIZABLE 隔離層級

這是最直接（也是最保守）的做法。在 SERIALIZABLE 模式下，SQL Server 會對查詢條件加「範圍鎖 (Range Lock)」

<br>

### 🔹 用「明確鎖」(Pessimistic Lock)

如果你不想用整個 SERIALIZABLE 隔離層級，也可以手動鎖定資料。
```sql
SELECT * FROM Doctors WITH (XLOCK, HOLDLOCK) WHERE OnCall = 1;
```

- XLOCK：強制加排他鎖
- HOLDLOCK：等效於 SERIALIZABLE 的鎖持續時間

這樣就確保在交易結束前，其他人不能同時查詢或修改這一批資料，本質上就是「悲觀鎖」策略

<br>

### 🔹 加入「業務層檢查」與「唯一條件」

在應用程式層，也可以加上「防守性邏輯」
```sql
BEGIN TRAN;

SELECT COUNT(*) FROM Doctors WHERE OnCall = 1 FOR UPDATE;

IF (count < 2)
   ROLLBACK;
ELSE
   UPDATE Doctors SET OnCall = 0 WHERE Id = @DoctorId;

COMMIT;
```

在同一交易裡「查 + 判斷 + 更新」一次完成。即使有其他人同時請假，也會被鎖機制擋住