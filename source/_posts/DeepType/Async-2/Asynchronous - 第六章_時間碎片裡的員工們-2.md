---
title: Asynchronous Programming - 第六章:時間碎片裡的員工們-2
date: 2024-09-26 09:22:05
categories: 未来よりの返歌
top_img: https://github.com/CHI-KEKE/pics/blob/main/Codesss/Async/logoAsync.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Codesss/Async/logoAsync.png?raw=true
tags:
    - Asynchronous Programming
toc:
toc_number:
comments :
---

{% tabs Asynchronous2%}

<!-- tab 共享資源沒有被妥善同步處理-->

共享資源是什麼？

- 一筆資料（例如某筆訂單、座位狀態）
- 一段記憶體（像共用的快取陣列）
- 一個硬體裝置（例如印表機、檔案寫入）
- 一個全域變數或物件

甚至是「系統的鎖資源本身」


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-resource/1_what_can_we_share.png)



當系統有了多個 Thread，一切看似井然有序、效率飛快，但只要一不小心，就會變成一場混亂的現場。這些混亂，我們統稱為同步問題（Synchronization Problems）。這些問題源自一個核心事實，多個執行緒試圖同時存取同一份共享資源，卻沒有適當協調。結果就像幾個員工搶著用一台影印機，可能導致不是印出錯內容、就是永遠卡在佇列。

同步問題常見的四大類型如下，我們以訂票系統來說明

![Image](https://i.imgur.com/AcKmuJf.png)


<!-- endtab -->

<!-- tab Race Condition（競態條件）-->


執行緒 1 和執行緒 2 同時查詢「座位 A1 是否可訂」，兩者皆發現「尚未被預訂」，於是幾乎同時發出訂位請求。

結果座位 A1 被重複預訂，兩人都收到「訂票成功」的通知，實際卻只有一張椅子。

這就是典型的競態條件，快的可能不是對的。若沒有加鎖機制來保護查詢與預訂之間的空窗期，資料就可能出現不一致


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-resource/2_race_condition.png)



<!-- endtab -->

<!-- tab Deadlock（死鎖）-->


假設使用者要一次預訂「座位 A1 + B1 套票」，系統設計會對這兩個座位加鎖以避免重複預訂

執行緒 2 成功鎖定 A1，接著等待 B1。同時，執行緒 3 成功鎖定 B1，卻也正在等待 A1

於是，兩者互相等待對方釋放鎖定，誰也動不了，訂票流程卡住，像兩個人在門口讓來讓去，一讓就是永遠



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-resource/3_deadlock.png)


<!-- endtab -->

<!-- tab Starvation（飢餓）-->


系統有一條排程規則，優先處理取消訂票的請求

結果執行緒 3（取消請求）被不斷插隊處理，而執行緒 2（普通預訂）則一直在等待排隊。久而久之，預訂請求始終得不到執行機會，這就是所謂的「飢餓問題」—— 雖然沒有死鎖，但某些執行緒就是被系統冷落了


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-resource/4_starvation.png)


<!-- endtab -->

<!-- tab Priority Inversion（優先級反轉）-->


執行緒 2（高優先級）負責訂票，它需要一份共享資源（例如票券資料表）。不巧的是，這份資源正被執行緒 3（低優先級）使用中。照理說應該等它用完就讓給高優先級的執行緒。但這時，執行緒 1（中優先級）不小心插隊進來，佔用了大量 CPU，導致低優先級的執行緒一直無法釋放資源，高優先級的訂票也只能乾等。

多執行緒可以讓系統飛快運行，但也像交通燈壞掉的十字路口，大家都在前進，卻也都在撞車邊緣徘徊。解決同步問題，靠的是設計良好的鎖定策略（如 Mutex、Semaphore）、資料保護機制（如 lock 區塊、atomic 操作），以及對流程優先權的細緻規劃


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-resource/4_starvation.png)



<!-- endtab -->

<!-- tab Multi-threading 做法-->


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-resource/5_thread_and_process.png)


一個主執行緒監聽請求，每收到一筆就從 Thread Pool 派一個 Thread 處理

記憶體共用，請求上下文易於共享

須注意 thread 數不能太多，否則上下文切換變重


<!-- endtab -->

<!-- tab Multi-processing-->

啟動多個 worker process，每個都有獨立的處理邏輯與記憶體空間

高隔離性，穩定性高，但彼此不能共享快取，會增加記憶體


| 決策維度                 | Multi-threading         | Multi-processing                |
| -------------------- | ------------- | ------------------- |
| **資源共享需求高**          | ✅             | ❌（隔離）               |
| **任務互相影響風險高（崩潰要分開）** | ❌             | ✅                   |
| **啟動速度要求高**          | ✅（Thread 啟動快） | ❌（Process 重）        |
| **系統安全性需求高**         | ❌（容易踩到別人的記憶體） | ✅（隔離清楚）             |
| **記憶體敏感（資源有限）**      | ✅             | ❌（每個 Process 都吃記憶體） |
| **要發揮多核心效能**         | ✅（可部分發揮）      | ✅（每個 Process 各跑核心）  |



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-resource/6_table.png)


<!-- endtab -->


<!-- tab API Web Server-->


> ✅ Multi-threading ＋ 非同步（Asynchronous I/O）


雖然 Multi-Processing 可以用來跑 Web Server，但大部分的高效能 Web Server 架構會傾向這樣的設計

- 一個 Process 負責接收請求，管理主流程
- 在該 Process 裡用多個執行緒（Thread）來同時處理多個請求
- 並進一步結合 async/await（非同步 I/O）來降低資源消耗

| 考慮面向                 | 說明                                                                     |
| -------------------- | ---------------------------------------------------------------------- |
| **資源共享需求**           | 每個請求都可能查詢資料庫、存取共用快取（如 Redis），用 Thread 不需額外 IPC                         |
| **記憶體成本**            | Thread 共用記憶體，比 Process 省空間、建立成本也低                                      |
| **回應速度需求**           | 使用 Thread + 非同步 I/O，能有效提升高併發處理能力（例如 ASP.NET Core、Node.js）              |
| **Thread Pool 技術成熟** | 現代 Runtime（例如 .NET、Java、Python）都有 Thread Pool 支援，能自動重用 Thread，效能高且避免開銷 |


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-resource/7_web_server.png)


<!-- endtab -->

<!-- tab Chrome 使用 Multi-Process 架構-->


防止單一頁面崩潰拖垮整個瀏覽器，如果你還記得早年的 IE 或 Firefox，當其中一個分頁當掉（或某個 JavaScript 無限迴圈），整個瀏覽器會整個凍住

這是因為

- 傳統瀏覽器使用單一 Process + 多 Thread 架構
- 所有頁面（分頁）都在同一個記憶體空間裡
- 一旦某個分頁發生記憶體錯誤、崩潰、資源飆高
- 整個 Process 就掛了，導致所有頁面同時消失

另外這樣做可以強化安全沙箱機制（Sandbox），Chrome 的「每個頁面是獨立 Process」搭配沙箱設計，讓 JavaScript 只能在自己的 Process 裡跑，每個分頁都不能直接操作系統資源（檔案、硬體），想跨分頁、存其他頁資料 → 必須透過 Chrome 的主程序做 IPC（跨進程通訊），這樣即使有惡意程式碼注入（XSS）、外部漏洞利用，也會被限制在該 Process 內，無法影響整體系統或其他分頁。這是「Multi-Process + Least Privilege」的安全設計


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-resource/8_chrome.png)


<!-- endtab -->

<!-- tab IDE-->


一個開發工具（像 Visual Studio）裡，它把所有功能放在同一個 Process，但拆成多條 Thread，同時做不同事，才能在不犧牲回應速度的情況下，把 IDE 做到又強又即時。如果你一邊打字，IDE 還要一邊補全、分析、檢查錯誤，不拆 `Thread` 的話，你每打一個字都會卡住，但若改採用多 Process 的架構，因為跨 Process 溝通太慢又太複雜，IDE 需要的是「極低延遲」而不是「強隔離」


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-resource/9_IDE.png)



<!-- endtab -->

{% endtabs %}