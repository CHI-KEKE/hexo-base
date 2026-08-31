---
title: Internal vs Public
date: 2025-11-18 23:51:11
categories: 蒼き盾
top_img: https://github.com/CHI-KEKE/pics/blob/main/Security/wolf.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Security/wolf.png?raw=true
tags:
    - 蒼き盾
toc:
toc_number:
comments :
---


{% tabs Internal vs Public%}

<!-- tab GitLab Self-Managed-->

GitLab Self-Managed 是公司自己安裝、自己維運的一套 GitLab，可以部署在公司內部網路、私有雲，甚至做成完全離線的環境。
和 GitLab.com（SaaS 版）最大的差別在於：這套 instance 完全由公司掌控，主機、網路、權限、備份、存取規則全都是自己管。

## 核心元件

| 元件 | 負責的事 |
|------|---------|
| Web / API 服務 | 你平常打開網頁、呼叫 API 的入口 |
| PostgreSQL | 儲存帳號、專案、issue、pipeline 等資料 |
| Redis | 快取與背景工作協調 |
| Gitaly / Git 儲存 | 真正存放 repository 的地方 |
| Sidekiq | 背景工作，例如寄通知、執行非同步任務 |
| Registry / Runner | 延伸元件，視公司是否啟用 |

把整套 GitLab 比做一間公司：

- **前台**（使用者介面）→ Web / API 服務
- **倉庫**（放程式碼）→ Gitaly / repo storage
- **帳務系統**（資料存放）→ PostgreSQL
- **跑腿工人**（非同步任務）→ Sidekiq
- **CI/CD 執行者**（跑 pipeline）→ Runner



![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/public-internal-internet/1-gitlab-like-company.png)



<!-- endtab -->



<!-- tab 公司網路怎麼建起來的-->


## 第 1 步：公司先申請一條網路線進辦公室

公司去找電信商（例如中華電信或其他 ISP）申請網路連線，常見形式有

- 辦公室寬頻
- 光纖
- 專線
- 固網服務

這一步完成後，Internet 就被拉進公司辦公室了。可以把它想成：**外面的公共道路已經通到公司大門口，但公司裡面還沒有走道、沒有房間編號**。所以這一步只是先有了「對外連線能力」。

## 第 2 步：公司買一台路由器

公司會在入口放一台設備，通稱為路由器、防火牆、Gateway 或邊界設備，角色像公司大門口的總管理員，負責兩件事

- **對外**：接到電信商那條線，讓公司可以上 Internet
- **對內**：開始建立公司自己的網路空間，讓內部設備先接到這台設備，再共用同一個出口出去

## 第 3 步：公司在裡面自己定一套地址

公司接上 Internet 之後，在內部自訂一套私有 IP 規則，例如：

- `192.168.1.x`
- `10.0.0.x`

路由器再依此分配：

- 筆電 → `192.168.1.10`
- 印表機 → `192.168.1.20`
- NAS → `192.168.1.30`

這些地址**只在公司內部有意義**，外面的 Internet 不認識它們，但公司內部設備彼此都認得。就像公司在空辦公室自己貼門牌，是公司自己編的



![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/public-internal-internet/2-company-network-1.png)



## 第 4 步：把所有設備都接進這個圈圈

員工筆電、桌機、印表機、NAS、伺服器全都接進內網後，公司就有了兩種能力：

**能力 1：內部彼此溝通**
- 電腦可以找到印表機、檔案伺服器、GitLab、內部資料庫

**能力 2：共用同一個出口上網**
- 所有設備對外看起來都從同一個出口出去
- 公司可以控制誰能上哪些網站、記錄流量、擋特定連線

到這一步，公司內網就已經成形了



![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/public-internal-internet/3-company-seperate.png)




## 第 5 步：公司內網和外網之間加一道門

光有內部互連還不夠，還要決定：

- 哪些內部設備能出去
- 外面能不能進來
- 哪些連線要擋、哪些服務要保護

所以公司通常會部署更正式的邊界設備

- 防火牆
- 邊界路由器
- ACL 規則
- NAT
- VPN gateway

這些東西的作用就是：**幫公司把「自己裡面」和「外面全世界」分隔清楚**。這就是為什麼大家會說「這是內網、那是外網」，而且公司自己在控制一個邊界

## 第 6 步：公司開始把服務放進內網

一開始內網裡可能只有員工電腦和印表機，後來公司會越放越多東西進來

- GitLab、Jenkins、SonarQube
- 資料庫、K8s
- HR 系統、監控系統、內部 API

這些服務只要住在公司自己那套私有地址系統裡，就會被稱為「內網系統」





![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/public-internal-internet/4-company-NAT.png)



<!-- endtab -->


<!-- tab DNS-->

DNS 不是某一家公司的專屬產品，它是一種把名字翻成 IP 的系統，例如：

- `gitlab.company.com` → `10.2.1.15`
- `www.google.com` → 某個公網 IP

「這個 DNS 是公司自己擁有的嗎？」這個問題要拆成兩個角度來看：

- 這個**名字**是不是公司自己的
- 這個 **DNS 伺服器**是不是公司自己管理的

這兩件事可以一樣，也可以不一樣。

## 情況 1：名字是公司的（網域所有權）

公司去註冊了 `company.com`，就可以在底下建立各種名稱：

- `gitlab.company.com`
- `vpn.company.com`
- `api.company.com`

這裡的「擁有」指的是：

- 公司有這個網域的註冊使用權
- 公司能決定底下開哪些名稱
- 公司能指定這些名稱對應到哪個 IP

不是說公司發明了 `.com`，而是公司租用 / 註冊了 `company.com` 這個名字空間。

## 情況 2：DNS 伺服器也是公司自己架的

公司除了擁有網域名稱，還可能自己架 DNS server，例如：

- 負責解析 `gitlab.company.com`
- 負責解析 `jenkins.company.com`
- 負責解析 `db.internal.company.com`

這種情況下，名字是公司的，DNS 伺服器也是公司自己管的。這在內網很常見，因為公司希望內部服務名稱只讓內部人查得到。

## 情況 3：名字是公司的，但 DNS 外包

公司擁有 `company.com`，但 DNS 記錄代管在第三方：

- Cloudflare
- Route 53
- DNSPod

這時候網域名稱是公司的，但 DNS 記錄的存放和回應是交給第三方做。就像：**房子是你的，但警衛是外包的**。

## 情況 4：內網 DNS 幾乎一定是公司自己管的

GitLab 內網情境通常會是這樣：

- GitLab 實際 IP：`10.2.1.15`
- 公司內部 DNS 記錄：`gitlab.company.local` → `10.2.1.15`

這筆 DNS 記錄只存在公司內部 DNS 裡：

- 公司內網的人查得到
- 外面的 Internet 查不到

所以這套 DNS 解析規則完全是公司自己的，因為它只在公司自己的網路範圍內成立


![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/public-internal-internet/5-network-and-DNS.png)



<!-- endtab -->


<!-- tab 內網-->

公司擁有自己的網路，說穿了就是先把設備接成一個小圈圈讓它們彼此溝通，再只留一個受控出口去接外面的 Internet

## 公司怎麼連上 Internet：NAT

NAT 可以把很多內網 IP 共用少數幾個公網 IP 對外上網。公司裡的內部設備

- 工程師電腦 `10.0.1.11`
- GitLab `10.0.1.20`
- 某台 service `10.0.2.35`

這些都只是內部地址，但當它們要連外（拉套件、打 GitHub API、查外部網站），公司出口設備會幫它們做轉換，看起來像是都從同一個公網 IP 出去

## 第一步：先有基礎網路設備

公司會部署這些設備來建立自己的網路範圍：

- Router、Switch
- Firewall
- Wi-Fi AP
- VPN 設備
- 雲端 VPC / 子網路

## 第二步：分配自己的內部 IP

每台設備都會拿到一個私有 IP：

- `10.x.x.x`
- `172.16.x.x` ~ `172.31.x.x`
- `192.168.x.x`

例如 GitLab 可能是 `10.0.1.20`、DB 是 `10.0.2.15`。這些地址在公司內部能互通，但不會直接暴露到 Internet

## 第三步：讓公司內部設備彼此能互通

公司會設定存取規則，例如

- 開發者電腦可以連 GitLab
- GitLab Runner 可以連 K8s
- K8s 服務可以連資料庫
- 訪客 Wi-Fi 不能碰這些東西

## 第四步：在內網上放服務

當公司把系統放進這個網路裡，這些就變成內部平台

- GitLab、Jenkins、SonarQube、Nexus
- Kubernetes、資料庫
- 內部後台、HR 系統、報表系統

只要這些服務主要住在公司管理的網路空間裡，通常只有公司內部或 VPN 才能連到，大家就會說這是公司內網系統


<!-- endtab -->

<!-- tab 每台機器實際拿哪個 IP-->


## A. 自動分配（DHCP）

員工筆電、手機、一般設備通常由 DHCP 自動發 IP，例如：

- 你連上公司 Wi-Fi
- DHCP server 發給你 `10.1.5.23`

管理方便，不用手動一台一台填。

## B. 固定分配（Static IP）

伺服器、GitLab、資料庫、K8s 節點這種，通常會用固定 IP 或保留固定配置，例如：

- GitLab 永遠是 `10.20.1.10`
- DB 永遠是 `10.20.2.10`

因為別的系統要穩定找到它們，IP 不能一直變。



<!-- endtab -->


<!-- tab 一步步-->


![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/public-internal-internet/6-REQUEST-FLOW-1.png)



## 第 1 步：外部使用者先輸入你的網址

假設外部使用者在瀏覽器輸入：`https://api.company.com/pr-webhook`

使用者手上只有這個名字，電腦還不知道真正要去哪個 IP，所以第一件事一定是先做 DNS 查詢。「api.company.com 這個名字到底代表哪個入口？」Route 53 常見做法是把網域用 alias record 指到 ELB/ALB

## 第 2 步：DNS 把這個名字翻成 AWS 入口

標準 AWS 做法中，`api.company.com` 是指向一個 ALB 的 DNS 名稱，例如 `*.elb.<region>.amazonaws.com`。每個 ALB 都有預設的 DNS 名稱，Route 53 可以用 alias 指到它。也就是說，你平常看到的公司網域，後面其實多半接的是 AWS Load Balancer，不是你的 app 本身

## 第 3 步：使用者實際連到的是 ALB，而不是你的程式

DNS 回答完之後，使用者的瀏覽器就去連那個 ALB。若 ALB 是 internet-facing，AWS 會讓它對外可達並透過公網 IP 接流量；若是 internal，它就只有私網可達，外部通常進不來。也就是說，外部使用者能連到你的服務，前提不是你的 Pod 公開，而是前面這個 LB 被做成可公開進入





![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/public-internal-internet/7-request-flow-step1-to-alb.png)



## 第 4 步：ALB 先處理最外層的連線

當請求到了 ALB，ALB 扮演第一層門口，常做的事包括接 HTTPS、看 Hostname、看 URL path、決定要往哪個後端 Target Group 送。Kubernetes 這邊的 Ingress 就是在告訴 ALB：「某個 host/path 應該送去哪個服務」

## 第 5 步：Ingress 在這裡像一本分流規則

Ingress 不是那台 ALB 本身，而是 K8s 裡的一個規則物件，裡面寫的邏輯像這樣

```BASH
api.company.com/pr-webhook  →  webhook-service
gitlab.company.com          →  gitlab-service
```

AWS Load Balancer Controller 會根據你建立的 Ingress，自動建立並配置 ALB

## 第 6 步：ALB 會被配置去找對的 Kubernetes Service

ALB 根據 Ingress 規則知道這筆請求要去 `webhook-service` 後，下一層不是直接找 Pod，而是先進到 Kubernetes 的 Service。Service 是一個抽象層，代表一組提供同樣功能的 Pods。換句話說，**Service 像櫃台，Pod 像坐在櫃台後面的人**。

## 第 7 步：Service 再把流量分到其中一個 Pod

流量進到 `webhook-service` 後，Kubernetes 會把它分到符合 selector 的某一個 Pod，並自動 load-balance。這就是為什麼你升版、重啟、擴 Pod 數量時，外部入口不需要跟著一直改——外面看的是 Service 或 ALB，後面 Pod 換了由 Kubernetes 自己接住

## 第 8 步：Pod 裡的程式才真的收到 HTTP request

到這一刻，你的應用程式才真正看到請求內容

- `method: POST`
- `path: /pr-webhook`
- `headers`
- `body`
- 來源 IP 或轉發標頭

對外使用者以為自己在打 `api.company.com`，但程式實際收到的是「經過 DNS、ALB、Ingress 規則、Service 轉送之後」的一筆 request。Service/Ingress 機制就是為了讓你不用把 Pod 直接暴露在外面


![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/public-internal-internet/7-2-request-flow-step2-to-pod.png)


## 第 9 步：你的程式處理完，再把回應一路送回去

回應路徑會反過來走

```BASH
Pod → Service → ALB → 使用者瀏覽器
```

對使用者來說，他只覺得 `api.company.com` 回了東西，不會知道後面哪個 Pod 處理了這次請求。這就是入口抽象的價值 **外面永遠只碰到穩定入口，裡面怎麼換 Pod、怎麼擴縮，盡量不影響外部**

```BASH
外部使用者
   |
   | 1. 輸入 https://api.company.com/pr-webhook
   v
DNS / Route 53
   |
   | 2. 回答：這個名字指到某個 ALB
   v
AWS ALB (internet-facing)
   |
   | 3. 接 HTTPS / 看 host / 看 path
   | 4. 依 Ingress 規則決定轉給 webhook-service
   v
Kubernetes Service (webhook-service)
   |
   | 5. 從多個 Pod 中選一個
   v
Pod
   |
   | 6. 你的程式真的收到 request
   v
回應原路返回
```



![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/public-internal-internet/7-3-request-flow-back.png)



<!-- endtab -->


<!-- tab summary-->


![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/public-internal-internet/8-final.png)


![cv](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Security/public-internal-internet/Network_Architecture_Blueprint.png)



<!-- endtab -->


{% endtabs %}