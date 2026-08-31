---
title: Socket Exhaustion - 當臨時的港口被耗盡
date: 2025-08-26 08:35:01
categories: 穿越雲層的信使
top_img: https://i.imgur.com/hZWviQz.png
cover : https://i.imgur.com/hZWviQz.png
tags:
    - Http
toc:
toc_number:
comments :
---

在浩瀚的網路之海，每一次 API 呼叫，就像放出一位無聲的信使。
他揹著我們的訊息，穿越纜線、翻越路由器，終於抵達遠方的伺服器。
然而，這段旅程並非沒有代價。

當信使啟程時，作業系統會替他分配一個「臨時的港口」（ephemeral port），讓他能安全啟航。
但若我們過於頻繁地呼喚、過於急切地派遣信使，這些港口就會一個接一個被佔滿。
直到最後，新信使無處登船——這就是 Socket Exhaustion 的隱憂。


![message](https://i.imgur.com/hZWviQz.png)

<br>

## 📨 TCP 4-tuple

要出海，信使需要四件通關憑證：

假設我們今天上 Google 首頁，其實就是建立一條 TCP 連線，這條連線會用到這「四件東西」來唯一識別這條通道：

| 項目                 | 說明                          |
| ------------------ | --------------------------- |
| 你的 IP（來源 IP）       | 例如：192.168.1.100            |
| 你的 Port（來源 Port）   | ⚠️ 是作業系統隨機分配的，例如 51234      |
| 伺服器的 IP（目的 IP）     | 例如：142.250.190.46           |
| 伺服器的 Port（目的 Port） | ✅ 這通常是 80（HTTP）或 443（HTTPS） |

這四個合起來叫做：TCP 4-tuple
就像護照、簽證、船票與碼頭位置，缺一不可。

<br>
<br>

## 📨 港口的數量：Ephemeral Port Range

你本機的 port 怎麼來的？

作業系統會在一段叫做「ephemeral port range」的範圍內選一個沒人用的。

| 系統      | 暫時 port 範圍（預設） |
| ------- | -------------- |
| Windows | 49152 到 65535  |
| Linux   | 32768 到 60999  |

也這意味著——
你同時最多只能有「幾萬名信使」同時出海。
雖然看似壯觀，卻遠比無限遼闊的網路要侷促得多。


<br>
<br>

## 📨 所以實際情況怎樣用光 port？

1. 每一個 HttpClient 預設會新建一個 TCP socket
2. 每個 socket 都要分配一個新的臨時 port
3. 這些連線即使 Dispose()，也會暫時留在 TIME_WAIT 狀態（2~4 分鐘）

👉 幾萬次下來，你的臨時 port 全部被用掉
👉 下一次你就無法開新連線了

```PLAINTEXT
SocketException: Address already in use
```

一個擁擠不堪的港口——船隻停滿碼頭，舊船尚未離開，新船卻急著進港，終於造成了癱瘓。


<br>
<br>

## 📨 一個 HttpClient 船隊，需要多少資源？

每次 new HttpClient()（預設）會建立新的 HttpMessageHandler，而 HttpMessageHandler 會新建 TCP 連線（Socket），每個 Socket 都會佔用作業系統層級的 TCP 連線資源，TCP Socket 會佔用一組 (你的IP, 本地Port, 對方IP, 對方Port)，而連線還會消耗：

- 作業系統的 Port 號（通常 client port）
- 記憶體（socket 結構本身 + buffer）
- 網路資源（TCP connection table）
- 網路卡處理能力（NIC queue）

更具體來說是這樣

| 資源類型         | 說明                                                  |
| ------------ | --------------------------------------------------- |
| 🧠 記憶體       | 每個 Socket 會佔用一塊記憶體儲存狀態（如 send/receive buffer）       |
| 🔢 本地 Port 號 | 每個 TCP 連線都需要一個本地 Port（通常從 49152 到 65535）<br>最多只有幾萬個 |
| 📊 OS 資源     | 作業系統內部的「連線表」會變大，效率下降                                |
| ⏳ 時間資源       | 即使你關掉連線，系統還要等 `TIME_WAIT` 清理（預設 240 秒）              |
| 🔄 網路開銷      | TCP handshake（建立時）跟 teardown（關閉時）都要消耗網路效能           |


<br>
<br>

## 📨 TIME_WAIT

當你關閉一個 TCP 連線時，系統會把這個連線「保留」在 TIME_WAIT 狀態一段時間（通常是 2-4 分鐘），避免封包延遲導致誤判。 

當一艘船返航，系統並不會立刻拆除港口的繫船柱。它會保留一段時間，避免遲來的舊訊息誤闖新船，造成混亂。
這段保留時間，便是 TIME_WAIT。


<br>
<br>

## 📨 HttpClient 和 HttpMessageHandler 是什麼關係

HttpClient：就像是「航運公司」的門面，方便下單出航。
HttpMessageHandler：則是港口總管，真正安排航道、連線與驗證細節。

每當你 new HttpClient()，背後就會新生一位 HttpClientHandler，管理這趟航程。

HttpClient 是「高層」的 API，讓你可以簡單地送出 HTTP 請求。HttpMessageHandler 是「低層」的實作，真正處理 HTTP 請求/回應的邏輯和連線細節。當你 new HttpClient() 時，背後會自動 new 一個叫做 HttpClientHandler 的預設 handler（繼承自 HttpMessageHandler）


<br>
<br>

## 📨 HttpClientHandler 存了什麼


它會管理一些「狀態性」的東西，例如：

| 資料                                            | 說明                                 |
| --------------------------------------------- | ---------------------------------- |
| **CookieContainer**                           | 如果你沒關掉自動 Cookie，就會記住伺服器送來的 cookies |
| **Proxy 設定**                                  | 是否透過某個 Proxy 傳送                    |
| **ServerCertificateCustomValidationCallback** | 憑證驗證邏輯                             |
| **Credentials**（帳密）                           | 基本驗證或 NTLM 的憑證資料                   |
| **Connection Pool**                           | TCP 連線池（這是重點）                      |

預設的 CookieContainer 行為，HttpClientHandler 預設 UseCookies = true，它內部會建立一個 新的 CookieContainer 實例，所以如果每次 new handler，就會有自己的 cookie 存儲空間，但是若重複使用 handler，它的 CookieContainer 就是共用的!


<br>
<br>

## 📨 HttpClientFactory

如何避免港口被佔滿？如何避免信使混用身份？

也就是
- 避免耗盡 socket（連線爆掉）
- 避免狀態混用（如 cookie、憑證被共用）

微軟給出的答案就是 IHttpClientFactory。

IHttpClientFactory 背後會幫忙共用 HttpMessageHandler，但會加上一層壽命控制機制（handler lifetime），預設是 TimeSpan.FromMinutes(2) 每 2 分鐘回收一次 handler

這代表每一組相同設定的 HttpClient，共用同一個 HttpClientHandler，那個 handler 就可以共用 TCP 連線池，不會每次都建立新的 TCP Socket，連線會被重複使用，不會爆掉！

IHttpClientFactory 把「設定」跟「Handler Pool」綁起來分開管理，每一組不同設定（BaseAddress、Header、自定義 Handler）都會建立獨立的 handler pool，也就是：只要你給不同的設定，它就會建立不同的 handler，狀態不會互相干擾，所以不會出現 A 使用者的 cookie 被 B 用走的情況！

| 功能 / 行為                  | new HttpClient   | 重用 handler | IHttpClientFactory ✅ |
| ------------------------ | ---------------- | ---------- | -------------------- |
| 連線重用                     | ❌ 每次都新連線         | ✅ 共用       | ✅ 共用                 |
| 狀態分離（cookie, credential） | ✅ 分離             | ❌ 會混用      | ✅ 自動隔離               |
| 資源回收                     | ❌ handler 不會自動釋放 | ❌ 要自己釋放    | ✅ 自動控制 handler 壽命    |
| 適合實務環境                   | ❌ 容易踩雷           | ⚠️ 要很小心    | ✅ 安全且彈性              |


若「單純改 header（如 Authorization）」不會產生新的 handler，也不會建立新的連線池，因為 IHttpClientFactory 的 handler 是根據「Client 註冊設定」來分配的，而不是 runtime 代入的 header 內容


換句話說

```CSHARP
var client = _httpClientFactory.CreateClient("MyApi");

client.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", "token-123"); // 設定 API 密鑰

await client.GetAsync("/data");
```

然後下一次：
```CSHARP
var client = _httpClientFactory.CreateClient("MyApi");

client.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", "token-XYZ"); // 換另一組密鑰

await client.GetAsync("/data");
```

這兩次都使用相同的 handler pool 與 socket pool，因為 IHttpClientFactory 在這兩次呼叫中使用的是同一個 "Named Client"（"MyApi"），設定 handler 的時候已經完，header 是後來才加的（屬於 request 層級的設定，不影響 handler 池）


如果我真的要不同密鑰分開 handler 就必須要「註冊多個 named clients 或 typed clients」，分開設定


<br>
<br>

## 📨 為什麼不能「每個 Service new 一個 HttpClient 然後重複使用」就好？

- 這個 HttpClient 背後的 Handler 永遠活著（沒人幫你清理）
- 你還是無法共享連線池給其他 API 或服務
- 你必須自己確保這個 Handler 不會持有不該共用的狀態（例如 Cookie）

🔥 手動管理 HttpClient 雖然可行，但風險與複雜度很高 => 而 HttpClientFactory 就是幫你 自動做這些事，還不會踩雷


## 簡單的實驗

建立 HttpClient , 發送 Reqeust
```CSHARP
async void Main()
{
	for(int i = 0;i < 10;i++)
	{
		using (var httpClient = new HttpClient())
		{
			var response = await httpClient.GetAsync("https://httpbin.org/get");
			var responseContent = await response.Content.ReadAsStringAsync();
			Console.WriteLine(responseContent);
		}
	}
}
```

查看港口狀態
```POWERSHELL
netstat -ano | findstr TIME_WAIT
```

netstat 是 Windows 提供的網路狀態檢查工具，用來查看電腦目前的網路連線、開啟的埠口與對應的行程 (process)。

常見參數：

- -a：顯示所有連線與監聽中的埠口 (Listening、Established、Time_Wait…)
- -n：以數字方式顯示 IP 與 Port（避免把 443 轉成 "https" 這種文字）
- -o：顯示對應的行程 PID (Process ID)

所以 netstat -ano 會列出 所有 TCP/UDP 連線狀態，包含本機 IP/Port、遠端 IP/Port、狀態 (如 ESTABLISHED、TIME_WAIT)、以及是哪個程式的 PID。

- | 是管線 (pipe)，把前一個 netstat -ano 的結果再丟給 findstr 去做文字搜尋。
- findstr 就像 Windows 的 grep，可以篩選出符合關鍵字的行。
- TIME_WAIT 是 TCP 連線的一種狀態，表示該連線已經關閉，但系統還保留一段時間（通常 1~4 分鐘）以確保延遲封包不會影響後續新的連線。

<br>
<br>

## 📨 結語

最終，我們發現，問題從來不在於信使太多，而在於我們是否懂得為他們安排合理的歸途。
HttpClientFactory 就像是港口的調度員，讓信使們能共用航道，又不至於混亂。
當我們不再讓每一位信使孤軍奮戰，而是讓他們在秩序中流動，才會真正體會：在雲端之間，持續的交流，並不是靠無窮的資源堆疊，而是依靠一套優雅的秩序。