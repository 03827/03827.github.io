# AsSplitQuery()


`AsSplitQuery()` 是 **Entity Framework Core 5**（EF Core 5）新增的一個查詢選項，用來控制 **Include 載入關聯資料時的 SQL 查詢策略**。
簡單講，它是為了解決「當你載入多層集合關聯時，EF 產生的 SQL 太巨大、資料重複太多、效能暴跌」這個老問題而誕生的。

---

## 🌱 它是什麼？

在 EF Core 裡，當你寫出像這樣的查詢：

```csharp
var orders = await ctx.Orders
    .Include(o => o.Items)
        .ThenInclude(i => i.Product)
    .Include(o => o.Customer)
    .ToListAsync();
```

EF 會生成一個很長的 SQL，把 `Orders`、`OrderItems`、`Products`、`Customers` 全部 JOIN 在一起。
這叫 **Single Query 模式**。

問題是——如果每張訂單有很多明細、每個明細又有商品，SQL 的結果會出現**重複行**（同一筆訂單資料重複 N 次），導致：

* 記憶體爆炸（重複資料太多）
* 傳輸量暴增
* SQL Server 執行計畫複雜、緩慢

於是 EF Core 團隊提供了新的選擇：

```csharp
.AsSplitQuery()
```

讓你把一個巨大的查詢「拆成多個小查詢」，每個查詢只載入一個層級的資料。

---

## ⚙️ 它如何工作？

當你使用 `AsSplitQuery()` 時，EF Core 會這樣做：

1. **第一個查詢**：抓主體（例如 Orders）

   ```sql
   SELECT * FROM Orders WHERE CreatedAt > '2025-01-01'
   ```

2. **第二個查詢**：抓第一層集合（OrderItems）

   ```sql
   SELECT * FROM OrderItems WHERE OrderId IN (1,2,3,4,5)
   ```

3. **第三個查詢**：抓子集合關聯（Products）

   ```sql
   SELECT * FROM Products WHERE Id IN (10,11,12,...)
   ```

EF Core 在記憶體中會**根據外鍵關聯鍵值自動把它們「合併」成一個完整的物件樹**，就像你用 `Include` 一樣可以拿到完整的結果，只是 SQL 被拆成多段。

---

## 💡 舉例比較

**沒有 AsSplitQuery（Single Query）**

```sql
SELECT o.Id, o.CustomerId, i.Id, i.OrderId, p.Id, p.Name
FROM Orders AS o
LEFT JOIN OrderItems AS i ON o.Id = i.OrderId
LEFT JOIN Products AS p ON i.ProductId = p.Id
```

如果一張訂單有 10 個明細、每個明細連到一個商品 →
同一筆訂單會重複出現在 10 行結果裡。

---

**有 AsSplitQuery（Split Query）**

```sql
SELECT * FROM Orders;
SELECT * FROM OrderItems WHERE OrderId IN (...);
SELECT * FROM Products WHERE Id IN (...);
```

結果由 EF 自動組裝，不會有重複行。

---

## ⚖️ 優缺點分析

| 模式                            | 優點             | 缺點                           | 適用場景           |
| ----------------------------- | -------------- | ---------------------------- | -------------- |
| **Single Query（預設）**          | 一次往返資料庫，減少網路延遲 | JOIN 太多造成笛卡兒乘積、記憶體暴增         | 一對一、多對一、少量資料   |
| **Split Query（AsSplitQuery）** | 避免笛卡兒乘積、查詢更穩定  | 需要多次查詢（多 round-trip），DB 往返略多 | 一對多、多層集合、資料量大時 |

---

## 🧠 延伸：全域設定

如果你希望專案預設都使用分查詢，可以在 `OnConfiguring` 或 `AddDbContext` 時設定：

```csharp
options.UseSqlServer(connString,
    o => o.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery));
```

也可反過來強制單查詢：

```csharp
.AsSingleQuery()
```

---

## ✅ 建議實務使用法

* 多層集合（像 `Order -> Items -> Product`）→ 用 `AsSplitQuery()`。
* 只有單一關聯（像 `Include(o => o.Customer)`）→ 用預設單查詢就好。
* 如果你要做報表、清單頁 → Split Query 幾乎都比較穩定。
* 若每個集合的資料量都很小（10 筆以下） → Single Query 仍最快。

---
