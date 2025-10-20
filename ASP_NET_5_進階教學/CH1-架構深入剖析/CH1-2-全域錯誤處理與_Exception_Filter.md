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
            app.UseExceptionHandler("/Home/Error");
            // The default HSTS value is 30 days. You may want to change this for production scenarios, see https://aka.ms/aspnetcore-hsts.
            app.UseHsts();
        }

        // 自訂「Correlation Id」中介層（選配，見下方）
        app.UseMiddleware<CorrelationIdMiddleware>();

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
        var request = ctx.HttpContext.Request;
        var accept = request.Headers["Accept"].ToString();
        
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

## 7. Correlation Id（跨系統追蹤）🧭

```csharp
public class CorrelationIdMiddleware
{
    public const string HeaderName = "X-Correlation-ID";
    private readonly RequestDelegate _next;

    public CorrelationIdMiddleware(RequestDelegate next) => _next = next;

    public async Task Invoke(HttpContext context)
    {
        var correlationId = context.Request.Headers[HeaderName].FirstOrDefault()
                            ?? Guid.NewGuid().ToString("N");
        context.Response.Headers[HeaderName] = correlationId;

        // 也可寫入 Items 供後續 Filter/Controller 使用
        context.Items[HeaderName] = correlationId;

        await _next(context);
    }
}
```

> 日誌結合：在 `ILogger` 的 scope 中把 Correlation Id 帶上，查問題時非常好用。

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
        if (y == 0) throw new DomainException("除數不可為 0", 400);
        var result = x / y;
        return View(model: result); // 對 HTML 頁面
    }

    [HttpGet("/api/calc")]
    public IActionResult CalcApi([FromQuery]int x, [FromQuery]int y)
    {
        if (y == 0) throw new DomainException("Divide by zero", 400);
        return Ok(new { result = x / y });
    }
}
```

- `GET /calc/10/0`

  - HTML：會被 `GlobalExceptionFilter` 轉向到 `/Error/400` → 顯示 400 錯誤頁

- `GET /api/calc?x=10&y=0`

  - JSON：回傳 `ProblemDetails`（`400` 與 `detail`）

---

## 9. 回應格式與內容協商（HTML / JSON）🧪

- 檢查 `Accept` header 或 AJAX header (`X-Requested-With`)
- MVC：`View()` for HTML、`ObjectResult(ProblemDetails)` for JSON
- API / 前後端分離：使用 `application/problem+json` 搭配 `ProblemDetails`（標準化）

---

## 10. 404/403 等非例外狀態碼處理 🪪

- `UseStatusCodePagesWithReExecute("/Error/{0}")`
- `ErrorController.ErrorByCode(int code)` 負責渲染 403/404/405…
- JSON 情境下仍可回 `ProblemDetails`，加上 `TraceIdentifier` 幫助追蹤。

---

## 11. 安全與實務守則 🔐

- **Production 禁用** `DeveloperExceptionPage`。
- 不要在錯誤頁或 JSON 回應中曝露 **堆疊、連線字串、檔案路徑**。
- 針對常見領域錯誤（驗證失敗、資源不存在、權限不足），**用自訂例外 + 對應狀態碼**。
- 所有錯誤**記錄日誌**，至少包含：`TraceId/CorrelationId`、`Route`、`UserId`（若有）。
- 對外 API 統一 `ProblemDetails`；前端可依 `status` 與 `title/detail` 呈現。
- 不要濫用例外處理流程來做業務邏輯分支；預期錯誤用**回傳值**或**模型驗證**更好。

---