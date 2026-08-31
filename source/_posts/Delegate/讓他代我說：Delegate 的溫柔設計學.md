---
title: 讓他代我說：Delegate 的溫柔設計學
date: 2024-11-07 17:57:00
categories: 風音子
top_img: https://i.imgur.com/zTSkblP.png
cover : https://i.imgur.com/zTSkblP.png
tags:
    - Delegate
toc:
toc_number:
comments :
---

在人生裡，我們總在不知不覺中進行著「委派」。
不是公司開會那種，而是日常裡那些再自然不過的小任務外包。

👦：「阿嬤，我要出門了，幫我跟媽媽說我會晚點回家～」
你沒直接說，而是把「傳話」這件事交給了阿嬤。這一刻，你不是孫子，你是個指派 callback 的主控者。

而在程式世界裡，這樣的行為就叫作：delegate ——
拜託別人幫你做某事，但怎麼做、什麼時候做，全憑對方自由發揮。

![Image](https://i.imgur.com/J9DiZjN.png)

```CSHARP

//// 定義要執行的任務的事件樣貌
delegate void PassMessage(string message);

public class You
{
    public void AskSomeoneToPassMessage(PassMessage messenger)
    {
        messenger("我會晚一點回家");
    }
}

//// 執行者自己決定怎麼執行
void GrandmaPassMessage(string msg)
{
    Console.WriteLine($"拎孫供 : {msg} 啦!");
}

```

```CSHARP

//// 決定選擇 *阿嬤傳話* 這個實作來完成任務
var me = new You();
me.AskSomeoneToPassMessage(GrandmaPassMessage);

```

🧠 這就是委派的哲學：

你只需要確保「有傳達」，至於阿嬤要用 Line、廣播、還是菜市場大喊三聲 —— 隨便她。

delegate 就像人生中那些「我先交代下去，實作交給命運」的時刻，它優雅地將控制權交出，卻依然維持邏輯的秩序。

## 🎤 為什麼需要 Delegate？

在設計程式時，我們常常希望 保留彈性，不要把「做法」寫死在初始的設計藍圖裡。因為：

- 我們可能事前無法預知會有哪些行為需求。

- 實作細節應該交由使用者決定，讓 class 本身保持「通用性」與「可重用性」。

這時候，delegate（委派）就像一個「可以外包的函式入口」，讓使用者自行註冊要做什麼。你只需要定義好「格式」，實際要怎麼做，呼叫端來決定。

### 生活中的例子：人際應對就是多型

人與人之間的相處中，我們總是會根據對象的不同，而有不同應對進退的模式，有人覺得這有點虛為

就我而言，在現實世界的種場合，人會有探測距離感的能力，或探測對方腦波是不是在同一個頻率的能力，如果我們在朋友面前講幹話說髒話可以表現出我們之間是可以這樣做的親近感，但與上司或父母做蔣幹畫會導致不可偉回的後果

上帝為了讓人類社會變得有趣，可能會這樣設計

```CSHARP

public class Person
{
	private string _name;
	private string _currentTalkMode;
	
	public Person(string name)
	{
		_name = name;
		_currentTalkMode = "Normal";
	}
	
	public void SetTalkMode(string mode)
	{
		_currentTalkMode = mode;
	}
	
	public void Speak(string message)
	{
		string processedMessage = "";
		switch(_currentTalkMode)
		{
			case "BOSS":
				processedMessage = $"尊敬的長官，{message}。此致！";
				break;

			case "FRIEND":
				processedMessage = $"白癡喔 {message} 幹XD 笑死";
				break;

			case "PARENTS":
				processedMessage = $"敬愛的父親母親，{message}";
				break;

			case "LOVER":
				processedMessage = $"{message} ❤ 討厭~ 你越壞我越愛";
				break;

			case "CUSTOMER":
				processedMessage = $"親愛的顧客，{message}，感謝您的支持！";
				break;

			case "INTERVIEW":
				processedMessage = $"關於{message}，我認為...";
				break;

			// 如果要加入新的說話方式，需要修改類別本身
			// case "NEW_MODE":
			//     processedMessage = "新的說話方式";
			//     break;

			default:
				processedMessage = message;
				break;	
		}

		Console.WriteLine($"{_name} 說: {processedMessage}");
	}

	public void Presentation(string message)
	{
		var content = "";
		switch(_currentTalkMode)
		{
			case "PTT":
				content = $"各位 30 cm, E Cup 鄉民大家好, {message}";
				break;
			case "Presentation":
				content = $"很榮幸今天有機會在這裡與大家分享...{message}";
				break;		
			// 另一堆判斷...	
		}
		
	}

	public void SpeakWithTone(string message, string tone)
	{
		// 更多判斷...
	}
}

```

🔍 問題點：

1. 新增說話模式時需修改類別 ➜ 違反開放封閉原則（OCP）

2. 難以測試特定邏輯 ➜ 行為跟類別耦合太深

3. 維護困難，行為分散於多個 switch 區塊，但執行的方式大同小異

...

### 委派版本：將「行為」抽換成參數

```CSHARP

// 定義說話的委派
public delegate string TalkDelegate(string message);

// 定義簡報的委派
public delegate string PresentationDelegate(string message);

   public class Person
    {
        private string _name;
        private TalkDelegate _talkBehavior;
        private PresentationDelegate _presentationBehavior;
        public Person(string name, TalkDelegate talkBehavior, PresentationDelegate presentationBehavior)
        {
            _name = name;
            _talkBehavior = talkBehavior;
            _presentationBehavior = presentationBehavior;
        }

        public void Speak(string message)
        {
            string processedMessage = _talkBehavior(message);
            Console.WriteLine($"{_name} 說 : {processedMessage}");
        }

        public void Presentation(string message)
        {
            string processedMessage = _presentationBehavior(message);
            Console.WriteLine($"簡報內容 : {processedMessage}");
        }

        public void ChangeTalkBehavior(TalkDelegate talkDelegate)
        {
            _talkBehavior = talkDelegate;
        }
    }

```

使用
```CSHARP

//// 使用 Person 時，在決定要怎麼實作對話方式
TalkDelegate talkToFriend = message => $"低能喔 {message} 幹XD 笑死";
TalkDelegate talkToBoss = message => $"尊敬的長官，{message}";
TalkDelegate talkToParents = message => $"敬愛的父親母親，{message}";
TalkDelegate talkToLover = message => $" {message} ❤ 討厭~ 你越壞我越愛";
TalkDelegate talkInterviewer = message => $" 我對這間公司極度充滿熱情, {message} ";
PresentationDelegate PttPresentation = message => $"各位 30cm ECup 大家好, {message} ";
PresentationDelegate CorpPresentation = message => $"各位長官好, {message} ";

var allen = new Person("Allen", talkToFriend, CorpPresentation);
var mochi = new Person("Mochi", talkToLover, PttPresentation);
allen.Speak("這是甚麼");
mochi.Speak("汪汪汪");
mochi.Presentation("我是一隻狗");

```

![Image](https://i.imgur.com/T4PqNcj.png)


因為我看到了傳入 & 傳出 的參數類型是一樣的，我們把它模板化了，使用者只要傳入符合模板的實作方式就可以成立!


## 🌱 Delegate 的寫法演進史

在 C# 中，delegate（委派）是一種型別安全的函式指標，用來封裝方法的參考。從 C# 2.0 到 C# 3.0，delegate 的語法變得越來越簡潔，也更容易上手。以下用三種常見寫法介紹 delegate 的演進過程：

### 📌 C# 2.0 基本寫法

從 C# 2.0 開始，當編譯器發現你要指派的方法與 delegate 型別相符時，就會自動幫你補上 new 的語法，讓你可以寫得更簡潔。

```CSHARP

//// 定義 delegate 型別
delegate void MessageHandler(string message);

//// 建立一個方法符合 delegate 簽章
void ShowInfo(string message)
{
    Console.WriteLine($"訊息: {message}");
}

//// 使用方法名稱直接指派（省略 new）
MessageHandler handler = ShowInfo;
handler("你好！");

```

ShowInfo 是一個符合 MessageHandler 簽章的方法（參數與回傳值一致），因此可以直接指定給 delegate 變數。

雖然你沒有寫 new MessageHandler(ShowInfo)，但編譯器會自動補上。

### 💡 C# 2.0 的 Anonymous Method（匿名方法）

當你懶得為一個只用一次的小功能寫一個獨立方法時，可以用匿名方法來快速定義 inline 的 delegate。

```CSHARP

MessageHanlder messageHandler2 = delegate(string message)
{
    $"通知 : {message}".Dump();
};

```

這種語法叫做 Anonymous Method，意思是「沒有名字的方法」，適合用在只會使用一次的小功能或 callback。

### 🚀 C# 3.0 的 Lambda Expression（Lambda 運算式）

從 C# 3.0 開始，我們可以使用更簡潔的 Lambda 表達式 來取代匿名方法。

```CSHARP

MessageHandler handler = (string message) =>
{
    Console.WriteLine($"通知: {message}");
};
handler("Lambda 寫法來囉！");

```

📦 進一步簡化： 如果參數只有一個，而且型別可以從上下文推論出來，就可以省略型別與括號：

```CSHARP

MessageHandler handler = message => Console.WriteLine($"通知: {message}");

```

=> 讀作「goes to」，左邊是輸入參數，右邊是要執行的內容。Lambda 表達式不只能取代 delegate，還是 LINQ 的核心語法之一。

![Image](https://i.imgur.com/bK32ol5.png)


## 為什麼 C# 的委派（Delegate）設計這麼實用？

在 C# 裡，delegate 是語言內建的一等公民（first-class citizen），可以把它想像成一個專門處理方法指派與呼叫的類型安全工具。

相比傳統的函式指標（像 C 語言那種），C# delegate 的設計結合了 物件導向（Object-Oriented） 的思想，讓委派不僅安全、好用，還可以整合進 class 架構中，更容易管理與擴充。

### 比喻：C# 的委派就像一間「正式的購物服務中心」
```CSHARP

// 定義一個購物委派，接受一個 int 作為金額
delegate void ShoppingDelegate(int money);

class ShoppingMall
{
    public void BuyClothes(int money)
    {
        Console.WriteLine($"買了{money}元的衣服");
    }

    public void Start()
    {
        // 建立一個「服務櫃台」（delegate）
        ShoppingDelegate service = new ShoppingDelegate(BuyClothes);
        
        // 系統會自動：
        // 1. 檢查服務是否存在
        // 2. 確保呼叫方式正確
        // 3. 管理服務的生命週期
        service(100);
    }
}

```

這樣做有什麼好處？

✅ A. 類型安全 Type-safe

```CSHARP

delegate void PaymentDelegate(decimal amount);

// ✅ 正確：參數類型匹配
PaymentDelegate payment = ProcessPayment;

// ❌ 錯誤：編譯器會阻止錯誤的類型使用
// PaymentDelegate payment = ProcessString; // 不會通過編譯

```

🔁 B. 支援多重委派（Multicast Delegate）

```CSHARP

delegate void NotifyDelegate(string message);

class NotificationSystem
{
    public void SendEmail(string message) { }
    public void SendSMS(string message) { }

    public void Setup()
    {
        // 一個委派可以同時指向多個方法
        NotifyDelegate notifier = SendEmail;
        notifier += SendSMS;  // 加入另一個處理方法
        
        // 一次呼叫，兩個服務都會執行
        notifier("有新商品上架！");
    }
}

```



## Source Code 分析

![Image](https://i.imgur.com/0EvIG2m.png)

讓我們根據 source code 向下剝一層看看背後是怎麼設計的

假設
```CSHARP

delegate void ShoppingDelegate(int money);
ShoppingDelegate service = new ShoppingDelegate(BuyClothes);

```

實際上，當我們定義這個 delegate 時，相當於編譯器生成一個這樣的 Class

```CSHARP

public sealed class ShoppingDelegate : MulticastDelegate
{
    // 儲存方法的地址
    private IntPtr methodPtr;

    // 儲存物件實例（如果是實例方法）
    private object target;       

    // 建構函數
    public ShoppingDelegate(object target, IntPtr methodPtr)
    {
        this.target = target;
        this.methodPtr = methodPtr;
    }

    // 調用方法
    public int Invoke(int a, int b)
    {
        // 通過儲存的地址調用方法
        return /* 使用 methodPtr 調用方法 */;
    }
}

```

所以當我們創建委託實例時，這相當於

```CSHARP

// 創建委託實例過程會
// a. 獲取 BuyClothes 方法的記憶體地址
// b. 創建 ShoppingDelegate 的實例
// c. 將方法地址儲存在委託實例中
ShoppingDelegate service = new ShoppingDelegate(BuyClothes);



// 使用委託此時會：
// a. 找到儲存的方法地址
// b. 用提供的參數調用該地址的方法
service(300); // 法1
service.Invoke(300); // 法2
```

圖解版本
![Image](https://i.imgur.com/U9sz9lc.png)

從根本上來說，delegate 就是一種封裝方法地址的類別，讓你：

✅ 可以把方法當成資料傳來傳去

✅ 可以在執行時決定要呼叫哪個方法（動態綁定）

✅ 支援單播、多播（Multicast）


相關文章可以參考
[C# 筆記：重訪委派－從 C# 1.0 到 2.0 到 3.0](https://www.huanlintalk.com/2009/01/delegate-revisited-csharp-1-to-2-to-3.html)

![Image](https://i.imgur.com/ePAc8LS.png)

委派物件的呼叫清單不會濾掉重複的函式參考，亦即同一個函式可以重複加入多次

## ☘️ 結語｜在變與不變之間，留一席位置給「委派」

委派（delegate），本質上是一種信任。

它讓我們可以放手，將「怎麼做」的決定權交給他人，或交給未來。
你只需定義好願景、格式、契約，其餘的交給世界去完成。

在 OOP 的世界裡，delegate 是將行為抽象化的容器，
它解耦了邏輯與實作，也打開了擴充與重用的大門。

如同我們在人生中學會放手、學會合作，
程式的設計也不必事事掌控，只要掌握核心原則，讓彈性自然流動。

這，正是軟體工程中的藝術。



<font color=#D3D3D3 style="font-size:18px;">&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;定義行為，而非干涉細節 —— 這是委派的優雅，也是設計的智慧
</font>