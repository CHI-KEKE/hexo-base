---
title: Closure
date: 2025-11-02 21:47:03
categories: 程思舞想
top_img: https://i.imgur.com/Zly1UDM.png
cover : https://i.imgur.com/Zly1UDM.png
tags:
    - 程思舞想
toc:
toc_number:
comments :
---

{% tabs Anonymous type%}


![whole_closure](https://github.com/CHI-KEKE/pics/blob/main/Code_Design/Closure/closure_whole.png)


<!-- tab Closure-->

Closure，就像人生的記憶機制。它讓一段邏輯，不只記得「要做什麼」，也記得「曾經與誰有關」。在我們離開某個場景之後，它仍攜帶著當時的狀態、情緒與環境，就像一個攜帶回憶的函式，延續著那份未完的故事。Closure 使「lambda 可以用外部變數」，但它的本質遠不只是「抓外部變數來用」，而是一種讓函式擁有「記憶狀態」的機制


![lambda_to_](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Closure/lambda_has_memory.png)
![not_disappear](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Closure/lambda_not_disappear.png)


<!-- endtab -->

<!-- tab 方法呼叫（傳參數）-->


立刻執行 → 當下的值是什麼就拿什麼

```csharp
void Show(int n) 
{
    Console.WriteLine(n);
}

for (int i = 0; i < 5; i++)
{
    Show(i); // 當下的 i 是多少就傳進去
}

//0
//1
//2
//3
//4
```


而 Lambda 並不是幫你把當下的值存起來，而是把「之後要用哪一個變數」記住，所以多個 Lambda 如果指向同一個變數，就一定會互相干擾，Lambda 的目的是「延後執行一段邏輯」，而不是「立即複製資料」

```csharp
var actions = new List<Action>();

for (int i = 0; i < 5; i++)
{
    actions.Add(() => Console.WriteLine(i)); // 只是記下「到時候要印 i」, 這時候把 i 參考包近來
}

actions.ForEach(a => a.Invoke()); // 現在才真正做, 此時的 i 已經都是 5

//5
//5
//5
//5
//5
```


這裡做的是宣告一個變數 i，而整個 for 迴圈裡的 i 只有「一個」，每次 Add 的時候 Lambda 並沒有執行，只是記下「未來要用 這個 i 來印」，所以 List 裡其實放的是 5 個「指向同一個 i 的行為」，不是 5 個不同的值，當 for 迴圈結束，i 已經變成 5，之後當開始 Invoke 會發現每個 Lambda 都去看同一個 i，所以全部印 5，關鍵是「它們用的是同一個變數來源」

> Lambda 表達式捕獲的是變數 `i` 的**參考**，而不是值的副本。當 Lambda 實際執行時，`i` 的值已經是迴圈結束後的 5



![trap](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Closure/lambda_trapped.png)
![trap_result](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Closure/lambda_trap_result.png)



## ✅ 解決方案

幫每個 Lambda 配一個「專屬變數」，讓它們不要共用同一個狀態

```csharp
var actions = new List<Action>();

for (int i = 0; i < 5; i++)
{
    int copy = i; // 建立區域變數副本
    actions.Add(() => Console.WriteLine(copy)); // 捕獲 copy 而不是 i, 每次的參考都不一樣
}

actions.ForEach(a => a.Invoke());

//0
//1
//2
//3
//4
```


每一次 loop 都建立一個新的區域變數 copy，每一個 Lambda 都只認得自己那顆 copy，所以狀態被「隔離」，不會再互相干預，這是一種刻意切斷共享狀態的寫法


![snapshot](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Closure/snapshot_resolve_trap.png)


<!-- endtab -->

<!-- tab 編譯器背後的轉換-->


![behind](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Closure/behind_the_scene.png)


```csharp
// 原始程式碼
for (int i = 0; i < 5; i++)
{
    actions.Add(() => Console.WriteLine(i));
}

// 概念上編譯器產生的程式碼
var displayClass = new ClosureDisplayClass();
for (displayClass.i = 0; displayClass.i < 5; displayClass.i++)
{
    actions.Add(displayClass.Lambda);
}

class ClosureDisplayClass
{
    public int i; // 共用的變數
    public void Lambda() => Console.WriteLine(i);
}
```


<!-- endtab -->

<!-- tab 點擊次數 Counter-->


![app_counter](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Closure/app_counter.png)


```csharp
public static Action CreateClickCounter()
{
    int count = 0;
    return () => {
        count++;
        Console.WriteLine($"你已經點了 : {count} 次!!@@@");
    };
}
```

`return () => { ... }` 捕捉了 count，所以 `count` 被放進編譯器隱藏生成的物件裡


## 編譯器轉換過程

```csharp
Func<int> CreateCounter()
{
    int count = 0;
    return () => ++count;
}
```

編譯器轉換版本
```csharp
class Closure
{
    public int count = 0;
    public int GetNext()
    {
        return ++count;
    }
}

Func<int> CreateCounter()
{
    Closure closure = new Closure();
    return closure.GetNext;
}
```

- `count` 原本是區域變數 → 被提升為 `Closure` 類別的欄位
- lambda 不再只是一個方法，而是「帶著記憶體（狀態）的 delegate」
- 即使 `CreateCounter()` 執行完畢，closure 物件還活著（因為 delegate 持有參考）

```csharp
var counter = CreateClickCounter();
counter(); // 你已經點了 : 1 次!!@@@
counter(); // 你已經點了 : 2 次!!@@@
counter(); // 你已經點了 : 3 次!!@@@
```


<!-- endtab -->

<!-- tab 事件計數器-->


![event](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Closure/app2_event_tracing.png)


```csharp
public static Action CreateEventCounter(string eventName)
{
    int count = 0;
    DateTime firstCall = DateTime.Now;
    
    return () => {
        count++;
        var elapsed = DateTime.Now - firstCall;
        Console.WriteLine($"{eventName} 觸發第 {count} 次，距離首次觸發已過 {elapsed.TotalSeconds:F1} 秒");
    };
}

// 使用方式
var buttonClickCounter = CreateEventCounter("按鈕點擊");
var apiCallCounter = CreateEventCounter("API 呼叫");

buttonClickCounter(); // 按鈕點擊 觸發第 1 次，距離首次觸發已過 0.0 秒
buttonClickCounter(); // 按鈕點擊 觸發第 2 次，距離首次觸發已過 1.2 秒
apiCallCounter();     // API 呼叫 觸發第 1 次，距離首次觸發已過 0.0 秒
```


<!-- endtab -->

<!-- tab 累積計算器-->


![accu](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Closure/app3_accumu.png)


```csharp
public static Func<int, int> CreateAccumulator(int initialValue = 0)
{
    int total = initialValue;
    
    return (int value) => {
        total += value;
        Console.WriteLine($"累積值: {total}");
        return total;
    };
}

// 使用方式
var accumulator = CreateAccumulator(100);
accumulator(10); // 累積值: 110
accumulator(20); // 累積值: 130
accumulator(-5); // 累積值: 125
```


<!-- endtab -->

<!-- tab 狀態機模擬-->


![state_history](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Closure/app4_statehistory.png)


```csharp
public static Action<string> CreateStateMachine()
{
    string currentState = "待機";
    var stateHistory = new List<string>();
    
    return (string newState) => {
        Console.WriteLine($"狀態轉換: {currentState} → {newState}");
        stateHistory.Add(currentState);
        currentState = newState;
        Console.WriteLine($"歷史記錄: [{string.Join(" → ", stateHistory)}] → {currentState}");
    };
}

// 使用方式
var stateMachine = CreateStateMachine();
stateMachine("運行中");  // 狀態轉換: 待機 → 運行中
stateMachine("暫停");    // 狀態轉換: 運行中 → 暫停
stateMachine("停止");    // 狀態轉換: 暫停 → 停止
```


<!-- endtab -->

<!-- tab 提供 Retry-->


![Retry](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Closure/app5_retry.png)


```csharp
var promotionTagIds = GetPromotionTagIds();
var outerIds = GetProductOuterIds();
var targetTypeEnum = PromotionTargetType.Product;

Action action = () => this.UpdateProductSkuOuterIdTag(promotionTagIds, outerIds, targetTypeEnum);
```


這裡的 lambda `() => ...` 就是個 Closure，它沒有參數，但它裡面用了「外部的變數」

- `promotionTagIds`
- `outerIds` 
- `targetTypeEnum`


<!-- endtab -->


{% endtabs %}