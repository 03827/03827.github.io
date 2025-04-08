
# 💻 Bootstrap 5 中階專案模板教學講解

## 🎯 教學目標

本教學將帶你逐步了解如何建立一個具備 **側邊欄、儀表板、RWD 表格與互動模態窗** 的中階 Bootstrap 專案結構。

---

## 🧱 頁面基本結構

```html
<!DOCTYPE html>
<html lang="zh-Hant">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Bootstrap 5 中階專案模板</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet" />
  <style>
    /* 自訂樣式 */
  </style>
</head>
<body>
  <!-- 內容區 -->
</body>
</html>
```

📌 使用 CDN 匯入 Bootstrap 5 並撰寫自訂 CSS 來調整外觀與動畫。

---

## 📚 頁面主架構（RWD）

```html
<div class="container-fluid">
  <div class="row">
    <!-- Sidebar -->
    <nav class="col-md-2 d-none d-md-block sidebar p-3">
      ...
    </nav>

    <!-- Main content -->
    <main class="col-md-10 ms-sm-auto col-lg-10 px-md-4">
      ...
    </main>
  </div>
</div>
```

📌 使用 `container-fluid` 搭配 `row` 分出兩欄：側邊欄與主內容區。`col-md-2` 表示中等螢幕以上占 2 欄寬，手機則隱藏（`d-none d-md-block`）。

---

## 📋 側邊欄（Sidebar）

```html
<nav class="col-md-2 d-none d-md-block sidebar p-3">
  <h5 class="text-center">選單</h5>
  <ul class="nav flex-column">
    <li class="nav-item"><a class="nav-link active" href="#">儀表板</a></li>
    <li class="nav-item"><a class="nav-link" href="#">報表</a></li>
    <li class="nav-item"><a class="nav-link" href="#">用戶管理</a></li>
  </ul>
</nav>
```

📌 使用 `nav` + `flex-column` 垂直導覽列；`active` 為當前頁面加上高亮樣式。

---

## 📊 儀表板統計卡片（Card）

```html
<div class="row g-4">
  <div class="col-md-4">
    <div class="card card-stats shadow-sm">
      <div class="card-body">
        <h5 class="card-title">本月訪客</h5>
        <p class="card-text fs-4">12,304</p>
      </div>
    </div>
  </div>
</div>
```

📌 使用 `card` 搭配 `shadow-sm` 加上輕微陰影，並透過 `.fs-4` 放大字體，`.card-stats:hover` 提供 hover 動畫。

---

## 🧾 響應式表格（Responsive Table）

```html
<div class="table-responsive">
  <table class="table table-hover">
    <thead>
      <tr><th>#</th><th>姓名</th><th>Email</th><th>註冊日期</th><th>狀態</th></tr>
    </thead>
    <tbody>
      <tr><td>1</td><td>王小明</td><td>ming@example.com</td><td>2025-04-01</td><td><span class="badge bg-success">啟用</span></td></tr>
    </tbody>
  </table>
</div>
```

📌 `table-responsive` 可橫向捲動（小螢幕）；`.badge` 表示狀態標籤。

---

## 📦 模態視窗（Modal）

```html
<!-- 按鈕觸發 -->
<button class="btn btn-primary" data-bs-toggle="modal" data-bs-target="#exampleModal">新增用戶</button>

<!-- 模態框本體 -->
<div class="modal fade" id="exampleModal" tabindex="-1">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title">新增用戶</h5>
        <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
      </div>
      <div class="modal-body">
        <form>
          <div class="mb-3"><label class="form-label">用戶名稱</label><input type="text" class="form-control" required></div>
          <div class="mb-3"><label class="form-label">Email</label><input type="email" class="form-control" required></div>
        </form>
      </div>
      <div class="modal-footer">
        <button class="btn btn-secondary" data-bs-dismiss="modal">取消</button>
        <button class="btn btn-primary">儲存</button>
      </div>
    </div>
  </div>
</div>
```

📌 按鈕使用 `data-bs-toggle="modal"` 與 `data-bs-target` 指定模態視窗；`modal-content` 包含標題、內容與按鈕。

---

## 🏁 結語

這個中階 Bootstrap 專案具備了：

- ✅ 響應式結構與排版（Grid + Utility）
- ✅ 卡片元件與表格整合
- ✅ 模態框互動
- ✅ 實務常見儀表板功能

你可以依此架構，延伸如後台系統、會員中心、分析報表等應用。

如果你需要這個專案的完整 HTML 或轉成 Vue/React 等框架，也可以請我幫忙！

---

> 👨‍🏫 本教學由 ChatGPT 擔任「網頁開發講師」誠意製作 🙌
