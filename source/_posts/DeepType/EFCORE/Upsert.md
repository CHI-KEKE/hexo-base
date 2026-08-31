---
title: Upsert
date: 2025-10-17 15:45:05
categories: 資料疆界的航圖
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/UpSert/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/UpSert/1_landing.png
tags:
    - EF CORE
toc:
toc_number:
comments :
---

{% btn 'https://blog.darkthread.net/blog/ef-core-upsert/',EF Core 新增或更新資料 (UPSERT) 的簡便寫法,far fa-hand-point-right %}

{% tabs Upsert%}

<!-- tab Upsert-->

這是一個非常典型的需求：  

> 傳入一筆資料，用 Primary Key（或唯一欄位）去比對資料庫。  
> - 若不存在 → 新增  
> - 若存在 → 更新欄位  

也就是所謂的 **Upsert（Update + Insert）**



## 會員/客戶資料整併

會員/客戶資料整併（CRM / User Profile）時，第三方登入回傳一組 ExternalId（Google/Facebook/LINE），系統收到最新暱稱、頭像、Email

- 沒有這個 ExternalId → 新增會員
- 已存在 → 更新最新資料（頭像、名稱、聯絡資訊）



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/UpSert/2_crm_data.png)


<!-- endtab -->

<!-- tab 傳統寫法的痛點-->


```csharp
existing.PropA = record.PropA;
existing.PropB = record.PropB;
existing.PropC = record.PropC;
// ...
```

問題是當 Entity 有三十個欄位時，這樣的程式碼不僅冗長、容易漏改，也很難維護。這種情況下，我們就該讓 EF Core 幫你「批次更新」屬性!


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/UpSert/3_too_many_properties.png)


## EF Core 的解法：CurrentValues.SetValues()


EF Core 提供了一個非常實用的 API
```csharp
_dbContext.Entry(existing).CurrentValues.SetValues(record);
```

這行的意思是把 record 物件中的所有屬性值，直接複製到 existing 這個追蹤中的實體上



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/UpSert/3_currentvalues_setvalues.png)


<!-- endtab -->


<!-- tab Entry-->

Entry 的型別是 EntityEntry<T>（或 EntityEntry），你可以把它想成 EF Core 對「某一個實體」的黑盒子控台，它記著這個實體現在的「狀態」、每個屬性的「目前值 / 原始值」、有哪些導覽屬性 (Navigation Properties)，以及哪些屬性被標記成「有修改」
```csharp
var entry = _dbContext.Entry(existing);
```
其中 existing 必須是被追蹤中的實體（通常你用 FirstOrDefaultAsync 查回來、而且沒有 AsNoTracking()，它就會被追蹤）



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/UpSert/4_entry.png)



可以從 entry 讀到很多重要資訊

- `entry.State`：Added/Unchanged/Modified/Deleted/Detached
- `entry.OriginalValues`：第一次從資料庫讀回來時的快照
- `entry.CurrentValues`：目前記錄在 Change Tracker 裡的屬性值（通常等於你物件上的值）
- `entry.Properties`：每個屬性的追蹤項目，可控制 IsModified 等旗標
- `entry.References` / `entry.Collections`：導覽屬性與集合

也就是說，Entry 是 Change Tracker 對單一實體的「狀態面板」


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/UpSert/4_entry_dashboard.png)




<!-- endtab -->


<!-- tab SetValues 在做什麼-->

把 source 物件裡「名稱相同」的屬性值，複製到 entry 目前追蹤的實體上，它只比對同名屬性：source 的屬性名稱必須和目標 Entity 的屬性名稱相同才會被複製（型別需可指派 / 相容），source 可以是同型別的實體、DTO、甚至匿名型別，但只處理純量屬性，不會遞迴處理導覽屬性（關聯實體）或集合。這避免了不小心把關聯整棵樹都覆寫，對 主鍵 / 唯讀 / 計算欄位的複製會受限。像 Identity 主鍵是不允許修改的，若硬改會拋例外



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/UpSert/5_properties_match_name.png)



他的修改偵測，就是把值塞進 CurrentValues 後，EF Core 會比較 OriginalValues 與 CurrentValues，把不同的屬性標記為 IsModified = true，SaveChanges() 時，EF 就會針對「被標記修改」的欄位產生 UPDATE。

SetValues = 用名字對齊、批量貼上屬性值 → 自動幫你標記「哪些欄位改了」


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/UpSert/6_property_matched.png)



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/UpSert/7_set_values_not_do.png)



## 🌊 實際範例：Upsert 方法

以下範例展示如何以「唯一名稱」進行新增或更新
```csharp
public async Task UpsertRD()
{
    // 模擬一筆新資料
    var addallen = new Rd()
    {
        RdName = "AllenLin001",
        DeptCode = "R1001",
    };

    // 查詢是否已存在同名資料
    var allen = await _dbContext.Rds.FirstOrDefaultAsync(rd => rd.RdName == "AllenLin001");

    if (allen == null)
    {
        // 不存在 → 新增
        await _dbContext.AddAsync(addallen);
    }
    else
    {
        // 存在 → 更新
        addallen.RdId = allen.RdId; // 避免更動自動編號欄位
        _dbContext.Entry(allen).CurrentValues.SetValues(addallen);
    }

    await _dbContext.SaveChangesAsync();
}
```


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/UpSert/8_upsert_code.png)



## 🌊 為什麼要設定 addallen.RdId = allen.RdId;


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/UpSert/9_align_ids.png)



假設 Id 欄位是資料庫自動遞增的主鍵（Identity），如果直接 SetValues()，EF Core 會嘗試將 record 裡的 Id 值也覆蓋掉，這樣會讓追蹤中的實體被誤認為「新的物件」，而導致錯誤 `The property 'RdId' is part of a key and cannot be modified.`

因此我們要手動設定
```csharp
addallen.RdId = allen.RdId;
```

讓兩邊的主鍵一致，EF Core 才能判定這筆資料是要更新 (Update)，而不是新增 (Insert)

```csharp

var scope = services.CreateScope();
var adRepository = scope.ServiceProvider.GetRequiredService<AdventureRepository>();
await adRepository.UpsertRD();

public async Task UpsertRD()
{
    //// 統一使用非同步 api
    var addallen = new Rd()
    {
        RdName = "AllenLin001",
        DeptCode = "R1001",
    };
    var allen = await _dbContext.Rds.FirstOrDefaultAsync(rd => rd.RdName == "AllenLin");
    if (allen == null)
    {
        await _dbContext.AddAsync(addallen);
    }
    else
    {
        addallen.RdId = allen.RdId;
        _dbContext.Entry(allen).CurrentValues.SetValues(addallen);
    }

    await _dbContext.SaveChangesAsync();
}
```


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/UpSert/9_identity_conflict.png)


<!-- endtab -->


<!-- tab summary-->


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/UpSert/10_table.png)


<!-- endtab -->


{% endtabs %}
