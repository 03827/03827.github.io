# 🧱 第 1 章：OOP 基礎觀念

## 🎓 教學目標

1. 瞭解物件導向（OOP）與程序式程式設計的差異
2. 掌握 OOP 的四大支柱：封裝、繼承、多型、抽象
3. 了解 C# 作為 OOP 語言的基本設計理念

---

## 🧭 1. 程式設計範式簡介

### ✅ 程序式（Procedural Programming）

- 將問題分解為一連串的「指令、流程與資料」
- 常用於小型腳本或處理流程線性簡單的場景

### ✅ 物件導向（Object-Oriented Programming）

- 強調「資料（狀態）」與「行為（方法）」封裝在一起
- 程式架構較具彈性與可重用性

| 比較項目   | 程序式                | 物件導向             |
| ---------- | --------------------- | -------------------- |
| 單元單位   | 函式、流程            | 類別、物件           |
| 資料與邏輯 | 分離                  | 封裝在物件中         |
| 重用方式   | 複製貼上              | 繼承與多型           |
| 範例語言   | C, Python（某些寫法） | C#, Java, C++, Swift |

---

## 🧭 2. OOP 的四大支柱

### 🟦 2.1 封裝（Encapsulation）

- 將資料和方法打包在一起，並隱藏內部的複雜實作
- 外部只能透過「方法」與「屬性」操作內部狀態

### 🟪 2.2 繼承（Inheritance）

- 子類別可以繼承父類別的屬性與方法
- 可減少重複程式碼

### 🟥 2.3 多型（Polymorphism）

- 同樣的方法呼叫，對不同物件有不同的行為
- 分為：

  - 多載（overload）
  - 覆寫（override）

### 🟨 2.4 抽象（Abstraction）

- 隱藏內部實作細節，只暴露必要的操作介面
- 實現方式：

  - 抽象類別
  - 介面（interface）

---

## 🛠 範例：OOP 四大支柱

```csharp
// 封裝 + 繼承 + 多型 + 抽象簡介
public abstract class Animal
{
    public string Name { get; set; }
    public abstract void Speak();      // 抽象方法
}

public class Dog : Animal
{
    public override void Speak()       // 多型 (override)
    {
        Console.WriteLine($"{Name}：汪汪！");
    }
}

public class Cat : Animal
{
    public override void Speak()
    {
        Console.WriteLine($"{Name}：喵喵～");
    }
}
```

### 📌 測試程式

```csharp
Animal dog = new Dog { Name = "小白" };
Animal cat = new Cat { Name = "小花" };
dog.Speak();   // 小白：汪汪！
cat.Speak();   // 小花：喵喵～
```

---

# 📘 第 2 章：封裝（Encapsulation）

## 🎯 教學目標

- 瞭解封裝在 OOP 中的角色與重要性
- 熟悉 C# 權限修飾詞的運作與應用
- 實作屬性 (`property`) 控制資料存取與驗證
- 建立具備資料保護機制的類別設計能力

---

## 🧱 封裝概念說明

### 什麼是封裝？

封裝是一種將**資料與行為綁在一起**，並**限制外部直接存取資料**的技術，
就像你使用微波爐時，只需要按按鈕（公開的介面），而不需要知道內部的電路如何運作（私有的實作）。

透過封裝可以：

- 隱藏內部實作細節（提高安全性）
- 控制對欄位的存取與驗證（提高穩定性）

---

## 🔐 權限修飾詞（Access Modifiers）

| 修飾詞      | 存取範圍說明                                                |
| ----------- | ----------------------------------------------------------- |
| `public`    | 任何地方都能存取                                            |
| `private`   | 僅限於類別內部可存取（封裝的核心）                          |
| `protected` | 僅限於類別內部與其子類別可存取                              |
| `internal`  | 僅限於同一命名空間/專案(Assembly)中的類別可存取（進階使用） |

---

## 📦 使用屬性封裝欄位，以 Facade（外觀）模式為例

### 核心概念：銀行櫃員就是一個封裝後的介面

想像一下你去銀行辦理提款。你只需要走到櫃檯，把你的提款單及資料交給櫃員，然後告訴他要提多少錢。

這位**櫃員就是一個「Facade」**。

在你看不見的背後，櫃員可能需要做好幾件事：

1.  **驗證帳戶**：確認你的帳號是否存在。
2.  **安全檢查**：確認你的密碼是否正確。
3.  **資金確認**：檢查你的帳戶餘額是否足夠。
4.  **執行扣款**：從你的帳戶中扣除金額。
5.  **記錄交易**：在銀行的交易日誌上留下紀錄。

你不需要親自去跟「帳戶系統」、「安全系統」、「資金系統」打交道。你只需要跟「櫃員」這個單一窗口溝通。Facade 模式就是扮演這位櫃員的角色，將背後複雜的流程簡化成一個簡單的操作。

---

### C\# 程式碼範例

#### 1\. 複雜的銀行後端子系統 (The Complex Subsystems)

這些是銀行內部處理不同業務的系統，客戶通常不會直接接觸它們。

```csharp
// 子系統1：帳戶驗證系統
public class AccountService
{
    // 模擬檢查帳戶是否存在
    public bool CheckAccount(string accountNumber)
    {
        Console.WriteLine($"  [子系統] 正在驗證帳號：{accountNumber}");
        if (accountNumber == "123-456-789")
        {
            return true;
        }
        Console.WriteLine("  [子系統] 錯誤：帳號不存在。");
        return false;
    }
}

// 子系統2：安全驗證系統
public class SecurityService
{
    // 模擬檢查密碼是否正確
    public bool CheckPinCode(string accountNumber, int pin)
    {
        Console.WriteLine("  [子系統] 正在驗證密碼...");
        if (accountNumber == "123-456-789" && pin == 8888)
        {
            return true;
        }
        Console.WriteLine("  [子系統] 錯誤：密碼錯誤。");
        return false;
    }
}

// 子系統3：資金管理系統
public class FundsService
{
    // 模擬帳戶餘額
    private decimal _balance = 10000m;

    // 模擬檢查餘額是否足夠
    public bool HasSufficientFunds(decimal amount)
    {
        Console.WriteLine("  [子系統] 正在檢查餘額...");
        if (_balance >= amount)
        {
            Console.WriteLine($"  [子系統] 目前餘額: {_balance:C}，足夠提款。");
            return true;
        }
        Console.WriteLine($"  [子系統] 錯誤：餘額不足！目前餘額: {_balance:C}");
        return false;
    }

    // 模擬存款
    public void Deposit(decimal amount)
    {
        _balance += amount;
        Console.WriteLine($"  [子系統] 成功存入 {amount:C}。");
    }

    // 模擬提款
    public void Withdraw(decimal amount)
    {
        _balance -= amount;
        Console.WriteLine($"  [子系統] 成功提出 {amount:C}。");
    }

    public void ShowBalance()
    {
        Console.WriteLine($"  [子系統] 目前帳戶最終餘額為: {_balance:C}");
    }
}
```

#### 2\. 銀行櫃員外觀類別 (The BankAccount Facade)

這個 `BankAccountFacade` 就是我們的櫃員。它簡化了所有操作，提供給客戶 `Deposit` 和 `Withdraw` 兩個簡單的方法。

```csharp
public class BankAccountFacade
{
    // Facade 持有所有子系統的參考
    private readonly string _accountNumber;
    private readonly int _pin;
    private readonly AccountService _accountService;
    private readonly SecurityService _securityService;
    private readonly FundsService _fundsService;

    // 客戶開戶時，就建立好所有需要的服務
    public BankAccountFacade(string accountNumber, int pin)
    {
        _accountNumber = accountNumber;
        _pin = pin;

        // 初始化所有子系統
        _accountService = new AccountService();
        _securityService = new SecurityService();
        _fundsService = new FundsService();
    }

    // 提供給客戶的「存款」單一方法
    public void Deposit(decimal amount)
    {
        Console.WriteLine($"\n======= 開始處理存款 {amount:C} =======");
        // 1. 驗證帳號
        if (!_accountService.CheckAccount(_accountNumber)) return;
        // 2. 驗證密碼
        if (!_securityService.CheckPinCode(_accountNumber, _pin)) return;

        // 3. 執行存款
        _fundsService.Deposit(amount);
        _fundsService.ShowBalance();
        Console.WriteLine("======= 存款處理完成 =======");
    }

    // 提供給客戶的「提款」單一方法
    public void Withdraw(decimal amount)
    {
        Console.WriteLine($"\n======= 開始處理提款 {amount:C} =======");
        // 1. 驗證帳號
        if (!_accountService.CheckAccount(_accountNumber)) return;
        // 2. 驗證密碼
        if (!_securityService.CheckPinCode(_accountNumber, _pin)) return;
        // 3. 檢查餘額
        if (!_fundsService.HasSufficientFunds(amount)) return;

        // 4. 執行提款
        _fundsService.Withdraw(amount);
        _fundsService.ShowBalance();
        Console.WriteLine("======= 提款處理完成 =======");
    }
}
```

#### 3\. 客戶端 (The Client)

客戶（也就是我們）只需要跟 `BankAccountFacade` 互動，完全不用理會後端的複雜系統。

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        // 客戶來到銀行，只需要提供帳號密碼給櫃員(Facade)
        var myAccount = new BankAccountFacade("123-456-789", 8888);

        // === 案例1：成功提款 ===
        myAccount.Withdraw(500m);

        // === 案例2：成功存款 ===
        myAccount.Deposit(2000m);

        // === 案例3：提款失敗 (餘額不足) ===
        myAccount.Withdraw(20000m);

        // === 案例4：使用錯誤的密碼進行操作 ===
        var wrongAccount = new BankAccountFacade("123-456-789", 9999);
        wrongAccount.Withdraw(100m);
    }
}
```

### 執行結果

執行程式後，輸出結果非常清晰地展示了整個流程：

```
======= 開始處理提款 NT$500.00 =======
  [子系統] 正在驗證帳號：123-456-789
  [子系統] 正在驗證密碼...
  [子系統] 正在檢查餘額...
  [子系統] 目前餘額: NT$10,000.00，足夠提款。
  [子系統] 成功提出 NT$500.00。
  [子系統] 目前帳戶最終餘額為: NT$9,500.00
======= 提款處理完成 =======

======= 開始處理存款 NT$2,000.00 =======
  [子系統] 正在驗證帳號：123-456-789
  [子系統] 正在驗證密碼...
  [子系統] 成功存入 NT$2,000.00。
  [子系統] 目前帳戶最終餘額為: NT$11,500.00
======= 存款處理完成 =======

======= 開始處理提款 NT$20,000.00 =======
  [子系統] 正在驗證帳號：123-456-789
  [子系統] 正在驗證密碼...
  [子系統] 正在檢查餘額...
  [子系統] 錯誤：餘額不足！目前餘額: NT$11,500.00

======= 開始處理提款 NT$100.00 =======
  [子系統] 正在驗證帳號：123-456-789
  [子系統] 正在驗證密碼...
  [子系統] 錯誤：密碼錯誤。
```

這個範例清楚地顯示，客戶端程式碼非常簡單，只需要呼叫 `myAccount.Withdraw()` 和 `myAccount.Deposit()`。所有帳戶、安全、資金的檢查邏輯都被完美地封裝在 `BankAccountFacade` 裡面了。

---

# 📘 第 3 章：繼承與多型（Inheritance & Polymorphism）

## 🎯 教學目標

- 理解繼承的概念與語法
- 能夠定義父類別與子類別
- 瞭解多型的應用方式（方法覆寫、虛擬方法）
- 熟悉 `base`, `virtual`, `override`, `new` 的用法

---

## 🧬 繼承的基本概念

### 定義

- 繼承（Inheritance）是讓一個類別（子類別）繼承另一個類別（父類別）的屬性與行為
- 可用來共用邏輯、擴充功能、建立層次關係

### 語法：

```csharp
public class Animal
{
    public string Name;

    public Animal(string name)
    {
        this.Name = name;
        Console.WriteLine("Animal 的建構函式被呼叫。");
    }

    public void Eat()
    {
        Console.WriteLine($"{Name} 正在吃東西。");
    }
}

//Dog是一種Animal，故Animal有的一般特性Dog也都會有。
public class Dog : Animal
{
    //Dog 將擁有 Name 及 Eat 功能
}
```

---

## 📚 使用 `base` 關鍵字

- `base` 可以呼叫父類別的建構子或成員

```csharp
public class Dog : Animal
{
    public Dog(string name) : base(name)
    {
        Console.WriteLine("Dog 的建構函式被呼叫。");
    }
}
```

---

## 🔁 方法覆寫（Override）

- C# 支援**虛擬方法**（virtual）與**覆寫方法**（override）

```csharp
public class Animal
{
    //虛擬方法 (virtual)，允許被子類覆寫
    public virtual void Speak()
    {
        Console.WriteLine("不特定的聲音");
    }
}

public class Cat : Animal
{
    //被子類別 Cat 覆寫
    public override void Speak()
    {
        Console.WriteLine("喵喵！");
    }
}
```

---

## ✂️ 使用 `new` 隱藏成員

```csharp
public class Pig : Animal
{
    public new void Speak()
    {
        Console.WriteLine("哼哼！");
    }

    //Pigs 獨有的 Method: 獻祭
    public void Sacrificed()
    {
        Console.WriteLine("我願走上神桌，只盼今後國泰民安、風調雨順!");
    }
}
```

📌 注意：`new` 並不會覆蓋原本的方法，而會讓父類別與子類別擁有 2 個同名的函式。

---

## 🔁 多型（Polymorphism）

- 多型允許用父類型參考子物件，呼叫虛擬方法時會執行實際類型的實作

```csharp
Animal a = new Cat();
a.Speak(); // 輸出：喵喵！ (override)

a = new Pig();
a.Speak(); // 輸出：不特定的聲音（new）

Pig _pig = new Pig();
_pig.Speak(); // 輸出：哼哼！（new）
```

---

## 🧪 實作範例

```csharp
List<Animal> zoo = new List<Animal>
{
    new Dog { Name = "小黃" },
    new Cat { Name = "小白" },
    new Pig { Name = "小黑" }
};

foreach (Animal animal in zoo)
{
    animal.Speak(); // 執行對應的多型方法
}
```

---

## 📝 練習題

### 題目 1：

定義以下類別繼承關係：

- `Vehicle`：基本資訊、方法 `Start()`
- `Car`、`Bus`、`Bike` 繼承 `Vehicle` 並覆寫 `Start()` 行為

### 題目 2：

建立 `Shape` 類別，並由其繼承出 `Circle` 與 `Square`，定義虛擬方法 `GetArea()` 並覆寫。

---

## ✅ 練習題解答

### 題目 1 解答：

```csharp
public class Vehicle
{
    public virtual void Start() => Console.WriteLine("啟動交通工具");
}

public class Car : Vehicle
{
    public override void Start() => Console.WriteLine("汽車發動！");
}

public class Bus : Vehicle
{
    public override void Start() => Console.WriteLine("公車啟動！");
}
```

### 題目 2 解答：

```csharp
public abstract class Shape
{
    public abstract double GetArea();
}

public class Circle : Shape
{
    public double Radius { get; set; }

    public override double GetArea() => Math.PI * Radius * Radius;
}
```

---

## 📌 型別轉型

```csharp
//向上轉型 (Upcasting，我個人稱之為「大轉小」)
Dog dog = new Dog();
Animal animal = dog;
animal.Speak(); //汪汪

//向下轉型 (Downcasting，我個人稱之為「小轉大」)
Pig pig = animal as Pig; //建議使用 as

//as 轉型成功，myDog 會得到物件參考；
//如果失敗（例如 animal 其實是 Dog），pig 會被設為 null，不會拋出例外。
//後續需要檢查 if (myDog != null)。

//強制轉型： Pig pig = (Pig)animal;
//如果轉型失敗，會直接拋出 InvalidCastException 例外。
```

---

## 📚 小結

| 概念 | 關鍵字                | 說明                   |
| ---- | --------------------- | ---------------------- |
| 繼承 | `:`                   | 擴充父類別功能         |
| 多型 | `virtual`, `override` | 執行時決定呼叫哪個方法 |
| 隱藏 | `new`                 | 編譯時決定隱藏原本成員 |

---

# 📘 第 4 章：抽象類別與介面（Abstract Classes & Interfaces）

## 🎯 教學目標

- 了解抽象類別與介面的差異與用途
- 實作抽象類別與介面
- 學會使用多介面設計、擴充與組合系統
- 熟悉抽象化在物件導向中的實務應用

---

## 🧱 抽象類別（abstract class）

### 定義

抽象類別是一種**不能被實例化(new)**的類別，它存在的目的就是為了被繼承。其中可包含：

- 已實作的方法與屬性（行為模板）
- 未實作的方法（抽象方法）

```csharp
public abstract class Animal
{
    public string Name { get; set; }

    public abstract void Speak();  // 抽象方法：子類別必須實作

    public void Sleep() => Console.WriteLine($"{Name} 在睡覺...");
}
```

---

## 🔻 實作抽象類別

```csharp
public class Dog : Animal
{
    public override void Speak()
    {
        Console.WriteLine($"{Name} 汪汪叫！");
    }
}
```

### 說明：

- 抽象類別提供**共用邏輯**
- 子類別**強制使用 override 關鍵字實作**抽象方法

---

## 📎 介面（interface）

### 定義

介面是一組**沒有實作**的方法與屬性的集合，僅定義規格，類似「合約、約定」。<br>
合約或約定的內容明白的定義了可以執行的方法名稱。

```csharp
public interface IFlyable
{
    void Fly();
}
```

### 實作：

```csharp
public class Bird : IFlyable
{
    public void Fly() => Console.WriteLine("小鳥飛起來了！");
}
```

---

## 🧩 多重介面實作

C# 支援類別實作多個介面：<BR>
Duck 可以飛也可以走，且飛跟走是由不同的生理系統負責。

```csharp
public interface IFlyable
{
    void Fly();
}

public interface IWalkable
{
    void Walk();
}

public interface WingSystem : IFlyable {
    void Charge();
}

public class Duck : WingSystem, IWalkable
{
    public void Fly() => Console.WriteLine("鴨子飛起來！");
    public void Walk() => Console.WriteLine("鴨子走起來！");
    public void Charge() => Console.WriteLine("鴨子振翅疾飛！");
}
```

鳥類大多數都會實作 `IFlyable` 及 `IWalkable`，但陸地生物大多數都不會實作 `IFlyable`，因此：

```csharp
// 這就是一個錯誤的用法，因為佩佩豬沒有實作 IFlyable。
Pig p佩佩豬 = new Pig();

// 1. 轉型失敗
IFlyable fly = p佩佩豬;

//2. 無法呼叫 Fly() 函式。
fly.Fly();
```

---

## 🆚 抽象類別 vs 介面 比較表

| 項目         | 抽象類別（abstract） | 介面（interface）                     |
| ------------ | -------------------- | ------------------------------------- |
| 可否有實作？ | ✅ 可有部分實作      | ❌ C# 7 前不可有任何實作, C# 8 後可以 |
| 繼承數量     | ❌ 僅限單一繼承      | ✅ 可多介面實作                       |
| 設計用途     | 提供模板與擴充機制   | 定義合約與行為規格                    |

---

## 📝 練習題

### 題目 1：

定義一個抽象類別 `Shape`，其中包含抽象方法 `GetArea()`，再由 `Rectangle`, `Circle` 繼承並實作。

### 題目 2：

定義介面 `IDriveable`，讓 `Car` 和 `Motorbike` 實作該介面並提供 `Drive()` 實作。

---

## ✅ 練習題解答

### 題目 1 解答：

```csharp
public abstract class Shape
{
    public abstract double GetArea();
}

public class Rectangle : Shape
{
    public double Width { get; set; }
    public double Height { get; set; }

    public override double GetArea() => Width * Height;
}
```

### 題目 2 解答：

```csharp
public interface IDriveable
{
    void Drive();
}

public class Car : IDriveable
{
    public void Drive() => Console.WriteLine("汽車行駛中");
}

public class Motorbike : IDriveable
{
    public void Drive() => Console.WriteLine("機車行駛中");
}
```

---

## 🧠 小結

- 抽象類別提供**共用結構與部分邏輯**
- 介面定義**行為合約**，適合建立系統協定
- <B><U>抽象類別適用於「is-a(是一種什麼：Animal - Pig - Speak)」，
  介面適用於「can-do(可以做什麼：IFlyable - Duck - Fly)」</U></B>

---

# 📘 第 5 章：類別設計原則與依賴注入

## 🎯 教學目標

- 學習如何設計低耦合、高內聚的類別結構
- 理解「單一職責原則」（SRP）與「依賴反轉原則」（DIP）
- 實作「建構子注入」與「介面注入」
- 理解依賴注入容器的基本角色（進階）

---

## 🧱 單一職責原則 SRP(Single Responsibility Principle)

> 一個類別應該只負責一件事情。

### ❌ 反例

```csharp
public class Report
{
    public string Title;
    public void Generate() { /* ... */ }
    public void SaveToFile() { /* IO 操作 */ }
}
```

🔍 問題：

- `Report` 類別負責兩件事：「產生報表」與「檔案處理」

---

### ✅ 重構後的 SRP

```csharp
public class ReportGenerator
{
    public string GenerateReport() => "報表內容...";
}

public class FileSaver
{
    public void Save(string content)
    {
        File.WriteAllText("report.txt", content);
    }
}
```

---

## 🌟 Open/Closed Principle（開放封閉原則）

---

### 📌 定義（Definition）

> **Open/Closed Principle（OCP）** 指的是：
>
> > **「軟體實體（如：類別、模組、函式）應對擴充開放，對修改封閉。」**
>
> 意即：
>
> - **可以透過擴展的方式來改變行為**
> - **但不要直接修改現有的程式碼**

這是 **SOLID** 原則中的第二個原則，是物件導向設計中提升可維護性與擴展性的核心思想。

---

## 🔍 第一原理拆解（First-Principles Breakdown）

| 元素         | 解釋                                                                   |
| ------------ | ---------------------------------------------------------------------- |
| **目標**     | 在不修改既有程式碼的前提下，允許系統行為的改變                         |
| **原因**     | 減少對現有穩定模組的影響，降低 bug、提高穩定性                         |
| **達成方法** | 使用抽象（interface / abstract class）與多型（polymorphism）來擴展功能 |
| **關鍵技術** | Interface, Inheritance, Delegation, Strategy Pattern                   |

---

## ❌ 錯誤示範（違反 OCP）

```csharp
public class DiscountCalculator
{
    public double CalculateDiscount(string customerType, double amount)
    {
        if (customerType == "Regular")
        {
            return amount * 0.9;
        }
        else if (customerType == "Premium")
        {
            return amount * 0.8;
        }
        else
        {
            return amount;
        }
    }
}
```

### 問題分析：

- 如果加入新的 `customerType`（如 VIP、Enterprise），就必須 **修改這段程式碼**。
- 違反 OCP，因為這不是「封閉對修改」的設計。

---

## ✅ 正確示範（符合 OCP）

### 1. 抽象策略介面：

```csharp
public interface IDiscountStrategy
{
    double ApplyDiscount(double amount);
}
```

### 2. 各種折扣策略（擴展不修改）：

```csharp
public class RegularDiscount : IDiscountStrategy
{
    public double ApplyDiscount(double amount) => amount * 0.9;
}

public class PremiumDiscount : IDiscountStrategy
{
    public double ApplyDiscount(double amount) => amount * 0.8;
}

public class NoDiscount : IDiscountStrategy
{
    public double ApplyDiscount(double amount) => amount;
}
```

### 3. 折扣計算器：

```csharp
public class DiscountCalculator
{
    private readonly IDiscountStrategy _discountStrategy;

    public DiscountCalculator(IDiscountStrategy discountStrategy)
    {
        _discountStrategy = discountStrategy;
    }

    public double Calculate(double amount)
    {
        return _discountStrategy.ApplyDiscount(amount);
    }
}
```

### 4. 使用方式：

```csharp
class Program
{
    static void Main()
    {
        IDiscountStrategy strategy = new PremiumDiscount(); // 可替換為 RegularDiscount, NoDiscount
        var calculator = new DiscountCalculator(strategy);

        double finalAmount = calculator.Calculate(1000);
        Console.WriteLine($"最終金額：{finalAmount}");
    }
}
```

---

## 🎯 達成 OCP 的效果：

| 項目        | 說明                                                             |
| ----------- | ---------------------------------------------------------------- |
| 🔄 擴展能力 | 新增 `EnterpriseDiscount` 僅需實作一個新類別，不需動到其他程式碼 |
| 🛡️ 安全性   | 不會破壞既有行為，降低風險                                       |
| 🧩 模組化   | 每個折扣策略是獨立模組，易於測試與替換                           |
| 💡 設計模式 | 使用了 **策略模式（Strategy Pattern）** 來實現 OCP               |

---

## 📚 延伸應用場景：

- 金流處理（不同支付方式）
- 訂單計算（不同促銷策略）
- 日誌系統（不同寫入方式：檔案、資料庫、雲端）
- 驗證邏輯（不同驗證方式：帳密、OTP、OAuth）

---

## 🧠 總結語：

| 屬性       | 價值                       |
| ---------- | -------------------------- |
| **開放性** | 可以新增功能               |
| **封閉性** | 不需動到穩定的程式碼       |
| **手段**   | 抽象化 + 多型性 + 設計模式 |

---

## 🔁 依賴反轉原則 DIP

> 高階模組不應該依賴低階模組，兩者應依賴抽象。

---

### ✅ 建構子注入（Constructor Injection）

```csharp
public interface ILogger
{
    void Log(string message);
}

public class FileLogger : ILogger
{
    public void Log(string message)
    {
        File.AppendAllText("log.txt", message + "\n");
    }
}

❌ //永遠只能用 FileLogger，以後有更進階好用的 Logger 也用不了。
public class UserService
{
    private readonly FileLogger _logger;

    public UserService(FileLogger logger)
    {
        _logger = logger;
    }

    public void CreateUser(string name)
    {
        _logger.Log($"使用者 {name} 建立成功");
    }
}

❌ // 這就變成高階的 UserService 依賴低階的 FileLogger
public class UserService
{
    private FileLogger _logger = new FileLogger();

    public void CreateUser(string name)
    {
        _logger.Log($"使用者 {name} 建立成功");
    }
}

✅ //依賴由注入來完成，且依賴的對象是抽象，以後各種 Logger 都可以被放進來用
public class UserService
{
    private readonly ILogger _logger;

    public UserService(ILogger logger)
    {
        _logger = logger;
    }

    public void CreateUser(string name)
    {
        _logger.Log($"使用者 {name} 建立成功");
    }
}
```

---

## 🧪 使用範例

```csharp
var logger = new FileLogger();
var service = new UserService(logger);
service.CreateUser("小明");
```

---

## 🔄 介面注入（Interface Injection）

### 實作範例

```csharp
public interface INotifier
{
    void Notify(string msg);
}

public class SmsNotifier : INotifier
{
    public void Notify(string msg) => Console.WriteLine($"SMS: {msg}");
}

public class NotificationService
{
    public void Send(INotifier notifier)
    {
        notifier.Notify("您有新通知");
    }
}
```

---

## 📝 練習題

### 題目 1：

實作一個 `ILogger` 介面與兩種實作類別（`ConsoleLogger`、`FileLogger`）

### 題目 2：

設計一個 `EmailService` 使用建構子注入 `ILogger`

---

## ✅ 解答參考

```csharp
public class ConsoleLogger : ILogger
{
    public void Log(string message) => Console.WriteLine($"[Console] {message}");
}

public class EmailService
{
    private readonly ILogger _logger;
    public EmailService(ILogger logger)
    {
        _logger = logger;
    }
}
```

---

## 📚 小結

- 低耦合的關鍵在於「依賴抽象」而非具體實作
- 使用注入模式可以讓程式模組更具可測試性與可維護性
- 單一職責原則能提升系統可維護性與擴充性

---

# 📘 第 6 章：泛型與集合（Generics & Collections）

## 🎯 教學目標

- 瞭解泛型的概念與用途
- 能定義泛型類別與泛型方法
- 熟悉 C# 內建泛型集合（List、Dictionary、Queue、Stack）
- 實作泛型資料結構與應用

---

## 📦 泛型概念（Generics）

### 問題背景

```csharp
public class IntBox
{
    public int Value { get; set; }
    public void Print()
    {
        Console.WriteLine(Value);
    }
}

public class StringBox
{
    public string Value { get; set; }
    public void Print()
    {
        Console.WriteLine(Value);
    }
}

public class BooleanBox
{
    public bool Value { get; set; }
    public void Print()
    {
        Console.WriteLine(Value);
    }
}
//......
//....
//.........
```

🔧 缺點：每種型別都需要一個類別 → 無法重用

---

### ✅ 泛型類別定義

```csharp
public class Box<T>
{
    public T Value { get; set; }

    public void Print()
    {
        Console.WriteLine(Value);
    }
}
```

### 使用：

```csharp
var intBox = new Box<int> { Value = 123 };
intBox.Print();

var stringBox = new Box<string> { Value = "Hello" };
stringBox.Print();
```

---

## 📦 泛型方法

```csharp
public class Printer
{
    public void Print<T>(T data)
    {
        Console.WriteLine(data);
    }
}
```

---

## 📚 常用泛型集合

### `List<T>`：可變長度陣列

```csharp
List<string> names = new List<string>();
names.Add("小明");
names.Add("小美");
```

### `Dictionary<K,V>`：鍵值對集合

```csharp
Dictionary<string, int> ages = new Dictionary<string, int>();
ages["小明"] = 20;
```

### `Queue<T>`：先進先出（FIFO）

```csharp
Queue<string> tasks = new Queue<string>();
tasks.Enqueue("任務A");
tasks.Dequeue(); // 出隊
```

### `Stack<T>`：後進先出（LIFO）

```csharp
Stack<string> history = new Stack<string>();
history.Push("頁面1");
history.Pop(); // 回到上一頁
```

---

## 🧪 實作：泛型 Stack

```csharp
public class MyStack<T>
{
    private List<T> _list = new List<T>();

    public void Push(T item) => _list.Add(item);
    public T Pop()
    {
        var item = _list[_list.Count - 1];
        _list.RemoveAt(_list.Count - 1);
        return item;
    }
}
```

---

## 📝 練習題

### 題目 1：

定義泛型類別 `Pair<T1, T2>`，可同時儲存兩種型別的值

### 題目 2：

使用 `Dictionary<string, List<int>>` 記錄每個學生的多次測驗成績

---

## ✅ 解答參考

```csharp
public class Pair<T1, T2>
{
    public T1 First { get; set; }
    public T2 Second { get; set; }
}
```

```csharp
Dictionary<string, List<int>> scores = new Dictionary<string, List<int>>();
scores["小明"] = new List<int> { 90, 85, 88 };
```

---

## 🧠 小結

| 類型              | 特性             |
| ----------------- | ---------------- |
| `List<T>`         | 動態陣列         |
| `Dictionary<K,V>` | 鍵值對查詢       |
| `Queue<T>`        | 排隊處理（FIFO） |
| `Stack<T>`        | 回朔堆疊（LIFO） |

泛型是撰寫可重複使用、型別安全程式碼的關鍵。

---

# 📘 第 7 章：進階泛型

- **泛型的核心思想：** 泛型允許我們在設計類別、介面和方法時，將「型別」本身作為一個參數。這讓我們可以編寫出 **型別安全 (Type-Safe)** 且可重用的程式碼，而無需為每種資料型別都重寫一次。
- **泛型解決的問題：** 在沒有泛型之前，若要寫一個可儲存任何型別的容器，通常使用 `object` 型別。但這樣做有兩個缺點：
  1.  **效能問題：** 值型別（如 `int`, `double`）存入時需要「裝箱」(Boxing)，取出時需要「拆箱」(Unboxing)，這會帶來額外的效能開銷。
  2.  **型別不安全：** 從容器取出 `object` 後，需要手動轉型回原始型別，一不小心就可能轉型失敗而導致執行時期錯誤。
- **泛型約束 (`where T : ...`)**
  - 有時我們需要對泛型參數 `T` 做一些限制，確保它具備某些能力。例如，我們可能需要 `T` 必須是一個類別、必須有預設建構函式，或必須實作某個介面。
  - 常用約束：`where T : class` (必須是參考型別), `where T : struct` (必須是實值型別), `where T : new()` (必須有公開無參數建構函式), `where T : SomeBaseClass`, `where T : ISomeInterface`。

```csharp
public class Box<T> where T : class
{
    public T Value { get; set; }
}

var b1 = new Box<int>();❌
var b2 = new Box<Dog>();✅

//

public class Box<T> where T : struct
{
    public T Value { get; set; }
}
var b3 = new Box<int>();✅
var b4 = new Box<Dog>();❌
```
