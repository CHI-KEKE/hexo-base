---
title: Bouncy Castle
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

不知道大家有沒有使用過 Bouncy Castle，我自己是在串接金流時誤打誤撞碰到，想當年(也才半年...)對 C# 加密、證書、簽章...都還沒甚麼概念的情況下，在網路上找到這個 Package，But...看名字還不太敢用，畢竟NuGet Gallery的 Candidate 多如腿毛 (這就是所謂的第一印象嗎?)，猶豫許久，後來發現公司專案也經有人引入並且使用過後，才鬆了一口氣...</br>

加解密、簽章、證書等等處理，Bouncy Castle 像瑞士刀一樣，引用 Utilities，抽一把武器來用解決你的需求，且後來發現，原來他是一個跨語言的套件(想想好像也合理)，直接 Google 搜可能還會看到一堆 Java 的 Sample


## Demo

現在有點難自己寫出來，所以先拿別人的Code來練習

取自 : 

https://blog.darkthread.net/blog/bouncy-castle/
https://gist.github.com/therightstuff/aa65356e95f8d0aae888e9f61aa29414
https://en.wikipedia.org/wiki/RSA_(cryptosystem)#Padding_schemes

我這邊只把 RSACryptoServiceProvider 改成使用 RSA

Code有點長，可收合

RSA 處理加解密、簽章驗證

```csharp

using System.Security.Cryptography;

namespace Bouncy_Castle
{
    public class NetCrypto : IRsaCrypto, IDisposable
    {
        //// 定義長度、算法
        private int _keySize = 2048;

        private string _hashAlg = "SHA256";

        //// key Provider (NET)
        private RSA _publicKey = null;

        private RSA _privateKey = null;

        //// key Provider 存取
        private RSA PublicKeyProvider => _publicKey ?? throw new ApplicationException("No public key is set.");

        private RSA PrivateKeyProvider => _privateKey ?? throw new ApplicationException("No private key is set.");

        public string PubKey
        {
            get => ExportPublicKey(PublicKeyProvider);
            set => ImportPublicKey(value);
        }

        public string PrivateKey
        {
            get => ExportPrivateKey(PrivateKeyProvider);
            set => ImportPrivateKey(value);
        }

        // 匯出公鑰
        private string ExportPublicKey(RSA rsa)
        {
            var parameters = rsa.ExportParameters(false);
            return Convert.ToBase64String(parameters.Modulus) + "|" + Convert.ToBase64String(parameters.Exponent);
        }

        // 匯入公鑰
        private RSA ImportPublicKey(string key)
        {
            var parts = key.Split('|');
            var modulus = Convert.FromBase64String(parts[0]);
            var exponent = Convert.FromBase64String(parts[1]);

            var rsa = RSA.Create();
            rsa.ImportParameters(new RSAParameters { Modulus = modulus, Exponent = exponent });
            return rsa;
        }

        // 匯出私鑰
        private string ExportPrivateKey(RSA rsa)
        {
            var parameters = rsa.ExportParameters(true);
            return Convert.ToBase64String(parameters.Modulus) + "|" +
                   Convert.ToBase64String(parameters.Exponent) + "|" +
                   Convert.ToBase64String(parameters.D) + "|" +
                   Convert.ToBase64String(parameters.P) + "|" +
                   Convert.ToBase64String(parameters.Q) + "|" +
                   Convert.ToBase64String(parameters.DP) + "|" +
                   Convert.ToBase64String(parameters.DQ) + "|" +
                   Convert.ToBase64String(parameters.InverseQ);
        }

        // 匯入私鑰
        private RSA ImportPrivateKey(string key)
        {
            var parts = key.Split('|');
            var modulus = Convert.FromBase64String(parts[0]);
            var exponent = Convert.FromBase64String(parts[1]);
            var d = Convert.FromBase64String(parts[2]);
            var p = Convert.FromBase64String(parts[3]);
            var q = Convert.FromBase64String(parts[4]);
            var dp = Convert.FromBase64String(parts[5]);
            var dq = Convert.FromBase64String(parts[6]);
            var inverseQ = Convert.FromBase64String(parts[7]);

            var rsa = RSA.Create();
            rsa.ImportParameters(new RSAParameters
            {
                Modulus = modulus,
                Exponent = exponent,
                D = d,
                P = p,
                Q = q,
                DP = dp,
                DQ = dq,
                InverseQ = inverseQ
            });
            return rsa;
        }

        //// 建構子1
        public NetCrypto(int? keySize = null, string hashAlgorithm = null!)
        {
            _keySize = keySize ?? _keySize;
            _hashAlg = hashAlgorithm ?? _hashAlg;
            var csp = RSA.Create();
            csp.KeySize = _keySize;
            _privateKey = _publicKey = csp;
        }

        //// 建構子2
        public NetCrypto(string pubKey, string privKey, string hashAlgorithm = null!)
        {
            _hashAlg = hashAlgorithm ?? _hashAlg;
            if (!string.IsNullOrEmpty(pubKey)) PubKey = pubKey;
            if (!string.IsNullOrEmpty(privKey)) PrivateKey = privKey;
        }

        //// 實例化
        public static NetCrypto FromPubKey(string pubKey, string hashAlgorithm = null!) =>
            new NetCrypto(pubKey, null!, hashAlgorithm);

        public static NetCrypto FromPrivKey(string privKey, string hashAlgorithm = null!) =>
            new NetCrypto(null!, privKey, hashAlgorithm);

        //// 加解密

        // fOAEP = false, use PKCS#1 v1.5 padding
        // fOAEP = true, use OAEP padding

        public byte[] Encrypt(byte[] plainBytes)
           => PublicKeyProvider.Encrypt(plainBytes, RSAEncryptionPadding.OaepSHA256);

        public byte[] Decrypt(byte[] cipherData)
            => PublicKeyProvider.Decrypt(cipherData, RSAEncryptionPadding.OaepSHA256);

        //// 簽章
        public byte[] Sign(byte[] data)
            => PrivateKeyProvider.SignData(data, new HashAlgorithmName(_hashAlg), RSASignaturePadding.Pkcs1);

        public bool Verify(byte[] data, byte[] signature)
            => PublicKeyProvider.VerifyData(data, signature, new HashAlgorithmName(_hashAlg), RSASignaturePadding.Pkcs1);

        public void Dispose()
        {
            _publicKey?.Dispose();
            _privateKey?.Dispose();
        }
    }
}

```

BouncyCastle 處理加解密、簽章驗證

```csharp

using Org.BouncyCastle.Crypto;
using Org.BouncyCastle.Crypto.Generators;
using Org.BouncyCastle.OpenSsl;
using Org.BouncyCastle.Security;

namespace Bouncy_Castle
{
    public class BouncyCastleCrypto : IRsaCrypto, IDisposable
    {
        //// 定義長度、算法
        private int _keySize = 2048;
        private string _hashAlg = "SHA-256withRSA";
        private string _cipherSuite = "RSA/ECB/PKCS1Padding";

        //// key Provider (BC)
        private AsymmetricKeyParameter _publicKeyParam = null;

        private AsymmetricCipherKeyPair _keyPair = null;

        //// key String
        private string _pubkey = null;

        private string _privateKey = null;

        //// 單獨密鑰、密鑰對 存取
        private AsymmetricKeyParameter AsymKeyParam => _publicKeyParam ?? throw new ApplicationException("公鑰未設定!");

        private AsymmetricCipherKeyPair AsymCipherKeyPair => _keyPair ?? throw new ApplicationException("私鑰未設定");

        //// Pubkey String 存取
        public string PubKey
        {
            get => _pubkey;
            private set
            {
                var pubKeyReader = new StringReader(value);
                var pemReader = new PemReader(pubKeyReader);
                _publicKeyParam = (AsymmetricKeyParameter)pemReader.ReadObject();
                _pubkey = value;
            }
        }

        //// Privatekey String 存取
        public string PrivateKey
        {
            get => _privateKey;
            set
            {
                var privateKeyReader = new StringReader(value);
                var pemReader = new PemReader(privateKeyReader);
                _keyPair = (AsymmetricCipherKeyPair)pemReader.ReadObject();
                _privateKey = value;
            }
        }

        //// 建構子1
        public BouncyCastleCrypto(int? keySize = null, string hashAlgorithm = null!, string cipherSuite = null!)
        {
            _keySize = keySize ?? _keySize;
            _hashAlg = hashAlgorithm ?? _hashAlg;
            _cipherSuite = cipherSuite ?? _cipherSuite;
            (PubKey, PrivateKey) = GenerateKeyPair();
        }

        //// 建構子2
        public BouncyCastleCrypto(string pubKey, string privKey, string hashAlgorithm = null!, string cipherSuite = null!)
        {
            _hashAlg = hashAlgorithm ?? _hashAlg;
            _cipherSuite = cipherSuite ?? _cipherSuite;
            if (!string.IsNullOrEmpty(pubKey)) PubKey = pubKey;
            if (!string.IsNullOrEmpty(privKey)) PrivateKey = privKey;
        }

        //// 藉由key string, 實例化
        public static BouncyCastleCrypto FromPubKey(string pubKey, string hashAlgorithm = null!, string cipherSuite = null!)
            => new BouncyCastleCrypto(pubKey, null!, hashAlgorithm, cipherSuite);

        public static BouncyCastleCrypto FromPrivKey(string privKey, string hashAlgorithm = null!, string cipherSuite = null!)
            => new BouncyCastleCrypto(null!, privKey, hashAlgorithm, cipherSuite);

        private (string pubKey, string privKey) GenerateKeyPair()
        {
            //// 動態建立keyPair
            var keyGen = new RsaKeyPairGenerator();
            keyGen.Init(new KeyGenerationParameters(new SecureRandom(), _keySize));
            var keyPair = keyGen.GenerateKeyPair();

            //// 密鑰以 PEM 格式存入 pubKey, privateKey 字串
            Func<AsymmetricKeyParameter, string> writeKey = (key) =>
            {
                var sw = new StringWriter();
                var pemWriter = new PemWriter(sw);
                pemWriter.WriteObject(key);
                pemWriter.Writer.Flush();
                return sw.ToString();
            };

            var pubKey = writeKey(keyPair.Public);
            var privateKey = writeKey(keyPair.Private);
            return (pubKey, privateKey);
        }

        //// 加解密
        public byte[] Encrypt(byte[] plainData)
        {
            var cipher = CipherUtilities.GetCipher(_cipherSuite);
            cipher.Init(true, AsymKeyParam);
            return cipher.DoFinal(plainData);
        }

        public byte[] Decrypt(byte[] cipherData)
        {
            var cipher = CipherUtilities.GetCipher(_cipherSuite);
            cipher.Init(false, AsymCipherKeyPair.Private);
            return cipher.DoFinal(cipherData);
        }

        //// 簽章
        public byte[] Sign(byte[] data)
        {
            var signer = SignerUtilities.GetSigner(_hashAlg);
            signer.Init(true, AsymCipherKeyPair.Private);
            signer.BlockUpdate(data, 0, data.Length);
            return signer.GenerateSignature();
        }

        public bool Verify(byte[] data, byte[] signature)
        {
            var signer = SignerUtilities.GetSigner(_hashAlg);
            signer.Init(false, AsymKeyParam);
            signer.BlockUpdate(data, 0, data.Length);
            return signer.VerifySignature(signature);
        }

        public void Dispose()
        {
        }
    }
}

```

測試

```csharp

using System.Drawing;
using System.Text;
using Bouncy_Castle;

Action<IRsaCrypto> RunTest = (crypto) =>
{
    Print(crypto.GetType().Name,ConsoleColor.Yellow);
    Print("加解密測試", ConsoleColor.Cyan);
    var plainText = "番茄頭";
    var plainData= Encoding.UTF8.GetBytes(plainText);
    Console.WriteLine($"明文: {plainText}");
    byte[] cipherData = null!;
    for(int i = 1; i< 3;i++)
    {
        Console.ForegroundColor = ConsoleColor.DarkBlue;
        cipherData = crypto.Encrypt(plainData);
        Console.WriteLine($"密文 : {Convert.ToBase64String(cipherData)}");
        var decryptedData = crypto.Decrypt(cipherData);
        Console.WriteLine($"解密第[{i}測]：{Encoding.UTF8.GetString(decryptedData)}");
        Console.ResetColor();
    };

    Print("簽章測試", ConsoleColor.Cyan);

    for (var i = 1; i < 3; i++)
    {
        Console.ForegroundColor = ConsoleColor.DarkBlue;
        Console.WriteLine($"簽章第[{i}測]：{plainText}");
        var signature = crypto.Sign(plainData);
        Console.WriteLine($"簽章第[{i}測]: {Convert.ToBase64String(signature)}");
        var verified = crypto.Verify(plainData, signature);
        Console.WriteLine($"驗證第[{i}測]: {(verified ? "PASS" : "FAIL")}");
        Console.ResetColor();
    }

    Print("===================================", ConsoleColor.Yellow);
};






RunTest(new NetCrypto());
RunTest(new BouncyCastleCrypto());


void Print(string msg, ConsoleColor color = ConsoleColor.White)
{
    Console.ForegroundColor = color;
    Console.WriteLine(msg);
    Console.ResetColor();
}

```

上面練習了一下別人寫的 Code，但可能不是自己遇到的狀況，目前還感覺不太出來使用 .NET 框架提供的方法與使用 BouncyCastle 的差異

這邊分享開頭引言提到的串接金流時遇到的情境

遙想當時，苦惱許久無法在最後一步做完簽章的動作，卡點是，今天讀進程式的 senderPrivateKey 是從 .key 檔案讀取 private key，但怎麼轉換格式都無法倒進 RSA 然後進一步在 Jose.JWT 製作 JWS

而最後是藉由 BouncyCastle 的 PrivateKeyFactory 製造 RSAPrivateKey Param ，Import進 RSA 容器後，才能成功

```csharp

    public static string GenerateJWSToken(string payload, string receiverPublicKey, string senderPrivateKey)
    {
        ////Get RSAPublicKey
        byte[] publicKeyBytes = Convert.FromBase64String(receiverPublicKey);
        X509Certificate2 receiverPublicCert = new X509Certificate2(publicKeyBytes);
        var receiverRSAPublicKey = receiverPublicCert.GetRSAPublicKey();

        ////Get RSAPrivateKey
        byte[] privateKeyBytes = Convert.FromBase64String(senderPrivateKey);
        RsaPrivateCrtKeyParameters privateKeyParam = (RsaPrivateCrtKeyParameters)PrivateKeyFactory.CreateKey(privateKeyBytes);
        RSAParameters rsaParameters = DotNetUtilities.ToRSAParameters(privateKeyParam);
        RSA rsa = RSA.Create();
        rsa.ImportParameters(rsaParameters);

        ////Encrypting and Signing
        var jweRequest = Jose.JWT.Encode(payload, receiverRSAPublicKey, JweAlgorithm.RSA_OAEP, JweEncryption.A256GCM, null);
        var jwsRequest = Jose.JWT.Encode(jweRequest, rsa, JwsAlgorithm.PS256, null);

        return jwsRequest;
    }

```

所以這邊先下個結論，BouncyCastle 讓我可以靈活的在我的煩惱中間跳進來，幫我轉換成我需要的某種 Class(在這邊就是 RsaPrivateCrtKeyParameters) 讓我可以繼續我的串接，這或許就是他的方便之處吧!

## 今日精神能量分析

精神能量 : 🪴🪴

今天發現 正妹 現在有 11 個夾子，但有一個焦掉了，是不是在同一盆裡面競爭太激烈阿，不太懂
