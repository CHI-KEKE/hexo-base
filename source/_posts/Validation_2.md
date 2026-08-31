



在資料綁定 (Model Binding) 過程沒有收到資料時，
會將結構型別 (struct type) 使用初始值當作預設值，
參考型別 (reference type) 的話則會顯示 null。

一般常見的 結構型別如 int、double、float、DateTime 等。
參考型別則如 string 及一般的 Class。

但 ASP.Net Core 有一個新玩意兒可以解決 - [BindRequired]，
設定方法只要把它原封不動掛上去可以了。


```csharp

public class Book
{
    [BindRequired]
    public int Id { get; set; }

    [Required]
    public string Title{ get; set; }
        
    public DateTime PublishDate { get; set; }
}


```



首先在 API Controller 中預設都會套用 [ApiController] 這個屬性，那套用這個屬性對於 Model Binding 有什麼影響呢？
套用了 [ApiController] 會有下列影響


必須要使用屬性路由 (Attribute Routing)
只要發生模型驗證失敗，就會自動回應 HTTP 400 (Bad Request)
自動套用模型繫結的預設規則
複雜型別預設就會自動套用 [FromBody] 屬性
參數型別如果是 IFormFile 或 IFormFileCollection 的話會自動套用 [FromForm] 屬性，且該屬性也只能用在這兩個型別
簡單模型或任何其他參數，全部都會自動套用 [FromQuery] 屬性。例如「路由參數」預設會自動套用 [FromQuery]


Model Binding 也有以下來源屬性可以設定

[FromQuery] — 只會從 Query String 中取值。預設只會套用在簡單模型上
[FromRoute] — 只會從路由資料取值。預設只會套用簡單模型，實務上通常不會特別套用
[FromForm] — 從 Requet body 取得值並且只會套用在 IFrom 或是 IFormFileCollection 複雜模型。
[FromBody] — 只會從 Request body 取值。預設套用在複雜模型。
[FromHeader] — 取得 Request header 的值並只能套用簡單模型。
[FromService] — 從 DI 容器中取得服務物件。


