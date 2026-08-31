---
title: OOM (一. Managed Heap & Generational Management)
date: 2024-06-04 21:12:10
categories: Others
top_img: https://i.imgur.com/AUBV7mo.png
cover : https://i.imgur.com/AUBV7mo.png
tags:
    - Memory
toc:
toc_number:
comments :
---

上班, 免不了要處理一些維運和監控的工作(請假)。然而,有時會遇到自己無法理解,也不知從何下手解決的異常情況,例如記憶體不足 (Out of Memory, OOM)。為了更好地應對這類問題,嘗試探索一下記憶體管理的相關知識,看看能夠理解到什麼程度

## Managed Heap

為了做到記憶體管理，在 .NET 中 當我們建立一個物件時，不是直接從 OS 的記憶體中分配，而是會從 .NET Runtime 已經預先分配的 Managed Heap 中分配記憶體。
舉凡一般的 Reference Type 操作以及資料庫連線，都在這個範圍內
Managed Heap 本質上是一長條的記憶體空間。.NET 會維護一個指向 Managed Heap 中下一個可用地址的指標。當 .NET 被要求創建一個新物件時，它會在這個地址分配所需的空間給這個物件，並將指標向前移動，以指向下一個可用地址。

Address    Size  Type
0x00ad21e0 16    System.Int32[]
0x00ad21f0 20    System.String
0x00ad2204 32    MyClass
0x00ad2224 12    System.Double

而 Unmanaged Heap 通常會牽涉到較底層的操作，例如 : Windows API、操作系統資料結構，甚至是 CLR 本身所需的記憶體、網路連接，這些就必須要自己去手動管理記憶體配置

## Generational Management

.NET 的 GC 使用了代管理來優化記憶體回收。物件根據它們的壽命被分配到不同的 Generation：

第 0 代（Generation 0, Gen 0）：新分配的物件。
第 1 代（Generation 1, Gen 1）：從 Gen 0 存活下來的物件。
第 2 代（Generation 2, Gen 2）：從 Gen 1 存活下來的物件，以及更長壽命的物件。

還有一個特殊的 Large Object Heap (LOH)，專門用來分配大於 85,000 bytes 的物件。


```csharp

void Main()
{
    Run_GenerationTest(); 
}

static void Run_GenerationTest()
{
    var small = new byte[84000];
    var large = new byte[90000];
    Console.WriteLine($"Small => G{GC.GetGeneration(small)}");
    Console.WriteLine($"Large => G{GC.GetGeneration(large)}");

    var test = new byte[1];
    Console.WriteLine($"Before G0 colection => G{GC.GetGeneration(test)}");
    GC.Collect(0);
    Console.WriteLine($"After 1st G0 collection => G{GC.GetGeneration(test)}");
    GC.Collect(0);
    Console.WriteLine($"After 2nd G0 collection => G{GC.GetGeneration(test)}");
    GC.Collect(1);
    Console.WriteLine($"After G1 collection => G{GC.GetGeneration(test)}");
    GC.Collect(2);
    Console.WriteLine($"After G2 collection => G{GC.GetGeneration(test)}");
}

```

所以不要隨便建一個肥而不用的物件阿


## 今日精神能量分析

精神能量 : 🕓

中午開會Q____Q

