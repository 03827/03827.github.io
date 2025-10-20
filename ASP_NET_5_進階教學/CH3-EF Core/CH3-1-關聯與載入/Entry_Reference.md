# Entry(someEntity).Reference(someField)

程式碼：

```csharp
await ctx.Entry(item.Product).Reference("ExtraSpec").LoadAsync();
```

看起來有點神祕，其實它是 **Entity Framework Core 的「顯式載入 (Explicit Loading)」API**，用來手動載入導覽屬性。
這裡的 `.Reference("ExtraSpec")` 就是「指定要載入哪一個**參照型導覽屬性（Reference Navigation Property）**」。

---

## 一、`Reference()` 是什麼意思？

在 EF Core 裡，每個實體的屬性可以分為兩類導覽：

| 類型                      | 用來表示                                  | 方法               |
|---------------------------|-------------------------------------------|--------------------|
| **Reference Navigation**  | 1對1 或 多對1 關聯（例如：Order → Customer） | `.Reference(...)`  |
| **Collection Navigation** | 1對多 關聯（例如：Order → Items）            | `.Collection(...)` |

所以：

* `.Reference(x => x.Customer)` → 載入單一參照
* `.Collection(x => x.Items)` → 載入集合

這裡你看到的 `.Reference("ExtraSpec")` 是「用屬性名稱（字串）」來指定要載入的參照屬性。

---

## 二、那 `ExtraSpec` 是什麼？

`"ExtraSpec"` 指的是某個 **導覽屬性的名稱**。
假設你的 `Product` 實體裡有一個額外的關聯（例如產品的詳細規格），像這樣：

```csharp
public class Product {
    public int Id { get; set; }
    public string Name { get; set; } = default!;
    public decimal Price { get; set; }

    // 一對一關聯到 ProductExtraSpec
    public ProductExtraSpec ExtraSpec { get; set; } = default!;
}

public class ProductExtraSpec {
    public int Id { get; set; }
    public int ProductId { get; set; }
    public string Manufacturer { get; set; } = default!;
    public string Warranty { get; set; } = default!;
    public Product Product { get; set; } = default!;
}
```

那這段：

```csharp
await ctx.Entry(item.Product).Reference("ExtraSpec").LoadAsync();
```

意思是：

> 「我已經有一個 `Product` 物件（它目前可能還沒載入 ExtraSpec），
> 現在我手動命令 EF 去載入這個 Product 的 `ExtraSpec` 參照資料。」

執行後，EF Core 會自動幫你執行 SQL，例如：

```sql
SELECT [p].[Id], [p].[ProductId], [p].[Manufacturer], [p].[Warranty]
FROM [ProductExtraSpec] AS [p]
WHERE [p].[ProductId] = @productId
```

載入完成後，`item.Product.ExtraSpec` 屬性就有資料了（可以直接用）。

---

## 三、那為什麼是字串 `"ExtraSpec"`？可以用 Lambda 嗎？

可以。EF Core 提供兩種寫法：

### ✅ 推薦：Lambda 寫法（型別安全）

```csharp
await ctx.Entry(item.Product).Reference(p => p.ExtraSpec).LoadAsync();
```

這樣能讓編譯器檢查屬性名稱，若未來改名也會自動更新。

### ⚠️ 不建議：字串寫法（動態）

```csharp
await ctx.Entry(item.Product).Reference("ExtraSpec").LoadAsync();
```

這寫法適合做「動態載入」（例如在 generic repository 或 runtime 決定屬性名），
但如果手寫在程式裡，建議用 Lambda。

---

## 四、顯式載入 (Explicit Loading) 的完整結構

再整理一下常見用法：

```csharp
// 載入單一參照
await ctx.Entry(order).Reference(o => o.Customer).LoadAsync();

// 載入集合
await ctx.Entry(order).Collection(o => o.Items).LoadAsync();

// 載入集合，附加條件
await ctx.Entry(order).Collection(o => o.Items)
    .Query()
    .Where(i => i.Quantity > 10)
    .LoadAsync();
```

EF Core 都會在背景生成對應的 SQL 並在記憶體中自動連結。

---

## 五、延伸：為什麼不用 Include？

`Include` 是「查詢時就一起載入」；
`Reference(...).LoadAsync()` 是「查回來後，再手動補載」。

所以這行程式碼通常出現在互動式流程裡：

```csharp
var order = await ctx.Orders.FindAsync(orderId);
// 初次只載入主檔，避免拉太多資料
...
// 使用者點開商品細節時，再顯式載入產品規格
await ctx.Entry(item.Product).Reference(p => p.ExtraSpec).LoadAsync();
```

這樣能精準控制「什麼時候查資料」，減少不必要的查詢與資料量。

---