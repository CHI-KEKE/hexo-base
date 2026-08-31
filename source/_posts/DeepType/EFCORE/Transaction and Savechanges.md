---
title: EFCore - Tranaction and SaveChanges
date: 2025-11-01 11:24
categories: 
top_img: https://github.com/CHI-KEKE/pics/blob/main/EF/ocean.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/EF/ocean.png?raw=true
tags:
    - 
toc:
toc_number:
comments :
---


{% tabs SaveChanges() 和 Transaction%}

<!-- tab SaveChanges() 和 Transaction 的關係-->

在資料庫的世界裡，一次或多次資料修改（INSERT、UPDATE、DELETE）必須放在 交易（Transaction） 裡，這樣如果其中一步錯了，整個動作都可以 rollback 回去，確保資料不會半途修改一半

SaveChanges() 就是 EF Core 幫你「把所有變動打包起來執行」的動作。而 EF Core 為了確保這個過程的資料一致性，在執行時會自動幫你開一個 Transaction



| 狀況                                     | EF Core 內部會自動開啟交易嗎？ | `SaveChanges()` 會自動 Commit 嗎？ | 備註                                             |
| -------------------------------------- | ------------------- | ----------------------------- | ---------------------------------------------- |
| ✅ 你 **沒有** 手動開交易                       | ✅ 是                 | ✅ 是                           | `SaveChanges()` 自動包一個 transaction，執行完自動 commit |
| ⚠️ 你 **手動開了交易 (`BeginTransaction()`)** | ❌ 否（你已經開了）          | ❌ 否（尊重你控制的交易）                 | `SaveChanges()` 只會執行 SQL，不會幫你 commit           |



當你呼叫 context.SaveChanges()，EF Core 做了這幾件事

- 檢查目前是否已有開啟中的 transaction

沒有 → 自己開一個（用 BeginTransaction()）
有 → 使用現有的 transaction

- 執行所有要更新的 SQL (UPDATE, INSERT, DELETE)

若是 EF 自己開的 transaction，就在完成後自動呼叫 Commit()，若是你開的 → EF 不會 commit，交給你決定


<!-- endtab -->

<!-- tab TransactionScope 的時間限制-->


## 🔹 BeginTransaction()

BeginTransaction() 沒有固定時間限制，它會一直存在，直到你呼叫 Commit()、Rollback()、或連線被關閉 / 中斷為止

<br>

## 🔹 TransactionScope


測試 timeout
```csharp
using var scope = services.CreateScope();
var context = scope.ServiceProvider.GetRequiredService<WebStoreDbContext>();

// ✅ 使用 TransactionScope with Timeout
var transactionOptions = new TransactionOptions
{
    IsolationLevel = System.Transactions.IsolationLevel.ReadCommitted,
    Timeout = TimeSpan.FromSeconds(5) // 🔥 交易最長可持續 5 秒
};

// ✅ 使用 TransactionScopeAsyncFlowOption.Enabled 可避免 async 時出錯
using var ts = new TransactionScope(
    TransactionScopeOption.Required,
    transactionOptions,
    TransactionScopeAsyncFlowOption.Enabled);

System.Console.WriteLine($"🕐 Transaction started at: {DateTime.Now:HH:mm:ss}");

var result = context.PromotionEngines
    .First(p => p.PromotionEngineId == 6344);

// 模擬長時間操作（這裡會超過 timeout）
Thread.Sleep(40000);

result.PromotionEngineName = "Test Timeout TransactionScope";
context.SaveChanges(); // ❌ 此時交易已超時，會丟 TransactionAbortedException

ts.Complete(); // 不會執行到這裡，因為上面已超時
System.Console.WriteLine($"✅ Transaction completed at: {DateTime.Now:HH:mm:ss}");
```


<!-- endtab -->

<!-- tab 錯誤記錄-->


 


## Timeout 的錯誤

```bash
An exception occurred in the database while saving changes for context type 'WTF.Console.Models.WebStoreDbContext'.
System.Transactions.TransactionException: The operation is not valid for the state of the transaction.
---> System.TimeoutException: Transaction Timeout
    --- End of inner exception stack trace ---
    at System.Transactions.TransactionState.EnlistPromotableSinglePhase(InternalTransaction tx, IPromotableSinglePhaseNotification promotableSinglePhaseNotification, Transaction atomicTransaction, Guid promoterType)
    at System.Transactions.Transaction.EnlistPromotableSinglePhase(IPromotableSinglePhaseNotification promotableSinglePhaseNotification, Guid promoterType)
    at System.Transactions.Transaction.EnlistPromotableSinglePhase(IPromotableSinglePhaseNotification promotableSinglePhaseNotification)
    at Microsoft.Data.SqlClient.SqlInternalConnection.EnlistNonNull(Transaction tx)
    at Microsoft.Data.SqlClient.SqlInternalConnection.Enlist(Transaction tx)
    at Microsoft.Data.SqlClient.SqlInternalConnectionTds.Activate(Transaction transaction)
    at Microsoft.Data.ProviderBase.DbConnectionInternal.ActivateConnection(Transaction transaction)
    at Microsoft.Data.ProviderBase.DbConnectionPool.PrepareConnection(DbConnection owningObject, DbConnectionInternal obj, Transaction transaction)
    at Microsoft.Data.ProviderBase.DbConnectionPool.TryGetConnection(DbConnection owningObject, UInt32 waitForMultipleObjectsTimeout, Boolean allowCreate, Boolean onlyOneCheckConnection, DbConnectionOptions userOptions, DbConnectionInternal& connection)
    at Microsoft.Data.ProviderBase.DbConnectionPool.TryGetConnection(DbConnection owningObject, TaskCompletionSource`1 retry, DbConnectionOptions userOptions, DbConnectionInternal& connection)
    at Microsoft.Data.ProviderBase.DbConnectionFactory.TryGetConnection(DbConnection owningConnection, TaskCompletionSource`1 retry, DbConnectionOptions userOptions, DbConnectionInternal oldConnection, DbConnectionInternal& connection)
    at Microsoft.Data.ProviderBase.DbConnectionInternal.TryOpenConnectionInternal(DbConnection outerConnection, DbConnectionFactory connectionFactory, TaskCompletionSource`1 retry, DbConnectionOptions userOptions)
    at Microsoft.Data.ProviderBase.DbConnectionClosed.TryOpenConnection(DbConnection outerConnection, DbConnectionFactory connectionFactory, TaskCompletionSource`1 retry, DbConnectionOptions userOptions)
    at Microsoft.Data.SqlClient.SqlConnection.TryOpen(TaskCompletionSource`1 retry, SqlConnectionOverrides overrides)
    at Microsoft.Data.SqlClient.SqlConnection.Open(SqlConnectionOverrides overrides)
    at Microsoft.Data.SqlClient.SqlConnection.Open()
    at Microsoft.EntityFrameworkCore.SqlServer.Storage.Internal.SqlServerConnection.OpenDbConnection(Boolean errorsExpected)
    at Microsoft.EntityFrameworkCore.Storage.RelationalConnection.OpenInternal(Boolean errorsExpected)
    at Microsoft.EntityFrameworkCore.Storage.RelationalConnection.Open(Boolean errorsExpected)
```



## 中間網路斷掉

```bash
fail: Microsoft.EntityFrameworkCore.Update[10000]
      An exception occurred in the database while saving changes for context type 'WTF.Console.Models.WebStoreDbContext'.
      Microsoft.EntityFrameworkCore.DbUpdateException: An error occurred while saving the entity changes. See the inner exception for details.
       ---> Microsoft.Data.SqlClient.SqlException (0x80131904): A transport-level error has occurred when receiving results from the server. (provider: Session Provider, error: 19 - Physical connection is not usable)
         at Microsoft.Data.SqlClient.SqlConnection.OnError(SqlException exception, Boolean breakConnection, Action`1 wrapCloseInAction)
         at Microsoft.Data.SqlClient.SqlInternalConnection.OnError(SqlException exception, Boolean breakConnection, Action`1 wrapCloseInAction)
         at Microsoft.Data.SqlClient.TdsParser.ThrowExceptionAndWarning(TdsParserStateObject stateObj, Boolean callerHasConnectionLock, Boolean asyncClose)
         at Microsoft.Data.SqlClient.TdsParserStateObject.ThrowExceptionAndWarning(Boolean callerHasConnectionLock, Boolean asyncClose)
         at Microsoft.Data.SqlClient.TdsParserStateObject.ReadSniError(TdsParserStateObject stateObj, UInt32 error)
         at Microsoft.Data.SqlClient.TdsParserStateObject.ReadSniSyncOverAsync()
         at Microsoft.Data.SqlClient.TdsParserStateObject.TryReadNetworkPacket()
         at Microsoft.Data.SqlClient.TdsParserStateObject.TryPrepareBuffer()
         at Microsoft.Data.SqlClient.TdsParserStateObject.TryReadByte(Byte& value)
         at Microsoft.Data.SqlClient.TdsParser.TryRun(RunBehavior runBehavior, SqlCommand cmdHandler, SqlDataReader dataStream, BulkCopySimpleResultSet bulkCopyHandler, TdsParserStateObject stateObj, Boolean& dataReady)
         at Microsoft.Data.SqlClient.SqlDataReader.TryConsumeMetaData()
         at Microsoft.Data.SqlClient.SqlDataReader.get_MetaData()
         at Microsoft.Data.SqlClient.SqlCommand.FinishExecuteReader(SqlDataReader ds, RunBehavior runBehavior, String resetOptionsString, Boolean isInternal, Boolean forDescribeParameterEncryption, Boolean shouldCacheForAlwaysEncrypted)
```


## 在 savechangrs 前 complete


```bash
fail: Microsoft.EntityFrameworkCore.Update[10000]
      An exception occurred in the database while saving changes for context type 'WTF.Console.Models.WebStoreDbContext'.
      System.InvalidOperationException: The current TransactionScope is already complete.
         at System.Transactions.Transaction.get_Current()
         at Microsoft.EntityFrameworkCore.Storage.RelationalConnection.get_CurrentAmbientTransaction()
         at Microsoft.EntityFrameworkCore.Update.Internal.BatchExecutor.Execute(IEnumerable`1 commandBatches, IRelationalConnection connection)
         at Microsoft.EntityFrameworkCore.Storage.RelationalDatabase.SaveChanges(IList`1 entries)
         at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.SaveChanges(IList`1 entriesToSave)
         at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.SaveChanges(StateManager stateManager, Boolean acceptAllChangesOnSuccess)
         at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.<>c.<SaveChanges>b__112_0(DbContext _, ValueTuple`2 t)
         at Microsoft.EntityFrameworkCore.SqlServer.Storage.Internal.SqlServerExecutionStrategy.Execute[TState,TResult](TState state, Func`3 operation, Func`3 verifySucceeded)
         at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.SaveChanges(Boolean acceptAllChangesOnSuccess)
         at Microsoft.EntityFrameworkCore.DbContext.SaveChanges(Boolean acceptAllChangesOnSuccess)
      System.InvalidOperationException: The current TransactionScope is already complete.
         at System.Transactions.Transaction.get_Current()
         at Microsoft.EntityFrameworkCore.Storage.RelationalConnection.get_CurrentAmbientTransaction()
         at Microsoft.EntityFrameworkCore.Update.Internal.BatchExecutor.Execute(IEnumerable`1 commandBatches, IRelationalConnection connection)
         at Microsoft.EntityFrameworkCore.Storage.RelationalDatabase.SaveChanges(IList`1 entries)
         at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.SaveChanges(IList`1 entriesToSave)
         at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.SaveChanges(StateManager stateManager, Boolean acceptAllChangesOnSuccess)
         at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.<>c.<SaveChanges>b__112_0(DbContext _, ValueTuple`2 t)
         at Microsoft.EntityFrameworkCore.SqlServer.Storage.Internal.SqlServerExecutionStrategy.Execute[TState,TResult](TState state, Func`3 operation, Func`3 verifySucceeded)
         at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.SaveChanges(Boolean acceptAllChangesOnSuccess)
         at Microsoft.EntityFrameworkCore.DbContext.SaveChanges(Boolean acceptAllChangesOnSuccess)
```


<!-- endtab -->


{% endtabs %}