---
title: Encapsulation
date: 2025-09-15 09:24:03
categories: OO
top_img: https://i.imgur.com/LF7L1Ws.png
cover : https://i.imgur.com/LF7L1Ws.png
tags:
    - OO
toc:
toc_number:
comments :
---

{% tabs Encapsulation%}

![Encapsulation](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/landing.png)

{% btn 'https://igouist.github.io/post/2020/07/oo-3-encapsulation/',菜雞與物件導向 (3): 封裝,far fa-hand-point-right %}

{% btn 'https://www.reddit.com/r/csharp/comments/1jrmexy/understanding_encapsulation_benefits_of/',Understanding encapsulation benefits of properties in C#,far fa-hand-point-right %}

<!-- tab 🪵 降低耦合-->

所謂「降低耦合」，就是讓物件的使用者能很直覺地透過公開的方法去操作，而不用關心內部的細節。這麼做有兩個好處

- 使用者能專注在更高層次的抽象，而不是陷在物件內部的實作

![Encapsulation](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/no_gear_but_switch.png?raw=true)

- 後續如果內部邏輯要修改，開發者比較不用擔心會影響到外部程式

![Encapsulation](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/2_layer.png)

<!-- endtab -->

<!-- tab 購物車-->

```csharp
public class ShoppingCart
{
    public List<string> Items = new List<string>();
}

var cart = new ShoppingCart();
cart.Items.Add("iPhone");
cart.Items.Clear(); // 嘿嘿，使用者直接把購物車清空
```

如果直接把集合公開，外部程式就能為所欲為：

隨便新增重複的商品、甚至一次把所有商品刪光，這樣一來，我們完全無法控制購物車的正確狀態

![cart](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/cart.png)


透過封裝，我們只提供必要的方法
```csharp
public class ShoppingCart
{
    private readonly List<string> items = new();

    public void AddItem(string item)
    {
        if (!items.Contains(item))
            items.Add(item);
    }

    public IReadOnlyList<string> GetItems() => items;
}
```

現在使用者只能透過 AddItem() 加入商品，我們就能保證「不會有重複」。未來如果還要加上「庫存檢查」或「折扣券驗證」，只要在 AddItem() 裡擴充就好，外部程式碼完全不用改

![cart](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/cart_refactor.png)


<!-- endtab -->

<!-- tab 維護物件的不變性-遊戲角色-->

封裝的核心是「維護物件的不變性」，確保物件在任何時刻都處於合法狀態

```csharp
public class Player
{
    public int Health;
}

var p = new Player { Health = 100 };
p.Health -= 500; // 外部可以直接把血調成 -400
```

如果血量是公開欄位，外部程式隨便一改就能把角色狀態弄壞，變得不合法

![player](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/player1.png)


我們把欄位封裝起來，提供合理的操作介面
```csharp
public class Player
{
    private int health;
    public int Health => health; // 只讀
    public Player(int initialHealth) => health = initialHealth;
    public void TakeDamage(int damage)
    {
        if (damage < 0) throw new ArgumentException("傷害不能小於 0");
        health = Math.Max(0, health - damage); // 保證血量不會小於 0
    }
}
```
現在，外部程式不能隨便改血量，只能透過 TakeDamage()。這樣我們就能確保遊戲角色永遠維持「健康 ≥ 0」的狀態

![player](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/player_health_check.png)

我們希望將控制權掌握在物件本身。也就是說封裝的關鍵在於

- 限制外部能做的事情（避免直接操作內部狀態）
- 提供抽象、穩定的操作界面（集中邏輯，方便維護與擴充）


<!-- endtab -->

<!-- tab Property-->


在 C# 裡，屬性（Property）表面上看起來跟欄位（Field）很像，但本質上卻是由 get 與 set 方法所構成的存取器。這兩個方法扮演的是「資料進出的守門人」

- set → 控制資料寫入物件時的規則（例如檢查長度、範圍、格式）
- get → 控制資料讀取出物件時的呈現方式（例如格式化、運算、遮罩處理）

這樣做的目的，不僅僅是「把邏輯塞進去」，更是確保 資料的一致性 與 維護的集中化

![get](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/property_as_guard.png)

例如一個薪資管理系統
```csharp
public class Employee
{
    public decimal Salary; // 直接公開
}
```

需求是這樣子，公司規定薪資顯示時必須四捨五入到整數

如果是公開欄位 → 每個顯示 Salary 的地方都得手動 Math.Round(emp.Salary, 0)。系統中只要有 50 個頁面用到薪資，就得改 50 次!


如果我們使用 property 的話
```csharp
public class Employee
{
    private decimal salary;
    public decimal Salary
    {
        set { salary = value; }
        get { return Math.Round(salary, 0); }
    }
}
```

- 只需要在 get 改一次 → 所有使用薪資的地方自動符合規則
- 封裝確保「顯示薪資 = 四捨五入」的邏輯集中在一處

![salary](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/salary.png)


<!-- endtab -->


<!-- tab Property and Field-->


![Property](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/wall.png)

在實際開發中，為什麼大家習慣用 Property 而不是 Field？我們可以從幾個角度來看。

### 編譯後的差異

- Field：直接對類別的記憶體位置進行讀寫，編譯器生成的就是單純的讀取／寫入指令。
- Property：實際上會被編譯成 get_Xxx() / set_Xxx(value) 兩個方法，程式呼叫時是在執行方法。

也就是說，語法上看起來一樣 (obj.Name)，但在 IL（中繼語言）層級，兩者完全不同。

這帶來一個影響：如果某個 DLL 一開始提供的是 field，後來改成 property，呼叫端舊程式就會找不到那個欄位，必須重新編譯才行。在早期的 .NET Framework 時代，常見的情境是「直接替換 DLL 升級，不重新編譯主程式」。這種情況下，field → property 的改動會導致 相容性問題。
而在現代的 Web App 環境，每次部署都會完整重新 build，因此這個問題就沒那麼嚴重了


![Field](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/private_in_memory.png)

<br>

### 工具與框架支援

另一個更實際的理由是：大部分主流框架都支援 Property，而不是 Field。

**JSON Serializer (System.Text.Json / Newtonsoft.Json)**

```csharp
using System.Text.Json;

public class UserWithProperty
{
    public string Name { get; set; }
}

public class UserWithField
{
    public string Name;
}

var obj1 = new UserWithProperty { Name = "Allen" };
var obj2 = new UserWithField { Name = "Allen" };

string json1 = JsonSerializer.Serialize(obj1);
string json2 = JsonSerializer.Serialize(obj2);

Console.WriteLine(json1); // {"Name":"Allen"}
Console.WriteLine(json2); // {}
```

預設情況下，System.Text.Json 只會序列化 public property，不會處理 field。如果用 field，輸出的 JSON 是 {}，資料就丟失了。(Newtonsoft.Json 預設也只處理 property，要啟用 [JsonProperty] 才能序列化 field。)

![JsonProperty](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/json.png)


**Entity Framework (EF Core)**
```csharp
public class ProductWithProperty
{
    public int Id { get; set; }  // EF 需要 key
    public string Name { get; set; }
}

public class ProductWithField
{
    public int Id;      // EF 預設不會當作 key
    public string Name; // EF 預設不會 mapping
}
```

EF Core 預設會把 public property 當作 column mapping，但 public field 不會被自動 mapping（除非你用 Fluent API 指定 builder.HasField("_name")）。結果就是你的 Name 根本不會存進資料庫。


因此，以 Property 來說，主流框架（JSON、EF、WPF、ASP.NET）都支援。而 Field = 預設不會被處理，必須額外寫設定、Attribute、Fluent API。所以 convention 上大家才會說：「公開的東西就用 Property，Field 永遠 private」

![EF](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/OO/ef.png)

<!-- endtab -->

{% endtabs %}