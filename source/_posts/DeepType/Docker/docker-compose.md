---
title: docker-compose
date: 2026-04-24 14:09:05
categories: Docker
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
tags:
    - Docker
toc:
toc_number:
comments :
---


{% tabs docker-compose%}

<!-- tab docker-compose-->

把「一堆需要一起啟動的容器」用一份設定檔講清楚，讓環境可以一鍵重現



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/docker-compose/1_why_need_dockercompose.png)



本質就是把多容器的操作流程「腳本化 + 結構化」


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/docker-compose/2_what_is_compose.png)



步驟大致上是

1. 先寫一個 docker-compose.yml : 定義有哪些服務（例如 web、db）、每個服務用哪個 image、要開哪些 port、環境變數是什麼
2. Compose 讀這個檔案 : 解析 services、networks、volumes 確認彼此的依賴關係
3. 建立資源 : 它會自動幫你建立 network（讓容器可以互相用名稱溝通）、建立 volume（資料持久化）



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/docker-compose/3_compose_step1_2.png)




4. 啟動多個 container : 按照設定一次把所有服務跑起來，幫我們處理啟動順序（例如 db 先起）


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/docker-compose/3_2_compose_step3_4.png)



5. 維持這個「整組環境」
   - up：全部啟動
   - down：全部關掉並清理
   - logs：一起看所有服務的 log



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/docker-compose/4_compose_up_down_logs.png)



<!-- endtab -->

<!-- tab 網站-->

今天有一個網站需要

- 前端（React）
- 後端（Node.js）
- 資料庫（MySQL）

如果不用 Compose 要手動做三次 docker run，因為 docker run 是在「開一台機器」，所以一個個 docker run 要記

- 誰先啟動
- IP 怎麼連
- port 怎麼對

就變得很容易亂掉，換一台電腦幾乎重來一次


如果用 Compose 就只要

```bash
docker compose up
```

三個服務一起起來，而且可以直接用 service name 溝通（例如 db:3306）


<!-- endtab -->


<!-- tab Compose 的限制-->


Compose 適合「單機環境」，不適合大規模分散式系統

- 不會自動擴展（scale 有限）
- 沒有高可用（HA）
- 不會跨機器調度

因此如果要多台機器自動 scaling、容器調度，我們就會用 Kubernetes



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/docker-compose/6_to_k8s.png)



<!-- endtab -->

<!-- tab Docker Compose vs Dockerfile-->

Dockerfile 是在「做出一台機器」，Compose 是在「把很多台機器排好一起運作」

Dockerfile 步驟

1. 從一個 base image 開始（例如 node、python）
2. 安裝套件（apt-get、npm install）
3. 複製程式碼進去
4. 設定啟動指令（CMD）


```bash
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
CMD ["npm", "start"]
```


結果會產出一個 image


Docker Compose 步驟

1. 定義多個 service（web、db、redis）
2. 指定每個 service 用哪個 image（可以是 Dockerfile build 出來的）
3. 設定網路、port、volume
4. 一次把全部 container 跑起來


```bash
services:
  web:
    build: .
    ports:
      - "3000:3000"

  db:
    image: mysql:8

  redis:
    image: redis:latest
```


結果是啟動一整組運行中的系統



![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/docker-compose/5_dockerfile_compose.png)



<!-- endtab -->


<!-- tab 手動連通不同 container-->

```bash

## Stag 1

## -d 讓 container 進入「背景執行模式」, 你會立刻拿到一個 container ID, container 繼續在背景跑（例如這裡的 Nginx 服務）, Docker 建立並啟動一個 container 終端機不會被這個 container 的輸出佔住
docker run -d -p8080:80 --name mynginx nginx 

 ## 查看
docker ps -a

## 進到已經在跑的 container 裡面互動操作
docker exec -it mynginx bash

## 查看 hosts, 172.17.0.2      fa19daaf33ac
cat /etc/hosts

## 跳出
exit 

## Stag 2

## 開一台 Alpine container，讓它在背景活著，而且保留互動終端機的能力
docker run -dit alpine

## 進入這個已經在跑的 container，並在裡面開一個 sh shell
docker exec -it b60ce6c54f2f8d6148346fb30eefd78c9c08a209e7581a9f810c44dbe30c922d sh

## apk 是 Alpine Linux 的套件管理工具（package manager), 在 Alpine 這個系統裡「安裝 curl 這個工具」
apk add curl

## 呼叫剛才的 nginx
curl 172.17.0.2
```



<!-- endtab -->



<!-- tab --link-->


```bash

docker ps
docker rm -f fa1
docker rm -f b60
docker run -d -p8080:80 --name myng nginx


## 讓這個 Alpine container 可以用 myng 這個名字找到 mynginx container。
docker run -dit --link myng:myng alpine

docker exec -it dd sh
apk add curl
curl myng


## 可以看到 Hots 有因為 --link 被添了一筆
cat /etc/hosts
exit
```


<!-- endtab -->


<!-- tab docker compose-->

```bash

wsl

## 建立資料夾
mkdir -p my-csharp-docker-demo/nginx
mkdir -p my-csharp-docker-demo/src/DemoApi
cd my-csharp-docker-demo

## 建立 docker-compose.yml
cat > docker-compose.yml <<'EOF'
services:
  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - api

  api:
    build:
      context: ./src/DemoApi
      dockerfile: Dockerfile
    environment:
      ASPNETCORE_ENVIRONMENT: Development
      ASPNETCORE_URLS: http://+:8080
      ConnectionStrings__DefaultConnection: Server=mysql;Port=3306;Database=demo_db;User=demo_user;Password=demo_password;
    depends_on:
      - mysql

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: 123456
      MYSQL_DATABASE: demo_db
      MYSQL_USER: demo_user
      MYSQL_PASSWORD: demo_password
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
EOF


## 建立 nginx/nginx.conf
cat > nginx/nginx.conf <<'EOF'
worker_processes 1;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile on;

    upstream demo_api {
        server api:8080;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://demo_api;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
EOF

## 建立 C# 專案檔 DemoApi.csproj
cat > src/DemoApi/DemoApi.csproj <<'EOF'
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

</Project>
EOF


## 建立 Program.cs
cat > src/DemoApi/Program.cs <<'EOF'
var builder = WebApplication.CreateBuilder(args);

var app = builder.Build();

app.MapGet("/", () =>
{
    return Results.Ok(new
    {
        Message = "Hello from ASP.NET Core Web API!",
        Environment = app.Environment.EnvironmentName,
        Time = DateTimeOffset.Now
    });
});

app.MapGet("/health", () =>
{
    return Results.Ok(new
    {
        Status = "Healthy",
        Service = "DemoApi"
    });
});

app.MapGet("/config", (IConfiguration configuration) =>
{
    var connectionString = configuration.GetConnectionString("DefaultConnection");

    return Results.Ok(new
    {
        DefaultConnection = connectionString
    });
});

app.Run();
EOF

## 建立 appsettings.json
cat > src/DemoApi/appsettings.json <<'EOF'
{
  "ConnectionStrings": {
    "DefaultConnection": ""
  }
}
EOF


## 建立 Dockerfile (heredoc)
cat > src/DemoApi/Dockerfile <<'EOF'
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

COPY DemoApi.csproj .
RUN dotnet restore

COPY . .
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app

COPY --from=build /app/publish .

EXPOSE 8080

ENTRYPOINT ["dotnet", "DemoApi.dll"]
EOF


## 確認結構
find . -maxdepth 3 -type f

## 啟動
docker compose up -d --build

## 查看 container
docker compose ps

## 測試 API
curl http://localhost:8080


## 看 log
docker compose logs -f
docker compose logs -f api

##　停掉
docker compose down ## 關閉並刪除這個 compose 專案建立的資源
docker compose down -v ## 會刪 MySQL 資料

## 砍 image
docker compose down --rmi all


## 清沒在用的東西
docker system prune
```



<!-- endtab -->



<!-- tab summary-->


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/docker-compose/docker-compose.png)


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/docker-compose/final.png)


<!-- endtab -->



{% endtabs %}