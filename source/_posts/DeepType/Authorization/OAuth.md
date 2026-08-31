---
title: OAuth 2.0
date: 2024-08-05 23:02:00
categories: Authorization
top_img: https://i.imgur.com/rJvffnd.png
cover : https://i.imgur.com/rJvffnd.png
tags:
    - 
toc:
toc_number:
comments :
---

{% tabs OAuth 2.0%}


![l](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Authorization/OAuth/landing.png?raw=true)


<!-- tab OpenID-->

讓我們想像一下，如果每次進入一個新的國家，都需要重新申請一個新的身份證會很麻煩，但這就是早期網路使用者的日常生活 —— 每個網站都需要一個新的帳號和密碼。於是，OpenID 的概念應運而生，它就像是網路世界的通用護照，為技術發展添加了人性化的一面


![op](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Authorization/OAuth/a_lot_of_passwords.png?raw=true)


故事開始於 Blaine Cook  試圖在 Twitter 上實現 OpenID。這就像是試圖讓你的護照（OpenID）不僅能證明你的身份，還能授權別人代表你處理一些事務。但 Blaine 和他的朋友發現，OpenID 的核心價值，是讓「證明你是誰」這件事可以被重複使用，而不用每個系統都自己養一套帳密系統，而不是授權（允許別人做什麼）

使用者已經在某個可信任的地方有身分，例如一個 OpenID Provider（身分提供者），接著他嘗試登入另一個網站，這個網站不想自己管理帳密，而網站把「你是誰的驗證責任」交給 OpenID Provider，因此網站不碰你的密碼，只相信驗證結果，OpenID Provider 回傳驗證成功，而網站只在乎一件事「你是不是你」


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Authorization/OAuth/authen_autho.png)


<!-- endtab -->

<!-- tab OAuth-->


於是，OAuth 的概念誕生，用來解決授權問題。如果說 OpenID 是護照，那麼 OAuth 就比較像授權書，允許特定的人或應用程式以你的名義執行特定的操作



Blaine Cook 約一幫狐群狗黨 —— David Recordon、Larry Halff ...，在 CitizenSpace 的一次 OpenID 聚會上碰面。大家討論得熱火朝天、面紅耳赤、大頭巨耳，Larry 正在為 Ma.gnolia 的儀表板小工具絞盡腦汁，也想找個好方法來整合 OpenID，最後他們得出一個的結論：欸斗，居然沒有一個開放標準來處理 API 訪問授權！

![Image](https://i.imgur.com/OCYCB7e.png)


這個想法像新冠一樣在線上線下傳播開。
到了 2007 年 4 月，事情開始變得正式起來。他們成立一個 Google 群組，並開始召集人選，Blaine Cook 對著前來的人說道: 讓我看看你的念吧!

經過一陣嚴格的篩選，組好隊開始摩拳擦掌準備著手起草協議的提案。2007 年 7 月，團隊終於拿出了一份初步規範。他們決定敞開大門，歡迎任何對此感興趣的人來貢獻自己的智慧。而 3 個月後，OAuth Core 1.0 的最終草案終於問世

接著，到 2012，OAuth 升級為 OAuth 2.0，他簡化開發過程，降低了實施難度，並且使用 HTTPS 進行傳輸層安全減少了開發者的負擔，同時保持了安全性並且引入了刷新 token 的概念。就像有效期的簽證，需要定期更新，還有其他一系列的改善

![p](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Authorization/OAuth/the_progress.png)


![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Authorization/OAuth/2.0_inprove.png)

<!-- endtab -->

<!-- tab OAuth 2.0 (Open Authorization)-->


下面這個畫面是不是很熟悉

![Image](https://i.imgur.com/stYgnPE.png)

現在許多知名的網站幾乎都可以支持使用 OpenId 登入，讓你不用再建立一個帳號密碼，更不用給這個網站自己的 Google 帳號密碼

OAuth 把「誰信任誰」這件事拆成三方，避免任何一方拿到不該拿的權力

三個角色各自只做一件事：

- 網站 / App（Client）想取得資料，但不該拿密碼
- OAuth Provider（如 Google、Microsoft），負責驗證身份，負責詢問使用者是否同意
- End-User，真正擁有帳號與資料的人，可以決定「給不給、給多少」

![a](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Authorization/OAuth/triangle.png?raw=true)


成功藉由 Google OAuth 登入後，網站就可以拿到一個 token，接著網站就可以透過這個 token 從 Google 取得用戶部分權限的資料，像是頭像、偏好...

可以參考 Google 文件的 Google OAuth 2.0 Flow
![Image](https://i.imgur.com/NYJcxAG.png)


1. 使用者在你的網站點擊「使用 Google 登入」，你的網站不負責驗證帳密，只負責「把人送到 Google」

![1](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Authorization/OAuth/step1.png)

2. 使用者在 Provider（Google）完成登入與同意授權、輸入帳密、勾選同意哪些資料可以被使用

3. Google 回傳一個 Authorization Code 給你的網站，這個 Code 本身不能用來存取資料，只是一張「一次性兌換券」

![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Authorization/OAuth/step2.png)


4. 你的後端用 Authorization Code 向 Google 換取 Token，這一步一定在 Server 端進行，Client Secret 不會曝光，網站拿著 Token 呼叫 Google API，只能存取使用者「同意過的範圍（scope）」


![3](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Authorization/OAuth/step3.png)


<!-- endtab -->

<!-- tab 為什麼中間會先回一個 Authorization Code，而且還暴露在 Url上-->


`Authorization Code` 的存在，是為了避免 Token 在瀏覽器或網路上被撿走，因為在身分驗證階段，用戶被導到 Auth Server (這裡是 Google)，在那邊同意授權 Postman 可以取得一定程度的各訊息後，會在瀏覽器被導回 Postman，此時，如果直接把 Token 在瀏覽器間傳遞，是相對高風險的作法，是可以被截取的，方法可能有很多種，例如取得用戶 Chrome 的歷史文件資訊、取得Log、中間人攻擊、社交工程、Referrer，因此，在純前端的傳遞，會先給一個 Grant (也就是 `Authorization Code`)，Postman Server 連同 ClientId / Secret 以及 Code 向 Google 要 token 去做後面一些比較敏感的資訊操作

![c](https://github.com/CHI-KEKE/pics/blob/main/Codesss/Authorization/OAuth/why_auth_code.png?raw=true)


<!-- endtab -->

<!-- tab 資料分享-->

參考影片 : https://www.youtube.com/watch?v=996OiexHze0&t=144s

演講的主要內容，包括 OAuth 2.0 和 OpenID Connect 的區別，在網路世界遇到的認證困難，例如不同 device 認證機制、delegated authorization problem(Yelp 曾經直接跟用戶使用 Google 帳密...)，接著說明 Delegated authorization with OAuth 2.0 以及 token解析，最後 demo OAuth，我覺得對 OAuth 有一點簡單的理解再看會是一個不錯的銜接教材(不會難度突然跳太多)

<!-- endtab -->

<!-- tab summary-->


![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Authorization/OAuth/summa.png)


<!-- endtab -->


{% endtabs %}