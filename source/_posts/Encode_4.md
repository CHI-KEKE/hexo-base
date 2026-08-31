---
title: Certificate
date: 2025-07-13 11:19:11
categories: 暗号者の秘密の部屋
top_img: https://i.imgur.com/ltxqyEt.png
cover : https://i.imgur.com/ltxqyEt.png
tags:
    - Encode
toc:
toc_number:
comments :
---

![Encode](https://i.imgur.com/ltxqyEt.png)

在數位世界的浩瀚星海中，有一種神秘的語言穿梭於不同的系統之間，它就是 JSON。今天，讓我們踏上一場探索編碼奧秘的旅程，揭開 Unicode 轉義背後的故事。

<br>
















先前提到的加密通信中，人類會有 發公鑰 或 交換公鑰 的行為，但實務上，除非這個人走到你旁邊，手寫這一段公鑰交給你，否則你無法確信這一段到手的公鑰訊息被動過甚麼手腳

所以人類又想到了，我們引入 "第三方公證人"吧 (考駕照也是要到駕訓班認證過，上交幾隻雞腿，然後才能拿到駕照上路吧)




當有一個傢伙他想要發布公鑰時，他就要走進這個公正的機關 (CA Certificate Authority)，在這個世界做資料驗證與證書頒發


於是就有了 "證書"(Certificate) 的概念


## Certificate

證書包含:
1.持有人姓名
2.發證機關
3.證書效期
4.證書持有人公鑰
5.其他訊息
6.Signature
...


以最生活化的例子果然還是 "HTTPS" 了吧，平時瀏覽網站時總會看到一個鎖在瀏覽器上，表示 "該網站是經由公正CA認證過的"

比如

今天 yolin.io 向 CA 機構申請一張證書:

1.填上一些訊息，也就是上述列表的資料包含自己的域名
2.自己也會生出一對公私鑰，把公鑰交給 CA
3.CA 會把資訊 Hash 後用自己的私鑰加密，得到 Signature
4.審核過後，CA 就會把證書頒發給 yolin.io，Allen 就把這個證書裝在自己的 Server上
5.逛網站的人進來時，發送了 HTTP Request， Server 回給 瀏覽器一個證書，瀏覽器把它安裝在自己身上
6.瀏覽器本身會有一些 root Certificate，其中會帶有 CA 的 公鑰，解密後比對 Signature
7.比對成功，瀏覽器用"證書持有人的公鑰"加密訊息，送給 Server，Server能夠解密後就可安全發送資訊了!


## Demo

.pfx (Personal Information Exchange) 格式: 包含了證書和對應的私鑰，以及可選的證書鏈，通常用於在環境間的導入和導出(畢竟不會自己在用不會只傳公鑰或私鑰吧(?))

```csharp

void Main()
{
	//// 讀檔
	string senderPrivateKeyPath = @"C:/Users/Allen Lin/Desktop/allen_generate_the_secret_key.pfx";
	string senderPrivateKeyPassword = "12345";
	
	//// 從取得的證書把公私鑰搞出來
	var cert = new X509Certificate2(senderPrivateKeyPath, senderPrivateKeyPassword);
	var privateKey = cert.GetRSAPrivateKey();
	var publicKey = cert.GetRSAPublicKey();
	
    //// 家長簽名
    var data = System.Text.Encoding.UTF8.GetBytes("我也想去沖繩挖");
    var signature = privateKey.SignData(data, HashAlgorithmName.SHA256, RSASignaturePadding.Pkcs1);
    Console.WriteLine("Signature: " + Convert.ToBase64String(signature));

    //// 老師比對家長簽名
    var verified = publicKey.VerifyData(data, signature, HashAlgorithmName.SHA256, RSASignaturePadding.Pkcs1);
    Console.WriteLine("Signature verified: " + verified);
}

```
## 今日精神能量分析

精神能量 : 👾

感覺這台電腦已經快堅持不住了，曾經的山盟海誓，說好的一起對抗巨人們(超大型巨人、猩猩巨人、獨裁巨人...) 的壓迫，仍然敵不過現實的打擊阿(拍拍......阿又當機了)

