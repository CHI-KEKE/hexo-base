---
title: Go
date: 2026-04-04 08:54:05
categories: Docker
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Random/Whale.png
tags:
    - Docker
toc:
toc_number:
comments :
---


{% tabs Go%}

<!-- tab Why-->

Docker 生態系的許多核心專案，例如

`Docker Engine`
`containerd`
`runc`

主體實作也大多以 Go 為主，這是因為 Go 很適合拿來開發這種「系統工具、伺服器程式、基礎設施軟體」


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/go/1_three_component_from_go.png)


Docker 這類軟體有幾個很重要的需求

- 能在不同平台運作
- 容易編譯與發佈
- 能處理大量並發工作
- 盡量降低部署與執行時的依賴
- 方便大型團隊長期維護

Go 在這幾件事情之間取得了平衡，所以非常適合 Docker 這種專案



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/go/2_what_infra_needs.png)


<!-- endtab -->


<!-- tab 編譯-->


編譯就是把人寫得懂的程式，整理成電腦能有效執行的結果，順便提早抓出一堆錯


平常寫的程式碼像這樣

```go
fmt.Println("Hello")
```

or


```csharp
Console.WriteLine("Hello");
```

這些文字是給人看的，不是 CPU 直接吃的，編譯器會做幾件事

- 檢查語法有沒有寫錯
- 檢查型別對不對
- 把程式整理成可執行的形式
- 幫你做一些優化

如果不經過編譯，很多錯要等到執行時才爆。先編譯的好處是很多問題在上線前就先攔下來


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/go/3_compiler.png)



<!-- endtab -->


<!-- tab 交叉編譯體驗-->

Go 可以在一個平台上，編譯出另一個平台可執行的程式

這件事叫做 交叉編譯（Cross Compilation）

例如，你現在是在 Windows 上開發，但想產生一個給 Linux 執行的程式，Go 通常可以直接做到

Go 交叉編譯範例
```bash
GOOS=linux ## 指定目標作業系統是 Linux
GOARCH=amd64 ## 指定目標 CPU 架構是 amd64（也就是常見的 x86_64）
go build ## 根據你設定的目標平台，編譯出對應的執行檔
```


而編譯產物常見是單一 binary，表示


## 交付簡單

我們只要把一個檔案給別人例如

```bash
myapp
```

或 Windows 上

```bash
myapp.exe
```


不用再另外附某個 runtime、一堆依賴套件、額外的啟動方式說明


## 部署簡單

上傳一個檔案到伺服器，給執行權限就能跑

```bash
chmod +x myapp
./myapp

```

這對 CLI 工具、微服務、批次工具都很方便


## 容器容易做小

Docker 裡你只要把 binary 丟進去就能跑。不用先裝一整套大型 runtime，映像檔通常更精簡



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/go/4_cross_compile.png)





而 .NET 跨平台也沒問題，然而麻煩點在於每次都要先想清楚


我要省空間、省安裝、還是省維運


C# / .NET 能跑在 Windows、Linux、macOS，但在「怎麼交付程式」這件事上有比較多分支


## Framework-dependent deployment

你的程式本體比較小，因為不把 runtime 打包進去。但目標機器要先安裝對應版本的 .NET

適合公司內部所有機器都已經裝好 .NET，並且想讓發佈檔案小一點，但維護 runtime 版本需有統一規範

## Self-contained deployment

這個方式會把 runtime 一起包進去，但表示目標機器通常不用另外安裝 .NET

適合無法控制使用者電腦環境，並且想降低「少裝一個東西就不能跑」的風險，拿到就比較容易直接跑，但檔案比較大


## Single-file

把東西盡量包成單一檔案，方便交付，不過實際行為仍要看設定與平台，背後不一定真的完全沒有其他依賴

## Native AOT / Trim

進一步把成品做得更快啟動、更小，但不是所有程式都能輕鬆套，尤其碰到反射、動態載入套件時，處理起來就會變複雜



Go 比較像「編譯完就帶走」，而 .NET 比較像「你先選發佈策略，再決定怎麼帶走」


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/go/6_net_deploy_strategy.png)


<!-- endtab -->



<!-- tab runtime-->

runtime 可以想成程式執行時的工作現場，少了它，程式像有零件圖卻沒有工廠。

runtime 是讓程式在執行時能正常運作的一套環境。它可能包含

- 記憶體管理
- 垃圾回收
- 型別系統支援
- 例外處理
- 基礎函式庫
- 執行中介碼的能力

在 .NET 裡，C# 程式會依賴 .NET runtime 來幫忙執行

而 Go 很多時候編譯後就把需要的東西整合進去了，執行端不太需要再補一個完整的大環境


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/go/5_runtime.png)


<!-- endtab -->

<!-- tab 效能優勢-->


Go 在 build 時就把主要程式編成目標平台可跑的機器碼，所以一般情況下，執行檔拿來就跑，不需要像 `.NET` 那樣主要依賴執行期 `JIT` 來把 `IL` 再編成機器碼。Go 官方 FAQ 也提到，`Go binary` 會把 Go runtime 一起帶進去


Go 在 run 的時候，通常不會再針對熱點函式做「第二次編譯優化」。相對來說，`.NET` 的 `tiered compilation` 會先 quick JIT，再把熱門方法升級成更優化版本，這是執行期持續優化的一部分。


以 `.NET` 來說

```BASH
dotnet build
dotnet run
```

`build` 通常先產生 `IL`，`run` 後 `JIT` 才開始把方法編成 machine code，而且熱門方法之後還可能再做更高等級最佳化。這就是 `.NET` 官方文件講的 `tiered compilation`


`Go`：偏 build-time 定案
`.NET`：偏 run-time 持續優化



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/go/7_buildtime_runtime.png)


<!-- endtab -->

<!-- tab 靜態連結 vs 動態連結-->


程式執行時需要的函式庫，是先一起包進去，還是到目標電腦上再找

寫好程式後，程式裡會用到很多函式庫，像字串處理、網路、加密、系統功能。經過編譯之後，還有一步叫做 link / 連結，要決定這些函式庫怎麼接進你的程式
這時就有兩種常見作法


## 靜態連結

把程式需要的函式庫內容，直接打進可執行檔裡。

結果就是可執行檔比較大，但拿到別台機器時，比較容易直接跑，因為很多東西已經包在裡面了


##　動態連結


可執行檔本身不把函式庫整個塞進去，只記得「執行時去系統上找某個共享函式庫」


結果就是

- 可執行檔比較小
- 多個程式可以共用同一套函式庫
- 但目標機器如果缺檔案、版本不對，就可能跑不起來


Docker 這種工具本身就要被安裝到很多環境上，假如它自己還很依賴複雜 runtime 或很多外部依賴，發佈和維護都會更麻煩。像 docker、一些 Docker CLI plugin、還有獨立工具，官方文件常直接叫你下載一個 binary，放到指定目錄就能用。像 Docker Scout 的安裝方式就是把 docker-scout binary 放進 plugin 目錄後設成可執行。而這種發佈方式和 Go 很搭，因為 Go 很常就是

編完一顆 binary，丟到機器上，直接跑



![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/go/8_static_link_dynamic_link.png)


<!-- endtab -->



<!-- tab 靜態連結 vs 動態連結-->


![dockerfile](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Devops/Docker/go/9_final.png)


<!-- endtab -->


{% endtabs %}
