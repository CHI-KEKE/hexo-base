---
title: EF Core - DbUpdateConcurrencyException
date: 2025-05-28 08:26:05
categories: 資料疆界的航圖
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Concurrency/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Concurrency/1_landing.png
tags:
    - EF CORE
toc:
toc_number:
comments :
---


{% tabs DbUpdateConcurrencyException%}


<!-- tab Google 文件-->

想像你和同事一起編輯一份 Google 文件，你正專心致志地輸入「我們應該大力加薪」，正當你按下儲存，結果——對方也在同一秒按下儲存，而他打的是：「大家應該自願減薪來展現團隊精神。」

🙃 到底誰的才算？你打的血汗心聲，還是對方的職場邪教宣言？
１
這，就是所謂的 **並行更新（Concurrency）問題**。  
而 EF Core 的 `DbUpdateConcurrencyException` 就是在提醒你：「欸欸欸，有人先改過這筆資料囉！」


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Concurrency/1_google_doc.png)


<!-- endtab -->

<!-- tab Concurrency（並行）-->

`Concurrency` 指的是「多個流程同時存取並可能修改同一筆資料」的情況，這在多人協作系統中非常常見，例如：

- 多位使用者同時編輯同一筆訂單或資料。
- 他們都按下儲存。
- 如果系統沒有檢查誰先誰後，那麼後儲存的人將會直接覆蓋掉前面的變更。

這會帶來許多潛在問題：

- ✅ 使用者的更新被悄悄蓋掉，卻毫無察覺  
- ✅ 系統記錄出現矛盾，無法追蹤真實修改者  
- ✅ 使用者困惑：「剛剛不是明明有改成功嗎？」  

因此，**我們需要一種方法來判斷資料是否在此期間已經被人修改過**。這正是 `RowVersion` 發揮作用的時刻


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Concurrency/1_concurrency.png)


<!-- endtab -->

<!-- tab DbUpdateConcurrencyException-->

`DbUpdateConcurrencyException` 是 EF Core 在進行資料更新時，偵測到資料已經被其他人修改，**導致資料版本與預期不符時所拋出的例外**

這種情況發生的流程如下圖所示：
![時序圖](https://github.com/CHI-KEKE/pics/blob/main/Codesss/EF/Exception/DBConcurrencyException_image.png?raw=true)

1. 使用者 A 和 B 幾乎同時查詢一筆資料。
2. A 進行修改並儲存成功（資料版本發生改變）。
3. B 依然以為資料還是原樣，照舊修改並送出。
4. EF Core 在更新時發現：資料的版本與原本不一致，因此 **拒絕更新並拋出 `DbUpdateConcurrencyException`**！

這個機制可以有效避免資料被「靜靜地」覆蓋，是多人編輯時的重要安全網


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Concurrency/1_dbupdateconcurrencyexception.png)


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Concurrency/2_concurrency_timeline.png)


<!-- endtab -->

<!-- tab RowVersion：守門的指紋-->

解決並發問題的推薦做法是使用資料庫的 `RowVersion` 欄位


## ✅ RowVersion 是什麼？

`RowVersion` 是 SQL Server 提供的一種特殊欄位型別（又稱 `TIMESTAMP`，兩者是同義詞），每次資料列被修改，它就會自動更新。

你可以把它想像成「資料的指紋」，只要內容有變，這個指紋也會跟著變。EF Core 就會根據這個指紋來判斷資料是否遭到更動。


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Concurrency/3_finger_print.png)



## 如何啟用 RowVersion？

<br>

只需要三個步驟，你就能啟用這個防線


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Concurrency/4_rowversion_3_steps.png)


#### ✅ Step 1：在資料表中新增 RowVersion 欄位

```SQL
ALTER TABLE RD
ADD RowVersion ROWVERSION; -- 或 TIMESTAMP (它們是同義詞)
```

型別：rowversion（SQL Server 的特殊自動遞增型別）

系統會自動管理，無需手動指定


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Concurrency/4_2_step1_db.png)




#### ✅ Step 2：在 C# Model 加上屬性
```CSHARP
public class RD
{
    public int RD_ID { get; set; }

    public string RD_Name { get; set; }

    [Timestamp] // 👈 標示這是 Concurrency Token
    public byte[] RowVersion { get; set; }
}
```

或是用 Fluent API：

```CSHARP
modelBuilder.Entity<RD>()
    .Property(r => r.RowVersion)
    .IsRowVersion();
```


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Concurrency/4_3-entity.png)


#### ✅ Step 3：照常更新，EF Core 會自動比對 RowVersion
```CSHARP
var entity = await context.Rds.FindAsync(1);
entity.RD_Name = "更新後名稱";

await context.SaveChangesAsync(); // EF Core 會幫你檢查 RowVersion
```
若更新失敗（RowVersion 不符），就會觸發 DbUpdateConcurrencyException



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Concurrency/4-4_ef.png)


<!-- endtab -->

<!-- tab 那發生例外時該怎麼辦？-->

當 SaveChanges 拋出 DbUpdateConcurrencyException，你可以選擇以下幾種處理方式：

<br>

## ✅ 方案 1：提示使用者重新整理資料
```CSHARP
try
{
    await context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    // 通知使用者：資料已被更新，請重新整理頁面
    Console.WriteLine("資料已被其他人修改，請重新整理並再試一次。");
}
```


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Concurrency/5_1_catch.png)



## ✅ 方案 2：讓使用者選擇要覆蓋、放棄還是比對差異（進階）

這需要自訂 UI 和差異比對流程，適合複雜表單或編輯畫面


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Concurrency/5_2-ui.png)



<!-- endtab -->

<!-- tab summary-->


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Concurrency/6_scenerios.png)


就像 Google 文件會提醒你「有其他人正在編輯這份文件」，
EF Core 搭配 RowVersion 和 DbUpdateConcurrencyException 提供了類似的保護機制。

在以下情境中，這種機制幾乎是必備保險：

- 多人編輯後台資料
- 長時間打開但尚未儲存的表單
- 使用者希望保證資料「不被別人偷偷改掉」

✅ 加上 RowVersion 欄位
✅ Entity 加上 [Timestamp] 屬性
✅ 捕捉例外，友善提示使用者

只要三步，讓你的系統資料一致性與使用體驗大幅提升。讓每一筆使用者的修改，都能被好好保護與尊重


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Concurrency/7_sum.png)


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/Concurrency/concurrency.png)

<!-- endtab -->


{% endtabs %}

