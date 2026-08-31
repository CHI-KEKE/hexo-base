---
title: Container
date: 2025-12-25 17:54:05
categories: Docker
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
tags:
    - Docker
toc:
toc_number:
comments :
---


{% tabs Container%}

<!-- tab 容器 Container-->

容器是一種「輕量化的應用執行環境」，讓你的應用程式能在任何地方執行，而且不會受到作業系統或環境差異的影響，本質上，是把「應用能不能跑」這件事，從環境問題，變成一個已經被打包解決的事!你不需要再賭「這台機器剛好跟你的一樣」，因為容器已經把所有必要條件一起帶著走

像錄影帶，裡面不只是程式碼，而是一段已經被錄好的「執行過程」：什麼時候啟動、用哪些資源、環境長什麼樣、第一秒該做什麼，只要你有能播放它的播放器（Docker），不管是在你的電腦、同事的機器、測試環境還是正式環境，播出來的結果都會一樣。


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/container/2_tape.png)



1. 我們所需要做的事情是，應用程式本身要準備好，也就是你的程式碼，可能是一個 API、後端服務或指令工具
2. 接著我們把應用需要的東西一起裝進來，包含語言執行環境（例如 Node、Python）、函式庫、系統套件，這一步是在解決「我電腦可以跑，你電腦不行」的問題
3. 形成一個獨立的執行空間，這個空間跟其他應用隔離，不會亂用彼此的資源或版本
4. 在任何地方啟動這個容器，不管是你的筆電、同事的電腦、測試機、正式環境，只要能跑容器，結果一樣! (╯✧∇✧)╯

它解決了團隊的溝通，有容器之後開發、測試、正式環境用同一個包，問題更容易重現，新人加入不用照一份神秘環境安裝文件，只要有一份 YAML，加上一個已經定義好的 image，就足夠讓這個應用在任何地方用同樣的方式被跑起來


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/container/3_run_everywhere.png)


<!-- endtab -->

<!-- tab 為什麼要「輕量化-->

因為我們想要執行特定的應用，並不需要一台新電腦，我們只要一個能跑你程式的空間!

如果不用容器，作法可能是裝一整套作業系統、再裝一堆你其實用不到的服務

就像要看一部劇，卻需要把整個請原班人馬重新搭建起來在演一次給你看，容器只保留「必要到不能再少的東西」，啟動快、佔空間小，更容易大量複製


<!-- endtab -->

<!-- tab 隔離-->

隔離是為了讓問題不要互相傳染

沒有隔離時會遇到許多問題

- A 專案升級套件 → B 專案壞掉
- 一個服務記憶體爆掉 → 整台機器卡死

現在容器把每個應用關在自己的空間裡，壞了也只有他自己臭酸，而且重啟、刪除都很乾脆，這對維運和除錯影響非常大

不同容器看不到彼此的

- Process (行程)
- Network (網路)
- File System (檔案系統)
- Hostname

如果在一個容器 A 執行
```bash
ps -aux
```

它只會看到自己的行程，完全看不到容器 B 的，這是因為每個容器有自己的 process namespace，就像你在旅館的不同房間，彼此聽不到聲音，看不到對方


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/container/5_namespace.png)


<!-- endtab -->

<!-- tab Deploy（部署方式的演進）-->

部署的本質不是「把程式跑起來」，而是「讓不同應用在同一台機器上，互不干擾、可預期地長期運行」，程式不是寫完就好，還要「安裝、升級、維護、共存」

## 傳統部署

當我們直接把 App 裝進作業系統，App 跟 OS、Library 強烈耦合、發生套件版本衝突，而且一個 App 出事，整台機器陪葬，這是傳統部署的痛點

例如有一台 Windows Server，A 專案需要 .NET 6、B 專案需要 .NET 8，結果只能硬升整台機器或複製一台新機器
或者在 Windows 上同時安裝 IIS 網站 + Apache Server，常會因 Port、Library 衝突導致錯誤

正因為幾乎沒隔離，會直接影響我們敢不敢多放服務、多切環境、多做實驗

<br>

## 虛擬化部署 (Virtualized Deployment)

用 VM 把「整個 OS」包起來，這可以解決衝突，但成本還是很高，例如使用 Hypervisor (虛擬機管理程式) 讓同一台機器能跑多個虛擬機 (VM)，每個 VM 有自己的作業系統（Guest OS），問題是每個 VM 都要一份 OS，非常重！啟動慢、浪費記憶體與 CPU，啟動一個服務像在重開一台電腦

VM 做到 OS 等級隔離，但浪費在「OS 重複存在」，他適合長期存在的環境，當隔離得越下層（Kernel 以下），成本就越高；越靠近應用，彈性就越大

<br>

## 容器部署 (Container Deployment)

容器部署不再複製 OS，只隔離「應用 + 函式庫」，大家共用 Kernel，資源使用極小化

容器的實際價值
```bash
docker run -d --name web1 nginx
docker run -d --name web2 httpd
docker run -d --name web3 dotnetapp
```

這三個 App 彼此隔離、不需要三份 OS、幾秒內就能起來

Container 是 Process / Namespace 等級隔離，他只為「應用本身」付成本，適合隨開隨丟的服務

當服務可以秒開，你的系統設計就不再害怕「重來一次」，這是 DevOps、CI/CD 能成立的底層原因之一，不只是方便而已


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/container/4_buil_env_difference.png)



<!-- endtab -->

<!-- tab cgroup (Control Group) 「資源控制」-->

cgroup 的本質就是先把資源「切好額度」，再讓每個程式在自己的額度裡跑，避免一個失控的程式把整台機器拖垮，不是為了效能最佳化，而是為了風險控制與公平性

假設
```bash
docker run -d --memory="512m" --cpus="1" nginx
```

1. Docker 啟動了一個容器，`docker run -d nginx` 本質上只是啟動一個 `Linux process`（只是被包在 `namespace` 裡）
2. Docker 幫這個 container 建立對應的 cgroup，在 Linux 底層建立一組資源控制規則，CPU、Memory、I/O 各自有對應的控制器
3. 設定 Memory 上限 `--memory="512m"`，kernel 會追蹤該 cgroup 使用的實體記憶體，一旦超過 → 觸發 OOM（不是整台機器，是這個 container）
4. 設定 CPU 使用配額 `-cpus="1"`，即使機器有 32 cores，scheduler 只會讓它「用到 1 顆的時間片」，容器內的程式「完全不知道」這件事，nginx 以為自己在正常跑，但 kernel 已經在後面幫你踩剎車

想像在一台 VM 上跑 API Server

- 如果沒有 cgroup，某個 worker memory leak 導致 RAM 吃滿 → 系統開始 swap，其他服務全部變慢，最後整台 VM 掛掉
- 如果有 cgroup，那個 container 先被 OOM kill，其他服務完全不受影響，你只需要重啟或修那一個 container

差別不是效能，是「事故範圍」



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/container/6_cgroup.png)


<!-- endtab -->

<!-- tab rootfs (Root File System)「容器的根檔案系統」-->

容器的 rootfs 是一個「看起來完整、實際上是拼接出來的檔案系統」，讓應用以為自己擁有整台機器，但其實只是被隔離的視角，每個容器都有自己獨立的根目錄 /，裡面包含應用需要的所有檔案、Library、設定。但實際上底層會共用宿主機的檔案，只在有修改時才產生差異層（Copy-on-Write），而 Docker Image 其實是一層層的 rootfs，它把整個檔案系統拆成一層層可重用的變化紀錄，目的是讓「相同的東西只存一次，不同的地方才另外付成本」。

```bash
ubuntu:latest
 ├─ layer 1: base system
 ├─ layer 2: apt packages
 ├─ layer 3: your app
```

像是用透明膠片疊起來的投影片，底層是共同的背景，上層各自疊加需要的內容

容器啟動時，一定會先有一個 /


對應用來說

- /bin
- /etc
- /usr
- /lib

這就是它「認知中的世界」，這個 / 就叫做容器的 Root File System，是應用能看到、能操作的最上層檔案系統，rootfs 不是從宿主機複製一整份出來，而是由 Image 的多個 layer「疊」出來，底層檔案實際上可能和其他容器共用，Docker 用 OverlayFS 把多層合成一個 rootfs，多個唯讀層（Image layers），上面再加一個可寫層（Container layer），對應用來說：只有一個 /，它只認 rootfs，不知道底下發生什麼事，不知道哪些檔案是共用的，不知道哪些是 Copy-on-Write，只知道在自己的系統裡跑

你在 container 裡執行
```bash
ls /
```
看到
```bash
bin  etc  lib  usr  var  app
```

但實際上

- /bin 可能來自 Ubuntu base layer
- /usr/lib 來自 apt install 的 layer
- /app 來自你 COPY 進來的 layer
- /var/log 的修改存在 container 自己的可寫層

👉 這些在磁碟上分散在不同地方，但在 rootfs 視角裡是「一個完整系統」


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/container/7_roofs.png)



<!-- endtab -->

<!-- tab 如何建立容器化應用-->

1. 建立一個簡單的 .NET Web API
```bash
dotnet new webapi -n MyApi
cd MyApi
```

2. 建立 Dockerfile (Docker 建立映像的過程中，會把每一行指令都變成一層（Layer）)
```dockerfile
# 1️⃣ Build 階段：產生執行檔
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /out

# 2️⃣ Runtime 階段：組出最終 rootfs
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /out .
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

- `Build stage` 建立一個「暫時的 rootfs」目的是產生可執行檔
- `Runtime stage` 建立一個「乾淨、最小化的 rootfs」只包含 Linux、.NET Runtime、MyApi.dll

- `FROM` 指定你的映像檔要「建立在誰之上」
- `mcr.microsoft.com/dotnet/aspnet:8.0` 這是 微軟官方維護 的 ASP.NET Runtime 容器映像。它裡面已經包含：
  - .NET 8 runtime
  - ASP.NET Core 的必要環境（Kestrel、環境變數設定等）
  - Linux 系統（通常是 Debian 或 Alpine）
- `WORKDIR /app`「把容器裡的工作目錄 (Working Directory) 設定成 /app」
  - 容器啟動後，它有自己的檔案系統（rootfs）
  - 想像進入容器時會有個「當前資料夾」，WORKDIR 就是設定那個資料夾，我坐在 /app 這個座位上開始工作
  - 所有接下來的命令（像 COPY、RUN、ENTRYPOINT）都會在 /app 底下執行
- `COPY . .` 「把本機（你的開發機）當前目錄下的所有檔案，複製到容器裡的 /app 目錄」
  - `前一個 .` 主機端（Host）的目前目錄
  - `後一個 .` 容器內的目前工作目錄（上面設成 /app）

3. 用 Dockerfile 當藍圖，去組出一個新的 rootfs（Image）

```bash
## 在這個目錄找 Dockerfile 把整個目錄內容（context）打包 讓 Dockerfile 裡的 COPY . . 使用這些檔案
docker build -t myapi .
docker run -p 8080:8080 myapi
```

![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/container/8_docker_file.png)

<!-- endtab -->



<!-- tab summary-->



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/container/docker-container.png)



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/container/9_final.png)


<!-- endtab -->

{% endtabs %}