---
title: Model
date: 2025-09-22 22:21:34
categories: Web
top_img: https://github.com/CHI-KEKE/pics/blob/main/MVC/pumpkin.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/MVC/pumpkin.png?raw=true
tags:
    - Web
toc:
toc_number:
comments :
---


Model 是系統「知道什麼」的部分，負責描述資料的結構與規則，是整個應用程式的資料核心

<br>

## 🎃 Seed

在開發或測試階段，我們常常會希望資料庫一開始就有資料方便測試，這時候就可以使用「Seed」方法。

```csharp
public class SchoolInitializer
{
    public static void Initialize(SchoolContext context)
    {
        // 確保資料庫存在（如果不存在就建立）
        context.Database.EnsureCreated();

        // 如果已經有學生資料就略過（避免重複加資料）
        if (context.Students.Any())
        {
            return;
        }

        var students = new List<Student>
        {
        new Student{FirstMidName="Carson",LastName="Alexander",EnrollmentDate=DateTime.Parse("2005-09-01")},
        new Student{FirstMidName="Meredith",LastName="Alonso",EnrollmentDate=DateTime.Parse("2002-09-01")},
        new Student{FirstMidName="Arturo",LastName="Anand",EnrollmentDate=DateTime.Parse("2003-09-01")},
        new Student{FirstMidName="Gytis",LastName="Barzdukas",EnrollmentDate=DateTime.Parse("2002-09-01")},
        new Student{FirstMidName="Yan",LastName="Li",EnrollmentDate=DateTime.Parse("2002-09-01")},
        new Student{FirstMidName="Peggy",LastName="Justice",EnrollmentDate=DateTime.Parse("2001-09-01")},
        new Student{FirstMidName="Laura",LastName="Norman",EnrollmentDate=DateTime.Parse("2003-09-01")},
        new Student{FirstMidName="Nino",LastName="Olivetto",EnrollmentDate=DateTime.Parse("2005-09-01")}
        };

        context.Students.AddRange(students);
        context.SaveChanges();
        var courses = new List<Course>
        {
        new Course{CourseID=1050,Title="Chemistry",Credits=3,},
        new Course{CourseID=4022,Title="Microeconomics",Credits=3,},
        new Course{CourseID=4041,Title="Macroeconomics",Credits=3,},
        new Course{CourseID=1045,Title="Calculus",Credits=4,},
        new Course{CourseID=3141,Title="Trigonometry",Credits=4,},
        new Course{CourseID=2021,Title="Composition",Credits=3,},
        new Course{CourseID=2042,Title="Literature",Credits=4,}
        };
        context.Courses.AddRange(courses);
        context.SaveChanges();
        var enrollments = new List<Enrollment>
        {
        new Enrollment{StudentID=1,CourseID=1050,Grade=Grade.A},
        new Enrollment{StudentID=1,CourseID=4022,Grade=Grade.C},
        new Enrollment{StudentID=1,CourseID=4041,Grade=Grade.B},
        new Enrollment{StudentID=2,CourseID=1045,Grade=Grade.B},
        new Enrollment{StudentID=2,CourseID=3141,Grade=Grade.F},
        new Enrollment{StudentID=2,CourseID=2021,Grade=Grade.F},
        new Enrollment{StudentID=3,CourseID=1050},
        new Enrollment{StudentID=4,CourseID=1050,},
        new Enrollment{StudentID=4,CourseID=4022,Grade=Grade.F},
        new Enrollment{StudentID=5,CourseID=4041,Grade=Grade.C},
        new Enrollment{StudentID=6,CourseID=1045},
        new Enrollment{StudentID=7,CourseID=3141,Grade=Grade.A},
        };
        context.Enrollments.AddRange(enrollments);
        context.SaveChanges();
    }
}
```

```csharp
// Add services to the container.
builder.Services.AddControllersWithViews();

// 加入 DbContext
builder.Services.AddDbContext<SchoolContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));



var app = builder.Build();  
            // 呼叫 Seed 方法
using (var scope = app.Services.CreateScope())
{
    var services = scope.ServiceProvider;
    var context = services.GetRequiredService<SchoolContext>();

    try
    {
        SchoolInitializer.Initialize(context);
    }
    catch (Exception ex)
    {
        // 可以記錄錯誤
        Console.WriteLine($"初始化資料庫時發生錯誤: {ex.Message}");
    }
}
```

<br>
<br>

## 🎃 OnModelCreating 是什麼？

OnModelCreating 是你告訴 EF：「這些資料表的細節，我要自己定義。」
你可以在這裡清楚告訴 EF：我要怎麼把 C# 的 class 對應到資料庫的 table 和欄位。
我們在繼承 DbContext 的資料上下文中 override 它
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // 各種設定放這裡
}
```


這是 Entity Framework 提供的一個模型建構的「擴充點」，用來：

- 自訂資料表名稱、欄位名稱，這樣資料表就叫 StudentTable，欄位叫 StudentName，不會自動加 s 或亂推論。
```csharp
public class Student
{
    public int Id { get; set; }
    public string FullName { get; set; }
}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Student>(entity =>
    {
        entity.ToTable("StudentTable"); // 自訂資料表名稱
        entity.Property(e => e.FullName)
              .HasColumnName("StudentName"); // 自訂欄位名稱
    });
}
```
- 定義主鍵、索引、關聯（關聯式資料庫的關係）
```csharp
modelBuilder.Entity<Student>(entity =>
{
    entity.HasKey(e => e.Id); // 設定主鍵欄位
});

modelBuilder.Entity<Student>(entity =>
{
    entity.HasIndex(e => e.FullName); // 建立索引
});

public class Student
{
    public int Id { get; set; }
    public string Name { get; set; }

    public List<Course> Courses { get; set; }
}

public class Course
{
    public int Id { get; set; }
    public string Title { get; set; }

    public int StudentId { get; set; }              // 外鍵
    public Student Student { get; set; }            // 導覽屬性
}

modelBuilder.Entity<Course>(entity =>
{
    entity.HasOne(c => c.Student)             // 一門課 有一個學生
          .WithMany(s => s.Courses)           // 一個學生 有很多課
          .HasForeignKey(c => c.StudentId);   // 外鍵欄位
});
```

<br>
<br>

## 🎃 DbContextOptions

是告訴 DbContext 要連哪個資料庫、用什麼方式連的「設定容器」。這段 constructor 會被 ASP.NET Core DI（依賴注入）自動呼叫，把連線資訊傳進來。
```csharp
public SchoolContext(DbContextOptions<SchoolContext> options) : base(options)
{
}
```

<br>
<br>

## LocalDB

LocalDB 是開發者用的輕量 SQL Server，他是開發測試用、免安裝完整版 SQL Server，可在本機開發、快速建立 DB

```JSON
"ConnectionStrings": {
  "DefaultConnection": "Server=(localdb)\\MSSQLLocalDB;Database=ContosoUniversity;Trusted_Connection=True;"
}
```


| 項目                              | 意義                     |
| ------------------------------- | ---------------------- |
| `Server=(localdb)\MSSQLLocalDB` | 指向本機 SQL Express 版     |
| `Trusted_Connection=True`       | 用目前 Windows 使用者登入，不需帳密 |

<br>
<br>

## 🎃 如何檢查 LocalDB 是否安裝

```POWERSHELL
sqllocaldb i 
## MSSQLLocalDB
```

<br>
<br>

## 🎃 連線伺服器 (Connect to Server)

Server name: (localdb)\MSSQLLocalDB

<br>
<br>

## 🎃 Index.cshtml view display 範例

```csharp
@model IEnumerable<ContosoUniversity.Models.Student>

@{
    ViewData["Title"] = "Index";
}

<h1>Index</h1>

<p>
    <a asp-action="Create">Create New</a>
</p>
<table class="table">
    <thead>
        <tr>
            <th>
                @Html.DisplayNameFor(model => model.LastName)
            </th>
            <th>
                @Html.DisplayNameFor(model => model.FirstMidName)
            </th>
            <th>
                @Html.DisplayNameFor(model => model.EnrollmentDate)
            </th>
            <th></th>
        </tr>
    </thead>
    <tbody>
@foreach (var item in Model) {
        <tr>
            <td>
                @Html.DisplayFor(modelItem => item.LastName)
            </td>
            <td>
                @Html.DisplayFor(modelItem => item.FirstMidName)
            </td>
            <td>
                @Html.DisplayFor(modelItem => item.EnrollmentDate)
            </td>
            <td>
                <a asp-action="Edit" asp-route-id="@item.ID">Edit</a> |
                <a asp-action="Details" asp-route-id="@item.ID">Details</a> |
                <a asp-action="Delete" asp-route-id="@item.ID">Delete</a>
            </td>
        </tr>
}
    </tbody>
</table>

```

<br>
<br>

## 🎃 使用 Scaffold 自動產生 MVC Controller 和 View（搭配 Entity Framework）

![alt text](https://github.com/CHI-KEKE/pics/blob/main/MVC/add-scaffold.png?raw=true)
- 在 Solution Explorer 中，右鍵點擊 Controllers 資料夾，選擇【Add > New Scaffolded Item（新增 Scaffold 項目）】。
- 在「Add Scaffold」對話框中，選擇：
    - MVC 5 Controller with views, using Entity Framework（使用 Entity Framework 的 MVC 5 控制器與 View）
    - 點選 [Add]（新增），接著在「Add Controller」對話框中做設定
    - 點擊 [Add]（新增） 後，Scaffolder 會自動幫你產生相關內容

這些 View 搭配 Controller，就能完成一個基本的 CRUD 畫面。

<br>
<br>

## 🎃 參考教學

{% btn 'https://learn.microsoft.com/en-us/aspnet/mvc/overview/getting-started/getting-started-with-ef-using-mvc/creating-an-entity-framework-data-model-for-an-asp-net-mvc-application',Tutorial: Get Started with Entity Framework 6 Code First using MVC 5,far fa-hand-point-right %}


Accessing Your Model's Data from a New Controller
https://learn.microsoft.com/zh-tw/aspnet/mvc/overview/getting-started/introduction/accessing-your-models-data-from-a-controller


Examining the Edit Action Methods and Views for the Movie Controller
https://learn.microsoft.com/en-us/aspnet/mvc/overview/getting-started/introduction/examining-the-edit-methods-and-edit-view


