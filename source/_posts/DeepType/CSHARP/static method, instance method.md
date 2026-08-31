---
title: static method, instance method
date: 2025-11-02 11:27:05
categories: 程思舞想
top_img: https://i.imgur.com/Zly1UDM.png
cover : https://i.imgur.com/Zly1UDM.png
tags:
    - 程思舞想
toc:
toc_number:
comments :
---

{% tabs static method vs instance method%}


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/instance_static/landing.png)

<!-- tab behaviors-->


有些事，必須「由我親自完成」。像是「寄信給某個人」、「打開我家的門」、「改變自己的心情」。這些行為，都需要知道**「我是誰」** ， 要有具體的對象，行為才有意義

![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/instance_static/if_should_know_me.png)


在程式裡，這樣的行為就像 實例方法（Instance Method）。它依附在一個物件上，只有當那個物件存在（例如某個字串、某個人、某個訂單），方法才能運作，因為它需要用到那個物件的內部狀態

但有些事，誰都能做。像是「把字變成大寫」、「計算兩個數的和」、「取得目前時間」。它們不需要知道「我是誰」，只要照著規則執行，就能得到相同的結果

![s](https://github.com/CHI-KEKE/pics/blob/main/Code_Design/instance_static/instance_and_static.png?raw=true)


<!-- endtab -->


<!-- tab 設計核心原則：是否依賴物件狀態-->


| 類型                         | 是否依賴物件狀態 | 範例                   | 設計意圖                 |
| -------------------------- | -------- | -------------------- | -------------------- |
| **實例方法 (Instance Method)** | ✅ 是      | `string.Substring()` | 方法的邏輯需要使用「該物件的資料」。   |
| **靜態方法 (Static Method)**   | ❌ 否      | `char.ToUpper()`     | 方法不依賴任何物件，只根據輸入參數運算。 |


![a](https://github.com/CHI-KEKE/pics/blob/main/Code_Design/instance_static/if_rely_on_state.png?raw=true)

<!-- endtab -->


<!-- tab 實例方法 : string.Substring()-->


因為它要「從這個字串」取出一部分，也就是它需要知道目前這個字串的內容。它的邏輯會使用 this（也就是目前字串的實例），像這樣

```csharp
public string Substring(int startIndex, int length)
{
    // 內部其實會使用 this.Length, this._firstChar 等欄位
    return new string(this._firstChar, startIndex, length);
}
```

沒有 this，就不知道要從哪一個字串取子字串。所以它必須是實例方法


![this](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/instance_static/substring.png)

<!-- endtab -->


<!-- tab 靜態方法 :　char.ToUpper()-->


char 是一個 value type (struct)，代表一個「單一字元」。把字元轉成大寫這個動作，不需要知道「這個字元所在的物件」或「任何內部狀態」，只要輸入一個字元，回傳一個新的字元即可

```csharp
public static char ToUpper(char c)
{
    if (c >= 'a' && c <= 'z')
        return (char)(c - 32);
    return c;
}
```

它不依賴於任何「實例」的資料，所以設計成靜態方法更合理、更輕量（不需 new 或產生實例）

![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/instance_static/toupper.png)

![d](https://github.com/CHI-KEKE/pics/blob/main/Code_Design/instance_static/char-32.png?raw=true)

<!-- endtab -->


<!-- tab 總結-->

![t](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/instance_static/decision_table.png)


| 判斷問題                    | 建議方法型態 |
| ----------------------- | ------ |
| 方法會使用到 `this`（物件內部資料）| ➤ 實例方法 |
| 方法僅根據傳入參數運算            | ➤ 靜態方法 |
| 方法需要被覆寫（多型）           | ➤ 實例方法 |
| 方法屬於整個類別的共同行為          | ➤ 靜態方法 |
| 是否需要在沒有任何實例的情況下使用      | ➤ 靜態方法 |



靜態方法，是「工具」；實例方法，是「行為」。當一個動作需要了解「我是誰」，它屬於物件；
當一個動作不需知道「我是誰」，它屬於類別。就像現實生活中，有些事要靠自己去完成（instance），有些事，只要依照規則，就能重現相同結果（static）


![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/instance_static/slogan.png)


![fj](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/instance_static/summa.png)

<!-- endtab -->


{% endtabs %}