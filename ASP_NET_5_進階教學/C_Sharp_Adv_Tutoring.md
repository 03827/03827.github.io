# 🧠 C# 9 進階語法教學講義

## 一、前言：C# 9 的精神與設計方向

C# 9 於 .NET 5 時期問世，是微軟讓 C# 更加「資料導向（data-oriented）」與「不可變性（immutability）」的重要里程碑。
在這一版中，語言變得更簡潔、更具表達力，特別針對：

* **不可變資料（Immutable Data）**
* **模式匹配（Pattern Matching）**
* **語法簡化與可讀性提升**

這些新特性讓 C# 能更貼近現代程式設計風格，尤其在資料導向、函數式設計與 Domain-Driven Design（DDD）上更具威力。

---

## 二、Record 型別（Records）

### 2.1 基本概念

`record` 是一種專為 **不可變物件（immutable objects）** 而設計的型別。
相較於 `class`，`record` 會自動幫你實作：

* `Equals()` 與 `GetHashCode()`（基於值而非參考）
* `ToString()` 的友善輸出
* 支援 **複製（copy）** 與 **部分更新（with expression）**

### 2.2 基本語法
```csharp
class Person {
    string Name {get; init;}
    int Age {get; init;}
}

var p1 = new Person() { Name = "Alice", Age = 30, };
var p2 = new Person() { Name = "Alice", Age = 30, };

// 值相同即相等
Console.WriteLine(p1 == p2); // False
```

```csharp
public record Person(string Name, int Age);

var p1 = new Person("Alice", 30);
var p2 = new Person(Age: 30, Name: "Alice");

// 值相同即相等
Console.WriteLine(p1 == p2); // True
```

這裡 `==` 比較的是「值」而非「參考」。
換成一般 `class`，結果會是 `False`。

### 2.3 `with` 運算子（見第 3 節）

Record 天生支援「部分修改複製」：

```csharp
var p3 = p1 with { Age = 31 };
```

這會產生一個新物件，原本的 `p1` 不會被改變。

### 2.4 Record 的繼承

```csharp
public record Employee(string Name, int Age, string Department) 
    : Person(Name, Age);
```

子 Record 自動繼承父 Record 的結構與行為，等值比較同樣以值為基礎。

---

## 三、`with` 表達式

`with` 用於複製 Record 或 init-only 物件，僅修改部分屬性。

```csharp
var p1 = new Person("Alice", 30);
var p2 = p1 with { Name = "Bob" };

Console.WriteLine(p2); // Person { Name = Bob, Age = 30 }
```

### 重點：

* 產生新物件，不會修改原始資料。
* 適合搭配 Record 或不可變模型（immutable model）。

---

## 四、`init` 存取子（init-only properties）

### 4.1 為什麼需要 `init`？

在過去，我們只能用 `set` 讓屬性可寫入，但那也允許在任何時刻被修改。
`init` 允許屬性只在初始化時設定，之後不可再改。

### 4.2 範例

```csharp
public class User
{
    public string Name { get; init; }
    public int Age { get; init; }
    public readonly string phone = "091111111111";
}

var u1 = new User { Name = "Tom", Age = 20 };
// u1.Age = 30; // ❌ 錯誤，init 屬性無法在初始化後修改
```

### 4.3 手動設置 Record 欄位內容

```csharp
public record UserProfile
{
    public string Account { get; init; }
    public string Email
    {
        //由Account 組成 a03827@taisugar.com.tw
        get => $"a{Account}@taisugar.com.tw";
    }
}
```

---

## 五、解構賦值

在 C# 中，`Deconstruct` 是一個語法糖裝置，讓你能夠**把物件「拆解」成多個變數**，就像解構 tuple 一樣。
`record` 型別特別支援這項功能，因為它天生就是以「資料」為中心的型別。

### 範例：

任何型別（不只是 record）都可以定義一個 `Deconstruct` 方法：

```csharp
//定義一個 Record
public record PersonRecord(string Name, int Age, int Gender);

//使用方式：n is "03827", g is 0
var (n, _, g) = new PersonRecord("03827", 18, 0);
Console.Write($"姓名：{n}, 性別：{g}");

//定義一個含有 Deconstruct 的 Class
public class PersonClass
{
    public string Name { get; init; }
    public int Age { get; set; }
    public int Gender { get; set; }    

    //注意，name 在最後面
    public void Deconstruct(out int age, out int gender, out string name)
    {
        name = Name;
        age = Age;
        gender = Gender;
    }    
}

//使用方式：age is 16, gen is 1
var (myAge, gen, _) = new PersonClass()
{
    Name = "03827",
    Age = 16,
    Gender = 1,
};

```

---

## 六、Pattern Matching 強化（模式比對）

C# 9 擴充了模式比對的能力，使 `switch` 更具邏輯表達力。

### 6.1 關係運算模式（Relational patterns）

```csharp
//switch when 的使用
int number = 15;
switch (number)
{
    case int n when n < 0:
        Console.WriteLine("負數");
        break;
    case int n when n >= 0 && n <= 10:
        Console.WriteLine("0 到 10 之間");
        break;
    case int n when n > 10:
        Console.WriteLine("大於 10");
        break;
    default:
        Console.WriteLine("未知");
        break;
}
```

```csharp
//switch 簡化 + when 達成子條件：
int number = 55;
string judge = number switch
{
    < 0 => "負數",
    >= 0 when number <= 10 => "0 到 10 之間",
    > 10 and <= 20 => "11 到 20 之間",
    int num when num > 20 && num < 30 => "21 到 29 之間",
    >= 30 when number % 2 == 0 => "30 以上的偶數",
    >= 30 when number % 2 == 1 => "30 以上的奇數",
    _ => "正整數",
};

Console.WriteLine($"數字：{number}, 判斷：{judge}"); //數字：55, 判斷：30 以上的奇數
```

```csharp
//物件配合 switch when
var person = new Person("03827", 16, 0);           
switch (person)
{
    case Person p when p.Age < 18:
        Console.WriteLine("未成年");
        break;
    case Person p when p.Age >= 18 && p.Age <= 150:
        Console.WriteLine("成年人");
        break;
    case Person p when p.Age > 150:
        Console.WriteLine("不是人");
        break;
    default:
        Console.WriteLine("未知");
        break;
}

```

``` csharp
//switch 簡化：
var person = new Person(Name: "Bob", Age: 70);
string category = person switch
{
    { Age: < 18 } => "未成年",
    { Age: >= 18 and < 65 } => "成年人",
    { Age: >= 65 } when person.Age <= 150 => "長者",
    Person p when p.Age > 150 => "不是人",
    null => "無資料",
    _ => "未知",
};

Console.WriteLine($"{person.Name} 是 {category}");
```

``` csharp
//多欄位條件：
string category = person switch
{
    { Age: < 18 } when person.Name.Length < 3 => "未成年、名字短",
    { Age: < 18, Name: { Length: 3 } } => "未成年、名字長度一般",
    { Age: < 18, Name: { Length: >= 4 } } => "未成年、名字長",

    //以下C# 10才支援
    { Age: >= 18 and < 65, Name.Length: < 3 } => "成年人、名字短",
    { Age: >= 18 and < 65, Name.Length: 3 } => "成年人、名字長度一般",
    { Age: >= 18 and < 65, Name.Length: >= 4 } => "成年人、名字短",

    null => "無資料",
    _ => "未知",
};
```

### 6.2 邏輯運算模式（Logical patterns）

可用 `and`, `or`, `not` 組合條件：

```csharp
if (x is >= 0 and <= 100)
    Console.WriteLine("x is 0~100");

if (x is not (>= 200 or <= 100))
    Console.WriteLine("x 不符合");

if (y is not null)
    Console.WriteLine("Value exists");
```

### 6.3 巢狀模式

```csharp
if (person is { Age: > 18, Name: "Alice" })
    Console.WriteLine("Adult named Alice");
```

---

## 七、Target-Typed `new` 表達式

C# 9 允許右邊的 `new()` 省略型別，只要從左邊能推斷出。

```csharp
List<int> numbers = new List<int>(); //old way
var numbers = new List<int>();

List<int> numbers = new (); //new way
Point p = new(3, 4);
Dictionary<string, int> dict = new();
```

讓程式碼更簡潔，尤其在泛型與長型別名稱中極具可讀性。

---

## 八、Covariant Return Types（共變回傳型別）

覆寫方法時可回傳更具體的子型別：

```csharp
abstract class Shape
{
    public abstract Shape Clone();
}

class Circle : Shape
{
    public override Circle Clone() => new Circle();
}

class Triangle : Shape
{
    public override Circle Clone() => new Circle();
}
```

這樣 `Clone()` 回傳型別更準確，避免多餘的轉型。

---

## 九、Static Lambda（靜態匿名函式）

C# 9 允許匿名函式標記為 `static`，以避免捕獲外部變數。

```csharp
int extra = 10;
Func<int, int> square = x => x * x + extra; //extra 可以讀取
Func<int, int> square2 = static x => x * x + extra; //extra 「無法」讀取
```

這能減少記憶體分配與閉包（closure）開銷。
若匿名函式未存取外部變數，建議一律使用 `static`。

延伸
```csharp
void calc(int a, int b)
{
    int extra = 20;
    Func<int, int, int> sum = (a, b) => a + b + extra; //extra 可以讀取
    Console.WriteLine(sum(2, 4)); //2 + 4 + 20 = 26;

    //two in, one out
    Func<int, int, int> sum2 = static (a, b) => a + b + extra; //extra 無法讀取

    //two in, no out
    Action<int, int> sum3 = (a, b) => Console.WriteLine(a + b + extra);

    int _cal(int _a, int _b, int _c, Action<int, int> act)
    {
        bool isSomethingDone = true;
        if (isSomethingDone && act is not null)
        {
            //呼叫 act 這個函式，用以通知 Something Done
            act(9, 20); //_cal 不知道 act 會跑些什麼，只知道我丟了什麼值進去
        }

        return _a + _b + _c; //extra 可以讀取
    }

    //(myA, myB) => Console.WriteLine(myA - MyB)
    //上面這個匿名函式知道自己要做些什麼，但不知道參數值會是什麼，由呼叫我的人丟進來。
    _cal(3, 6, 9, (myA, myB) => Console.WriteLine(myA - myB));

    _cal(3, 6, 9, sum3);

    static int _cal2(int _a, int _b, int _c)
    {
        int ex = 5;
        return _a + _b + _c + ex + extra; //extra 無法讀取
    }

    Console.WriteLine(_cal(a, b, 6));
}

calc(2, 4);
```
---
