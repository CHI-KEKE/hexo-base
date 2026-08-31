---
title: Blue-Green Deployment — 設計概念
date: 2026-05-12 08:38:00
categories: CI / CD
top_img: https://github.com/CHI-KEKE/pics/blob/main/CICD/feather.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/CICD/feather.png?raw=true
tags:
    - 
toc:
toc_number:
comments :
---


{% tabs Blue-Green-Deployment-Concept%}


<!-- tab 意義-->

準備兩套幾乎一樣的正式環境，一套持續服務使用者，另一套靜默部署新版，確認無誤後再把流量切過去


> 不在使用者正在踩的地板上施工，而是先把隔壁房間裝潢好、確認能住，再把人帶過去


![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1/1-do-not-use-construction.png?raw=true)



## 部署流程

```mermaid
flowchart TD
    A[👤 使用者流量] --> B[🔵 Blue 環境\n舊版持續服務]
    C[🟢 Green 環境] --> D[靜默部署新版]
    D --> E{健康檢查}
    E -->|✅ 通過| F[切流量至 Green]
    E -->|❌ 失敗| G[保持 Blue 上線\n放棄本次部署]
    F --> H[Blue 降為備援\n隨時可切回]
```


## 藍綠 vs 金絲雀

兩者常被混淆，但著重點不同：

| | 藍綠佈署 | 金絲雀佈署 |
|:---:|:---:|:---:|
| **重點** | 架構（兩套環境） | 放量策略 |
| **流量方式** | 一次全切或漸進 | 先少量真實流量驗證，再逐步擴大 |
| **適合情境** | 需要快速回滾保底 | 需要低風險漸進驗收 |



![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1/2-blue-green-canary-diff.png?raw=true)




藍綠佈署搭配 `weighted target group`，同樣可以做到漸進放量，兼顧兩種策略的優點


![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1/3-why-alb-weight.png?raw=true)


## ALB Weighted Target Group

AWS ALB 的流量分配機制：**同一個入口依照比例，把請求分配到不同 Target Group**。


```mermaid
graph LR
    User([👤 使用者]) --> ALB[⚖️ ALB\nApplication Load Balancer]
    ALB -->|100%| Blue[🔵 Blue Target Group\nMWeb22]
    ALB -->|0%| Green[🟢 Green Target Group\nMWeb23]

    style Blue fill:#4A90D9,color:#fff
    style Green fill:#7DC67E,color:#fff
```

切換時調整權重比例，不改 DNS、不動機器，流量即時生效：

```
切換前：Blue 100% / Green 0%
漸進式：Blue 50%  / Green 50%   ← 觀察穩定後
切換後：Blue 0%   / Green 100%
```


## 我們的實作與標準藍綠的差異

我們的版本是 **三色機器輪替 + First Machine 驗證 + ALB/ASG 控制** 的藍綠變體：

```mermaid
flowchart LR
    subgraph standard["📖 標準藍綠"]
        direction TB
        s1[部署新版\n至整套 Green] --> s2[健康檢查]
        s2 --> s3[全量切流量\nBlue → Green]
    end

    subgraph ours["🏢 我們的做法"]
        direction TB
        o1[確認 Green=0%\nBlue=100%] --> o2[部署新版\n至 First Machine 單台]
        o2 --> o3[RD 鎖 hosts\n單台驗證]
        o3 --> o4[逐步輪替\n其餘機器]
        o4 --> o5[調整 ALB 權重\n切流量]
    end
```

| | 標準藍綠 | 我們的做法 |
|:---:|:---:|:---:|
| **部署目標** | 整套 Green 環境 | First Machine 單台先行 |
| **驗證方式** | 環境層級健康檢查 | RD 鎖 hosts 對單台測試 |
| **風險控制粒度** | 新舊環境切換 | 每步固定角色、單台驗證、逐步推進 |
| **人為判斷負擔** | 需判斷這次新版在哪個顏色 | 角色固定，不需判斷 |

RD 每次不再需要思考：*新版在哪個顏色？這台有流量嗎？我要鎖哪個 IP？*
角色固定、流程固定 — 這是以**長期維運**角度設計的佈署策略。



![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1/4-fix-machine-color.png?raw=true)


<!-- endtab -->


<!-- tab 角色-->

Step 1 中三台機器各司其職：**Red 離開線上接受新版測試、Blue 繼續扛流量、Green 悄悄開機等待就緒**。風險被控制在單台，線上服務不受影響。

## Step 1 機器狀態全覽

```mermaid
graph TB
    User([👤 使用者]) --> ALB[⚖️ ALB]

    ALB -->|🔵 0%| Blue[SG-HK-MWeb22\nBlue]
    ALB -->|🟢 100%| Green[SG-HK-MWeb23\nGreen]

    Red[SG-HK-MWeb21\nRed] <-.->|🔍 鎖 hosts 測試| RD[👨‍💻 RD]

    subgraph online["🌐 線上服務中"]
        Blue
        Green
    end

    subgraph staging["🔧 新版驗證中（無線上流量）"]
        Red
    end

    style Blue fill:#4A90D9,color:#fff
    style Green fill:#7DC67E,color:#fff
    style Red fill:#D9534F,color:#fff
```


## 🔴 SG-HK-MWeb21 — Red（新版測試機）

- **B 線操作**：Green 就緒後，B 線呼叫 `UnregisterEC2fromELB` 將 Red 從 ELB 移除，等待 180s connection draining
- **A 線操作**：下載新版 code → 同步到 Red → 關閉 CDN → IISReset → Health Check → 通知 RD
- **流量狀態**：Step 1 期間移出 ELB，無線上流量
- **為何先選這台**：角色固定為「每輪第一台」，RD 每次都知道鎖這台，不需要判斷

> Red 下線是由 **B 線** 執行（RegisterEC2 Green → UnregisterEC2 Red），不是 A 線。A 線只負責部署新版。


![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1/5-red.png?raw=true)



## 🔵 SG-HK-MWeb22 — Blue（線上主力）

- **Step 1 任務**：無操作，全程留在 ELB
- **流量狀態**：✅ **Step 1 全程在線接流量**，B 線結束時 ASG weight 降至 0%，但 ELB 仍 registered
- **角色意義**：Step 1 期間持續兜底；Step 1 結束後與 Green 同時在 ELB 內接流量


![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1/6-blue.png?raw=true)


## 🟢 SG-HK-MWeb23 — Green（新主力上線）

- **B 線操作**：StartAllEC2 → Ping 通 → `RegisterEC2toELB`（加入 ELB）→ Health Check 通過 → 等 ASG InService → **漸進放量 10% → 20% → … → 100%**
- **流量狀態**：Step 1 結束時 ASG weight = **100%**，已成為主要導流機器
- **角色意義**：B 線核心任務，Green 就緒是 Step 1 完成的關鍵條件


![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1/7-green.png?raw=true)



## Step 1 結束後三機狀態

| 機器 | 顏色 | ELB 狀態 | Step 1 關鍵操作 |
|:---:|:---:|:---:|:---|
| SG-HK-MWeb21 | 🔴 Red | ❌ 已移除 | B線 Unregister → A線部署新版 → RD 測試 |
| SG-HK-MWeb22 | 🔵 Blue | ✅ 仍在 | 無操作，全程在線接流量 |
| SG-HK-MWeb23 | 🟢 Green | ✅ 已加入 | B線 Register → Health Check → 漸進放量 |

<!-- endtab -->


<!-- tab 切流量-->

切流量不是改 DNS、也不是把機器手動上下線，而是**調整 ALB Target Group 的 Weight 權重**，同一個入口、即時生效、可以漸進也可以一次全切


![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1/target-groups-web.png?raw=true)



## DNS 切換 vs ALB 權重切換

| | DNS 切換 | ALB 權重切換 |
|:---:|:---:|:---:|
| **生效速度** | 受 TTL 影響，可能延遲數分鐘 | 即時 |
| **漸進控制** | 不支援 | ✅ 支援任意比例 |
| **回滾速度** | 慢 | ✅ 改數字即回滾 |
| **機器操作** | 無需動機器 | 無需動機器 |

## 三種流量狀態

```mermaid
graph LR
    subgraph s1["① 切換前（Step 1 確認狀態）"]
        direction TB
        A1[⚖️ ALB] -->|100%| B1[🔵 Blue]
        A1 -->|0%| G1[🟢 Green]
    end

    subgraph s2["② 漸進切換（可選）"]
        direction TB
        A2[⚖️ ALB] -->|50%| B2[🔵 Blue]
        A2 -->|50%| G2[🟢 Green]
    end

    subgraph s3["③ 切換完成"]
        direction TB
        A3[⚖️ ALB] -->|0%| B3[🔵 Blue]
        A3 -->|100%| G3[🟢 Green]
    end

    s1 -->|Step 2+| s2
    s2 -->|確認穩定| s3

    style B1 fill:#4A90D9,color:#fff
    style B2 fill:#4A90D9,color:#fff
    style B3 fill:#4A90D9,color:#fff
    style G1 fill:#7DC67E,color:#fff
    style G2 fill:#7DC67E,color:#fff
    style G3 fill:#7DC67E,color:#fff
```

## Step 1 的流量切換實際做了什麼

Step 1 **不只是查**，B 線會完整執行流量切換的全部動作：

```mermaid
flowchart TD
    Q1[確認 Green weight = 0%\nBlue weight = 100%] --> R[RegisterEC2toELB\nMWeb23 加入 ELB]
    R --> HC[Health Check MWeb23\n多個 endpoint 確認通過]
    HC --> U[UnregisterEC2fromELB\nMWeb21 移出 ELB]
    U --> D[Wait 180s\nConnection Draining]
    D --> P{持續 poll\nGreen ASG InService}
    P -->|not ready| P
    P -->|✅ ready| GW[漸進放量\nBlue:90% Green:10%]
    GW --> GW2[Blue:80% Green:20%]
    GW2 --> GW3[... 每步約 40s ...]
    GW3 --> DONE[Blue:0% Green:100%]

    style Q1 fill:#4A90D9,color:#fff
    style DONE fill:#5BA85B,color:#fff
```

## 漸進放量細節

Green InService 後，B 線不是直接一次全切，而是每隔約 40 秒調一步：

```
Blue:90% / Green:10%  → 09:34:26
Blue:80% / Green:20%  → 09:35:07
Blue:70% / Green:30%  → 09:35:46
...（每步約 40 秒）
Blue:0%  / Green:100% → 09:39:00
```

> 這正是藍綠佈署搭配漸進放量的實際體現 — 讓真實流量先小比例打到 Green，確認穩定後再全切，降低一次全切帶來的風險。

<!-- endtab -->


<!-- tab 平行跑兩條線-->

Step 1 的兩條線**完全獨立、互不等待**，Jenkins 同時啟動兩者，等兩條都完成才進入 Step 2。這樣設計的核心原因是：**A 線部署 code 約 43 秒，B 線等 Green InService 約 16 分鐘 — 若循序執行就要等 17 分鐘，平行只需等最慢的那條。**


![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1/8-tracks.png?raw=true)



## 時序圖

```mermaid
sequenceDiagram
    participant JK as ⚙️ Jenkins
    participant A as 🔴 A 線 Red
    participant B as 🟢 B 線 Green ASG
    participant RD as 👨‍💻 RD
    participant AWS as ☁️ AWS

    JK->>+A: 啟動 Prepare First Machine
    JK->>+B: 啟動 Prepare Green ASG

    Note over A,B: ⚡ 同時執行，互不等待

    A->>A: 發 Slack 通知
    A->>A: 下載 WebAPI / MobileWebMall artifact
    A->>A: 同步到 SG-HK-MWeb21 red
    A->>A: 關閉 red CSS / JS CDN
    A->>A: IISReset + Health Check
    A-->>RD: 🔔 通知鎖 hosts 測試
    A->>-JK: ✅ A 線完成（約 43 秒）

    B->>AWS: 確認 Green weight = 0%
    B->>AWS: 確認 Blue weight = 100%
    B->>AWS: StartAllEC2
    B->>B: Ping red / blue / green
    B->>AWS: RegisterEC2toELB（MWeb23 加入 ELB）
    B->>B: Health Check MWeb23 多個 endpoint
    B-->>RD: Slack：MWeb21 即將下線
    B->>AWS: UnregisterEC2fromELB（MWeb21 移出 ELB）
    B->>B: Wait 180s connection draining
    B->>AWS: 持續 poll Green ASG InService
    Note over B,AWS: ~9 分鐘後 Green InService ✅
    B->>AWS: SetWeight Blue:90% / Green:10%
    B->>AWS: SetWeight Blue:80% / Green:20%
    Note over B,AWS: 每步約 40 秒，共 10 步
    B->>AWS: SetWeight Blue:0% / Green:100%
    B->>-JK: ✅ B 線完成（約 16 分鐘）

    JK->>JK: 兩條線都完成 → 進入 Step 2
```

## 平行的設計意義

| | 循序執行 | 平行執行 |
|:---:|:---:|:---:|
| **A 線時間** | 約 43 秒 | 約 43 秒（同時進行） |
| **B 線時間** | 約 16 分鐘 | 約 16 分鐘（同時進行） |
| **總等待時間** | ≈ 17 分鐘 | ≈ 16 分鐘 |
| **兩線有依賴？** | — | ❌ 完全獨立 |

## Step 1 結束後三機狀態

Step 1 走完，**MWeb22（Blue）與 MWeb23（Green）同時在 ELB 內接流量**；MWeb21（Red）已離線，部署新版供 RD 驗收：

| 機器 | 顏色 | ELB 狀態 | 流量 | Step 1 關鍵操作 |
|:---:|:---:|:---:|:---:|:---|
| SG-HK-MWeb21 | 🔴 Red | ❌ 已移除 | 無 | B線 Unregister → A線部署新版 → RD 測試 |
| SG-HK-MWeb22 | 🔵 Blue | ✅ 仍在 | ✅ **在線** | 無操作，全程接流量 |
| SG-HK-MWeb23 | 🟢 Green | ✅ 已加入 | ✅ **在線** | B線 Register → Health Check → 漸進放量 |

> `SetWeight` 是 ASG/ALB 層的路由設定，**hk-mweb-group1 ELB 只認 registered 狀態**。Blue 從未被 Unregister，所以兩台都在線上接流量。

## 兩條線的失敗行為

```mermaid
flowchart TD
    A[A 線失敗] --> F[Step 1 整體 Fail\nJenkins 中斷]
    B[B 線失敗] --> F
    F --> N[🔔 Slack 通知失敗]
    N --> M[人工介入處理]
```

> 任一條線失敗，Step 1 即中止，不會進入 Step 2。這確保「第一台沒驗過」或「Green 機器沒就緒」時，流程不會繼續推進。

<!-- endtab -->


<!-- tab machine steps 說明-->

A 線「Prepare First Machine」實際上由三個循序 step 組成，共同完成「把第一台機器部署好、讓 RD 可以驗收」這件事


## 三個 Step 的分工

| Step | 核心任務 | 完成標誌 |
|:---:|:---|:---|
| **Prepare** | 確認角色、通知開始、關 CDN | Slack 發出、red CDN 已關 |
| **Sync** | 下載新版、同步 code 到機器 | 檔案同步完成 |
| **Check** | 重啟服務、健康確認、通知 RD | Health Check 回 200、RD 收到通知 |

## 執行流程

```mermaid
flowchart TD
    subgraph P["① Prepare First Machine"]
        P1[確認 SG-HK-MWeb21 為 red 角色] --> P2[🔔 發 Slack：Step 1 開始]
        P2 --> P3[關閉 red CSS / JS CDN]
    end

    subgraph S["② Sync First Machine"]
        S1[從 Jenkins 下載\nWebAPI artifact] --> S2[下載\nMobileWebMall artifact]
        S2 --> S3[解壓縮]
        S3 --> S4[同步到 MWeb21 red]
    end

    subgraph C["③ Check First Machine"]
        C1[IISReset] --> C2[等待 IIS 重啟]
        C2 --> C3[Health Check HTTP 請求]
        C3 --> C4{回應 200?}
        C4 -->|✅ 通過| C5[🔔 通知 RD 鎖 hosts 測試]
        C4 -->|❌ 失敗| C6[retry / Slack 報錯]
    end

    P --> S --> C
```

## Jenkins 與 RD 的交接點

```mermaid
sequenceDiagram
    participant JK as ⚙️ Jenkins
    participant Red as 🔴 MWeb21 Red
    participant RD as 👨‍💻 RD

    JK->>Red: ① Prepare：確認角色、關 CDN
    JK->>Red: ② Sync：下載 artifact → 同步 code
    JK->>Red: ③ Check：IISReset → Health Check
    JK-->>RD: 🔔 Health Check 通過，可以鎖 hosts 了
    RD->>Red: 本機 hosts 指向 MWeb21
    RD->>Red: 瀏覽器測試新版功能
    RD-->>JK: 確認沒問題 → 繼續 Step 2
```

> Jenkins 跑完這三個 step，球就交到 RD 手上。RD 驗收完畢後，才會繼續推進後續部署



![f](https://github.com/CHI-KEKE/pics/blob/main/CICD/Blue-Green-Deployment-1/final.png?raw=true)


<!-- endtab -->


{% endtabs %}
