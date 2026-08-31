---
title: Blue-Green Deployment — Step 1 實作
date: 2026-05-12 08:39:00
categories: CI / CD
top_img: https://github.com/CHI-KEKE/pics/blob/main/CICD/feather.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/CICD/feather.png?raw=true
tags:
    - 
toc:
toc_number:
comments :
---


{% tabs Blue-Green-Deployment-Step1%}


<!-- tab 流程圖-->

Step 1 的核心是 **平行跑兩條線**：A 線部署第一台測試機、B 線同步把 Green group 的機器準備起來，兩者同時進行互不等待，全部完成才進入 Step 2


![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1-implement/1-2-line.png?raw=true)



```mermaid
flowchart TD
    J1[⚙️ Jenkins 啟動] --> J2[載入 Jenkinsfile / Shared Library]
    J2 --> J3[Checkout WebStore master]
    J3 --> J4[Copy HK Prod 設定檔]
    J4 --> S1[進入 Step 1]
    S1 --> PAR{⚡ 平行執行}

    PAR --> A1
    PAR --> B1

    subgraph A["🔴 A 線：Prepare First Machine"]
        direction TB
        A1[發 Slack：第一步開始] --> A2[下載 WebAPI / MobileWebMall\nartifact]
        A2 --> A3[同步到 SG-HK-MWeb21 red]
        A3 --> A4[關閉 red CSS / JS CDN]
        A4 --> A5[IISReset + Health Check]
        A5 --> A6[🔔 通知 RD 鎖 hosts 測試]
    end

    subgraph B["🟢 B 線：Prepare Green ASG"]
        direction TB
        B1[確認 Green weight = 0%\nBlue weight = 100%] --> B2[StartAllEC2]
        B2 --> B3[Ping red / blue / green]
        B3 --> B4[RegisterEC2toELB\nMWeb23 加入 ELB]
        B4 --> B5[Health Check MWeb23\n多個 endpoint 確認]
        B5 --> B6[UnregisterEC2fromELB\nMWeb21 移出 ELB]
        B6 --> B7[Wait 180s draining]
        B7 --> B8{持續 poll\nGreen ASG InService}
        B8 -->|not ready| B8
        B8 -->|✅ ready| B9[漸進放量\n10% → 20% → ... → 100%]
        B9 --> B10[🔔 SetWeight Blue:0% Green:100%]
    end

    A6 --> JOIN[兩條線完成]
    B10 --> JOIN
    JOIN --> S1END[Step 1 結束]
    S1END --> S2[進入 Step 2 ➡️]
```

| 線 | 目的 | 完成條件 |
|:---:|:---|:---|
| **A 線** | 把第一台 red 機器同步新版，讓 RD 可以鎖 hosts 測試 | Health Check 通過 + Slack 通知送出 |
| **B 線** | Green 加入 ELB、Red 移除 ELB、ASG 漸進放量至 Green:100% | SetWeight Blue:0% / Green:100% 完成 |

> Step 1 走完：**MWeb22（Blue）與 MWeb23（Green）同時在 ELB 內接流量**；MWeb21（Red）已離線，部署新版供 RD 驗收。hk-mweb-group1 ELB 只認 registered 狀態，Blue 從未被 Unregister，所以兩台都在線



![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1-implement/2-three-tools.png?raw=true)


![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1-implement/3-characters.png?raw=true)


<!-- endtab -->


<!-- tab 劇本、工具、程式碼、設定、憑證、執行環境-->

部署前需要備齊四件事：**知道跑什麼（劇本）、知道用什麼工具、知道部署哪個版本、知道憑證在哪**。這些全部在進入 Step 1 之前就定義好

## 工具分工架構

```mermaid
graph TD
    JF["📋 Jenkinsfile\n劇本：決定做什麼、什麼順序、哪些 parallel"]
    SL["📦 Shared Library\n公司封裝的共用工具"]
    CK["🎂 Cake Script\n實際執行每一個動作"]
    INF["🖥️ Infrastructure\nEC2 / IIS / ALB / S3 / Slack"]

    JF -->|呼叫| SL
    JF -->|呼叫| CK
    SL -->|提供共用方法給| CK
    CK -->|操作| INF

    style JF fill:#F5A623,color:#fff
    style SL fill:#7B68EE,color:#fff
    style CK fill:#5BA85B,color:#fff
    style INF fill:#555,color:#fff
```

---

## 📋 Jenkinsfile — 劇本

Jenkinsfile 是流程的指揮官，決定「做什麼、什麼時候做、誰和誰可以一起跑」：

| 決策項目 | 說明 |
|:---|:---|
| Check Build Status | 要不要在部署前先確認上一個 build 是否成功 |
| Copy Artifact | 要不要從 Jenkins artifact 下載設定檔 |
| Step 1 結構 | 哪些 stage 要跑、順序為何 |
| Parallel 範圍 | A 線 / B 線 哪些工作可以同時跑 |
| deployWithASG.cake | 是否呼叫 Cake script 執行 ASG 相關操作 |
| Timeout | 每個 stage 最多等多久，避免卡死 |
| Credentials 包裝 | 把機密資料安全地傳給後續步驟 |

## 📦 Shared Library — 共用工具

因為多站台（HK / SG / TW...）部署邏輯高度重疊，公司封裝了 Jenkins Shared Library，避免每個 Jenkinsfile 重複實作相同邏輯：

- **Slack 通知** — 統一格式發佈署進度訊息
- **共用 deployment function** — 跨站台可直接呼叫的部署方法
- **共用 credentials 包裝** — 統一管理 token / key 的注入方式
- **共用錯誤處理** — stage 失敗時的統一通知與 abort 邏輯
- **共用 log 檢查** — 解析 IIS / Health Check 回應的共用邏輯

## 🔖 佈署哪一個版本

Checkout 到正確的 commit，確認這次部署用的是哪一版程式碼。

正式部署必須能回答以下三個問題，否則出問題時無從追查：

> 1. 我部署的是哪個 **branch**？
> 2. 是哪個 **commit**？
> 3. 這個 commit 是從哪個 **release** merge 進來的？

## ⚙️ 設定檔

正式環境的設定（DB 連線、API endpoint、Feature Flag...）不會 commit 進 repo，而是透過 **Copy Artifact** 從 `nineyi.configuration` 這個獨立 Jenkins job 取得，解壓後放入 workspace，Cake script 才能讀到正式環境資料。

## 🔑 Credentials

部署過程需要多組 credentials，全部在 Jenkinsfile 中統一包裝注入：

| Credential | 用途 |
|:---|:---|
| Slack Token | 發送部署通知 |
| Git Token | Checkout repo / 建立 Git Tag |
| Artifact Token | 從 Jenkins 下載 build artifact |
| Internal API Key | 呼叫 infra API（StartAllEC2、Health Check 等） |




![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1-implement/4-jenkinsfile-and-cake.png?raw=true)




<!-- endtab -->

<!-- tab Copy Artifact-->

正式環境的設定檔不放在應用程式 repo 裡，而是由獨立的 `nineyi.configuration` job 管理。Copy Artifact stage 的任務就是：**在部署前把正確版本的設定檔抓下來，放進 workspace，讓後續 Cake script 能讀到正式環境資料**

## 執行流程

```mermaid
flowchart TD
    A[進入 Copy Artifact stage] --> B[跑在 SG-HK-JKCD-S1]
    B --> C[Checkout\nnineyi.webstore.mobilewebmall master]
    C --> D[確認 commit\na772663e...]
    D --> E[執行 copyArtifacts]
    E --> F[從 nineyi.configuration\nmaster build 3893\n複製 artifact]
    F --> G[得到 ApplicationConfig.zip]
    G --> H[放入 workspace\nD:/ws/workspace/.webstore.mobilewebmall_master_4/]
    H --> I[解壓縮\nunzip ApplicationConfig.zip]
    I --> J[✅ 解出 728 個設定檔]
    J --> K[Copy Artifact 完成\n進入 Step 1]

    style J fill:#5BA85B,color:#fff
    style K fill:#4A90D9,color:#fff
```

## 本次執行資訊

| 項目 | 值 |
|:---|:---|
| **執行節點** | SG-HK-JKCD-S1 |
| **來源 Job** | nineyi.configuration » master |
| **Build 編號** | #3893 |
| **下載檔案** | ApplicationConfig.zip |
| **解出檔案數** | 728 個設定檔 |
| **放置路徑** | `D:\ws\workspace\.webstore.mobilewebmall_master_4\` |

## 為什麼設定檔要分開管理

| 原因 | 說明 |
|:---|:---|
| **安全性** | DB 密碼、API Key 不能 commit 進應用程式 repo |
| **環境隔離** | 同一份 code 可部署到 dev / staging / prod，各自讀不同設定 |
| **版本可追蹤** | 設定有獨立的 build 編號，出問題時可回查是哪版設定造成的 |
| **職責分離** | 設定變更由維運控管，不需要動應用程式 code |


![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1-implement/5-code-and-config.png?raw=true)



<!-- endtab -->


<!-- tab Cake-->

Jenkinsfile 負責**編排**，Cake 負責**執行**。Jenkinsfile 告訴 Jenkins 什麼時候跑什麼，Cake 則是真正動手做事的腳本層，負責讀設定、操作機器、查 AWS、發通知。

## Cake 在 Step 1 做的事

```mermaid
graph LR
    CK["🎂 Cake Script\ndeployWithASG.cake"]

    CK --> A["📖 讀設定\nred / blue / green 機器清單\nASG 名稱 / ALB 設定"]
    CK --> B["☁️ 查 AWS\nEC2 狀態 / ASG 權重\nTarget Group 健康狀態"]
    CK --> C["🖥️ 操作機器\n同步 code 到 red\nIISReset / CDN 關閉"]
    CK --> D["🔍 Health Check\nHTTP 打 health endpoint\n確認服務回 200"]
    CK --> E["🔔 發 Slack\n部署進度通知"]
    CK --> F["🏷️ Git 操作\n建立 release tag\n確認 commit"]

    style CK fill:#5BA85B,color:#fff
```

## Cake 引用的工具與用途

| 套件 | 用途 |
|:---|:---|
| `AWSSDK.EC2` | 查 EC2 / ASG 狀態、觸發 StartAllEC2、確認 InService |
| `AWSSDK.S3` | 取得部署檔、設定檔或備份 |
| `Cake.Http` | 呼叫 infra API、admin API、Health Check endpoint |
| `Cake.Powershell` | 執行 PowerShell 指令（Test-Connection、IISReset 等） |
| `Cake.Slack` | 發送部署進度通知到指定 Slack channel |
| `Cake.Git` | 建立 Git Tag、讀取 commit / branch 資訊 |
| `StackExchange.Redis` | 處理 cache 清除或設定快取 |
| `Polly` | retry 機制、timeout 控制、失敗重試策略 |
| `NineYi.Cake.Lib` | 公司內部封裝的部署方法（共用邏輯） |

## Jenkinsfile vs Cake — 各自的邊界

| | Jenkinsfile | Cake |
|:---:|:---:|:---:|
| **角色** | 指揮官 | 執行者 |
| **決定** | 跑什麼、何時跑、parallel 範圍 | 怎麼跑、呼叫哪個 API、怎麼 retry |
| **可見度** | Jenkins UI pipeline 視圖 | Cake log 輸出 |
| **修改影響** | 流程結構改變 | 執行細節改變 |


![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1-implement/6-cake-execute-core.png?raw=true)


<!-- endtab -->


<!-- tab 這段流程慢在哪-->

Step 1 最耗時的不是部署新版 code，而是 **B 線等 Green ASG InService、再執行漸進放量的整段流程**。

## 時間對比

| 任務 | 耗時 | 說明 |
|:---|:---:|:---|
| A 線（Sync first machine） | `00:00:43` | 下載 + 同步 + IISReset + Health Check |
| B 線（Prepare Green ASG） | `≈ 16 分鐘` | Register → Drain → poll InService → 漸進放量 |

> B 線比 A 線慢 **約 22 倍**。整個 Step 1 的瓶頸就是等 Green group 就緒，再一步步把流量切過去。

## B 線的完整時間軸（以 HK log 為例）

```
09:23:07  B 線啟動，確認 Green=0% / Blue=100%
09:23:18  RegisterEC2toELB（MWeb23 加入 ELB）
09:23:xx  Health Check MWeb23 endpoints 通過
09:24:55  UnregisterEC2fromELB（MWeb21 移出 ELB）
          Wait 180s connection draining
09:27:xx  開始 poll Green ASG InService（約每 20 秒一次）
09:34:25  Green group is all ready（InService ✅）
09:34:26  SetWeight Blue:90% / Green:10%
09:35:07  SetWeight Blue:80% / Green:20%
09:35:46  SetWeight Blue:70% / Green:30%
...       每步約 40 秒
09:39:xx  SetWeight Blue:0%  / Green:100% ← B 線完成
```


![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1-implement/7-two-lines-time.png?raw=true)


![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1-implement/8-tool-up-steps.png?raw=true)


## 為什麼先確認 Green = 0%

B 線第一步是查 Green 的 ALB 權重：

```
Group  = Green
Weight = 0%
```

確認 Green 當下沒有正式流量，才能安全地把機器關掉再重開。  
若 Green 還有流量就強行開機輪替，使用者請求可能打到尚未就緒的機器。

## StartAllEC2 → InService 的完整鏈路

呼叫 API 開機只是起點，Windows EC2 要跑完整條鏈路才算真正 InService：

```mermaid
flowchart TD
    S["☁️ StartAllEC2\nInstanceComponentName=MWeb-HK\nInstanceType=c6i.xlarge&region=hk"]
    S --> A[EC2 電源啟動]
    A --> B[Windows OS 開機]
    B --> C[網路介面 / DNS 就緒]
    C --> D{Test-Connection\nSG-HK-MWeb23}
    D -->|❌ False\n一開始 ping 不通| WAIT[⏳ 持續輪詢等待]
    WAIT --> D
    D -->|✅ True| E[IIS / Web Service 啟動]
    E --> F[應用程式 Warm Up]
    F --> G[Health Check URL 回 200]
    G --> H[Target Group 判定 Healthy]
    H --> I[ASG Lifecycle Hook 完成]
    I --> IS[✅ InService]

    style S fill:#F5A623,color:#fff
    style IS fill:#5BA85B,color:#fff
    style WAIT fill:#D9534F,color:#fff
```

## Windows EC2 本來就慢

Linux container 啟動可能只要幾秒，Windows + IIS 機器每個階段都更耗時：

| 階段 | 慢的原因 |
|:---|:---|
| Windows 開機 | OS 本身初始化流程長 |
| IIS 啟動 | Application Pool 初始化、module 載入 |
| 應用程式 Warm Up | .NET 第一次 JIT 編譯、connection pool 建立 |
| Health Check 通過 | 要等應用程式真的能回應，不只是 port 開著 |
| ASG 狀態更新 | AWS 需要連續多次 healthy 才更新狀態 |

## 診斷指標

若某次 B 線特別慢，可以依序確認這六個節點：

| # | 確認項目 | 對應現象 |
|:---:|:---|:---|
| 1 | EC2 從 StartAllEC2 到 `running` 花多久 | AWS Console 看 instance state |
| 2 | Windows 開機到可以 ping 花多久 | log 裡 Test-Connection 第幾次變 True |
| 3 | IIS / Web Service 啟動花多久 | Event Log / IIS log 時間戳 |
| 4 | Health Check URL 何時開始回 200 | Cake log 裡 health check 回應時間 |
| 5 | Target Group 何時變 healthy | AWS Target Group 健康狀態時間軸 |
| 6 | ASG lifecycle hook 是否有等待腳本 | ASG activity log |



![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1-implement/9-different-problems.png?raw=true)


<!-- endtab -->


{% endtabs %}
