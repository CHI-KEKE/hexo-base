---
title: Cammand
date: 2026-01-11 08:47:05
categories: Docker
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
tags:
    - Docker
toc:
toc_number:
comments :
---

{% tabs Cammand%}

<!-- tab 指令-->



![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/command/1_docker_easysay.png)



Docker 把「你要操作的東西」放在指令正中間，為了讓人一眼就知道「你在動誰、要對它做什麼」

- `docker`：告訴系統你要使用 Docker 這套工具，就像進入一個控制台，後面的內容都在 Docker 的語境裡
- `container` / `image` / `volume` / `network`（對象）：明確指定你要操作的資源種類
先講「我要動的是容器？映像檔？還是網路？」
- `run` / `ps` / `rm` / `build`（指令）：對這個對象要做的行為，在對象已經清楚的前提下，再說你要做什麼動作
- 參數：補充條件與細節，像是名稱、port、detach、env，這些都是「怎麼做」的細節


工具 → 目標 → 動作 → 細節

例如
```bash
docker container rm my-nginx
```

用 docker
對 container
做 rm（刪除）
目標是 my-nginx

這句話幾乎可以直接唸成白話，不需要先背語法規則! Docker 的 CLI 明顯優先考慮「可讀性」

<!-- endtab -->


<!-- tab 觀察 container 狀態-->

```bash
# 列出目前在執行中的 container, 如果 container 已經 stop 看不到
docker container ls

# 列出所有 container
## 預設不是 -a 是因為在日常操作時，99% 真正關心的是「現在跑著什麼」，而不是「歷史遺跡」
docker container ls -a
```
![docker_container_list_a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/docker_container_ls_a.png)

挖靠，經過剛剛這樣理解，輸出這些指令跟呼吸一樣呢!

<br>

## 1️⃣ CONTAINER ID

這個 container 在 Docker 世界裡真正的唯一身分證，真正的完整 ID 很長，CLI 只顯示前幾碼，所有 Docker 內部操作其實都靠它，名字可以改，ID 不會變

<br>

## 2️⃣ IMAGE

這個 container 是「用哪一張 image 生出來的」，告訴你它的來源藍圖，不是「現在在跑什麼」，而是「它本來是什麼」
，container ≠ image，它只是 image 的一個實例

<br>

## 3️⃣ COMMAND

container 啟動時「PID 1 跑的是什麼指令」，通常來自 image 的 CMD 或 ENTRYPOINT，Docker 只負責啟動這個指令，之後就不管了，container 的生命週期 = 這個 command 的生命週期

<br>

## 4️⃣ CREATED

這個 container「被生出來」多久了，跟有沒有在跑無關，即使 stop 了，時間還是會累積，container 是一個物件，不是一個瞬間行為

<br>

## 5️⃣ STATUS

現在這個 container「活著嗎？死了多久？怎麼死的？」，看到的常見狀態像這樣

- Up 45 seconds → 正在跑
- Exited (0) → 正常結束
- Exited (1) → 程式出錯

<br>

## 6️⃣ PORTS

這個 container 有沒有「影響到主機對外的網路」，例如 0.0.0.0:8080 → 8080/tcp，意思是主機 8080 對應到 container 內 8080，只有 port mapping 的 container，才真的「碰得到外面的世界」

<br>

## 7️⃣ NAMES

人類用來記住 container 的暱稱，沒指定就由 Docker 自動產生（例如 nostalgic_kare），可以用來取代 CONTAINER ID 操作



s

![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/command/2_container_7_fields.png)



<!-- endtab -->

<!-- tab 觀察 Image-->

```bash
docker image ls
```

![docker_image_list](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/image_list.png)

<br>

## 1️⃣ REPOSITORY

這張 image 在「邏輯上」屬於哪一個專案 / 名稱空間，例如 node、redis、nginx、yoyo88147/ashleylaiodemoweb，而官方 image 通常只有名字（node），自己或公司 image 會有 namespace（user/repo），這是給人分類與搜尋用的，不是唯一身分

<br>

## 2️⃣ TAG

同一個 repository 底下的「版本標籤」，我們看到的 latest、18、20、20-updated、<none>，tag 只是「指標」，同一個 image 可以被多個 tag 指到

<br>

## 3️⃣ IMAGE ID

Docker 真正認得的 image 身分證，這才是 image 的本體，就算 tag 被改、被刪，IMAGE ID 還是同一個，Docker 世界裡，永遠是 ID 說了算

<br>

## 4️⃣ CREATED

這個 image 被 build 出來的時間，不是你 pull 的時間，是 image「誕生」的時間，image 是一個產物，不是即時生成的東西

<br>

## 5️⃣ SIZE

這張 image 解壓後大概要吃掉多少磁碟空間，你可以立刻看出差異，例如 alpine：8.31MB、node：1.0GB+



![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/command/3_image_5_fields.png)


<!-- endtab -->

<!-- tab 拉取 api server image-->

```bash
# 先把 image 從 DockerHub 或其他 registry 上 pull 下來(namespace/repository:tag)
 docker image pull mcr.microsoft.com/dotnet/samples:aspnetapp
```

![docker_image_pull](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/image_pull_aspnet.png)

docker image pull 本質不是「下載檔案」，而是「確認本機是否已經擁有某個 image 版本的內容指紋」

<br>

## 1️⃣ 判斷你有沒有指定 tag

如果出現
```bash
Using default tag: latest
```

代表只給了 repository，Docker 自動補上 :latest 實際查詢的是 mcr.microsoft.com/dotnet/samples:lastest

<br>

## 2️⃣ 到 registry 查詢這個 tag 指向哪個 image

```bash
latest: Pulling from dotnet/samples
```

Docker 到遠端 registry（預設是 Docker Hub）問 dotnet/samples 現在指向哪一個 image？

<br>

## 3️⃣ 取得 image 的「內容指紋（Digest）」

```bash
Digest: sha256:17685f38aedd...
```

本質是這張 image 的內容，經過雜湊後的唯一結果，只要內容有一個 byte 不一樣，digest 就一定不一樣

<br>

## 4️⃣ 比對本機是否已經有相同 digest

```bash
Status: Image is up to date for dotnet/samples
```

代表本機已經有一張 image，而且 digest 完全一樣，所以不需要再下載任何 layer，pull ≠ 一定會下載


![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/command/5_docker_pull_digest.png)


## 5️⃣ 顯示完整的 image reference

mcr.microsoft.com/dotnet/samples:aspnetapp，Docker 把剛剛「隱含省略」的資訊補齊 

- registry：docker.io
- repository：mcr.microsoft.com/dotnet/samples
- tag：laspnetapp

[registry]/[namespace]/[repository]:[tag]

namespace 通常是使用者名稱或組織名稱，用來「分隔誰擁有這個 repository」，Namespace = image 的擁有者空間


```bash
# 列出 image 確認
docker image ls
```

找不到時也可以嘗試看看直接搜尋

```bash
docker search {keyword}
```

<!-- endtab -->

<!-- tab 將 server 在本地跑起來-->


![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/command/4_image_vs_container.png)


docker run 的本質是用一張 image，立刻生成一個 container，設定好它怎麼活、怎麼連外、什麼時候死

```bash
docker container run -it --rm -p 3001:8080 mcr.microsoft.com/dotnet/samples:aspnetapp
```

<br>

## 1️⃣ docker container run

這不是單一動作，而是組合技，他是 run = create + start（加一堆設定）

- 沒有 image → 先 pull
- 有 image → 建立 container

建好 container → 立刻啟動


![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/command/5_2_docker_run_pull_create_start.png)


## 2️⃣ -it

其實是兩個 flag

- -i：保持 STDIN 開啟
- -t：配置一個 pseudo-TTY

本質是讓 container「像在前台跑的程式」一樣跟你互動，如果沒有 -it，程式照跑，但無法輸入、也不好看輸出，加上 -it 的啟動方式，會讓終端機視窗被佔用，但好處是，可以比較方便地看到 container 中的 application 的 log 的顯示。

<br>

## 3️⃣ -p 3001:3000

port mapping，格式是 `主機 port : container 內 port`

這行的意思是瀏覽器連 localhost:3001，而實際請求被轉進 container 的 3000，container 世界是封閉的，port 是打洞的方式

Container 裡的程式先決定自己用哪個 port，例如 Node.js 服務在 container 裡 listen(3000)，這個 port 是 image 作者寫死或約定好的，Docker 啟動 container 時，container 自己是「封閉世界」，而 Container 裡的 3000，外面的 Host 是完全看不到的，就算服務有跑，也只有 container 內部能連!

-p local:container 是在建立一條對外通道
- container port：服務實際在聽的門口（不能亂改）
- local port：Host 幫你開一個對外入口（你可以選）

Docker 負責把流量轉送，使用者連 localhost:3001，Docker 幫你轉進 container 的 3000


![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/command/6_port_mapping.png)


## 4️⃣ --rm

告訴 Docker，這個 container 一旦停掉就順手把它刪掉，本質是一個「用完即丟」的 container，非常適合 demo、本機測試、試跑別人的 image

<br>

## 測試

用瀏覽器試試看 Localhost，port number 要換成啟動 container 指令中的 local port#

<!-- endtab -->


![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/command/7_table.png)



![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/command/docker_decoded.png)


{% endtabs %}