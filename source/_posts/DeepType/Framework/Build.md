---
title: 編譯
date: 2026-05-01 08:28:05
categories: Framework
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Framework/Build/landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Framework/Build/landing.png
tags:
    - Framework
toc:
toc_number:
comments :
---


{% tabs 編譯%}

<!-- tab bin / obj-->

當我們按下「Build」時...


編譯器先把 .cs 程式碼轉成中間結果放在 obj 資料夾，在 obj 裡面會產生有

- 暫存 DLL
- 編譯中間檔
- 自動產生的程式碼（例如 Razor）

當確認所有東西都沒問題後，才會輸出最終結果到 bin 而裡面會有

- .dll（主要程式）
- .exe（如果是應用程式）
- 相依套件 DLL（NuGet）
- 設定檔


也就是說真正執行網站時用的是 bin，而編譯過程中產生一堆暫存在 obj



![1](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Framework/Build/1_obj_bin.png)



有時候我們會看到刪掉 bin / obj 的建議來解決版本問題，例如我們若遇到錯誤 ReflectionTypeLoadException，是因為舊的編譯垃圾如果還被拿來用，很可能 bin 裡殘留舊版 DLL、新版 NuGet 已經下載，但舊 DLL 還在、執行時載到錯的版本，而我們刪掉它，就會強制全部重新編譯 + 重新產生正確 DLL



![1](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Framework/Build/2_delete_bin.png)


<!-- endtab -->



<!-- tab csproj Include-->


首先，以編譯器本人的角度來說，他不會自己去「找檔案」，它只會編譯「專案明確告訴它要編譯的清單」


當我們啟動 Build，其實是啟動 MSBuild，而 MSBuild 會先讀 .csproj 檔，這個檔案是一份「建置腳本」（不是單純設定檔），而在 .csproj 裡有這些東西

```csharp
<ItemGroup>
  <Compile Include="A.cs" />
  <Compile Include="B.cs" />
</ItemGroup>
```

這代表一件事「只有這些檔案要被拿去編譯」，MSBuild 會把這些 <Compile> 項目整理成一個清單，接著呼叫 C# 編譯器（csc.exe）


舊 .NET Framework 的設計理念是完全掌控編譯內容，只是其中代價就是容易漏檔案，因為專案要編譯什麼必須明確寫出來



![1](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Framework/Build/3_msbuild_and_list.png)


## .NET CORE


現代專案改成「預設全抓」，因為少犯錯比精細控制更重要，SDK-style .csproj 其實內建一條規則


```csharp
<Compile Include="**\*.cs" />
```

意思是整個專案底下所有 .cs → 全部自動加入，因此當我們新增檔案時，直接新增就能用



![1](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Framework/Build/4_sdk_is_get_all.png)




所謂的 什麼是 SDK-style .csproj 是說，它把原本冗長的建置設定「收進一個 SDK 模板」，讓你只寫差異，其他全部用預設幫你做好。當我們開 .csproj 檔案時，會看到

```csharp
<Project Sdk="Microsoft.NET.Sdk">
```

這就是 SDK-style 的標誌，而這 SDK 其實在安裝 .NET SDK 時就已經存在了



![1](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Framework/Build/dotnet-sdk.png)




## type could not be found


有時候我們會遇到這個錯誤，但可能性有幾個

- using 錯？
- namespace 錯？
- reference 壞掉？

但實際上有可能是我們在開發時，那個型別根本沒被編進 dll!



![1](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Framework/Build/6_type_could_not_be_found.png)



## Visual Studio


在 Visual Studio 中透過「加入新項目」或「加入現有項目」新增檔案時，VS 會自動在 csproj 補上 `<Compile Include>`


1. Solution Explorer → 找到對應專案
2. 右鍵 → 加入 → 現有項目
3. 選取該 `.cs` 檔案

而這就是 IDE 的方便之處


![1](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Framework/Build/5_vs_autoadd.png)



<!-- endtab -->

<!-- tab 版本錯誤案例解析-->


## packages.config

`packages.config` 是 .NET Framework 專案（舊式 csproj）用來記錄 **NuGet 套件相依關係**的 XML 檔案，位於每個專案的根目錄


```xml
<?xml version="1.0" encoding="utf-8"?>
<packages>
  <package id="Newtonsoft.Json" version="12.0.3" targetFramework="net45" />
  <package id="Microsoft.AspNet.Mvc" version="4.0.40804.0" targetFramework="net45" />
  <package id="Microsoft.AspNet.Razor" version="2.0.20710.0" targetFramework="net45" />
  <package id="RazorEngine" version="3.10.0" targetFramework="net45" />
</packages>
```

每個 `<package>` 記錄

- `id`：NuGet 套件名稱
- `version`：確切版本號
- `targetFramework`：目標 .NET 版本



## packages.config vs PackageReference（.NET Core）


| 項目 | packages.config（.NET Framework） | PackageReference（.NET Core / .NET 5+） |
|------|----------------------------------|----------------------------------------|
| 格式 | 獨立的 XML 檔案 | 直接寫在 `.csproj` 裡 |
| 套件存放 | `packages\` 資料夾（在 repo 旁邊） | NuGet 全域快取（`~/.nuget/packages`） |
| 相依解析 | 每個專案獨立列出所有直接 + 間接相依 | 自動解析，只需列直接相依 |
| 版本衝突處理 | 手動 binding redirect | 自動統一版本 |


## packages\ 資料夾


`packages.config` 宣告的套件會被下載到 solution 根目錄的 `packages\` 資料夾：

```bash
nineyi.scm.nmqv2\
  packages\
    Microsoft.AspNet.Razor.2.0.20710.0\
      lib\
        net40\
          System.Web.Razor.dll    ← Razor 2.0
    Microsoft.AspNet.Razor.3.0.0\
      lib\
        net45\
          System.Web.Razor.dll    ← Razor 3.0
    RazorEngine.3.10.0\
      ...
```



**注意**：不同版本的套件可以同時存在 packages 資料夾，但 build 時只有一個版本的 dll 會進 bin





![1](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Framework/Build/7_packagejson_packreference.png)




## 版本衝突是怎麼發生的


案例

```bash
整個 solution 的 packages.config 分布：

ERP\Backend\BE\packages.config
  → Microsoft.AspNet.Razor 2.0   ← 大多數專案

ERP\Backend\BLV2\packages.config
  → RazorEngine 3.10.0
  → Microsoft.AspNet.Razor 3.0   ← 唯一用 3.0 的專案

Build SCM Frontend NMQV2 時：
  → 兩個 Razor 版本都被複製到 bin
  → 3.0 蓋掉 2.0（後複製者獲勝）
  → MVC 4.0 還在（用 Razor 2.0 編譯）
  → 衝突形成
```



## 分析 

- `packages.config` = 這個專案用了哪些 NuGet 套件、哪個版本
- 每個專案有自己的 `packages.config`，同一 solution 不同專案可以宣告同一套件的不同版本
- Build 時所有 dll 複製到同一個 bin，**版本衝突是無聲的**，不會有編譯錯誤
- 版本衝突只在 runtime 才爆出來，而且通常是難以追蹤的環境相關錯誤
- .NET Core / .NET 5+ 的 PackageReference 格式改善了這個問題，版本解析更智慧，因為他把「每個專案各自決定版本」改成「整個建置過程統一算出一個一致版本」，衝突在 build 階段就被攔下來


在舊的 packages.config 模式，每個專案都有自己的套件版本清單，Build 時只是「各自把 DLL 複製到 bin」


```bash
A 專案 → Newtonsoft.Json 10
B 專案 → Newtonsoft.Json 12
```


最後 bin 裡可能只剩一個版本（誰蓋掉誰不一定）



![1](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Framework/Build/8_how_version_conflict_happen.png)


而在 PackageReference 模式（.NET Core / .NET 5+），套件直接寫在 .csproj


```csharp
<ItemGroup>
  <PackageReference Include="Newtonsoft.Json" Version="13.0.1" />
</ItemGroup>
```


![1](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Framework/Build/8_conflic_version_fix.png)


Build 時，MSBuild 會先「解析整個依賴樹」（dependency graph）包含直接引用的套件、套件依賴的套件（transitive dependencies），接著做「版本決策」（dependency resolution），同一個套件只能選一個版本，而常規則是優先選「最高相容版本」或遵守你明確指定的版本，如果版本衝突會導致無法解析直接 build 失敗，最後才還原（restore）套件並編譯，而且 DLL 不再隨便散在專案資料夾，而是集中在全域快取（global packages folder）

```bash
Step1：偵測到 Razor 2.0 vs 3.0
Step2：嘗試統一版本（通常選 3.0）
Step3：
   如果 MVC 相容 → OK
   如果 MVC 不相容 → ❌ build fail
Step4：bin 只有一個 Razor
```


<!-- endtab -->







{% endtabs %}