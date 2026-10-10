---
title: in
date: 2026-09-24 14:10:00
categories: Payment
top_img: 
cover : 
toc:
toc_number:
comments :
tags:
---

{% tabs time %}




<!-- tab 測試情境 -->



## 要測試什麼規則？

假設通知發送失敗後，必須等待 **至少 5 分鐘**，才可以重新發送。

我們希望驗證以下三種情境：

| 上次失敗時間 | 現在時間 | 經過多久 | 預期結果 |
|---|---|---|---|
| 10:00:00 | 10:04:59 | 4 分 59 秒 | 不可重試 |
| 10:00:00 | 10:05:00 | 剛好 5 分鐘 | 可以重試 |
| 10:00:00 | 10:05:01 | 5 分 1 秒 | 可以重試 |

要驗證這條規則，測試需要控制兩個時間：

1. **上次失敗時間**：可以直接建立測試資料指定。
2. **現在時間**：如果程式直接使用 `DateTime.Now`，就會讀取電腦當下的時間，無法由測試直接指定。




## 為什麼「現在時間」需要可以控制？


測試設定資料後，到實際呼叫方法進行判斷，中間仍然會經過時間。

```text
測試設定資料
    ↓ 時間持續前進
呼叫實際方法
    ↓ 時間持續前進
方法讀取 DateTime.Now
```

如果測試的是「已經過了 10 分鐘」，這點時間差通常不影響結果。

但如果測試的是「還差 1 毫秒才滿 5 分鐘」，判斷結果就可能因為執行速度而改變。

**我們希望測試結果取決於程式邏輯，而不是測試當下電腦跑得多快。**


## TimeProvider 如何解決？


讓程式透過一個可以替換的「時鐘」取得時間：

| 執行情境 | 使用的時鐘 | 取得的時間 |
|---|---|---|
| 正式執行 | `TimeProvider.System` | 真實的現在時間 |
| 單元測試 | 測試建立的時鐘替身 | 測試指定的固定時間 |

這不會修改電腦的系統時間，只會影響使用該時鐘的程式。

接下來用同一條重試規則，比較兩種寫法。範例是簡化的重試判斷，方便理解時間如何影響測試。



<!-- endtab -->





<!-- tab 直接使用 DateTime.Now -->
### ① 實際程式碼

程式直接讀取系統時間，判斷距離上次失敗是否已滿 5 分鐘。

```csharp
public class RetryChecker
{
    /// <summary>
    /// 判斷距離上次失敗是否已滿五分鐘。
    /// </summary>
    /// <param name="lastFailedTime">上次失敗時間。</param>
    /// <returns>已滿五分鐘時回傳 true，否則回傳 false。</returns>
    public bool CanRetry(DateTime lastFailedTime)
    {
        return DateTime.Now >= lastFailedTime.AddMinutes(5);
    }
}
```

### ② 單元測試程式碼

這個測試想驗證：

> 還差 1 毫秒才滿 5 分鐘，應該不能重試。

```csharp
using System;
using Xunit;

public class RetryCheckerTests
{
    /// <summary>
    /// 驗證距離五分鐘門檻還差一毫秒時，不可重試。
    /// 此寫法使用真實時間，可能因執行耗時而不穩定。
    /// </summary>
    [Fact]
    public void CanRetry_OneMillisecondBeforeThreshold_ReturnsFalse()
    {
        // Arrange：安排測試資料。
        var checker = new RetryChecker();

        var lastFailedTime = DateTime.Now
            .AddMinutes(-5)
            .AddMilliseconds(1);

        // Act：執行實際判斷。
        var result = checker.CanRetry(lastFailedTime);

        // Assert：預期還沒滿五分鐘，不能重試。
        Assert.False(result);
    }
}
```

### ③ 模擬執行過程

假設測試設定資料的瞬間，真實時間是 `10:00:00.000`：

```text
設定上次失敗時間：

10:00:00.000 − 5 分鐘 + 1 毫秒
＝ 09:55:00.001

所以可以開始重試的時間是：

09:55:00.001 + 5 分鐘
＝ 10:00:00.001
```

接著，測試呼叫 `CanRetry()`。假設方法執行到讀取時間時，已經過了 2 毫秒：

| 步驟 | 真實時間 | 發生什麼事 |
|---|---|---|
| 測試設定資料 | 10:00:00.000 | 上次失敗時間設為 09:55:00.001 |
| 到達重試門檻 | 10:00:00.001 | 已經滿 5 分鐘 |
| 方法讀取 `DateTime.Now` | 10:00:00.002 | 判斷可以重試，回傳 `true` |
| 測試驗證 | — | 原本預期 `false`，測試失敗 |

時間順序如下：

```text
10:00:00.000       10:00:00.001       10:00:00.002
設定測試資料 ─────── 到達重試門檻 ─────── 實際執行判斷
「還不能重試」                         「已經可以重試」
```

### 為什麼這個測試不穩定？

**程式判斷沒有錯，是測試原本安排的情境，在判斷前已經改變了。**

測試想驗證「還差 1 毫秒」，但方法真正讀取時間時，可能已經超過門檻。

- 執行得夠快，可能得到 `false`，測試通過。
- 執行稍慢，可能得到 `true`，測試失敗。
- 中間停下來除錯，也會讓真實時間繼續前進。

因此，直接使用 `DateTime.Now` 並不是完全不能測試，而是**難以穩定重現精準的時間邊界**。


<!-- endtab -->




<!-- tab 注入 TimeProvider -->


### ① 實際程式碼

改成透過注入的 `TimeProvider` 取得時間。

`GetUtcNow()` 回傳 UTC 時間，因此這個範例統一使用 `DateTimeOffset` 表示時間。

```csharp
public class RetryChecker
{
    private readonly TimeProvider _clock;

    /// <summary>
    /// 建立重試判斷器。
    /// </summary>
    /// <param name="clock">用來取得現在時間的時鐘。</param>
    public RetryChecker(TimeProvider clock)
    {
        _clock = clock;
    }

    /// <summary>
    /// 判斷距離上次失敗是否已滿五分鐘。
    /// </summary>
    /// <param name="lastFailedTime">上次失敗時間。</param>
    /// <returns>已滿五分鐘時回傳 true，否則回傳 false。</returns>
    public bool CanRetry(DateTimeOffset lastFailedTime)
    {
        return _clock.GetUtcNow() >= lastFailedTime.AddMinutes(5);
    }
}
```

判斷規則仍然相同：

```text
現在時間 ≥ 上次失敗時間 + 5 分鐘
```

改變的是：**「現在時間」改由傳入的時鐘提供。**

### ② 單元測試程式碼

使用 NSubstitute 建立一個時鐘替身，固定回傳指定時間。

```csharp
using System;
using NSubstitute;
using Xunit;

public class RetryCheckerTests
{
    /// <summary>
    /// 驗證距離五分鐘門檻還差一毫秒時，不可重試。
    /// </summary>
    [Fact]
    public void CanRetry_OneMillisecondBeforeThreshold_ReturnsFalse()
    {
        // Arrange：固定現在時間為 UTC 10:00:00.000。
        var now = new DateTimeOffset(
            2026, 10, 7, 10, 0, 0, TimeSpan.Zero);

        var clock = Substitute.For<TimeProvider>();
        clock.GetUtcNow().Returns(now);

        var checker = new RetryChecker(clock);

        // 上次失敗時間為 09:55:00.001。
        // 相對於固定的現在時間，還差一毫秒才滿五分鐘。
        var lastFailedTime = now
            .AddMinutes(-5)
            .AddMilliseconds(1);

        // Act：執行真正的重試判斷。
        var result = checker.CanRetry(lastFailedTime);

        // Assert：尚未滿五分鐘，不能重試。
        Assert.False(result);
    }
}
```

關鍵是這兩行：

```csharp
var clock = Substitute.For<TimeProvider>();
clock.GetUtcNow().Returns(now);
```

意思是：

> 建立一個測試時鐘。每次呼叫它的 `GetUtcNow()`，都回傳指定的 `now`。

**單純建立 `TimeProvider` 並不會自動固定時間；是測試中的 `Returns(now)` 設定了固定回傳值。**

### ③ 模擬執行過程

以下時間均為 UTC：

| 步驟 | 程式讀到的時間 | 發生什麼事 |
|---|---|---|
| 測試設定時鐘 | 10:00:00.000 | 固定現在時間 |
| 測試設定失敗時間 | — | 上次失敗時間為 09:55:00.001 |
| 呼叫 `CanRetry()` | — | 真實世界的時間仍然前進 |
| 方法呼叫 `_clock.GetUtcNow()` | **10:00:00.000** | 測試時鐘仍回傳固定時間 |
| 執行判斷 | — | 尚未到 10:00:00.001，回傳 `false` |
| 測試驗證 | — | 符合預期，測試通過 |

```text
真實世界：
開始測試 ───── 過了 2 毫秒 ───── 停下來除錯 10 分鐘

測試時鐘：
10:00:00.000 ── 10:00:00.000 ── 10:00:00.000
```

即使中間停下來除錯，測試時鐘仍然回傳相同時間，因此能穩定重現「還差 1 毫秒」的情境。

### ④ 正式執行時怎麼辦？

正式環境使用系統時鐘：

```csharp
var checker = new RetryChecker(TimeProvider.System);
```

如果透過 DI 建立物件，就註冊：

```csharp
services.TryAddSingleton<TimeProvider>(TimeProvider.System);
```

這表示：如果還沒有註冊 `TimeProvider`，就提供系統時鐘。

因此，正式環境仍會取得真實時間；只有測試才換成指定時間的替身。





<!-- endtab -->





<!-- tab  一次驗證三種情境 -->




### 用同一個測試，驗證門檻前、門檻當下、門檻後

把「上次失敗時間」固定在 UTC `10:00:00`，再分別指定不同的現在時間：

```csharp
using System;
using NSubstitute;
using Xunit;

public class RetryCheckerBoundaryTests
{
    /// <summary>
    /// 驗證五分鐘重試門檻前、當下與之後的判斷結果。
    /// </summary>
    /// <param name="elapsedSeconds">距離上次失敗經過的秒數。</param>
    /// <param name="expected">預期是否可以重試。</param>
    [Theory]
    [InlineData(299, false)]
    [InlineData(300, true)]
    [InlineData(301, true)]
    public void CanRetry_AroundFiveMinuteThreshold_ReturnsExpected(
        int elapsedSeconds,
        bool expected)
    {
        // Arrange：上次失敗時間固定為 UTC 10:00:00。
        var lastFailedTime = new DateTimeOffset(
            2026, 10, 7, 10, 0, 0, TimeSpan.Zero);

        // 根據測試情境，指定現在時間。
        var now = lastFailedTime.AddSeconds(elapsedSeconds);

        var clock = Substitute.For<TimeProvider>();
        clock.GetUtcNow().Returns(now);

        var checker = new RetryChecker(clock);

        // Act：執行真正的重試判斷。
        var result = checker.CanRetry(lastFailedTime);

        // Assert：確認結果符合規則。
        Assert.Equal(expected, result);
    }
}
```

這三筆測試資料分別代表：

| 測試資料 | 假裝現在 | 預期 |
|---|---|---|
| `299, false` | 10:04:59 | 不可重試 |
| `300, true` | 10:05:00 | 可以重試 |
| `301, true` | 10:05:01 | 可以重試 |

**不需要真的等 5 分鐘，三種情境就能依序執行。**

### 使用假的時間，還算是在測真正的程式嗎？

算，因為我們只控制「現在時間」，並沒有替程式決定判斷結果。

```text
測試提供：
上次失敗時間 + 固定的現在時間
                ↓
真正的 CanRetry() 執行比較
                ↓
測試確認回傳結果是否正確
```

如果實際程式不小心把 `>=` 改成 `>`：

```csharp
return _clock.GetUtcNow() > lastFailedTime.AddMinutes(5);
```

「剛好滿 5 分鐘」那筆測試就會失敗，因為程式錯誤地回傳了 `false`。

這正是測試要幫我們發現的問題。




<!-- endtab -->



{% endtabs %}
