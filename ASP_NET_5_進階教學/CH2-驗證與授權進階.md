# 第 2 章｜驗證與授權進階

---

## 2.1 身分驗證（Authentication）基礎概念

🔐 **目標**：理解 ASP.NET Core 的身份驗證架構，包括 Scheme、Handler、Challenge、Forbid 等核心概念。

### 什麼是「Scheme」與「Handler」

* **Scheme**：一種命名配置，定義要使用哪種認證機制（如 Cookies、JWT Bearer、API Key）。
* **Handler**：Scheme 的實作類別，負責實際驗證邏輯（如解析 Token、查使用者、發出 Challenge）。
* **Challenge / Forbid**：

  * `Challenge`：當需要登入但尚未登入時（例如 `[Authorize]` 的要求）。
  * `Forbid`：已登入但沒有授權時。

### 範例：最小化的 Authentication 設定

```csharp
public void ConfigureServices(IServiceCollection services)
{
    //1. BasicAuth -> 給這個驗證一個名稱「BasicAuth」，可以加入多個不同的驗證，以不同名稱分隔即可。
    //2. AuthenticationSchemeOptions -> 想要驗證什麼內容，什麼KEY，預設驗證的東西很少。
    //3. BasicAuthHandler -> 自定義你的驗證方法。
    services.AddAuthentication("BasicAuth")
        .AddScheme<AuthenticationSchemeOptions, BasicAuthHandler>("BasicAuth", null);

    services.AddAuthorization();
    services.AddControllers();
}

public void Configure(IApplicationBuilder app)
{
    app.UseRouting();
    //...
    //....
    
    // UseAuthentication 必定要在 UseAuthorization 之前
    app.UseAuthentication();
    app.UseAuthorization();
    
    //...
    //....

    app.UseEndpoints(endpoints => endpoints.MapControllers());
}
```

---

## 2.2 自訂驗證處理器（AuthenticationHandler）

⚙️ **目標**：自行撰寫一個 `BasicAuthHandler`，驗證 Header 中的帳密。

### 範例程式：Basic Authentication Handler

```csharp
public class BasicAuthHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    public BasicAuthHandler(
        IOptionsMonitor<AuthenticationSchemeOptions> options,
        ILoggerFactory logger,
        UrlEncoder encoder,
        ISystemClock clock)
        : base(options, logger, encoder, clock)
    { }

    //驗證時 HandleAuthenticateAsync 會被 .Net 呼叫，因此裡面就是你的驗證邏輯
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.ContainsKey("Authorization"))
            return Task.FromResult(AuthenticateResult.Fail("Missing Authorization Header"));

        var header = Request.Headers["Authorization"].ToString();
        if (!header.StartsWith("Basic ", StringComparison.OrdinalIgnoreCase))
            return Task.FromResult(AuthenticateResult.Fail("Invalid Scheme"));

        var credentials = Encoding.UTF8.GetString(Convert.FromBase64String(header.Substring(6))).Split(':');
        if (credentials.Length != 2)
            return Task.FromResult(AuthenticateResult.Fail("Invalid Credentials"));

        var username = credentials[0];
        var password = credentials[1];

        if (username != "admin" || password != "1234")
            return Task.FromResult(AuthenticateResult.Fail("Unauthorized"));

        var claims = new[] { new Claim(ClaimTypes.Name, username) };
        var identity = new ClaimsIdentity(claims, Scheme.Name);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, Scheme.Name);

        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

**說明**：

* 驗證通過後會產生 `ClaimsPrincipal`，內含身份資訊。
* `[Authorize]` 就是依據此 `User` 物件判斷是否已登入。
* 若失敗，系統會回應 `401 Unauthorized`。

**測試方式：**

```
GET /api/secure
Header: Authorization: Basic YWRtaW46MTIzNA==
```

---

## 2.3 JWT（JSON Web Token）驗證

📜 **目標**：學習使用 `AddJwtBearer()` 建立 Bearer Token 架構，理解 Token 結構、簽章、時效與刷新機制。

### JWT 結構

JWT 是由三段組成的 Base64 字串：

```
Header.Payload.Signature
```

* **Header**：指定演算法與型別（通常為 `HS256`）
* **Payload**：放入使用者 Claims（例如帳號、角色、時效）
* **Signature**：由前兩段 + Secret Key 雜湊簽章驗證完整性

### Startup 設定

```csharp
public void ConfigureServices(IServiceCollection services)
{
    var key = Encoding.UTF8.GetBytes("SuperSecretKeyForJwtDemo");
    services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(options =>
        {
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidateAudience = true,
                ValidateIssuerSigningKey = true,
                ValidIssuer = "MyApp",
                ValidAudience = "MyAppUsers",
                IssuerSigningKey = new SymmetricSecurityKey(key),
                ClockSkew = TimeSpan.Zero // 減少時間差允許
            };
        });

    services.AddControllers();
}
```

### 建立 Token 範例

```csharp
[ApiController]
[Route("api/token")]
public class TokenController : ControllerBase
{
    [HttpPost]
    public IActionResult CreateToken(string user傳送給Server用來產生驗證Token的資料) 
    {
        var claims = new[]
        {
            new Claim(ClaimTypes.Name, "Ada"),
            new Claim(ClaimTypes.Role, "Admin")
        };
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes("SuperSecretKeyForJwtDemo"));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        var token = new JwtSecurityToken(
            issuer: "MyApp",
            audience: "MyAppUsers",
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(30),
            signingCredentials: creds);
        
//WriteToken(token) 回傳後，Client 端(通常是App/Server to Server 這一類沒有 cookie 機制的應用程式)，就拿到之後每次溝通用的Token了。
//每次跟Api Server溝通時，都要傳送這個 Token 過來進行驗證，Token很重要，被偷了就可以假冒他人對Server進行 API 存取，等同就是帳號密碼的一種。
//其實台糖單一入口機制的 V 值也可「算是」驗證Token的一種，只是我們並不完全依賴 V 值進行驗證。
        return Ok(new { token = new JwtSecurityTokenHandler().WriteToken(token) });
    }
}
```

**測試：**

```
POST /api/token → 取得 token
GET /api/secure → Header: Authorization: Bearer <token>
```

---

## 2.4 授權策略與多層控制

🛂 **目標**：區分 Policy、Roles、Claims，不只控制「登入與否」，還能針對「誰能做什麼」。

### 基礎授權（Roles & Claims）

```csharp
//[Authorize]就是啟動驗證的重點
[Authorize(Roles = "Admin")]
[ApiController]
[Route("api/admin")]
public class AdminController : ControllerBase
{
    [HttpGet]
    public IActionResult Dashboard() => Ok("Welcome Admin!");
}
```

### Policy-based 授權

```csharp
services.AddAuthorization(options =>
{
    options.AddPolicy("SeniorOnly", policy =>
        policy.RequireClaim("Level", "Senior", "Leader"));
});
```

```csharp
[Authorize(Policy = "SeniorOnly")]
[HttpGet("reports")]
public IActionResult Reports() => Ok("Reports for Senior or Leader only");
```

### 自訂授權需求（Requirement / Handler）

```csharp
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int Age { get; }
    public MinimumAgeRequirement(int age) => Age = age;
}

public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, MinimumAgeRequirement requirement)
    {
        if (context.User.HasClaim(c => c.Type == ClaimTypes.DateOfBirth))
        {
            var birth = DateTime.Parse(context.User.FindFirst(ClaimTypes.DateOfBirth)!.Value);
            var age = DateTime.Today.Year - birth.Year;
            if (age >= requirement.Age)
                context.Succeed(requirement);
        }
        return Task.CompletedTask;
    }
}
```

註冊：

```csharp
services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
services.AddAuthorization(options =>
{
    options.AddPolicy("AdultOnly", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(18)));
});
```

使用：

```csharp
[Authorize(Policy = "AdultOnly")]
public IActionResult BuyAlcohol() => Ok("You are old enough!");
```

---

## 2.5 防偽機制（Anti-Forgery）與 CSRF

🧩 **目標**：理解 CSRF（跨站請求偽造）風險，知道如何在 MVC 與 SPA 架構中防護。

### 背景知識

CSRF（Cross-Site Request Forgery，跨站請求偽造）攻擊：
1. 你剛登入了「A銀行」的網站，此時你的瀏覽器是有合法的 cookie ，可以進行轉帳之類的行為。
2. 同時你又開啟了某個惡意網站，這個網站有個超連結點了會讓你執行「A銀行」的轉帳命令，如：
```html
<img src="https://bank.com/transfer?to=03827&amount=9999" />

<!--或是-->

<form action="https://bank.com/transfer" method="POST" id="f">
  <input name="to" value="03827" />
  <input name="amount" value="9999" />
</form>
<script>document.getElementById('f').submit();</script>
```

3. 你只是開個圖片或點個連結，你就轉了9999給我了。
### 傳統 MVC 防護

```csharp
services.AddControllersWithViews().AddRazorRuntimeCompilation();

[HttpPost]
[ValidateAntiForgeryToken]
public IActionResult SubmitForm(FormModel model)
{
    // 驗證通過才執行
    return RedirectToAction("Success");
}
```

### SPA（前後端分離）策略

* 採用 **JWT Bearer**（無 Cookie）→ 自然無 CSRF 風險。
* 若仍需 Cookie，請設定：

  * `SameSite=Strict` 或 `Lax`
  * 加上 `XSRF-TOKEN` header 驗證
  * 僅允許 `POST/PUT/DELETE` 透過自家網域送出

---

## 2.7 多層授權控制實務

🏗️ **目標**：整合不同層級授權，從端點層到業務層一致控管。

| 層級                     | 控制手段                                 | 範例         |
|--------------------------|------------------------------------------|--------------|
| **Endpoint**             | `[Authorize]`                            | 限制整個 API |
| **Controller / Action**  | `[Authorize(Policy="AdminOnly")]`        | 特定功能     |
| **Service / Repository** | `IAuthorizationService.AuthorizeAsync()` | 資料列驗證   |
| **前端**                 | 禁止顯示按鈕、導頁                        | UI 顯示邏輯  |

---

## 2.8 小結

* **Authentication**：誰你說你是誰。
* **Authorization**：你能做什麼。
* **JWT** 適合 API 架構，輕量、跨平台。
* **Policy/Handler** 提供彈性授權策略。
* **CSRF 防護** 在 Cookie 模式中必須配置。
* **多層授權** 可強化安全縱深。

---
