# Chapter 5｜查詢翻譯與 SQL 映射（Query Translation & SQL Mapping）

## 5.1 什麼是「查詢翻譯」？

當你在 EF Core 中寫 LINQ 時，系統會：

1. **解析 LINQ 樹 (Expression Tree)**
   → 分析 Select、Where、Join、GroupBy 等節點。
2. **判斷哪些節點可翻譯成 SQL**。
3. **組合成 SQL 命令字串**（送給 Provider，如 SQL Server）。
4. **執行 SQL、反序列化成實體**。

例如：

```csharp
var query = ctx.Orders
    .Where(o => o.Total > 100)
    .Select(o => new { o.Id, o.Customer.Name });
```

翻譯後的 SQL 可能是：

```sql
SELECT [o].[Id], [c].[Name]
FROM [Orders] AS [o]
INNER JOIN [Customers] AS [c] ON [o].[CustomerId] = [c].[Id]
WHERE [o].[Total] > 100
```

---

## 5.2 Client Evaluation：隱形的效能殺手

在 EF Core 2.x 時期，任何無法轉成 SQL 的部分都會在**記憶體端執行**（client-side evaluation）。
這樣做有時候無害，但一不小心就會讓整張表拉進記憶體。

---

### 5.2.1 錯誤示範

```csharp
var expensiveOrders = ctx.Orders
    .Where(o => IsExpensive(o.Total)) // ⚠️ 無法轉成 SQL！
    .ToList();
```

這裡的 `IsExpensive()` 是 C# 方法，資料庫不懂它。
結果：EF 先 SELECT 整張 Orders → 再在記憶體中執行篩選。

```sql
SELECT * FROM [Orders];
```

然後才在 C# 迴圈裡比對 → 你的伺服器爆了。

---

### 5.2.2 正確作法：改成 SQL 能懂的表示法

```csharp
var expensiveOrders = ctx.Orders
    .Where(o => o.Total > 1000) // ✅ 能翻譯成 SQL
    .ToList();
```

---

### 5.2.3 如果真的需要呼叫 .NET 函數？

可以使用 `EF.Functions` 提供的內建轉譯函數：

```csharp
var result = await ctx.Products
    .Where(p => EF.Functions.Like(p.Name, "%Sugar%"))
    .ToListAsync();
```

這會翻成：

```sql
WHERE [p].[Name] LIKE N'%Sugar%'
```

---

### 5.2.4 客製 SQL 對應的 .NET 函數（UDF 映射）

假設你有 SQL Server 的函數：

```sql
CREATE FUNCTION dbo.DiscountPrice(@price decimal(10,2))
RETURNS decimal(10,2)
AS
BEGIN
    RETURN @price * 0.9
END
```

你可以在 EF 模型裡映射它：

```csharp
public static class DbFunctionsExtensions
{
    [DbFunction("DiscountPrice", "dbo")]
    public static decimal DiscountPrice(decimal price) => throw new NotSupportedException();
}
```

使用：

```csharp
var query = ctx.Products
    .Select(p => new {
        p.Name,
        Discount = DbFunctionsExtensions.DiscountPrice(p.Price)
    });
```

生成 SQL：

```sql
SELECT [p].[Name], [dbo].[DiscountPrice]([p].[Price]) AS [Discount]
FROM [Products] AS [p];
```

---

## 5.3 原生 SQL 查詢（FromSqlRaw / FromSqlInterpolated）

有時候，你需要更精確控制 SQL，或使用存放程序、視圖。
EF Core 提供了兩個方法：

| 方法                                         | 特性                        |
|----------------------------------------------|-----------------------------|
| `FromSqlRaw("SELECT ...")`                   | 原生 SQL（需手動參數化）      |
| `FromSqlInterpolated($"SELECT ... {param}")` | 會自動轉成參數化 SQL，較安全 |

---

### 5.3.1 基本範例

```csharp
var recentOrders = ctx.Orders
    .FromSqlInterpolated($"SELECT * FROM Orders WHERE CreatedAt > {DateTime.UtcNow.AddDays(-7)}")
    .ToList();
```

翻譯後 SQL：

```sql
exec sp_executesql
N'SELECT * FROM Orders WHERE CreatedAt > @p0',
N'@p0 datetime2(7)',
@p0='2025-10-08T00:00:00'
```

---

### 5.3.2 使用視圖

假設你有資料庫視圖：

```sql
CREATE VIEW vw_SalesSummary AS
SELECT c.Name AS Customer, SUM(o.Total) AS TotalSpent
FROM Orders o
JOIN Customers c ON c.Id = o.CustomerId
GROUP BY c.Name;
```

建立模型：

```csharp
[Keyless]
public class SalesSummary
{
    public string Customer { get; set; } = default!;
    public decimal TotalSpent { get; set; }
}
```

註冊：

```csharp
modelBuilder.Entity<SalesSummary>()
    .ToView("vw_SalesSummary")
    .HasNoKey();
```

查詢：

```csharp
var list = await ctx.Set<SalesSummary>().ToListAsync();
```

---

### 5.3.3 呼叫存放程序（Stored Procedure）

```sql
CREATE PROCEDURE GetOrdersByCustomer @customerId INT
AS
SELECT * FROM Orders WHERE CustomerId = @customerId;
```

C#：

```csharp
var id = 42;
var orders = await ctx.Orders
    .FromSqlRaw("EXEC GetOrdersByCustomer @customerId={0}", id)
    .ToListAsync();
```

---

## 5.4 IQueryable 擴充：打造可重用查詢介面

EF Core 的 `IQueryable` 查詢是延遲執行的。
你可以用**擴充方法**把常用查詢封裝起來。

---

### 5.4.1 範例：篩選與排序邏輯封裝

```csharp
public static class OrderQueryExtensions
{
    public static IQueryable<Order> Recent(this IQueryable<Order> q, int days)
        => q.Where(o => o.CreatedAt >= DateTime.UtcNow.AddDays(-days));

    public static IQueryable<Order> WithCustomer(this IQueryable<Order> q)
        => q.Include(o => o.Customer);

    public static IQueryable<Order> OrderByTotal(this IQueryable<Order> q)
        => q.OrderByDescending(o => o.Total);
}
```

使用起來像這樣：

```csharp
var query = ctx.Orders
    .Recent(30)
    .WithCustomer()
    .OrderByTotal();

var result = await query.AsNoTracking().ToListAsync();
```

> ✅ 保持 LINQ 可組合性
> ✅ 寫法乾淨且仍能被翻譯成 SQL

---

### 5.4.2 將查詢擴充與 Compiled Query 結合

```csharp
static readonly Func<AppDbContext, int, IAsyncEnumerable<OrderSummary>> _compiled =
    EF.CompileAsyncQuery((AppDbContext db, int days) =>
        db.Orders.Recent(days)
           .Select(o => new OrderSummary
           {
               Id = o.Id,
               Total = o.Total,
               Customer = o.Customer.Name
           }));

await foreach (var order in _compiled(ctx, 7))
{
    Console.WriteLine($"{order.Customer} spent {order.Total}");
}
```

---

## 5.5 常見查詢翻譯陷阱

| 問題                                             | 範例                                                                | 原因與解法                 |
|--------------------------------------------------|---------------------------------------------------------------------|----------------------------|
| 無法翻譯的 C# 方法                               | `.Where(o => Regex.IsMatch(o.Name, "abc"))`                         | Regex 不能轉 SQL；用 `Like` |
| 在查詢中用 `.ToList()`                           | `.Where(o => list.Contains(o.Id))`                                  | 子查詢應用 `.Contains()`   |
| `.Select(x => new { Sub = x.Related.ToList() })` | 子集合轉換應該留到 client；先取主體再分批載入                        |                            |
| `.Any()` 放錯層                                  | `.Where(o => o.Items.Any(i => i.Price > 100))` ✅，但放在 client 會錯 |                            |
| `.GroupBy` 轉不成 SQL                            | LINQ 的分組轉譯有限，請直接使用 `Select()` + 匯總函數                |                            |

---
1. `client` 指的是 C# 端。
2. 簡單說，不要在查詢的過程加入ToList這一類的取得資料的語法，否則條件過濾就會變成 C# 做，而不是 SQL Server 做。

---

### 5.5.1 例：GroupBy 轉譯

EF Core 支援直接翻譯成 SQL 的 GroupBy：

```csharp
var summary = await ctx.Orders
    .GroupBy(o => o.CustomerId)
    .Select(g => new {
        Customer = g.Key,
        Total = g.Sum(o => o.Total)
    })
    .ToListAsync();
```

生成 SQL：

```sql
SELECT [CustomerId], SUM([Total]) AS [Total]
FROM [Orders]
GROUP BY [CustomerId]
```

但**不能**在查詢中直接 `.ToList()` 或 `.SelectMany()`
否則會退回 client-side grouping。

---

## 5.6 高階技巧：原生 SQL + ORM 混搭

有時你想要手寫 SQL 以求極致效能，但仍希望保持 ORM 對象映射。

```csharp
var result = await ctx.Products
    .FromSqlRaw(@"
        SELECT Id, Name, Price
        FROM Products
        WHERE Price > 100
        ORDER BY Price DESC
    ")
    .AsNoTracking()
    .ToListAsync();
```

甚至可以在查詢中混用 `FromSqlRaw` + `LINQ`：

```csharp
var query = ctx.Products
    .FromSqlRaw("SELECT * FROM Products WHERE Price > 100")
    .Where(p => p.Name.Contains("糖"))
    .OrderBy(p => p.Name);
```

> EF Core 會將第二段 LINQ 翻譯包在外層 SQL（嵌套查詢）。

---

## 5.7 自定義查詢視圖模型（Projection Mapping）

若報表類查詢回傳非實體類型，可用匿名類型或 DTO。

```csharp
var dto = await ctx.Orders
    .Select(o => new OrderReportDto
    {
        OrderId = o.Id,
        Customer = o.Customer.Name,
        Count = o.Items.Count,
        Total = o.Items.Sum(i => i.Quantity * i.UnitPrice)
    })
    .ToListAsync();
```

生成 SQL：

```sql
SELECT [o].[Id], [c].[Name],
    (SELECT COUNT(*) FROM [OrderItems] WHERE [OrderId] = [o].[Id]) AS [Count],
    (SELECT SUM([Quantity]*[UnitPrice]) FROM [OrderItems] WHERE [OrderId]=[o].[Id]) AS [Total]
FROM [Orders] AS [o]
JOIN [Customers] AS [c] ON [o].[CustomerId] = [c].[Id];
```

> ✅ SQL 執行端計算完再傳回 → 減少記憶體負擔。

---

## 5.8 小結：查詢翻譯黃金守則

| 原則                      | 重點                                      |
|---------------------------|-------------------------------------------|
| **1. SQL 能懂的才快**     | 保證查詢能完全轉譯；避免呼叫 .NET 函數。    |
| **2. 客製函數要映射**     | 用 `[DbFunction]` 對應 SQL 函數。          |
| **3. 查詢能重用就預編譯** | 使用 `EF.CompileQuery()`。                 |
| **4. 複雜查詢拆分**       | 主查詢 → 子查詢 → 分批載入。               |
| **5. 原生 SQL 不可怕**    | `FromSqlRaw` + Keyless Entity 是整合利器。 |
| **6. 報表請投影**         | 永遠只取必要欄位。                         |
| **7. 驗證翻譯結果**       | 在開發時開啟 `.LogTo(Console.WriteLine)`。 |

---

## 5.9 Group 補充
`GroupBy` 在 C# 世界裡很好用，但在 EF Core 裡，
只有「部分用法」能翻譯成 SQL，
大多數人不知不覺就會寫出“退回記憶體分組”的程式（效能崩潰版）。

下面我們來看三組對照範例：
從「能翻譯」→「無法翻譯」→「正確改寫」。

---

# 🧩 觀念：GroupBy 有兩種語意

1. **SQL 型 GroupBy（翻譯為 `GROUP BY`）**

   * 必須「緊接著聚合（Sum / Count / Max / Min / Avg）」或 `Select` 匯總。
   * EF Core 可以轉譯。

2. **LINQ 型 GroupBy（分組後處理群組集合）**

   * 每個群組是一個 `IEnumerable<T>`。
   * EF Core 無法翻譯，會退回 client-side。

---

## ✅ 範例一：能翻譯成 SQL 的 GroupBy（正確用法）

```csharp
var summary = await ctx.Orders
    .GroupBy(o => o.CustomerId)
    .Select(g => new
    {
        CustomerId = g.Key,
        OrderCount = g.Count(),
        TotalAmount = g.Sum(o => o.Total)
    })
    .ToListAsync();
```

EF Core 會翻譯成漂亮的 SQL：

```sql
SELECT [o].[CustomerId],
       COUNT(*) AS [OrderCount],
       SUM([o].[Total]) AS [TotalAmount]
FROM [Orders] AS [o]
GROUP BY [o].[CustomerId];
```

> ✅ 完全在 SQL 端執行 → 超快
> ✅ 無需拉整張表進記憶體
> ✅ 可結合 HAVING, ORDER BY, LIMIT 等語法

---

## ⚠️ 範例二：無法翻譯的 GroupBy（錯誤用法）

```csharp
var grouped = await ctx.Orders
    .GroupBy(o => o.CustomerId) // ⚠️ 這裡只分組，沒聚合
    .ToListAsync();              // ⚠️ 問題來了
```

這樣做會出現以下警告或例外（取決於 EF 版本）：

> ⚠️ `The LINQ expression 'GroupBy' could not be translated and will be evaluated locally.`
> （無法翻譯 GroupBy，將在 client-side 執行。）

執行流程如下：

1. EF Core 把整張 Orders 表查回來：

   ```sql
   SELECT * FROM [Orders];
   ```
2. 然後在 C# 記憶體中執行分組 (`.GroupBy`)。

   * 如果有 10 萬筆訂單 → 全部載入記憶體。
   * 分組在應用層執行 → CPU 飆高。

---

### 📉 示意：記憶體爆炸流程圖

```
EF 查詢：SELECT * FROM Orders
    ↓
記憶體：產生 Dictionary<CustomerId, List<Order>>
    ↓
分組邏輯在 C# 端運行
    ↓
GC：🤢
```

---

## ⚠️ 範例三：更隱晦的錯誤 — 想直接選群組項目

```csharp
var result = await ctx.Orders
    .GroupBy(o => o.CustomerId)
    .Select(g => g.First()) // ⚠️ 想取每組第一筆
    .ToListAsync();
```

你可能以為這樣等價於：

```sql
SELECT TOP(1) ... FROM Orders GROUP BY CustomerId
```

但其實 EF Core 會報錯：

> ❌ `InvalidOperationException: The LINQ expression ... could not be translated...`

因為 SQL 的 `GROUP BY` 語法中，
沒有「取每組第一筆」這種語意。
EF Core 也不知道怎麼翻譯。

---

## ✅ 範例四：正確改寫「取每組第一筆」

解法一（子查詢）：

```csharp
var result = await ctx.Orders
    .GroupBy(o => o.CustomerId)
    .Select(g => g
        .OrderByDescending(o => o.CreatedAt)
        .FirstOrDefault()) // ✅ EF Core 6+ 可翻譯
    .ToListAsync();
```

EF Core 6 以後能翻譯成 SQL：

```sql
SELECT [o].[Id], [o].[CustomerId], [o].[CreatedAt], [o].[Total]
FROM [Orders] AS [o]
WHERE [o].[Id] IN (
    SELECT TOP(1) [o0].[Id]
    FROM [Orders] AS [o0]
    WHERE [o0].[CustomerId] = [o].[CustomerId]
    ORDER BY [o0].[CreatedAt] DESC
);
```

> ✅ 整體在 SQL 層執行，安全又快。

---

## ✅ 範例五：「分組後處理群組」改寫成兩階段查詢

有時候你真的想在 C# 處理每組內容（例如群組排序、分析等）。
那就要**顯式地分兩階段**進行：

```csharp
// 第 1 階段：取必要資料
var list = await ctx.Orders
    .Select(o => new { o.CustomerId, o.Total })
    .AsNoTracking()
    .ToListAsync(); // 這裡明確知道資料量可接受

// 第 2 階段：在記憶體分組
var groups = list
    .GroupBy(o => o.CustomerId)
    .Select(g => new
    {
        CustomerId = g.Key,
        MaxTotal = g.Max(x => x.Total)
    });
```

> ✅ 這樣的 client-side 分組是**有意為之**，不是意外產生。
> ✅ 你清楚資料量可接受，沒有隱藏成本。

---

## 🧠 一圖總結：GroupBy 能不能翻譯？

| 寫法                                                  | 是否可翻譯 | 翻譯結果            | 建議       |
|-------------------------------------------------------|------------|---------------------|------------|
| `.GroupBy(...).Select(g => new { g.Key, g.Count() })` | ✅          | SQL GROUP BY        | ✅ 優先使用 |
| `.GroupBy(...).Select(g => g.First())`                | ⚠️         | EF6+ 可（生成子查詢） | 可接受     |
| `.GroupBy(...).ToList()`                              | ❌          | Client evaluation   | ❌ 會拖慢   |
| `.GroupBy(...).Select(g => g)`                        | ❌          | Client evaluation   | ❌ 會拖慢   |
| `.GroupBy(...).SelectMany(...)`                       | ❌          | Client evaluation   | ❌ 慎用     |

---

## 💬 延伸說明：為何 EF 會這麼挑？

因為 SQL 的 `GROUP BY` 本質上是一個「聚合操作」，
它產出的不是群組集合，而是一張新的匯總結果表。
而 C# 的 `.GroupBy()` 回傳的是 `IEnumerable<IGrouping<TKey,TElement>>`，
這是一堆「分組後的 List」。

也就是說：

* SQL 世界的分組 → 表層級（聚合）。
* LINQ 世界的分組 → 物件層級（集合）。

EF Core 只能翻譯成 SQL 世界能理解的那一種。

---

## ✳️ 小結與建議

**可翻譯的 GroupBy 特徵：**

1. `GroupBy` 後**緊接著** `Select()` 聚合；
2. `Select` 裡只使用 `Key` 與聚合函數（`Sum`, `Count`, `Avg`, `Max`, `Min`）；
3. 不要在 `GroupBy` 之後立刻 `.ToList()`；
4. 不要在群組內再 `.ToList()` 或 `.Where()`；
5. 想要群組後邏輯，請分成兩個階段（SQL 聚合 + Client 分析）。

---
