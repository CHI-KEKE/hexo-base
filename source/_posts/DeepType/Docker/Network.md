---
title: Network
date: 2026-04-01 10:31:05
categories: Docker
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
tags:
    - Docker
toc:
toc_number:
comments :
---


{% tabs Network %}

<!-- tab Docker 網路-->


```bash
docker network ls 
```

這個指令會請 Docker 把目前主機上的網路清單列出來


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/network/image-8.png)


畫面上通常會看到幾個欄位

`NETWORK ID`：這個網路的識別碼
`NAME`：網路名稱
`DRIVER`：這個網路使用哪種驅動方式
`SCOPE`：這個網路的作用範圍



假設有一個 web 容器和一個 mysql 容器，我們希望它們可以互相連線。這時可能先用 `docker network ls`，如果看到有一個自己建立的 app-network，那你就知道接下來應該把 web 和 mysql 都放進這個網路裡，這樣它們才可以透過容器名稱互相找到對方。如果你沒有先查清單，可能會發生兩個容器各自在不同網路裡，結果程式一直連不到資料庫，還以為是帳號密碼錯了，其實只是網路根本不通

```bash
docker network create app-network

docker run -d --name mysql --network app-network mysql
docker run -d --name web --network app-network my-web-image

```


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/network/1_docker_network_ls.png)



<!-- endtab -->




<!-- tab docker0-->

docker0 就是 Docker 在主機裡先搭好的一個虛擬網路接線板，container 接上來之後才有地方可以互相碰到

他是 Linux 上的一個 虛擬 bridge 裝置，bridge 可以先把它想成「網路橋接器」或「虛擬交換器」，是 Linux 在作業系統裡模擬出來的一個網路元件

Docker 建立 container 時，會把 container 的虛擬網卡接到這個 docker0 上。這樣多個 container 就像都插到同一台交換器，自然比較容易互通

可以先記成

docker0 在主機上
container 的網路會接進來
它負責把這些 container 串在一起



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/network/6_docker0.png)


<!-- endtab -->


<!-- tab subnet-->

subnet 就是一段可分配出去的 IP 範圍，讓同一張網路裡的設備各自拿一個地址

假設有一張內部網路，它總不能所有 container 都用同一個 IP，所以 Docker 會先準備一整段地址池，然後從裡面發號碼給每個 container，例如 `172.17.0.0/16` 這就是一個 subnet

172.17.x.x 這一大段地址屬於同一組網路範圍，Docker 會從裡面分配 IP 給 container

172.17.0.2
172.17.0.3
172.17.0.4

這些都可能是同一個 subnet 裡的位址。所以 subnet 把它理解成一個網路底下能用的地址範圍

<!-- endtab -->



<!-- tab Layer-2 區段-->

Layer-2 區段 可以想成同一個交換網路裡的成員，大家在資料鏈結層上是直接接在一起的。這個詞來自 OSI 模型裡的第 2 層，也就是資料鏈結層。這一層比較偏「設備怎麼在同一張網路上彼此轉送資料」

同一個 Layer-2 區段，表示它們像是接在同一台交換器下面
而不同 Layer-2 區段，表示它們中間通常要靠路由或其他機制才會互通

放到 Docker 這個場景裡，意思就是多個 container 都接到 docker0，docker0 幫它們形成一個共同的二層網路環境，所以它們彼此可以直接在這個區段內通訊，也就是同一個交換環境裡的範圍



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/network/7_subnet_layer2.png)




<!-- endtab -->


<!-- tab network stack-->

Container 啟動時，Docker 不只是開一個行程，它還會幫這個 container 建立一個獨立的 Network Namespace。

這個 Namespace 可以想成一個獨立的小型網路環境，在這個小環境裡，container 會看到自己的

- 網路介面
- IP 位址
- 路由表
- Port 使用狀態


所以 container 裡看到的網路，不是直接等於主機本身的網路，container 以為自己有一套完整的網路設定，但其實那是 Linux 幫它隔出來的一個獨立空間


而所謂的 network stack，就是這整套網路運作組合包含

- 網卡怎麼收發資料
- IP 怎麼配置
- 封包要往哪裡送
- 哪些 port 正在被程式監聽


因為每個 container 都有自己的 network stack，所以不同 container 可以各自都用 80 port、都設定自己的 IP、都維護自己的路由，不會直接互相衝突，但它們也不是完全憑空連，Docker 還要再用 bridge、veth、network 等機制，把這些獨立的網路空間接出去，這樣 container 才能和其他 container 或外部世界通訊


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/network/3_network_stack.png)


假設同時開了兩個網站 container

web-1
web-2

這兩個 container 裡的程式都在監聽 80 port。如果今天沒有 `Network Namespace` 這種隔離機制，那同一台主機上兩個服務都搶 80 port，馬上就撞了。但因為它們各自在自己的網路空間裡，所以會變成 web-1 的世界裡自己在用 80、web-2 的世界裡自己也在用 80，在它們各自的 container 內看，這完全成立。

真正要不要讓外部連進來，才交給 Docker 額外做 port mapping，例如：

```bash
docker run -d -p 8080:80 my-web
docker run -d -p 8081:80 my-web
```

主機的 8080 轉到第一個 container 的 80
主機的 8081 轉到第二個 container 的 80

就會很明顯感受到 container 內的 port 是各玩各的，主機對外怎麼接，是另一層設定



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/network/2_self_port.png)


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/network/9_port_mapping.png)


<!-- endtab -->



<!-- tab veth pair-->

veth pair 是一對成對出現的虛擬網路線，一頭接在 Container 裡，一頭接在宿主機或 bridge 上，專門負責把 Container 的網路送出去。Container 自己有網路空間沒用，重點是要有一條線把它接回外面的世界，veth pair 幹的就是這件事


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/network/5_veth_pair.png)


1. Docker 建立 Container 時，先幫它準備獨立的 Network Namespace，這樣 Container 會有自己的網路環境
2. 接著建立一對 veth pair，這一對是成雙成對的虛擬網路介面，可以把它想成一條網路線的兩端，其中一端放進 Container 的 Network Namespace 裡，這一端通常會變成 Container 裡看到的 eth0。而另一端留在宿主機這邊，它通常會被接到 docker0 這種 Linux bridge 上
3. 當 Container 送出封包時，資料會先從 Container 裡的 eth0 出去，進到這組 veth pair 的另一端。bridge 再決定怎麼轉送資料
4. 可能送去另一個 Container，也可能送去宿主機外部網路。反方向也一樣，外面的資料可以經過 bridge，再從 veth pair 的另一端進回 Container


可以把它想成

- `Container` 是一間房間
- `eth0` 是房間內的網路孔
- `veth pair` 是牆內那條真正的線，把單一 Container 接出來
- `docker0` 是走廊上的交換器，把很多 Container 的線接在一起


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/network/4_network_component_room.png)


<!-- endtab -->

<!-- tab Container 內查看網路介面-->


執行 `docker container exec -it alpine1 ip addr`觀察網路介面

![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/network/image-10.png)



lo：回環介面，IP 為 127.0.0.1/8，也就是 Container 自己跟自己通訊時會走的路
eth0：連到 docker0 的 veth，通常 IP 為 172.17.0.X/16，eth0 是 Container 裡最重要的網路介面，通常就是它對外連線時使用的介面


<!-- endtab -->

<!-- tab 在 Host 上查看整體網路結構-->

![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/network/image-11.png)

因為 Container 本身只是隔離出來的小世界，它要跟別的 Container 或外網講話，就一定要先有一條能接回 Host 網路的路，而 docker0 + veth + NAT 正是在幫它鋪這條路


1. Container 裡有自己的網卡 `eth0`，當 Container 要送出封包時，會先從自己看到的 `eth0` 出發，對 Container 來說，這就像它自己的正常網路介面
2. eth0 背後接的是一組 veth pair，veth 可以想成一條成對出現的虛擬網路線。一端放在 Container 內，通常可視為它的 `eth0`；另一端留在 Host 上
3. 封包從 Container 端穿到 Host 端，封包先進入 Container 那一側的 `veth-container`，接著會從另一頭 `veth-host` 冒出來。因為這兩端是配對的，所以資料會像走網路線一樣被送過去
4. Host 端的 veth 會接到 `docker0`，`docker0` 是 Docker 預設建立的 `Linux bridge`。它很像一台虛擬交換器，負責把多個 Container 的網路介面接在一起。如果目標是其他 Container，封包可以在 bridge 內部轉送，只要對方也接在同一個 docker0 上，封包就能直接在這個 bridge 裡被轉送過去。這就是同一台 Host 上多個 Container 能互相溝通的原因。
5. 如果目標是外部網路，封包會交給 Host 處理，當封包不是送往同一個 bridge 內的其他 Container，而是要去外網，Host 就會接手後續路由與轉送。


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/network/10_packet_out.png)



6. Host 透過 iptables 的 NAT 規則改寫來源 IP，Container 通常使用的是內部私有 IP，外部世界不認得也不能直接回傳。所以 Host 會做 NAT，把封包來源位址改成 Host 對外的 IP，這樣外部主機才知道回應要送回哪裡。
7. 改寫完成後，封包就從 Host 的實體網卡送出去，到達外部服務


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/network/11_nat.png)



8. 回程封包先回到 Host，外部主機看到的來源是 Host IP，所以回應會先送回 Host。Host 再依 NAT 紀錄把封包轉回正確的 Container
9.  Host 會根據連線追蹤資訊，知道這份回應本來是替哪個 Container 發出去的。於是它把封包反向送回 docker0，再經過對應的 veth，最後回到 Container 的 eth0


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/network/12_packet_in.png)



<!-- endtab -->




<!-- tab summary-->


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/network/13_arch_diagram.png)



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/network/Visualizing_Docker_Networks.png)



<!-- endtab -->


{% endtabs %}