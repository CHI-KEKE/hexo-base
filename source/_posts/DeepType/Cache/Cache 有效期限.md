---
title: Time To Live
date: 2026-03-14 14:02:03
categories: 落葉下的存檔
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/TTL/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/TTL/1_landing.png
tags:
    - 
toc:
toc_number:
comments :
---

{% tabs Time To Live%}


<!-- tab Cache Time-->

TTL 的本質是拿「資料新鮮度」去交換「系統速度、成本和穩定性」


![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/TTL/2_tradeoff.png)


<!-- endtab -->

<!-- tab 這筆資料變動快不快-->

第一步先看資料本身的更新頻率


- 商品名稱、分類這種通常很少改
- 庫存、價格、優惠活動可能常常變
- 使用者個人首頁資料可能又跟行為密切相關

TTL 本質上是根據資料變動速度來決定壽命。如果資料一天才改一次，你卻設 5 秒，快取幾乎沒價值，還會一直回源查資料庫。反過來，如果資料每分鐘都會變，你卻設 1 小時，使用者看到舊資料的機率就很高



![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/TTL/3_how_frequent_is_dataupdate.png)


<!-- endtab -->


<!-- tab 資料舊一點能不能接受-->

不是所有資料都需要絕對即時。例如


- 最新新聞列表晚 30 秒更新，通常很多人能接受
- 商品庫存晚 10 分鐘更新，可能就會出事
- 交易餘額、付款狀態，如果延遲顯示就很危險

這一步重要，因為 TTL 設計真正要回答的是這份資料最久可以舊到什麼程度，業務還能接受。如果不先定義**容忍度**，你就會只靠感覺亂設 TTL。最後不是快取命中率太低，就是資料過期太久，兩邊都不討好。



![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/TTL/4_how_customer_feel.png)

<!-- endtab -->


<!-- tab 用「更新頻率 + 可接受延遲」一起決定 TTL-->

- 資料更新很少，容忍延遲高 → TTL 可以長
- 資料更新很頻繁，容忍延遲低 → TTL 要短

資料更新頻繁，但讀多寫少，且可接受短暫舊資料 → 可以考慮中等 TTL

例如

- 地區列表：1 天
- 商品詳細頁：10 分鐘
- 熱門排行榜：1 分鐘
- 庫存：10 秒到 30 秒，甚至不靠 TTL，改用主動失效
- 使用者權限：幾分鐘內，且權限變更時要主動清掉



![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/TTL/6_timesetting_example.png)


TTL 沒有標準答案，它永遠綁著業務場景


![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/TTL/5_decisiontable.png)





<!-- endtab -->


<!-- tab 「靠時間過期」還是「事件觸發失效」-->

比較穩的做法常常是 TTL 當保底，而資料更新事件來時主動刪 cache，例如商品價格被修改時，直接清掉商品快取；這樣下一次讀取就會拿到新資料，而不是傻傻等 TTL 到期


![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/TTL/7_ttl_is_just_base.png)


<!-- endtab -->


<!-- tab 進行中活動快取-->

根據 ttl 可能會預拉 30 min 後開始的活動

```csharp
var ongoingPromotions = await cacheManager.GetAsync(
    cacheKey,
    async () =>
    {
        var request = new GetPromotionEngineOngoingRequestEntity
        {
            Type = new List<PromotionEngineTypeDefEnum>
            {
                PromotionEngineTypeDefEnum.RewardReachPriceWithPoint2,
                PromotionEngineTypeDefEnum.RewardReachPriceWithRatePoint2,
                PromotionEngineTypeDefEnum.RewardReachPriceWithCoupon
            },
            StartDate = DateTime.Now.AddDays(-7), //// 為避免ATM等非即時付款 因時間差導致判斷失誤
            EndDate = DateTime.Now.AddMinutes(30), //// 避免快取期間的活動未被拉取
        };

        var promotionServiceClient = _serviceProvider.GetRequiredService<IPromotionServiceClient>();

        return await promotionServiceClient.GetPromotionsAsync(shopId, request);
    }, 5);
```

<!-- endtab -->


<!-- tab 大量 key 同時失效 -->

如果很多 key 都設一樣的 TTL，例如全部都 300 秒，可能在某個時間點一起失效，造成瞬間大量請求打回資料庫，這就是常見的快取雪崩。所以常會加一點隨機值 300 秒 ± 30 秒、600 秒 + random(0~120)
這樣做是為了把過期時間打散。不這樣做的話，系統平常很順，某個整點或尖峰時段卻突然被回源流量打爆，問題會很難查


![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/TTL/8_avalanchepng.png)



<!-- endtab -->

<!-- tab 監控回頭調整，不用一次定生死-->

TTL 設計不是設定完就結束，還要看

- cache hit rate 高不高
- 回源流量有沒有太大
- 使用者有沒有看到過舊資料資料庫
- 壓力有沒有下降
- 某些 key 是否特別容易失效後打爆後端



![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Cache/TTL/9_final.png)


<!-- endtab -->


{% endtabs %}

