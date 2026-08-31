---
title: LINQ - SelectMany
date: 2025-09-21 11:49:11
categories: LINQ
top_img: https://i.imgur.com/cBfeEDY.png
cover : https://i.imgur.com/cBfeEDY.png
tags:
    - LINQ
toc:
toc_number:
comments :
---

艾倫家的地下室有三個抽屜，每個抽屜都要世界的秘密，我們想將它們都攤開

- 最外層是地下室
- 次一層表示抽屜們
- 下一層表示機密文件

Select
```plaintext
[[照片,日記],[戰爭情報,國家情報],[美女照]] 
```

使用 Select，我們等於搬著三個抽屜離開

SelectMany
```plaintext
[照片,日記,戰爭情報,國家情報,美女照] 
```
SelectMany，讓我們每一樣任務物品都可以輕易地取出排列整齊，還可以偷偷把美女照藏起來

<br>
<br>

## 🐛 如果世界上沒有 SelectMany

今天我們有這樣一個集合，有 7 個老師，每位老師有 3 名學生，共 21 個學生
```CSHARP

List<Teacher> teachers = new List<Teacher>
        {
            new Teacher("Allen",new List<Student>{ new Student(100),new Student(90),new Student(80) }),
            new Teacher("Bill",new List<Student>{ new Student(100),new Student(90),new Student(60) }),
            new Teacher("Charlie",new List<Student>{ new Student(100),new Student(90),new Student(40) }),
            new Teacher("Duck",new List<Student>{ new Student(100),new Student(90),new Student(60) }),
            new Teacher("Elma",new List<Student>{ new Student(100),new Student(90),new Student(50) }),
            new Teacher("Fish",new List<Student>{ new Student(100),new Student(90),new Student(60) }),
            new Teacher("Google",new List<Student>{ new Student(20),new Student(10),new Student(33) })
        };

```

需求是這樣，請把不及格的學生抓出來校長要*照顧*一下

```CSHARP

List<Student> students = new List<Student>();
foreach(var teacher in teachers)
{
    //// 因為 stduent 關聯在 teacher 之下
    foreach(var student in students)
    {
        if(student.Score < 60>)
        {
            students.Add(student);
        }
    }
}

students.Dump();
```

結果可以拿到
![Image](https://i.imgur.com/3OOLYb9.png)

但我們看到巢狀就頭暈貧血發作要怎麼辦

別急，我們還有 Query Expressions !
```CSHARP
var students = from t in teachers
               from s in students
               where s.Score < 60
               select s;
```

這種作法增加了 "SQL" 長期使用者的易讀性，描述了程式應該完成的任務 (就是所謂的 Declarative Programming)，而不指定具體的步驟，也就是，閱讀程式碼的人一眼知道你想幹嘛而不是研究具體的步驟怎麼處理


由於我們因為太喜歡鏈式風格，我們不死心的嘗試 SELECT 兩次

但問題在這，他的維度還是停留在 teachers 這一層， SELECT 100 次也沒有用
```CSHARP

var failingStudents = teachers
    .Select(t => t.Students.Where(s => s.Score < 60)).Select(teacher => teacher.Select(s => s));


//// IEnumerable<IEnumerable<Student>>
```

等等，Aggregate 呢? 他讓我們自己定義一個邏輯，把一堆資料累加、合併、轉換成想要的結果。
```CSHARP

var failingStudents = teachers
    .Select(t => t.Students.Where(s => s.Score < 60)) // Still By Teacher
    .Aggregate(new List<Student>(), (acc, students) =>  // Manually flatten the list
    {
        acc.AddRange(students);
        return acc;
    }).Dump();

```

喔...好累 ﾟÅﾟ)


<br>
<br>

## 🐛 SelectMany

操作資料時，我們總有一堆集合，每個集合裡還有一堆7788 的東西，我想把所有東西攤開來處理
本質上：SelectMany = 「先選出多個東西」+「自動攤平成一個集合」
```CSHARP
teachers.SelectMany(t => t.Students).Where(s => s.Score < 60).ToList();
```

<br>

###　方法簽章
```CSHARP

public static IEnumerable<TResult> SelectMany<TSource, TCollection, TResult>(
    this IEnumerable<TSource> source, 
    Func<TSource, IEnumerable<TCollection>> collectionSelector, 
    Func<TSource, TCollection, TResult> resultSelector);

```

參數中，若傳入 resultSelector，讓我們同時處理 TSource(原本集合的每一個元素) 及 TCollection(子集合中的每一個元素)
```CSHARP
teachers.SelectMany(t => t.Students
                    , (t, s) => new {t.Name, s.Score})
        .Where(pair => pair.Score < 60).Dump();
```

```plaintext
[
  { Name = "張老師", Score = 55 },
  { Name = "陳老師", Score = 40 },
  { Name = "李老師", Score = 59 },
  ...
]
```

<br>

### SourceCode 

其實 SelectMany 的本質還是：幫你把這「雙層迴圈」內建起來，我們看 sourcecode 便知

SelectMany 有 4 種 overload，為了因應軟體設計的本質之一 : 彈性

從一個集合的每一項取出「一組子集合」，然後攤平成一條清單（扁平化 flat），但我們在寫程式的時候，有時：

- 只需要子集合
- 有時候需要 原本那一項的 index
- 有時候還需要 原本那一項 + 子集合元素同時用來產生結果

| Overload 編號 | 方法簽章                                                                                                                              | 支援 index？ | 支援原始項目 + 子項目同時投影？ | 常見用途                   |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------- | --------- | ----------------- | ---------------------- |
| **#1**      | `SelectMany(Func<TSource, int, IEnumerable<TResult>> selector)`                                                                   | ✅ 是       | ❌ 否               | 需要知道元素位置（index）時       |
| **#4**      | `SelectMany(Func<TSource, IEnumerable<TResult>> selector)`                                                                        | ❌ 否       | ❌ 否               | 最常見情況，純攤平子集合           |
| **#2**      | `SelectMany(Func<TSource, IEnumerable<TCollection>> collectionSelector, Func<TSource, TCollection, TResult> resultSelector)`      | ❌ 否       | ✅ 是               | 想要保留父子關係進行投影           |
| **#3**      | `SelectMany(Func<TSource, int, IEnumerable<TCollection>> collectionSelector, Func<TSource, TCollection, TResult> resultSelector)` | ✅ 是       | ✅ 是               | 想要保留 index 與父子關係進行複雜處理 |

<br>

### 最基本版（最常見）

```CSHARP

public static IEnumerable<TResult> SelectMany<TSource, TResult>(
    this IEnumerable<TSource> source, Func<TSource, int, IEnumerable<TResult>> selector)
{
    if (source == null)
    {
        throw Error.ArgumentNull(nameof(source));
    }

    if (selector == null)
    {
        throw Error.ArgumentNull(nameof(selector));
    }

    return SelectManyIterator(source, selector);
}

```

1. 判斷 source 及 selector 是否為空，為空的話丟 ArgumentNull 例外
2. 傳回 SelectManyIterator
3. 真正的處理邏輯，被包在一個「私有的 iterator 方法」裡面。

```CSHARP

private static IEnumerable<TResult> SelectManyIterator<TSource, TResult>(
    IEnumerable<TSource> source, Func<TSource, int, IEnumerable<TResult>> selector)
{
    int index = -1;
    foreach (TSource element in source)
    {
        checked
        {
            index++;
        }

        foreach (TResult subElement in selector(element, index))
        {
            yield return subElement;
        }
    }
}

```

這是一個 「iterator」方法，會用 yield return 把資料一筆一筆生出來。

- 初始化 index（因為第一次加一後才是 0）
- 外層迴圈：把 source 中的每個項目（例如老師）一筆一筆拿出來。
- 安全地幫 index 加 1，確保不會超出整數範圍（溢位會拋出例外）
- 這就是關鍵的「第二層迴圈」：
  - selector 是你自己提供的轉換邏輯
  - 它回傳的是一組子集合（例如學生清單）
  - 所以你還要再迴圈一次，把子集合攤平
  - 每一筆學生（子集合元素）都用 yield return 一筆一筆回傳。

<br>
<br>

## 🐛 使用 SelectMany 來處理關聯式資料比手動使用 JOIN 語法更好

在使用 Entity Framework（EF）操作資料庫時，當我們需要查詢「一對多」或「多對一」的關聯資料時，有兩種常見寫法：

🅰 傳統 SQL 思維 — 使用 join
```csharp
var queryB = from s in context.Schools
             join t in context.Teachers on s.Id equals t.IdSchool
             select t;
```

🅱 更現代 LINQ 思維 — 使用 SelectMany 搭配導覽屬性
```csharp
var queryA = context.Schools.SelectMany(s => s.Teachers);
```

哪一種方式比較好？在大資料量的情況下，效能會差很多嗎？使用 SelectMany 是不是只是寫法漂亮，但實務上沒效率？

先說結論，在 Entity Framework 中，優先使用 SelectMany 搭配「導覽屬性（Navigation Properties）」的方式通常是更好、更乾淨、更推薦的寫法。

### 語意更貼近物件導向
```csharp
school.Teachers
```
這樣的寫法表達的是：「我從每個學校中，拿出它的老師們」
這種語意清楚又直觀，不需要你去關心「外鍵欄位叫什麼」，你只需要透過模型之間的導覽屬性操作即可。

### EF 會自動產生 JOIN，效能不會比較差

SelectMany
```csharp
var teachers = context.Schools
                      .Where(s => s.Name == "XX中學")
                      .SelectMany(s => s.Teachers)
                      .Where(t => t.Department == "資訊科")
                      .ToList();
```
EF 背後會根據導覽屬性與關聯的中繼資料，自動產生SQL

JOIN
```CSHARP
var teachers = from s in context.Schools
               join t in context.Teachers on s.Id equals t.IdSchool
               where s.Name == "XX中學" && t.Department == "資訊科"
               select t;
```
功能一樣，但 SelectMany 讀起來更像是在查詢「物件的資料」，而不是「操作資料表」。


雖然 SelectMany 通常是更好的選擇，但有些情況下 join 還是必要的，例如

- 你要查詢兩個彼此沒有導覽屬性的資料表
- 你要查詢左外連接（Left Join）
- 你只想查出欄位（非物件）或匿名類型組合資料

<br>
<br>

## 🐛 參考文章

{% btn 'https://peterhpchen.github.io/DigDeeperLINQ/12_HowToUseSelectMany.html#%E5%8A%9F%E8%83%BD%E8%AA%AA%E6%98%8E',SelectMany的應用,far fa-hand-point-right %}

<br>

{% btn 'https://stackoverflow.com/questions/38311230/chain-selectmany-instead-of-using-a-join-statement?newreg=43ff320edfd84529aa6ae9a8b5d8e28f',Chain SelectMany instead of using a JOIN statement,far fa-hand-point-right %}