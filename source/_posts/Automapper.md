---
title: AutoMapper
date: 2024-07-02 23:11:05
categories: Others
top_img: https://i.imgur.com/KXBUhrt.png
cover : https://i.imgur.com/KXBUhrt.png
tags:
    - C#
toc:
toc_number:
comments :
---

![Image](https://i.imgur.com/lBmW1cC.png)

organic and gluten-free ?

最近，對研發工具有些興趣。在眾多工具中，AutoMapper 作為研究對象。儘管我對它的了解還停留在一知半解的階段，但我決定以此為切入點，探索「使用工具」這個主題背後的含義。

通過研究 AutoMapper，我開始思考：一個好的工具應該具備哪些特質？它如何影響我們的工作方式和思維模式？在軟體開發中，我們應該如何權衡使用現成工具和自行開發之間的利弊？

# 自訂轉換邏輯

例如在 DB 我們有這個 Model

```csharp

public class CountryProfile
{
	public long CountryProfile_Id { get; set; }
	public string CountryProfile_Code { get; set; }
	public string CountryProfile_Name { get; set; }
	public string CountryProfile_EnglishName { get; set; }
	public string CountryProfile_AliasCode { get; set; }
	public DateTime CountryProfile_CreatedDateTime { get; set; }
	public string CountryProfile_CreatedUser { get; set; }
	public byte[] CountryProfile_Rowversion { get; set; }
}

```

但傳到 Controller 層時，我們不需要都拿出來用，我們可以定義

```csharp

public class CountryProfileEntity
{
	public long Id { get; set; }

	public string Name { get; set; }

	public string EnglishName { get; set; }

	public string CountryCode { get; set; }

	public string AliasCode { get; set; }

}

```

AutoMapper
```csharp

public class CountryProfileEntityMappingProfile : Profile
{
	public CountryProfileEntityMappingProfile()
	{
		CreateMap<CountryProfile, CountryProfileEntity>()
			.ForMember(i => i.Id, s => s.MapFrom(src => src.CountryProfile_Id))
			.ForMember(i => i.Name, s => s.MapFrom(src => src.CountryProfile_Name))
			.ForMember(i => i.EnglishName, s => s.MapFrom(src => src.CountryProfile_EnglishName))
			.ForMember(i => i.CountryCode, s => s.MapFrom(src => src.CountryProfile_Code))
			.ForMember(i => i.AliasCode, s => s.MapFrom(src => src.CountryProfile_AliasCode));		
	}
}

```
.ForMember 用於指定目標類別（例如 CountryProfileEntity）中的屬性（例如 Id、Name、EnglishName 等）的映射設定。
.MapFrom 用於指定如何從來源類別（例如 CountryProfile）的特定屬性中獲取值，並將其映射到目標類別的相應屬性上。

之後要取得 CoiuntryProfile 的地方，只要呼叫 Mapper.Map, 例如 : 
```csharp

using (var transactionScope = new TransactionScope(TransactionScopeOption.Required))
{
    using (var context = CountryDbContext.CreateNew(isReadOnly: true))
    {
        var query = context.CountryProfile.Valids().Where(c => c.CountryProfile_Code == profileCode).FirstOrDefault();

        var result = Mapper.Map<CountryProfileEntity>(query);

        return result;
    }
}

```

這就是基本的 AutoMapper 的使用方法，並且他也有打算處理一些問題，例如...

# Handling null collections

當使用 AutoMapper 映射集合屬性時，如果來源值為 null，AutoMapper 會將目標字段映射為空集合，而不是設置目標值為 null，可以避免後續使用時發生null reference Exception

官方文件說明 : https://docs.automapper.org/en/stable/Lists-and-arrays.html#handling-null-collections

例如一般情況下
```csharp

public class Source
{
	public IEnumerable<string> Items { get; set; }
}

public class Destination
{
	public IEnumerable<string> Items { get; set; }
}

Source source = new Source { Items = null };

Destination dest = new Destination { Items = source.Items };

try
    {
        foreach (var item in dest.Items) //// 異常
        {
            Console.WriteLine(item);
        }
    }
catch (NullReferenceException ex)
{
    Console.WriteLine($"NullReferenceException caught: {ex.Message}");
}

/***

NullReferenceException caught: Object reference not set to an instance of an object.

***/

```

改為 Automapper

```csharp

var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<Source, Destination>(); // 沒有額外的設置，使用默認行為
});

IMapper mapper = config.CreateMapper();

// 範例：來源屬性為 null 的情況
Source source = new Source { Items = null };

Destination dest = mapper.Map<Destination>(source);

Console.WriteLine($"Destination.Items is {(dest.Items == null ? "null" : "not null")}"); 

/***

Destination.Items is not null

***/

```

但如果希望 Null 就好好轉換成 Null，可以設定 cfg.AllowNullCollections = true; 來讓所有為 Null 的空集合轉換對象保持 Null


再看到一個使用場景，我們想從資料庫不打算撈出全部屬性的資料，而是根據 Mapping 後的需求取出來，AutoMapper 也有提供 ProjectTo這個工具

# IQueryable 直接映射

假設我們有這樣的資料要映射成Dto

```csharp

public class Order
{
	public int Id { get; set; }
	public Customer Customer { get; set; }
	public ICollection<OrderItem> OrderItems { get; set; }
}

public class Customer
{
	public string FullName { get; set; }
}

public class OrderItem
{
	public int Id { get; set; }
	public Product Product { get; set; }
	public int Count { get; set; }
	public decimal ItemPrice { get; set; }
}

public class Product
{
	public string ShortName { get; set; }
	public string Description { get; set; }
}

public class OrderDto
{
	public int Id { get; set; }
	public string CustomerName { get; set; }
	public List<OrderLineItemDto> LineItems { get; set; }
}

public class OrderLineItemDto
{
	public int Id { get; set; }
	public string ProductName { get; set; }
	public string Description { get; set; }
	public int Count { get; set; }
	public decimal Price { get; set; }
}

```

一般來說我們可能這樣手動處理
```csharp

var orderDtos = context.Orders.Select(order => new OrderDto
{
    Id = order.Id,
    CustomerName = order.Customer.FullName,
    LineItems = order.OrderItems.Select(item => new OrderLineItemDto
    {
        Id = item.Id,
        ProductName = item.Product.ShortName,
        Description = item.Product.Description,
        Count = item.Count,
        Price = item.ItemPrice
    }).ToList()
}).ToList();

```

但使用 AutoMapper 可以這樣處理
```csharp

// 在某處配置 AutoMapper
var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<Order, OrderDto>()
        .ForMember(dest => dest.CustomerName, opt => opt.MapFrom(src => src.Customer.FullName));
    cfg.CreateMap<OrderItem, OrderLineItemDto>()
        .ForMember(dest => dest.ProductName, opt => opt.MapFrom(src => src.Product.ShortName))
        .ForMember(dest => dest.Description, opt => opt.MapFrom(src => src.Product.Description))
        .ForMember(dest => dest.Price, opt => opt.MapFrom(src => src.ItemPrice));
});

IMapper mapper = config.CreateMapper();

//// IQuerable
var orders = context.Orders

//// 映射
var orderDtos = orders.ProjectTo<OrderDto>(mapper.ConfigurationProvider).ToList();

```


#  一些可能發生的問題

先說說在使用可能會遇到的困難

## 維護上有漏掉的可能性

例如

```csharp

public class Source
{
    public int Id { get; set; }
    public string Name { get; set; }
}

public class Destination
{
    public int Id { get; set; }
    public string FullName { get; set; }
}

var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<Source, Destination>()
       .ForMember(dest => dest.FullName, opt => opt.MapFrom(src => src.Name));
});

IMapper mapper = config.CreateMapper();


```

假設某天有人重構了 Source ，將 Name 屬性更改為 FirstName，忘記去更新 AutoMapper 配置中的映射關係，就會出現一些意料之外的結果


## 繼承關係無法顯示的在配置中看出

映射的屬性中，無法直觀地看到繼承的關係。

```csharp

public class BaseEntity
{
    public int Id { get; set; }
}

public class Order : BaseEntity
{
    public string OrderNumber { get; set; }
}

public class OrderDto
{
    public int Id { get; set; }
    public string OrderNumber { get; set; }
}

var config = new MapperConfiguration(cfg => {
    cfg.CreateMap<Order, OrderDto>();
});

var mapper = config.CreateMapper();
var order = new Order { Id = 1, OrderNumber = "12345" };
var orderDto = mapper.Map<OrderDto>(order); // 繼承關係不直觀


```

但或許可以在配置中明確寫出(但實際使用時每個人都會遵守規範嗎?)

```csharp

var config = new MapperConfiguration(cfg => {
    cfg.CreateMap<BaseEntity, OrderDto>();
    cfg.CreateMap<Order, OrderDto>();
});

var mapper = config.CreateMapper();
var order = new Order { Id = 1, OrderNumber = "12345" };
var orderDto = mapper.Map<OrderDto>(order); // 清晰地映射繼承關係

```


## 映射配置中編寫業務邏輯

假設我們有一個場景，根據訂單項目的數量來計算總價格。這是一個典型的業務邏輯，應該放在業務層，而不是映射配置中。

把業務邏輯放在 AutoMapper 中
```csharp

public class Order
{
    public int Id { get; set; }
    public ICollection<OrderItem> OrderItems { get; set; }
}

public class OrderItem
{
    public int Count { get; set; }
    public decimal ItemPrice { get; set; }
}

public class OrderDto
{
    public int Id { get; set; }
    public decimal TotalPrice { get; set; }
}

var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<Order, OrderDto>()
        .ForMember(dest => dest.TotalPrice, opt => opt.MapFrom(src => src.OrderItems.Sum(item => item.Count * item.ItemPrice))); // 業務邏輯放在映射配置中
});

IMapper mapper = config.CreateMapper();

var order = new Order
{
    Id = 1,
    OrderItems = new List<OrderItem>
    {
        new OrderItem { Count = 2, ItemPrice = 10.0m },
        new OrderItem { Count = 1, ItemPrice = 20.0m }
    }
};

var orderDto = mapper.Map<OrderDto>(order);
Console.WriteLine($"Order ID: {orderDto.Id}, Total Price: {orderDto.TotalPrice}");

```

這就造成職責的混亂且後續維護的不好發現有這個邏輯的存在


# 什麼時候是較好的使用時機

原則上不會太複雜、牽扯到業務邏輯，且重複並彼此需求一致性高的 Mapping再多處發生時, 是較為適用的場景，例如一些經典狀況

## RESTful API 開發

```csharp

[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IMapper _mapper;
    private readonly MyDbContext _context;

    public OrdersController(IMapper mapper, MyDbContext context)
    {
        _mapper = mapper;
        _context = context;
    }

    [HttpGet]
    public ActionResult<IEnumerable<OrderDto>> GetOrders()
    {
        var orders = _context.Orders.Include(o => o.OrderItems).ThenInclude(oi => oi.Product).ToList();
        var orderDtos = _mapper.Map<IEnumerable<OrderDto>>(orders);
        return Ok(orderDtos);
    }

    [HttpGet("{id}")]
    public ActionResult<OrderDto> GetOrder(int id)
    {
        var order = _context.Orders.Include(o => o.OrderItems).ThenInclude(oi => oi.Product).FirstOrDefault(o => o.Id == id);
        if (order == null)
        {
            return NotFound();
        }
        var orderDto = _mapper.Map<OrderDto>(order);
        return Ok(orderDto);
    }
}

```

## MVC 中，常需要將 Model 映射到 ViewModel，以便於在 View 中展示資料 (Index, Detail...)



# 結語

其實之前在實習時有玩過這個東西，但也僅僅是用用看的程度，總之就是用起來滿直觀且 Mapping 也看起來會乾淨許多，但後讀到一篇文:　

- AutoMapper's Design Philosophy(AutoMapper 的 TechLead 出場說明)
https://www.jimmybogard.com/automappers-design-philosophy/



作者提到，許多人會遇到不好的經驗往往是在沒有理解工具被發展出來的精隨，因此在某種程度的濫用下引發了不少問題

AutoMapper 的誕生背景是為了解決 MVC 程式中模型設計的問題，MVC 框架在 ASP.NET MVC 對 Model 的定義中並不明確，因此作者在開發中遇到的一些問題：

- 賦值的 Code 冗長且重複性高
- DTO 的命名和成員名稱缺乏統一性。( Item, PriceItem... 各自為政)
- Null Reference 異常問題
- View Model 毫無意義地缺乏一致性
- 處理缺失資料的 Code 容易出錯，並且經常被遺漏
- 為所有這些賦值編寫的測試很容易出錯

所以最後決定乾脆開發一個工具，它可以：

- 強制執行目標類型的命名約定
- 消除所有的空引用異常
- 讓測試變得非常容易

因此 AutoMapper 就誕生了，其運作原理是強制執行約定。並假設

- 你的 target 是 source 的一個 subset
- 你的 target 上的所有內容都是要 Mapping 的
- Target Member 名稱與 source 的名稱完全相同

制定一套可強制執行的設計規則，幫助你在開發過程中簡化流程。

所以在引用任何工具前，好好盤點清楚，這些是你要解決的問題嗎?


## 精神能量分析

精神能量 : 🎞️

每當談及 AI 這個話題時，人們似乎急於表現自己對 AI 的見解，生怕落後於這股科技浪潮。他們熱衷於分享最新的 AI 趨勢，彷彿這樣就能證明自己走在時代前沿。然而，在這種急於表現的氛圍中，往往伴隨著一種無形的焦慮感的散播，人人歌頌著效率主義，這到底是在使用工具還是被工具吞噬呢?

當然，我覺得焦慮的產生並不能完全歸咎於他人。它更多地反映了個人的心理狀態：我們是否容易被他人的言論所影響？我們是否擁有堅定的人生態度和自處之道？(這就是我的忍道!?)

最近，Winston 推薦了幾部經典老動畫，如《海潮之聲》和《狼的孩子雨和雪》。他生動描繪了這些作品中的繪師與導演如何在他心中留下深刻的人性印記。我在想，或許在這些作品中，我們能找到一些答案。
