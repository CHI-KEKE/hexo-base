---
title: DateTimeOffset
date: 2024-05-12 21:23:05
categories: 
top_img: https://github.com/CHI-KEKE/pics/blob/main/Codesss/Time/clock-wolf.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/Codesss/Time/clock-wolf.png?raw=true
tags:
    - 
toc:
toc_number:
comments :
---



{% tabs 資料庫存 UTC%}

<!-- tab 煩惱-->

工作上時不時就會碰到時間的處理

DB 該存甚麼時間? 

部署到不同環境會不會有時區問題?

收發推播會不會因為時間處理不周導致收發有問題?


<!-- endtab -->

<!-- tab DateTime.Now and DateTime.UtcNow-->

首先我們先簡單確認一下這兩者在我們台灣的本機執行會印出什麼時間:

```csharp

	DateTime.Now.Dump();   //// 5/12/2024 9:16:01 PM
	DateTime.UtcNow.Dump(); //// 5/12/2024 1:16:01 PM

```

DateTime.Now 給的是那台機器的 Local Time，所以我在我的本機執行，抬頭看看時鐘，嗯嗯一模一樣

(我曾經有在 Code 裡面看到 DateTime.Now.ToLocalTime() 的寫法..)

DateTime.UtcNow 給的是目前的世界標準時間，而台灣、香港都比標準時間所在的位置快了 8 個小時，所以會得到比時鐘少 8 小時的結果


<!-- endtab -->

<!-- tab DateTime.Kind-->

承襲上一段，雖然好像理所當然，但假如這個時間在程式碼中經歷了百般轉折傳到你的 Service 時，你要怎麼判斷他是 UTC 還是 LocalTime 呢

```csharp
	var time1 = DateTime.Now.Dump();   //// 5/12/2024 9:16:01 PM
	var time2 = DateTime.UtcNow.Dump(); //// 5/12/2024 1:16:01 PM
	
	
	time1.Kind.Dump(); // Local
	time2.Kind.Dump(); // Utc
```

<!-- endtab -->

<!-- tab 時區轉換-->

假如，今天我們與前端溝通好，優惠券傳過來的時間格式一率使用 Utc 時間，到我們的 API 時，我想要檢查這檔優惠券設定的過期時間是否合理，我們應該要統一時區做比較

```csharp
	var expiredTimeJson = JsonConvert.SerializeObject(DateTime.Today.AddDays(-1).ToUniversalTime());
	var expiredTime = JsonConvert.DeserializeObject<DateTime>(expiredTimeJson);
	
	if(expiredTime.ToUniversalTime() < DateTime.Now.ToUniversalTime())
	{
		"過期時間不得小於現在，請重新設定!".Dump();
	}
	else
	{
		"設定成功".Dump();
	}

```


<!-- endtab -->

<!-- tab 日期轉字串-->

舉個例子，當我們想在 return message 註記一個明確的時間格式，很常看到的做法就是 ToString()

參考 : https://www.c-sharpcorner.com/blogs/date-and-time-format-in-c-sharp-programming1
![Image](https://i.imgur.com/lKsjXni.png)

```csharp

var message = $" 訂單更新時間 : {result.OrderUpdatedDateTime.ToString("yyyy/MM/dd HH:mm:ss")}";

```

值得注意的是，如果單純 ToString() 而不指定格式，會使用當下執行續的 CultureInfo (文化特性)

不同的 CultureInfo 會得到甚麼結果
```csharp
	var date = DateTime.Now;
	date.ToString().Dump(); // 5/12/2024 11:06:46 PM (應該是我系統語言調成英文)
	date.ToString(new CultureInfo("en-us")).Dump(); // 5/12/2024 11:06:15 PM
	date.ToString(new CultureInfo("zh-tw")).Dump(); // 2024/5/12 下午 11:06:15
	date.ToString(new CultureInfo("zh-hk")).Dump(); // 12/5/2024 下午 11:06:15
	date.ToString(new CultureInfo("ms-MY")).Dump(); // 12/05/2024 11:06:15 PTG
```


<!-- endtab -->

<!-- tab 字串轉 DateTime-->

有兩種模式，DateTime.Parse & DateTime.ParseExact，通常建議使用後者較好除錯

ParseExact 指定特定的時間格式
```csharp

void Main()
{
	string externalDateString = "2024/05/12";
	string externalDateString2 = "2024-05-12";
	
	//// ParseExact
	DateTime parsedDate = ParseExternalDate(externalDateString);
	Console.WriteLine(parsedDate); // 輸出: 5/12/2024 12:00:00 AM

	DateTime parsedDate2 = ParseExternalDate(externalDateString2); // FormatException : String '2024-05-12' was not recognized as a valid DateTime.
	Console.WriteLine(parsedDate2);
	
	
	//// Parse (格式不一樣也給過，較容易有預期之外的結果)
	DateTime parseDate3 = DateTime.Parse(externalDateString2);
	Console.WriteLine(parseDate3); // 輸出: 5/12/2024 12:00:00 AM

}

public DateTime ParseExternalDate(string dateString)
{
	string format = "yyyy/MM/dd";
	DateTime parsedDate = DateTime.ParseExact(dateString, format, CultureInfo.InvariantCulture);
	return parsedDate;
}


```

<!-- endtab -->

<!-- tab DateTimeOffset-->

最後談到 DateTimeOffset，必須要推薦一篇串文

參考文章 : https://stackoverflow.com/questions/4331189/datetime-vs-datetimeoffset

講者生動地做了一個 DateTime vs DateTimeOffset 的比喻，感覺應該是個滿好的 mentor

我這裡也帶來一個故事試著說明，DateTimeOffset 這個工具解決了甚麼問題

從前有一跨國公司，他們有一個困擾了很久的問題：如何讓分佈在不同時區的員工準確地參與國際會議。

<!-- endtab -->

<!-- tab 會議邀約-->





### 第一場會議邀約

某天，公司的總部（位於倫敦，UTC 時區）安排了一場重要的全球會議，時間定為 1 月 1 日下午 3 點。他們通知了所有分公司的負責人，但問題就這麼來了 ——

Tokyo 分公司（UTC+9）的員工收到通知後，系統只記錄了「1 月 1 日下午 3 點」，並且沒有時區信息。東京的負責人猜測這是他們本地時間，但實際上應該是 倫敦時間。
NewYork 分公司（UTC-5）的員工，也以為這是紐約當地時間，結果晚了整整 5 小時才參加。
Australia 分公司（UTC+11）的員工認為這是他們的時間，結果早到了 8 小時。
最終，這場會議變成了一場災難，沒有人能準時參加！ @_________@


### 第二場會議邀約

為了解決這個問題，IT 部門決定使用 DateTime 記錄會議時間。他們為每個分公司分別記錄了當地時間，並通知各地負責人。

倫敦總部記錄的會議時間：2025-01-01 15:00:00
東京分公司：2025-01-01 23:00:00
紐約分公司：2025-01-01 10:00:00
悉尼分公司：2025-01-02 01:00:00
看似解決了問題，但實際上他們發現：

這些時間並沒有包含時區資訊，無法確定它們是 UTC 時間 還是當地時間。
如果一個員工從紐約飛到倫敦，他需要手動計算當地時間。
夏令時間（DST）改變時，部分地區的時間錯誤變得更加頻繁。
IT 部門的主管感到非常頭痛，因為 DateTime 缺乏對時區的準確支持。

### 第三次的會議...不能再錯了!

他們引入了 DateTimeOffset，這個神奇的類型能夠同時記錄「日期時間」和「與 UTC 的偏移量」。系統重新設計後，會議時間的記錄變成了這樣：

倫敦總部：2025-01-01 15:00:00+00:00（UTC 時間偏移量為 +0）
東京分公司：2025-01-01 23:00:00+09:00（UTC+9）
紐約分公司：2025-01-01 10:00:00-05:00（UTC-5）
悉尼分公司：2025-01-02 01:00:00+11:00（UTC+11）
現在，無論員工在哪裡，他們的系統都可以自動轉換會議時間為當地時間。例如：

紐約的一名員工飛到了倫敦，系統會自動將會議時間從 2025-01-01 10:00:00-05:00 轉換為 2025-01-01 15:00:00+00:00，確保他知道倫敦的實際時間。
悉尼的員工再也不需要手動計算時差，因為系統已經知道偏移量。

```CSHARP

// 1. 定義會議的 UTC 時間
DateTimeOffset meetingUtcTime = new DateTimeOffset(2025, 1, 1, 15, 0, 0, TimeSpan.Zero); // UTC+0

// 2. 定義參與者的時區資訊
var participants = new List<Participant>
{
	new Participant("倫敦總部", TimeZoneInfo.FindSystemTimeZoneById("GMT Standard Time")),
	new Participant("Tokyo 分公司", TimeZoneInfo.FindSystemTimeZoneById("Tokyo Standard Time")),
	new Participant("Eastern 分公司", TimeZoneInfo.FindSystemTimeZoneById("Eastern Standard Time")),
	new Participant("AUS 分公司", TimeZoneInfo.FindSystemTimeZoneById("AUS Eastern Standard Time"))
};

// 3. 計算每個參與者的本地時間
Console.WriteLine("國際會議安排（各地本地時間）：");
foreach (var participant in participants)
{
	DateTimeOffset localMeetingTime = TimeZoneInfo.ConvertTime(meetingUtcTime, participant.TimeZone);
	Console.WriteLine($"{participant.Name}: {localMeetingTime.ToString("yyyy-MM-dd HH:mm:ss zzz", CultureInfo.InvariantCulture)}");
}

// 4. 驗證時間比較，確認會議時間一致
Console.WriteLine("\n驗證：所有人使用 UTC 時間參加會議");
foreach (var participant in participants)
{
	Console.WriteLine($"{participant.Name} 將準時參加會議於 UTC 時間: {meetingUtcTime}");
}


//國際會議安排（各地本地時間）：
//倫敦總部: 2025-01-01 15:00:00 +00:00
//Tokyo 分公司: 2025-01-01 23:00:00 +09:00
//Eastern 分公司: 2025-01-01 10:00:00 -05:00
//AUS 分公司: 2025-01-02 02:00:00 +11:00

//驗證：所有人使用 UTC 時間參加會議
//倫敦總部 將準時參加會議於 UTC 時間: 2025-01-01 15:00:00 +00:00
//Tokyo 分公司 將準時參加會議於 UTC 時間: 2025-01-01 15:00:00 +00:00
//Eastern 分公司 將準時參加會議於 UTC 時間: 2025-01-01 15:00:00 +00:00
//AUS 分公司 將準時參加會議於 UTC 時間: 2025-01-01 15:00:00 +00:00

```

<!-- endtab -->


<!-- tab 實作練習-->

使用 DateTimeOffset 的好處在於，我今天拿到一個時間，我有了更多的判斷依據，先來看看怎麼組裝一個 DateTimeOffset

```csharp

	//// 組裝
	TimeSpan utcOffset = TimeSpan.FromHours(8); //// 8 hr
	DateTimeOffset dateTimeOffset = new DateTimeOffset(DateTime.Now, utcOffset);
	dateTimeOffset.Dump(); //// 5/14/2024 4:31:59 PM +08:00
	
	//// 直接取得現在時間
	DateTimeOffset.Now.Dump(); //// 5/14/2024 4:31:59 PM +08:00

```

簡單來說就是時間跟時區偏移量組起來啦!

在做時間比對時，會發現，DateTimeOffset 會把時區考慮進來，而 DateTime 只會比較值
```csharp

	DateTime utcTime = new DateTime(2024, 5, 14, 0, 0, 0, DateTimeKind.Utc);
	DateTime localTime = new DateTime(2024, 5, 14, 8, 0, 0, DateTimeKind.Local);

	Console.WriteLine(utcTime.ToString("yyyy-MM-dd HH:mm:ss")); // 輸出: 2024-05-14 00:00:00
	Console.WriteLine(localTime.ToString("yyyy-MM-dd HH:mm:ss")); // 輸出: 2024-05-14 08:00:00

	Console.WriteLine(utcTime == localTime); // 輸出: False

	DateTimeOffset utcTime2 = new DateTimeOffset(2024, 5, 14, 0, 0, 0, TimeSpan.Zero);
	DateTimeOffset localTime2 = new DateTimeOffset(2024, 5, 14, 8, 0, 0, TimeSpan.FromHours(8));

	Console.WriteLine(utcTime2.ToString("yyyy-MM-dd HH:mm:ss zzz")); // 輸出: 2024-05-14 00:00:00 +00:00
	Console.WriteLine(localTime2.ToString("yyyy-MM-dd HH:mm:ss zzz")); // 輸出: 2024-05-14 08:00:00 +08:00

	Console.WriteLine(utcTime2 == localTime2); // 輸出: True

```

這個結果會非常好用，例如我要比較哪一個時間比較早時，我不用去擔心現在是本地還是 Utc 時間，更不用去統一時間格式


<!-- endtab -->



{% endtabs %}
