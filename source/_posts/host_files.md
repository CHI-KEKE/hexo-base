---
title: 關於 host file ( Windows )
date: 2024-07-26 15:35:05
categories: Others
top_img: https://i.imgur.com/3Td7Wnr.png
cover : https://i.imgur.com/3Td7Wnr.png
tags:
    - 
toc:
toc_number:
comments :
---

開始研究 host files 這東西的契機是，在看動畫瘋時一直被廣告煩得很不爽，然後就沒想說隨便 Google 看看有沒有什麼招，沒想到有人有一樣的想法而且還把心思動到 host file 上，覺得這個用法滿有意思的


# host file

網路就像一個巨大的城市，而每個網站都是這個城市中的一個建築物。在這個城市裡，每棟建築物都有一個獨特的地址（IP 地址）。但是，我們的頭腦無法記住地址，所以我們給每棟建築物起了名字（域名）。
現在，想像你的電腦會幫你連接到你想去的建築物，有兩種方式來幫你找到正確的地址：

1.打電話給城市的訊息中心（DNS Server）詢問地址。
2.查看一本特殊的私人電話簿（hosts 文件）。

hosts 文件的運作機制如下：

優先級：當你想訪問一個網站時，你的電腦首先會查看 hosts 文件（私人電話簿）。
格式：hosts 文件中的每一行都像是電話簿中的一個條目，包含一個 IP 地址和對應的域名。
覆蓋 DNS：如果 hosts 文件中有你要找的網站，電腦就會直接使用這個地址，而不會去詢問 DNS Server。

hosts 文件就像是一個可以自定義的、優先級很高的小型 DNS 系統。它給了用戶更多控制權。


# 測試

1.開啟 host file
	
方法一. 

	手動去預設路徑 : C:\Windows\System32\drivers\etc\hosts

方法二.

	1.WIN + R
	2.輸入 %WinDir%\System32\Drivers\Etc

方法三.

	1.在 PowerShell 輸入 notepad $PROFILE
	2.加入 function hosts { notepad c:\windows\system32\drivers\etc\hosts }
	3.存檔
	4.重新開啟 PowerShell
	5.輸入 hosts

2.編輯 host file

我們指定 www.google.com 給 127.0.0.1 (本機)，簡單說就是新增一條 127.0.0.1 www.google.com.tw 然後存檔


3.確認結果

接著在瀏覽器上，像平常一樣跑去 google.com.tw

結果可以看到行為不會像平常正常的 Google 網頁
![Image](https://i.imgur.com/rwSuzZg.png)

或者我們也可以用 ping 的方式快速確認

修改後
![Image](https://i.imgur.com/Pz2s7Pi.png)


4.如果沒有成功，可以嘗試清除 DNS Cache

CLI
```POWERSHELL

ipconfig /flushdns

```

如果還是沒成功而且是用 Chrome可以嘗試

chrome://net-internals/#dns

參考 : https://superuser.com/questions/723703/why-is-chromium-bypassing-etc-hosts-and-dnsmasq

# 常見用途

想像我們正為公司準備一個重要的網站升級。新版網站已經開發完成，但你還不能貿然將它發布到公眾可見的網絡上。畢竟，萬一有什麼小問題，可能會影響公司的形象。這時候，Hosts 文件就派上用場了


你有一個域名，比如說 "allen.com"。目前，這個域名指向的是舊版網站。你想在不影響現有用戶的情況下，先自己看看新版網站在真實域名下的表現。
正常情況下，當你在瀏覽器輸入 "allen.com" 時，你的電腦會向 DNS Server 詢問這個域名對應的 IP 地址。但是現在，你不想使用公開的 DNS 記錄，因為那樣所有人都會看到新網站。
這時，你可以打開 host file 這個文件，加一行：

12.34.56.78  allen.com

這裡的 "12.34.56.78" 是存放新網站的 Server IP 地址。
接著，當你在瀏覽器中輸入 "allen.com" 時，你的電腦不再詢問 DNS 服務器，而是直接查看 Hosts 文件。
結果就是，你看到的是新版網站，而世界上其他所有人看到的依然是舊版網站。你可以盡情地測試、調整，確保一切完美無缺。等到你百分之百滿意了，再正式更新 DNS 記錄，讓全世界看到新網站。


# 綁架網頁

其實剛才的實作也是綁架網頁的一種做法，把使用者常常瀏覽的公開網頁指向特定的 IP，尤其是沒事跑去下載對岸的軟體就有可能發生這種事情，什麼 360 衛士、快播Qvod、西瓜影音、搜狐影音...


# 封鎖網站

中華電信的色情守門員利用類似的作法來審查是否為黑名單網站，但他們是在 DNS 上動手腳


# 手機也有 host file 嗎?

有趣的是，手機也有 host file，但 iOS 需要透過越獄 (jailbreak) 才能去修改這個文件，不知道會不會因此影響更新以及保固，沒事還是不要亂試比較好(拿朋友的來試(?))

教學文可參考 : https://www.ptt.cc/bbs/iOS/M.1414987281.A.683.html

# 動畫瘋擋廣告實驗

教學 : https://home.gamer.com.tw/creationDetail.php?sn=5103036

我們來試試看 2024 是不是還適用

打開動畫瘋，選擇同意
![Image](https://i.imgur.com/RWlS4ye.png)

煩死人的彈出式廣告出現啦
![Image](https://i.imgur.com/UUkLngS.png)


經過實際測試後，加了一些東西

```TEXT

0.0.0.0 www.googletagmanager.com
0.0.0.0 www.google-analytics.com
#0.0.0.0 ajax.googleapis.com
0.0.0.0 connect.facebook.net
0.0.0.0 fundingchoicesmessages.google.com
0.0.0.0 adservice.google.com.tw
0.0.0.0 adservice.google.com
0.0.0.0 tpc.googlesyndication.com
0.0.0.0 pagead2.googlesyndication.com
0.0.0.0 ads.adaptv.advertising.com
0.0.0.0 s0.2mdn.net
0.0.0.0 ads.aralego.com
0.0.0.0 cdn.ampproject.org
0.0.0.0 googleads.g.doubleclick.net
0.0.0.0 adservice.google.comS
0.0.0.0 r1---sn-u5oxu-un5e.googlevideo.com
0.0.0.0 r1---sn-u5oxu-un5e.gvt1.com
0.0.0.0 r2---sn-u5oxu-un5e.googlevideo.com#127.0.0.1 r2---sn-u5oxu-un5e.gvt1.com
0.0.0.0 r3---sn-u5oxu-un5e.googlevideo.com
0.0.0.0 r3---sn-u5oxu-un5e.gvt1.com
0.0.0.0 redirector.gvt1.com
0.0.0.0 www.gstatic.com
0.0.0.0 safeframe.googlesyncdication.com
0.0.0.0 google_ads_iframe
0.0.0.0 bridge3.653.0_en.html
0.0.0.0 securepubads.g.doubleclick.net
0.0.0.0 pubads.g.doubleclick.net
#0.0.0.0 bahamut.akamaized.net
#0.0.0.0 i2.bahamut.com.tw

```

雖然還是要看 30 sec 廣告(畢竟還是要支持平台)，但至少不用每次都要再關掉一次彈出式視窗了! 


# 開源的 host files 管理介紹

網路上有人製作了防追蹤、擋廣告的集大成 host file : 

HOSTS
https://github.com/StevenBlack/hosts/
![Image](https://i.imgur.com/NTCgpiE.png)
看起來到現在都還有在維護

這個 Repo 合併多個 hosts 文件來源 & 分類，它提供了 30 幾種不同的 hosts 版本，包括基礎版與加入各種 Extensions（如假新聞、賭博、色情、社交媒體等）的版本，使用上也提供使用 Docker 容器來自動化維護 hosts 文件的方法，包括自動讀取白名單等等

另外，文件建議使用 0.0.0.0 而不是 127.0.0.1，因為

1. 當系統嘗試連接到 0.0.0.0 時，它會立即失敗
2. 不干擾可能在本地計算機上運行的 Web 服務器
3. 明確使用 0.0.0.0 表示不可訪問的 domain


不過要注意 Windows 系統對大型 hosts 文件支援上可能會有問題，如果 hosts 文件太大，可能會導致會連網站有問題，因為在 DNS Client 啟動時，它會讀取 hosts 文件並將其內容載到記憶體中。如果 hosts 文件很大 (比如包含成千上萬條記錄), 這個載入可能會占用大量資源，所以作者提供了兩種作法:

1. disable-dnscache-service-win.cmd 的腳本來禁用 DNS Cache service
2. 使用 Windows CLI Hosts Compress 來壓縮 hosts 文件，並且提供 PowerShell 版本的壓縮腳本


## 今日精神能量分析

精神能量 : 🤸‍♂️

昨天去小巨蛋溜冰，覺得這種技巧型運動，在練習怎麼控制自己身體的過程滿有趣的