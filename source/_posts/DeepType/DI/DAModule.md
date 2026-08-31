---
title: Autofac - 舊框架注入使用案例
date: 2026-05-03 08:26:05
categories: 注入之森
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/DAModule/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/DAModule/1_landing.png
tags:
    - Dependency Injection
toc:
toc_number:
comments :
---

{% tabs Autofac%}

<!-- tab Autofac-->

今天我們的研究專案是是背景 Worker，它不像 Web 專案有 ASP.NET 內建 DI 容器，因此需要自己建立，Autofac 是當時 .NET Framework 生態系中最成熟的選擇



![helper](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/DAModule/2_why_autofac.png)



那他怎麼管理？完整生命週期是這樣

```bash
程式啟動
    ↓
BatchUploadProcess.InitContainerBuilder()
    ↓
new ContainerBuilder()
    ↓
builder.RegisterModule(new DAModule())  ← 每個 DB 各一個
    ↓
builder.RegisterModule(new ServiceModule()) ← BL 層
    ↓
builder.Build() → IContainer
    ↓
每次 Task 執行時
container.BeginLifetimeScope() → ILifetimeScope
    ↓
scope.Resolve<ISalesOrderDataExportService>()
    → Autofac 自動注入 ISalesOrderGroupRepository
    → 自動注入 ERPDBEntitiesV2（ReadOnly or RW）
```


![helper](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/DAModule/3_builder_manual.png)




例如某個 BatchUploadProcess.cs 組裝順序是這樣

```csharp
// DA 層（資料庫）
builder.RegisterModule(new NineYi.ERP.DA.ERPDBV2.Modules.DAModule());
builder.RegisterModule(new NineYi.ERP.DA.ERPReadDBV2.Modules.DAModule());
builder.RegisterModule(new NineYi.ERP.DA.ERPDBAR.Modules.DAModule());
builder.RegisterModule(new NineYi.WebStore.DA.WebStoreDBV2.Modules.DAModule());
builder.RegisterModule(new NineYi.CRM.DA.CRMDB.Modules.DAModule());
 
// BL 層（商業邏輯）
builder.RegisterModule(new NineYi.ERP.Backend.BLV2.Modules.ServiceModule());
builder.RegisterModule(new NineYi.SCM.Frontend.BLV2.Modules.ServiceModule());
 
// 工具
builder.RegisterModule<NLogLoggerAutofacModule>();  // Logger
builder.RegisterModule(new ValidatorModule());       // 驗證
builder.RegisterModule(new MapperModule());          // AutoMapper
```

Autofac 就是這個專案的「組裝說明書」

- **DAModule** 定義「怎麼建立資料庫連線」
- **ServiceModule** 定義「哪個 Service 用哪個 Repository」
- **InitContainerBuilder()** 把所有說明書組合起來
- Task 執行時直接從容器拿現成的物件，不用自己 new


只要介面有被 `AsImplementedInterfaces()` 掃進去，建構子直接宣告就能注入


![helper](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/DAModule/4_as_implemented_interface.png)


<!-- endtab -->


<!-- tab EFHook-->


![helper](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/DAModule/5_ef_hooks.png)


EFHooks 是一個 EF 的攔截機制，在 **SaveChanges()** 前後自動執行額外邏輯， AOP（面向切面）的概念，Hook = 讓每個 Repository 不用重複寫 CreateTime = DateTime.Now 這類 boilerplate，統一由 EFHooks 在 SaveChanges 時自動處理，Hook 是 `As<IPreActionHook>()` → `SaveChanges` 時自動觸發，開發者不需要呼叫，例如以下三個 Hook


## UpdateSystemInfoPreInsertHook

INSERT 前自動填系統欄位，例如新增一筆資料時，自動設定

- CreateBy / CreateTime
- UpdateBy / UpdateTime


不需要每個 Repository 手動 set，統一由 Hook 處理


## UpdateSystemInfoPreUpdateHook

UPDATE 前自動更新系統欄位，例如修改一筆資料時，自動更新

- UpdateBy / UpdateTime


## ReplaceDeleteByValidFlagPreDeleteHook

若有開發者實作 DELETE 時系統會改成「軟刪除」，也就是攔截 DELETE 操作，把 `DELETE FROM table WHERE id = X` 偷換成 `UPDATE table SET ValidFlag = 0 WHERE id = X` 確保資料不真的被刪掉，只是標記為無效



![helper](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/DAModule/6_hook_types.png)



## WebStoreDBV2 多兩個 Hook

```csharp
builder.RegisterType<InsertIOPreInsertHook>().As<IPreActionHook>();
builder.RegisterType<InsertIOPreUpdateHook>().As<IPreActionHook>();
```


這兩個是 WebStore 特有的，INSERT/UPDATE 時會額外記錄 IO log（操作紀錄），用於稽核追蹤



![helper](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/DAModule/7_special_hook_auditlog.png)




<!-- endtab -->


<!-- tab ReadWrite / ReadOnly-->

ERP vs ERP.ReadOnly 是兩台不同的 SQL Server

```csharp
// ReadWrite → 主庫
builder.RegisterType<ERPDBEntitiesV2>()
    .WithParameter(... GetConnectionString("ERP"))
    .InstancePerDependency();

// ReadOnly → AlwaysOn 唯讀副本
builder.RegisterType<ERPDBEntitiesV2>()
    .WithParameter(... GetConnectionString("ERP.ReadOnly"))
    .Keyed<ERPDBEntitiesV2>("ReadOnly")
    .InstancePerDependency();

```


兩者的差異

```bash
┌───────┬───────────────────┬──────────────────────────────┐
│       │ ERP（ReadWrite）  │ ERP.ReadOnly                 │
├───────┼───────────────────┼──────────────────────────────┤
│ 對應  │ 主庫（Primary）   │ AlwaysOn 副本（Secondary）   │
│ 機器  │                   │                              │
├───────┼───────────────────┼──────────────────────────────┤
│ 讀/寫 │ 可讀可寫          │ 只能讀                       │
├───────┼───────────────────┼──────────────────────────────┤
│ Resol │ scope.Resolve<ERP │ scope.ResolveNamed<ERPDBEnti │
│ ve    │ DBEntitiesV2>()   │ tiesV2>("ReadOnly")          │
│ 方式  │                   │                              │
├───────┼───────────────────┼──────────────────────────────┤
│ 用途  │ INSERT / UPDATE / │ 查詢（報表、匯出）           │
│       │ DELETE            │                              │
├───────┼───────────────────┼──────────────────────────────┤
│ 負載  │ 主庫承擔寫入壓力  │ 副本分擔讀取流量             │
└───────┴───────────────────┴──────────────────────────────┘
```

這樣設計是為了分散資料庫負載，像是訂單匯出這種「大量 SELECT」的操作如果打到主庫，會競爭寫入的 lock，影響正常下單。因此寫入（下訂單、更新狀態）打主庫，而讀取（匯出報表、查詢清單）打 `AlwaysOn 副本`，`ConnectionStringHelper.GetConnectionString("ERP.ReadOnly")` 實際上就是讀取 config 裡的另一組連線字串，指向不同的 SQL Server IP



![helper](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/DAModule/8_readonly.png)



在這個專案的設計是，有沒有 `.Keyed<T>("ReadOnly")` 可用來決定要怎麼 Resolve

```csharp
// DAModule 裡兩種寫法

// 無 Key → 用 Resolve<T>()
builder.RegisterType<ERPDBEntitiesV2>()
    .WithParameter(...GetConnectionString("ERP"))
    .InstancePerDependency();

// 有 Key → 用 ResolveNamed<T>("ReadOnly")
builder.RegisterType<ERPDBEntitiesV2>()
    .WithParameter(...GetConnectionString("ERP.ReadOnly"))
    .Keyed<ERPDBEntitiesV2>("ReadOnly")
    .InstancePerDependency();

```

使用時在 Repository 建構子宣告時加 [KeyFilter]

```csharp
// 一般 ReadWrite（預設）
public SalesOrderGroupRepository(ERPDBEntitiesV2 context)
{ ... }

// 指定 ReadOnly
public SalesOrderGroupRepository(
    [KeyFilter("ReadOnly")] ERPDBEntitiesV2 context)
{ ... }

// 或在 scope 直接 Resolve
using (ERPDBEntitiesV2 ctx = this._lifetimeScope.ResolveNamed<ERPDBEntitiesV2>("ReadOnly"))
```

怎麼判斷某個 DB 支不支援 ReadOnly？看 DAModule 有沒有 `.Keyed<T>("ReadOnly")`




![helper](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/DAModule/9_key_filtered.png)



<!-- endtab -->


<!-- tab 一個整合範例-->


```csharp
// 從 DAModule 設定，可以推論出下面這段 Repository 合法
public class SalesOrderGroupRepository : ISalesOrderGroupRepository
{
    private readonly ERPDBEntitiesV2 _context;

    // ← 因為 DAModule 有 RegisterAssemblyTypes + AsImplementedInterfaces 所以 Autofac 知道要注入這個 Repository
    // ← 沒有 KeyFilter = 拿 ReadWrite 主庫連線
    public SalesOrderGroupRepository(ERPDBEntitiesV2 context)
    {
        _context = context; // ← SaveChanges 時 Hook 自動填 CreateTime、做軟刪除
    }
}
```

看 `DAModule` 的三件事

→ `RegisterAssemblyTypes`（能否注入）
→ `RegisterType<Hook>`（有無攔截）
→ `.Keyed`（有無 ReadOnly）

這三件事決定了在 Repository 裡怎麼寫建構子

<!-- endtab -->


<!-- tab summary-->


![helper](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/DAModule/daModule.png)


![helper](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/DependencyInjection/DAModule/11_final.png)


<!-- endtab -->


{% endtabs %}