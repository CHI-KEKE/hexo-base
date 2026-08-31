---
title: Image Layer
date: 2026-01-03 10:31:05
categories: Docker
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
tags:
    - Docker
toc:
toc_number:
comments :
---


{% tabs Image Layer%}

<!-- tab Image Layer-->

30 歲的那天我睡著了，隔天醒來，用「30 歲為止的 Image」，啟動了一個新的 Container，這個 Image 是過去經驗的凍結結果，你沒辦法在 Container 裡直接改 Image，把某一層人生經驗 hot fix，你只能用現在這份 Image 決定今天要跑出怎樣的一個實例



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/image-layer/1_30_years_old.png)



Image 就是那一疊已經整理好、不能再亂改的「環境變更記錄」，他是一份「如何把環境一步步疊起來的凍結結果」，而不是一個正在運作的東西，這些變更記錄加起來，組成一個「可執行的環境起點」



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/image-layer/2_what_is_image.png)



比較一下就是

`Image`「我『應該長怎樣』的說明書 + 已經準備好的環境層」
`Container`「照著這份說明書，真的跑起來的一次實例」



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/image-layer/3_image_container.png)



Image 結構是多個 layer，每一層都是一次檔案系統的變更（新增 / 修改 / 刪除），最終組合成一個完整檔案系統視圖，他可以被重複掛載、可以被重複使用，用來作為 container 的起始基準




![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/image-layer/4_layers.png)



<!-- endtab -->

<!-- tab 唯獨層、可寫層實驗-->

## 確認 Docker 環境

```bash
docker version
docker info
```

![docker_version](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Server/Devops/Docker/docker_version.png)
![docker_info](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Server/Devops/Docker/docker_info.png)

<br>

## 拉 node:20（會看到多個 layer）

```bash
docker pull node:20
```

![docker_pull_node20](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Server/Devops/Docker/docker_pull_node20.png)

看到「Pulling fs layer / Download complete / Extracting」那種多層輸出，這就是「node:20 不是只有單獨一層」，代表 image 是由多個唯讀 layer 疊起來的（不是單一檔案、也不是單一層）



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/image-layer/5_1_pull_node20.png)



## 開第一個 container（從 node:20 起, 用互動 shell）

Docker 會把 node:20 的唯讀 layers 拿來用，再額外加上一層 container 專屬的可寫 layer（這次的改動都會記在這層），接著進到 container 後，建立檔案
```bash
## 起一個 instance
docker run --name n20a -it node:20 bash

## 在 container 寫個檔案
echo "hello from container n20a" > /AAA.txt
ls -l /AAA.txt
cat /AAA.txt

## 順便看一下目前所在 container 的 hostname 更有「這是某一次實例」的感覺
hostname

## 離開 container，停止互動
exit
```


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/image-layer/5_2_write.png)



## 開第二個 container（同樣從 node:20 起）並驗證看不到 AAA.txt

```bash
## 使用相同 IMAGE 起第二個 instance
docker run --name n20b -it node:20 bash

## 在第二個 container 裡 列出 AAA
ls -l /AAA.txt
## 發現在第一個 container 做的檔案，不會回寫到 image，所以第二個 container 看不到! 因為 n20b 也會有自己的可寫 layer，但它是全新的、乾淨的

## 離開互動
exit
```

因為我們把 image 當「可重複使用的模版」，而 container 是「這次開機的暫存筆記本」，我們在筆記本寫的東西，不會回滲進模版，所以下次再拿模版開新筆記本，當然看不到上次寫的內容!



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/image-layer/5_3_start_another_nothing.png)


<!-- endtab -->

<!-- tab 形成新的 layer-->

把第一個 container commit 成新 image（形成新的 layer）

```bash
## docker container commit
docker container commit n20a node:20-updated

## 用新 image 起第三個 container，驗證 AAA.txt 出現了
docker run --name n20c -it node:20-updated bash
ls -l /AAA.txt
cat /AAA.txt

## 離開互動
exit
```

看得到證明 commit 成功，這個檔案已經不是 container 專屬，而是 image 的一部分，即使離開 container， image 還在，可重複使用



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/image-layer/5_4_commit.png)


<!-- endtab -->

<!-- tab 驗證「Docker 不會重複存兩份 node:20（layer 共用）」-->

## 看 image 的 layer 歷史

```bash
docker history node:20
docker history node:20-updated
```

![docker_history_node20](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Server/Devops/Docker/docker_history_node20.png)



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/image-layer/6_layer_history.png)



<!-- endtab -->


<!-- tab summary-->



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/image-layer/7_final.png)


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/image-layer/image-layer.png)


<!-- endtab -->



{% endtabs %}
