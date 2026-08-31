---
title: VM vs Docker Container
date: 2026-03-31 08:24:05
categories: Docker
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
tags:
    - Docker
toc:
toc_number:
comments :
---


{% tabs VM vs Docker Container %}

<!-- tab 概念-->

`VM` 是把「整台電腦」重做一份出來。它有自己的硬體模擬層、自己的作業系統、自己的開機流程，就像真的多養了一台機器。

`Container` 則不同，它直接借用同一個 `Host OS` 的核心，只把應用程式執行所需的空間隔開來，不需要額外的作業系統。

兩者解決的核心問題其實不一樣：
- `VM` 解的是「我需要一台完整、隔離的電腦」
- `Container` 解的是「我需要一個乾淨、一致的程式執行環境」

你要的是一個能安全執行程式的環境，但不是每次都值得為了跑一個程式，重新養一整套作業系統。理解這個出發點的差異，才能知道什麼時候該用哪個



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/vm_container/1_vm_container_simple.png)


<!-- endtab -->

<!-- tab VM 的思考方式-->

`VM` 的核心概念是：**模擬出一台完整的電腦**，讓上面跑的作業系統完全感知不到自己是虛擬的。

它的啟動流程走的是完整的開機路徑：

- `Hypervisor` 先虛擬出一組硬體資源
  - 分配 `vCPU`、`RAM`、虛擬磁碟給這台 VM
  - VM 內部看到的這些資源，跟真實硬體沒有差別
- 虛擬機走完整的開機流程
  - `BIOS` / `UEFI` 初始化
  - 找到 `bootloader`（例如 GRUB）
  - 載入 `OS kernel` 進記憶體
  - 核心初始化完成後，把基礎系統服務一一拉起來
  - 例如 `systemd`、網路管理、時間同步、日誌服務等
- 最後才輪到你的應用程式
  - `Nginx`、`Node.js`、`MySQL` 這類服務，全都是在整套 OS 就緒後才啟動的

這代表 `VM` 先把「那台電腦」完整活起來。app 只是這台機器上眾多住戶的其中一個，開機流程根本不在意它



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/vm_container/2_the_whole_machine.png)


<!-- endtab -->

<!-- tab Container 的思考方式-->

`Container` 的核心概念是：**不用重開一台電腦，直接切一塊隔離的執行空間給程式跑**。

它的啟動流程比 VM 短得多：

- 執行 `docker run`，告訴 Docker daemon：我要一個隔離的執行環境
- Docker 從 `image` 取出應用所需的所有東西
  - 包含 app 本身、`library`、執行環境設定
  - 用 `overlay filesystem` 疊成容器看到的完整檔案系統視圖
- `Linux kernel` 用 `namespace` 切出隔離空間
  - 讓容器有自己的行程視角、網路介面、掛載點、`hostname`
  - 容器內部感覺像是一個獨立環境，但其實共用同一個 `kernel`
- 用 `cgroup` 設定資源上限
  - 限制這個容器最多能用多少 `CPU`、`RAM`、`IO`、`PIDs`
  - 防止某個容器失控，把整台 Host 的資源吃光
- 直接啟動你指定的主程式
  - 容器裡的 `PID 1` 通常就是你要跑的 app 本身
  - 整個過程不需要跑完 OS 開機流程

這代表 `Container` 的設計哲學是 **環境是為程式服務的，程式才是主角**，而不是讓程式去等一台機器開好


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/vm_container/3_pure_seperate_app.png)


<!-- endtab -->

<!-- tab 為什麼這樣設計，結果就會差很多-->

`VM` 因為要把整套 `OS` 開起來，每個環節都有成本：`BIOS` 初始化、`kernel` 載入、系統服務啟動，每一步都在燒時間和記憶體。

`Container` 直接借用 `Host` 的 `Linux kernel`，只補齊應用需要的隔離空間和相依套件，省掉了整個 OS 開機流程，所以在幾個維度上自然差很多：

| 維度 | VM | Container |
|------|-----|-----------|
| **啟動時間** | 通常數十秒到數分鐘 | 通常不到 1 秒 |
| **記憶體佔用** | 每台 VM 需要獨立的 OS 記憶體 | 多個容器共用同一個 kernel，記憶體更省 |
| **密度** | 一台機器能跑的 VM 數量有限 | 同一台機器可以同時跑更多容器 |
| **隔離程度** | 強，擁有獨立 kernel，互不干擾 | 較弱，共用 kernel，存在逃逸風險 |

它們一開始就是針對不同問題設計的，所以在這些維度上的差距，不是其中一個設計得比較差，而是取捨本來就不同。



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/vm_container/4_shorttime.png)


<!-- endtab -->

<!-- tab namespace-->

`Container` 本身並沒有一套獨立的 `OS`，它是靠 `Linux kernel` 提供的 `namespace` 機制，讓每個容器「看起來像活在自己的世界裡」。

`namespace` 的作用是**視角隔離**：讓容器只能看到屬於自己的資源，感知不到同一台主機上的其他容器。

| Namespace 類型 | 隔離的是什麼 | 實際效果 |
|----------------|-------------|---------|
| `PID` | 行程編號 | 容器只看得到自己內部的行程，看不到 Host 或其他容器的 process |
| `NET` | 網路介面 | 容器有自己的 `IP`、虛擬網卡、路由表 |
| `MNT` | 掛載點 | 容器有自己的檔案系統視圖，不會看到 Host 的其他目錄 |
| `UTS` | 主機名稱 | 容器可以有自己獨立的 `hostname` |
| `IPC` | 行程間通訊 | `shared memory`、`semaphore` 等資源與外部隔離 |
| `USER` | 使用者身份 | 容器內的 `UID` / `GID` 可以與 Host 不同，增加安全性 |

這六種 `namespace` 加在一起，就構成了容器「感覺像獨立環境」的基礎。但要強調的是：這只是**視角上的隔離**，容器仍然共用同一個 `Linux kernel`，並不是真正擁有一套自己的 OS。


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/vm_container/5_namespace.png)


<!-- endtab -->

<!-- tab cgroup-->

`namespace` 解決的是「看到什麼」，而 `cgroup`（Control Groups）解決的是「能用多少」。

就算容器只看得到自己的行程，如果沒有限制，它還是可以把整台 Host 的 `CPU`、`RAM` 全部吃掉。`cgroup` 就是防止這件事發生的機制。

| 資源類型 | 限制說明 | 實際用途 |
|----------|---------|---------|
| `CPU` | 最多使用幾顆核心、佔用多少比例 | 避免單一容器獨占所有運算資源 |
| `Memory` | 最多使用多少 `RAM` | 超出上限會觸發 `OOM Kill`，保護 Host 穩定性 |
| `IO` | 磁碟讀寫與網路流量的頻寬上限 | 防止 IO 密集容器拖垮整體效能 |
| `PIDs` | 最多能建立幾個行程 | 防止容器透過 `fork bomb` 耗盡系統資源 |

`cgroup` 與 `namespace` 共同構成了容器隔離的兩個核心支柱：一個負責**隔離視角**，一個負責**限制用量**



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/vm_container/6_cgroup.png)


<!-- endtab -->

<!-- tab 抽象-->

雖然 `VM` 和 `Container` 都在做「隔離」這件事，但它們抽象的層次根本不同：

- `VM` 抽象的是「**整台機器**」
  - 它模擬的是硬體層，讓 `OS` 以為自己跑在真實的電腦上
  - 你可以在 VM 裡裝任何 OS、改任何核心設定，完全不受 Host 限制
  - 每台 VM 都是一個獨立、完整的電腦實體

- `Container` 抽象的是「**程式執行環境**」
  - 它不模擬硬體，而是把 `OS` 以上的那一層打包起來
  - 包含 `library`、設定檔、執行環境，但不包含 `kernel`
  - 多個容器共用同一個 `kernel`，只是各自看到不同的環境視角

這個抽象層次的差異，決定了它們適合的使用情境、能達到的隔離程度，以及你在操作時感受到的重量差異


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/vm_container/8_abstraction.png)



<!-- endtab -->

<!-- tab 開一個測試環境跑舊系統-->

假設你有一套老系統，只能跑在特定版本的 `Ubuntu`，甚至需要特定的 `kernel module` 才能正常運作。

這種情境用 `VM` 比較合理，原因是：

- 你需要的不只是「跑一個 app」，而是**完整的 OS 控制權**
- `VM` 可以讓你自由指定 `OS` 版本、安裝客製化 `kernel`、調整底層系統行為
- 整套環境可以打包成 `VM image`，原封不動搬到另一台機器上，就像複製一台伺服器

如果改用 `Container` 會遇到根本性的限制：

> `Container` 共用 `Host` 的 `Linux kernel`，你沒辦法在容器裡換 `kernel`、載入特定 `kernel module`，或是改變底層 OS 行為。這些需求 `Container` 天生就做不到。

所以當你的需求涉及「我需要掌控整個作業系統」而不只是「我需要跑這個程式」，`VM` 才是正確答案


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/vm_container/9_when_to_use_vm.png)



<!-- endtab -->

<!-- tab 快速部署一個 Web API-->

假設你今天做了一個 `Node.js` API，需要在本機開發、測試環境、預備環境、正式環境都能跑，而且行為要完全一致。

這種情境用 `Container` 非常適合，因為你可以用 `Dockerfile` 把這三樣東西一起封進 `image`：

- 程式碼本身
- 執行環境（`Node.js` 版本、runtime 設定）
- 所有相依套件（`npm install` 的結果）

打包成 `image` 之後，不管在哪台機器、哪個環境執行 `docker run`，跑起來的結果都是一樣的。這就是「**環境一致性**」，也是 `Container` 最核心的價值之一。

如果改用 `VM` 來做同樣的事：

> 你得在每台機器上分別安裝 OS、設定 `Node.js` 版本、執行 `npm install`、維護系統更新。每次部署都是一次完整的環境準備，交付會變重，速度也慢很多。

`Container` 把「環境」變成了可攜帶的產出物，讓你的部署流程從「裝環境再跑程式」變成「直接跑已經準備好的環境」


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/vm_container/10_when_to_use_container.png)


<!-- endtab -->

<!-- tab 啟動快不代表所有情境都適合用 Container-->

`Container` 的優勢很明顯：啟動快、資源省、部署方便。但這些優勢建立在「共用 `kernel`」這個前提上，而這個前提本身就帶來了限制。

以下情境仍然適合用 `VM`，而不是 `Container`：

- **強依賴特定 `kernel` 行為的系統**
  - 例如需要載入特定 `kernel module`、調整 `kernel parameter`，或依賴特定版本 OS 的底層行為
  - `Container` 共用 `Host kernel`，無法替換或客製化

- **需要完整 OS 控制權的場景**
  - 例如跑 Windows 應用程式、需要 GUI 環境、或要模擬完整的網路拓撲
  - `Container` 在 Linux Host 上只能跑 Linux 容器，無法跨 OS

- **對隔離要求非常高的任務**
  - 例如多租戶環境中執行不信任的程式碼，或金融、醫療等對安全隔離有法規要求的場景
  - `Container` 的隔離是軟體層級，`kernel` 漏洞可能造成逃逸；`VM` 的 `Hypervisor` 隔離更為強固

選擇工具要從需求出發，而不是從工具的特性反推需求



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/vm_container/7_trades_offs.png)


<!-- endtab -->

<!-- tab summary-->

`VM` 是先把舞台整個蓋好才讓演員上場，`Container` 是先清一塊位置就直接開演。

這個差異直接決定了平常最有感的兩件事：**啟動速度** 跟 **操作重量**。

今天如果只是想跑一個 `Nginx`，去看 `VM` 那套 `BIOS`、`bootloader`、`systemd` 的流程，就會理解它為什麼慢。因為它根本不是「啟動一個程式」，而是在「啟動一台機器」，程式只是裡面的住戶之一

`Container` 先快點把你的 app 拉起來，其他的都是服務它的


最後一個思考框架：

| | `VM` | `Container` |
|-|------|-------------|
| **抽象層次** | 整台機器 | 程式執行環境 |
| **隔離機制** | `Hypervisor` 硬體虛擬化 | `namespace` + `cgroup` |
| **啟動成本** | 高（需要跑完整 OS） | 低（直接跑 app） |
| **適合情境** | 需要完整 OS 控制、強隔離 | 部署一致、快速擴展 |




![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/vm_container/11_stage.png)



<!-- endtab -->

{% endtabs %}

