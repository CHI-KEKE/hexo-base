

## Encoding.UTF8.GetBytes(yourString)


因為 .NET 的 string 在記憶體中是用 UTF-16 存的（每個字兩個 byte）
當執行
```CSHARP
Encoding.UTF8.GetBytes(yourString)
```

發生的事情是

- .NET 看到一個字串（UTF-16）
- 用 UTF-8 的規則重新編碼
- 把每一個字拆成 UTF-8 所需要的 byte
- 回傳一個 byte[]

也就是 🔄 UTF-16 → UTF-8 → byte[]

以字「你」為例
UTF-16 編碼（.NET string 的內部格式）是 0x4F60
UTF-8 會把這個字轉成三個 byte E4 BD A0


📘 .NET 字串 → 用自己的語言（UTF-16）存
🌍 外面世界（檔案、網路、API）都講 UTF-8
🔧 GetBytes() 就是「把話翻譯成外面世界能懂的 byte」


## Stream

Stream 是一種資料讀寫的管道

Stream 不關心資料從哪來

你可以一段一段讀

也可以一段一段寫

Stream 就像一根水管：
💧「資料變成水，讓你慢慢流出來或流進去。」

MemoryStream 是 .NET 幫你把 byte[] 包裝成一種「可以像檔案一樣讀取」的東西

你可以：

用 StreamReader 讀它

用 API 當成 InputStream 傳入

用 JSON 序列化寫進去

用某些套件當作「虛擬檔案」處理

但所有資料都存於記憶體，不會寫到磁碟

👉 MemoryStream = 一個在記憶體裡，用來存放 byte 資料的容器（Stream 版本）。



	using (var writer = new StreamWriter(path, true))
	{
		await writer.WriteLineAsync(DateTime.Now.ToString("HH:mm:ss"));
	}




async void Main()
{
	var urls = new List<string>()
	{
		"https://www.gutenberg.org/cache/epub/1342/pg1342.txt",
		"https://www.gutenberg.org/cache/epub/1661/pg1661.txt"
	};
	
	await DonwloadFilesAsync(urls);
}

public static async Task DonwloadFilesAsync(IList<string> urls)
{
	var getContentTasks  = new List<Task<byte[]>>();
	foreach(var url in urls)
	{
		// 呼叫 DownloadFileAsync(url) → ⚠️ 這一步會「建立並啟動」一個非同步任務。
		// 把那個回傳的 Task<byte[]> 加進 getContentTasks List。
		// 所以在這一步已經啟動任務
		// 相當於 var task = DownloadFileAsync(url); // 🚀 下載開始！ getContentTasks.Add(task); // 🧺 放進任務清單
		getContentTasks.Add(DownloadFileAsync(url));
	}

	var contents = await Task.WhenAll(getContentTasks);
	
	foreach(var content in contents)
	{
		$"Sunccessfully donwloaded conetnt, length : {content.Length}".Dump();
		var fileString = System.Text.Encoding.UTF8.GetString(content);
		var preview = fileString.Substring(0, Math.Min(50, fileString.Length));
		$"Preview : {preview}".Dump();
	}
}

public static async Task<byte[]> DownloadFileAsync(string url)
{
	using(var client = new HttpClient())
	{
		byte[] content = await client.GetByteArrayAsync(url);
		File.WriteAllBytes(@"C:\Users\Allen Lin\Desktop\MyLab\Lab.同步異步\x.txt",content);
		return content;
	}
}

