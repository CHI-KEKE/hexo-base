---
title: 資料庫存 UTC 嗎
date: 2026-05-04 10:03:03
categories: Time
top_img: https://github.com/CHI-KEKE/pics/blob/main/Codesss/Time/clock-wolf.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Codesss/Time/clock-wolf.png?raw=true
tags:
    - 
toc:
toc_number:
comments :
---


{% tabs 資料庫存 UTC%}

<!-- tab 策略 : 資料存 UTC, 顯示才轉時區-->

資料庫存的是「世界上真正發生的那一刻」，畫面顯示才是「這個使用者看到的當地時間」


![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Time/time_save_db/1_moment_different_zone.png?raw=true)



假設有一個訂單建立時間：**2026-05-04 10:00:00**，若我們直接存「台灣時間 10:00」，問題是資料庫只看到：**2026-05-04 10:00:00**，它不知道這是台灣時間、日本時間、紐約時間，還是倫敦時間，因此實際我們會這樣設計


1. 使用者在台灣建立訂單，時間 2026-05-04 10:00:00 GMT+8
2. 轉成 UTC 2026-05-04 02:00:00 UTC
3. 資料庫存 created_at = 2026-05-04T02:00:00Z，其中 Z 代表 UTC
4. 美國使用者查看這筆訂單，假設他在紐約，當地時間是 UTC-4，資料庫拿出 2026-05-04T02:00:00Z
5. 前端轉成紐約時間 2026-05-03 22:00:00
6. 台灣使用者看到 2026-05-04 10:00，而紐約使用者看到 2026-05-03 22:00，兩個人看到的時間不同，但指向的是同一個真實事件發生時間



![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Time/time_save_db/2_time_change_flow.png?raw=true)



而會建議在前端轉，不是後端是因為後端通常不知道使用者真正想用哪個時區看時間，而前端是最接近使用者的，使用者可能人在台灣，但瀏覽器語系是英文，也可能人在日本，但帳號設定想看台灣時間，
還可能使用手機跨國旅行，另外後端太早把時間轉掉，後面的人就失去彈性，也容易重複轉換出錯，API 最好回傳標準 UTC 字串，讓前端明確知道這是一個可轉換的時間點



![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Time/time_save_db/3_why_frontend_change.png?raw=true)



<!-- endtab -->


<!-- tab 夏令時間 DST 的陷阱-->

夏令時間（DST, Daylight Saving Time）是在一年中的某段時間，把時鐘「人為往前調一小時」。人類把時鐘往前撥，讓作息看起來更早，藉此多用白天少開燈

有些國家的時間不是固定加幾小時，直接存本地時間很容易撞到不存在或重複的時間，例如美國有夏令時間。


例如，某一天凌晨 2 點可能直接跳到 3 點，代表 02:30 這個時間在當地根本不存在。

另一種情況是時間往回調，凌晨 1 點會出現兩次，所以如果只存 2026-11-01 01:30:00，這可能導致不知道它第一次 1:30，還是第二次 1:30



![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Time/time_save_db/4_dst_trap.png?raw=true)



## IANA 時區

IANA 時區不是單純的 UTC+8，而是一整套會變動的時間規則，例如

```bash
Asia/Taipei
America/New_York
Europe/Berlin
```

這些不是固定 offset，而是包含

- DST 規則
- 歷史變動
- 未來政策更新

所以不能只存 UTC-5

因為它無法表達夏天 UTC-4 OR 冬天 UTC-5

遇到這情況等於是需要存 09:00 + America/New_Yor，如此一來，系統會自動跟著 DST 調整



![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Time/time_save_db/5_iana_contain_different_rule.png?raw=true)


<!-- endtab -->


<!-- tab Unix Timestamp-->

Unix Timestamp（Unix 時間戳）是當我們只在乎「時間先後與計算」，不在乎「人類怎麼看時間」，就用 Unix timestamp，而 Unix timestamp = 從 1970-01-01 00:00:00 UTC 到現在過了幾秒


![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Time/time_save_db/6_unix_timestamp.png?raw=true)


## 登入過期時間

現在時間：1777896000，設定 token 1 小時後過期 expire_at = now + 3600，因此我們僅需驗證 `if (now > expire_at)` → 過期，完全不用管時區、DST、格式


## 排序資料


效能考量存成這樣

1777896000
1777899600
1777903200

直接 numeric sort，比字串或日期快很多


## 倒數計時

```JAVASCRIPT
const remain = expireAt - now;
```

可以直接算剩幾秒




使用時機是，只要是「機器邏輯」在用的時間，就該用 timestamp


- token 過期
- cache TTL
- log 記錄
- 事件排序
- 效能敏感系統


![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Time/time_save_db/7_why_timestamp_machine.png?raw=true)


<!-- endtab -->


<!-- tab summary-->


![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Time/time_save_db/8_table.png?raw=true)


![2](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Time/time_save_db/9_final.png?raw=true)



<!-- endtab -->


{% endtabs %}

