# ASP.NET Core 5 MVC：全域錯誤處理與 Exception Filter 教學 📘

## 1. 為什麼需要「全域錯誤處理」？🧯

- **一致性**：所有 Controller/Action 都能產生一致的錯誤頁或 JSON，避免各自為政。
- **安全性**：藏起堆疊、連線字串等敏感資訊。
- **可觀測性**：統一紀錄例外、加入 Correlation Id 以追蹤跨服務請求。
- **使用者體驗**：友善的錯誤頁、API 則用標準化 `ProblemDetails`。

## 2. 專案骨架與 Startup 設定 🧱

### Startup.cs（.NET 5 標準寫法）

```csharp
// Startup.cs
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddControllersWithViews(options =>
        {
            // 全域註冊自訂 Exception Filter（下面會實作）
            options.Filters.Add<GlobalExceptionFilter>();
        })       
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env, ILogger<Startup> logger)
    {
        if (env.IsDevelopment())
        {
            // Dev 顯示開發者例外頁（內含堆疊），勿於正式環境啟用
            app.UseDeveloperExceptionPage();
        }
        else
        {
            //使用自訂錯誤處理時，就不要再用 .Net 預設的
            //app.UseExceptionHandler("/Home/Error");
            
            // The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
            app.UseHsts();
        }

        // 自訂錯誤處理
        app.UseMiddleware<GlobalExceptionHandlingMiddleware>();

        app.UseStaticFiles();
        app.UseRouting();

        app.UseAuthorization();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapDefaultControllerRoute();
        });
    }
}
```
---

## 3. 自訂 Exception Filter（發生在Controller內的錯誤處理）🛡️

###  GlobalExceptionFilter

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;
using Microsoft.Extensions.Logging;

public class GlobalExceptionFilter : IExceptionFilter
{
    private readonly ILogger<GlobalExceptionFilter> _logger;

    public GlobalExceptionFilter(ILogger<GlobalExceptionFilter> logger)
    {
        _logger = logger;
    }

    public void OnException(ExceptionContext context)
    {
        var ex = context.Exception;

        // 記錄錯誤（含 Correlation Id）
        var traceId = context.HttpContext.TraceIdentifier;
        _logger.LogError(ex, "Unhandled exception. TraceId: {TraceId}", traceId);
        
        // 轉跳回單一入口，或是某個你指定的路徑
        context.Result = MakeResult(context, ex.Message, traceId);
        context.ExceptionHandled = true;
    }

    private IActionResult MakeResult(ExceptionContext ctx, string message, string traceId)
    {
        return Redirect("https://ap.taisugar.com.tw/單一入口網址");
    }    
}
```

> 已在 `Startup.cs` 中透過 `options.Filters.Add<GlobalExceptionFilter>();` 全域註冊。

---

## 4. 自訂 Exception Handling Middleware（發生在Controller之前的錯誤處理）🧵

若你想在 **進 MVC 之前** 就攔截所有例外（含靜態檔案、非 MVC 管線），可以用自訂 Middleware。

> 一般和 `UseExceptionHandler` 擇一；若兩者並存，請務必清楚呼叫順序與責任分工。

```csharp
public class GlobalExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionHandlingMiddleware> _logger;

    public GlobalExceptionHandlingMiddleware(RequestDelegate next, ILogger<GlobalExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task Invoke(HttpContext context)
    {
        try
        {
            //如果都沒發生錯誤就一路往下執行，直至 Controller/Action
            await _next(context);
            //請求完成要做的事
        }        
        catch (Exception ex)
        {
            //還沒到 Controller 之前就發生錯誤了
            _logger.LogError(ex, "Unhandled exception");
            await WriteProblem(context, 500, "Unexpected server error.");
        }
    }

    private static async Task WriteProblem(HttpContext ctx, int statusCode, string detail)
    {
        ctx.Response.StatusCode = statusCode;
        ctx.Response.ContentType = "application/problem+json";
        
        //標準化 ProblemDetails
        var payload = System.Text.Json.JsonSerializer.Serialize(new
        {
            title = "Request Error",
            status = statusCode,
            detail,
            instance = ctx.TraceIdentifier
        });
        await ctx.Response.WriteAsync(payload);
    }
}
```

> 註冊：`app.UseMiddleware<GlobalExceptionHandlingMiddleware>();`
> 注意：與 `UseExceptionHandler` 擇一；若使用 `UseExceptionHandler` + `ErrorController`，通常不需要這個自訂中介層。

---

## 8. Controller 範例（觸發例外、Ajax 與 HTML）🧪

```csharp
// Controllers/SampleController.cs
using Microsoft.AspNetCore.Mvc;

public class SampleController : Controller
{
    [HttpGet("/calc/{x:int}/{y:int}")]
    public IActionResult Calc(int x, int y)
    {
        if (y == 0) throw new Exception("除數不可為 0", 400);
        var result = x / y;
        return View(model: result); // 對 HTML 頁面
    }

    [HttpGet("/api/calc")]
    public IActionResult CalcApi([FromQuery]int x, [FromQuery]int y)
    {
        if (y == 0) throw new Exception("Divide by zero", 400);
        return Ok(new { result = x / y });
    }
}
```

- `GET /calc/10/0`

  - HTML：會被 `GlobalExceptionFilter` 轉向到 `/Error/400` → 顯示 400 錯誤頁

- `GET /api/calc?x=10&y=0`

  - JSON：回傳 `ProblemDetails`（`400` 與 `detail`）

---