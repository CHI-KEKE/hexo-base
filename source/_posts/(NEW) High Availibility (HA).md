



HA（High Availability，高可用性）


## HA 之前，先搞懂 Stateful vs Stateless

✅ Stateless component 是什麼？
像是：Application Server（AS）/ Nginx

特性是：不會保存任何「狀態」（state），也就是沒有 session、使用者資料、執行進度這些。

所以它可以橫向擴充（scale out）非常容易，想加幾台就加幾台，反正大家都是 stateless。

如何維持 HA（高可用）？

靠 heartbeat（心跳）機制判斷哪台死掉。

有問題就換台 server 接手即可。

✅ Stateful component 是什麼？
像是：PostgreSQL / Kafka / RabbitMQ

這類會保存狀態資料，例如資料庫內的使用者資料、訊息佇列的訊息等。

所以要維持 HA，比起 stateless 難很多，因為 state 不能亂丟、不能亂同步。

 二、Redis 是什麼？Stateless 嗎？
如果 Redis 只是做 caching（快取），就可以當作 stateless 看待（只是暫時性資料，掉了沒差）。

但如果 Redis 當作主資料來源，那就是 stateful，要好好維運，cache-miss 可能會出事。

📌 三、Stateless 的 HA 很容易做到
範例：我們架了幾個 nginx，只要判斷有沒有在跑、有沒有心跳訊號就好。

而且 nginx 本身還可以接上 monitoring 工具，萬一 IP 有異常（例如被改掉）也能警告。

所以：Stateless 非常容易做 HA。

📌 四、Stateful 就麻煩多了
比如 RDBMS（關聯式資料庫）：

需要 Master-Slave replication，甚至要引入 NoSQL 的架構來避開這些麻煩。

所以很多時候，我們會選擇讓系統的 component 盡量 stateless。

例如 PostgreSQL 很強，但如果可以不接觸它，就先別碰。



❗️當你設計高可用系統時，要先分類：哪些元件是 Stateless、哪些是 Stateful
Stateless 就像是跑腿的員工，不用記事情，誰上都行；

Stateful 就像是管帳的阿姨，資料存在她腦中，不能亂換人。


