---
title: Redis - 分散式架構（Sharding）
date: 2026-01-10 23:30:11
categories: 落葉下的存檔
top_img: https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Cache/CacheSql.png?raw=true
tags:
toc:
toc_number:
comments :
---

{% tabs Cache - 分散式架構（Sharding）%}


[redis](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/?utm_source=chatgpt.com)


<!-- tab 分散式架構-->

當整個城市只有一條主幹道，不管你是上班、回家、買菜，全部擠在同一條路

Sharding 是什麼？他直接多鋪幾條平行道路，每條路有自己的流量上限、有自己的責任區，目的就是不要讓任何一條路成為整座城市的瓶頸，Redis Cluster 會把資料自動 sharding 到多個節點上，讓資料集和負載可以水平擴展




![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Sharding/1_cars.png)


<!-- endtab -->

<!-- tab Redis 分散式架構（Sharding）-->

分散式的 Redis 就是 Sharding 的概念，Sharding 的本質是為了讓「單台 Redis 永遠只處理自己撐得住的資料量」，不同的 Key/Value 會放到不同的 Redis 機器上，由系統（Redis Cluster）用「Hash」的方式計算放在哪裡


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Sharding/2_shard.png)


資料量與流量逐漸成長過程中，若把所有 key 都丟在同一台，記憶體、單執行緒會被撐滿，因此大流量站台通常架構上會搭配 Sharding ，讓每台 Redis 只負責自己那一部分 key，不共享資料、不互相鎖，Client 或 Redis Cluster 幫你處理 routing，你只管用 key，不用自己寫分配邏輯


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Sharding/3_hash_slots.png)


比如：

```bash
hash("user:1") % 3 = 0 ➜ 放 Redis A
hash("user:2") % 3 = 1 ➜ 放 Redis B
```

1. Redis Cluster 先把 key 算到某個 hash slot
2. 總共有 16,384 個 slots
3. 每個節點負責其中一部分 slots
4. 客戶端依 slot 路由到對應節點，或收到 redirect 後重送到正確節點





![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Sharding/4_redirect.png)


<!-- endtab -->

<!-- tab Hot Key 為什麼會讓 Sharding 失效-->

你就算有 10 台 Redis，只要所有人都打同一個 key，還是等於只有一台在撐

```bash
GET homepage:popularProducts
```


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Sharding/5_hot_key.png)



當所有人都打這個 key，結果同一個 Redis node、同一條執行緒、同一把 key 的請求佇列，即使每次 GET 很快，排隊一多就慢，如果還有 SET 更新、快取重建，則讀寫會被序列化，整個 key 卡住


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Sharding/6_hot_key_and_set_slow.png)


解 Hot Key 是「不要讓請求都撞在同一把 key 上」

```bash
GET homepage:popularProducts:groupA
GET homepage:popularProducts:groupB
```

把資料切成多份（分類 / 區塊 / 隨機），得以讓請求自然分散，如此一來，在 Redis Cluster，不同 key 可能落在不同 node，就算在同一 node ，每把 key 的排隊壓力變小


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Sharding/7_break_hotkeys.png)



<!-- endtab -->



<!-- tab summary-->


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/Sharding/9_final.png)


<!-- endtab -->



{% endtabs %}