---
title: EF Core - TransactionScope vs BEGIN TRAN/COMMIT
date: 2025-10-17 17:45:05
categories: 資料疆界的航圖
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Commit/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Commit/1_landing.png
tags:
    - EF CORE
toc:
toc_number:
comments :
---


{% btn 'https://blog.darkthread.net/blog/transcope-and-commit-tran/',TransactionScope與COMMIT TRAN,far fa-hand-point-right %}

![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Commit/1_landing.png)

{% tabs TransactionScope vs BEGIN TRAN/COMMIT%}


<!-- tab transactionScope or BEGIN TRAN-->

「如果用 `TransactionScope` 包了一段程式，裡面又在 SQL 裡手動開 `BEGIN TRAN`，那最後誰決定要不要真的 commit？」


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Commit/1__question.png)


<!-- endtab -->


<!-- tab TransactionScope-->

只要你在 using 範圍內操作資料庫，EF 或 ADO.NET 會自動參與這個「環境交易 (`Ambient Transaction`)」。如果沒呼叫 `scope.Complete()`，在離開 `using` 時就會自動 `rollback`

```csharp
using (var scope = new TransactionScope())
{
    // 這裡做的所有 DB 操作（只要使用相同的連線）都會自動參與這個交易
    DoSql1();
    DoSql2();

    // 決定是否 commit
    scope.Complete();
}
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Commit/2_nested_relation.png)



<!-- endtab -->

<!-- tab BEGIN TRAN / COMMIT TRAN-->

這是 T-SQL 層級的交易，寫在 SQL 裡面
```sql
BEGIN TRAN
    INSERT INTO Orders ...
COMMIT TRAN
```

這是 資料庫端的交易。它可以存在於

- 一個獨立的連線（沒有 `Ambient Transaction`）
- 某個由 .NET 傳進來的環境交易裡面

<!-- endtab -->

<!-- tab 兩者怎麼互動-->

當你在 C# 中用 TransactionScope 包起來後，你建立的每個資料庫連線，預設會自動加入這個環境交易 (`Ambient Transaction`)，所以 BEGIN TRAN 這個「SQL 裡的交易」其實只是「子交易」或「巢狀交易」，它會參與外部的環境交易 (`Ambient Transaction`)

```bash
TransactionScope (老大)
 ├── Connection 1：SQL 操作
 └── Connection 2：SQL 操作（裡面有 BEGIN/COMMIT TRAN）
```

當 TransactionScope 最後沒有 .Complete()，整個 `Ambient Transaction` 會被 Rollback，即使第二段 SQL 在資料庫層看起來「COMMIT 成功」，實際上那只是該連線內的子交易結束，最終整個分散式交易 (MSDTC) 會通知資料庫 rollback

<!-- endtab -->

<!-- tab 強迫建立不同的連線，確保觸發 MSDTC-->

```csharp
static string cs = "Data Source=...省略...;Application Name={0}";
static string getDiffCnStr()
{
    return string.Format(cs, Guid.NewGuid());
}
```

每次建立連線時，都使用不同的 Application Name。因為 SQL Server 的連線池 (Connection Pool) 會根據連線字串內容共用連線，這樣做可以強迫建立不同的連線，確保之後會觸發 MSDTC 分散式交易


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Commit/3_trigger_msdc_prepared.png)


<!-- endtab -->

<!-- tab 操作-->

## 清空測試用的 LockLab 表

```csharp
// 將LockLab資料表清空
truncateTable();
```


![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Commit/3_2_truncate_table.png)


## 建立TransactionScope


```csharp
using (TransactionScope tx = new TransactionScope())
{
}

```


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Commit/3_3_start_transactionscope.png)



## 以隨機式連線字串建立連線，確保觸發分散式交易


```csharp
using (SqlConnection cn = new SqlConnection(getDiffCnStr()))
```


## 第一段 sql, Insert


```csharp
//以隨機式連線字串建立連線，確保觸發分散式交易
//參考: http://blog.darkthread.net/post-2010-11-12-msdtc-2008.aspx 
using (SqlConnection cn = new SqlConnection(getDiffCnStr()))
{
    cn.Open();
    var cmd = cn.CreateCommand();
    cmd.CommandText = 
    "INSERT INTO LockLab VALUES ('TEST','201302', 1)";
    cmd.ExecuteNonQuery();
}
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Commit/4_commit_1.png)



## 第二段 SQL 使用COMMIT TRAN


```csharp
//第二段SQL 使用COMMIT TRAN
using (SqlConnection cn = new SqlConnection(getDiffCnStr()))
{
    cn.Open();
    var cmd = cn.CreateCommand();
    cmd.CommandText = @"
    BEGIN TRAN
    INSERT INTO LockLab VALUES ('JEFF','201302', 1)
    COMMIT TRAN
    ";
    cmd.ExecuteNonQuery();
}
```


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Commit/4_2_commit.png)


## 觸發 Rollback

```csharp
throw new ApplicationException("刻意觸發錯誤!");
//tx.Complete()不會被執行，Transaction應Rollback
tx.Complete();
```


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Commit/4_3_exception.png)



## Overview


```csharp

// 將LockLab資料表清空
truncateTable();

// 建立TransactionScope
using (TransactionScope tx = new TransactionScope())
{
    try
    {
        //以隨機式連線字串建立連線，確保觸發分散式交易
        //參考: http://blog.darkthread.net/post-2010-11-12-msdtc-2008.aspx 
        using (SqlConnection cn = new SqlConnection(getDiffCnStr()))
        {
            cn.Open();
            var cmd = cn.CreateCommand();
            cmd.CommandText = 
            "INSERT INTO LockLab VALUES ('TEST','201302', 1)";
            cmd.ExecuteNonQuery();
        }

        //第二段SQL 使用COMMIT TRAN
        using (SqlConnection cn = new SqlConnection(getDiffCnStr()))
        {
            cn.Open();
            var cmd = cn.CreateCommand();
            cmd.CommandText = @"
            BEGIN TRAN
            INSERT INTO LockLab VALUES ('JEFF','201302', 1)
            COMMIT TRAN
            ";
            cmd.ExecuteNonQuery();
        }

        throw new ApplicationException("刻意觸發錯誤!");
        //tx.Complete()不會被執行，Transaction應Rollback
        tx.Complete();
    }
    catch (Exception ex)
    {
        Console.Write(ex.ToString());
    }
}
Console.Read();

static void truncateTable()
{
    using (SqlConnection cn = new SqlConnection(getDiffCnStr()))
    {
        cn.Open();
        var cmd = cn.CreateCommand();
        cmd.CommandText = "TRUNCATE TABLE LockLab";
        cmd.ExecuteNonQuery();
    }
}
```

| 步驟 | 動作                                  | 狀態                                        |
| -- | ----------------------------------- | ----------------------------------------- |
| 1  | 第一段 SQL 插入 `'TEST'`                 | 在交易中，未提交                                  |
| 2  | 第二段 SQL 插入 `'JEFF'` 並 `COMMIT TRAN` | 子交易 Commit，但仍屬於外層交易                       |
| 3  | 丟出 Exception，未呼叫 `tx.Complete()`    | TransactionScope 結束時 Rollback 整個 MSDTC 交易 |
| 4  | **結果**                              | `LockLab` 仍是空表（兩筆資料都被 Rollback）           |


即使第二段 SQL 在程式裡明確執行了 COMMIT TRAN，只要它是在 TransactionScope 的範圍內，就會被視為外層環境交易 (`Ambient Transaction`) 的子交易。當外層 TransactionScope 沒呼叫 .Complete()，整個分散式交易會 Rollback → ✅ 結果是所有 Insert 都不會留下。這就是為什麼這段程式執行完後 LockLab 仍是空的



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Commit/4_5_locklab_empty.png)


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Commit/5_msdc_all_rollback.png)


<!-- endtab -->

<!-- tab summary-->


![ˇ](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Commit/6_table_timeflow.png)


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Commit/commit.png)


<!-- endtab -->

{% endtabs %}