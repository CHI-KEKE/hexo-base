---
title: Socket
date: 2025-08-25 09:45:01
categories: 穿越雲層的信使
top_img: https://i.imgur.com/hZWviQz.png
cover : https://i.imgur.com/hZWviQz.png
tags:
    - Http
toc:
toc_number:
comments :
---

在這個時代，我們打下一行簡單的程式碼，資料便能穿越海底電纜、飛越路由器之間的節點，抵達遙遠的伺服器。這一切對我們來說只是「一次 API 呼叫」，卻在背後上演著一段浩瀚的旅程。

想像有一位無聲的信使，他從你的程式出發，手中捧著訊息，沿著電纜與協定鋪展的道路一路前行。途中他會經過城市的路由器、跨越國界的骨幹網路、穿梭雲端的防護牆，最終抵達遙遠的伺服器大門。完成使命後，他又沿著相同的路徑回返，把遠方的回應帶回到你眼前。

這位信使，就是 Socket。他是隱形的橋樑，是程式與世界的通道。在鍵盤與螢幕的另一端，他靜靜地，卻堅定地奔走於無形的網路之海。


![message](https://i.imgur.com/hZWviQz.png)

<br>

## 📨 Socket 是什麼？

Socket 可以理解為「程式」與「網路」之間的連接點（endpoint）。  
它讓你的程式能透過 **IP** 和 **Port** 與外界通訊，發送或接收資料。

舉例來說：  
- 當你寫了一個 **Web Server**，你會讓它透過一個 Socket 在 **80 port** 上「監聽」(listen)。  
- 瀏覽器想連到你的網站時，會建立一個 client socket，去「連線」你的 server socket。  

而 Port 是區分應用程式（Process）的編號，在網路世界，IP address 可以定位到「哪台機器」。
但一台機器可能同時有很多程式在跑（例如：Web Server、Database、Chat App…）。所以需要 Port 來分辨「要把資料交給哪個程式」。

Socket 是「通訊的端點」，它的身份由 (IP, Port, Protocol) 唯一決定。

例如：

192.168.1.10:80/TCP → 你的 Web Server
192.168.1.10:443/TCP → 你的 HTTPS Server

當程式建立 Socket 並綁定到某個 Port，OS 才知道「當這個 Port 有封包進來，要交給哪個 Socket（也就是哪個程式）」。

<br>
<br>

##　📨 Socket 在 API 系統的作用是甚麼

讓我們一層層剝洋蔥式的理解 Socket 的職責是甚麼

<br>

### 🔹 第 1 層

Socket 是程式語言中一個「可以讓你傳送/接收網路資料」的物件。在程式碼中，我們會寫：

Socket s = new Socket(...);


這樣我們就拿到了一個「Socket 實體」，然後我們可以用它 .Connect()、.Send()、.Receive()。但這只是表層：這個物件本身其實只是呼叫作業系統提供的功能。

<br>

### 🔹 第 2 層：Socket 是 OS 提供的「通訊機制」

本質上，Socket 是作業系統（OS）提供的一種抽象出來的溝通方式。它讓一個應用程式（例如 .NET 程式）可以透過「網路卡」去發資料。可以這樣想：Socket = 程式 ↔ 作業系統 ↔ 網路卡 ↔ 網路世界

<br>

### 🔹 第 3 層：Socket 就是一個「被記錄在 OS 裡的通訊狀態」

在作業系統裡，其實會幫你 開一塊記憶體空間，儲存這個 Socket 的相關狀態，例如：

- 對方 IP 和 Port 是誰
- 本機的 IP 和 Port 是誰
- 目前這個通訊是不是已連線
- Buffer 裡收到的資料
- 有沒有發送錯誤、封包遺失…

這塊記憶體就是 Socket 在系統層級的「本體」，我們透過 Socket 這個物件，只是在操作這塊系統資源而已。

<br>

### 🔹 第 4 層

Socket 是作業系統內部 實作 TCP 或 UDP 協定的一個介面點（Interface）
網路傳輸其實是由底層的「協定」在操作（例如 TCP、UDP）。這些協定其實超級複雜，要處理：

- 封包的拆解和重組
- 順序處理
- 重傳機制
- 錯誤檢查
- 握手（TCP 的三次握手）
- 對方的確認訊息…

這些東西我們人類程式設計師如果每次都要自己寫，根本寫不完！所以作業系統幫我們包裝好這一切，把這些底層細節都藏在一個「Socket API」的介面裡，我們透過這個介面就可以簡單的做網路通訊。



因此，Socket 的本質是，作業系統中，為一段網路通訊所開的一個「溝通通道」與其狀態管理。

- 它存在於 OS 的核心層
- 它封裝了網路協定的實作（如 TCP）
- 它提供 API 讓應用程式能「像寫檔案一樣」收送資料（.Send()、.Receive()）

<br>
<br>

## 📨 C# HttpClient 發送 Request 到 Stripe 的旅程

### 程式碼

```CSHARP
var httpClient = new HttpClient();
var response = await httpClient.GetAsync("https://api.stripe.com/v1/charges");
```

當你呼叫 GetAsync() 時，HttpClient 做了幾件事：

- 解析網址 https://api.stripe.com，查 DNS 取得 IP，例如 104.18.11.84，
- 透過內部的 SocketsHttpHandler 來建立連線（如果還沒連過），決定這是 HTTPS，所以要走 TLS 加密協定

<br>

### 建立一個 TCP Socket + TLS

.NET 的內部是透過 Socket API 跟作業系統溝通：
```CSHARP
Socket socket = new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp);
socket.Connect(stripeIP, port 443); // HTTPS 預設 port
```

建立一個 TCP socket（也就是一個連線的「端點」）

- TCP 會開始三次握手（Three-way Handshake）：
- SYN → SYN-ACK → ACK
- 然後在 TCP 上建立 TLS 連線（加密的溝通管道）

<br>

### 作業系統封包處理 → 網卡傳送

電腦會將資料封裝成：

- TCP Segment → IP Packet → Ethernet Frame
- 加上 MAC、IP Header、Port、Checksum、Sequence…
- 送往「預設閘道」 → 你的家用路由器

<br>

### 資料開始穿越網路（跳躍節點）

這部分稱為：「網際網路傳輸」，會經過許多跳點（Hop）：

- 你家路由器
- ISP（中華電信、遠傳...）
- 國際骨幹網路（Backbone routers）
- Cloudflare（Stripe 用它的 CDN）
- Stripe 的 Load Balancer
- Stripe 的後端 API 伺服器

中間這些路由器會根據 目的 IP 把資料「轉送」給下一個節點，直到到達 Stripe 的伺服器。

你的程式
   ↓
.NET 的 HttpClient
   ↓
Socket (開 TCP 連線 + TLS)
   ↓
作業系統封包
   ↓
網路卡送出 → 路由器 → ISP → 全球網路 → Stripe
                                           ↓
                                 Stripe Server 接收與處理
                                           ↓
                         回傳 Response（同路徑回來）
                                           ↓
你的程式收到 response → 繼續邏輯

<br>
<br>

## 📨 結語

每一次呼叫 API，看似只是 .GetAsync() 的一行程式，其實背後是一場跨越數千公里的旅程。
Socket 就像一位無聲的信使，在雲層、電纜與協定之間奔走，將你的訊息送往遠方伺服器，再帶著回應回到你手中。或許下次當你敲下一行簡單的程式碼時，能想起這位默默穿越世界的旅人，他替你連結起程式與宇宙，替你在看不見的黑暗裡，點亮了一條光的路徑。