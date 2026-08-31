---
title: Concurrency-safe
date: 2026-08-27 08:44:00
categories: DB
top_img: 
cover : 
toc:
toc_number:
comments :

---

## 先看情境

轉帳這種先查詢、再判斷、最後修改的邏輯，就算包在 `BEGIN` 與 `COMMIT` 之間，同時有兩筆請求進來時，餘額還是有可能算錯。原因不是 transaction 沒生效，而是 transaction 保證的東西，跟表面上看起來的並不完全一樣。

假設一開始有三個帳戶：

- A：100 元
- B：100 元
- C：100 元

現在幾乎同時來了兩個請求，一個要從 `A` 轉 80 元給 `B`，另一個要從 `A` 轉 80 元給 `C`。應用程式常見的寫法是這樣：

```sql
BEGIN;
balance = SELECT balance FROM accounts WHERE id = 'A';
IF balance >= 80 THEN
    UPDATE accounts SET balance = balance - 80 WHERE id = 'A';
    UPDATE accounts SET balance = balance + 80 WHERE id = 'B';
END IF;
COMMIT;
```

另一個處理另一筆轉帳的執行緒，同時做著幾乎一模一樣的事情，只是把最後的收款帳戶換成 `C`。

即使整段邏輯用 `BEGIN` 與 `COMMIT` 包起來，也不代表這兩筆轉帳彼此看不到對方、不會互相干擾。因為 transaction 其實同時在處理兩件不同的事，一件是這段操作要嘛全部成功、要嘛全部失敗，另一件是多個 transaction 同時執行時，彼此能看到什麼、會不會互相踩到對方的資料。

這兩件事分別對應 `ACID` 裡的兩個字母：

- Atomicity（原子性）：保證一個 transaction 內的多個操作，要嘛全部成功，要嘛全部沒發生，不會卡在做一半的狀態。
- Isolation（隔離性）：決定多個 transaction 同時執行時，彼此能看到什麼版本的資料，能不能同時修改同一筆資料。
- 兩者的關係：Atomicity 顧的是單一 transaction 內部，Isolation 顧的才是多個 transaction 之間。轉帳算錯，通常是 Isolation 沒處理好。

其中又以 Isolation 最關鍵，因為 Atomicity 只保證單一 transaction 不會做一半失敗，並不保證兩個同時執行的 transaction 不會互相干擾。

## Atomicity 解決什麼問題

假設一筆轉帳需要兩個操作，先扣款再加款：A 扣 80，接著 B 加 80。

如果沒有 transaction 保護，這兩步之間任何一個環節出狀況，都可能只完成一半：

1. A 扣款成功：`A` 的餘額從 100 變成 20，這一步已經寫入資料庫。
2. 伺服器在這裡當機：`B` 加款的那一步還沒執行，程式就中斷了。

結果：`A` 變成 20，`B` 還是 100，中間的 80 元從系統裡憑空消失，帳目對不起來。

Transaction 的 Atomicity 能保證的是，把這兩個操作包進同一個 `BEGIN` 到 `COMMIT` 之間之後，最終只會有兩種結果，不會停在半途：

- 全部成功：A 餘額 20，B 餘額 180，允許。
- 全部失敗（rollback）：A 餘額 100，B 餘額 100，允許。
- 只扣款、沒加款：A 餘額 20，B 餘額 100，不允許。

所以 Atomicity 解決的問題，是一個 transaction 做到一半失敗這種情境，可以理解成整組操作只有全部成功或全部失敗兩種結局，這就是 `ACID` 裡的 `A`。不過這件事完全沒有觸及另一個同時執行的 transaction 會不會看到中間狀態、會不會跟這筆轉帳互相干擾，那是 Isolation 要處理的範圍。

## Isolation 在問什麼

Isolation 問的是，多個 transaction 同時執行的時候，彼此可以看到什麼，會不會互相踩到對方還沒確定的資料。回到最開始的情境，兩筆轉帳同時進行時，實際發生的時序如下：

1. T1 與 T2 各自 `BEGIN`。
2. T1 讀 `A.balance`，回傳 100；T2 也讀 `A.balance`，回傳 100。
3. T1 判斷 100 >= 80，成立；T2 也判斷 100 >= 80，成立。
4. T1 執行 A 扣 80、B 加 80；T2 執行 A 扣 80、C 加 80。
5. T1 `COMMIT`；T2 `COMMIT`。

兩邊都讀到 `A = 100`，都判斷餘額足夠，於是都各自完成了自己的轉帳。最後的結果是：A = 20，B = 180，C = 180。

問題所在：系統實際上等於發出去了 160 元（B 收到 80，C 收到 80），但 `A` 一開始只有 100 元。這種兩個 transaction 各自檢查條件時都成立，合併起來卻超出實際限制的狀況，就是 race condition。

而最容易被忽略的地方在於，`BEGIN` 與 `COMMIT` 本身並不會自動保證這種事不會發生。以 PostgreSQL 為例，預設的 isolation level 是 `READ COMMITTED`。這個等級能保證的仍然是 Atomicity 那部分，但如果應用程式的邏輯是先 `SELECT`、在應用程式裡判斷、之後才 `UPDATE`，這種讀取與判斷之間的空窗期，並不會被 `READ COMMITTED` 自動堵住，仍然需要額外處理 concurrency，例如接下來提到的三種方法。

## 方法一：用 FOR UPDATE 鎖住那一筆資料

以 PostgreSQL 為例，可以在 `SELECT` 的時候加上 `FOR UPDATE`：

```sql
BEGIN;
SELECT balance
FROM accounts
WHERE id = 'A'
FOR UPDATE;
```

這裡的 `FOR UPDATE` 是關鍵，效果可以理解成，先執行的 transaction 等於在跟資料庫說，正在處理帳戶 `A`，在這筆處理完成之前，其他 transaction 不准修改這一筆資料。實際的執行時序如下：

1. T1 執行 `SELECT A FOR UPDATE`，取得 row lock，A = 100。
2. T2 也執行 `SELECT A FOR UPDATE`，想取得同一把鎖，被阻塞（BLOCKED）。
3. T1 判斷 100 >= 80，成立，執行 A 扣 80、B 加 80，`COMMIT`，釋放鎖。
4. T2 才能繼續，讀到最新的 A = 20，判斷 20 >= 80，不成立，拒絕轉帳。

先執行的 `T1` 完成扣款並提交之後，鎖才釋放，接手的 `T2` 這時讀到的已經是最新的餘額 20，判斷 20 是否大於等於 80 不成立，於是拒絕這筆轉帳。最終結果：`A` 剩下 20，`B` 收到轉帳變成 180，`C` 因為餘額不足被擋下，維持 100，兩筆轉帳依序處理，沒有超額。

這個做法的核心概念，是先讀資料、再由應用程式根據資料做判斷，所以在 `SELECT` 的當下就把這筆資料鎖住，屬於悲觀式的做法，先阻止其他 transaction 有機會碰到這筆還在處理中的資料。

## 方法二：把檢查條件寫進 UPDATE 本身

另一種做法是不要先 `SELECT`、判斷、再 `UPDATE`，而是讓 `UPDATE` 本身就帶有條件：

```sql
UPDATE accounts
SET balance = balance - 80
WHERE id = 'A'
  AND balance >= 80;
```

接著只要檢查這行語句實際影響了幾筆資料，也就是 `affected rows`，如果是 1 才代表扣款真的成功。

流程：T1 執行 `UPDATE`，A 扣款成功變成 20，接著 T2 執行 `UPDATE`，`balance >= 80` 不成立，`affected rows` 等於 0，轉帳失敗。

假設 `T1` 與 `T2` 幾乎同時對 `A = 100` 發出這個條件式 `UPDATE`，資料庫層級會保證這兩個 `UPDATE` 不會同時套用在同一筆資料上。先完成的那一個會把 `A` 改成 20，後面那一個再檢查 `balance >= 80` 時，因為 20 已經不滿足這個條件，這次 `UPDATE` 影響的資料筆數就是 0，代表這筆轉帳沒有真的發生。

這個做法之所以重要，是因為原本讀 `balance`、應用程式判斷、修改 `balance` 三個分開的步驟，被壓縮成資料庫裡的一個 atomic operation，也就是把檢查與寫入合併成同一個原子動作，不再留下讓兩個 transaction 都通過檢查的空窗期。

## 方法三：使用 Serializable isolation

`SERIALIZABLE` 這個 isolation level 想表達的概念是，這些 transaction 就算實際上是同時執行的，最終呈現出來的結果，必須等價於把它們一個接一個依序執行過一次。

- 實際發生的狀況：`T1` 與 `T2` 在時間軸上是真正並行、互相重疊執行的。
- 資料庫保證看到的結果：結果必須等同於 `T1` 先跑完、`T2` 才開始，或是反過來 `T2` 先跑完、`T1` 才開始，兩種順序中的其中一種。
- 做不到的時候：資料庫判斷兩者無法安全地排出一個合法順序，就會讓其中一個 transaction 發生 serialization conflict 而被 `ROLLBACK`。

當發生 serialization conflict 而被回滾時，應用程式必須自行 retry 這個 transaction，這個重試機制是使用 `SERIALIZABLE` 時很重要的一部分。所以 `SERIALIZABLE` 並不是資料庫用某種魔法讓所有 transaction 永遠都成功，而是如果沒辦法得到一個合法的序列化執行結果，資料庫寧願讓其中一個 transaction 失敗，交給應用程式重試。這種做法允許多個 transaction 同時執行，但資料庫保證最終結果等價於某一種依序執行的排列方式，至於底層具體怎麼實作這件事，屬於各家資料庫自己的實作細節，可能用鎖、MVCC、predicate 或 range locking、conflict detection 加上 abort 及 retry，甚至混合使用多種手段。

PostgreSQL 的 `SERIALIZABLE` 是一個很好的例子，它採用的機制叫做 Serializable Snapshot Isolation，簡稱 `SSI`。它並不是單純看到某個 transaction 讀了 `A` 就把 `A` 整個鎖死、不准其他 transaction 再讀取，而是允許兩邊都正常讀取與繼續執行，改由資料庫在背後持續追蹤彼此的依賴關係：

1. T1 讀 A = 100；T2 讀 A = 100。
2. 兩邊都繼續執行。
3. DB 偵測兩者之間的 dependency，發現可能形成不合法的 serialization 結果。
4. 其中一個 transaction 被 `ABORT`，應用程式收到失敗，重新 retry 這個 transaction。

所以方法三的核心精神，比較接近不一定要一開始就阻止兩個 transaction 同時執行，但保證最後不會產生一個不可能由任何一種依序執行方式得到的結果，跟方法一那種一開始就悲觀鎖住資料的做法，在設計哲學上是不同方向。

## 為什麼這類問題 RDBMS 通常處理得比較完整

如果把問題範圍限定在多筆資料之間存在複雜關係，而且在 concurrency 之下仍然必須維持 business invariant 這一類情境，關聯式資料庫通常確實有比較成熟、完整、自然的處理方式。這不是說 NoSQL 完全做不到，而是關聯式資料庫從設計核心開始，就把 transaction、concurrency control、constraints、關聯性當成第一級公民在對待。以 PostgreSQL、MySQL 的 InnoDB 引擎、SQL Server 為例，通常可以直接取得一整套現成工具：

- Transaction 控制：`BEGIN` / `COMMIT` / `ROLLBACK`，以及 Atomicity 保證。
- Isolation Level：`READ COMMITTED`、`REPEATABLE READ`、`SERIALIZABLE`。
- Concurrency Control：MVCC、Row Lock、`SELECT ... FOR UPDATE`。
- Constraints：`PRIMARY KEY`、`UNIQUE`、`FOREIGN KEY`、`CHECK`、`NOT NULL`。

這些工具可以互相組合，用來維護資料上的種種前提條件。舉例來說，一套銀行系統裡通常會有帳戶、交易紀錄、客戶、卡片、貸款這些彼此之間存在大量關聯的資料，而且系統通常要同時滿足下面這些前提：

- 帳戶不能被提領超過實際餘額。
- 一筆交易不能只完成一半就停在中間狀態。
- 帳戶不存在的情況下，不能產生對應的交易紀錄。
- 同一個 `payment_id` 不能被扣款兩次。
- 刪除客戶資料時，不能留下沒有對應客戶的孤兒帳戶。
- 多人同時操作同一筆資料時，結果仍然必須正確。

結論：這種需要同時維護多筆關聯資料、又要求 concurrency 下仍然正確的問題，正是關聯式資料庫從一開始就擅長處理的 workload。

## 換成 DynamoDB，思考方式差異更明顯

DynamoDB 目前也提供 ACID transaction，可以透過 `TransactWriteItems` 把多個操作組成一個 all-or-nothing 的 transaction。所以「NoSQL 沒有 transaction」這個說法，已經不成立。不過 DynamoDB 有一個非常關鍵的設計概念，就是 `Partition Key`，整個資料模型、擴充能力與效能表現，都高度圍繞著 partition 在運作。

因此在設計 DynamoDB 的資料結構時，思考的起點通常跟關聯式資料庫不太一樣：

- DynamoDB 的思考起點：存取模式（access pattern）長什麼樣子。`Partition Key` 該怎麼設計。資料要不要 denormalize、哪些資料應該放在一起、怎麼避免 hot partition。
- 關聯式資料庫的思考起點：Entity 之間的關聯是什麼、要 normalize 成哪些資料表、foreign key 怎麼建立、查詢時要怎麼 `JOIN`。

兩者並不是誰優誰劣的關係，而是在優化不同的問題。用 DynamoDB 硬做複雜關聯查詢，等於是在使用它比較昂貴、比較複雜的能力；相反地，這正是 PostgreSQL 這類關聯式資料庫本來就擅長的 workload。

## 整體回顧

轉帳這類先讀取、再判斷、最後修改的邏輯，光靠 `BEGIN` 與 `COMMIT` 包起來並不足夠，因為 `ACID` 裡的 Atomicity 只保證單一 transaction 不會做到一半失敗，真正決定多個 transaction 同時執行會不會互相干擾的，是 Isolation。要解決 Isolation 帶來的 race condition，常見有三種方向，各自的取捨如下：

- 方法一：FOR UPDATE。在 `SELECT` 當下就悲觀鎖住那一筆資料，其他 transaction 必須等待，實作直覺，但等待中的 transaction 會被阻塞。
- 方法二：Conditional UPDATE。把檢查條件寫進 `UPDATE` 語句本身，靠 `affected rows` 判斷是否成功，把檢查與寫入合併成一個原子操作，不需要額外鎖。
- 方法三：Serializable。允許 transaction 同時執行，資料庫在背後偵測衝突，一旦發現無法排出合法順序就讓其中一方失敗，交由應用程式 retry。

再往上一層看，這類多筆資料互有關聯，又要求 concurrency 下仍然正確的問題，關聯式資料庫從設計核心開始就把 transaction、isolation level、row lock、各種 constraint 當成第一級公民，通常能提供比較完整的現成工具。DynamoDB 這類 NoSQL 資料庫雖然也已經有 ACID transaction，但設計思維更圍繞在 `Partition Key` 與存取模式上，跟關聯式資料庫的正規化、外鍵、`JOIN` 思維是不同的優化方向，兩者是在解決不同性質的問題，而不是互相取代的關係。
