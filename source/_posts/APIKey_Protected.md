---
title: .NET CORE 環境變數
date: 2024-09-22 10:24:34
categories: Others
top_img: https://i.imgur.com/nQ2Epa9.png
cover : https://i.imgur.com/nQ2Epa9.png
tags:
    - 
toc:
toc_number:
comments :
---

# 將 APIKey 存進環境變數

參考 : https://blog.darkthread.net/blog/secure-apikey-for-console-app/

```CSHARP

byte[] additionalEntropy = { 9, 8, 7, 6, 5 };
Func<string, string> GetSecureEnvVar = (varName) => {
    var val = Environment.GetEnvironmentVariable(varName, EnvironmentVariableTarget.User);
    if (!string.IsNullOrEmpty(val))
    {
        try
        {
            val = Encoding.UTF8.GetString(
                ProtectedData.Unprotect(Convert.FromBase64String(val), additionalEntropy, DataProtectionScope.CurrentUser));
            return val;
        }
        catch
        {
            Console.WriteLine("非有效加密格式，請重新輸入");
        }
    }
    Console.Write($"請設定[{varName}]：");
    val = Console.ReadLine() ?? string.Empty;
    //加密後存入環境變數
    var enc =
        Convert.ToBase64String(
            ProtectedData.Protect(Encoding.UTF8.GetBytes(val), additionalEntropy, DataProtectionScope.CurrentUser));
    Environment.SetEnvironmentVariable(varName, enc, EnvironmentVariableTarget.User);
    return val;

```

拷貝了作法實際跑了一次確實可以 Work，缺點是 ProtectedData 這個 Class 是 Windows Data Protection API (DPAPI) 的一個封裝，專門為 Windows 設計。它使用 Windows 用戶帳戶相關的加密密鑰，因此在其他操作系統上不可用，並且EnvironmentVariableTarget.User 在所有主要操作系統上都存在，但 User 級別的環境變量主要是 Windows 的概念。

![Image](https://i.imgur.com/On4yApi.png)

因此自己這邊也實作了一次跨平台可以用的 AES加解密，但 Key 還要再另外存

```CSHARP

Func<string, string> GetAllPlatformSecureEnvVar = (varName) => {
    var val = Environment.GetEnvironmentVariable(varName, EnvironmentVariableTarget.User);
    if (!string.IsNullOrEmpty(val))
    {
        try
        {
            val = Decrypt(val);
            return val;
        }
        catch
        {
            Console.WriteLine("非有效加密格式，請重新輸入");
        }
    }
    Console.Write($"請設定[{varName}]：");
    val = Console.ReadLine() ?? string.Empty;
    var enc = Encrypt(val);
    Environment.SetEnvironmentVariable(varName, enc, EnvironmentVariableTarget.User);
    return val;
};

var myApiKey2 = GetAllPlatformSecureEnvVar("MY_API_KEY_2");
Console.WriteLine($"ApiKey 讀取測試成功 - {myApiKey2}");
}
static string Encrypt(string plainText)
{
using (var aes = Aes.Create())
{
    aes.Key = Encoding.UTF8.GetBytes(Environment.GetEnvironmentVariable("AESSecretTest", EnvironmentVariableTarget.User));
    aes.IV = new byte[16];

    using (var encryptor = aes.CreateEncryptor(aes.Key, aes.IV))
    using (var ms = new System.IO.MemoryStream())
    {
        using (var cs = new CryptoStream(ms, encryptor, CryptoStreamMode.Write))
        using (var sw = new System.IO.StreamWriter(cs))
        {
            sw.Write(plainText);
        }
        return Convert.ToBase64String(ms.ToArray());
    }
}
}

static string Decrypt(string cipherText)
{
using (var aes = Aes.Create())
{
    aes.Key = Encoding.UTF8.GetBytes(Environment.GetEnvironmentVariable("AESSecretTest", EnvironmentVariableTarget.User));
    aes.IV = new byte[16];

    using (var decryptor = aes.CreateDecryptor(aes.Key, aes.IV))
    using (var ms = new System.IO.MemoryStream(Convert.FromBase64String(cipherText)))
    using (var cs = new CryptoStream(ms, decryptor, CryptoStreamMode.Read))
    using (var sr = new System.IO.StreamReader(cs))
    {
        return sr.ReadToEnd();
    }
}
}

```

比較 AES 和 ProtectedData 的使用，兩者各有優缺點。讓我們深入分析一下：
AES（Advanced Encryption Standard）：

優點：

跨平台：可以在任何支持 .NET 的平台上使用。
靈活性：可以自定義密鑰和初始化向量（IV），適用於各種場景。
性能：通常比 ProtectedData 快。
廣泛支持：在需要跨系統或跨應用程序共享加密資料時更方便。

缺點：

密鑰管理：需要自行管理加密密鑰，這可能成為安全隱患。
實現複雜性：正確實現 AES 加密需要考慮多個安全因素，如 IV 管理、填充模式等。

ProtectedData：

優點：

簡單易用：API 簡單，不需要管理密鑰。
操作系統級別的安全性：利用 Windows 的內建安全機制。
用戶或機器特定：加密數據與特定用戶或機器綁定，增加了安全性。

缺點：

平台限制：主要適用於 Windows 系統。
可移植性差：加密資料難以在不同機器或用戶間移動。
有限的控制：無法自定義加密參數。


如果 App 只在 Windows 上運行，並且主要關注易用性和操作系統級別的安全性，使用 ProtectedData 是個好選擇。
如果需要跨平台兼容性，或者需要更多對加密過程的控制，那麼 AES 會更合適。


# ASPNETCORE_ENVIRONMENT

文件 : https://learn.microsoft.com/en-us/aspnet/core/fundamentals/environments?view=aspnetcore-8.0&WT.mc_id=DT-MVP-4015686

設定環境變數有幾種方式，有以下優先順序

1. .NET CLI 命令列參數: 
使用 .NET CLI 執行 ASP.NET Core 應用程式時，例如使用 dotnet run 命令，可以使用命令列參數覆蓋環境變數。這些參數的優先順序高於任何其他設定方式，包括環境變數本身和 launchSettings.json 檔案。

2. 程式碼設定: 在程式碼中使用 WebApplicationOptions.EnvironmentName 設定環境變數，這將會覆蓋其他設定方式，例如環境變數和 launchSettings.json 檔案。

3. launchSettings.json: 在本機開發環境中， launchSettings.json 檔案中的環境變數設定會覆蓋系統環境變數。

4. 作業系統環境變數: 您可以設定系統或使用者級別的環境變數，這些設定會影響所有使用該環境變數的應用程式。

5. Azure 應用程式服務設定: 如果您將 ASP.NET Core 應用程式部署到 Azure 應用程式服務，則可以在 Azure 入口網站中設定應用程式設定，這些設定相當於環境變數。
   
另外 ASPNETCORE_ENVIRONMENT vs. DOTNET_ENVIRONMENT 的情況

在大多數情況下， ASPNETCORE_ENVIRONMENT 環境變數的優先順序高於 DOTNET_ENVIRONMENT。
但是，當使用 WebApplicationBuilder 建立主機時， DOTNET_ENVIRONMENT 的優先順序會高於 ASPNETCORE_ENVIRONMENT。


# 環境變數說明、CLI設定環境變數

參考 : https://blog.miniasp.com/post/2022/05/28/Sum-up-ASPNETCORE-Environment-Variables


## 精神能量分析

精神能量 : 😺🐕

最近要買圍欄圍住毛吉