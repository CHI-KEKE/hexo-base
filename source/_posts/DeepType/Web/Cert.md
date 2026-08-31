---
title: Cert
date: 2025-09-25 08:28:34
categories: Web
top_img: https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
tags:
    - Web
toc:
toc_number:
comments :
---


{% tabs Cert%}


<!-- tab 憑證-->

憑證是一個「資料包」，裡面同時包含

- 網域資訊（CN/SAN）
- 公鑰（Public Key）
- 其他欄位（有效期限、用途…）
- CA 的數位簽章（Signature）

公鑰只是其中一個欄位，功能是給 TLS 握手用（以及驗證伺服器真的有私鑰）


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Cert/1_what_is_cert.png)


<!-- endtab -->

<!-- tab 公鑰-->

公鑰是非對稱加密（Public Key Cryptography）裡的一半鑰匙，通常是一對

公鑰（Public Key）：可以公開給所有人
私鑰（Private Key）：只能放在伺服器手上，不能外流

在 HTTPS 裡面，公鑰主要用來做兩件事

協助建立加密連線
瀏覽器拿到憑證的公鑰後，可以在握手過程中完成「只有真正持有私鑰的那台伺服器才接得住」的驗證/交換流程（不同握手版本細節不同，但核心概念是這樣）。

驗證連到的伺服器真的握有私鑰，因為只有持有私鑰的人，才做得到某些對應的加密/簽名運算


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Cert/2_public_and_private.png)


<!-- endtab -->

<!-- tab 驗證流程-->

- 在瀏覽器輸入 https://example.com

- 瀏覽器跟伺服器開始建立 HTTPS 連線（握手過程）

- 伺服器送出「憑證」給瀏覽器

- 瀏覽器檢查憑證內容
  - 這張憑證寫的網域是不是 example.com（看 CN / SAN）
  - 有沒有過期（有效期限）
  - 這張憑證是不是某個 CA 簽發的（發行者 / 簽章）
  - 瀏覽器再用自己電腦/系統內建的「信任清單」去確認這個 CA 我信不信
  - 檢查都通過 → 顯示 🔒，接著用憑證裡的公鑰等資訊把加密通道建立起來

任一關不通過 → 跳警告（例如：不受信任、過期、網域對不上）


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Cert/3_check_cert.png)


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Cert/3_tls_handshake.png)

<!-- endtab -->

<!-- tab CN / SAN-->

憑證要能「列出我可以代表哪些網域」，瀏覽器才知道你不是拿別人的證來冒充。現在一張憑證常常要同時涵蓋多個網域/子網域（例如 example.com、www.example.com、api.example.com），主要靠 SAN 來列清單，如果連的是 www.example.com 但憑證只寫 example.com，瀏覽器就會說「名字對不上」直接警告


<!-- endtab -->

<!-- tab Root Certificate (根憑證)-->

公開網際網路之所以能運作，是因為我們自己的作業系統出廠時就先塞好一批「大家公認的 CA 根憑證」，例如今天去一家新網站，網站憑證顯示「由某 CA 簽發」。而瀏覽器不會因為它自稱 CA 就相信，而是會一路追到根憑證，「這家 CA 的根憑證有沒有被我系統列為可信？」，有就放行，而沒有（例如公司內部自建 CA，或陌生 CA）則會跳警告或需要手動安裝那張根憑證


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Cert/4_chain_of_trust.png)


<!-- endtab -->

<!-- tab 本機開發環境自簽的 Root Certificate-->

自己做一個「本機專用的發證機關」，再把它加入電腦的信任清單，這樣在 https://localhost 或自訂網域跑 HTTPS 才不會每次都被瀏覽器跳紅字嚇阻

![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Cert/5_local_teat_hard.png)

在做需要 HTTPS 的功能，例如 Service Worker / PWA、HTTP-only / Secure cookie 行為測試、OAuth / SSO redirect URI 需要 https、WebAuthn、Camera/Mic、某些 API 在非安全來源會被限制，而這些在純 http://localhost 或被警告擋住時很麻煩。因此自簽 Root CA + 本機信任後，就可以在本機用「像正式環境一樣」的 HTTPS 行為測試


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Cert/6_why_https.png)


<!-- endtab -->

<!-- tab 不同的憑證-->

公有 CA 發的憑證（正式用，像 Let’s Encrypt, DigiCert），瀏覽器/OS 預設信任，線上網站一定要用這種。

自簽憑證 (Self-signed certificate)（本機開發、測試用），例如 localhost 憑證，這種瀏覽器不會自動信任，要自己手動安裝

而開發憑證 (dotnet dev-certs)，.NET Core / Visual Studio 自動產生的，目的是讓你在 https://localhost 測試時不會跳「不安全」。


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Cert/7_dev_and_prod.png)


## 用 .NET CLI 產生/管理一張「開發用、自簽」的 HTTPS 憑證


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Cert/8_net_cert.png)


1. 執行 dotnet dev-certs https，.NET CLI 會產生/管理一張「開發用、自簽」的 HTTPS 憑證
2. 再執行 dotnet dev-certs https --trust，會把這張開發憑證加進 OS 的信任機制（不同平台行為略不同），讓瀏覽器顯示 🔒 而不是警告
3. ASP.NET Core（Kestrel/IIS Express）啟動時就能用這張憑證在 https://localhost:xxxx 提供 HTTPS


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Cert/8_net_cli.png)


```bash
# 檢查目前是否有可用的開發憑證
dotnet dev-certs https --check

# 清掉舊的開發憑證（常用來解決「突然不被信任」或過期等問題）
dotnet dev-certs https --clean

# 重新建立並信任（最常見的修復組合）
dotnet dev-certs https --trust
```


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Cert/8_net_cert_command.png)


<!-- endtab -->



<!-- tab https 與憑證-->

公開網際網路之所以能運作，是因為我們自己的作業系統出廠時就先塞好一批「大家公認的 CA 根憑證」，例如今天去一家新網站，網站憑證顯示「由某 CA 簽發」。而瀏覽器不會因為它自稱 CA 就相信，而是會一路追到根憑證，「這家 CA 的根憑證有沒有被我系統列為可信？」，有就放行，而沒有（例如公司內部自建 CA，或陌生 CA）則會跳警告或需要手動安裝那張根憑證



## 沒憑證，TLS 沒辦法做「身分驗證」

HTTPS 連線一開始，伺服器必須拿出一個證據讓瀏覽器相信「我真的代表這個網域（example.com）」、「而且這把公鑰真的屬於我」，這個證據就是憑證：網域（SAN/CN）+ 公鑰 + CA 簽章，少了它，瀏覽器根本不知道對方是不是冒牌貨，MITM（中間人攻擊）會變得容易

## 沒憑證，現代瀏覽器通常直接不給你建立 HTTPS

可以把 TLS 想成「談好一組加密方式與鑰匙」的流程。在 Web 的世界裡，瀏覽器對「安全」有硬規則，網站要提供 HTTPS，基本上就得出示憑證。伺服器如果沒證書／不送證書，TLS 握手會直接失敗，而握手失敗 = HTTPS 連線建立不起來 = 你連不上，所以從使用者角度就是沒有憑證，瀏覽器連不上 HTTPS


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Cert/9_keys.png)


<!-- endtab -->


<!-- tab summary-->


![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Web/Cert/cert.png)


<!-- endtab -->

{% endtabs %}

