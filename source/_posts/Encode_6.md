---
title: Url Encode
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


在瀏覽器輸入網址時，是否有注意過上面的編碼跟想像的不太一樣呢?


舉個例子:

有一個電商網站，允許用戶搜索商品，並對搜索結果進行分頁和排序

原始提交的搜尋:
https://www.example.com/search?q=電視機&brand=大同&price=1000-2000&page=1&sort=price_desc

編碼後:
https://www.example.com/search?q=%E9%9B%BB%E8%A6%96%E6%A9%9F&brand=%E5%B0%8F%E7%B1%B3&price=1000-2000&page=1&sort=price_desc


那如果直接把尚未編碼的 URL 在瀏覽器直接發 Request 會怎麼樣?

其實有一個很簡單的測試方法，直接在 Google 中文搜尋後，上面的瀏覽器 URL Bar 就會顯示:

https://www.google.com/search?q=火車快飛&sca_esv=88708a24d6188b8d&sca_upv=1&rlz=1C1CHBD_zh-TWTW1075TW1075&ei=9zlgZu-GDqOO2roPoamR-Qo&ved=0ahUKEwiv3vLjnMSGAxUjh1YBHaFUJK8Q4dUDCA8&uact=5&oq=火車快飛&gs_lp=Egxnd3Mtd2l6LXNlcnAiDOeBq-i7iuW_q-mjmzIFEC4YgAQyBRAAGIAEMgUQABiABDIFEAAYgAQyBRAAGIAEMgUQABiABDIFEAAYgAQyBRAAGIAEMgUQABiABDIFEAAYgARIqV5Q5QZY_ltwBngAkAEAmAGwAqAB6SSqAQgwLjIxLjUuMbgBA8gBAPgBAZgCE6ACiBaoAgDCAgoQABiwAxjWBBhHwgILEAAYgAQYsQMYgwHCAhEQLhiABBixAxjRAxiDARjHAcICCBAAGIAEGLEDwgIOEC4YgAQYsQMYgwEYigXCAgQQABgDwgIOEC4YgAQYsQMYgwEY1ALCAgsQLhiABBixAxjUAsICCBAuGIAEGLEDwgIOEAAYgAQYsQMYgwEYigXCAh0QLhiABBixAxiDARiKBRiXBRjcBBjeBBjgBNgBAcICBxAAGIAEGA3CAgcQLhiABBgNwgIIEAAYgAQYogTCAggQABiiBBiJBcICCBAuGIAEGNQCwgINEAAYgAQYsQMYQxiKBcICChAAGIAEGLEDGA3CAg0QABiABBixAxiDARgNmAMB4gMFEgExIECIBgGQBgq6BgYIARABGBSSBwYzLjEyLjSgB_hp&sclient=gws-wiz-serp

但 當你複製這一段到另外一個 瀏覽器 的 URL Bar，會發現它會自動做 Encode，所以答案是，不會怎麼樣它會自動做編碼!



## Url Encoding在做什麼?

URL encoding 的主要目的是確保數據能夠在 Web 上有一個一致、不會被誤解讀的標準

還記得，在第一篇講 Base64 時，有提到編碼本質上就像是翻譯成另一個國家的語言嗎?

URL Encode也是這麼回事，但 URL 為甚麼也需要編碼? 又為何需要編碼成 ASCII ?


## Why?

早期，當 URL 的設計者希望 URL 可以通過書面轉錄的方式傳遞時,他們需要確保 URL 中使用的字符都是可寫的,即能夠手寫或機打。由於 ASCII 字符集正好滿足這一要求,因此 URL 的構成字符被限定在 ASCII 字符集中。

然而，隨著 Web 的發展和國際化的需求，對 URL 中使用非 ASCII 字符的支持變得越來越重要。為了解決這個問題，URL 引入了百分號編碼 (Percent-encoding) 的機制，允許將非 ASCII 字符編碼為 ASCII 字符的形式。
通過百分號編碼，URL 可以包含各種語言的字符，同時仍然維持與 ASCII 字符集的兼容性。

使用百分符號的原因是甚麼，大概就是最大限度地減少歧義和誤解吧，較少情況下會用%做為某種功能用途

接著我們就會想，那我們要規範哪些字符需要做 Encode 哪些不用 ?

## What characters need to be encoded?

1.ASCII Control characters

例如 : NULL (空字符), Return, Escape, File Separator ...等，這些 ASCII 控制字符主要用於控制設備、格式化文本以及在通信協議中傳遞特殊指令。
然而，在 URL 中，這些字符可能會導致問題，因為它們是 Print 不出來的，並且在某些情況下可能會被解釋為特殊指令而不是普通字符。
為了避免潛在的問題，在建構 URL 時，建議對這些控制字符進行 "persent encoding "。例如，如果 URL 中需要包含一個換行符 (LF), 應該將其編碼為 "%0A"。這樣可以確保 URL 的正確性和可讀性，並避免這些控制字符被錯誤地解釋或處理。

2.Non-ASCII characters

例如 : 中日文， "嘴砲" 在 URL 中會被編碼為 "%E5%98%B4%E7%A0%B2"

3.Reserved characters

例如在 Url 本身就具有特殊功能的 "/","?","="...等都是保留字，如果有其他意義上的用途需要被編碼

4.Unsafe characters

空格和引號...等不安全字符通常也需要進行編碼,以確保 URL 的正確性和可讀性。這是因為這些字符在某些情況下可能會導致問題或被誤解

舉例:

未編碼: https://www.example.com/search?q=hello world
編碼後: https://www.example.com/search?q=hello%20world 或 https://www.example.com/search?q=hello+world

未編碼: https://www.example.com/search?q="hello world"
編碼後: https://www.example.com/search?q=%22hello%20world%22




可以用這個網站玩玩 : https://www.urldecoder.io/




下面再舉一個例子

## HTML form data in HTTP requests

當我們 Submit HTML form 時，form data 需要以一種標準化的格式發送到 Server。這個標準格式就是 application/x-www-form-urlencoded。
它規定了如何將 form data 編碼為一個字符串，以便在 HTTP 請求中正確的傳輸。

具體來說，當一個 HTML form 被提交時，瀏覽器會將 form 字段的名稱和值編碼為 key-value pairs , 並用 '&' 符號將它們連接起來。對於每個 key-value pair,key 和 value 之間用 '=' 符號分隔。例如：


name1=value1&name2=value2&name3=value3


在這個過程中，form 字段的名稱和值都會經過 URL encoding。這意味著特殊字符（如空格、'&'、'=' 等）和非 ASCII 字符都將被替換為相應的編碼形式。例如：

空格會被編碼為 %20
'&' 會被編碼為 %26
'=' 會被編碼為 %3D
 中文字符 ' 你好 ' 會被編碼為 % E4% BD% A0% E5% A5% BD

通過 URL encoding, form data 中的特殊字符和非 ASCII 字符就可以安全地在 HTTP 請求中傳輸，而不會被誤認為分隔符或其他特殊字符。



由於好奇心與感受性(很重要)，我們進一步再問， Encode 的做法是甚麼?

## How are characters URL encoded?

當我們說 URL 編碼將 "非 ASCII 字符" 轉換為 ASCII 字符的表示形式時，實際上是指 編碼後的 URL 只包含 ASCII 字符集中的字符。這是通過以下步驟做的:

1.對非 ASCII 字符進行 UTF-8 編碼:

首先，將非 ASCII 字符轉換為其在 UTF-8 編碼中的字節序列。

例如，字符 "é" 的 UTF-8 編碼為兩個字節：0xC3 和 0xA9。
中文字符 "你" 的 UTF-8 編碼為三個字節：0xE4、0xBD 和 0xA0。


2.將 UTF-8 字節轉換為十六進制表示:

將每個 UTF-8 字節的值轉換為兩位的十六進制數。
十六進制數使用數字 0-9 和字母 A-F 表示。

例如，字節 0xC3 轉換為十六進制數 "C3", 字節 0xA9 轉換為十六進制數 "A9"。
對於中文字符 "你", 字節 0xE4、0xBD 和 0xA0 分別轉換為十六進制數 "E4"、"BD" 和 "A0"。


3.將十六進制表示與百分號組合:

在每個十六進制數前加上百分號 ("%"), 形成 URL 編碼的表示形式。
例如，"C3" 變為 "% C3","A9" 變為 "% A9"。
對於中文字符 "你","E4"、"BD" 和 "A0" 分別變為 "% E4"、"% BD" 和 "% A0"。


4.組合編碼後的字符:

將百分號和十六進制數組合起來，形成完整的 URL 編碼表示。
例如，字符 "é" 的 URL 編碼為 "% C3% A9"。
中文字符 "你" 的 URL 編碼為 "% E4% BD% A0"。



最後非 ASCII 字符的 UTF-8 編碼值被轉換為了 ASCII 字符的表示形式。

雖然編碼後的 URL 包含百分號和十六進制數，但這些字符本身都屬於 ASCII 字符集。

重要的是要理解，URL 編碼的目的不是將非 ASCII 字符直接轉換為 ASCII 字符，而是將它們轉換為 ASCII 字符的表示形式。編碼後的 URL 本身只包含 ASCII 字符，但這些 ASCII 字符表示了原始 URL 中的非 ASCII 字符


## 今日精神能量分析

精神能量 : 🧗

最近平日都會健身，有健身教練帶似乎比較有能堅持的感覺