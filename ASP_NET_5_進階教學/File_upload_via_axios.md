# 前端手動組 `FormData`
```csharp
class ModelA
{
    public string name { get; set; }
    public int age { get; set; }
    public string phone { get; set; }
}

class ModelB
{
    public IFormFile fileAttach { get; set; }
}

class MyModel
{
    public ModelA main { get; set; }
    public List<ModelB> details { get; set; }
}
```

也就是：

* `main` → 一筆主資料（`name`、`age`、`phone`）
* `details` → 多筆明細，每一筆都是一個 `ModelB`，裡面有 `fileAttach`

在這種情況下，**前端送的 FormData 鍵名要符合陣列格式**，也就是用索引 `details[0].fileAttach`、`details[1].fileAttach` …。
ASP.NET Core 的 model binder 看到這樣的鍵名，就會自動組出 `List<ModelB>`。

---

## ✅ 後端 Action

一樣維持：

```csharp
[HttpPost]
[ValidateAntiForgeryToken]
[RequestSizeLimit(100L * 1024 * 1024)] // 例：100MB
[RequestFormLimits(MultipartBodyLengthLimit = 50L * 1024 * 1024)] //50MB
public async Task<IActionResult> SubmitForm([FromForm] MyModel model) // [FromForm]
{
    var main = model.main;
    var details = model.details;

    if (main == null) return BadRequest("沒有主檔");
    if (details == null || details.Count == 0) return BadRequest("沒有明細檔");

    //取得附件
    foreach (var d in details)
    {
        if (d.fileAttach == null || d.fileAttach.Length == 0)
            continue;

        var ext = Path.GetExtension(d.fileAttach.FileName);
        var saveName = Path.GetRandomFileName() + ext;
        var path = Path.Combine(_env.WebRootPath ?? _env.ContentRootPath, "uploads", saveName);
        Directory.CreateDirectory(Path.GetDirectoryName(path)!);

        //存檔
        await using var fs = System.IO.File.Create(path);
        await d.fileAttach.CopyToAsync(fs);
    }

    return Ok(new { message = "OK", main, detailCount = details.Count });
}
```

---

## ✅ 前端：用 axios + FormData

### 1. 單筆主資料 + 多筆 detail 檔案，用 JavaScript 動態新增檔案欄位

```js
const fd = new FormData();
fd.append("main.name", "王小明");
fd.append("main.age", 30);
fd.append("main.phone", "0912345678");

const files = document.getElementById("files").files; // 多選
for (let i = 0; i < files.length; i++) {
  fd.append(`details[${i}].fileAttach`, files[i]);
}


await axios.post("/Upload/SubmitForm", fd, {
  onUploadProgress: (evt) => console.log(`進度: ${Math.round((evt.loaded*100)/evt.total)}%`)
});
```

---

## 💡重點整理

1. **主檔巢狀屬性 → `main.xxx`**
2. **明細清單 → `details[index].屬性`**
3. **多檔上傳**時用索引命名，model binder 會自動組出 List。
4. **不要手動設定 Content-Type**，讓 axios 自己產生 multipart boundary。
5. Anti-Forgery Token 一樣丟進 FormData 或用 header。

---

這樣一來，後端的 `MyModel` 就能完整接收到：

```csharp
model.main.name
model.main.age
model.main.phone
model.details[0].fileAttach
model.details[1].fileAttach
...
```

想更進階還可以讓每個 `ModelB` 除了檔案外還有描述欄位（例如 `description`），照樣加成
`details[0].description`、`details[0].fileAttach` 就行。

---

## 補充
1. RequestSizeLimit：允許整個封包大小上限。
2. FormOptions.MultipartBodyLengthLimit：允許封包裡面的 Body 欄位大小上限。
   * 未手動設制時，預設大小為 128MB。
3. 常見的大小設制方法

   1. `Startup.cs`

    ```csharp
    services.Configure<FormOptions>(o =>
    {
        o.MultipartBodyLengthLimit = 1L * 1024 * 1024 * 1024;
    });
    ```

   2. `appsettings.json` or `program.cs`

   ```json
   { "Kestrel": { "Limits": { "MaxRequestBodySize": 1073741824 } } }
   ```

   ```csharp
   webBuilder.ConfigureKestrel(options =>
   {
       options.Limits.MaxRequestBodySize = 1L * 1024 * 1024 * 1024;
   });
   ```

   3. `web.config`

   ```xml
   <requestFiltering>
     <requestLimits maxAllowedContentLength="1073741824" />
   </requestFiltering>
   ```

   4. `Action`

   ```csharp
   [RequestSizeLimit(1L * 1024 * 1024 * 1024)]
   [RequestFormLimits(MultipartBodyLengthLimit = 1L * 1024 * 1024 * 1024)]
   public IActionResult Upload(IFormFile file) { ... }
   ```
4. 判斷優先順序：由上到下為阻檔上傳的優先判斷

   1. **IIS**

      * 例：`requestFiltering.requestLimits.maxAllowedContentLength`、`uploadReadAheadSize`。
      * 這層比 Kestrel 更外面；IIS 先擋，App 不會執行。

   2. **Kestrel（.NET 內建伺服器）**

      * `MaxRequestBodySize`（`appsettings.json` 或 `program.cs`）。
      * 超過時請求會被拒收；控制器不會被叫到。

   3. **ASP.NET Core「Action 級」限制**

      * `[RequestSizeLimit]`：整個 request 上限      

   4. **表單解析層（MVC 模型繫結）**

      * `[RequestFormLimits]` / `FormOptions.MultipartBodyLengthLimit`：**只影響 multipart/form-data 解析**。
      * 即使前面都放行，若這裡較小，解析 multipart 時一樣會丟 413/例外。