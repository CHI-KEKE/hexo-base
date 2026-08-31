---
title: Container is environment
date: 2026-04-27 10:14:05
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

概念上是把 **環境** 抽離出你的電腦，讓專案去哪裡跑都長一樣，不再被本機設定搞壞，此時我們可以透過 Docker container 取代「本機安裝 Node.js」，來提供一致的開發環境



![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/docker-is-env/2_seperate_env_from_files.png)



<!-- endtab -->


<!-- tab 原本流程開發的問題-->

1. 本機先裝 Node.js
2. 進專案資料夾
3. 跑 `npm install`
4. 跑 `npm start`


雖然在自己的電腦開完了，但自己的電腦要裝 Node 18、同事可能裝 Node 16、CI/CD 用 Node 20


![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/docker-is-env/1_version.png)



<!-- endtab -->


<!-- tab 改用 Docker 自建環境開發-->


現在改成 Docker 做的事

```bash

# 還沒有react

#### 建一個測試資料夾
mkdir docker-react-test
cd docker-react-test

#### 用 Docker 建立 React 專案
docker container run -it --rm -w=/app -v "${PWD}:/app" node:18 npx create-react-app .

#### 用 Docker 啟動 React
docker container run -it --rm -p 3000:3000 -w=/app -v "${PWD}:/app" node:18 npm start

####
npm i
npm start 


# 已有 react 專案

#### cd 你的-react-專案, 確認裡面有 package.json
cd react

#### 依賴是由 container 裡的 Node.js 安裝的，不是本機 Node.js。
#### 這一步會在你的專案裡產生：node_modules / package-lock.json
docker container run -it --rm -w=/app -v "${PWD}:/app" node:18 npm install

## (1) node:18 : 啟動一個已經裝好 Node.js 18 的環境, 等於你「不用在本機安裝 Node」
## (2) -v "${PWD}:/app", 把你本機的專案資料夾，掛到 container 裡, container 其實在用你的原始碼
## (3) -w=/app, container 一進去就在 /app（你的專案目錄）, 等同 cd 進專案
## 在 container 裡啟動 React, 就像平常在本機做的事
docker container run -it --rm -p 3000:3000 -w=/app -v "${PWD}:/app" node:18 npm start

## test
http://localhost:3000
```


<!-- endtab -->


<!-- tab volume 掛載（bind mount）-->

container 本身通常是短暫的，用完就刪。但你的 React 原始碼不能跟著消失，所以要把本機資料夾掛進 container。這樣你在 container 裡改動、安裝依賴，本機資料夾也會看到結果



![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/docker-is-env/3_shorttime-container-and-longtime-file.png)



我們執行 Docker 時加上

```bash
-v "${PWD}:/app"

```

`${PWD}` 是「你本機的資料夾」
`/app` 是「container 裡的路徑」



![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/docker-is-env/4_volumn_is_key.png)





Docker 做的事是把這兩個路徑 “接在一起”，接起來之後


- container 看到 /app 裡的檔案 = 其實就是你本機的檔案
- container 在 /app 新增檔案 = 本機也會出現
- container 修改檔案 = 本機同步被改


![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/docker-is-env/5_machine-container-sync.png)



所以在 container 裡跑 `npm install`


實際發生的是 node_modules 被安裝在 /app，而 /app 其實是本機資料夾，所以本機就會看到 node_modules


假設本機是 Node 20 而 Docker 用的是 node:18


```bash
docker container run -v "${PWD}:/app" node:18 npm install
```


- 安裝行為 → 用 Node 18 規則
- node_modules → 寫在你本機

所以 node_modules 是「用 Node 18 算出來的結果」，即使本機是 Node 20，也完全沒參與



![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/docker-is-env/6-use-container-npm-rule.png)



<!-- endtab -->



<!-- tab summary-->


![command](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/docker-is-env/docker-env.png)


<!-- endtab -->


{% endtabs %}