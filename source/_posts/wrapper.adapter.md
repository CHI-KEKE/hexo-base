
#### 1.5.2 Wrapper

Wrapper 模式是一種常見的設計模式，它可以用來包裝現有的物件，提供額外的功能或改變其行為。在泛型的幫助下，我們可以建立通用的 Wrapper 類別來處理任何型別的物件

##### 🎯 Wrapper 基本概念

Wrapper 類別的主要目的是：
- 🔐 **封裝原始物件**：提供對底層物件的受控存取
- 🎨 **增強功能**：在不修改原始類別的情況下添加新功能
- 🛡️ **保護資料**：控制對包裝物件的存取方式
- 🔄 **轉換介面**：為不相容的介面提供適配

##### 📝 實際應用範例

以下範例展示了泛型 Wrapper 的使用和一些需要注意的問題：

```csharp
void Main()
{
    var items = new List<int> { 1, 2, 3 };
    var wrappers = CreateWrapper2<int>(items);

    var store = new List<Wrapper<int>>();
    store.AddRange(wrappers);

    // ❌ 這會回傳 false，因為 Wrapper 沒有實作相等性比較
    var storeContainsAnyWrappers = wrappers
        .Any(wrapper => store.Contains(wrapper)).Dump(); // = false	
}

public IEnumerable<Wrapper<T>> CreateWrapper<T>(IEnumerable<T> items)
{
    return items.Select(item => new Wrapper<T>(item));
}

public IEnumerable<Wrapper<T>> CreateWrapper2<T>(IEnumerable<T> items)
{
    return items.Select(item =>
    {
        Console.WriteLine($"Create wrapper for {item}");
        return new Wrapper<T>(item);
    }).ToList();
}

public class Wrapper<T>
{
    private readonly T _item;

    public Wrapper(T item)
    {
        _item = item;
    }
}
```

##### 🔍 問題分析

在上述範例中，`storeContainsAnyWrappers` 會回傳 `false`，這是因為：

**🚨 物件參考比較問題：**
1. `CreateWrapper2` 建立了新的 `Wrapper<T>` 實例
2. `store.AddRange(wrappers)` 將這些實例加入到 store 中
3. `Contains()` 使用預設的參考相等性比較
4. 即使包裝的值相同，但 `Wrapper` 物件是不同的實例

##### ✅ 解決方案

**方案一：實作 IEquatable<T> 介面**
```csharp
public class Wrapper<T> : IEquatable<Wrapper<T>>
{
    private readonly T _item;

    public Wrapper(T item)
    {
        _item = item;
    }

    public T Item => _item;

    public bool Equals(Wrapper<T> other)
    {
        if (other is null) return false;
        if (ReferenceEquals(this, other)) return true;
        return EqualityComparer<T>.Default.Equals(_item, other._item);
    }

    public override bool Equals(object obj)
    {
        return obj is Wrapper<T> wrapper && Equals(wrapper);
    }

    public override int GetHashCode()
    {
        return EqualityComparer<T>.Default.GetHashCode(_item);
    }

    public static bool operator ==(Wrapper<T> left, Wrapper<T> right)
    {
        return EqualityComparer<Wrapper<T>>.Default.Equals(left, right);
    }

    public static bool operator !=(Wrapper<T> left, Wrapper<T> right)
    {
        return !(left == right);
    }
}
```

**方案二：使用記錄類型 (Record)**
```csharp
public record Wrapper<T>(T Item);

// 使用方式
void Main()
{
    var items = new List<int> { 1, 2, 3 };
    var wrappers = items.Select(item => new Wrapper<int>(item)).ToList();
    
    var store = new List<Wrapper<int>>();
    store.AddRange(wrappers);
    
    // ✅ 現在會回傳 true，因為 record 自動實作了相等性比較
    var storeContainsAnyWrappers = wrappers
        .Any(wrapper => store.Contains(wrapper)); // = true
}
```

> **🌟 重點提醒**
> 
> 1. **相等性問題**：Wrapper 類別需要適當實作相等性比較，否則 `Contains()` 等方法可能無法正常工作
> 2. **記錄類型優勢**：C# 9+ 的 record 類型自動提供值相等性，非常適合作為簡單的 Wrapper
> 3. **延遲執行**：注意 LINQ 的延遲執行特性對 Wrapper 建立時機的影響
> 4. **功能擴展**：Wrapper 模式可以用來添加日誌、快取、驗證等橫切關注點










https://ithelp.ithome.com.tw/articles/10205321


