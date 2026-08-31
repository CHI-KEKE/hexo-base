---
title: Dependency Injection - 測試
date: 2025-05-13 22:07:05
categories: 程思舞想
top_img: https://i.imgur.com/xAbpKgd.png
cover : https://i.imgur.com/xAbpKgd.png
tags:
    - Dependency Injection
toc:
toc_number:
comments :
---

## 測試


to Feng，這篇介紹 Autofac 觀念的文章 https://blog.darkthread.net/blog/autofac-notes-1/ ，有舉了一個範例，在完全不修改主要程式的情況下抽換不同功能的元件。如果要做單元測試，這超級重要。

例如：某個依賴 WebApi 提供內容的函式，單元測試期間需模擬不同 WebAPI 傳回結果，此時可透過 DI 將 WebAPI 服務元抽換成假元件傳回特定值以驗證不同狀況下邏輯正確。範例：https://blog.darkthread.net/blog/aspnet-core-di-practice/