---
title: 組合的力量與設計哲學
date: 2024-07-16 14:08:05
categories: 設計図鑑物語
top_img: https://i.imgur.com/LF7L1Ws.png
cover : https://i.imgur.com/LF7L1Ws.png
tags:
    - Design
toc:
toc_number:
comments :
---

![design](https://i.imgur.com/LF7L1Ws.png)

在上一篇文章中，我們看到繼承帶來的限制與設計陷阱。本篇將把鏡頭轉向另一個更自由、更靈活的設計哲學 —— 組合（Composition）。

組合就像是把系統拆解成可重組的積木，每個模組只負責一個任務，清晰、精準、可插拔，讓程式不僅易於擴充，也更能回應未來的變化。

## 🌸 組合怎麼解決問題?

### 用組合代替繼承，解決行為歧義（Ambiguity of Behavior）

讓戰鬥法師「擁有」戰士和法師的能力，而不是「繼承」它們。

```CSHARP

public class Battlemage
{
    private Warrior _warrior = new Warrior();
    private Mage _mage = new Mage();

    public void MeleeAttack() => _warrior.Attack();
    public void CastSpell() => _mage.CastSpell();
}

```

### 用 Interface 來分離行為，避免多繼承，解決屬性重複（Duplicate State）

讓 Warrior 和 Mage 各自實現自己的行為，但把狀態留給 Character，並且避免通過多繼承直接繼承狀態。

```CSHARP

public class Character
{
    public int Health { get; set; }
}

public interface IWarrior
{
    void Attack();
}

public interface IMage
{
    void CastSpell();
}

public class Battlemage : Character, IWarrior, IMage
{
    public void Attack() { /* 實現近戰攻擊 */ }
    public void CastSpell() { /* 實現魔法攻擊 */ }
}

```

這樣屬性只存在於 Character 中，行為分開，避免了屬性重複的問題。


### 用組合與單一職責原則（Single Responsibility Principle）解耦處理功能耦合過緊的問題

```CSHARP

public interface IStorable
{
    void Save();
    void Load();
}

public interface ICompressible
{
    void Compress();
    void Decompress();
}

public class File : IStorable
{
    public void Save() { /* 保存邏輯 */ }
    public void Load() { /* 載入邏輯 */ }
}

public class CompressedFile : IStorable, ICompressible
{
    private File _file = new File();

    public void Save() => _file.Save();
    public void Load() => _file.Load();
    public void Compress() { /* 壓縮邏輯 */ }
    public void Decompress() { /* 解壓縮邏輯 */ }
}

```

這樣每個 Class 專注於自己的功能，CompressedFile 可以靈活組合功能而不需要多繼承。 

### 當有新需求時，不需動現有 Class，只要「新增組件」即可

我們可以建一個新的 Class 來實現這個功能，然後將它提供給需要使用的物件。這種方法允許我們動態地增加物件的能力。

```csharp

public class SmartHome
{
    private List<ISmartDevice> devices = new List<ISmartDevice>();

    public void AddDevice(ISmartDevice device)
    {
        devices.Add(device);
    }

    public void ControlAllDevices(bool turnOn)
    {
        foreach (var device in devices)
        {
            if (turnOn) device.TurnOn();
            else device.TurnOff();
        }
    }
}

public interface ISmartDevice
{
    void TurnOn();
    void TurnOff();
}

```


### 組合的延伸：清晰邊界、不依賴 protected 與 virtual

```csharp

public class GameObject
{
    public List<IComponent> Components { get; } = new List<IComponent>();

    public void AddComponent(IComponent component)
    {
        Components.Add(component);
    }

    public void Update()
    {
        foreach (var component in Components)
        {
            component.Update();
        }
    }
}

public interface IComponent
{
    void Update();
}

public class PhysicsComponent : IComponent
{
    public void Update() { /* 更新物理狀態 */ }
}

public class RenderComponent : IComponent
{
    public void Update() { /* 更新渲染 */ }
}

```

### 封裝變化點：策略模式（Strategy Pattern）

讓「付款方式」可替換，而不用動到主要流程。

```csharp

public class PaymentProcessor
{
    private IPaymentStrategy paymentStrategy;

    public void SetPaymentStrategy(IPaymentStrategy strategy)
    {
        paymentStrategy = strategy;
    }

    public void ProcessPayment(decimal amount)
    {
        paymentStrategy.Pay(amount);
    }
}

public interface IPaymentStrategy
{
    void Pay(decimal amount);
}

public class CreditCardPayment : IPaymentStrategy
{
    public void Pay(decimal amount) { /* 信用卡支付邏輯 */ }
}

public class PayPalPayment : IPaymentStrategy
{
    public void Pay(decimal amount) { /* PayPal支付邏輯 */ }
}


```

## 👾 組合的挑戰

每個實作都要重複寫同樣邏輯。這就是組合的痛點之一：重複。

### 重複實作行為

```csharp

public interface IAudioPlayer
{
    void Play();
    void Pause();
    void Stop();
    void SkipTrack();
}

public class MP3Player : IAudioPlayer
{
    public void Play() { Console.WriteLine("MP3播放器開始播放"); }
    public void Pause() { Console.WriteLine("MP3播放器暫停"); }
    public void Stop() { Console.WriteLine("MP3播放器停止"); }
    public void SkipTrack() { Console.WriteLine("MP3播放器跳過曲目"); }
}

public class CDPlayer : IAudioPlayer
{
    public void Play() { Console.WriteLine("CD播放器開始播放"); }
    public void Pause() { Console.WriteLine("CD播放器暫停"); }
    public void Stop() { Console.WriteLine("CD播放器停止"); }
    public void SkipTrack() { Console.WriteLine("CD播放器跳過曲目"); }
}

public class StreamingPlayer : IAudioPlayer
{
    public void Play() { Console.WriteLine("串流播放器開始播放"); }
    public void Pause() { Console.WriteLine("串流播放器暫停"); }
    public void Stop() { Console.WriteLine("串流播放器停止"); }
    public void SkipTrack() { Console.WriteLine("串流播放器跳過曲目"); }
}

```

每個播放器類都需要實現相同的方法（Play, Pause, Stop, SkipTrack），導致了大量的重複


# C# 的解法：Default Interface Methods（C# 8 起）

C# 雖然不支援多重繼承，但允許一個類別實作多個介面（Interface），而從 C# 8 開始，介面也可以擁有「預設實作方法（Default Interface Methods）」。這是一個極具彈性的設計，它讓我們可以在不破壞既有實作的前提下，為介面新增行為。

```csharp

interface IExample
{
    void Ex1();                                      // ok
    void Ex2() => Console.WriteLine("IExample.Ex2"); // ok
}

```

這樣做有幾個好處：

✅ 不會破壞現有實作的相容性

✅ 實作端可以選擇是否覆寫預設邏輯

✅ 可讓介面具備部分「共享邏輯」，減少重複


### 多重能力的組合：以音訊播放為例

我們可以透過多個專注功能的介面來組合出不同的播放器行為：

```csharp
public interface IAudioFile
{
	string FileName { get; set; }
	void Play() => Console.WriteLine($"Playing {FileName}");
}

public interface IStreamable
{
	void Stream() => Console.WriteLine("Streaming audio");
}

public interface ICompressible
{
	void Compress() => Console.WriteLine("Compressing audio");
}

// MP3 文件 Interface
public interface IMP3File : IAudioFile, IStreamable, ICompressible
{
	void ApplyID3Tags() => Console.WriteLine("Applying ID3 tags");
}

// FLAC 文件 Interface
public interface IFLACFile : IAudioFile, ICompressible
{
	void ApplyVorbisComment() => Console.WriteLine("Applying Vorbis comment");
}

// 實現
public class MP3Player : IMP3File
{
	public string FileName { get; set; }

	// 可以選擇性地覆蓋 Interface 預設方法
	public void Play() => Console.WriteLine($"MP3 player playing: {FileName}");
}

public class FLACPlayer : IFLACFile
{
	public string FileName { get; set; }

	// 使用 IAudioFile 的預設 Play 方法
}

```

使用
```CSHARP

IMP3File mp3 = new MP3Player { FileName = "song.mp3" };
mp3.Play();        // 自定義實作
mp3.Stream();      // 來自 IStreamable 的預設實作
mp3.Compress();    // 來自 ICompressible 的預設實作
mp3.ApplyID3Tags();// IMP3File 的預設實作

IFLACFile flac = new FLACPlayer { FileName = "audio.flac" };
flac.Play();       // 來自 IAudioFile 的預設實作
flac.Compress();   // 來自 ICompressible 的預設實作
flac.ApplyVorbisComment(); // IFLACFile 的預設實作	

```

這樣的設計實現了：

- 可選擇性覆寫介面邏輯

- 多重功能自由組合，彈性高

- 不會造成繼承階層混亂，也避免菱形問題

相比繼承，介面 + 預設實作的組合方式，不僅提升了模組性與可維護性，也讓行為邏輯的來源更加清晰明確。


## ☘️ 結語

當我們思考類別設計時，與其專注在「它是什麼（is-a）」的血緣邏輯，不如轉向「它能做什麼（can-do）」的能力邏輯。
這種思考會讓我們更常考慮這些關鍵設計原則與工具：

✅ Interface：定義行為合約，清楚邊界

✅ Composition：彈性組合，按需注入

✅ SOLID：促進職責分離與擴展彈性

✅ Contract：清晰定義期望與使用方式

✅ Strategy：封裝變化、延遲決策

用組合思維設計的系統，不僅更能應對需求變動，也更容易測試、維護與擴充。

## 📖 參考

[Joost's Dev Blog](https://joostdevblog.blogspot.com/2014/07/why-composition-is-often-better-than.htmlhttps://tw.twincl.com/programming/*662v)

[default interface methods](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-8.0/default-interface-methods)
