---
title: EF Core - InvalidOperationException
date: 2025-05-28 08:26:05
categories: 
top_img: https://github.com/CHI-KEKE/pics/blob/main/EF/ocean.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/EF/ocean.png?raw=true
tags:
    - 
toc:
toc_number:
comments :
---

程式世界裡最令人頭痛的，從來不是錯誤訊息，而是**「有時錯，有時對」**的那種模糊不明。你是不是也曾遇過：明明昨天的程式還好好的，今天一跑卻爆了 InvalidOperationException？

這不是你寫錯，而是你不小心跨越了 DbContext 的界線。就像在海上行船，一不小心就漂出了領海，進入無人知曉的風暴。

這篇文章將帶你從根源理解 InvalidOperationException 的幾個常見場景，並透過範例與圖解，一次性釐清觀念，避免未來再被神祕的錯誤訊息追殺。

![ocean](https://github.com/CHI-KEKE/pics/blob/main/EF/ocean.png?raw=true)

<br>
<br>

## 💥 DbContext 實例重複使用｜一艘船不能載兩個船長

### 問題來源
當你在多個請求或執行緒間重複使用同一個 DbContext 實例，就可能觸發 InvalidOperationException。為什麼？因為…

❗ DbContext 不是 Thread-Safe 的物件。

它內部維護著實體狀態（Added, Modified, Deleted…），而這些狀態並不是為多執行緒設計的。你讓多個線程同時操控 DbContext，就像讓兩個船長同時掌舵 —— 翻船。

```CSHARP
public class UserService
{
    private static readonly AppDbContext _context = new AppDbContext(); // ⚠️ 危險

    public Task AddUserAsync(User user)
    {
        _context.Users.Add(user);
        return _context.SaveChangesAsync(); // 多執行緒同時進來會爆炸
    }
}
```

### ✅ 解法

- 使用依賴注入（DI），確保 DbContext 的生命週期是 Scoped，每次請求都取得全新的實例。
```CSHARP
services.AddDbContext<AppDbContext>(options => ...); // 預設為 Scoped
```

- 若在非同步情境下同時處理多筆資料（如 Task.WhenAll），請為每個 Task 各自建立 DbContext 實例。(但不建議這樣操作)

<br>
<br>

## 💥 SaveChanges 誤用｜看似原子，其實碎裂

### 問題來源

你可能以為包在 BeginTransaction() 裡，資料就會「全成功或全失敗」。但事實上：

> ❗ 每次 SaveChanges() 本身就會觸發 mini-transaction。

若你在一個交易中呼叫多次 SaveChanges()，只要其中一次失敗，先前成功的部份就無法自動 Rollback！


危險寫法：
```CSHARP
public void SaveChangesInTransaction()
{
    using (var transaction = _dbContext.Database.BeginTransaction())
    {
        _dbContext.Users.Add(new User { Name = "User1" });
        _dbContext.SaveChanges(); // 儲存第一筆資料

        _dbContext.Users.Add(new User { Name = "User2" });
        _dbContext.SaveChanges(); // 儲存第二筆資料

        transaction.Commit(); // 提交事務
    }
}
```

### ✅ 解法

```CSHARP
using (var transaction = _dbContext.Database.BeginTransaction())
{
    _dbContext.Users.Add(new User { Name = "User1" });
    _dbContext.Users.Add(new User { Name = "User2" });

    _dbContext.SaveChanges(); // ✅ 一次寫入，成功才 Commit
    transaction.Commit();
}
```

<br>
<br>

## 💥 跨 DbContext 協同操作｜兩條船不是同一隊

###　問題來源
當你在同一個邏輯流程中操作了多個 DbContext 實例，而這些實例所連線的資料庫彼此不共享同一個底層連線（DbConnection）或交易（Transaction），就無法讓它們共同參與一個真正的資料庫交易（transaction）。

EF Core 偵測到交易混亂時，會直接拋出 InvalidOperationException，EF Core 設計上，每個 DbContext 都是獨立的狀態管理者與資料庫連線管理者。它內部的行為包含：

- 自己的 ChangeTracker
- 自己的 DbConnection 實例（除非特別共用）
- 自己的 Transaction 管理

當你用兩個 DbContext 嘗試進行同一筆交易時，它們可能無法互相同步狀態也不知道對方的交易存在與否


```CSHARP
using (var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
{
    var context1 = new AppDbContext();
    var context2 = new AppDbContext();

    context1.Users.Add(new User { Name = "User1" });
    context1.SaveChanges();

    context2.Users.Add(new User { Name = "User2" });
    context2.SaveChanges(); // 💥 這裡可能會拋 InvalidOperationException 或分散式交易錯誤

    scope.Complete();
}
```

### ✅ 解法

- 解法一：強制共用底層連線與交易（但不推薦，複雜易錯）
- 解法二：改為使用單一 DbContext 處理整個流程


<br>
<br>

## 💥 漏設 TransactionScopeAsyncFlowOption.Enabled｜異步交易上下文遺失

### 問題來源
當你在 C# 中使用 TransactionScope 包裹非同步方法（例如 SaveChangesAsync()）時，如果沒有設定 TransactionScopeAsyncFlowOption.Enabled，那麼在 await 之後，交易上下文（Transaction Context）會中斷，導致 EF Core 無法感知目前交易，最後拋出

```PLAINTEXT
System.InvalidOperationException: A TransactionScope must be disposed on the same thread that it was created.
```

C# 的 async/await 是「狀態機（state machine）」，不是單純的同步堆疊函數呼叫。在 await 的那一行執行完後，程式控制權會「跳出去」，等任務完成再「跳回來」。如果你沒有特別設定「交易流動選項」，這個跳出去與跳回來的過程會讓交易上下文丟失。而 TransactionScope 是靠 Ambient Transaction（環境交易） 的概念在工作的，依賴 ThreadStatic 儲存當前交易。一旦 await 讓你的執行換了執行緒（例如從 ThreadA 換到 ThreadB），原本儲存在 ThreadA 的 ambient transaction context 就不見了。

### ✅ 解法

在使用 TransactionScope 時，設置 TransactionScopeAsyncFlowOption.Enabled 以支持異步操作中的事務流。

```CSHARP
using (var transactionScope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled))
{
    // 非同步操作
    await dbContext.SaveChangesAsync();
    transactionScope.Complete();
}
```


## 🌊 結語｜界線不是限制，是守護

InvalidOperationException 表面看來只是個錯誤訊息，實則是在提醒我們：

「你越界了，請回頭。」

在 EF Core 裡，DbContext 就像一艘船，它有明確的範圍與規則。你若硬是拉它去別的領海跑，它就會給你個錯誤作為警告。
學會掌握交易的邊界、資料的原子性、與上下文的獨立性，不只是技術層面的最佳實踐，更是讓系統穩定與可維護的基石。