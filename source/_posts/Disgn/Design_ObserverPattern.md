---
title: 考到除草證照 5 級 !! 吉伊家長含淚喝采
date: 2025-08-15 08:33:00
categories: 設計図鑑物語
top_img: https://i.imgur.com/LF7L1Ws.png
cover : https://i.imgur.com/LF7L1Ws.png
tags:
    - Delegate x Observer Pattern
toc:
toc_number:
comments :
---

![design](https://i.imgur.com/LF7L1Ws.png)

那天早上，廣播突然響起 ——「各位家長請注意！吉伊寶寶順利考取除草證照 5 級！」
不到三秒，家長們已經刷了滿滿的貼圖，含淚喝采

這則喜訊，像是一場同步播放的快閃演出，正在教室自習的悄悄看了手機，忍不住笑到被同桌戳了一下；咖啡廳裡打工的，拉花時多畫了一片草葉致敬；剛進辦公室的設計師，立刻在筆記本上畫了「五級除草王」的手繪海報；外送路上的長，順路把好消息送到各個巷口；還有剛下班的夜班便利商店店員，抱著零食袋邊走邊笑說：「值得熬夜等這條消息！」

如果你覺得這畫面很熟悉，那是因為它就像程式世界中的「觀察者模式（Observer Pattern）」—— 一個事件發生，所有「訂閱」它的人會同時收到通知，不用一個個去問。

<br/>

<font color=#D3D3D3 style="font-size: 20px;">&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;
同一個消息，在不同的心裡綻放成各自的顏色。
</font>

<br/>

如果你覺得這畫面很熟悉，那是因為就是程式世界中的 **觀察者模式（Observer Pattern）** ——  當一個事件發生，所有訂閱它的人會同時收到通知，不用一個個去問。

<br>

## 🪵 廣播器


### 情境

想像有一台「家長廣播器」，只要一播送，所有家長都會同時聽到並作出反應。  

<br>

### 技術要點
- 使用 `Action<string>` 表示一個接收訊息的方法  
- `+=` 訂閱事件，`-=` 退訂事件  
- 事件觸發時，所有訂閱者的方法都會被依序呼叫  

```CSHARP
void Main()
{
    var pub = new ParentBroadcaster();
    var sub1 = new ParentSubscriber("學生家長");
    var sub2 = new ParentSubscriber("咖啡師家長");
    var sub3 = new ParentSubscriber("夜班店員家長");

    // 家長訂閱喜訊廣播
    pub.onAnnouncement += sub1.ReceiveAnnouncement;
    pub.onAnnouncement += sub2.ReceiveAnnouncement;
    pub.onAnnouncement += sub3.ReceiveAnnouncement;

    // 發送喜訊
    pub.Announce("吉伊考到除草證照 5 級！！");
}

public class ParentBroadcaster
{
    public Action<string> onAnnouncement;

    public void Announce(string message)
    {
        Console.WriteLine($"📢 家長廣播器啟動：{message}");
        onAnnouncement?.Invoke(message);
    }
}

public class ParentSubscriber
{
    private readonly string role;
    public ParentSubscriber(string roleName) => role = roleName;

    public void ReceiveAnnouncement(string message)
    {
        Console.WriteLine($"[{role}] 收到喜訊 → {message}");
    }
}
```

```PLAINTEXT
📢 家長廣播器啟動：吉伊考到除草證照 5 級！！
[學生家長] 收到喜訊 → 吉伊考到除草證照 5 級！！
[咖啡師家長] 收到喜訊 → 吉伊考到除草證照 5 級！！
[夜班店員家長] 收到喜訊 → 吉伊考到除草證照 5 級！！
```

<br>

### 📢 廣播器模式的問題：訊息送出去了，但後果誰知道？

本質問題，就像打開一個「廣播器」把喜訊喊出去，但：

- 發送者完全不知道誰有聽到、誰聽不懂、誰執行成功了什麼行為
- 有些訂閱者可能早就不在現場（物件不存在），還在執行 += 後殘留的方法
- 如果其中一個訂閱者 throw exception，可能會中斷整個通知流程


處理上可以使用 GetInvocationList() 拿到每個訂閱者，逐一 try/catch 包裝，避免一個人出錯全體沉沒，若訂閱者很多，考慮紀錄與追蹤每次事件觸發的處理情況（加上 log / 回傳結果），並且避免 event 為 public，改用封裝後對外只提供 Subscribe/Unsubscribe 的方法，集中管理

<br>
<br>

## 🪵 Func<T, bool> 做「投票/驗證」觀察者（有回傳值的觀察）

### 情境

訂單要出貨前，會詢問多個「驗證員」（Observers）。每位驗證員回傳 bool（通過/不通過）。Subject 聚合所有回覆，全部 true 才放行。

多播 delegate 的 Func 只會回傳最後一個結果，所以我們自己維護一群驗證員寶寶，逐一執行並彙總。

<br>

### 實作
```CSHARP
using System;
using System.Collections.Generic;
using System.Linq;

namespace ObserverWithFunc_Vote
{
    // 事件資料
    record Order(int Id, decimal Amount, string Destination);

    // Subject：出貨審核器
    class ShipmentApproval
    {
        private readonly List<Func<Order, bool>> _validators = new();

        public void Subscribe(Func<Order, bool> validator) => _validators.Add(validator);
        public void Unsubscribe(Func<Order, bool> validator) => _validators.Remove(validator);

        public bool Approve(Order order)
        {
            Console.WriteLine($"[Approval] 審核訂單 #{order.Id} 金額 {order.Amount} 目的地 {order.Destination}");
            // 逐一詢問觀察者
            foreach (var v in _validators)
            {
                var ok = v(order);
                Console.WriteLine($" - 驗證員回覆：{ok}");
                if (!ok) return false; // 只要一票否決就結束（提早停止，提高效能）
            }
            return true;
        }
    }

    class Program
    {
        static void Main()
        {
            var approval = new ShipmentApproval();

            // 訂閱：金額上限審核
            approval.Subscribe(o => o.Amount <= 5000m);
            // 訂閱：黑名單國家禁止
            approval.Subscribe(o => o.Destination != "NowhereLand");
            // 訂閱：奇數單號拒絕（只是示範）
            approval.Subscribe(o => o.Id % 2 == 0);

            var o1 = new Order(1002, 1200m, "Taipei");
            var o2 = new Order(1003, 800m, "NowhereLand");

            Console.WriteLine($"訂單 {o1.Id} 是否通過：{approval.Approve(o1)}");
            Console.WriteLine();
            Console.WriteLine($"訂單 {o2.Id} 是否通過：{approval.Approve(o2)}");
        }
    }
}

```

<br>

### 訂閱者越多，阻力越大


這類模式是「只要一票否決，整個事件就中止」，這種設計常見在：

- 較強流程控制的場景（例如：出貨前多重審核）
- 各個驗證點彼此不認識，卻會隱性影響最終決策

👉 問題出在：「越多人參與驗證，就越可能有一票否決，而且不容易知道是哪一票」

因此應讓每個驗證者可選擇中止或傳遞給下一位，為每個驗證者提供識別資訊，審核失敗時能追蹤是哪一位阻擋的

<br>
<br>

## 🪵 主題事件中心（Action<Event> + 條件過濾 Func<Event, bool>）

有一個通用的事件中心（Subject）。每個訂閱者不只提供「要執行的動作 Action<Event>」，還能附帶一個「過濾條件 Func<Event,bool>」，只有當事件符合條件才會觸發通知（像 Slack/Line 的關鍵字通知）。

### 實作
```CSHARP
using System;
using System.Collections.Generic;

namespace ObserverWithAction_Filter
{
    // 事件資料
    class AppEvent
    {
        public string Topic { get; init; } = "";
        public string Message { get; init; } = "";
        public int Severity { get; init; }  // 1~5
    }

    // 觀察者條目：包含過濾器與處理器
    class ObserverEntry
    {
        public Func<AppEvent, bool> Filter { get; }
        public Action<AppEvent> Handler { get; }

        public ObserverEntry(Func<AppEvent, bool> filter, Action<AppEvent> handler)
        {
            Filter = filter;
            Handler = handler;
        }
    }

    // Subject：事件中心
    class EventHub
    {
        private readonly List<ObserverEntry> _observers = new();

        public Action<AppEvent> PublishLogger; // 額外：對外顯示每次 Publish 的 hook（非必要）

        public IDisposable Subscribe(Func<AppEvent, bool> filter, Action<AppEvent> handler)
        {
            var entry = new ObserverEntry(filter, handler);
            _observers.Add(entry);
            // 回傳 IDisposable 以支援 using/退訂
            return new Unsubscriber(_observers, entry);
        }

        public void Publish(AppEvent e)
        {
            PublishLogger?.Invoke(e);
            foreach (var o in _observers)
            {
                if (o.Filter(e))
                    o.Handler(e);
            }
        }

        private class Unsubscriber : IDisposable
        {
            private readonly List<ObserverEntry> _list;
            private readonly ObserverEntry _entry;
            private bool _disposed;

            public Unsubscriber(List<ObserverEntry> list, ObserverEntry entry)
            { _list = list; _entry = entry; }

            public void Dispose()
            {
                if (_disposed) return;
                _list.Remove(_entry);
                _disposed = true;
            }
        }
    }

    class Program
    {
        static void Main()
        {
            var hub = new EventHub
            {
                PublishLogger = e => Console.WriteLine($"[Publish] Topic={e.Topic}, Sev={e.Severity}, Msg={e.Message}")
            };

            // 只想聽 error 類（Severity >= 4）
            var errorSub = hub.Subscribe(
                e => e.Severity >= 4,
                e => Console.WriteLine($"[ErrorListener] 收到高嚴重事件：{e.Message}"));

            // 只想聽 Topic=Order 的訊息
            using var orderSub = hub.Subscribe(
                e => e.Topic == "Order",
                e => Console.WriteLine($"[OrderListener] 訂單事件：{e.Message}"));

            // 發佈幾個事件
            hub.Publish(new AppEvent { Topic = "Order", Severity = 2, Message = "建立訂單 #9001" });
            hub.Publish(new AppEvent { Topic = "System", Severity = 5, Message = "記憶體不足" });
            hub.Publish(new AppEvent { Topic = "Order", Severity = 4, Message = "付款失敗" });

            // 退訂 error 監聽
            errorSub.Dispose();

            Console.WriteLine("-- 退訂 Error 監聽後 --");
            hub.Publish(new AppEvent { Topic = "System", Severity = 5, Message = "磁碟滿了" });
        }
    }
}
```

<br>

### 彈性太大，反而變成「不可預測的黑箱」

這種實作非常靈活，但也非常容易產生：

訂閱過多、條件錯亂、難以追蹤 → 誰處理什麼、處理幾次、處理順序都無法預測，無人訂閱的事件變成「沉默訊息」，若忘記 Dispose()，事件中心會長期保存訂閱者，導致記憶體洩漏（強參考）

訂閱時加入識別名稱或 Tag，幫助後續 log 與監控，並且加入診斷工具（例如列出目前有多少訂閱者、針對什麼事件），使用 WeakReference 或記得在 Dispose() 釋放訂閱
並且可加入 fallback 處理器（例如無人訂閱時預設執行某動作）

<br>
<br>

## 🪵 結語

Observer Pattern 就像吉伊家長的世界，一個事件的誕生，可以同時被許多人感知、響應，並產生各自的故事。

- 「所有人都聽到」→ 廣播器模式
- 「全部同意才能過」→ 投票 / 驗證模式
- 「針對性通知」→ 主題事件中心模式

用對模式，程式就能更優雅地傳遞訊息，而你，也能像吉伊家長一樣，第一時間收到最重要的那份喜訊。