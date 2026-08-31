---
title: Generic - 型越の魂
date: 2025-10-27 08:28:05
categories: 程思舞想
top_img: https://i.imgur.com/Zly1UDM.png
cover : https://i.imgur.com/Zly1UDM.png
tags:
    - 程思舞想
toc:
toc_number:
comments :
---

![Generic](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/landing.png)

{% tabs Generic%}

<!-- tab 靈魂與肉身-->

「靈魂，不屬於肉身；邏輯，也不該屬於型別。」

我們都知道，肉體只是外殼，靈魂才是本質。人如此，程式亦然。在軟體的世界裡，邏輯就像靈魂，它決定了一段程式「該怎麼思考」；而型別（Type）就像肉體，負責「具體地呈現」

![Generic](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/concept_1.png)

<!-- endtab -->

<!-- tab 想解決甚麼問題-->

![Generic](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/first_cage.png?raw=true)

在泛型出現以前，我們若想讓容器能「存放不同型別」的資料通常會用 object 來實作

```csharp
ArrayList list = new ArrayList();
list.Add(123);
list.Add("ABC");

// 問題來了
int n = (int)list[0];     // OK
int m = (int)list[1];     // ❌ runtime error
```

這樣雖然「兼容」了所有型別，但有兩個致命缺點

- 你要手動轉型 (casting)，而且有風險（可能轉錯）
- 編譯器無法幫你檢查型別錯誤，只能等執行時爆掉

![靈魂](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/old_1.png)

泛型 (Generics) 是一種「延後決定型別」的程式設計方法，它讓你寫出對型別沒有限制，但仍然安全可重用的程式碼，也就是說在不同型別之間共用邏輯，但又不放棄型別資訊
```csharp
List<int> list = new List<int>();
list.Add(123);
// list.Add("ABC"); // ❌ 這行編譯就不通過
```

✅ 泛型同樣「通用」，但同時「型別安全」，也讓編譯器能幫你防錯

![Generic](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/build_time_safe.png)

<!-- endtab -->

<!-- tab 型別參數化-->

泛型的核心概念叫「型別參數化（Type Parameterization）」
意思是說，就像函式可以接收不同的「值」參數，泛型則能接收不同的「型別」作為參數，這讓你的程式不再依賴某一種固定型別，而是變得「對型別開放、對邏輯封閉」，換句話說，你把邏輯抽出來，讓它在不同型別之間自由穿梭

![型別參數化](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/type_params.png)

<!-- endtab -->

<!-- tab 「重複邏輯，只是型別不同」-->

有沒有那種時候，**你發現自己在寫幾乎相同的程式碼，唯一的差別只是處理的資料型別不同**

在物件導向程式設計中，我們追求的是 **DRY 原則（Don't Repeat Yourself）**，但傳統的繼承和多型無法解決「邏輯相同但型別不同」的問題。泛型的出現就是為了解決這個根本矛盾

泛型可以為您做到

- **邏輯抽象化**：將演算法從具體型別中抽離出來
- **型別延遲綁定**：在使用時才決定具體要處理什麼型別  
- **編譯時期特化**：編譯器為每個型別產生專用的程式碼

<br>

#### 資料驗證器

不同型別的資料驗證邏輯

**❌ 重複程式碼的做法：**
```csharp
// 字串驗證器
public class StringValidator
{
    private readonly List<Func<string, (bool IsValid, string Error)>> _rules = new();
    
    public StringValidator AddRule(Func<string, (bool, string)> rule)
    {
        _rules.Add(rule);
        return this;
    }
    
    public ValidationResult Validate(string value)
    {
        foreach (var rule in _rules)
        {
            var (isValid, error) = rule(value);
            if (!isValid)
                return ValidationResult.Fail(error);
        }
        return ValidationResult.Success();
    }
}

// 整數驗證器（幾乎一模一樣的邏輯！）
public class IntValidator
{
    private readonly List<Func<int, (bool IsValid, string Error)>> _rules = new();
    
    public IntValidator AddRule(Func<int, (bool, string)> rule)
    {
        _rules.Add(rule);
        return this;
    }
    
    public ValidationResult Validate(int value)
    {
        foreach (var rule in _rules)
        {
            var (isValid, error) = rule(value);
            if (!isValid)
                return ValidationResult.Fail(error);
        }
        return ValidationResult.Success();
    }
}

// 還要寫 DecimalValidator、DateTimeValidator... 無窮無盡 😱
```

![rep_valid](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/recep_validators.png)


**✅ 泛型大哥登場!**
```csharp
public class Validator<T>
{
    private readonly List<Func<T, (bool IsValid, string Error)>> _rules = new();
    
    public Validator<T> AddRule(Func<T, (bool, string)> rule)
    {
        _rules.Add(rule);
        return this;
    }
    
    public ValidationResult Validate(T value)
    {
        foreach (var rule in _rules)
        {
            var (isValid, error) = rule(value);
            if (!isValid)
                return ValidationResult.Fail(error);
        }
        return ValidationResult.Success();
    }
}

public class ValidationResult
{
    public bool IsValid { get; private set; }
    public string ErrorMessage { get; private set; }
    
    public static ValidationResult Success() => new() { IsValid = true };
    public static ValidationResult Fail(string error) => new() { IsValid = false, ErrorMessage = error };
}
```

**使用**
```csharp
void Main()
{
    // 字串驗證：檢查長度和內容
    var stringValidator = new Validator<string>()
        .AddRule(s => s?.Length > 0 ? (true, "") : (false, "字串不能為空"))
        .AddRule(s => s?.Length <= 50 ? (true, "") : (false, "字串長度不能超過50"));
    
    Console.WriteLine(stringValidator.Validate("Hello World").IsValid); // True
    Console.WriteLine(stringValidator.Validate("").ErrorMessage);       // "字串不能為空"
    
    // 整數驗證：檢查範圍
    var intValidator = new Validator<int>()
        .AddRule(i => i >= 0 ? (true, "") : (false, "數值必須大於等於0"))
        .AddRule(i => i <= 100 ? (true, "") : (false, "數值不能超過100"));
    
    Console.WriteLine(intValidator.Validate(50).IsValid);     // True
    Console.WriteLine(intValidator.Validate(-10).ErrorMessage); // "數值必須大於等於0"
    
    // 日期驗證：檢查是否為未來日期
    var dateValidator = new Validator<DateTime>()
        .AddRule(d => d > DateTime.Now ? (true, "") : (false, "日期必須是未來時間"))
        .AddRule(d => d.Year <= 2030 ? (true, "") : (false, "日期不能超過2030年"));
    
    Console.WriteLine(dateValidator.Validate(DateTime.Now.AddDays(1)).IsValid); // True
}
```

![valida_T](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/validator_T.png)
![THOUSAND](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/VALITOR_t2.png)

<br>

#### 鏈式建構器模式 (Query Builder)

![Q_BUILDER](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/q_builder.png)

靈活構建查詢，比較快速簡單的做法可以自己手製查詢建構器

```csharp
public class QueryBuilder<T>
{
    private readonly List<Func<T, bool>> _conditions = new();
    private readonly List<Func<T, object>> _orderByKeys = new();
    
    public QueryBuilder<T> Where(Func<T, bool> condition)
    {
        _conditions.Add(condition);
        return this;
    }
    
    public QueryBuilder<T> OrderBy<TKey>(Func<T, TKey> keySelector)
    {
        _orderByKeys.Add(item => keySelector(item));
        return this;
    }
    
    public IEnumerable<T> Execute(IEnumerable<T> source)
    {
        var result = source.AsEnumerable();
        
        // 套用所有條件
        foreach (var condition in _conditions)
        {
            result = result.Where(condition);
        }
        
        // 套用排序（簡化版本）
        if (_orderByKeys.Any())
        {
            result = result.OrderBy(_orderByKeys.First());
        }
        
        return result;
    }
}

// 使用範例
void Main()
{
    var players = new[]
    {
        new { Name = "Alice", Level = 50, Score = 1200 },
        new { Name = "Bob", Level = 30, Score = 800 },
        new { Name = "Charlie", Level = 70, Score = 2000 }
    };
    
    // 同一個 QueryBuilder 可以處理任何型別！
    var builder = new QueryBuilder<Player>();

    if (levelFilter.HasValue)
        builder.Where(p => p.Level >= levelFilter.Value);

    if (scoreFilter.HasValue)
        builder.Where(p => p.Score >= scoreFilter.Value);

    if (sortBy == "Score")
        builder.OrderBy(p => p.Score);
    else if (sortBy == "Level")
        builder.OrderBy(p => p.Level);

    var highLevelPlayers = builder.Execute(players);
    
    foreach (var player in highLevelPlayers)
    {
        Console.WriteLine($"{player.Name}: Lv.{player.Level}, Score: {player.Score}");
    }
    // 輸出：Alice: Lv.50, Score: 1200
    //       Charlie: Lv.70, Score: 2000
}
```

<!-- endtab -->

<!-- tab 讓「資料結構或工具類別」可重複使用-->

這是泛型的另一個核心應用：**將「演算法邏輯」與「資料型別」徹底分離**

<br>

#### Repository<T> —— 資料存取的萬用倉庫

Repository（儲存庫）是一種常見於 DDD（Domain-Driven Design） 或企業架構的設計模式。它的目的是把「資料存取邏輯（CRUD）」從業務邏輯中分離出來，我們希望能用同樣的方式操作「使用者(User)」或「產品(Product)」資料，而不需要為每種資料重寫一份 CRUD

泛型在這裡的角色是：

「讓一個資料操作模板（Repository）能處理任意實體型別。」

這樣就可以

- 用同樣的方法（Add, Remove, GetAll）操作不同資料表
- 保證所有資料存取都一致
- 方便後期切換儲存方式（例如從記憶體 → 資料庫）

實作 InMemoryRepository
```csharp
void Main()
{
	var userRepo = new InMemoryRepository<User>();
	var alice = new User()
	{
		Name = "Alice"
	};
	var bob = new User()
	{
		Name = "Bob"
	};
	userRepo.Add(alice);
	userRepo.Add(bob);
	userRepo.GetAll().Dump();
	userRepo.Remove(bob);
	userRepo.GetAll().Dump();
}


public interface IRepository<T>
{
	void Add(T item);
	void Remove(T item);
	IEnumerable<T> GetAll();
}


public class InMemoryRepository<T> : IRepository<T>
{
	private readonly List<T> storage = new();
	public void Add(T item)
	{
		Console.WriteLine($"新增 {typeof(T).Name} : {item}");
		storage.Add(item);
	}

	public IEnumerable<T> GetAll()
	{
		return storage;
	}

	public void Remove(T item)
	{
		Console.WriteLine($"移除 {typeof(T).Name} : {item}");
		storage.Remove(item);
	}
}


public class User
{
	public string Name { get; set; }
	public override string ToString() => Name;
}

public class Product
{
	public string Name { get; set; }
	public decimal Price { get; set; }
	public override string ToString() => $"{Name} (${Price})";
}
```

![repo](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/repo_save_split.png)

<br>

#### Result<T> —— 一致的回傳結果容器

如果我們希望統一處理「成功」與「失敗」的情況，而不使用例外（Exception），這時就會用到 Result<T> 模式，我們希望讓「回傳結果」同時包含

- 成功與否（IsSuccess）
- 錯誤訊息（Error）
- 成功時的資料（Value）

泛型的角色讓 Result 可以包裝任何型別的結果，例如：

- Result<int>：運算結果
- Result<User>：資料查詢結果
- Result<string>：API 回覆內容

```CSHARP
void Main()
{
	var cal = new Calculator();
	cal.Divide(3,0).ToString().Dump();
	cal.Divide(3,2).ToString().Dump();
}

public class Error
{
	public string Code { get; }
	public string Message { get; }

	public Error(string code, string message)
	{
		Code = code;
		Message = message;
	}

	public override string ToString() => $"{Code}: {Message}";
}

public class Result<T>
{
	public bool IsSuccess { get; }
	public Error Error { get; }
	public T Value { get; }

	private Result(T value)
	{
		Value = value;
		IsSuccess = true;
	}

	private Result(Error error)
	{
		Error = error;
		IsSuccess = false;
	}

	public static Result<T> Success(T value) => new(value);
	public static Result<T> Failure(string code, string message) => new(new Error(code, message));

	public override string ToString() =>
		IsSuccess ? $"✅ Success: {Value}" : $"❌ Error: {Error}";
}


public class Calculator
{
	public Result<double> Divide(double a, double b)
	{
		if(b == 0)
			return Result<double>.Failure("Code_Zero","除數不可以為0!!");
		return Result<double>.Success(a/b);
	}
}

/// ❌ Error: 除數不可以為0!!
/// ✅ Success: 1.5
```

![result](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/resukt_T.png)

<!-- endtab -->

<!-- tab 總結-->

![overview](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/overview.png)
<!-- endtab -->

<!-- tab 學習-->

![life_is_learning](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/LIFE_Is_learning.png)

在人生中，我們常太早「決定型別」，太早定義自己是什麼人、適合什麼工作、應該走哪條路
但就像泛型的精神一樣，型別可以延後決定。重要的，是你背後的邏輯與原則!

你學會「學習的方法」而不是「某一門技能」，那就像你寫了一個 Learn<T> 泛型函式 ，未來不管學鋼琴、學程式、學語言，你都能套用同一套思考邏輯

```csharp
public class Learner<T>
{
    public void Learn(T subject)
    {
        Console.WriteLine($"正在學習 {typeof(T).Name}...");
    }
}
```

學習「如何學習」，而不是只學「某個東西」。這是泛型思維!

![life_is_learning](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Generic/learning_to_learning.png)

<!-- endtab -->

{% endtabs %}