---
title: Redis - 
date: 2025-08-17 08:53:11
categories: 落葉下的存檔
top_img: https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true
tags:
toc:
toc_number:
comments :
---

![""](https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true)



## StackExchange.Redis

它是由 Stack Exchange 團隊（就是經營 Stack Overflow 的那群人）開發維護的
所以這個套件名字就叫做 StackExchange.Redis。

🧠 本質上，它就像是你（.NET 程式）跟 Redis 這個外部系統中間的一個翻譯官，透過這個翻譯官，對 Redis 下指令、讀寫資料、訂閱事件、操作快取。

🔧 StackExchange.Redis 幫我們做了哪些事？

- 建立連線（像 ConnectionMultiplexer.Connect(...)）
- 存資料（像 db.StringSet("key", "value")）
- 取資料（像 db.StringGet("key")）
- 訂閱/發布消息（Pub/Sub）
- 效能最佳化（自動管理連線、批次處理、壓縮）

## ConnectionMultiplexer

ConnectionMultiplexer 是一個管理 Redis 連線的「總管」，幫你處理所有連線、發送指令、接收資料、訂閱事件等等。它會幫你：

- 建立一條或多條 TCP 連線到 Redis 伺服器
- 管理這些連線（重試、斷線、重連）
- 在你發出 Redis 指令時，自動選最適合的通道送出
- 共用連線給不同的 .NET service 使用（不用每次都重連）

對 Redis 發出一個指令 SET key value，這個指令是要透過「網路」傳送給 Redis Server。這就像：

「你要把訊息裝進一台郵車，從你家送到 Redis 的倉庫」

而這台郵車的控制中心，就是 ConnectionMultiplexer。


Multiplexer 本意（多工器）是「把多個輸入合併到同一條輸出通道」

在這裡的意思就是一個 Redis 連線（TCP 連線）可以同時處理多個指令、多個請求。
你整個系統的不同功能、不同 service 都可以共用這個連線，不用每次連 Redis 都重新開一條新連線，更有效率、更省資源，這也是為什麼 ConnectionMultiplexer 要設計成「共用一次就好（singleton）」


本質上連線就與 SQL DB 有很大的不同

| 項目     | Redis (`ConnectionMultiplexer`) | EF Core / SQL         |
| ------ | ------------------------------- | --------------------- |
| 連線方式   | **長時間常駐連線**（常駐 TCP 連線）          | **短暫連線**（每次查詢才開連線）    |
| 幾時開連線？ | 啟動時開，整個應用共用                     | 每次操作才開（lazy connect）  |
| 幾時關連線？ | 應用程式結束、或手動 Dispose              | 查詢/儲存完畢後立即關閉          |
| 是否共用？  | ✅ 整個系統共用一個 multiplexer          | ❌ 每次 DbContext 獨立建立連線 |
| 效能特性   | 為極速設計，需要持久連線                    | 適合複雜查詢，不需長駐連線         |


Redis 是記憶體型「即時存取」服務，它預期你會不斷、快速、頻繁地跟它互動，所以設計就是讓你開一條連線一直保持連線，避免每次斷開再重連的開銷，所以使用 ConnectionMultiplexer 是「持久連線」，而且推薦全系統共用一個。

## IDatabase

IDatabase 是 StackExchange.Redis 提供的一個介面（interface），代表你要操作的 Redis 資料庫（Database）。

它的職責很簡單：

✅ 提供你一組方法，讓你可以對 Redis 做：

- 設定資料（Set）
- 取得資料（Get）
- 操作清單、集合、字典、過期時間、等等...


常看到這段程式碼：

```CSHARP
var db = connection.GetDatabase();
```

就是：

從 ConnectionMultiplexer 拿到一個你要操作的「Redis 資料庫代理人」— 也就是 IDatabase。


Redis 本身有支援多個 database（預設最多 16 個，從 0 到 15）。你可以把它想像成：

Redis 是一整個資料倉庫，裡面有 16 間資料室。每間資料室都有自己的資料。而 GetDatabase() 就是在指定你要去哪一間房間。

範例：

```CSHARP
var db0 = redis.GetDatabase(0);  // 第 0 間
var db5 = redis.GetDatabase(5);  // 第 5 間
```

但現實中，大多數人都只用第 0 間（預設），因為 Redis 的多 DB 功能其實是很基礎的功能，不支援 cross-db operations，也不適合複雜管理。


❓為什麼叫做「IDatabase」？

雖然它名字叫 IDatabase，但它不是代表像 SQL Database 那麼複雜的概念，而是一個簡化的、針對 Redis 所設計的資料操作介面。

它不是用來建表格、定 schema，而是：「我想跟 Redis 的某個區塊互動，幫我提供工具就好」