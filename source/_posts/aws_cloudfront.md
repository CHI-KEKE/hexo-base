---
title: CloudFront
date: 2024-08-27 
categories: Others
top_img: https://i.imgur.com/ayE9tp1.png
cover : https://i.imgur.com/ayE9tp1.png
tags:
    - AWS
toc:
toc_number:
comments :
---

CloudFront 就是 AWS 在全球各地設置了許多 Server，稱為 edge locations，這些 Server 可能存了網站的副本內容（如圖片、影片、網頁...等）。當使用者訪問網站時，他們會從最近的邊緣位置獲取內容，台灣雖然不是 AZ (Availability Zones) 但也設有 edge location!

![Image](https://i.imgur.com/56shTJf.png)

![Image](https://i.imgur.com/Oy7gwTI.png)

# 幾個重要的名詞解釋

Origin：

所有的原始文件（如圖片、影片、網頁等）都存放在這裡
可以是 Amazon S3 存儲桶、EC2 伺服器，或者是由多台 EC2 伺服器組成的負載均衡系統


Edge Server：

這些就像是分布在世界各地的小型倉庫
它們儲存源站內容的副本
當用戶請求內容時，如果 Edge Server 有該內容，就直接提供
如果沒有，就會向 Origin 請求，然後儲存並提供給用戶


Distribution：

這是 CDN 服務的入口點。
就像是一個智能導航系統。
當用戶訪問網站時，這個系統會自動將他們引導到最近的 Edge Server


Web Distribution：

這是專門為網站設計的分發類型。
處理常見的網站文件，如圖片、CSS、JavaScript 等
也支持某些視頻流媒體格式。


# Cache

預設檔案 cache 在 edge 的時間為 24 hours (default expiration time)，超過就會過期，最短可以設定為 0 second，但沒有 maximum expiration time 可以設定快取在 edge location 的靜態資源，會有 TTL (Time to Live) 的屬性，不會永久存在，為了讓使用者可以很快取得最新的資料，可以執行 CDN 清理的工作或是讓快取失效 (invalidate)，但因為需要重新回源，因此會有額外費用產生

# 好處

從 origin service 傳出的資料變少了，成本下降 (CDN 傳輸單位成本比 EC2 instance 低)


# DNS 查詢可能造成延遲

CDN 需要進行 DNS 查詢來找到最近的邊緣服務器。
如果 DNS 查詢很慢，可能會影響整體性能。


# Query String 的影響

預設情況下，帶有 Query String 的 URL 請求會繞過 CDN 快取。
這會導致請求直接發送到 Origin，降低性能。
解決方法可以將 Query String 設置為 Cache Key 的一部分。

假設經營了一個線上服裝店，網址是 www.coolclothes.com。

產品頁面 URL：https://www.coolclothes.com/products/t-shirt
這個頁面會被 CloudFront 快取，提供快速訪問。

今天想顯示不同顏色的 T-shirt：

https://www.coolclothes.com/products/t-shirt?color=red
https://www.coolclothes.com/products/t-shirt?color=blue

預設情況下，CloudFront 會將這些 Request 視為不同的請求，不使用快取。
結果就是，每次請求都會回到 Origin，減慢了加載速度。

因此，我們可以將 'color' 這個 query string key 設置為 Cache Key 的一部分。現在，CloudFront 會分別 Cache 紅色和藍色 T-shirt 的頁面。

也就是說

https://www.example.com/product?color=red&size=large 和
https://www.example.com/product?color=red&size=small 會共享同一個快取。

但 https://www.example.com/product?color=blue&size=large 會有不同的快取。


參考:
https://aws.amazon.com/about-aws/whats-new/2016/08/announcing-query-string-whitelisting-for-amazon-cloudfront/
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/QueryStringParameters.html



# 特定請求類型的限制

PUT、POST、PATCH、OPTIONS、DELETE 等方法的請求會直接發送到 Origin。
動態請求也不會使用區域邊緣緩存。

使用 CloudFront 的 "Cache-Control" Header 來控制快取行為。
實施部分頁面快取，只動態生成頁面的某些部分。
使用 CloudFront 函數或 Lambda@Edge 進行一些輕量級的動態內容處理。

區分動態和靜態內容：

靜態內容：圖片、CSS、JavaScript 文件等。
動態內容：個性化頁面、實時數據、用戶特定資訊等。

# 提高 CDN 效能的方法

最直接的方法是延長快取的過期時間。
但要平衡快取時間和內容更新頻率。


# 收費



CloudFront 的收費模式主要包括以下幾個部分：

儲資料費用（如果適用）：

如果使用 Amazon S3 來存原始內容，會產生 S3 的存儲費用。
這不是 CloudFront 直接的費用，但與整體服務相關。


內容分發費用：

這是 CloudFront 的核心收費。
當 Edge Location（邊緣位置）將內容傳送給用戶時產生。
在賬單中顯示為 "region-DataTransfer-Out-Bytes"。
費用根據數據傳輸量和用戶所在地區而變化。


回源流量費用：

當 Edge Location 需要從原始伺服器（Origin）獲取新內容時產生。
在賬單中顯示為 "region-DataTransfer-Out-OBytes"。
這反映了 CDN 更新快取內容的成本。


可選功能的額外費用：

使用字段級加密（Field-Level Encryption）會產生額外費用。
這提高了敏感數據的安全性。
使用 Origin Shield 作為額外的緩存層也會增加費用。
這可以進一步提高性能並減少對原始伺服器的請求。


參考資料

官網 :　https://aws.amazon.com/tw/cloudfront/pricing/

圖解 AWS CloudFront 收費模式！ : https://www.leyun.cloud/cc-91


# 更新 Edge Locations 資訊的延遲問題


想像你是一家全球性的線上書店的網站管理員。你剛剛決定使用 AWS 的服務來提升網站性能。你在美國東部（Virginia）創建了一個 S3 bucket 來存電子書封面圖片，並且設置了 CloudFront 來分發這些圖片，提高全球用戶的訪問速度，你的網站在更新後，開始使用新的 CloudFront URL 來顯示書籍封面。

第一天的情況是，美國用戶Amy 在紐約瀏覽你的網站，一切正常，圖片顯示迅速，因為她離 S3 bucket所在的 Virginia 很近，資訊早早同步，而鏡頭轉到歐洲用戶　Ben 在倫敦訪問網站，發現有些書籍封面無法顯示。他的瀏覽器顯示 HTTP 307 錯誤，因為在歐洲的 AWS 系統還不知道你的新 S3 bucket 的確切位置。

24 小時後，全球用戶無論是 Amy、Ben，所有用戶都能順利瀏覽網站，圖片加載迅速，HTTP 307 錯誤完全消失。


這反應了　S3 bucket 的系統狀態的資訊已經同步到全球所有 AWS 區域。
CloudFront 可以正確地將請求 Routing 到最近的 Edge Locations，並從那裡獲取 S3 中的圖片。


# Lab

參考

30 天鐵人賽介紹 AWS 雲端世界 - 8:　CloudFront 與 建立檔案 CDN 服務
https://ithelp.ithome.com.tw/m/articles/10192080

Day20 X CDN
https://ithelp.ithome.com.tw/articles/10277764


# 精神能量分析

精神能量 : 🪔