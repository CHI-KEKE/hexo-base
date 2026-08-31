---
title: 架站 (一. Cloudfare體驗)
date: 2024-05-04 12:04:10
categories: Others
top_img: https://i.imgur.com/hyHuUUs.png
cover : https://i.imgur.com/hyHuUUs.png
tags:
    - Tools
toc:
toc_number:
comments :
---

想來玩玩看 Cloudfare 的 CDN，


## 將網站託管到 cloudflare

具體來說，這個託管就是將網站的 DNS 記錄 (A, CNAME...) 和流量管理轉移到 Cloudflare 平台上，Cloudflare 會去解析您的域名，將流量導向正確的 IP 地址，
如此一來，訪客的流量會經過 Cloudflare，這樣可以利用 Cloudflare 的 CDN、緩存、壓縮等功能來加速和優化網站。

1.首先要先去買網域，看是 GoDaddy還是啥的

Appwork 時期買的，趁續約金額變成 40 倍前再拿來做做實驗 (網站還順便被數落一番)
![Image](https://i.imgur.com/AgunUk7.png)


2.重新設定 Name Server 為 Cloudfare

![Image](https://i.imgur.com/DJaukqI.png)


## 確認 cloudflare DNS 設定

1.確認 Proxy Status => Proxied
![Image](https://i.imgur.com/XV2tOcB.png) 

如此一來，當有人透過 cofstyle.shop 瀏覽網站時，Cloudflare DNS 在解析這個 Domain 會回傳其 CDN Proxy 主機的 Anycast IP 而非我的主機真實 IP，Cloudflare 在全球各地放了很多台相同 IP 的主機，從美國、日本、台灣連這個 IP 會連上距離最近，速度最快的那一台，可快取的靜態檔案會直接從主機上傳回，動態內容才轉發到實際網站，CDN Proxy 主機接收到網站回應後轉送給瀏覽器。

好處有:

IP 隱藏：Cloudflare 會隱藏您的真實 IP 地址，只顯示 Cloudflare 的 Anycast IP。這可以有效防止直接針對您的伺服器 IP 的攻擊。
使用 Cloudfare 的 DDoS 保護：當網站受到攻擊時，Cloudflare 會使用他們的緩解攻擊流量機制。
CDN 加速 : Cloudflare 的全球 CDN 網絡會將您的網站內容 Cache 到全球的伺服器上(還是一直很好奇這具體怎麼做到的)，從而縮短訪(奧)客與伺服器的距離，提升加載速度。

## 設定 SSL/ TLS

![Image](https://i.imgur.com/NDuGhp0.png)

我這裡選 Flexible做測試，所以 Client 到 Proxy 走 HTTPS，Proxy 到網站走 HTTP

Cloudflare 原本會幫我們向 Let's Encrypt 申請一張 Wild Card 憑證 (本例為 *.dot-net.cloud)，但看起來今年 9 月後會有變卦

![Image](https://i.imgur.com/Jpd5PVC.png)







...To Be Continuted








## 今日精神能量分析

精神能量 : 
