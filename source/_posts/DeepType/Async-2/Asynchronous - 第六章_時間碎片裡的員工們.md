---
title: Asynchronous Programming - 第六章:時間碎片裡的員工們
date: 2024-09-26 09:22:05
categories: 未来よりの返歌
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-process/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-process/1_landing.png
tags:
    - Asynchronous Programming
toc:
toc_number:
comments :
---

[huanlintalk](https://www.huanlintalk.com/2013/04/csharp-notes-multithreading-1.html)
[莫力全 Kyle Mo](https://oldmo860617.medium.com/)
[Program,Process,Thread](https://programming.im.ncnu.edu.tw/J_Chapter9.htm)
[青耀隨筆談](https://qing-yao.blogspot.com/2016/08/writeByMind-2.html?source=post_page-----94a40721b492--------------------------------)

{% tabs Asynchronous2%}

<!-- tab Program、Process、Thread-->

想像你是一位準備開店的廚師，桌上攤開一本厚重的菜譜，這就是你的 `Program`（程式碼集合）。它記錄了每道菜的做法、調味的比例、上菜的順序……但這些指示仍停留在紙上，尚未進入現實。它像是還未走進世界的夢，只存在於設計之中。

當你真正開張營業、招呼客人，這份菜譜就被實體化成了一間 `Process`（處理程序）一間運作中的餐廳。這間餐廳開始佔據空間（記憶體）、使用瓦斯爐（CPU）、冰箱（硬碟），並啟動了你的夢想。每一間正在運作的餐廳，都是一個獨立的 `Process`，就像你電腦上同時開著 Word、Chrome 和 Spotify，一家店煮咖哩、一家店沖咖啡，彼此不會互相干擾。。


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-process/2_program_process.png)




但餐廳開起來只是第一步，真正讓整個流程流動的，是餐廳裡的「人」—— 這些人，正是 `Thread`（執行緒）。一家只有一位大廚的小餐館，也許只能一張一張煎魚、一道一道上菜（單執行緒）；但一家人潮洶湧的熱炒店，就需要多位大廚同時開火、服務生穿梭上菜（多執行緒），讓整個店面井然有序地高速運作。
![Image](https://i.imgur.com/FxoT20S.png)
![Image](https://i.imgur.com/lfrGXhO.png)


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-process/3_thread.png)


<!-- endtab -->

<!-- tab 當世界還只能一條路-->

![Image](https://i.imgur.com/v5i7hCT.png)

**單一任務（Single Tasking）／線性執行（Sequential Execution）**

單一任務的本質，是用「一次只做一件事」來換取可預測性與思考成本的最低化，這不是因為它快，而是因為在資源極少、系統複雜度不能失控的年代，人類需要一種「腦袋能跟得上」的執行模型

在計算機的早期時代，程式的執行方式非常單純，就像一位職人，一次只能處理一件事。這種模型稱為 單一任務（Single Tasking） 或 線性執行（Sequential Execution）。

想像這位職人開了一家只有他一人的餐廳，他會專心地切菜、煮湯、上菜，但這些事必須一件一件來，切菜沒完，湯就不能開始煮，湯沒煮完，客人就只能等著挨餓

這種方式行為可預測、邏輯簡單，對初學者或早期系統來說特別友善。然而缺點是任何一個步驟卡住（如等待輸入、磁碟存取），整間餐廳就陷入停擺，所有任務只能排隊等待，效率極低。

舉例來說，當你按下「列印」鍵，整個作業系統會乖乖等印表機印完文件，才能繼續幫你開啟 Word 文件或播放音樂。那是一個「同步的純粹年代」


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-process/4_single_sequential.png)


<!-- endtab -->

<!-- tab 餐廳開多間了，但廚師還是只有一個-->

![Image](https://i.imgur.com/k15z9Sm.png)

為了把「會一起死的東西」切開，讓錯誤不要擴散，就是 `Process`（處理程序） 的誕生。每個 `Process` 就像一間獨立餐廳，擁有

- 自己的廚房（記憶體空間）
- 自己的設備與員工（變數與執行環境）
- 自己的顧客（任務）

這種設計讓穩定性大幅提升。即使某間餐廳著火了（`Process` 當掉），其他餐廳依舊照常營運，互不影響，作業系統負責管理與切換 Process，OS 會決定「現在 CPU 要跑哪一間餐廳」

![Image](https://i.imgur.com/pkkSXkE.png)


在單核心 CPU 下，永遠同一時間只跑一個 Process，其他的都是「在等」
當某間餐廳（例如 `Process`）陷入無限迴圈，CPU 被困在那裡無法脫身，導致整個城市的其他餐廳也停擺，即使其他餐廳閒置（如程式在等待磁碟 I/O），這位廚師也不能輕易跳去支援，因為 `Process` 的排班方式是以整間餐廳為單位，切換成本高、彈性低。CPU 的時間無法被有效運用，導致浪費



<!-- endtab -->

<!-- tab 員工來了—— Thread 的誕生-->

![Image](https://i.imgur.com/4no7koj.png)

Thread 的本質，其實是在把「CPU 的使用權」切得更細。這不是為了變出更多 CPU，而是避免有人在等的時候，整間餐廳都跟著空轉。
資源隔離的單位仍然是 Process——記憶體、位址空間、安全邊界，這些界線沒有變；變的是，CPU 不再只排「整間餐廳」，而是開始排「餐廳裡的員工」



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-process/5_time_slice.png)


於是 Thread 成為 CPU 排程的最小單位。Scheduler 把 CPU 的時間切成一小片一小片，輪流借給各個 Thread，用的速度快到人類感覺不出切換，只會以為它們是在「同時」執行，其實只是高速輪流而已。這讓每間餐廳（Process）終於不用只靠一位廚師硬撐。開始請來多位員工（Thread）分工合作，一位專心備料、一位負責煮湯、一位負責上菜

他們共用同一間廚房──同一個記憶體空間；但每個人都有自己的筆記本與待辦清單，也就是各自獨立的呼叫堆疊與區域變數。所以大家可以同時做不同的事，而不會踩到彼此的進度，整體效率自然就上來了。

從結構上來看，每個 Process 都包含一份記憶體空間，以及一個以上的 Thread。而每個 Thread 內部，則帶著自己的 Stack，用來記錄從 main 開始一路走到現在的函數呼叫路徑，同時保存暫存器狀態，像是 Program Counter、Stack Pointer，確保 CPU 隨時知道「我剛剛跑到哪裡」。也因為這樣，Thread 雖然能存取相同的共享資源（例如同一個 Object），但它們的局部變數彼此獨立，互不干擾——就像共用廚房，卻不會偷看對方的筆記本。

最後一定要釐清的一點是：Thread 並不是切出一塊硬體資源專屬使用。它們共享的是同一顆 CPU，只是透過時間片（Time Slice）分配的方式，輪流上場。就像只有一口爐子的廚房，所有廚師都得排隊用火。只是每個人分到的時間短到幾乎感覺不到切換，看起來才會像是「大家同時在煮」。

<!-- endtab -->

<!-- tab 換人上場的代價：執行緒的切換成本-->

![Image](https://i.imgur.com/5Gkamyc.png)


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-process/6_context_switch.png)


雖然執行緒的出現讓整體系統更靈活，但在「換人上場」的瞬間，其實暗藏了不少代價，這就是所謂的上下文切換（Context Switch）。而從人類角度來說，我們每天也在做類似的切換：回家寫程式前，你可能正忙著回主管的訊息、思考晚餐要吃什麼。當你轉身坐下來面對鍵盤，腦袋是不是需要一段時間才能進入「開發模式」？這種「切換情境」的疲憊，就是 Context Switch 的真實寫照

<!-- endtab -->

<!-- tab CPU 的角色輪替-->

對於只有一顆 CPU 的電腦來說，同一時間只能執行一件事情。當作業系統同時載入多個程式，每個程式裡又有多個執行緒在等待「上場」，這時就必須靠排程器（Scheduler） 切割 CPU 的時間。

這就像一座只有一個舞台的小劇場，而每個執行緒都是候場的演員。每個人只能輪流上台，表演幾秒鐘後，就必須讓出舞台給下一位。這樣的切換過程就叫做 Context Switch。

每一次 Context Switch，系統都要進行三個步驟：

- 存檔：把目前執行緒的暫存器資料（如程式計數器、堆疊指標等）保存下來，像是演員記下自己演到哪一句台詞。
- 挑人：由排程器選出下一位要上台的執行緒，如果這位演員來自另一部戲（另一個 Process），還要換佈景（切換虛擬位址空間）。
- 載入：把新執行緒的暫存器資料載入，讓 CPU 可以無縫接續新的任務。


Context Switch 會帶來幾種資源損耗

- CPU 時間消耗，每次保存與恢復暫存器、更新排程資料，都是額外的 CPU 工作，等於花掉寶貴的運算時間在「換人」上。
- 快取失效（Cache Miss），CPU 有快取記憶體來加速存取，但執行緒一換，原本載入的資料可能馬上就沒用了，只能重新載入新資料，效能反而降低。
- 記憶體訪問延遲，不同 Process 間的切換涉及到虛擬記憶體管理，例如頁表（Page Table）與 TLB 的更新，這會造成更多的記憶體存取延遲。
- 指令管線刷新（Pipeline Flush），現代 CPU 使用管線化（Pipeline）技術來提升執行效率。但每次切換 Thread 時，原本排好的指令隊伍得清空，重新安排，導致效率下降。
- 系統管理開銷，作業系統還得額外維護執行緒的狀態、排程計數器、優先權等系統資料結構，這些也都是成本。
- 記憶體額外負擔，每個執行緒都需要獨立的堆疊與暫存空間來儲存它的「上下文」，這會吃掉不少記憶體。

不只是執行效能，執行緒數量的多寡也影響 .NET 的垃圾回收（GC） 機制。在進行資源回收時，CLR 必須暫停所有執行緒（Stop the World），等回收結束後再讓它們恢復運作。Thread 越多，越難同步停下、又越慢能重啟。同樣情況也發生在除錯時：當你設定中斷點，整個應用程式的所有 Thread 都會被暫停，直到你按下「繼續」或「單步執行」，這些執行緒才會再次「醒來」。



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-process/7_stop.png)



太多 Thread，就像請了太多員工卻讓他們輪流站崗、互相打斷，反而讓餐廳變得混亂。好用的不是 Thread 多，而是分配得宜、配合默契。



![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-process/6_2_context_switch_disadvantage.png)



<!-- endtab -->

<!-- tab 當 Thread 遇上外來者：非託管 DLL 的插手-->

在我們設計多執行緒系統時，光考慮 .NET 世界內部的行為還不夠。有時候，一些「外來者」的行動，也會影響整體效率，這些外來者，就是 `Unmanaged DLL`（非託管動態連結庫）。

![Image](https://i.imgur.com/KBBbpQW.png)

`Unmanaged DLL` 就像是一位外包廠商：你可以叫它來幫忙某些工作（如影像處理、硬體驅動、系統呼叫），但它不住在你家（不受 .NET 管理），你也無法輕易規範它的生活作息（記憶體管理、錯誤處理等）


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-process/8_unmanaged_dll.png)


這些 DLL 多半是使用 C、C++ 等原生語言撰寫，直接操作底層資源，與作業系統互動密切。相較之下，`Managed DLL`（受管程式庫） 是由 .NET CLR 管理的一等公民，會自動處理記憶體分配與垃圾回收，像是你家的家人，規則一致、互動流暢

![Image](https://i.imgur.com/EgXIRDw.png)

傳統的 .NET Framework 中，每當程式創建或銷毀執行緒時，CLR 會主動通知所有已載入的 Unmanaged DLL：「嘿，我要加一個新員工（Thread）囉！」或者「這位員工要離職了！」

這讓 DLL 有機會做初始化或清理的動作，例如

- 分配這條執行緒要使用的原生資源
- 登記 Thread Local Storage（執行緒區域儲存）
- 釋放或歸還所佔記憶體

但也因此，每一次執行緒的生命周期都可能帶來額外的隱藏成本。這就像你每請一位外包廠商來幫忙，都必須多花一段時間辦入廠證、做教育訓練、設定帳號密碼，離開時還要交接、回收門禁卡

因此若在高頻創建與銷毀 Thread 的應用中（例如即時影像處理、交易撮合系統），大量與 Unmanaged DLL 交互，這些額外的進出場流程可能造成效能瓶頸，甚至非預期的行為。尤其當這些 DLL 寫得不夠穩定，還可能造成

- 記憶體洩漏（未釋放資源）
- 程式崩潰（未處理例外）
- 執行緒被鎖住（未釋放鎖定）


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-process/9_framework.png)



而 .NET Core 改用更輕量的 Thread 建立機制，不再預設廣播 Thread 建立與終結事件給所有 Unmanaged DLL。除非某個 DLL 明確要求這些通知（例如註冊 TLS 回調），否則 CLR 會避免這些開銷。

這樣的設計帶來幾個好處：

- 減少 Thread 建立/銷毀 的成本
- 提升高頻 Thread 操作的效能穩定性
- 避免不必要的 native 層干擾與風險


如果正在開發的是 .NET Core 或 .NET 6+ 的應用，尤其又在意效能、Thread 池行為或 P/Invoke 穩定性，這樣的改善會讓你更安心地使用多執行緒架構，而不必太擔心與 Unmanaged DLL 的互動成本
但如果還在跑傳統 .NET Framework（特別是 Windows Forms、WPF 桌面應用），那就需要特別注意 Thread 與 native DLL 的互動模式，甚至考慮改用 Thread Pool 或 Task-based 架構以避免高頻 Thread 操作。


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-process/9_2_core.png)


<!-- endtab -->



<!-- tab summary-->


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-6-threads-process/10_final.png)


<!-- endtab -->


{% endtabs %}