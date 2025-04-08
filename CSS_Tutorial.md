
# 🎓 CSS 初學者完整教學講義

> 👨‍🏫 本教學以「邊學邊看原始碼」的方式，逐段解說 CSS 的概念與範例，並說明其在實務中的使用情境。

---

## 📌 基本樣式設定

### 範例原始碼

```css
body {
  font-family: Arial, sans-serif;
  line-height: 1.6;
  margin: 20px;
  background-color: #f9f9f9;
}

h1, h2 {
  color: #333;
}
```

### 教學說明

- `font-family`: 設定字體，`sans-serif` 為無襯線字體，適合螢幕閱讀。
- `line-height`: 行距設為 1.6，讓文字更容易閱讀。
- `margin`: 頁面外邊距為 20px，避免畫面太擁擠。
- `background-color`: 設定頁面背景色為淡灰色。
- `color`: 設定標題文字顏色為深灰（#333），提升對比度與可讀性。

---

## 🎨 顏色與背景應用

### 範例原始碼

```css
.color-example {
  color: blue;
}

.bg-example {
  background-color: lightyellow;
  padding: 10px;
}
```

### 教學說明

- `color`: 設定文字顏色。
- `background-color`: 設定背景色。
- `padding`: 內距，讓文字不會緊貼邊界。

---

## 🅰️ 字體大小與粗細

```css
.font-example {
  font-size: 20px;
  font-weight: bold;
}
```

- `font-size`: 設定字體大小。
- `font-weight`: 設定字重為粗體（`bold`）。

---

## 📦 邊框與內距

```css
.box-example {
  border: 2px dashed green;
  padding: 15px;
  margin-top: 10px;
}
```

- `border`: 設定邊框樣式（寬度、虛線、顏色）。
- `padding`: 內距。
- `margin-top`: 上方外距。

---

## 🧾 Class 與 ID 的應用

```css
.class-example {
  color: red;
}

.id-example {
  color: purple;
}
```

- `.` 表示類別選擇器（class），可應用於多個元素。
- `#` 表示 ID 選擇器（id），只用於單一元素。

---

## 📐 Flexbox 彈性盒子排版

### 容器與項目設定

```css
.flex-container {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.flex-item {
  background-color: #add8e6;
  padding: 20px;
  flex: 1;
  text-align: center;
  min-width: 100px;
}

.flex-container.row {
  flex-direction: row;
}

.flex-container.column {
  flex-direction: column;
}
```

### 教學說明

- `display: flex`: 啟用 Flex 排版。
- `gap`: 設定項目間距。
- `flex-wrap: wrap`: 項目太多時自動換行。
- `flex: 1`: 每個子項目均分剩餘空間。
- `flex-direction`: 控制主軸排列方向（row：水平，column：垂直）。
- `min-width`: 設定子項目的最小寬度，避免過小。

---

## 📊 Grid 網格排版（額外補充）

```css
.grid-container {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 10px;
}

.grid-item {
  background-color: #ffe4b5;
  padding: 20px;
  text-align: center;
}
```

- `display: grid`: 啟用網格排版。
- `grid-template-columns`: 建立三欄等寬結構。
- `gap`: 格子間距。
- `1fr`: 表示分攤可用空間的比例單位。

---

## 🎨 使用 CSS 變數

```css
:root {
  --main-color: #3498db;
  --accent-color: #2ecc71;
}

.variable-example {
  color: var(--main-color);
  background-color: var(--accent-color);
  padding: 10px;
}
```

- `:root`: 定義全域變數。
- `--變數名稱`: 宣告自訂變數。
- `var(--變數名稱)`: 使用變數。

---

## 🧬 巢狀選擇器

```css
.nesting-parent {
  color: #333;
}

.nesting-parent .child {
  color: #999;
}
```

- `.nesting-parent`: 父層元素。
- `.child`: 子層元素。
- 巢狀結構可以避免樣式污染整個頁面。

---

## 📏 尺寸單位教學

### ✅ px（像素）

```css
.fixed-size {
  width: 200px;
  height: 100px;
}
```

### ✅ %（百分比）

```css
.relative-size {
  width: 50%;
  height: 30%;
}
```

### ✅ em

```css
.em-size {
  font-size: 2em;
  margin-top: 1.5em;
}
```

### ✅ rem

```css
.rem-size {
  font-size: 1.5rem;
  margin-bottom: 2rem;
}
```

### ✅ vh / vw（視口）

```css
.viewport-size {
  width: 50vw;
  height: 50vh;
}
```

### ✅ 混合單位

```css
.mixed-size {
  width: 50%;
  height: 200px;
}
```

---

## 🧾 結語建議

這份教學涵蓋了：

- CSS 基本語法與結構  
- 顏色、字體、邊框、間距等常用樣式  
- Flexbox 彈性盒子技巧  
- Grid 網格佈局  
- 尺寸單位與響應式設計  
- CSS 變數與巢狀選擇器  

✅ 建議搭配 DevTools 練習觀察元素變化。  
✅ 下一步可以深入學習 Media Query、動畫、SCSS、BEM 命名法等進階主題。
