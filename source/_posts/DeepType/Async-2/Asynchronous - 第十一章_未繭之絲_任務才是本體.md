---
title: Asynchronous Programming - 第十一章:未繭之絲：任務才是本體
date: 2025-07-04 09:22:05
categories: 未来よりの返歌
top_img: https://github.com/CHI-KEKE/pics/blob/main/Codesss/Async/logoAsync.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Codesss/Async/logoAsync.png?raw=true
tags:
    - Asynchronous Programming
toc:
toc_number:
comments :
---

[使用 .NET Async/Await 的常見錯誤](https://blog.darkthread.net/blog/common-async-await-mistakes/)
[正確理解 C# async 與非同步](https://www.opasschang.com/docs/understand-csharp-asyn)

{% tabs Asynchronous%}

<!-- tab 蠶的軀殼-->

在多執行緒的森林裡，有些線，早已抽離了蠶的軀殼，它們不需繭的庇護，也能沿著記憶結點游走。我們誤以為是 await 編織了非同步的絲線，卻忘了真正編織的是那些未繭的任務。只要 Task 尚在，無論繭在與否，線都還在地底延展，繼續牽動著下一個狀態機的呼吸

![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/silkworm.png)

<!-- endtab -->

<!-- tab 非同步的本質：回傳 Task 才是真的非同步-->

非同步的核心是 `Task` 或 `Task<T>` 型別。只要方法回傳 `Task`，它就是非同步，因為 `Task` 代表著「進行中的工作」。是否使用 `await` 不影響方法本身是否非同步，`await` 只決定要不要在這裡把結果拿回來、接著做後續的邏輯。`await` 的本質是「把非同步操作的結果，用同步的方式 等待完成後，接著執行後續的程式碼」。


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-11-task-is-key/1_task_is_key.png)


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-11-task-is-key/2_await_is_wait.png)


## 沒用 await

```CSHARP
public Task<int> GetDataAsync() {
    return GetDataFromDbAsync(); // 直接把 Task 回傳
}
```

GetDataFromDbAsync() 會回傳一個執行中的 Task。呼叫端收到後可以選擇：先做其他事 → 等需要時再 `await`

```CSHARP
var t = GetDataAsync(); // t 是 Task<int>，執行中
// ... 可做其他事
var result = await t;   // 在需要時等待結果
```


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-11-task-is-key/3_no_await_is_fine.png)


## 加 await → 只是把「等待」內嵌在方法內

```CSHARP
public async Task<int> GetDataAsync() {
    return await GetDataFromDbAsync();
}
```

差別在於

- 有 await 時，C# 編譯器會多生成一個狀態機
- 狀態機裡會記住「執行到哪裡停下來」與「完成後去哪裡繼續」
- 如果 Task 失敗了，例外也會從 await 處直接拋出


加 await 是為了「攔住例外」與「插入後續邏輯」，不是為了讓它變成非同步! 回傳 Task 是一種合約，我負責啟動，你負責決定什麼時候等，並且 GetDataFromDbAsync() 在沒有 await 的情況下 不會自動變成 fire-and-forget，只有在 Task 沒被任何人接住、存起來、或 await 時，才是 fire-and-forget


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-11-task-is-key/4_await_cost.png)


<!-- endtab -->

<!-- tab 什麼時候可以不加 await？什麼時候一定要？-->

## 【可以不加 await 的情況】

方法裡沒有後續邏輯要等 `Task` 結果處理，最後一行直接 return 執行後的 `Task` 就好，不需要多餘的狀態機

- Library 或中介層只把別人的 `Task` 包裝後丟出去，呼叫端自己決定要不要 `await`
- 組合場景 : 例如用 `Task.WhenAll`、`Task.WhenAny` 等

👉 這樣做能省掉狀態機生成，更高效，也不會多出執行緒上下文切換



![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-11-task-is-key/6_when_to_skip_await.png)


## 【一定要加 await 的情況】

有兩個情境，一定要 await，否則執行結果永遠抓不到執行中的例外，只抓同步階段的

<br>

#### 🔹 try/catch

```CSHARP
try {
    return await SomeAsync();
} catch (Exception ex) {
    // 這裡可以抓到 SomeAsync 執行期間的例外
}
```

因為 `await` 才會讓例外回到目前執行緒，try/catch 才能接住！


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-11-task-is-key/7_should_await_try_catch.png)


#### 🔹 using

```CSHARP
using (var conn = new SqlConnection(...)) {
    return await conn.DoSomethingAsync(); 
    // 執行完才 Dispose()
}
```

馬上離開 using，可能還在用就被釋放！await 確保資源在非同步操作完成後才釋放


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-11-task-is-key/8_should_await_dispose.png)



<!-- endtab -->

<!-- tab 狀態機的工作：記得「下一步要去哪裡」-->

await 產生狀態機（MoveNext）來銜接「尚未完成的 Task」與「後續邏輯」，await 的存在價值不是等結果，而是讓「還沒跑完的程式碼」在未來某一刻接得回來繼續跑

## 1️⃣ 編譯器看到 async + await

編譯器會幫你生一個 狀態機 struct/class，原本的方法內容被拆進 MoveNext()


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-11-task-is-key/9_await_machine.png)


## 2️⃣ 執行到 await someTask

如果 someTask 已完成，直接往下跑（同步路徑），如果 someTask 尚未完成，當下執行緒「退出」，記住目前狀態（state），把「後續要跑的 MoveNext」掛到 Task 的 continuation，本質就是把後續邏輯包成  continuation，等 Task 完成後再回來跑


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-11-task-is-key/10_await_continue.png)


## 3️⃣ Task 完成時

Task 觸發 continuation，狀態機的 MoveNext() 再被呼叫一次，依 state 決定要走哪個分支繼續執行


- 有後續邏輯要執行 → 必須用 await。
- 只是 return await ... → 不需要，除非包在 try/catch 或 using。


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-11-task-is-key/11_await_future_follow.png)

<!-- endtab -->


<!-- tab summary-->


![silkworm](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-11-task-is-key/5_await_table.png)


<!-- endtab -->

{% endtabs %}