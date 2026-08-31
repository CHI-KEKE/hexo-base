---
title: 批次作業架構設計與替代方案
date: 2026-05-20 11:00:03
categories: 設計図鑑物語
top_img: https://github.com/CHI-KEKE/pics/blob/main/Codesss/Design_Pattern/batchupload/1_landing.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Codesss/Design_Pattern/batchupload/1_landing.png?raw=true
tags:
    - 
toc:
toc_number:
comments :
---


{% tabs Batchupload%}


<!-- tab 整體架構全貌-->

## 系統邊界與角色

BatchUpload 橫跨三個系統與四張 DB 表，各自負責不同的職責：

```
┌─────────────────────────────────────────────────────────────────────┐
│  （商店後台）                                                          │
│                                                                     │
│  ① 商店操作員上傳 Excel                                               │
│  ② 建立 BatchUpload 主記錄（Status: Init）                            │
│  ③ 發送主 MQ                                                    │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  MQ 訊息
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  MQ 專案 ── BatchUploadProcess（主 MQ）                             │
│                                                                     │
│  ④  LoadData                                                        │
│     → 讀取 Excel（全量載入記憶體）                                     │
│     → 逐列解析，JSON 序列化                                           │
│     → 寫入 BatchUploadData（Status: Init）                           │
│     → BatchUpload 主檔 Status → ReadyToProcess                       │
│                                                                     │
│  ⑤  DoValidate                                                      │
│     → 逐一驗證每筆 BatchUploadData                                    │
│     → 驗證失敗 → BatchUploadData Status: ValidateFailed              │
│                  寫入 BatchUploadMessage（錯誤明細）                  │
│     → 驗證通過 → BatchUploadData Status: ReadyToProcess              │
│                                                                     │
│  ⑥  發動子 Task（發一個 MQ ，Job: BatchUploadTask）                 │
│     → BatchUpload 主檔 Status → InProcess                           │
└──────────────────────────┬──────────────────────────────────────────┘
                           │  NMQ 訊息（子 Task）
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│  MQ 專案 ── BatchUploadTaskProcess（子 Task）                         │
│                                                                     │
│  ⑦  撈 N 筆 ReadyToProcess 的 BatchUploadData（N = ProcessCount）    │
│                                                                     │
│  ⑧  逐筆執行 DoProcess（業務邏輯）                                    │
│     → 成功 → BatchUploadData Status: ProcessSuccess                 │
│     → 業務失敗 / Exception 達上限                                    │
│              → BatchUploadData Status: ProcessFailed                │
│              → 寫入 BatchUploadMessage（錯誤明細）                   │
│     → Exception 未達上限 → 更新 UpdatedDateTime（等待 retry）         │
│                                                                     │
│  ⑨  查 DB 是否還有 ReadyToProcess                                   │
│     ├─ 有 → re-queue 自己（等 delay 秒後再跑，回到步驟 ⑦）            │
│     └─ 無 → BatchUpload 主檔 Status → Finish                        │
│             記錄 BatchUploadAverageTime（供下次預估用）               │
└─────────────────────────────────────────────────────────────────────┘
```



![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Design_Pattern/batchupload/3-whole.png?raw=true)



## DB 表關係

```
BatchUpload（主檔，一批次一筆）
  │
  ├──< BatchUploadData（工作佇列，一個工作單元一筆）
  │       │
  │       └──< BatchUploadMessage（錯誤明細，驗證或處理失敗各一筆）
  │
  └──< BatchUploadAverageTime（平均耗時統計，供前端顯示預估時間）
```

| 表 | 建立時機 | 用途 |
|----|---------|------|
| BatchUpload | 後台觸發時 | 主記錄，控制整體狀態 |
| BatchUploadData | LoadData 階段 | 工作佇列，每個工作單元一筆 |
| BatchUploadMessage | DoValidate / DoProcess 失敗時 | 錯誤明細，供使用者下載 |
| BatchUploadAverageTime | Finish 時更新 | 歷史耗時統計，前端預估用 |



![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Design_Pattern/batchupload/2_keywords.png?raw=true)




## 狀態流轉全圖

```
BatchUpload 主檔 Status：

Init ──LoadData完──▶ ReadyToProcess ──發子Task──▶ InProcess ──全部完成──▶ Finish
                                                              │
                                                              └──發生錯誤──▶ Error

BatchUploadData Status：

Init ──LoadData寫入──▶ Init
     ──DoValidate──▶  ReadyToProcess  （通過驗證）
                  │   ValidateFailed   （驗證失敗）
                  │
                  └──DoProcess──▶  ProcessSuccess  （業務成功）
                                   ProcessFailed   （業務失敗 or Exception 達上限）
                                   ReadyToProcess  （Exception 未達上限，等待 retry）
```

<!-- endtab -->


<!-- tab 現況架構與設計動機-->

## 現況架構：後台觸發 → 主 MQ → 子 MQ 自迴圈

```
後台（SMS）
  → 建立 BatchUpload 主記錄
  → 發送主 NMQ 訊息

主 NMQ（BatchUploadProcess）
  → LoadData：解析 Excel，每個工作單元序列化成 JSON 寫入 BatchUploadData 表
  → DoValidate：逐一驗證，更新 StatusDef（ReadyToProcess / ValidateFailed）
  → 發動一個子 Task

子 Task（BatchUploadTaskProcess）
  → 每次撈 N 筆 ReadyToProcess 資料執行
  → 執行完 → 查 DB 還有沒有 ReadyToProcess
      ├─ 有 → re-queue 自己（等待 delay 秒後再跑）
      └─ 無 → 更新主檔狀態為 Finish
```

## 為什麼這樣設計？

核心限制來自 **NMQ 本身有執行時間上限**。如果一個 Task 試圖處理所有資料：

```
1000 筆 × 3 秒/筆 = 50 分鐘 → 超過 NMQ timeout → Task 失敗
且失敗後不知道跑到哪裡，無法續做
```

因此工作必須切分，而「主 NMQ 解析 + 子 Task 自迴圈」的選擇有以下邏輯：

1. **主 NMQ 執行時間可控**：只負責解析，跑完就結束
2. **子 Task 每次執行量固定**：ProcessCount 可調，不會超時
3. **StatusDef 天然記錄進度**：斷點續做不需要額外設計
4. **子 Task 完全無狀態**：每次從 DB 重新撈，崩潰重啟不影響
5. **不需要預先知道總筆數**：主 NMQ 結束前不用算好要建幾個子 Task

**最大的代價**是串行 + re-queue delay 累積，這是為了「簡單、可追蹤、容錯」所付出的時間成本



![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Design_Pattern/batchupload/4-limit-and-how.png?raw=true)



![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Design_Pattern/batchupload/5_problems.png?raw=true)




![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Design_Pattern/batchupload/6_solutions.png?raw=true)



<!-- endtab -->


<!-- tab MQ + Consumer 並行-->


**現況痛點：** 子 Task 串行執行，re-queue delay 逐批累積，大量資料時整體耗時極長。

```
Excel 上傳
  → 主 NMQ 解析每列
  → 每列直接發一條 MQ Message（如 RabbitMQ / Azure Service Bus / Kafka）
  → 多個 Consumer Instance 同時消費，天然並行
  → 不需要 re-queue，處理完就結束
```


## 優點

- **吞吐量大幅提升**：多個 Consumer 並行，理論上 N 倍速
- **無 re-queue delay**：不需要等待下一輪才能繼續
- **Consumer 可水平擴展**：壓力大時直接加 Consumer 數量



![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Design_Pattern/batchupload/7_parallel.png?raw=true)



## 缺點

- **失去 DB 狀態追蹤**：現況每列狀態都在 DB 可查，改用 MQ 後這個能力消失，需要另外設計追蹤機制
- **斷點續做複雜**：MQ Message 消費失敗後的重試邏輯需要自行設計（Dead Letter Queue 等）
- **MQ 基礎設施需要維運**：引入新的系統依賴
- **順序保證困難**：若業務需要同一使用者的資料按順序處理，MQ 並行會破壞順序


![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Design_Pattern/batchupload/8_cost_of_parelle.png?raw=true)


## 適合情境

資料量大、各筆獨立無依賴、對執行速度要求高、可以接受引入 MQ 基礎設施。


<!-- endtab -->


<!-- tab Streaming 分批讀取-->


**現況痛點：** LoadData 一次將整個 Excel 讀進記憶體，大檔案時有 OOM 風險，且主 NMQ 必須等全部解析完才能進行後續步驟。


## Chunk-based Streaming（串流分批讀取）

```
LoadData 改為 streaming 讀取
  → 每讀 500 列 → 立即 validate → 立即寫入 BatchUploadData
  → 主 NMQ 不需要等整個 Excel 解析完才繼續
  → 第一批 500 列寫完後就可以立刻發動子 Task，邊解析邊執行
```

### 優點

- **記憶體尖峰降低**：不再需要一次把整個 Excel 載入記憶體
- **主 NMQ 時間縮短**：解析與執行可以 pipeline 重疊
- **大檔案不再卡死**：分批讀取對檔案大小不敏感


![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Design_Pattern/batchupload/9_chunk_streaming.png?raw=true)


## 缺點

- **設計複雜度提高**：需要管理 stream 位置、分批寫入的 transaction
- **跨列驗證困難**：若驗證邏輯需要看所有資料（如全域查重複）就無法分批
- **子 Task 發動時機複雜**：需要決定什麼時候開始發子 Task，避免解析還沒完就以為資料已齊全


![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Design_Pattern/batchupload/10_cost_of_chunksteaming.png?raw=true)



## 適合情境

Excel 普遍較大（萬列以上），且驗證邏輯以單列為主、不需要跨列關聯。


<!-- endtab -->


<!-- tab 單一 Job 自迴圈-->


**現況痛點：** 主 Task / 子 Task 是兩種不同的概念，中間透過 `BatchUploadExecuteTaskTypeEnum` 做間接映射，需要較長的總耗時


## 做法

BatchUploadData 表的存在，本身就記錄了「LoadData 有沒有跑過」的事實：

```
BatchUploadData 沒有資料  →  LoadData 從未執行
BatchUploadData 有資料    →  LoadData 已完成，可直接進入 DoProcess
```

這讓「判斷目前在哪個階段」不需要額外的 Task 種類，只需要看主檔 Status 即可。


```
後台 → 直接發送 BlackListBatchUploadJob（不需要先有主 Task）

BlackListBatchUploadJob 每次執行：
  → 看主檔 Status
      ├─ Init      → LoadData（讀 Excel → 寫 BatchUploadData）
      │              → DoValidate
      │              → 主檔 Status 改成 InProcess
      └─ InProcess → 跳過 LoadData，直接 DoProcess

  → DoProcess（撈 N 筆 ReadyToProcess 執行）

  → 查 DB 是否還有 ReadyToProcess
      ├─ 有 → re-queue 自己（等 delay 秒後繼續）
      └─ 無 → 主檔 Status 改成 Finish
```


## 與現況的差異對照

| | 現況 | 單一 Job 自迴圈 |
|---|---|---|
| Task 種類 | 主 Task + 子 Task 兩種 | 只有一種 Job |
| 類型映射 | ExecuteTaskTypeEnum 間接對應 | Job class 本身就是類型 |
| 觸發方式 | SMS → 主 Task → 子 Task | SMS → xxxBatchUploadJob |
| 階段判斷 | 靠 Task 種類區分 | 靠主檔 Status 區分 |
| BatchUploadData 表 | 仍然需要 | 仍然需要（工作佇列職責不變） |


## 優點

- **概念更扁平**：只有一種 Job，不需要理解主/子的分工
- **命名直覺**：`BlackListBatchUploadJob`、`SalePageBatchUploadJob` 一看就知道是什麼
- **ExecuteTaskTypeEnum 可以廢除**：Job class 本身就代表批次類型
- **觸發路徑更短**：後台直接觸發目標 Job，不需要主 Task 再轉發




![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Design_Pattern/batchupload/11_single_job.png?raw=true)





### 缺點與需注意的邊界

- **LoadData 中途失敗的防禦**：若 LoadData 執行到一半崩潰，BatchUploadData 寫了部分資料但主檔 Status 還是 Init，下次 Job 執行時會重跑 LoadData 並清除舊資料，需要確保 LoadData 的冪等性（重跑不會產生重複資料）
- **共用邏輯需要 base class**：現況共用邏輯集中在 BatchUploadBaseService，提案中每個 Job 需繼承 base class，共用邏輯的維護方式需要重新設計
- **遷移成本**：現有所有批次類型都需要改寫


![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Design_Pattern/batchupload/12_single_job_cost.png?raw=true)



<!-- endtab -->



<!-- tab summary-->



![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Design_Pattern/batchupload/table.png?raw=true)


![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Design_Pattern/batchupload/final.png?raw=true)


<!-- endtab -->



{% endtabs %}