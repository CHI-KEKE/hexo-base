---
title: Middleware - Sequence
date: 2025-09-26 17:07:34
categories: Middleware
top_img: https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
cover : https://github.com/CHI-KEKE/pics/blob/main/WebAPI/tree.png?raw=true
tags:
    - Middleware
toc:
toc_number:
comments :
---

{% tabs Sequence%}

![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Sequence/landing.png)


<!-- tab 關於順序-->


關於 Middleware 的順序，要把「會影響後面所有處理的規則」放前面、把「需要前面資訊才做得了的事」放後面，這樣每個 middleware 才拿得到它該拿的上下文，而且不會做白工或做錯事，順序決定了「誰先看到 Request」、「誰能先改/短路 Response」、「誰能拿到路由/使用者/權限資訊」


![s](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Sequence/core_of_sequence.png)


<!-- endtab -->

<!-- tab ExceptionHandler（最外層）-->

他把後面任何地方炸掉的例外「接住」，統一回應格式/錯誤頁，放最前是因為它要能包住整條管線，才能攔到所有後面丟出的 exception，假如放在後面，前面炸了它接不到，你就會看到預設 500、或連你想要的錯誤 JSON 都回不來


![ex](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Sequence/exception_first.png)


<!-- endtab -->

<!-- tab HSTS（嚴格要求瀏覽器以後都用 HTTPS-->

HSTS 作用是在 HTTPS 回應裡加上 Strict-Transport-Security header，讓瀏覽器記住「以後別再用 HTTP」，為什麼在 HttpsRedirection 之前是因為 HSTS 是「回應 header」，你希望只要是 HTTPS 的回應（包含後面成功/失敗）都能帶上這個 header，如果放太後面，就只有少數回應才帶到 header（例如被前面某個 middleware 提前 short-circuit 就帶不到），HSTS 效果打折。HSTS 通常只在 Production 開，避免開發環境把瀏覽器「鎖死」成只能 HTTPS


![hsts](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Sequence/hsts.png)


<!-- endtab -->

<!-- tab HttpsRedirection（把 HTTP 導去 HTTPS）-->

如果是 HTTP，直接 301/307 轉到 HTTPS，後面就不用處理了（短路），這是「安全前置」，你不想讓任何認證/授權/業務邏輯在 HTTP 上跑，所以如果放後面，可能已經跑完 routing、甚至跑到 auth 才 redirect，浪費資源，還可能在 HTTP 上暴露行為差異


![redirection](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Sequence/httpredirection.png)


<!-- endtab -->

<!-- tab Static Files（靜態檔案：js/css/images）-->

遇到 /css/site.css 這類請求，直接回檔案，不必進 MVC/API，為什麼在 Routing 前面是因為靜態檔案大多不需要路由解析與授權，越早回越省也避免被路由規則「誤攔截」，如果放 Routing 後每個靜態檔都要走一輪路由匹配等於是浪費，甚至被某些路由吃掉導致 404 或回錯內容


![static](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Sequence/static_file.png)


<!-- endtab -->

<!-- tab Routing（建立 Endpoint 資訊）-->

這裡根據路徑/方法，算出「這次要打到哪個 endpoint（controller action / minimal api）」並把結果放進 HttpContext。為什麼要在 CORS/Auth 之前是因為很多策略（尤其 CORS policy、Auth policy）需要知道「目標 endpoint 是誰」才能套用正確規則，所以如果放後面，CORS 可能不知道該用哪個 policy，Auth/Authorization 也可能無法拿到 endpoint metadata，導致策略不生效或全走預設


![Routing](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Sequence/routing.png)


<!-- endtab -->

<!-- tab CORS（跨網域規則）-->

用於處理跨域需求（含 preflight OPTIONS），加上 Access-Control-* headers。為什麼在 Authentication/Authorization 前面是因為 Preflight OPTIONS 通常不該被要求登入/授權，不然瀏覽器根本發不出真正的請求，CORS header 必須出現在回應上，否則前端就算拿到 200 也會被瀏覽器擋掉，如果放在 auth 之後，結果就是 OPTIONS 被 401/403，前端看起來像「API 壞了」但其實是 CORS 順序錯。

![cors](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Sequence/cors_trap.png)


<!-- endtab -->

<!-- tab Authentication（驗證你是誰）-->

用於解析 token/cookie，建立 HttpContext.User（ClaimsPrincipal），她為什麼在 Authorization 前面是因為授權要根據「你是誰、你有哪些 claim/role」來判斷，所以如果放在授權後面，授權看到的 user 可能還是匿名，結果你明明帶了 token 也會被判 401/403


![authen](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Sequence/authentication.png)


![mix](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Sequence/athen_autho.png)


<!-- endtab -->


<!-- tab Authorization（你能不能做這件事）-->

根據 endpoint 的 [Authorize]、policy、role 等，決定放行或擋下，它是「進門最後一道門禁」，要在真的進到 action/minimal api 之前先做完。如果放到 endpoint 後等於先讓人進去執行了才說不行，門禁直接失效（或變成你要在每個 action 自己寫判斷，超難維護）

![autho](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Sequence/authorization.png)


<!-- endtab -->

<!-- tab Custom middlewares（你自己插的）-->

是 request/response logging、transaction id、租戶辨識、rate limiting、審計、特定 header 檢查…，為什麼多放在 Authorization 後、Endpoint 前是因為，這時候已經拿到 HttpContext.User（知道誰）、也知道目標 endpoint（知道要去哪），但還沒真正執行業務端點，所以適合做「決策、封鎖、統計、補充上下文」，但也不是絕對，假如想記錄所有請求（含靜態檔）就放更前面，只想記錄 API 且要 user 資訊就放 auth 之後，想擋掉超大 body 放更前面（避免讀取浪費）


![custom](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Sequence/custom.png)


![end](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Sequence/endpoint.png)


<!-- endtab -->

<!-- tab 實境演練-->

前端 https://app.example.com 呼叫 API https://api.example.com/orders，帶 JWT


瀏覽器先發 OPTIONS /orders（preflight），正確順序會這樣

1. Routing 知道是 /orders → CORS 放行並回 CORS headers → 直接 204（不需要 auth）→ 前端才會送真正的 GET /orders，而錯誤順序（Auth 在 CORS 前）：OPTIONS 被當成未登入 → 回 401 → 前端永遠打不到 GET /orders
2. 真正的 GET /orders, Authentication 先把 JWT 解析成 User, Authorization 看 endpoint 要求 role=Admin，不符合就 403
3. custom middleware（例如 audit log）可以記「誰」在打「哪個 endpoint」，成功/失敗都記得到

<!-- endtab -->



<!-- tab Short-circuit（短路）會讓「後面永遠不執行」-->

把會短路的東西放前面是省資源，但也代表後面的功能（例如加 header、統計、壓縮）可能永遠吃不到那種回應。可能發生我有寫某 middleware 怎麼沒生效」，其實是前面 StaticFiles/HttpsRedirection/CORS preflight 已經回了，後面根本沒跑。


大致上的通用順序是這樣
```csharp
// 1. 錯誤處理 & HTTPS
app.UseExceptionHandler("/Home/Error");
app.UseHsts();

// 2. 基礎 Middleware
app.UseHttpsRedirection();
app.UseStaticFiles();

// 3. Routing
app.UseRouting();

// 4. 驗證 & 授權 (需要在 Routing 之後，但在 Endpoints 之前)
app.UseAuthentication();
app.UseAuthorization();

// 5. 對應 Endpoints
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

```


![cheet](https://raw.githubusercontent.com/CHI-KEKE/pics/refs/heads/main/Code_Design/Middleware/Sequence/whole.png)


<!-- endtab -->


{% endtabs %}