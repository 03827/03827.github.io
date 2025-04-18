# C# 教學

---

## 第 1 小時：認識 C# 與 .NET 平台

### 🎯 教學目標
- 瞭解 C# 語言的特性與用途
- 認識 .NET 平台的發展與組成
- 熟悉開發工具與執行方式
- 撰寫第一支 Hello World 程式

### 📚 教學內容
1. C# 是什麼？（微軟於 2000 年推出的語言）
2. .NET 平台架構：.NET Framework、.NET Core、.NET 6+（跨平台）
3. 開發工具介紹：Visual Studio、VS Code、.NET CLI
4. 建立專案與執行程式：使用命令列 `dotnet new console`

### 🧑‍💻 範例程式
```csharp
using System;

class Program
{
    static void Main()
    {
        Console.WriteLine("Hello, World!");
    }
}
```

### 📝 小練習
1. 嘗試更改輸出文字為你的名字。
2. 在程式中加入兩行輸出。

---

## 第 2 小時：基本語法與變數

### 🎯 教學目標
- 學會宣告與使用變數
- 熟悉資料型別與轉型
- 掌握字串插值與常用函式

### 📚 教學內容
1. 資料型別：int, float, double, string, bool, char
2. 變數與常數的宣告
3. 字串插值與組合
4. 類型轉換：Parse(), Convert, ToString()

### 🧑‍💻 範例程式
```csharp
int age = 18;
string name = "Linda";
string lastName = "Chen";
bool isMember = true;
Console.WriteLine("1. 姓名: {0}, 年齡: {1}, 會員: {2}", name, age, isMember);
Console.WriteLine("1-1. 姓名: {2}, 年齡: {1}, 會員: {0}", name, age, isMember);
Console.WriteLine("1-2. 姓名: {2}, 年齡: {1}, 會員: {0}\n", isMember, age, name);

age = age + 2;
name = name + ' ' + lastName;
Console.WriteLine($"2. 姓名: {name}, 年齡: {age}, 會員: {isMember}");

age += 2;
name += " " + 'I';
Console.WriteLine($"3. 姓名: {name}, 年齡: {age}, 會員: {isMember}");

age++;
name = name + "I";
Console.WriteLine($"4. 姓名: {name}, 年齡: {age}, 會員: {isMember}");

++age;
Console.WriteLine($"5. 姓名: {name}, 年齡: {age}, 會員: {isMember}");

Console.WriteLine($"6. 姓名: {name}, 年齡: {age++}, 會員: {isMember}");
Console.WriteLine($"7. 姓名: {name}, 年齡: {++age}, 會員: {isMember}");
```

### 📝 小練習
1. 宣告一個 double 型別的身高，並輸出。
2. 將字串 "123" 轉換為整數並相加。

---

## 第 3 小時：控制流程（條件判斷與迴圈）

### 🎯 教學目標
- 理解條件判斷邏輯
- 熟悉各種迴圈語法
- 培養流程控制的邏輯思維

### 📚 教學內容
1. if, else if, else
2. switch-case 語句
3. for, while, do-while 迴圈
4. break 與 continue 控制流程

### 🧑‍💻 範例程式
```csharp
// if-else 條件判斷
int score = 85;
if (score >= 90)
{
    Console.WriteLine("優秀");
}
else if (score >= 70)
{
    Console.WriteLine("及格");
}
else
{
    Console.WriteLine("不及格");
}

// switch-case 條件選擇
string grade = "B";
switch (grade)
{
    case "A":
        Console.WriteLine("90-100 分");
        break;
    case "B":
        Console.WriteLine("80-89 分");
        break;
    case "C":
        Console.WriteLine("70-79 分");
        break;
    default:
        Console.WriteLine("70 分以下");
        break;
}

// for 迴圈
for (int i = 1; i <= 5; i++)
{
    if (i == 3) continue;
    Console.WriteLine($"第 {i} 次");
}

// while 迴圈
int count = 1;
while (count <= 3)
{
    Console.WriteLine($"while 迴圈次數: {count}");
    count++;
}

// do-while 迴圈
int num = 1;
do
{
    Console.WriteLine($"do-while 迴圈次數: {num}");
    num++;
} while (num <= 3);
```

### 📝 小練習
1. 撰寫一段判斷成績等級的 if-else 程式。
2. 使用 while 迴圈印出 1 到 10。

---

## 第 4.5 小時：方法與參數

### 🎯 教學目標
- 定義與呼叫方法
- 使用參數與回傳值
- 理解方法多載與預設參數

### 📚 教學內容
1. 方法語法結構（參數與回傳值）
2. 傳值、傳參考（ref/out）
3. 預設參數與多載 overload

### 🧑‍💻 範例程式
```csharp
static int Multiply(int x, int y = 2)
{
    return x * y;
}

Console.WriteLine(Multiply(5));
```

### 📝 小練習
1. 撰寫一個回傳 BMI 值的方法。
2. 嘗試使用 ref 傳遞變數，修改值後再輸出。

---

## 第 6.5 小時：物件導向設計

### 🎯 教學目標
- 建立類別與物件
- 使用屬性與方法
- 瞭解繼承與多型

### 📚 教學內容
1. 類別與建構子
2. 屬性（get/set）
3. 繼承與 override
4. 抽象類別與介面（簡介）

### 🧑‍💻 範例程式
```csharp
class Animal
{
    public virtual void Speak() => Console.WriteLine("發出聲音");
}

class Dog : Animal
{
    public override void Speak() => Console.WriteLine("汪汪!");
}

Animal pet = new Dog();
pet.Speak();
```

### 📝 小練習
1. 建立一個類別 Car，包含品牌與啟動方法。
2. 使用繼承建立不同類型的車輛（如 Truck、Bike）。

---

## 第 7.5 小時：例外處理與檔案操作

### 🎯 教學目標
- 理解 try/catch 結構
- 掌握檔案讀寫基本技巧

### 📚 教學內容
1. try / catch / finally
2. throw new Exception
3. 使用 File.ReadAllText / WriteAllText

### 🧑‍💻 範例程式
```csharp
try
{
    string content = File.ReadAllText("data.txt");
    Console.WriteLine(content);
}
catch (Exception ex)
{
    Console.WriteLine($"錯誤訊息: {ex.Message}");
}
```

### 📝 小練習
1. 嘗試將資料寫入檔案後再讀出。
2. 模擬除以零的例外情況並捕捉錯誤。

---

## 第 9 小時：集合與 LINQ

### 🎯 教學目標
- 使用集合（List, Dictionary）儲存資料
- 學會基本 LINQ 查詢語法

### 📚 教學內容
1. 陣列與 List<T>
2. Dictionary<TKey, TValue>
3. LINQ 語法 (Where, Select)

### 🧑‍💻 範例程式
```csharp
List<int> numbers = new List<int> { 1, 2, 3, 4, 5 };
var evens = numbers.Where(n => n % 2 == 0);

foreach (var n in evens)
{
    Console.WriteLine(n);
}
```

### 📝 小練習
1. 建立一個包含姓名的 List，使用 LINQ 查詢特定名字。
2. 使用 Dictionary 儲存產品與價格，並列出所有項目。

---

## 第 10 小時：小型專案實作

### 🎯 教學目標
- 實作聯絡人管理系統
- 整合課程所學：類別、方法、集合、控制流程

### 📚 教學內容
1. 建立 Contact 類別
2. 使用 List<Contact> 管理資料
3. 撰寫新增、查詢、刪除方法

### 🧑‍💻 專案範例架構
```csharp
class Contact
{
    public string Name { get; set; }
    public string Phone { get; set; }
}

List<Contact> contacts = new List<Contact>();
```

### 📌 功能建議
1. 新增聯絡人功能
2. 查詢聯絡人（依姓名）
3. 刪除聯絡人
4. 顯示所有聯絡人

### 📝 挑戰題
- 將資料儲存到檔案，下次啟動時讀取
- 將 LINQ 套用在查詢功能上
- 建立簡易選單介面（用 switch 實作）

