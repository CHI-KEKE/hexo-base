---
title: docker-daemon、docker engine、runc、containerd
date: 2026-04-06 08:06:05
categories: Docker
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
tags:
    - Docker
toc:
toc_number:
comments :
---


{% btn 'https://docs.docker.com/engine/?utm_source=chatgpt.com',docker engine,far fa-hand-point-right %}



{% tabs docker-daemon%}

<!-- tab daemon-->

daemon 這個詞在電腦世界裡，起源自早期 MIT 的作業系統研究圈，後來被 Unix 沿用，因為系統裡總有一堆沒人想手動一直顧的瑣事，所以工程師乾脆替這些工作取了一個像隱形幫手的名字，工程師會用 daemon 來稱呼背景工作的程式


`sshd`、`httpd` 這些名字後面的 d，就是這個文化留下來的痕跡


當我們開了一台多人共用的電腦，總不能每次有人要上網、登入、列印、記 log，才有人衝出來手動開一個程式幫忙，所以系統會先放一些常駐背景的程式在那邊

- `sshd`：有人連 SSH 進來時接手
- `httpd` / `nginx`：有人送 HTTP request 時接手
- `cron`：時間到了就跑排程
- `dockerd`：有人下 Docker 指令時接手


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/daemon/1_what_is_d.png)


<!-- endtab -->


<!-- tab webserver daemon-->

Nginx 和 Apache 都可以算是 web server daemon，它們是提供 HTTP 服務的背景常駐程式，也就是你常說的 Web Server

當開啟 https://example.com 的時候，背後大概是這樣

1. 主機上的 Nginx 已經先在背景執行，它監聽 443 port
2. 瀏覽器送出 HTTPS request
3. Nginx 收到之後，可能先處理憑證
4. 如果是靜態頁面，就直接把 HTML/CSS/JS 回給你
5. 如果是動態網站，就把請求轉給 Node.js、PHP、Java 之類的後端
6. 最後把結果回傳給瀏覽器

所以雖然我們看到的是一個網站，背後其實是 daemon 一直在顧入口


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/daemon/2_webserver_daemon.png)


<!-- endtab -->


<!-- tab dockerd-->



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/daemon/3_client_server.png)


dockerd 是 Docker 的背景總管程式，我們在 terminal 打的 docker 指令，真正接手去管理 image、container、network、volume 的，就是 dockerd，Docker 官方把它描述成「持續運行、管理 containers 的 process」，而 Docker Engine 也被官方定義成一個 client-server 架構，其中 server 端就是 dockerd


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/daemon/4_dockerd_include.png)


<!-- endtab -->


<!-- tab Docker Engine-->

Docker Engine 是整套 Docker 執行核心，它包含 Docker daemon（dockerd）、REST API、以及 Docker CLI client
，官方文件把它定義成一個 client-server technology，裡面有三塊很核心的東西


- dockerd 這個 long-running daemon
- REST API
- docker CLI client



<!-- endtab -->


<!-- tab OCI 規範-->


`OCI` 規範是 `container` 世界的共同語言，不先講好格式和動作，不同工具根本接不起來


`OCI` 是 `Open Container Initiative` 制定的一組標準，它的工作之一就是建立 `container image formats` 和 runtime 的正式規範，讓相容實作能跨平台運作。OCI 官網也列出目前三大規範：`Runtime`、`Image`、`Distribution`

- `OCI Runtime Spec`：規定一個低層 `runtime` 應該怎麼把 `container` 跑起來
- `OCI Image Spec`：規定 `image` 的格式該怎麼描述



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/daemon/5_OCI.png)


<!-- endtab -->

<!-- tab containerd-->


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/daemon/6_containerd.png)


`dockerd` 把 `container` 生命週期管理交給 `containerd`，再由 `containerd` 用 `runc` 去真的執行

- Docker Engine uses containerd for managing the container lifecycle
- By default, containerd uses runc as its container runtime



- `dockerd`：Docker 世界的總管，管 API、image、network、volume、整體協調
- `containerd`：中間那層，專門處理 container lifecycle 和 image management
- `runc`：最底層 runtime，真的照 OCI 規格把 container process 跑起來



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/daemon/7_runc.png)


<!-- endtab -->


<!-- tab summary-->


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/daemon/8_all_flow.png)



<!-- endtab -->



{% endtabs %}