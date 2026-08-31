---
title: Enum - 限制，是自由的基礎
date: 2024-04-29 22:45:10
categories: 程思舞想
top_img: https://i.imgur.com/Zly1UDM.png
cover : https://i.imgur.com/Zly1UDM.png
tags:
    - Enum
toc:
toc_number:
comments :
---

{% tabs Enum - 限制，是自由的基礎%}


<!-- tab Constraint-->

凡可列舉者，皆應被命名，用數字或字串記錄分類，像在沙地上寫字。風吹就散，誰都能改，誰都會迷路。
唯有列舉（Enum），能為混亂命名，能讓程式碼之神歸位，如編鐘之列、星辰之座。
有些人會說，Enum 限制了自由，但真相是：「沒有邊界的自由，終究會走向混亂。」

說到這裡，應該不難理解，Enum 的本質，可以減少混亂值、提升可讀性、強化型別安全，杜絕魔法數字與野生字串，我們會說，有結構的程式碼，是對未來同事最好的善意。
當你看到一個架構龐大的 C# 專案，結果一個收藏 Enum 的資料夾都沒有時，個人認為，隔天就可以提交離職信了(當天先去收個驚)

<br/>

<font color=#D3D3D3 style="font-size: 20px;">&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;Enum 用來代表「一組固定、有限、離散的值」，
 </font>

 <font color=#D3D3D3 style="font-size: 20px;">&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;讓我們的程式碼更有語意、更安全、更容易維護。
 </font>

<br/>


<!-- endtab -->


<!-- tab 處理訂單的狀態-->

處理訂單的狀態時（待處理、處理中、完成、取消），當我們 hardcode 字串在做判斷的話會長這樣

```CSHARP

if (status == "待處理")
{
    Console.WriteLine("訂單尚未處理。");
	//// 處理訂單...
}


```


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/Enum2/1_hardcode_like_sand.png)



更好的做法是，定義一組訂單狀態列表，大家遵循或修改都依照這個列表

```CSHARP

public enum OrderStatus
{
    Pending = 0,       // 待處理
    Processing = 1,    // 處理中
    Completed = 2,     // 完成
    Canceled = 3       // 已取消
}

if (status == OrderStatus.Pending)
{
    Console.WriteLine("訂單尚未處理。");
}

```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/Enum2/1__enum_is_noincontinue.png)


<!-- endtab -->

<!-- tab Description-->

有時候，我們會希望 Enum 可以對應一段更具描述性的資訊。這可以透過在 Enum 上掛 Description Attribute 來實現。

藉由引用 System.ComponentModel.DescriptionAttribute namespace 可以使用 [Description(""]


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/Enum2/description.png)


例如傳簡訊

```CSHARP
void Main()
{
	var desc = SMSTemplateEnum.RegisterMember.GetDescription();
	Console.WriteLine(desc); // 👉 會員註冊
}

public static class EnumExtensions
{
	public static string GetDescription(this Enum value)
	{
		Type type = value.GetType();
		var memberInfo = type.GetMember(value.ToString());

		if (memberInfo != null && memberInfo.Length > 0)
		{
			var attrs = memberInfo[0].GetCustomAttributes(typeof(DescriptionAttribute), false);
			if (attrs != null && attrs.Length > 0)
			{
				return ((DescriptionAttribute)attrs[0]).Description;
			}
		}

		return value.ToString(); // 沒設定 Description 就回傳欄位名稱
	}
}

public enum SMSTemplateEnum
{
	[Description("會員註冊")]
	RegisterMember = 0,

	[Description("官網導下載")]
	OfficialToAppDownLoad = 1,

	[Description("會員卡管理")]
	MembershipCard = 3,
}

```


![Image](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/Enum2/description_2.png)


<!-- endtab -->

<!-- tab Enum 當作查詢主鍵，對應到處理器-->


我們把用 Enum 來「判斷」，升格到 「查表」，查的是規則、邏輯、設定、服務、函式，甚至整段流程


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/Enum2/3_strategy_1.png)


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/Enum2/3_2_strategy.png)


```CSHARP

//// 用戶類型 enum
public enum UserType
{
    Normal,
    VIP,
    New
}

//// 定義折扣策略介面
public interface IUserDiscountStrategy : IDiscountStrategy
{
    UserType UserType { get; }
}

//// 實作不同類型折扣策略介面
public class VIPStrategy : IUserDiscountStrategy
{
    public UserType UserType => UserType.VIP;
    public decimal CalculateDiscount(decimal price) => price * 0.8m;
}

public class NormalStrategy : IUserDiscountStrategy
{
    public UserType UserType => UserType.Normal;
    public decimal CalculateDiscount(decimal price) => price;
}

public class NewUserStrategy : IUserDiscountStrategy
{
    public UserType UserType => UserType.New;
    public decimal CalculateDiscount(decimal price) => price * 0.9m;
}

//// 註冊
builder.Services.AddTransient<IUserDiscountStrategy, NormalStrategy>();
builder.Services.AddTransient<IUserDiscountStrategy, VIPStrategy>();
builder.Services.AddTransient<IUserDiscountStrategy, NewUserStrategy>();
builder.Services.AddSingleton<DiscountStrategyFactory>();

//// 實作工廠模式
public class DiscountStrategyFactory
{
    private readonly Dictionary<UserType, IUserDiscountStrategy> _strategyMap;

    public DiscountStrategyFactory(IEnumerable<IUserDiscountStrategy> strategies)
    {
        _strategyMap = strategies.ToDictionary(s => s.UserType);
    }

    public IDiscountStrategy GetStrategy(UserType userType)
    {
        if (_strategyMap.TryGetValue(userType, out var strategy))
        {
            return strategy;
        }

        throw new NotSupportedException($"不支援的用戶類型：{userType}");
    }
}

//// 呼叫
var factory = serviceProvider.GetRequiredService<DiscountStrategyFactory>();
var userType = UserType.VIP;
var price = 1000m;
var strategy = factory.GetStrategy(userType);
var finalPrice = strategy.CalculateDiscount(price);
Console.WriteLine($"計算後價格：{finalPrice}");


```


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/Enum2/3_3_strategy.png)


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/Enum2/3_4_stragegy.png)


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/Enum2/3_5_strategy.png)


<!-- endtab -->


<!-- tab 結語-->


![4](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/Enum2/1_enum_is_decoupling.png)


正如人生中，良好的規矩不是束縛，而是保護。寫程式不是只有快，而是留下可被理解、被擴展、被信任的邏輯。
Enum 看似微小，卻是尊重邏輯、尊重團隊、尊重未來的證明。每一個 Enum，就像種下一棵可以遮風避雨的樹。
它不是最華麗的語法，卻用來建築有溫度的家。

某日有開發者問我：「這不過是一個小小的 enum，何必那麼認真？」

我微笑回答：「你所謂的小小 enum，是他日我所依靠的大樹陰涼。」



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Enum/Enum2/enum_2.png)



<!-- endtab -->


{% endtabs %}
