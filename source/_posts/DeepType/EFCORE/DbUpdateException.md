---
title: EF Core - DbUpdateException
date: 2025-05-28 08:26:05
categories: 資料疆界的航圖
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/DBUpdateException/1__red_flag.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/DBUpdateException/1__red_flag.png
tags:
    - EF CORE
toc:
toc_number:
comments :
---

{% tabs DbUpdateException%}

<!-- tab hidden reef-->

你是否曾在 SaveChanges() 時迎來晴天霹靂，一行紅字 DbUpdateException 打斷了你的程式？這就像在大海航行時，突然撞上一座看不見的暗礁──平時無聲無息，一出事卻攪得你天翻地覆。這篇文章，我們就來一起潛入這片資料海域，探查六種常見導致 DbUpdateException 的暗礁地形。每一種都有它的風險與成因，但只要理解清楚，就能順利避開，繼續駛向資料的彼岸。


![r](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/DBUpdateException/1__red_flag.png)


![r](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/DBUpdateException/1_six_part.png)


<!-- endtab -->

<!-- tab 主鍵（Primary Key）衝突，一碼歸一人-->

主鍵（Primary Key）是資料的身份證號。若你試圖插入一筆資料，但主鍵與資料庫中已存在的資料相同，那就如同系統收到兩張一模一樣的身分證，無法分辨誰是誰，只能報錯。

```CSHARP
context.Users.Add(new User { Id = 1 }); // Id = 1 已存在
context.SaveChanges(); // boom!
```

## 常見原因

- 手動指定主鍵值，未使用自動遞增
- 測試資料未清空


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/DBUpdateException/2_primary_key.png)


<!-- endtab -->

<!-- tab 外鍵（Foreign Key）違反，找不到對應人-->

外鍵（Foreign Key）是資料之間的橋樑。若你新增一筆訂單，卻引用了一位不存在的客戶，那就像寄信到一個虛構地址──沒有收件人，自然寄不出去

```CSHARP
context.Orders.Add(new Order { CustomerId = 999 }); // Customers 表沒有 Id=999
context.SaveChanges(); // 報錯
```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/DBUpdateException/3_foreign_key.png)


<!-- endtab -->

<!-- tab 必填欄位（NOT NULL）為 null : 不該留白的愛-->

有些欄位規定不可為 null，就像是報名表不能漏填姓名。若你在儲存時忘記提供資料庫期待的欄位值，它會用最嚴厲的方式提醒你：「打妹！」

```CSHARP
context.Products.Add(new Product { Name = null }); // Name 是 NOT NULL 欄位
context.SaveChanges(); // 觸發錯誤
```


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/DBUpdateException/4_not_null.png)


<!-- endtab -->

<!-- tab 欄位長度限制超過（MaxLength/Size） : 你話太多了-->

若欄位被設定為 nvarchar(50)，但你給了超過 50 字元的字串，那就像在行李箱塞入一整隻鯨魚──太長，裝不下，自然塞不進去。

```CSHARP
product.Description = new string('A', 1000); // 限制為 255 字，實際給了 1000
context.SaveChanges(); // 出錯
```

##　作法

- 使用 MaxLength 屬性限制輸入
- 在 UI 端或伺服端做前置驗證


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/DBUpdateException/5_max_len.png)



<!-- endtab -->

<!-- tab 資料類型不相符 : 能飛不代表能跑-->

資料庫欄位有明確型別，你給的值若型別錯誤，轉換失敗就會直接炸開。例如欄位預期是數字，你卻給了一串無法解析的字串。

```CSHARP
order.TotalAmount = Convert.ToDecimal("abc"); // 無法轉換為 decimal
context.SaveChanges(); // boom again!
```

##　作法

- 加強轉型檢查
- 使用 TryParse 等方式進行防禦型處理


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/DBUpdateException/6_type.png)



<!-- endtab -->

<!-- tab 違反唯一約束（Unique Constraint） : 這世界只容一個你-->


唯一約束（Unique Constraint）就像是資料世界的排他戀愛：Email 是唯一的，不能同時給兩個人。若你試圖插入一筆已存在的 Email，資料庫會毫不猶豫地拒絕這段三角戀。

```CSHARP
context.Users.Add(new User { Email = "existing@email.com" }); // 已存在
context.SaveChanges(); // 再次出錯
```

- 儲存前先查詢是否存在相同值
- 或於資料庫層加上 Try-Catch 處理違反例外


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/DBUpdateException/7_uniq.png)



<!-- endtab -->

<!-- tab summary-->


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/DBUpdateException/8_final.png)


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/EF/DBUpdateException/dbupdate.png)


<!-- endtab -->


{% endtabs %}
