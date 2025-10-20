
---

# Chapter 2｜交易與一致性（Transaction & Consistency）

---

## 2.1 為什麼要有交易？

所謂「交易」（Transaction），就是一組**要嘛全部成功，要嘛全部回滾**的資料庫操作。
ACID 四大原則：

| 原則                        | 說明                                                 |
|-----------------------------|------------------------------------------------------|
| **A – Atomicity（原子性）**   | 一組操作不可分割。任一子操作失敗 → 全部撤銷。          |
| **C – Consistency（一致性）** | 執行前後，資料必須符合所有約束（外鍵、唯一性、商業規則）。 |
| **I – Isolation（隔離性）**   | 不同交易之間的操作不能互相干擾（有不同的隔離等級）。    |
| **D – Durability（持久性）**  | 一旦提交（Commit），變更永久寫入。                       |

EF Core 幫我們在大多數情況下自動維持這四點，但當你開始混合：

* 多次 `SaveChangesAsync()`，
* 多個 `DbContext`，
* 或跨資料庫／外部服務（例如 MQ、API）時，
  就必須自己處理「交易範圍」與「一致性策略」。

---

## 2.2 EF Core 中交易的三層結構

EF Core 交易機制分為三層：

1. **隱含交易（Implicit Transaction）**
   每次 `SaveChanges()` 都會自動開一個交易包住所有變更。
   👉 適合「一次 SaveChanges 就夠」的單步操作。

2. **顯式交易（Explicit Transaction）**
   由你手動開啟交易（`BeginTransaction()`），可包住多個 `SaveChanges()` 或原生 SQL。
   👉 適合「多步驟需要原子性」的情境。

3. **外部交易（External Transaction）**
   多個 `DbContext` 或外部系統共用同一個交易，例如 ADO.NET Transaction 或 TransactionScope。
   👉 適合跨多資料來源或舊系統整合。

---

## 2.3 範例 1：隱含交易（自動）

```csharp
using var ctx = new AppDbContext(options);

// 這裡 SaveChanges() 自動開啟並提交一個交易
ctx.Orders.Add(new Order { CustomerId = 1, CreatedAt = DateTime.UtcNow });
await ctx.SaveChangesAsync(); // 若失敗自動回滾
```

在這情況下，EF Core 幫你包好「BEGIN TRANSACTION → COMMIT / ROLLBACK」整個流程。

---

## 2.4 範例 2：顯式交易（手動控制）

當你需要：

* 同時修改多張表；
* 混合 `SaveChanges` 與 `ExecuteSqlRaw`；
* 或要延遲提交直到全部步驟完成。

就應該手動開交易。

```csharp
await using var ctx = new AppDbContext(options);
await using var tx = await ctx.Database.BeginTransactionAsync();

try
{
    var order = new Order { CustomerId = 1, CreatedAt = DateTime.UtcNow };
    ctx.Orders.Add(order);
    await ctx.SaveChangesAsync();

    await ctx.Database.ExecuteSqlRawAsync(
        "UPDATE Products SET Stock = Stock - {0} WHERE Id = {1}", 5, 1001);

    await tx.CommitAsync();
}
catch (Exception)
{
    await tx.RollbackAsync();
    throw;
}
```

---

## 2.5 範例 3：跨多個 DbContext 共用交易

有時候你需要用兩個 `DbContext` 處理不同模型或資料庫，但仍希望在同一個交易中。

### 2.5.1 - 同一資料庫的多個 Context：

```csharp
await using var ctx1 = new SalesContext(options);
await using var tx = await ctx1.Database.BeginTransactionAsync();

await using var ctx2 = new InventoryContext(options);
await ctx2.Database.UseTransaction(tx.GetDbTransaction()); // 共用同一交易

ctx1.Orders.Add(new Order { CustomerId = 1 });
ctx2.Products.First().Stock -= 5;

await ctx1.SaveChangesAsync();
await ctx2.SaveChangesAsync();

await tx.CommitAsync();
```

---

### 2.5.2 - Transaction 用法注意

```csharp
//🚫 超級不優秀 - 要加上 using 以及使用回傳的 IDbContextTransaction 物件操作後續
//1. 未使用 using，若是忘了 Commit/Rollback 有可能會佔用DB連線，讓其他人的連線卡住
//2. 在不知情的情況下，若使用同一個DBContext想要先後啟用多個BeginTransactionAsync，會發Exception
//3. 較難保證在錯誤時一定呼叫到Rollback與釋放。
await _context.Database.BeginTransactionAsync();
await _context.Database.CommitTransactionAsync();

```

```csharp
//❌ 不優秀 - 未使用 using，若是忘了 Commit/Rollback 有可能會佔用DB連線，讓其他人的連線卡住
var tx = await _context.Database.BeginTransactionAsync();
await tx.CommitAsync();
```

```csharp
//⚠️ 勉強可以 - 未使用 using，若是忘了 Commit/Rollback，因為使用了 using， 當離開using範圍時會自動呼叫 Rollback 及 DisposeAsync，雖然資料沒有真正寫入，但至少不會佔用連線
using var tx = await _context.Database.BeginTransactionAsync();
await tx.CommitAsync();
```

```csharp
//✅ 優秀作法 - await using 會使用非同步方式釋放資源，由系統自行尋找適當時機釋放
await using var tx = await _context.Database.BeginTransactionAsync();
await tx.CommitAsync();
```

---

## 2.6 隔離等級（Isolation Level）

不同隔離等級會影響鎖的持有範圍與一致性。
常見五種：

| 隔離等級                           | 保證程度         | 可能發生的問題       | 備註                |
|------------------------------------|------------------|----------------------|---------------------|
| Read Uncommitted                   | 最低             | 髒讀（讀到未提交資料） | 幾乎不用            |
| Read Committed                     | 預設             | 無髒讀；可能非重複讀  | EF Core 預設        |
| Repeatable Read                    | 保證同筆資料一致 | 幻讀可能發生         |                     |
| Serializable                       | 最嚴格；完全一致  | 可能鎖競爭劇烈       |                     |
| Snapshot / Read Committed Snapshot | 使用版本而非鎖   | 不阻塞讀寫           | SQL Server 推薦開啟 |

可在交易啟動時指定：

```csharp
await using var tx = await ctx.Database.BeginTransactionAsync(IsolationLevel.Serializable);
```

---

在資料庫交易（Transaction）中，**隔離等級（Isolation Level）** 是控制「多個交易同時執行時彼此之間能看到多少彼此尚未提交的變更」的機制。這是資料庫一致性的重要支柱之一，用來在「效能」與「正確性」之間取得平衡。

而在 **Entity Framework Core（EF Core）** 中，隔離等級主要是透過 `TransactionScope` 或 `DbContext.Database.BeginTransaction(IsolationLevel)` 來設定。EF Core 本身不創造新的隔離等級，而是直接使用底層資料庫（例如 SQL Server、PostgreSQL、MySQL）的原生隔離行為。

---

### 2.6.1 常見的隔離等級

以下是 SQL 標準定義的主要隔離等級，EF Core 都支援設定：

1. **Read Uncommitted（未提交讀取）**
   允許交易讀取其他交易尚未提交的資料。

   * 可能發生：「**髒讀（Dirty Read）**」
   * 優點：速度快。
   * 缺點：資料可能錯誤或不一致。
   * 幾乎不建議在生產環境使用。

2. **Read Committed（已提交讀取）**
   只允許讀取其他交易已提交的資料。

   * 防止「髒讀」，但仍可能發生「**不可重複讀（Non-repeatable Read）**」。
   * 多數資料庫的**預設隔離等級**（如 SQL Server）。

3. **Repeatable Read（可重複讀取）**
   一旦在同一交易中讀取過一筆資料，再次讀取時結果保證一致。

   * 防止「髒讀」與「不可重複讀」，但仍可能發生「**幻影讀（Phantom Read）**」。
   * 適合需要穩定查詢結果的情境，如報表產生。

4. **Serializable（可序列化）**
   最嚴格的隔離級別，模擬所有交易依序執行。

   * 防止髒讀、不可重複讀與幻影讀。
   * 代價：效能大幅下降，容易導致鎖競爭。

5. **Snapshot（快照隔離）**
   使用版本控制的方式讀取資料，避免鎖定。

   * 可防止髒讀、不可重複讀與幻影讀。
   * 在 SQL Server 必須明確啟用 `ALLOW_SNAPSHOT_ISOLATION`。
   * 適合高併發系統，整合「一致性」與「效能」。

---

### 在 EF Core 中使用

假設你要手動指定隔離等級，可以這樣寫：

```csharp
using var transaction = await context.Database.BeginTransactionAsync(IsolationLevel.Serializable);
try
{
    // 進行資料操作
    await context.SaveChangesAsync();

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

這樣 EF Core 就會以 `Serializable` 隔離等級開啟資料庫交易。

---

### 實務建議

* **預設使用 Read Committed**：在大部分情境下已足夠。
* **分析問題再提升隔離等級**：不要一開始就用 Serializable，會犧牲效能。
* **考慮 Snapshot**：在高併發讀取的系統中，是現代化又高效的折衷方案。

---

這三個名詞是理解資料庫「交易隔離性」的核心。它們描述的是在多個交易同時運作時，讀寫資料可能出現的三種異常現象。讓我們用一個輕鬆但準確的例子來拆解它們。

---

### 1) 髒讀（Dirty Read）

**定義**：一個交易讀取了「另一個交易尚未提交」的資料。

**想像場景**：
交易 A 更新了某筆資料，但還沒提交（commit）。
交易 B 立刻去讀這筆資料，看到了更新後的值。
此時如果交易 A 發生錯誤、回滾（rollback），那交易 B 實際上讀到的是**不存在的資料變更**，就像讀到一份還沒寫完的草稿。

**比喻**：
有人在白板上寫字，你偷看了一眼，結果他擦掉重寫了。你腦中記得的內容根本沒被保留下來。

**會發生在**：`Read Uncommitted`

---

### 2) 不可重複讀（Non-repeatable Read）

**定義**：同一個交易中，兩次讀取同一筆資料，結果卻不一樣。

**想像場景**：
交易 A 讀取某筆資料（值為 100）。
交易 B 在 A 的交易還沒結束前，把這筆資料改成 200 並提交。
交易 A 再次讀取時，資料變成了 200。
A 會驚訝：「我剛才明明看到 100！」

**比喻**：
你在寫一份報告，第一次看統計表是 100，過一會兒再看成了 200。數字沒錯，但你無法確定什麼才是真實的「同一時刻」的狀態。

**會發生在**：`Read Committed`（可防髒讀，但不防不可重複讀）

---

### 3) 幻影讀（Phantom Read）

**定義**：同一個交易中，兩次執行「查詢條件相同的集合查詢」，但第二次卻出現了**新的或消失的資料列**。

**想像場景**：
交易 A 查詢「薪水大於 50000 的員工」——得到 10 筆。
交易 B 插入一個新員工，薪水是 80000，並提交。
交易 A 再次執行相同查詢，這次變成 11 筆。
新的那筆資料就像幻影一樣出現。

**比喻**：
你在統計名單時，一邊點名、一邊有人偷偷進來坐下。下一次清點，發現人數變多了。

**會發生在**：`Repeatable Read`（在這層級可防不可重複讀，但仍可能出現幻影讀）

---

### 隔離等級與異常對照表

| 隔離等級         | 髒讀 | 不可重複讀 | 幻影讀 |
|------------------|------|------------|--------|
| Read Uncommitted | ✅    | ✅          | ✅      |
| Read Committed   | ❌    | ✅          | ✅      |
| Repeatable Read  | ❌    | ❌          | ✅      |
| Serializable     | ❌    | ❌          | ❌      |

✅ 表示可能發生；❌ 表示被防止。

---

### 2.7 隔離異常實例：

假設有兩張表：

Accounts:
| Id | Balance |
|----|---------|
| 1  | 100     |

---

Orders:
| Id | Amount |
|----|--------|
| 1  | 120    |
| 2  | 80     |
| 3  | 200    |

---

## 用 EF Core 重現（C#）

> 下面用兩個併發任務模擬兩個工作階段。請注意：為了示範交錯，我們使用 `Task.Delay` 來控制時序。實務上請用你熟悉的 DI 建置 `DbContext`。

### 1) 髒讀（Dirty Read）

```csharp
var a = Task.Run(async () =>
{
    await using var ctx = new DemoContext();
    await using var tx = await ctx.Database.BeginTransactionAsync(IsolationLevel.ReadCommitted);

    var acct = await ctx.Accounts.FindAsync(1);
    acct!.Balance += 100; // -> 200（尚未提交）
    await ctx.SaveChangesAsync();

    await Task.Delay(2000); // 暫停，讓 B 讀到未提交資料（髒讀需要 B 用 ReadUncommitted）
    await tx.RollbackAsync(); // 回滾，回到 100
});

var b = Task.Run(async () =>
{
    await Task.Delay(500); // 確保 A 已經更新但未提交
    await using var ctx = new DemoContext();
    await using var tx = await ctx.Database.BeginTransactionAsync(IsolationLevel.ReadUncommitted);

    var val = await ctx.Accounts.Where(x => x.Id == 1).Select(x => x.Balance).FirstAsync();
    Console.WriteLine($"B sees (dirty): {val}"); // 多半會看到 200
    await tx.CommitAsync();
});

await Task.WhenAll(a, b);
```

### 2) 不可重複讀（Non-repeatable Read）

```csharp
var a = Task.Run(async () =>
{
    await using var ctx = new DemoContext();
    await using var tx = await ctx.Database.BeginTransactionAsync(IsolationLevel.ReadCommitted);

    var v1 = await ctx.Accounts.Where(x => x.Id == 1).Select(x => x.Balance).FirstAsync();
    Console.WriteLine($"A first read: {v1}");

    await Task.Delay(1500); // 等 B 提交更新

    var v2 = await ctx.Accounts.Where(x => x.Id == 1).Select(x => x.Balance).FirstAsync();
    Console.WriteLine($"A second read: {v2}"); // 與 v1 不同 => 不可重複讀

    await tx.CommitAsync();
});

var b = Task.Run(async () =>
{
    await Task.Delay(500);
    await using var ctx = new DemoContext();
    await using var tx = await ctx.Database.BeginTransactionAsync(IsolationLevel.ReadCommitted);

    var acct = await ctx.Accounts.FindAsync(1);
    acct!.Balance += 100;
    await ctx.SaveChangesAsync();
    await tx.CommitAsync();
});

await Task.WhenAll(a, b);
```

### 3) 幻影讀（Phantom Read）

```csharp
var a = Task.Run(async () =>
{
    await using var ctx = new DemoContext();
    await using var tx = await ctx.Database.BeginTransactionAsync(IsolationLevel.RepeatableRead);

    var c1 = await ctx.Orders.Where(x => x.Amount >= 100).CountAsync();
    Console.WriteLine($"A first count: {c1}");

    await Task.Delay(1500); // 等 B 插入一筆符合條件的資料

    var c2 = await ctx.Orders.Where(x => x.Amount >= 100).CountAsync();
    Console.WriteLine($"A second count: {c2}"); // c2 可能 > c1 => 幻影讀

    await tx.CommitAsync();
});

var b = Task.Run(async () =>
{
    await Task.Delay(500);
    await using var ctx = new DemoContext();
    await using var tx = await ctx.Database.BeginTransactionAsync(IsolationLevel.ReadCommitted);

    /*
    這個行為會導致 SaveChanges 持續等待直至Task A完成了才會執行
    var acctB = await ctx
        .Accounts.FromSqlInterpolated($@"select * from Accounts where [Id] = {2}")
        .FirstOrDefaultAsync();
    
    acctB.Balance += 50;
    await ctx.SaveChangesAsync();
    */

    //這個行為無須等待Task A，可直接執行 SaveChanges
    ctx.Orders.Add(new Order { Amount = 150 });

    await ctx.SaveChangesAsync();
    await tx.CommitAsync();
});

await Task.WhenAll(a, b);
```

> 若把 A 的 `RepeatableRead` 改成 `Serializable`，B 的插入會被阻擋直到 A 結束，`c2` 就會等於 `c1`，不會看到幻影。

---

## 小抄：各隔離等級能否觀察到異常

* `READ UNCOMMITTED`：可能髒讀、不可重複讀、幻影讀
* `READ COMMITTED`：阻擋髒讀，但仍可能不可重複讀、幻影讀
* `REPEATABLE READ`：阻擋髒讀、不可重複讀，但仍可能幻影讀
* `SERIALIZABLE`：三者都阻擋（代價是鎖競爭加劇）

---

## 2.8 防止交易錯誤與異常處理模式

### 2.8.1 鎖競爭／死結重試（結合第 3 章）

加上重試策略：

```csharp
var policy = Policy
    .Handle<SqlException>(ex => ex.Number == 1205)
    .WaitAndRetryAsync(3, i => TimeSpan.FromMilliseconds(100 * i));

await policy.ExecuteAsync(async () =>
{
    await using var tx = await ctx.Database.BeginTransactionAsync();
    // ...
    await tx.CommitAsync();
});
```

### 2.8.2 SaveChanges 異常自動回滾

EF Core 的 `BeginTransaction()` 會在 `Commit()` 前自動保留 rollback 能力。
如果發生任何例外，`Dispose()` 時會回滾。

```csharp
await using var tx = await ctx.Database.BeginTransactionAsync();
try
{
    // ...
    await ctx.SaveChangesAsync();
    await tx.CommitAsync();
}
finally
{
    // 若未 commit，transaction.Dispose() 時自動 rollback
}
```

---

## 2.9 多層系統的交易設計模式

| 系統層級               | 建議做法                                                   |
|------------------------|------------------------------------------------------------|
| Web API 層             | 不要手動開交易；呼叫 Application Service 時由 Service 負責。 |
| Application Service 層 | 每個 Use Case 自成一個短交易。                              |
| 背景批次/工作          | 明確 BeginTransaction；搭配重試與批次大小控制。              |
| 跨界事件               | Outbox Pattern 或 Saga Pattern。                            |

---

## 2.10 小結

**EF Core 的交易設計心法：**

* ✅ 一次 `SaveChanges()` → EF 自動包交易。
* ✅ 多步操作 → 用 `BeginTransaction()` 包住。
* ✅ 跨多 Context → `UseTransaction()` 共用。
* ✅ 跨系統一致性 → Outbox Pattern。
* ✅ 避免長交易 → 縮短鎖持有時間。
* ✅ 發生死結/錯誤 → 自動回滾 + 重試。

---