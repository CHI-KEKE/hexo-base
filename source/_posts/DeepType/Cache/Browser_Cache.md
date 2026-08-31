---
title: 刷新之間，藏著多少重來與等待 - 網頁快取
date: 2024-08-24 16:10:11
categories: 落葉下的存檔
top_img: https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true
tags:
    - Cache
toc:
toc_number:
comments :
---

{% tabs Cache - 網頁快取%}

<!-- tab 清晨七點半-->

清晨七點半，你手捧著一杯還冒著熱氣的黑咖啡，筆電剛開機，PChome 的首頁已經靜靜等著。
今天，是傳說中的限量開搶日 —— PS5 Spider-Man 2 限定版主機，早上八點整正式開賣。
明明一個月前，你才下定決心要戒掉電動，結果看到廣告那一瞬間，理智直接下線：「反正只是收藏，不一定要打開來玩。」這麼說服自己之後，還特地設了三個鬧鐘，把商品頁加進書籤，信用卡也提前擺在桌上待命。

滑鼠懸停在「重新整理」按鈕上，指尖微微冒汗，內心默唸著：「這不算破戒，這叫做支持正版！」

當秒針跳到 7:59:59，你一口氣按下刷新，畫面瞬間閃動。那張帥氣的限量主機圖，早已從最近的新竹 CDN Edge Server 快取而來，商品庫存與折扣倒數，則剛剛從台北資料中心即時更新，因為你的瀏覽器在快取過期後，發送了 If-Modified-Since 的精確詢問。至於整個商品頁的 HTML 和 CSS，早就在昨天深夜「練習預購」的時候，被你的硬碟 Disk Cache 偷偷保存了下來，靜靜等著這個命運的早晨。

從源伺服器，到全球分散的 CDN 節點，再到本地硬碟與記憶體，資料在無數快取層之間跳躍、檢查、確認，只為了在你一瞬間的點擊裡，呈現出這場秒殺體驗。

這篇文章，我們會沿著資料流的路徑，拆解每一層 Cache 技術如何協同運作，支撐起這個幾乎無感卻極致流暢的網頁世界。

<!-- endtab -->

<!-- tab 🔹X-Cache 是「快取結果的回報」-->

![Image](https://i.imgur.com/D7nQSV8.png)

<br>

## 🔹X-Cache 是「快取結果的回報」

這個是**CDN 或反向代理（例如 CloudFront, Varnish）**等中介設備自己加上的 Response Header。

用來告訴你：「這個回應有沒有從快取來？」

**HIT**	有從快取取出，不用麻煩後端伺服器
**MISS**	沒有快取，乖乖去請求原始伺服器
**EXPIRED**	有快取但過期了，重新請求
**REFRESH_HIT**	有過期快取，但重新驗證後確認可以使用

這主要在 CDN / Proxy 層面，與 Cache-Control 搭配，可以用來檢查你的快取策略是否正確生效。

<!-- endtab -->

<!-- tab 🔹Cache-Control 是「告訴瀏覽器要怎麼快取」-->

這個 Header 是伺服器傳給瀏覽器（或 CDN、Proxy）的命令，用來控制「可以快取嗎？快取多久？快取在哪裡？」

**no-store**	完全不要快取（記憶體或磁碟都不准）
**no-cache**	可以存，但每次都要跟伺服器 revalidate
**max-age=3600**	可以快取 3600 秒（1 小時）
**public**	可由任何快取機制（CDN、proxy）存取
**private**	只允許私人快取（瀏覽器）儲存
**must-revalidate**	一旦過期，不能使用舊的快取，一定要驗證


![Image](https://i.imgur.com/Sc80Zff.png)
[點我看高清無碼](https://www.mermaidchart.com/raw/e484994e-9d2b-4922-b51a-f5da7d650f51?theme=light&version=v0.1&format=svg)

<!-- endtab -->

<!-- tab ❔ 資源 Cache 在哪裡-->

![Image](https://i.imgur.com/Q1JF41d.png)

## 🔹Memory Cache（記憶體快取）

存在於瀏覽器的 RAM 記憶體中。壽命：非常短暫，只活在目前這一個分頁或 Session 中。
例如：你打開一個網頁，圖片載入後存到 memory cache；只要不重整、也不關掉這個分頁，再點擊其他同頁的功能，就會從 memory cache 取出，不必重新請求。
適合存快速要取用、但不必永久保留的東西，像是：圖片、CSS、JS、AJAX 回應資料


**存取最快（因為在記憶體），一關掉分頁或刷新，馬上消失**

<br>

## 🔹Disk Cache（磁碟快取）

位置：儲存在使用者裝置的硬碟（磁碟）中。壽命比 memory cache 長很多，即使關閉網頁或電腦，下次開啟還可能使用。
適合存靜態資源（圖片、CSS、JS、字型）、版本不常變的檔案

**速度比 memory cache 慢，但可跨 session 使用，可以設定過期時間（用 HTTP Cache-Control、ETag 等 header 控制）**

<br>

## 🔹Service Worker Cache（服務工作快取）

位置屬於瀏覽器的 Cache Storage（不是 memory，也不是 disk cache），由 Service Worker 控制。壽命：非常長，直到你程式主動清除為止。適合存離線網頁（offline first）、PWA（Progressive Web App）用來存整個 HTML 頁面或資料 API

**你可以自訂快取邏輯（像是「先抓快取、再抓網路」、「只抓網路失敗才抓快取」等），可控制快取的版本、更新機制、移除過期資源**

✅ 忽略這兩項快取的方式

- DevTools > Network > Disable Cache（用來測試時忽略這個快取）
- DevTools > Application > Clear Storage 可清空
- Ctrl + F5

<!-- endtab -->

<!-- tab 🕒 Expires：設定「絕對時間」-->

在 HTTP/1.0 時代，快取的控制主要靠 Expires Header，它會告訴瀏覽器：「這份資源在某個特定的時間點後就不再有效」

Expires: Wed, 15 May 2025 12:00:00 GMT
這種方式有幾個缺點：

- 依賴客戶端時間：若用戶裝置的時鐘不準，判斷會出錯。
- 不易更新內容：資源一旦設定好過期時間，在到期前很難更新。
- 不適用於動態資源：像即時資訊或價格，不該用固定時間控制。

<!-- endtab -->

<!-- tab 🕒 Cache-Control：設定「相對時間」-->

在 HTTP/1.1 中，Cache-Control 提供更彈性的控制方式，像是用 max-age 指定「從現在起多久內有效」，單位是秒：

Cache-Control: max-age=31536000，表示設定的是一年（31536000 秒），即是這個資源在未過期前，瀏覽器不會再次請求。若同時設定 Expires 與 Cache-Control，以 Cache-Control 為主。

<!-- endtab -->

<!-- tab 🕒 🔹快取過期後怎麼辦-->

資源過期後，不代表一定要整份重新下載。若伺服器端的資料根本沒變動，直接重新傳一份不僅浪費頻寬，也沒意義。因此，我們會採用「條件式快取（Conditional Request）」來判斷資料是否有更新。

## Last-Modified + If-Modified-Since

伺服器可以在 Response 裡帶上 Last-Modified：Wed, 15 May 2024 08:00:00 GMT，當快取過期後，瀏覽器會在下一次請求時夾帶：If-Modified-Since: Wed, 15 May 2024 08:00:00 GMT
接著，若伺服器判斷這段期間內沒有更新資料，則會回傳：HTTP/1.1 304 Not Modified 表示「快取可以繼續用，不用重新下載」。

⚠️ 缺點：若伺服器只是打開檔案又關掉，編輯時間也會變，會誤判成資料變更。

<br>

## ETag + If-None-Match

ETag 是伺服器產生的一段檔案唯一識別碼（常為 hash），例如：ETag: "abc123xyz"
下一次請求時，瀏覽器會帶上： If-None-Match: "abc123xyz"
當伺服器比較這段識別碼後：

- 若沒變動 → 回 304，繼續使用快取
- 若變動 → 回傳新的檔案內容

✅ ETag 更精準，適合用來比對是否變更內容，不只是檔案時間。

<br>

## 🔹如果我們「每次」都想要確認是否有新資料

**Cache-Control: no-cache**
這個設定不是「完全不快取」，而是指「每次都先詢問伺服器」，是否有更新。若無變更，依然可以使用舊的快取：

**Cache-Control: no-store**
這才是完全不留快取、不儲存任何資料的設定，適合應用於敏感資訊（例如：信用卡頁面）：


最後我們可以想像一家電商平台

- 商品圖片可設長時間快取（max-age 一年）
- 商品價格頁面則可設定較短時間，或使用 ETag/Last-Modified 機制
- 購物車或會員資訊頁面應設定 no-store，確保隱私與即時性

![Image](https://i.imgur.com/dag2Pez.png)
[點我看高清無碼](https://www.mermaidchart.com/raw/368c414b-cf08-49eb-8738-61cf8e99faea?theme=light&version=v0.1&format=svg)

<!-- endtab -->

{% endtabs %}