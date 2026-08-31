---
title: Asynchronous - 第一章:雲端中的未竟之事
date: 2024-05-07 11:04:05
categories: 未来よりの返歌
top_img: https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-1/1_landing.png
cover : https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-1/1_landing.png
tags:
    - Asynchronous Programming
toc:
toc_number:
comments :
---


{% tabs Asynchronous - 第一章:雲端中的未竟之事%}

<!-- tab 呼叫器-->


> 我拿著號碼呼叫器，坐在靠窗的位置。

咖啡廳裡播放著熟悉的爵士樂，氣味是熱牛奶與咖啡豆交融後的溫暖。呼叫器還沒響，但我確定店員剛剛有聽見我點了那杯熟成黑咖啡。

我不確定她現在是不是正在打奶泡，或是還在處理上一張訂單；
我也不確定我該不該起身確認一下。
但最終我選擇坐著，等她做好準備、等震動響起，等那杯「尚未完成的事」，被端到我手中。


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-1/3_await_coffee.png)


<!-- endtab -->

<!-- tab 🎵 章節一：傳訊以後的沉默 —— S3 上傳的非同步陷阱-->

這是一段將資料上傳到 S3 的程式碼
```csharp
private async Task UpdateS3DataAsync(string s3RecordData, string s3Path)
{
    using var stream = new MemoryStream(Encoding.UTF8.GetBytes(s3RecordData));
    var request = new PutObjectRequest { BucketName = this._bucketName, InputStream = stream, Key = s3Path };

    var response = await this._amazonS3.PutObjectAsync(request);
    if (response.HttpStatusCode != HttpStatusCode.OK)
    {
        throw new ApplicationException($"處理備份時發生錯誤 - 上傳至S3錯誤：{response.ResponseMetadata} StatusCode : {response.HttpStatusCode},備份失敗不可往下執行");
    }

    this._logger.LogInformation($"商品頁資料備份至S3新增完畢, 位置：{s3Path}");
}
```

測試了幾次發現，資料一直沒有帶成功上傳，也沒有錯誤，就這樣像是沒有執行過的程式碼一樣
問題不在這段程式碼本身，而在於呼叫端的態度 —— 沒有 await，沒有 GetAwaiter()，甚至連 Task.Result 都沒取，就這樣任由程式自己「走下去」。

```csharp
// 問題呼叫：完全沒有等待結果
_testService.UploadToS3Async(shopProductBadgePair.Key, productBadge.ProductBadge_Id, salepageIds);

```


呼叫端若忽略了 await，非同步邏輯就變成了放生的任務 —— 它會在背景默默執行，但主流程將完全不關心它是否完成、是否失敗，就像一封你永遠不會去確認已讀的訊息。因此，我們可以修正為以下方式來確保資料有上傳完成


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-1/2_disappear_upload_case.png)


<!-- endtab -->

<!-- tab 明確等待，確保任務完成-->

```csharp

//// 以同步的方式等待方法執行完成
await _testService.UploadToS3Async(shopProductBadgePair.Key, productBadge.ProductBadge_Id, salepageIds);
```
這是最簡潔、最直接的修正方式 —— await 就是對這個任務的一種承諾：「我等你，我不會繼續執行接下來的程式碼，直到你完成。」

<!-- endtab -->

<!-- tab 先執行、後等待，彈性處理任務進度-->


```csharp

//// 先交出不前的狀態，但呼叫端可以先繼續往下執行
Task task = _testService.UploadToS3Async(shopProductBadgePair.Key, productBadge.ProductBadge_Id, salepageIds);

//// ...做其他事情

//// 事情都最完了，等待最後非同步任務的結果
await task;
$"{task.IsCompleted}".Dump();


```
這種寫法給了你更多自由，你可以先發出請求、做點別的事情，等所有任務都在執行的路上，再一起收尾。適合需要進行多工處理或並行任務的場景。


<!-- endtab -->

<!-- tab 非同步方法其實長這樣-->

回頭來看看，我們到底是怎麼「傳出一段訊息」的？
非同步方法本身到底做了什麼？為什麼一定要 await 它？
根據 Microsoft 的官方說明，只要在方法前面加上 async 修飾詞，就會啟用下面兩項非常關鍵的功能：

<br>

{% note info %}

標記為 async 的方法內可以使用 await。這個 await 就像是故事的中場休息——它告訴編譯器：「這裡先不要急著往下走，等我拿到結果再繼續。」與此同時，控制權會被交還給呼叫它的上一層邏輯，讓系統可以繼續跑別的事情。
被 async 標記的方法，會自動回傳一個 Task（或 Task<T>）給呼叫端，也就是說：你可以 await 這個方法本身。
{% endnote %}

<br>

舉個例子來說，我們在上個章節中這樣寫：

```CSHARP
await this._amazonS3.PutObjectAsync(request);
```
這一行的意思是：「我把 request 送出去了，請你等我拿到結果再執行下面的程式碼。」
換句話說，await 就像是說：「這段話我說到一半，我會回來說完，你先忙別的去。」

整體流程以 Microsoft 官方提供的圖為例，這裡其實描述了整個非同步的生命週期
![Image](https://i.imgur.com/7CokFn9.png)

<br>

{% note info %}

第 6 步：當程式執行到 {% label await %}，它會先「記住目前的位置」（也就是等下回來繼續的地方），然後把控制權交還給呼叫端，讓其他任務可以先繼續跑。此時，非同步任務會在背景悄悄啟動（例如發出 API、寫檔、上傳 S3 等等）。

等到背景任務真的完成時，程式會「回到剛剛記住的地方」，繼續往下執行剩下的程式碼。
{% endnote %}


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-1/5_lifetime.png)


這就像是：

你去便利商店寄了一張明信片，寄完之後不會站在櫃檯等它被送達，你會先去做別的事，等郵差送達後，系統才會通知你「欸，有結果囉」，你再繼續後續處理。

- async：讓方法能「中途暫停」、「回傳 Task」
- await：告訴方法「請等它完成我才要往下做」
- Task：是一張「未完成的回應紙條」，你可以隨時等它完成，或是暫時放著做別的事

透過這組搭檔，我們就能讓程式邏輯在遇到等待的地方，暫停住不會阻塞整個流程，進一步實現真正的非同步處理。


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-1/4_async_await_task.png)



<!-- endtab -->

<!-- tab 你說等等，那我就等等-->

寫段小實驗來驗證 —— 你會發現， {% label await %} 並不會卡住整個程式，它只是放話：「你慢慢來，我先去忙別的事。」

```csharp
async void Main()
{
	var startTime = DateTime.Now.ToString("HH:mm:ss");

	Task task = LogAsync();

	var endTime = DateTime.Now.ToString("HH:mm:ss");
	$"{startTime} -- {endTime}".Dump();

    await task;
    $"完成沒 : {task.IsCompleted}".Dump();
}

public async Task LogAsync()
{
	await Task.Delay(5000);
	string path = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments), "log.txt");

	using (var writer = new StreamWriter(path, true))
	{
		await writer.WriteLineAsync(DateTime.Now.ToString("HH:mm:ss"));
	}
}
```
我們先紀錄起始時間，再呼叫 LogAsync()，但不立刻等它完成，而是先繼續往下執行並記錄時間。
然後才透過 await task，等待這個工作真正結束。

這時候你會發現，主 Thread 根本沒有停下來卡住，它很優雅地說：「我先記下這個任務，等你回來再處理你剩下的部分。」

這裡的 Task.Delay(5000) 代表非同步等待 5 秒鐘，不是睡覺、不是打瞌睡，是說：「我這 5 秒不會佔用主 Thread 資源，我只是『設定好』等待，然後主 Thread 你可以去做別的事情。」

寫入檔案的部分我們也加上了 await，原因是 StreamWriter.WriteLineAsync() 也是一個 I/O 操作，它可能會花時間等磁碟寫入完成，因此我們也要等它一下。

<br>

- Task task = LogAsync();
👉 啟動非同步任務，但不等它完成，像是說「你先去跑」。

- await task;
👉 這才是認真地說：「等等，把你剛剛沒做完的事情收尾一下。」

- "{startTime} -- {endTime}".Dump();
👉 是你觀察程式有沒有「阻塞」的重要證據。

這樣設計的好處是：你既可以自由安排執行順序，也可以在適當時機做收尾補位。

像一間有秩序的咖啡店：你點完單之後不是站在那等待還像個傻逼死盯著咖啡師做咖啡，而是拿著呼叫器去找位子，等震動再回來取飲料。await 就是那個呼叫器，溫柔而精準地通知你：「現在可以繼續了。」


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-1/6_get_task_and_fly.png)



<!-- endtab -->

<!-- tab 洗內褲的故事...-->

接著讓我們說一個洗內褲的故事...

```csharp
void Main()
{
	"準備來洗內褲囉!".Dump();
	Thread.Sleep(TimeSpan.FromSeconds(1));

	//// 建立洗內褲的任務
	var task = new Task<int>(() => 
	{
		int time = new Random().Next(1,5);
		Thread.Sleep(TimeSpan.FromSeconds(time));
		return time;
	});
	
	Thread.Sleep(TimeSpan.FromSeconds(1));
	"洗衣機開始洗內褲...".Dump();
	
    //// 開始洗吧 暫時不管他
	task.Start();
	Thread.Sleep(TimeSpan.FromSeconds(1));
	"先去看一下進擊的巨人，好在意結局阿".Dump();
	Thread.Sleep(TimeSpan.FromSeconds(3));
	
	"嗚嗚好感動，我的內褲呢?".Dump();
	Thread.Sleep(TimeSpan.FromSeconds(1));
	$"洗衣機 : {task.IsCompleted}".Dump();
	
    //// 確認洗完了嗎
	if(task.IsCompleted)
	{
		$"洗內褲洗了 : {task.Result} 秒".Dump();
	}
	else
	{
		"等等，快洗完了".Dump();
		task.Wait();
		$"洗內褲洗了 : {task.Result} 秒鐘".Dump();
	}
	
	"洗完了，曬一曬，吃麥當勞".Dump();
}

//// 準備來洗內褲囉!
//// 洗衣機開始洗內褲...
//// 先去看一下進擊的巨人，好在意結局阿
//// 嗚嗚好感動，阿內褲洗完了沒?
//// 洗衣機 : True
//// 洗內褲洗了 : 4 秒
//// 洗完了，曬一曬，來去吃麥當勞


```

想像你是執行緒本尊，一條誠懇又努力的主 Thread 
你每天負責處理重要邏輯，但偶爾也要負責一些大件事（比如洗內褲，尺寸 XXXXXXXL、滿是泥巴、極度資源密集），這種事如果你親自處理，就只能原地卡死。但如果你能派一個任務出去處理這件事（像是 new Task(() => ...)），你就能在等待的這段時間裡，去追劇、去 dump log、去感動、去活著。一個會安排等待、懂得資源切換的程式，才是真正的高效能。
![Image](https://i.imgur.com/K7a2j8n.png)

<!-- endtab -->

<!-- tab 用 Throughput 的眼光看非同步 —— 效率不是來自加速，而是來自不浪費-->

很多人學非同步時會問：「這樣程式會跑得比較快嗎？」

但事實上，非同步的核心價值不是讓單一任務變快，而是讓你在同一段時間內完成更多任務——這是產能（Throughput）的提升，而不是效能（Performance）的改變。

我們用工廠來想像一下：

假設一位工廠員工在製作產品 A。製作途中有個「焊接」步驟需要請外包來完成。這時，他把產品 A 交給外包，然後立刻開始製作產品 B，而不是站在原地等 A 回來。等外包完成後，把產品 A 送回工廠，這時再由另一位員工將它包裝完成。

這樣的流程中，產品 A、B 各自都花了一樣的製作時間，但因為我們不浪費等待的空檔，整體工廠在同一段時間內完成了更多產品，這就是非同步帶來的「Throughput 增加」。


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-1/throughout_and_performance.png)


<!-- endtab -->

<!-- tab 非同步的最佳戰場：I/O Bound vs CPU Bound-->

非同步最適合處理「I/O Bound 工作」—— 也就是那些大部分時間都在等別人回覆的事情。

例如：

- 從硬碟讀寫檔案
- 查資料庫
- 呼叫外部 API
- 與外部設備、服務通訊

這些操作都會讓 Thread 處於「等人回信」的狀態。如果你用同步方式處理，就會浪費一個 Thread 傻傻等著。
但如果你用非同步方式，就能在等待期間釋放該 Thread 去做別的事，等到回信來了再接著處理剩下的工作。

但如果今天是 CPU Bound 的工作就不同了—— 比如你要同時運算一堆複雜公式、解壓縮大量圖片或跑深度學習模型，這些都是「全程都在燒腦」的任務。這時 Thread 根本沒有閒著，也就沒有什麼好「釋放」的空間了，非同步的優勢自然就會變得不明顯。


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-1/8_if_async_need.png)



<!-- endtab -->

<!-- tab 那杯咖啡，終究會響起提示音-->


![f](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Codesss/Async/async-1/final.png)



程式的等待，不是站著發呆，也不是對任務的不信任。
它更像是 —— 拿著呼叫器，在咖啡廳找個角落坐下，
你知道咖啡師正在忙、奶泡正在打，你也知道：等到一切都準備好，那杯屬於你的咖啡，終究會震動提醒你「可以來取了」。

非同步程式設計教會我們的，是一種更成熟的節奏感：不要硬卡在原地，不要什麼事都自己扛，學會把長時間的任務交出去，讓資源被更有效地運用，然後 —— 在正確的時候，回頭、收尾、完成它。

這一路從 S3 上傳、LINQPad 實驗、洗內褲的故事、到 I/O 與 CPU Bound 的分析，我們看見的不是一堆語法或技術細節，而是：在程式裡，我們也可以安排一種從容，學會怎麼等待，也學會怎麼不等待。

所以，當你下次寫下 await，就像你走進熟悉的咖啡店，點了一杯熟成黑咖啡，轉身找個靠窗的位置坐下，你知道 —— 它在做事，你可以安心做別的，等咖啡響的那刻，再走過去，剛剛好。


<!-- endtab -->


{% endtabs %}


