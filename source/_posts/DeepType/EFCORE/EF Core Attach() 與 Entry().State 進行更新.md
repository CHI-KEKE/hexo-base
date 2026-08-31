---
title: EF Core Attach() 與 Entry().State 進行更新
date: 2025-10-17 15:45:05
categories: 資料疆界的航圖
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/State/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/State/1_landing.png
tags:
    - EF CORE
toc:
toc_number:
comments :
---


{% btn 'https://blog.darkthread.net/blog/ef-core-attach/',使用 EF Core Attach() 與 Entry().State 進行更新,far fa-hand-point-right %}


{% tabs Attach%}

<!-- tab 小船-->

在資料的海洋裡，每一筆 Entity 都像一艘小船。有的剛啟航（Added），有的正返港（Modified），有的靜靜停泊（Unchanged），而有的早已脫離航道（Detached）。

EF Core 就像港口的管理員，它用 Entry() 觀察著每艘船的狀態，用 Attach() 告訴它：「這艘船其實還在港裡，只是暫時沒動而已。」



<!-- endtab -->

<!-- tab 什麼是 Entry()-->

在 EF Core 中，每個資料表對應一個「Entity 類別」，例如 Student
```csharp
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

有時候，我們想知道 EF Core 現在怎麼看這個物件，例如：它是不是新資料？是不是被修改過？是不是要刪掉？


這時候可以用
```csharp
var entry = context.Entry(student);
```

這行會回傳一個 EntityEntry 物件。可以想成：EF Core 給你一張「狀態說明書」，上面寫著這個 student 物件目前的狀態



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/State/1_manager.png)



<!-- endtab -->

<!-- tab Entity 的五種狀態 (EntityState)-->

entry.State 就是 EF Core 對這個物件的「態度」，常見有 5 種狀態

| 狀態            | 意思            | SaveChanges() 時會發生什麼事 |
| ------------- | ------------- | --------------------- |
| **Added**     | 這是新資料，還沒存進資料庫 | 會執行 `INSERT`          |
| **Modified**  | 資料有被修改過       | 會執行 `UPDATE`          |
| **Deleted**   | 要刪除這筆資料       | 會執行 `DELETE`          |
| **Unchanged** | 資料跟資料庫一樣，沒變   | 不動作                   |
| **Detached**  | 不被追蹤（EF 不理它）  | 不動作                   |


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/State/3_states.png)



<!-- endtab -->

<!-- tab Entry(record).State = EntityState.Modified 是什麼意思-->

這樣設定代表：「EF Core，這個物件已經被修改了，等一下請幫我 UPDATE。」

例如
```csharp
var student = new Student { Id = 1, Name = "小明" };
context.Entry(student).State = EntityState.Modified;
context.SaveChanges();
```

這樣就算你沒有真的改過 Name，EF Core 還是會執行，因為你直接告訴 EF：「整個物件都改了」，EF 只好「整筆更新」
```sql
UPDATE Students SET Name = '小明' WHERE Id = 1;
```


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/State/4_modified_update.png)


<!-- endtab -->

<!-- tab 如果只想更新部分欄位-->


```csharp
context.Entry(student).Property(x => x.Name).IsModified = true;
```

這樣 EF Core 只會更新 Name 欄位，不會動其他欄位。這就比較精確，也比較有效率


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/State/5_property_is_modified.png)


<!-- endtab -->

<!-- tab Attach(record) 是什麼-->


Attach() 是讓 EF Core 開始追蹤一個物件，但假設它「目前沒有被修改」

```csharp
context.Attach(student);
```

等同於

```csharp
context.Entry(student).State = EntityState.Unchanged;
```

EF Core 的想法是：「OK，我知道這個物件存在資料庫，而且它跟資料庫目前的資料是一樣的。」

所以當你呼叫 SaveChanges() 時，EF 不會做任何事


但如果你之後修改它
```csharp
student.Name = "小華";
context.SaveChanges();
```
這時 EF Core 就會知道「喔！Name 改了！」，於是執行
```sql
UPDATE Students SET Name = '小華' WHERE Id = 1;
```


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/State/6_attach_unchanged.png)


<!-- endtab -->

<!-- tab Primary Key 的狀況-->


如果你 Attach() 一個物件時，沒設定主鍵 (Primary Key)，EF Core 會以為它是「新資料」，狀態就會變成 Added，因為 EF Core 不知道它要對應哪一筆資料。
反之，如果主鍵有值，它會判定為「舊資料」，狀態是 Unchanged


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/State/7_primary_key_add_change.png)


<!-- endtab -->

<!-- tab 單純 Attach() 未指定 Id 值-->


```csharp
this._printColorThing("=== Exp1 Attach / 不指定 Id===", ConsoleColor.Magenta);
var newGenrd = _generateCuteRd();
_dbContext.Attach(newGenrd);
this._printColorThing("未指定自動跳號 Primary Key 時，State = \" + entEntry.State", ConsoleColor.Green);
await _dbContext.SaveChangesAsync();
```

EF Core 看到沒有主鍵 (Id) 值，會誤以為這是新資料，所以 record 的 State 會是 Added，SaveChanges() 時會執行 INSERT，須注意若資料表 Date 欄位有 UNIQUE 限制，而你插入的日期在資料庫裡已經存在，那就會違反 UNIQUE 限制，產生錯誤。

```bash
=== Exp1 Attach / 不指定 Id===
warn: Microsoft.EntityFrameworkCore.Model.Validation[10400]
      Sensitive data logging is enabled. Log entries and exception messages may include sensitive application data; this mode should only be enabled during development.
未指定自動跳號 Primary Key 時，State =  Added
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (20ms) [Parameters=[@p0='CUTE' (Size = 5), @p1='CuteRD' (Size = 100)], CommandType='Text', CommandTimeout='30']
      SET IMPLICIT_TRANSACTIONS OFF;
      SET NOCOUNT ON;
      INSERT INTO [RD] ([DeptCode], [RD_Name])
      OUTPUT INSERTED.[RD_ID]
      VALUES (@p0, @p1);
```



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/State/8_attach_conflict.png)


<!-- endtab -->


<!-- tab 先查一筆，再 Attach()-->


```csharp
this._printColorThing("=== 實驗二 Attach / 查詢過同一筆無法 Attach", ConsoleColor.Magenta);
var newGenrd2 = _generateCuteRd();
var lookupRd = await _dbContext.Rds.FirstAsync(rd => rd.RdName == newGenrd2.RdName);
newGenrd2.RdId = lookupRd.RdId;
_dbContext.Attach(newGenrd);
try
{
    _dbContext.Attach(newGenrd2);
}
catch (Exception ex)
{
    this._printColorThing(ex.ToString(), ConsoleColor.Red);
}
```


因為這筆資料已經被 dbCtx 查出來（已被追蹤），EF Core 的追蹤系統中已經有相同 Id 的實體存在，再 Attach() 一個相同 Id 的物件會出錯


```bash
=== 實驗二 Attach / 查詢過同一筆無法 Attach
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (2ms) [Parameters=[@__newGenrd2_RdName_0='CuteRD' (Size = 100)], CommandType='Text', CommandTimeout='30']
      SELECT TOP(1) [r].[RD_ID], [r].[DeptCode], [r].[RD_Name]
      FROM [RD] AS [r]
      WHERE [r].[RD_Name] = @__newGenrd2_RdName_0
System.InvalidOperationException: The instance of entity type 'Rd' cannot be tracked because another instance with the key value '{RdId: 1004}' is already being tracked. When attaching existing entities, ensure that only one entity instance with a given key value is attached.
   at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.IdentityMap`1.ThrowIdentityConflict(InternalEntityEntry entry)
   at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.IdentityMap`1.Add(TKey key, InternalEntityEntry entry, Boolean updateDuplicate)
   at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.IdentityMap`1.Add(TKey key, InternalEntityEntry entry)
   at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.IdentityMap`1.Add(InternalEntityEntry entry)
   at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.StateManager.StartTracking(InternalEntityEntry entry)
   at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.InternalEntityEntry.SetEntityState(EntityState oldState, EntityState newState, Boolean acceptChanges, Boolean modifyProperties)
   at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.InternalEntityEntry.SetEntityState(EntityState entityState, Boolean acceptChanges, Boolean modifyProperties, Nullable`1 forceStateWhenUnknownKey, Nullable`1 fallbackState)
   at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.EntityGraphAttacher.PaintAction(EntityEntryGraphNode`1 node)
   at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.EntityEntryGraphIterator.TraverseGraph[TState](EntityEntryGraphNode`1 node, Func`2 handleNode)
   at Microsoft.EntityFrameworkCore.ChangeTracking.Internal.EntityGraphAttacher.AttachGraph(InternalEntityEntry rootEntry, EntityState targetState, EntityState storeGeneratedWithKeySetTargetState, Boolean forceStateWhenUnknownKey)
   at Microsoft.EntityFrameworkCore.DbContext.SetEntityState(InternalEntityEntry entry, EntityState entityState)
   at Microsoft.EntityFrameworkCore.DbContext.SetEntityState[TEntity](TEntity entity, EntityState entityState)
   at Microsoft.EntityFrameworkCore.DbContext.Attach[TEntity](TEntity entity)
   at WTF.Console.Repository.AdventureRepository.AttchStateExp() in C:\91APP\AI_Devs\Random\nine1.wtf.did.i.code\WTF.Console\WTF.Console\Repository\AdventureRepository.cs:line 182
```


<!-- endtab -->

<!-- tab Attach() 指定 Id 值的 Entity-->

```csharp
int rdId;
this._printColorThing("=== 實驗二 Attach / 查詢過同一筆無法 Attach", ConsoleColor.Magenta);
var newGenrd2 = _generateCuteRd();
var lookupRd = await _dbContext.Rds.FirstOrDefaultAsync(rd => rd.RdName == newGenrd2.RdName);
rdId = lookupRd.RdId;
newGenrd2.RdId = lookupRd.RdId;
try
{
    _dbContext.Attach(newGenrd2);
}
catch (Exception ex)
{
    this._printColorThing(ex.ToString(), ConsoleColor.Red);
}

//// 想要新的 DbContext，需要透過新 Scope 建立，否則會有追蹤相同實體錯誤
using (var scope = _serviceProvider.CreateScope())
{
    var dbCtx = scope.ServiceProvider.GetRequiredService<AdventureWorks2022DbContext>();
    this._printColorThing("=== 實驗三 Attach / 指定 Id", ConsoleColor.Magenta);
    var newGenrd3 = _generateCuteRd();
    newGenrd3.RdId = rdId;
    var entry3 = dbCtx.Attach(newGenrd3);
    this._printColorThing($"未指定自動跳號 Primary Key 時，State = {entry3.State}", ConsoleColor.Green);
    await dbCtx.SaveChangesAsync();
}
```

```bash
=== 實驗三 Attach / 指定 Id
未指定自動跳號 Primary Key 時，State = Unchanged
```

因有設定 Id，EF Core 知道這是「舊資料」，所以狀態是 Unchanged，SaveChanges() 時什麼都不會發生，因為 EF 以為「資料跟資料庫一樣」



<!-- endtab -->

<!-- tab Attach() 後修改某屬性-->


```csharp
using (var scope = _serviceProvider.CreateScope())
{
    var dbCtx = scope.ServiceProvider.GetRequiredService<AdventureWorks2022DbContext>();
    this._printColorThing("=== 實驗三 Attach / 指定 Id", ConsoleColor.Magenta);
    var newGenrd3 = _generateCuteRd();
    newGenrd3.RdId = rdId;
    var entry3 = dbCtx.Attach(newGenrd3);
    this._printColorThing($"未指定自動跳號 Primary Key 時，State = {entry3.State}", ConsoleColor.Green);
    await dbCtx.SaveChangesAsync();

    this._printColorThing("=== 實驗四 Attach 後修改某欄位", ConsoleColor.Magenta);
    newGenrd3.RdName = "AllenIsGood";
    await dbCtx.SaveChangesAsync();
    this._printColorThing("更新異動了的欄位!", ConsoleColor.Green);
}
```

EF Core 的變更追蹤機制會偵測到屬性變更，自動把 State 從 Unchanged 改成 Modified，SaveChanges() 時只會執行
```sql
UPDATE RD SET RDName = 'AllenIsGood' WHERE Id = 1004;
```


```bash
=== 實驗四 Attach 後修改某欄位
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (10ms) [Parameters=[@p1='1004', @p0='AllenIsGood' (Size = 100)], CommandType='Text', CommandTimeout='30']
      SET IMPLICIT_TRANSACTIONS OFF;
      SET NOCOUNT ON;
      UPDATE [RD] SET [RD_Name] = @p0
      OUTPUT 1
      WHERE [RD_ID] = @p1;
```

**這是最正常、最安全的更新方式**


<!-- endtab -->


<!-- tab Entry().State = EntityState.Modified-->


```csharp
using (var scope = _serviceProvider.CreateScope())
{
    var dbCtx = scope.ServiceProvider.GetRequiredService<AdventureWorks2022DbContext>();
    this._printColorThing("=== 實驗五 Entry().State = EntityState.Modified", ConsoleColor.Magenta);
    var newGenrd4 = this._generateCuteRd();
    newGenrd4.RdId = rdId;
    dbCtx.Entry(newGenrd4).State = EntityState.Modified;
    await dbCtx.SaveChangesAsync();
    this._printColorThing("更新所有欄位，不管是否與原來相同", ConsoleColor.Green);
}
```

```sql
UPDATE [RD] SET [DeptCode] = @p0, [RD_Name] = @p1
OUTPUT 1
WHERE [RD_ID] = @p2;
```

```bash
=== 實驗五 Entry().State = EntityState.Modified
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (7ms) [Parameters=[@p2='1005', @p0='CUTE' (Size = 5), @p1='CuteRD' (Size = 100)], CommandType='Text', CommandTimeout='30']
      SET IMPLICIT_TRANSACTIONS OFF;
      SET NOCOUNT ON;
      UPDATE [RD] SET [DeptCode] = @p0, [RD_Name] = @p1
      OUTPUT 1
      WHERE [RD_ID] = @p2;
更新所有欄位，不管是否與原來相同
```

這樣做會把整個物件標示為「被修改」，即使只有一個欄位真的改了，EF Core 仍會執行


<!-- endtab -->

<!-- tab SetValues() 只更新有修改的欄位-->


```csharp
using (var scope = _serviceProvider.CreateScope())
{
    var dbCtx = scope.ServiceProvider.GetRequiredService<AdventureWorks2022DbContext>();
    this._printColorThing("=== 實驗六 SetValues()", ConsoleColor.Magenta);
    var newGenrd6 = this._generateCuteRd();
    var dbRd = dbCtx.Rds.Find(rdId);
    newGenrd6.RdId = rdId;
    dbCtx.Entry(dbRd).CurrentValues.SetValues(newGenrd6);
    await dbCtx.SaveChangesAsync();
    this._printColorThing("只會更新異動欄位", ConsoleColor.Green);
}
```

SetValues() 會把來源物件的屬性值貼到已追蹤的 newGenrd6，但它只會標示出真正不同的屬性為 Modified
結果就是只更新異動的欄位

```sql
UPDATE [RD] SET [RD_Name] = @p0
OUTPUT 1
WHERE [RD_ID] = @p1;
```


```bash
=== 實驗六 SetValues()
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (3ms) [Parameters=[@__p_0='1004'], CommandType='Text', CommandTimeout='30']
      SELECT TOP(1) [r].[RD_ID], [r].[DeptCode], [r].[RD_Name]
      FROM [RD] AS [r]
      WHERE [r].[RD_ID] = @__p_0
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (5ms) [Parameters=[@p1='1004', @p0='rrD6' (Size = 100)], CommandType='Text', CommandTimeout='30']
      SET IMPLICIT_TRANSACTIONS OFF;
      SET NOCOUNT ON;
      UPDATE [RD] SET [RD_Name] = @p0
      OUTPUT 1
      WHERE [RD_ID] = @p1;
只會更新異動欄位
```



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/State/9_setvalues.png)




<!-- endtab -->

<!-- tab SetValues() 的靈活應用-->


SetValues() 可以吃不同型別的來源，只要屬性名稱對得上

**✅ ViewModel (常見) 、✅ 匿名類別**
```csharp
using (var scope = _serviceProvider.CreateScope())
{
    var dbCtx = scope.ServiceProvider.GetRequiredService<AdventureWorks2022DbContext>();
    this._printColorThing("=== 實驗七 SetValues(任意物件)", ConsoleColor.Magenta);
    //var newGenrd6 = this._generateCuteRd();
    this._printColorThing($" rdId : {rdId}", ConsoleColor.DarkGreen);
    var dbRd = dbCtx.Rds.Find(rdId);
    this._printColorThing($" name : {dbRd.RdName}, deptCode : {dbRd.DeptCode}", ConsoleColor.DarkGreen);
    dbCtx.Entry(dbRd).CurrentValues.SetValues(new {
        RdName = "RandomName2",
        DeptCode = "RD123"
    });
    await dbCtx.SaveChangesAsync();
    this._printColorThing("只會更新異動欄位", ConsoleColor.Green);
}
```

```sql
UPDATE [RD] SET [DeptCode] = @p0, [RD_Name] = @p1
OUTPUT 1
WHERE [RD_ID] = @p2;
```

```bash
=== 實驗七 SetValues(任意物件)
 rdId : 1008
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (4ms) [Parameters=[@__p_0='1008'], CommandType='Text', CommandTimeout='30']
      SELECT TOP(1) [r].[RD_ID], [r].[DeptCode], [r].[RD_Name]
      FROM [RD] AS [r]
      WHERE [r].[RD_ID] = @__p_0
 name : CuteRD, deptCode : CUTE
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (9ms) [Parameters=[@p2='1008', @p0='RD123' (Size = 5), @p1='RandomName2' (Size = 100)], CommandType='Text', CommandTimeout='30']
      SET IMPLICIT_TRANSACTIONS OFF;
      SET NOCOUNT ON;
      UPDATE [RD] SET [DeptCode] = @p0, [RD_Name] = @p1
      OUTPUT 1
      WHERE [RD_ID] = @p2;
只會更新異動欄位
```

在此更新了 RdName、DeptCode，因為都與 dbRd 值有所不同



**✅ Dictionary<string, object>**
```csharp
using (var scope = _serviceProvider.CreateScope())
{
    var dbCtx = scope.ServiceProvider.GetRequiredService<AdventureWorks2022DbContext>();
    this._printColorThing("=== 實驗八 SetValues(Dictionary<string, object>) ===", ConsoleColor.Magenta);
    this._printColorThing($" rdId : {rdId}", ConsoleColor.DarkGreen);
    var dbRd = dbCtx.Rds.Find(rdId);
    this._printColorThing($" name : {dbRd.RdName}, deptCode : {dbRd.DeptCode}", ConsoleColor.DarkGreen);
    dbCtx.Entry(dbRd).CurrentValues.SetValues(new Dictionary<string,object>()
    {
        {"RdName", "RandomName3"},
        {"DeptCode", "RD456"}
    });
    await dbCtx.SaveChangesAsync();
    this._printColorThing("只會更新異動欄位", ConsoleColor.Green);
}
```

```sql
UPDATE [RD] SET [DeptCode] = @p0, [RD_Name] = @p1
OUTPUT 1
WHERE [RD_ID] = @p2;
```

```bash
=== 實驗八 SetValues(Dictionary<string, object>) ===
 rdId : 1009
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (3ms) [Parameters=[@__p_0='1009'], CommandType='Text', CommandTimeout='30']
      SELECT TOP(1) [r].[RD_ID], [r].[DeptCode], [r].[RD_Name]
      FROM [RD] AS [r]
      WHERE [r].[RD_ID] = @__p_0
 name : CuteRD, deptCode : CUTE
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (9ms) [Parameters=[@p2='1009', @p0='RD456' (Size = 5), @p1='RandomName3' (Size = 100)], CommandType='Text', CommandTimeout='30']
      SET IMPLICIT_TRANSACTIONS OFF;
      SET NOCOUNT ON;
      UPDATE [RD] SET [DeptCode] = @p0, [RD_Name] = @p1
      OUTPUT 1
      WHERE [RD_ID] = @p2;
只會更新異動欄位
```

EF Core 會比對屬性名稱 → 只更新對得上的欄位 → 只更新值真的不同的欄位!


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/State/9_2_different_source_setvalues.png)


<!-- endtab -->


<!-- tab summary-->

每個 Entity，其實都是我們在資料海上的分身。有時我們被追蹤（Tracked），有時選擇脫離（Detached）；有時整個重來（Modified），有時只是微調（Property.IsModified = true）。Attach() 教我們——不是所有變動都需要重新出發；Entry().State 告訴我們——理解狀態，比盲目更新更重要。

在日常開發中，懂得何時該「讓 EF Core 靜靜觀察」，何時該「主動標記變更」，就像在大海上航行時，懂得何時升帆、何時收錨


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/State/10_table.png)


<!-- endtab -->


{% endtabs %}