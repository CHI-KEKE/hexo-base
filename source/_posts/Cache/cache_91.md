


## Redis byteout

byteout（或稱「bytes_out」） 表示 Redis 伺服器 送出給客戶端的資料量，也就是「Redis 從伺服器傳出去的流量」。

客戶端發請求 → Redis 回應資料

這些回應的資料（字串、結果、查詢內容）加起來的總位元組數，就是 byteout

Redis 的運作核心在於「記憶體內存取」+「網路通訊」。
所有操作最終都透過 TCP 傳輸資料給客戶端。
因此：

bytes_in：代表接收到的資料量（客戶端 → Redis）

bytes_out：代表傳送出去的資料量（Redis → 客戶端）

這兩者都是 Redis 網路 I/O 層的關鍵監控指標，能幫你了解 Redis 的流量負載情況。

本質上，byteout 高通常意味著：

回傳資料太大（例如 GET 很大的 value、LRANGE 取太多元素）

客戶端請求頻繁（流量大、QPS 高）

資料被重複讀取（Cache 未命中、熱點 Key 被重複讀）

可能有惡意或異常行為（例如攻擊者持續 Dump 資料）


## 切緩


https://docs.google.com/spreadsheets/d/1GD8O4Gw0q77tcOzCvlYsZT3nGoPlMCkyuHS-DZjBI5Y/edit?gid=69393153#gid=69393153



## 監控

https://monitoring-dashboard.91app.io/d/jvzlt94Sk/e690b6-e8b3bc-e79ba3-e68ea7-tw-redis?orgId=2&refresh=30s&from=now-1h&to=now&timezone=Asia%2FTaipei&var-ENV=TW-Prod&var-AWS=kYZD-B7Vk&var-RedisCluster=tw-prod-cms-cache-2-001&var-RedisCluster=tw-prod-cms-cache-2-002&var-MWebCache=$__all&var-OutputCache=$__all&var-TempOutputCache=temp-output-cache-1-001&var-TempOutputCache=temp-output-cache-1-002&var-ReplWebRedis=$__all&var-TempReplWebRedis=$__all&var-CommercecloudCache=$__all

說明: 觀察到任一座的MemoryUsage超過70%，需清理資料
Redis清除皆須將參數貼至Slack，由TTL Review後才能執行


## 清除 cache

https://docs.google.com/document/d/12PltL6GVBiQ79SprPqkMWqVeRNb34p1igCehSmbNZIQ/edit?tab=t.0#heading=h.ikekx8yssddk



 Redis異常處理SOP
1. 發現人員優先至monitor頻道回報，並通知指揮官
2. 相關人員進入Meet會議室同步資訊(已在monitor頻道增設Meet連結書籤，以加速人員進入Meet時間)
3. 依據異常原因處置
異常處置：
  a. 觀察Elmah確認是單台MWeb機器無法連上或是全面無法連上 - 若是單台MWeb無法連上則請Infra Team將異常MWeb下線，處理後再恢復上線
  b. Node Failover(Node Status會顯示) - 約五分鐘內會自動恢復，恢復後確認Elmah是否有持續出現錯誤，若錯誤未停止則進行IIS回收
  c. Node網路異常 - 確認異常Node，若短時間內未恢復，請Infra Team新增一個Node進行替換
  d. Cluster異常 - 請Infra Team新增一座Redis後切換DNS，切換完畢若Elmah持續出現錯誤則進行IIS回收






## Redis 降級作業

https://91app.slack.com/archives/G06A3GDC7/p1762850138836419


## 「repl redis」是什麼？

repl 是 replication（複寫、主從同步） 的縮寫。
所以 repl redis 指的就是 Redis 的主從架構 (Master-Replica Setup)。

也就是說：

一台 Redis Master 負責寫入。

一台或多台 Replica Node（從節點）負責同步資料、或分擔讀取壓力。

你作業中提到：

「06:00 ~ 07:00 repl redis 移除分流」

意思是：
那個時間段要調整或移除 某個 Redis 複寫節點（Replica），可能是要：

減少不必要的分流節點（節省成本）；

或將部分流量導回主節點；

或結束臨時建立的分流測試架構




## 為什麼「output cache」也在 redis

「output cache」這種 Redis 通常用來存放：

頁面輸出快取（HTML / API 結果）；

或產出後端計算結果（報表、查詢結果等）。

這類資料：

存取頻率高（例如前台每個人開網頁都會 hit）；

需要快速輸出（直接給前端、CDN 或應用層）；

但不需要持久保存（失效可再生）。

Redis Repl 架構的本質

Redis 的複寫是「非同步」且「單向」的資料流：

Master 接收所有寫入；

Replica 透過 TCP stream 接收 RDB/AOF 同步資料；

Replica 可提供只讀查詢（常用於讀寫分離）。

你這份作業裡的「移除分流」與「降級」操作代表：

系統目前正在 縮減 Redis 資源 或 回收非必要 Replica 節點；

這些 Replica 可能原本用來：

分攤負載（前台流量高峰時期），

或作為 failover / 測試用途；

因流量下降或成本考量，就把這些 Replica 移除、改回單主節點或小機型。



## temp-output-cache 清除排程


https://91app.slack.com/archives/G06A3GDC7/p1762870968601109



## key 清除ci

http://ci5.91dev.tw:8080/view/Server/job/Prod.MobileWebMall.Redis%20-%20CircuteBreak%20Switch%20by%20Shop/


http://ci5.91dev.tw:8080/view/Server/job/Prod.MobileWebMall.Redis%20-%20CircuteBreak%20Switch%20by%20Shop/
P.S. 調整後，約五分鐘內生效，別慌 




*限速器設定值 Redis 關鍵字: ExceededTimesLimiterSettings

2024/11/29 確認可以撈取，請記得確認當下Redis Connection是否更新
使用方式: 請改App.config裡的ExecMode,ShopId, SkuIds




<!-- tab 購物車限速器-->



加入購物車限速器(BySKUId)
加入購物車限速器 By MemberOrUnloginId-SaleProductSKUId (全店設定)
參數說明:
Enabled: 限速器開關
MaxTimes: 單位時間內最高限速
CacheSeconds: 單位時間間隔秒數
LockedSeconds: 鎖定秒數 (待表一段觸發限速，須等待的時間)

Dashboard連結
** 需修改ShopId，單店設定需要有shopStaticSetting才可編輯參考PR

全店預設: Cache:Prod:WebAPI:CircuitBreakerService:ExceededTimesLimiterSettings-20200826:ShoppingCart-InsertItem-SaleProductSKUId-zh-TW


加入購物車限速器(ByShop)
加入購物車限速器 By ShopId (可by店設定，現為全店設定。若需by店設定須請 RD 協助設定 ShopStaticSettings)

Dashboard連結
** 需修改ShopId，單店設定需要有shopStaticSetting才可編輯參考PR
全店預設:  Cache:Prod:WebAPI:CircuitBreakerService:ExceededTimesLimiterSettings-20200826:ShoppingCart-InsertItem-ShopId-zh-TW


進入購物車P1
視情況調整redis參數
Cache:Prod:Shopping:CircuitBreakerService:ExceededTimesLimiterSettings-20230512--735268:ShoppingCart-CartsCreate-MemberId-zh-TW
參數說明:
Enabled: 限速器開關
MaxTimes: 單位時間內最高限速
CacheSeconds: 單位時間間隔秒數
LockedSeconds: 鎖定秒數 (待表一段觸發限速，須等待的時間)

<!-- endtab -->

<!-- tab 購物車預覽的功能-->


購物車PreView關閉(活動前)
關閉購物車預覽的功能

Cache:Prod:WebAPI:CircuitBreakerService:GetShoppingCartPreview-20200821:DisabledShopIds-zh-TW




<!-- endtab -->

<!-- tab 斷路器-->


Cache:Prod:WebAPI:CircuitBreakerService:ShoppingCartV2GetCount-20210220:DisabledShopIds-zh-TW
