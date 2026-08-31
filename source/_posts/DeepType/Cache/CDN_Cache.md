---
title: Content Delivery Network（CDN）
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

{% tabs Cache - CDN%}

<!-- tab 在勤美綠園道上-->

在一個大熱天的午後，你走在勤美綠園道上，突然想吃點冰冰涼涼的甜點。你走了 3 分鐘，到附近的全家便利商店，打開冰櫃看到一盒寫著「東京限定布丁」的商品，決定嚐嚐這份來自職人之國的綿密滋味

等等 為什麼我在台中，卻能在便利商店買到「東京限定」的布丁？

如果是從日本工廠直寄過來，理論上要等上好幾天、還可能被海關卡住。但今天我們能即時吃到，是因為「全家便利商店」早就跟日本合作，把布丁大量送到全台分店的冷藏庫裡，你走路 3 分鐘就能買到！

<!-- endtab -->

<!-- tab 🍮 用布丁理解 CDN：就近提供內容-->

CDN 預先把內容（如圖片、影片、靜態網頁）複製到遍佈世界各地的節點，當使用者要存取網站時，就能從最近的節點取得資料，而不用跨越半個地球回到原始主機

📦 布丁=靜態資源
🏪 全家分店=CDN 節點
🏭 日本原廠=你的主機（origin server）

![Image](https://i.imgur.com/oSqghEy.png)

<!-- endtab -->

<!-- tab 👾 技術面看 CDN Cache 怎麼運作？-->

CDN 快取的本質，是把「大家會重複看的東西」提前放在離使用者最近的地方，避免每個人都去敲一次主機，以我們維護的網站 example.com 為例，若有設定 CDN 快取行為，流程大致如下

- 使用者首次造訪網站，CDN 上尚無該資料 → 称為 Miss，轉向主機取得資料
- CDN 快取該份資料，並依照 `Cache-Control` 設定保存，例如 1 小時
- 接下來 1 小時內，其他使用者造訪時會直接從 CDN 取得資料 → 稱為 Hit

這樣一來，可以大幅減少主機壓力，加快載入速度，有上 CDN 但沒變快，問題可能出在 `Cache-Control: no-store` 或時間設太短，剛存就過期

![Image](https://i.imgur.com/AMyEQxg.png)
[點我看高清無碼](https://www.mermaidchart.com/raw/368c414b-cf08-49eb-8738-61cf8e99faea?theme=light&version=v0.1&format=svg)

<!-- endtab -->

<!-- tab CloudFront Distribution-->

CDN 實作可以選用 CloudFront，本質上是在選「跟 AWS 原生整合的一整套流量、快取與安全決策系統」，是「阻力最小、失誤最少」的選擇

在 AWS 建立一個 CloudFront Distribution（指定 S3 或 ALB 當 Origin）時，AWS 會自動幫你產生一個全球唯一的網域名稱，形式通常長這樣 https://d3k9abc123xyz.cloudfront.net，我們不能自己指定這個字串，是 AWS 派的

- d3k9abc123xyz：AWS 內部產生的唯一 ID
- .cloudfront.net：CloudFront 的全球入口網域

CloudFront 本身是靠 DNS 在做流量導向，當使用者請求 d3k9abc123xyz.cloudfront.net，實際發生的是

1. DNS 查詢 cloudfront.net
2. AWS 回傳「離使用者最近、狀況最好的 Edge 節點 IP」
3. 使用者直連該 Edge 節點

這就是 CDN 能「就近服務」的根本原因，但實務上很少用這個 cloudfront.net 網址，一定會做 Custom Domain（自訂網域），例如想要 https://img.example.com/毛吉.png

1. 先在 CloudFront 設定 Alternate Domain Name（CNAME），告訴 CloudFront：「之後 img.example.com 也是我」
2. 在 DNS（例如 Route 53）加一筆 CNAME / Alias，img.example.com → d3k9abc123xyz.cloudfront.net
3. 使用者實際流程變成 img.example.com → DNS 查詢 → 指到 CloudFront Distribution → 導向最近的 Edge

<!-- endtab -->

<!-- tab 用 S3 + CloudFront 建立 CDN 快取-->

實際操作時，只要透過 AWS 的

- S3（儲存桶） 上傳靜態資源
- CloudFront（CDN） 發佈出去

CloudFront 的本質是一個「可控的全球快取＋轉送層」，他可以幫 S3 擋下「大量重複又沒價值的請求」，讓 S3 只在真的需要時才出手

並設定 `Cache-Control`: `max-age=3600`，就能讓使用者在 1 小時內都從 CDN 節點快取圖片，而不會一直回源

[S3 + CloudFront 快取設定詳解](https://ithelp.ithome.com.tw/articles/10208628)

舉例來說，可以將圖片 `毛吉.png` 上傳到 S3，經由 CloudFront 發佈出去，並設定 `Cache-Control: max-age=3600`，這樣就能讓使用者在 1 小時內都從 CDN 拿到這張圖片，而非每次都跑一次 S3 載入，
連結教學包含設定資料來源 ( Origin Settings )、設定 cache 行為 ( Default Cache Behavior Settings )，教學滿完整的 !

<!-- endtab -->

<!-- tab CDN 的本質是什麼？-->

> 💡「用空間（💰）換時間」—— Trade Space for Time

我們犧牲了更多儲存空間（在世界各地備份），換來用戶就近取用、速度飛快

<!-- endtab -->

<!-- tab 適合的情境-->

如果將 HTML、JS、CSS、圖片等靜態資源預先快取至 CDN，即使主機掛掉，只要 CDN 尚在快取期限內，使用者仍能載入這些資源！適合用在官網、文件站、展示型網站、Landing Page 等不需登入的頁面，但如果是購物車、結帳、會員登入這類動態資料不能快取 → 主機掛了就沒救

有些進階 CDN（如 Cloudflare、Akamai）甚至提供 Always Online 模式，讓主機斷線時仍能顯示快取頁面或 Origin Shielding：多一層節點幫忙緩解主機壓力，不過說到底，它們也只是緩衝支援者，無法接手主機的「大腦邏輯」

<!-- endtab -->

<!-- tab 🔹頻寬-->

用戶每次都從主機載入大量圖片、影片，頻寬會爆炸。透過 CDN 快取，大部分流量都分攤在 CDN 上，有效節省原始主機的頻寬成本

<!-- endtab -->

<!-- tab 🔹攻擊防範（DoS / DDoS）-->

CDN 廣泛部署的特性，使它也能抵擋部分流量型攻擊。像 Cloudflare 就能在邊緣節點過濾異常請求，減少主機壓力。

<!-- endtab -->

<!-- tab 🤔 我的網站有需要使用 CDN 嗎?-->

如以上所提及，CDN 作為「緩衝 + 提速 + 節流」的輔助工具，融入一個完整的架構設計之中，還包含這些作用

- 多機熱備援（Active-Active）
- 負載平衡器（Load Balancer）
- API 的容錯機制（Failover API）
- 分散式資料庫或快照備援策略

我們可以把 CDN 想成是災難來臨前，在冰箱裡囤好糧食的好幫手，他可以為我們爭取寶貴時間，但「我只是架個部落格或 Side Project，有必要用到 CDN 嗎？」，如果網站流量小、訪客地區集中，單靠主機處理應該是沒問題的，甚至效能更穩定

CDN 能發揮巨大價值在於使用者來自不同國家或洲別、想提升網站載入速度、SEO 成效、宣傳活動期間高流量湧入、容易遭遇 DDoS 或異常流量干擾，所以說某個角度來看，CDN 就像買保險，平常或許沒差，但一旦出事，它能撐住你網站的第一道防線

<!-- endtab -->

<!-- tab Serverless CDN 與邊緣計算（Edge Computing）-->

現在的 CDN 已不再只是單純的「內容快取系統」，而正朝向「邊緣運算平台」的方向發展，像 Cloudflare Workers、Akamai EdgeWorkers 等技術，讓我們能在 CDN 節點上直接執行邏輯

這代表什麼？未來我們可以在 CDN 上處理驗證與權限邏輯、實作 A/B 測試等動態內容分流、快取可預測的 API 結果，降低主機負擔，這類應用，也就是所謂的 Edge Computing，它讓我們不只在「最近的地方取資料」，更能在「最近的地方計算邏輯」。換句話說，CDN 不只是冰箱，更像是一間設在你巷口的迷你雲端廚房

<!-- endtab -->

{% endtabs %}