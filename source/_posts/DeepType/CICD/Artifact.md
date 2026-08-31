---
title: Artifact
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


{% tabs Artifact%}


<!-- tab Artifact-->

Artifact 就是「程式碼經過建置後產生、可以被保存與交付的成品」，讓後面的部署流程不用重新猜你的程式怎麼變成可執行狀態


![c](https://github.com/CHI-KEKE/pics/blob/main/CICD/Artifact/1_what_is_artifact.png?raw=true)


<!-- endtab -->


<!-- tab 前端 Node.js 生態系的 build 流程-->


![c](https://github.com/CHI-KEKE/pics/blob/main/CICD/Artifact/2_artifact_in_flow.png?raw=true)



假設有一個 React 專案

```bash
my-app/
  src/
    App.tsx
    main.tsx
  package.json
  package-lock.json
  vite.config.ts

```

## 轉成某種「可以被執行或部署的結果」

Pipeline 會執行

```bash
## 根據 package.json 安裝專案需要的套件, 執行後會產生或更新 node_modules/ package-lock.json
## node_modules/      實際下載回來的套件
## package-lock.json  紀錄這次安裝到的精確版本
npm install 

## OR 
## 這行也是安裝套件，但它比較適合 CI/CD，可以理解成 Clean Install，它會根據 package-lock.json 安裝完全一致的套件版本
## 會先刪掉 node_modules/ 一定照 package-lock.json 安裝，如果 package.json 跟 package-lock.json 對不起來，會直接失敗
npm ci


## 執行 package.json 裡定義的 build script
## 把 TypeScript 編譯成 JavaScript
## 把 React/Vue 原始碼轉成瀏覽器看得懂的 HTML/CSS/JS
## 壓縮 JS 和 CSS
## 產生正式環境用的檔案
## 移除開發用資訊
## 建置後常見會出現：
##  dist/
##  build/
##  .next/
##  out/
npm run build
```

這個階段會把原始碼轉成某種「可以被執行或部署的結果」


```bash
dist/
app.jar
build/
target/
```

## 將建置結果打包

系統可能會把 build 出來的結果壓成一包

```bash
## 這行是在把 dist/ 資料夾壓縮成一個檔案 app.tar.gz
## tar       打包工具
## -c        create，建立一個新的壓縮包
## -z        使用 gzip 壓縮
## -f        指定輸出的檔案名稱
## app.tar.gz  產生的檔案
## dist/        要被打包進去的資料夾
tar -czf app.tar.gz dist/
```

或產生

```bash
app.jar
frontend-build.zip
my-service-image.tar
```


這個打包後的東西，就是常見的 artifact，CI/CD 會把這個成品上傳到某個地方保存，例如

- GitHub Actions Artifacts
- GitLab Job Artifacts
- Jenkins Archive Artifacts
- Nexus
- Artifactory
- S3
- Container Registry


![c](https://github.com/CHI-KEKE/pics/blob/main/CICD/Artifact/4_artifact_repo.png?raw=true)



後續的部署 job 就可以直接下載它


部署階段下載 artifact


```bash
download artifact
deploy artifact to server
```

也就是拿已經 build 好的成品 → 放到測試環境 / 正式環境，這樣可以確保「測試過的那一份」跟「部署出去的那一份」是同一份


![c](https://github.com/CHI-KEKE/pics/blob/main/CICD/Artifact/3_same_in_diff_env.png?raw=true)


<!-- endtab -->


<!-- tab Artifact 跟 Docker Image 的關係-->


![c](https://github.com/CHI-KEKE/pics/blob/main/CICD/Artifact/5_image_is_artifact.png?raw=true)



Docker image 也可以被視為一種 artifact，只是它包的不只是程式，還包含執行環境。在現代部署裡，很多團隊不是上傳 .zip 或 .jar，而是 build Docker image

```bash
docker build -t my-api:1.0.0 .
docker push registry.example.com/my-api:1.0.0
```

這時候 artifact 就可能是 `registry.example.com/my-api:1.0.0`，也就是一個 container image

它裡面可能包含

- 應用程式
- runtime
- 環境依賴
- 啟動指令
- 檔案結構


![c](https://github.com/CHI-KEKE/pics/blob/main/CICD/Artifact/6_docker_image_layers.png?raw=true)



所以 Docker image 可以理解成更完整的 artifact，一般 zip artifact 比較像只包程式成品，而 Docker image 是連執行這個程式需要的環境也一起包好



![c](https://github.com/CHI-KEKE/pics/blob/main/CICD/Artifact/7_zip_vs_image.png?raw=true)


<!-- endtab -->



<!-- tab ASP.NET Core Web API 專案-->

有一個 ASP.NET Core Web API 專案

```bash
MyApi/
  Controllers/
  Services/
  Program.cs
  appsettings.json
  MyApi.csproj

```


```bash
## 根據 .csproj 還原 NuGet 套件
dotnet restore
## → 用 Release 模式編譯
## → 因為前面 restore 過了，所以不再 restore
dotnet build -c Release --no-restore

## → 產生正式部署用檔案
## → 因為前面 build 過了，所以不再 build
dotnet publish -c Release -o publish --no-build

## → 把 publish/ 壓成 app.tar.gz
## → 上傳成 CI/CD artifact
tar -czf app.tar.gz publish/
```


最後部署端拿到 app.tar.gz

解壓縮後得到

```bash
publish/
  MyApi.dll
  MyApi.deps.json
  MyApi.runtimeconfig.json
  appsettings.json
```


就可以用 `dotnet publish/MyApi.dll`，或由 systemd / IIS / Docker / Kubernetes 去啟動它



![c](https://github.com/CHI-KEKE/pics/blob/main/CICD/Artifact/8_publish_to_run.png?raw=true)


<!-- endtab -->

<!-- tab ASP.NET Core Web API 專案-->



![c](https://github.com/CHI-KEKE/pics/blob/main/CICD/Artifact/artifact.png?raw=true)



<!-- endtab -->


{% endtabs %}