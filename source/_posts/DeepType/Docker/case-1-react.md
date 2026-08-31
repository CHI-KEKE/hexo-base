---
title: Case 1 - Build a REACT APP Image
date: 2026-04-08 07:32:05
categories: Docker
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
tags:
    - Docker
toc:
toc_number:
comments :
---

{% tabs Case 1 - Build a REACT APP%}




![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/case-1-react/1_react_multi-stage.png)



<!-- tab 事前準備-->


使用最簡單的方式建立一個 react template

```bash
npm create vite@latest myweb -- --template react
npm run dev
```

<!-- endtab -->


<!-- tab Dockerfile-->


![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/case-1-react/2_why_multistage.png)



```dockerfile
# build stage
### FROM：選擇基底映像（base image），這裡用的是官方的 node:18-alpine
### alpine：表示這是基於 Alpine Linux 的輕量版本，檔案體積小（數 MB 級別）
### as builder：給這個階段取一個名字叫 builder，後面可以用 COPY --from=builder 來取這個階段的成果
### 第一個階段專門負責「編譯（build）」程式碼，因為 Node.js 開發工具比較重（安裝很多套件），不需要帶到最終生產映像裡。多階段建置可以讓最終 image 更小。
#### 若有 node 版本問題改這
FROM node:20-alpine AS builder

# 建立工作目錄
### 設定工作目錄（Working Directory）。後續的 COPY、RUN 指令都會以 /app 為基準，不需要每次都打絕對路徑
### 保持檔案結構整潔，所有檔案都集中在 /app
WORKDIR /app

# 複製相依檔案 :　把 package.json 跟 package-lock.json 複製到 image 中
### 把本地端的 package.json 和 package-lock.json 複製進去 container 的 /app 目錄
### 安裝依賴只需要這兩個檔案。提前複製它們（而不是一開始就複製全部檔案）有助於 Docker build cache，避免每次有程式碼變動就重裝依賴。
COPY package*.json ./

# 安裝相依套件
### npm ci：用 package-lock.json 精準安裝所有相依套件（比 npm install 更確保一致性）
### npm cache clean --force：清除 npm 的快取，減少映像檔大小
### 使用 npm ci 可以確保 build 環境與本地一致，避免版本不符
### 清 cache 減少不必要的體積
RUN npm ci && npm cache clean --force

# 把所有檔案複製到 image 中
### 把專案資料夾的所有檔案複製進 container（此時 node_modules 已經裝好了）
### 需要完整的專案檔案（包含 src、public 等）才能執行編譯
COPY . .

# 執行 build
### 執行 package.json 裡定義的 build 指令，通常是 react-scripts build 或 Vue/Angular 的 build
### 會在 /app/build（React）或 /app/dist（Vue）等資料夾輸出純靜態檔案
### 這個階段的目的是把程式碼編譯成靜態 HTML/CSS/JS 檔案，生產環境只需要這些檔案，不需要 Node.js
RUN npm run build

#################################
# production stage
### 選用 nginx:alpine 作為最終的基底 image。Nginx 是高效能 Web Server，適合用來提供靜態檔案
### 我們的 build 輸出已經是純靜態檔案，不需要 Node.js Runtime。把靜態檔案交給 Nginx 來 serve，更輕量也更快
FROM nginx:alpine

# 建立工作目錄
### 設定工作目錄到 Nginx 預設的靜態檔案路徑 /usr/share/nginx/html
### 讓後面的 COPY 直接複製檔案到正確位置，不需要額外設定
WORKDIR /usr/share/nginx/html

# 從 builder 階段裡的 /app/build 複製到目前位置（WORKDIR）
### --from=builder：從第一階段（builder）裡複製檔案
### /app/build 是第一階段的 build 輸出資料夾
### . 是目前的工作目錄（Nginx 預設靜態檔案路徑）
### 只把最終需要的靜態檔案帶進來，Node.js 和 node_modules 都不會進到最終映像裡，保持映像檔精簡
COPY --from=builder /app/dist .
```


## 第一階段（builder）

- 安裝套件
- 編譯前端程式
- 輸出靜態檔案


![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/case-1-react/3_build_stage.png)




## 第二階段（production）

- 只留下靜態檔案
- 用 Nginx 提供服務



![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/case-1-react/4_prodct_stage.png)



Dockerfile 有兩個 stage，build 過程會經過兩個 image 階段，預設最後只產出最後一個 image，builder 階段通常只是中間站，不是要部署的正式 image


最後這個 image 裡面會有：

1. nginx:alpine 本身的內容

- Linux 基底環境（Alpine）
- Nginx 執行檔
- Nginx 預設設定
- 啟動 Nginx 所需的檔案

2. 靜態網站檔案

從 builder 複製過來的

- index.html
- JS bundle
- CSS
- 圖片
- 其他 dist 內的靜態資源

而且這些檔案會被放到 /usr/share/nginx/html

這正是 Nginx 預設拿來對外提供靜態檔案的目錄



![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/case-1-react/5_compare_load.png)


<!-- endtab -->

<!-- tab 建立 Docker image-->

```bash
## -t 是 tag（標籤） 的縮寫
docker image build -t yoyo88147/myweb .
```

建立完成，可以用 docker image ls 確認看看是否存在



![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/case-1-react/6_image_check.png)


<!-- endtab -->

<!-- tab 試跑 & 推送到 Docker Hub-->


```bash
## 試跑
docker container run -it -p 8081:80 --rm yoyo88147/myweb


## 訪問
http://localhost:8081

## 把 image 推送到 Docker Hub
#### 須注意 帳號需為 yoyo88147
####　docker login 登入
docker image push yoyo88147/myweb
```



![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/case-1-react/7_final.png)


<!-- endtab -->

{% endtabs %}