---
title: Hallucination
date: 2026-15-16 11:10:11
categories: AI
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/AI/landing.jpg
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/AI/landing.jpg
tags:
    - 
toc:
toc_number:
comments :
---

{% tabs Hallucination%}


<!-- tab 什麼是 AI 幻覺-->

AI 產生了看起來合理、語氣很肯定，但其實不正確、沒有根據，或與事實不一致的內容，更精確來說，生成式 AI 在缺乏可靠依據、理解錯誤、推理失準或資料不足的情況下，產生錯誤、虛構、誤導性或無法驗證的輸出


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/AI/hallucination/1_definition.png)



例如

- 編出不存在的論文、書籍、網址
- 說某個 API 有某個參數，但其實文件裡沒有
- 把兩個相似概念混在一起
- 明明不知道，卻用很肯定的語氣回答
- 對你的程式碼做出錯誤判斷
- 引用不存在的法律、規格、版本資訊
- 把「可能」講成「一定」



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/AI/hallucination/2_examples.png)


但現在的 AI 沒有聰明到可以「故意說謊」，事實上是模型在「預測下一段最像答案的文字」時，產生了形式上像答案、實質上卻錯誤的結果，大型語言模型的核心能力是根據上下文，預測接下來最合理的文字，但「最合理的文字」不等於「最真實的文字」，這就像一個人很會接話、很會寫文章、很會模仿專業語氣，但他其實他，一定真的知道答案


所以本質上它比較像是看過大量文字後，學會「什麼樣的問題通常會接什麼樣的回答」，當它不知道答案時，可能會根據語言模式補出一個「看起來合理」的答案


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/AI/hallucination/3_what_is_the_mech.png)




例如你問某某冷門 C# 套件的最新 API 怎麼用？

若模型記憶中沒有準確資料(因為這可能是你本地開發的東西)，它可能會根據其他類似套件的寫法，生出一段「很像官方用法」但其實不存在的程式碼


你問他：「XX 餐廳是不是米其林一星？」，他沒有查資料，但他看過很多美食文章，於是很自然地回答「是，它在 2023 年獲得米其林一星。」其實完全是編的

<!-- endtab -->

<!-- tab 不知道-->

就像某些情況人的行為一樣，他可能缺乏「不知道」本能，正常情況下如果不知道，通常會說我不確定，我查一下，而不是不懂裝懂，以人類來說，這可能出於某種情境下的壓力導致需要說謊，但 AI 的訓練是另外一回事，他就像從小被教導說，你出生的目標是要「給出有幫助的回答」，所以它有時會傾向變成補完答案，而不是停下來承認不確定


這也是為什麼 AI 幻覺很危險，很吃使用者的判斷能力


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/AI/hallucination/4_i_dont_know.png)


<!-- endtab -->


<!-- tab 要求它說明依據-->


當我們接收到一些資訊時，可以請他說明依據

所以 prompt 我們可以追問

- 你這個判斷的依據是什麼？
- 哪一段資訊支持這個結論？
- 這是你推論的，還是有明確來源？
- 你不確定的地方有哪些？


而這就是 AI 相對於人類的優勢之一，你不需要考慮他的感受，**他不會因為覺得自己被質疑而暴躁氣餒**


![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/AI/hallucination/5_no_feeling.png)


例如說 AI 原本說這個 API 支援 includeDetails=true

我們可以追問

這個參數是官方文件明確寫的嗎？
還是你根據類似 API 推測的？

如果它開始說「可能」、「通常」、「類似情境下」，那就代表這其實只是不確定的事



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/AI/hallucination/6_how_to_handle.png)



<!-- endtab -->


<!-- tab 把問題改成「可驗證」的形式-->

有時候 AI 幻覺，是因為問題太大、太抽象，例如我們問

```bash
這段程式是不是有問題？

```
因為問題比較泛化，因此 AI 可能會從不同的面相猜測

我們會說比較好的問法是

```bash
這段 C# 程式在 user == null 時會不會拋 NullReferenceException？請逐行分析，不確定的地方要標出來，不要假設我沒提供的程式碼
```

這樣它比較不容易亂補


<!-- endtab -->


<!-- tab 列出不確定性評級-->


我們可以要求 AI 回答時時分成「確定」、「推測」，以供我們對資訊的判斷先有初步的認知

例如

## 確定
...

## 推測
...


因為很多時候 AI 的價值不是直接給答案，而是幫你整理哪些是已知事實、哪些是合理猜測、下一步該查什麼



![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/AI/hallucination/7_categorize.png)


<!-- endtab -->



<!-- tab 提供實測方法-->


如果是程式問題，最好的方式是讓答案可以被測試，不要烙費時間跟他辯論

例如

- 寫 unit test
- 寫最小重現範例
- 查官方 source code
- 跑一次 command
- 打 API 看實際 response
- 查 log / DB / trace
- 比對 package version


我們可以把 AI 當作幫你設計驗證方法的人，例如問它

- 你剛剛的說法要怎麼驗證？
- 請給我一個最小可重現範例
- 請列出我應該查哪些 log 欄位
- 請給我 SQL / curl / unit test 來驗證

這會比單純問「你確定嗎」更有效




![2](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/AI/hallucination/8_tests.png)


<!-- endtab -->

{% endtabs %}