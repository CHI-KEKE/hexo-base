購物車操作時帶上 uniqueKey，其實不完全是為了防重播攻擊，而是為了：

保證資料正確性（資料的歸屬與一致性）

避免資料混淆或誤用（區分同一帳號不同來源）

有時也可以延伸作為防止重播、防偽等手段

🧠 拆解購物車 uniqueKey 的用途（真實意圖）
🔹1️⃣ 識別這個「購物流程的上下文」【主要目的】
✅ 舉例說明：

當使用者進行購物行為（點選商品 → 加入購物車 → 結帳），系統會幫他產生一個：

CartSessionId = "CART-20251118-8FDE23C8F231"


這個 uniqueKey（或叫 CartId、CartSessionId、TransactionId...）：

代表一組購物流程

在操作「新增商品、刪除商品、選擇優惠」時都要帶上

就像「你去 IKEA 拿購物推車時的車號」

🔹2️⃣ 避免同帳號操作被混淆
📦 假設情境：

小明開了兩個瀏覽器分頁，各自買不同商品

如果沒有 uniqueKey，可能會把 A 分頁的商品加到 B 分頁的購物車中

加上 uniqueKey，每次操作都清楚知道是在哪一台車上加的

🔹3️⃣ 支援匿名用戶購物

很多電商允許「沒登入」也能先加商品進購物車

這時就要用 uniqueKey 當作這個「匿名用戶」的識別

否則不知道這些商品該歸誰

✅ 通常這會儲存在 Cookie / LocalStorage / Session
localStorage.setItem("cartKey", "8FDE23C8F231")

🔹4️⃣ 可延伸作為 Replay 防護用途（進階用途）

雖然 uniqueKey 本身不是為了防 replay 而生，但可以被用來達成：

應用	說明
防止重複提交	使用 uniqueKey 判斷請求是否執行過
防止多次加購同商品	每次操作都要檢查 uniqueKey + 商品Id 是否存在
防止重播攻擊	搭配 TimeStamp、Nonce 也可以判斷請求是否合法

