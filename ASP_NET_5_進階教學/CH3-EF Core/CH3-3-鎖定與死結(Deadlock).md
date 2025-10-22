# Chapter 3｜鎖定與死結（Locking & Deadlock）

## 3.1 鎖競爭 (Lock Contention)

**競爭** = 兩個交易同時要求同一資源（例如同一列），但鎖相容性衝突。
例如：

* 交易 A：更新某筆資料 → 取得 **X 鎖**
* 交易 B：想要讀取該資料 → 嘗試取得 **S 鎖** → 等待 A 釋放

1. `以上這種等待是正常的`
2. `但如果等待鏈條變成**循環**，就會形成「死結 (Deadlock)」。`

---

## 3.2 什麼是死結（Deadlock）

**定義：**
兩個或以上交易互相持有對方需要的鎖，彼此等待 → 永遠無法前進。
資料庫偵測到這種循環，就會中止其中一個交易（犧牲者），丟出錯誤：

```
SqlException (Number=1205): Transaction (Process ID X) was deadlocked on ...
```

---

## 3.3 範例：重現一個死結

假設我們有兩張表：

```csharp
public class Order
{
    public int Id { get; set; }
    public int ProductId { get; set; }
    public int Status { get; set; }
}

public class Product
{
    public int Id { get; set; }
    public int Stock { get; set; }
}
```

**交易 A**：先更新 `Order`，再更新 `Product`
**交易 B**：先更新 `Product`，再更新 `Order`

這樣就可能互鎖。

```csharp
private static async Task TxA(DbContextOptions<AppDbContext> opt)
{
    await using var ctx = new AppDbContext(opt);
    await using var tx = await ctx.Database.BeginTransactionAsync();

    var order = await ctx.Orders.FirstAsync(o => o.Id == 1);
    order.Status = 1;
    await ctx.SaveChangesAsync();

    await Task.Delay(2000); // 模擬業務邏輯停頓

    var product = await ctx.Products.FirstAsync(p => p.Id == order.ProductId);
    product.Stock -= 1;
    await ctx.SaveChangesAsync();

    await tx.CommitAsync();
}

private static async Task TxB(DbContextOptions<AppDbContext> opt)
{
    await using var ctx = new AppDbContext(opt);
    await using var tx = await ctx.Database.BeginTransactionAsync();

    var product = await ctx.Products.FirstAsync(p => p.Id == 1);
    product.Stock -= 1;
    await ctx.SaveChangesAsync();

    await Task.Delay(2000);

    var order = await ctx.Orders.FirstAsync(o => o.ProductId == product.Id);
    order.Status = 2;
    await ctx.SaveChangesAsync();

    await tx.CommitAsync();
}
```

如果同時執行 `TxA` 與 `TxB`，幾乎一定會造成死結。

---

## 3.6 鎖順序圖解（文字版）

```
TxA 與 TxB 同時執行 →
TxA: |--- 鎖 Order(id = 1) -----等待 Product(id = 1)-----|
TxB: |--- 鎖 Product(id = 1) ---等待 Order(id = 1)-------|
結果：雙方互等 → Deadlock
```

---

## 3.7 如何診斷死結

### (1) SQL Server 工具

* 使用 **SQL Profiler** 或 **Extended Events**，啟用「deadlock graph」事件。
* SQL 語法 查詢：

  ```sql
  SELECT * FROM sys.dm_tran_locks;
  SELECT * FROM sys.dm_os_waiting_tasks;
  ```

### (2) EF Core 偵測

打開 EF Core log：

```csharp
services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(connString)
       .EnableSensitiveDataLogging()
       .LogTo(Console.WriteLine, LogLevel.Information));
```

當你看到大量相似 SQL，卻突然出現 `SqlException 1205` → 就是死結。

---

## 3.8 改善策略與程式範例

### (A) 固定存取順序（最重要原則）

所有交易若更新多張表，必須遵守固定順序（例如依 Id 升冪、或固定表順序）。
只要順序一致，死結就幾乎不會發生。

```csharp
private static async Task TxB_Fixed(DbContextOptions<AppDbContext> opt)
{
    await using var ctx = new AppDbContext(opt);
    await using var tx = await ctx.Database.BeginTransactionAsync();

    // 改為與 TxA 一致的順序：先更新 Order 再更新 Product
    var order = await ctx.Orders.FirstAsync(o => o.Id == 1);
    order.Status = 2;
    await ctx.SaveChangesAsync();

    await Task.Delay(200);

    var product = await ctx.Products.FirstAsync(p => p.Id == order.ProductId);
    product.Stock -= 1;
    await ctx.SaveChangesAsync();

    await tx.CommitAsync();
}
```

---

### (B) 縮短交易時間（短即是美德）

不要把交易跨太多操作或 UI。
正確做法：

* 查詢資料時不開交易。
* 寫入時才開交易，儘快提交。

```csharp
// 查詢階段 (無交易)
var dto = await ctx.Orders.AsNoTracking().SingleAsync(o => o.Id == id);

// 更新階段 (短交易)
await using var tx = await ctx.Database.BeginTransactionAsync();
var entity = await ctx.Orders.FirstAsync(o => o.Id == dto.Id);
entity.Status = 2;
await ctx.SaveChangesAsync();
await tx.CommitAsync();
```

```csharp
// ❌ 不應該：不要把外部操作放在交易裡
await tran.Begin();
DoSomethingHttp(); // 耗時操作
SaveChanges();
await tran.Commit();

// ✅ 正確做法
DoSomethingHttp(); // 交易外

await tran.Begin();
SaveChanges();
await tran.Commit();

```
---

### (C) 增加索引以避免大範圍鎖

缺索引會讓 SQL Server 掃描整張表（Table Scan），導致表鎖。
建立合適索引是預防死結最根本的方式之一。

```sql
CREATE INDEX IX_Orders_ProductId ON Orders(ProductId);
CREATE INDEX IX_Products_Id ON Products(Id);
```

---

### (D) 使用適當的隔離等級與版本控制

#### 1. 調整隔離等級

```csharp
await using var tx = await ctx.Database.BeginTransactionAsync(IsolationLevel.ReadCommitted);
```

* `ReadCommitted` → 預設，穩定且常用。
* `Serializable` → 鎖得更嚴，適合報表但高風險死結。
* `ReadCommittedSnapshot`（DB 層設定）→ 用版本讀取，降低鎖競爭。

```sql
ALTER DATABASE YourDb SET READ_COMMITTED_SNAPSHOT ON;
```

#### 2. 使用 EF 的樂觀鎖（RowVersion）

在 `Order` 中加上：

```csharp
[Timestamp]
public byte[] RowVersion { get; set; } = default!;
```

更新時自動檢查資料是否被他人改過，避免長時間鎖住列。

---

### (E) 悲觀鎖（Pessimistic Lock）
1. 當有多人要 `Select Orders` 時，**第一個**取得鎖的人可以順利的進行讀取，其他人只能等待取得鎖的人執行 `Commit` 釋放鎖，再由下一個人取得鎖。可用 SQL Hint 強制鎖策略，例如 `UPDLOCK` 、 `XLOCK` 、 `ROWLOCK`。

**用法：**

```csharp
await using var tx = await ctx.Database.BeginTransactionAsync();

var pending = await ctx.Orders
    .FromSqlRaw(@"SELECT TOP(1) * FROM Orders WITH (UPDLOCK, ROWLOCK)
                  WHERE Id = 1")
    .SingleAsync();

    pending.Status = 1;

    await Task.Delay(2000);

await ctx.SaveChangesAsync();

await tx.CommitAsync();
```

---

2. 可以自訂最大等待時間

**用法：**

```csharp
//本次連線，最多等待解鎖30秒
await ctx.Database.ExecuteSqlRawAsync("SET LOCK_TIMEOUT 30000");
try
{
    var acct = await ctx.Accounts
        .FromSqlInterpolated($@"
            SELECT * FROM Accounts
            WITH (UPDLOCK, ROWLOCK)
            WHERE [Id] = {2}")
        .FirstOrDefaultAsync();
}
catch (Microsoft.Data.SqlClient.SqlException ex) when (ex.Number == 1222)
{
    // 1222代表等待解鎖逾時
    throw new TimeoutException("查詢超過 30 秒仍被鎖定，已中止。", ex);
}
finally
{
    await ctx.Database.ExecuteSqlRawAsync("SET LOCK_TIMEOUT -1");
}
```

---

### (F) Retry（死結重試策略）

就算防範完善，仍有可能偶發死結。
要讓程式能自動重試：

```csharp
var policy = Policy
    .Handle<SqlException>(ex => ex.Number == 1205)
    .WaitAndRetryAsync(3, i => TimeSpan.FromMilliseconds(100 * i));

await policy.ExecuteAsync(async () =>
{
    await using var tx = await ctx.Database.BeginTransactionAsync();
    // 寫入邏輯
    await ctx.SaveChangesAsync();
    await tx.CommitAsync();
});
```

---

### (G) 分批處理（Batch Update）

大範圍更新容易產生死結。
解法：每次處理少量資料。

```csharp
const int batchSize = 100;
int skip = 0;
List<Order> batch;

do
{
    batch = await ctx.Orders
        .Where(o => o.Status == 0)
        .OrderBy(o => o.Id)
        .Skip(skip)
        .Take(batchSize)
        .ToListAsync();

    foreach (var o in batch)
        o.Status = 1;

    await ctx.SaveChangesAsync();
    skip += batchSize;
} while (batch.Count > 0);
```

---

## 3.9 死結調查工具：實際操作

假如你要在 SQL Server 中分析死結，可以用：

```sql
-- 查看目前鎖定情況
SELECT request_session_id AS SessionID,
       resource_type AS Resource,
       resource_description AS ResourceDesc,
       request_mode AS LockMode,
       request_status AS Status
FROM sys.dm_tran_locks;

-- 查看等待關係
SELECT session_id, blocking_session_id, wait_type, wait_time, wait_resource
FROM sys.dm_os_waiting_tasks
WHERE blocking_session_id IS NOT NULL;
```

或用「SQL Server Management Studio → Activity Monitor → Processes → 查看 Blocking」。

---

## 3.10 整合策略實例：完整修正版程式

綜合以上原則，以下是安全又穩定的交易模板：

```csharp
public async Task<bool> SafeUpdateAsync(AppDbContext ctx, int orderId, int productId)
{
    var retry = Policy
        .Handle<SqlException>(ex => ex.Number == 1205)
        .WaitAndRetryAsync(3, i => TimeSpan.FromMilliseconds(100 * i));

    return await retry.ExecuteAsync(async () =>
    {
        await using var tx = await ctx.Database.BeginTransactionAsync(IsolationLevel.ReadCommitted);

        // 固定順序：Order → Product
        var order = await ctx.Orders.FirstAsync(o => o.Id == orderId);
        var product = await ctx.Products.FirstAsync(p => p.Id == productId);

        order.Status = 2;
        product.Stock -= 1;

        await ctx.SaveChangesAsync();
        await tx.CommitAsync();

        return true;
    });
}
```

此版本同時具備：

* 明確交易範圍；
* 一致存取順序；
* ReadCommitted 隔離；
* 自動死結重試；
* 短暫持鎖範圍。

---

## 3.11 核心原則總結

| 主題         | 核心原則                                              |
|--------------|-------------------------------------------------------|
| **鎖**       | 交易在操作資料時會自動鎖定行、頁或表，以保護一致性。     |
| **死結**     | 兩個交易互相持有對方需要的鎖，造成循環等待。            |
| **避免方法** | 一致存取順序、短交易、正確索引、READ_COMMITTED_SNAPSHOT。 |
| **修復策略** | Retry 機制、自動回滾、悲觀鎖或樂觀併發。                 |
| **診斷工具** | EF Core Log、SQL Profiler、DMV (`sys.dm_tran_locks`)。   |

---