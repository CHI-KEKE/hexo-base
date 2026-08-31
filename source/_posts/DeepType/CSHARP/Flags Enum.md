---
title: Flag Enum - 人生應該有更多種可能性 
date: 2024-04-29 22:45:10
categories: 程思舞想
top_img: https://i.imgur.com/Zly1UDM.png
cover : https://i.imgur.com/Zly1UDM.png
tags:
    - Enum
toc:
toc_number:
comments :
---

{% tabs Flag Enum%}


![m](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Enum/FlagEnum/live_not_single_answer.png)


<!-- tab Lives-->


上回聊了 Enum 對這世界建立秩序的重要性，但還有一個問題是，一個東西可能會同時擁有「多個狀態」，我們希望可以用「一個變數」表示出這些狀態的「組合」。正如同人不只是單一角色，而是多重身分的集合，在人生裡，我們每個人都不只是一種角色：

可能是學生、同時是 朋友的情感諮商專家、同時是家庭的支撐者、同時是某個創作者、夢想家，普通的 Enum 是「單選題」


![l](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Enum/FlagEnum/beautiful.png)

<!-- endtab -->


<!-- tab Single Answer-->


例如有個權限控管的 Enum

```CSHARP

enum FilePermission
{
    Read,
    Write,
    Execute
}
```

導致只能寫

```CSHARP
var permission = FilePermission.Read;
```


![single](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Enum/FlagEnum/enum_is_single.png)


通常我們會希望同時可以 Read 和 Write，怎麼辦？這時候可能會想用 List、Array、HashSet 等來處理，但這樣會比較麻煩，判斷時也比較繞口，十分痛苦

```CSHARP
var permissionList = new List<FilePermission> { FilePermission.Read, FilePermission.Write };
if (permissionList.Contains(FilePermission.Read)) { ... }
```

![per](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Enum/FlagEnum/list_is_bad.png)


<!-- endtab -->

<!-- tab Flag Enum-->



這時候 Flag Enum 超人就出來救場了!
![Image](https://i.imgur.com/eMExQLb.png)

程式碼變成可以這樣尻
```CSHARP

[Flags]
public enum FilePermission
{
    None    = 0,
    Read    = 1 << 0,   // 0001 = 1
    Write   = 1 << 1,   // 0010 = 2
    Execute = 1 << 2,   // 0100 = 4
    Delete  = 1 << 3    // 1000 = 8
}

```

因此，今天若要表示一個檔案可以：Read、Write、Execute 就可以這樣寫：

```CSHARP
var permission = FilePermission.Read | FilePermission.Write | FilePermission.Execute;
```

判斷的時只要用位元運算

```CSHARP
if (permission & FilePermission.Write) != 0)
{
    Console.WriteLine("允許寫入");
}
```

因此我們說 

✅ Flag Enum 可以解決「狀態可組合」的情境


![help](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Enum/FlagEnum/flag_is_help.png)


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Enum/FlagEnum/use_or_to_combine.png)


![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Enum/FlagEnum/use_and_to_check.png)



<!-- endtab -->


<!-- tab Biswise-->


而 Flag Enum 是為了讓多個二進位狀態可以被「一個整數值」同時表達出來，它把每個選項設計成「2 的次方數值」，如此一來就可以用「位元運算」組合它們！這裡每個值都是一個「位元」，可以獨立開關

因次上面的例子，實際值會變成：

0001 (Read)
0010 (Write)
0100 (Execute)

= 0111 (7)


![bit_is_switch](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Enum/FlagEnum/bit_is_switch.png)


<!-- endtab -->

<!-- tab 那些喜歡玩遊戲的人-->


喜歡玩遊戲的人從小一定不只玩一種

```CSHARP

var ruby = Games.MapleStory | Games.LeetCode | Games.Playing_Kids;
var bruce = Games.MineCraft | Games.Chain_Together | Games.Tetris_Battle;
Console.WriteLine(ruby); //// MapleStory, LeetCode, Playing_Kids

public enum Game
{
    None = 0,
    MapleStory = 1 << 0,
    Bubble_Bobble = 1 << 1,
    MineCraft = 1 << 2,
    Tetris_Battle = 1 << 3,
    Chain_Together = 1 << 4,
    LeetCode = 1 << 5,
    Playing_Kids = 1 << 6
}


```


![g](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Enum/FlagEnum/game_play.png)


<!-- endtab -->

<!-- tab 進階的需求 : 一串文字轉化成 FlagEnum-->

![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Enum/FlagEnum/string_to_enum.png)

突然想到這個問題要怎麼做，結果發現...實作起來有點難欸

```CSHARP

//// 一樣定義一組權限控管的 Enum
[Flags]
public enum Permission
{
	None = 0,
	Read = 1 << 1,
	Execute = 1 << 2,
	Delete = 1 << 3
}

//// 定義一個 string 的擴充方法來處理 字串資料轉 FlagEnum
public static class StringExtension
{
    //// 泛型處理挺麻煩
	public static T ToFlagEnum<T>(this string jsonArrayOrString) where T : struct, Enum
	{
		string[] items;
		try{
			items = JsonSerializer.Deserialize<string[]>(jsonArrayOrString);
		}
		catch{
			items = jsonArrayOrString.Split(',',StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries);
		}
		
        //// 因為是泛型的關係，編譯期間是無法得知 Enum 底層的數值是什麼型別
        //// 因此需要先轉換成底層數數值型別才能做位元運算 (byte , int, short...)
		var underLyingPrimitiveType = Enum.GetUnderlyingType(typeof(T));
		var result = Convert.ChangeType(0,underLyingPrimitiveType);

        //// 接著對傳入值一樣解析成數值型別
        //// 對齊後 處理位元運算
		foreach(var item in items)
		{
			if (Enum.TryParse<T>(item, ignoreCase : false, out var value))
			{
				var parsedEnumValue = Convert.ChangeType(value,underLyingPrimitiveType);
				result = underLyingPrimitiveType switch
				{
					Type t when t == typeof(byte) => (byte)result | (byte)parsedEnumValue,
					Type t when t == typeof(sbyte) => (sbyte)result | (sbyte)parsedEnumValue,
					Type t when t == typeof(short) => (short)result | (short)parsedEnumValue,
					Type t when t == typeof(ushort) => (ushort)result | (ushort)parsedEnumValue,
					Type t when t == typeof(int) => (int)result | (int)parsedEnumValue,
					Type t when t == typeof(uint) => (uint)result | (uint)parsedEnumValue,
					Type t when t == typeof(long) => (long)result | (long)parsedEnumValue,
					Type t when t == typeof(ulong) => (ulong)result | (ulong)parsedEnumValue,
					_ => throw new NotSupportedException($"不支援的 enum 底層型別：{parsedEnumValue}")
				};
			}
		}
		
        //// 最後記得,在轉回 Enum 類型!
		return (T)Enum.ToObject(typeof(T),result);
	}
}


void Main()
{
	var jsonArray = "[\"Read\",\"Execute\"]";
	jsonArray.ToFlagEnum<Permission>().Dump();
}

```

![j](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Enum/FlagEnum/json_desr.png)

![a](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Enum/FlagEnum/underlying_type.png)


![c](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Enum/FlagEnum/bit_final.png)



<!-- endtab -->

<!-- tab 結語-->



![summ](https://github.com/CHI-KEKE/pics/blob/main/Code_Design/Enum/FlagEnum/summ.png?raw=true)


想像你玩 RPG 遊戲（角色扮演），

傳統 Enum　告訴你，你只能選「劍士」或「法師」或「弓箭手」，只能一個。
而 Flag Enum 讓我們知道了更多可能性，你可以同時是「劍士 + 魔法師 + 商人」，混搭出自己的流派。


<font color=#D3D3D3 style="font-size: 18px;">&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;真正強的人不是選對一個社會定義好的角色從一而終、至死方休
 </font>

<font color=#D3D3D3 style="font-size: 18px;">&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;而是「組合出屬於自己的角色」。
 </font>



Flag Enum 教我們：人不該被一個標籤定義，我們是由多種價值與狀態組合而成的豐富存在。

你學的每一項技能、每一次反省、每一段成長，都會成為你生命中某個位元的開關，這些開關，一起組合，定義了獨一無二的你，人不是「身兼兩職」這麼簡單，而是用一個靈魂去承載兩種甚至更多角色的重量與愛。

你是黑夜中為自己撐傘的人，
也是清晨裡為他人煮咖啡的手。

你是會寫程式的工程師，
也是半夜聽朋友哭訴的傾聽者。

你是教室裡認真解題的學生，
也是家裡照顧弟妹的大人。

你是職場上冷靜果斷的主管，
也是回家後蹲下身來哄孩子的媽媽。


如果你今天能被程式的 Flag Enum 觸動心弦，那是因為你早就明白，人生不是單選題
願你繼續組合、疊加、拓展，願你記得：不是所有人都能被簡化為一個 enum，但你能以 Flag 的名義，自由存在


![aaa](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Enum/FlagEnum/live_should_have_many_posiibilit.png)

<!-- endtab -->


{% endtabs %}