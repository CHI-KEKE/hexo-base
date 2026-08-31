---
title: Process test
date: 2026-05-16 10:57:05
categories: Docker
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
tags:
    - Docker
    - 
toc:
toc_number:
comments :
---



{% tabs Process%}

<!-- tab Node Container-->

假設你的電腦完全沒有安裝 Node.js，但你想要測試 Node 20 的行為

以前可能會想「那我是不是要去官網下載 Node？還要處理版本管理？裝錯版本怎麼辦？」



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/test-process/2_node-in-container.png)


用 Docker 的話，可以直接這樣做

```bash
## -i :--interactive 啟動互動模式，保持標準輸入的開放
## -t : --tty 讓 Docker 分配一個虛擬終端機(pseudo-TTY)，並且綁定到容器的標準輸出上
## node:20 : 啟動這個 container 所依據的 image
## /bin/bash：容器啟動後要執行的命令
## bash 啟動就是一個 shell 程式 Bash 是一個「命令解譯器」。它會等待你的輸入，解析指令，執行指令，顯示結果。類似 powershell
docker container run -it node:20 /bin/bash
```

這時候我們已經不在我們原本的環境中了，而是「進入」了 container 中，可以在這裡執行 `node -v` 指令，會發現這個環境中已經安裝了 node，且版本是 20.5.1


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/test-process/node_env.png)



## ps aux

ps aux 的價值是讓你看清楚「現在這個環境到底有哪些程式正在活著」，學 Docker 不能只會啟動 container，也要知道 container 裡面到底跑了什麼


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/test-process/3_psaux_meaning.png)


```bash
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   4196  3636 pts/0    Ss   07:29   0:00 bin/bash
root           8  0.0  0.0   8120  4020 pts/0    R+   07:30   0:00 ps aux
```


這裡有個重點是 PID 1 -> /bin/bash，這代表 Bash 是這個 container 裡的第一個 process，也是主要 process。如果輸入
`exit`，Bash 結束了，PID 1 也結束了，container 通常就會停止。這就是為什麼很多 container 看起來「一跑就關掉」，因為它的主程式執行完了



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/test-process/bash-exit-container-finish.png)


## 測試 node 運作

建立一個測試檔案：

```bash
echo "console.log('Hello from container')" > app.js
node app.js

```

這時候是在 container 裡執行 Node 程式，不是在你原本電腦環境裡執行


<!-- endtab -->



<!-- tab ash-->


Alpine Linux 是一個非常輕量的 Linux 發行版，常被拿來做 Docker image 的基底，因為它體積很小，通常比 Ubuntu、Debian 這類 image 小很多


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/test-process/alpine-small.png)



Alpine Linux 使用 ash（Almquist shell）作為預設的 shell，它是一個輕量級的 shell，體積小但功能完整，了解 Alpine 使用 ash，是為了知道「你進到這個 container 後，真正接住你指令的是誰」，這會影響你能用哪些 shell 語法、哪些工具預設存在、以及為什麼有些 bash 寫法會突然不能跑



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/test-process/ash-is-translator.png)


```bash

## alpine：基於 Alpine Linux 的輕量級映像檔，通常只有 5-10 MB 大小
## ash：Alpine Shell，比 bash 更輕量但功能相似的 shell
docker run -it alpine ash


ps aux

## 過濾特定 Process
ps aux | grep ash

## 離開容器
ctrl + D
```

輸入的指令，都是先交給 ash 解讀，ash 會負責判斷你輸入的是什麼意思，然後幫你執行對應的程式，可以把 ash 想成跟 Alpine container 溝通的翻譯員

```bash
ls
pwd
echo hello
ps aux

```


<!-- endtab -->

<!-- tab summary-->


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/test-process/final.png)


<!-- endtab -->

{% endtabs %}