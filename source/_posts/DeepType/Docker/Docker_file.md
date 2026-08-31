---
title: Dockerfile
date: 2026-01-04 14:09:05
categories: Docker
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
tags:
    - Docker
toc:
toc_number:
comments :
---


{% tabs Dcokerfile%}

<!-- tab Dcokerfile-->

Dockerfile 是「怎麼做」的規則，而 Image 是「照規則做完後」的結果；一個負責描述過程，一個負責被拿來用，Dockerfile 本質上就是一份「把環境準備流程寫成程式碼」的說明書，讓電腦每次都用一模一樣的步驟，生出一模一樣的執行環境


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Server/Devops/Docker/dockerfile.png)



假設寫了一個後端 API，你的電腦可以跑，而同事電腦版本不合，可能是因為環境完全不同，如果沒有 Dockerfile，只能用嘴巴或 README 描述：「先裝這個、再裝那個、版本要對喔」，而有 Dockerfile，等於是直接說「照這份 Dockerfile build，就一定能跑。」


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/dockerfile/2_everyone_can_run.png)


<!-- endtab -->

<!-- tab 工作流程-->


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/dockerfile/3_docekerfile_core.png)


## 撰寫 Dockerfile

1. 先寫 Dockerfile ，其中裡面要寫清楚從哪個基底環境開始、要裝哪些套件、程式放哪、容器啟動要幹嘛
2. 設定工作目錄，決定之後所有指令是在容器裡的哪個資料夾執行
3. 準備需要的檔案，把程式碼、設定檔複製進容器
4. 安裝依賴套件，用指令把程式需要的套件一次裝好
5. 設定啟動方式，告訴 Docker 這個環境「跑起來」時，要執行哪個指令

👉 Docker 會照這個順序，一行一行執行，最後產生一個「映像檔（Image）」


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/dockerfile/4_dockerfile_image_container.png)



## 執行 build

Docker 讀 Dockerfile，一行一行照做產生 Image，每一步都變成一層（layer），最後組合成一個 Image


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/dockerfile/7_dockerfile_layers.png)


## Image 被拿來用

可以開 Container、可以丟給同事、可以丟上 Registry

當我需要 Ubuntu + Node 18 + npm install 來跑 server.js，Image 表示「好，這一包已經全部準備好了」，如果程式壞了，就改 Dockerfile → 重新 build → 新 Image，而不是「進去 Image 裡面手改」


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/dockerfile/8_adject_dockerfile_rebuild.png)



<!-- endtab -->

<!-- tab Dockerfile 實作-->


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/dockerfile/5_dockerfile_simple_steps.png)



## 準備

```bash
## 準備資料夾
mkdir dockerfile-demo
cd dockerfile-demo

## 建立App
New-Item server.js -ItemType File

## Dockerfile 建立
New-Item Dockerfile
```

<br>

## server.js

```javascript
//node server.js 執行專案
const http = require("http");

http.createServer((req, res) => {
  res.end("Hello World\n");
}).listen(3000);

console.log("Server running at http://localhost:3000");
```

<br>

## Dockerfile 撰寫

```dockerfile
# 1) 選一個 base image（這是「唯讀層」的起點）
FROM node:20-alpine

# 2) 設定工作目錄（之後 COPY / RUN 都以這裡為基準）
WORKDIR /app

# 3) 把程式複製進 image（這會形成一層新 layer）
COPY server.js .

# 4) 宣告容器會用到的 port（只是宣告，真正開放要靠 -p）
EXPOSE 3000

# 5) 容器啟動時預設執行的命令
CMD ["node", "server.js"]
```

FROM、WORKDIR、COPY、EXPOSE、CMD，這就是 Dockerfile 最核心的骨架


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/dockerfile/6_dockerfile_commands.png)



## Build：從 Dockerfile 做出 image

```bash
## 把「Dockerfile + 當下目錄內容」➜ 組裝成一個 Image
## 幫這個 image 命名 + 打 tag
## .（最後那個點）, 「我把 目前這個資料夾交給 Docker，請你照 Dockerfile 製作 image」
## 把 . 目錄「打包」 , 傳給 Docker daemon , Dockerfile 裡的 COPY 只能用這個範圍內的檔案
docker build -t hello-dockerfile:1 .


## 驗證 image 出現
## docker images 列出目前電腦上所有 Docker images
## |（pipe）把左邊的輸出，交給右邊處理
## 從輸入的每一行中，找出「包含 hello-dockerfile 的行」
docker images | Select-String hello-dockerfile
```

<br>

## Run：用 image 啟動 container

```bash
## 把容器的 3000 映射到你本機 3000
docker run --name hello1 -p 3000:3000 -d hello-dockerfile:1

## 驗證容器有在跑
docker ps | Select-String hello1

## 測試
curl http://localhost:3000
```

<br>

## 進入 container 看檔案

```bash
## 進去 shell
docker exec -it hello1 sh

## 在裡面看 /app
ls -l
cat server.js
exit
```

## 清理

```bash
## 進去 shell
docker stop ./hello1
docker rm ./hello1
docker rmi hello-dockerfile:1
```

<!-- endtab -->


<!-- tab Dockerfile 實作-->



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/dockerfile/9_final.png)


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/dockerfile/dockerfile.png)


<!-- endtab -->


{% endtabs %}