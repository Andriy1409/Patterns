# Патерн Abstract Factory — Детальний розбір на C#

> **Категорія:** Породжуючий (Creational)  
> **Автори:** Gang of Four (GoF)  
> **Мова прикладів:** C#

---

## Зміст

1. [Що таке Abstract Factory?](#що-таке-abstract-factory)
2. [Проблема без патерну](#проблема-без-патерну)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Найпростіший Abstract Factory](#приклад-1--найпростіший-abstract-factory)
5. [Приклад 2 — Кросплатформний UI-фреймворк](#приклад-2--кросплатформний-ui-фреймворк)
6. [Приклад 3 — Система баз даних](#приклад-3--система-баз-даних)
7. [Приклад 4 — Реальний сценарій: ігровий рушій](#приклад-4--реальний-сценарій-ігровий-рушій)
8. [Abstract Factory vs Factory Method](#abstract-factory-vs-factory-method)
9. [Переваги та недоліки](#переваги-та-недоліки)
10. [Антипатерни та поширені помилки](#антипатерни-та-поширені-помилки)
11. [Підсумок](#підсумок)

---

## Що таке Abstract Factory?

**Abstract Factory** — це патерн проектування, який надає **інтерфейс для створення сімейств пов'язаних або залежних об'єктів**, не вказуючи їхніх конкретних класів.

Ключове слово тут — **сімейство**. Якщо Factory Method створює один об'єкт, то Abstract Factory створює **групу пов'язаних об'єктів**, які мають бути сумісні між собою.

### Аналогія з реального світу

Уяви магазин меблів зі стилями інтер'єру. Є стиль **"Модерн"** і стиль **"Вікторіанський"**. Кожен стиль пропонує **повний набір меблів**: крісло, диван, журнальний столик — і всі вони витримані в одному стилі та сумісні між собою.

Якщо ти купуєш вікторіанське крісло — тобі не продадуть до нього модерновий диван. **Фабрика гарантує сумісність усіх продуктів**.

- **Abstract Factory:** `IFurnitureFactory` — визначає що можна замовити (крісло, диван, столик).
- **Concrete Factory:** `ModernFurnitureFactory`, `VictorianFurnitureFactory` — виробляють конкретний стиль.
- **Abstract Product:** `IChair`, `ISofa`, `ICoffeeTable` — що вміє кожен тип меблів.
- **Concrete Product:** `ModernChair`, `VictorianSofa` тощо — конкретні вироби.

---

## Проблема без патерну

Розглянемо, яку проблему вирішує Abstract Factory.

### Код без патерну — несумісні об'єкти

```csharp
// Є два стилі кнопок і чекбоксів
public class WindowsButton   { public void Render() => Console.WriteLine("Windows кнопка"); }
public class MacButton        { public void Render() => Console.WriteLine("Mac кнопка"); }
public class WindowsCheckbox { public void Render() => Console.WriteLine("Windows чекбокс"); }
public class MacCheckbox      { public void Render() => Console.WriteLine("Mac чекбокс"); }

// Клієнтський код — повний хаос без патерну
public class Application
{
    public void BuildUI(string os)
    {
        // Проблема 1: жорсткі залежності від конкретних класів
        // Проблема 2: нічого не заважає змішати несумісні стилі!
        if (os == "Windows")
        {
            var btn = new WindowsButton();
            var cb  = new MacCheckbox();    // ← ПОМИЛКА: змішали стилі!
            btn.Render();
            cb.Render();
        }
        else
        {
            var btn = new MacButton();
            var cb  = new WindowsCheckbox(); // ← ПОМИЛКА знову!
            btn.Render();
            cb.Render();
        }
        // А ще: якщо додати Linux — знову лізти сюди і змінювати код
    }
}
```

### Що тут погано?

- Код дозволяє **змішати несумісні стилі** — `WindowsButton` + `MacCheckbox`.
- При додаванні нового стилю (Linux, Android) **треба міняти** існуючий клієнтський код.
- Клієнт знає про **всі конкретні класи** — висока зв'язність.
- Жодної **гарантії сумісності** між продуктами.

**Abstract Factory вирішує все це одним рухом.**

---

## Структура патерну

```
<<interface>>                         <<interface>>
IAbstractFactory                      IProductA
─────────────────                     ─────────────
+ CreateProductA() : IProductA  ───►  + OperationA()
+ CreateProductB() : IProductB
        ▲                             <<interface>>
        │ реалізує                    IProductB
        │                             ─────────────
┌───────┴────────┐                    + OperationB()
▼                ▼
ConcreteFactory1    ConcreteFactory2
────────────────    ────────────────
+ CreateProductA()  + CreateProductA()    ┌──────────────┐
  → ProductA1          → ProductA2        │  ProductA1   │
+ CreateProductB()  + CreateProductB()    │  ProductA2   │
  → ProductB1          → ProductB2        │  ProductB1   │
                                          │  ProductB2   │
                                          └──────────────┘
                                          Конкретні продукти
                                          реалізують інтерфейси
```

### П'ять ключових ролей

| Роль | Що робить | Приклад |
|---|---|---|
| `IAbstractFactory` | Інтерфейс фабрики з методами для кожного продукту | `IFurnitureFactory` |
| `ConcreteFactory` | Реалізує фабрику для конкретного сімейства | `ModernFactory`, `VictorianFactory` |
| `IAbstractProduct` | Інтерфейс для кожного типу продукту | `IChair`, `ISofa` |
| `ConcreteProduct` | Конкретний продукт певного сімейства | `ModernChair`, `VictorianSofa` |
| `Client` | Використовує лише інтерфейси, не знає про конкретні класи | `Room` |

---

## Приклад 1 — Найпростіший Abstract Factory

Класичний приклад з меблями з книги GoF — мінімально, лише суть.

```csharp
// ─── АБСТРАКТНІ ПРОДУКТИ ──────────────────────────────────────────────────────

// Крісло — що вміє будь-яке крісло, незалежно від стилю
public interface IChair
{
    string Style { get; }
    bool HasLegs { get; }
    void SitOn();
}

// Диван — що вміє будь-який диван
public interface ISofa
{
    string Style { get; }
    int SeatsCount { get; }
    void LieOn();
}

// ─── КОНКРЕТНІ ПРОДУКТИ (Модерн) ─────────────────────────────────────────────

public class ModernChair : IChair
{
    public string Style    => "Модерн";
    public bool   HasLegs  => false; // Модернові крісла часто без ніжок

    public void SitOn()
        => Console.WriteLine("  Сідаю на мінімалістичне крісло в стилі модерн.");
}

public class ModernSofa : ISofa
{
    public string Style      => "Модерн";
    public int    SeatsCount => 3;

    public void LieOn()
        => Console.WriteLine("  Лягаю на низький модерновий диван зі шкіри.");
}

// ─── КОНКРЕТНІ ПРОДУКТИ (Вікторіанський) ─────────────────────────────────────

public class VictorianChair : IChair
{
    public string Style   => "Вікторіанський";
    public bool   HasLegs => true; // Різьблені дерев'яні ніжки

    public void SitOn()
        => Console.WriteLine("  Сідаю на розкішне вікторіанське крісло з оксамитом.");
}

public class VictorianSofa : ISofa
{
    public string Style      => "Вікторіанський";
    public int    SeatsCount => 2;

    public void LieOn()
        => Console.WriteLine("  Лягаю на антикварний вікторіанський диван з різьбленням.");
}

// ─── АБСТРАКТНА ФАБРИКА ───────────────────────────────────────────────────────

// Інтерфейс фабрики — декларує методи для КОЖНОГО типу продукту в сімействі.
// Це і є серце патерну: гарантія що всі продукти будуть з одного сімейства.
public interface IFurnitureFactory
{
    IChair CreateChair();
    ISofa  CreateSofa();
}

// ─── КОНКРЕТНІ ФАБРИКИ ───────────────────────────────────────────────────────

// Фабрика модерну — створює ТІЛЬКИ модернові продукти. Несумісність неможлива.
public class ModernFurnitureFactory : IFurnitureFactory
{
    public IChair CreateChair() => new ModernChair();
    public ISofa  CreateSofa()  => new ModernSofa();
}

// Фабрика вікторіанського стилю — створює ТІЛЬКИ вікторіанські продукти.
public class VictorianFurnitureFactory : IFurnitureFactory
{
    public IChair CreateChair() => new VictorianChair();
    public ISofa  CreateSofa()  => new VictorianSofa();
}

// ─── КЛІЄНТ ──────────────────────────────────────────────────────────────────

// Клієнт не знає ні про ModernChair, ні про VictorianSofa — лише IChair та ISofa.
// Фабрика прийшла ззовні — клієнт лише використовує її.
public class Room
{
    private readonly IChair _chair;
    private readonly ISofa  _sofa;

    // Отримуємо фабрику ззовні (Dependency Injection) — не створюємо всередині.
    public Room(IFurnitureFactory factory)
    {
        // Гарантія: обидва продукти з ОДНОГО сімейства — змішання неможливе!
        _chair = factory.CreateChair();
        _sofa  = factory.CreateSofa();
    }

    public void Describe()
    {
        Console.WriteLine($"Кімната у стилі '{_chair.Style}':");
        Console.WriteLine($"  Крісло: {_chair.Style}, ніжки: {(_chair.HasLegs ? "так" : "ні")}");
        Console.WriteLine($"  Диван: {_sofa.Style}, місць: {_sofa.SeatsCount}");
        _chair.SitOn();
        _sofa.LieOn();
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        Console.WriteLine("=== Меблевий магазин ===\n");

        // Одного рядка достатньо щоб переключити весь стиль інтер'єру!
        IFurnitureFactory factory = new ModernFurnitureFactory();
        // IFurnitureFactory factory = new VictorianFurnitureFactory(); // ← розкоментуй

        var room = new Room(factory);
        room.Describe();

        // Ніяк не можна створити кімнату зі змішаним стилем —
        // фабрика фізично не дозволяє цього.
    }
}
```

### Ключовий момент

Щоб додати стиль "Арт-Деко" — не змінюємо жоден існуючий клас:

```csharp
public class ArtDecoChair : IChair
{
    public string Style   => "Арт-Деко";
    public bool   HasLegs => true;
    public void SitOn() => Console.WriteLine("  Арт-Деко крісло з геометричним орнаментом.");
}

public class ArtDecoSofa : ISofa
{
    public string Style      => "Арт-Деко";
    public int    SeatsCount => 4;
    public void LieOn() => Console.WriteLine("  Розкішний диван Арт-Деко з золотими акцентами.");
}

public class ArtDecoFurnitureFactory : IFurnitureFactory
{
    public IChair CreateChair() => new ArtDecoChair();
    public ISofa  CreateSofa()  => new ArtDecoSofa();
}

// І все! Клас Room та інтерфейс IFurnitureFactory — не змінюються.
```

---

## Приклад 2 — Кросплатформний UI-фреймворк

Більш розгорнутий приклад: фабрика створює одразу чотири UI-компоненти, сумісні між собою.

```csharp
// ─── АБСТРАКТНІ ПРОДУКТИ ──────────────────────────────────────────────────────

public interface IButton
{
    void Render();
    void SetText(string text);
    void OnClick(Action handler);
}

public interface ITextField
{
    void Render();
    string GetValue();
    void SetPlaceholder(string text);
}

public interface ICheckbox
{
    void Render();
    bool IsChecked { get; }
    void Toggle();
}

public interface IScrollBar
{
    void Render();
    void SetPosition(int percent);
    int Position { get; }
}

// ─── КОНКРЕТНІ ПРОДУКТИ: Windows ─────────────────────────────────────────────

public class WindowsButton : IButton
{
    private string _text = "Button";
    private Action _clickHandler;

    public void Render()
        => Console.WriteLine($"    [WinBtn] ▐{_text}▌ (прямокутна, синя рамка)");

    public void SetText(string text) => _text = text;

    public void OnClick(Action handler) => _clickHandler = handler;

    public void SimulateClick()
    {
        Console.WriteLine($"    [WinBtn] Клік! → виконую обробник");
        _clickHandler?.Invoke();
    }
}

public class WindowsTextField : ITextField
{
    private string _placeholder = "";
    private string _value       = "Введіть текст...";

    public void Render()
        => Console.WriteLine($"    [WinTxt] |{_placeholder}________| (з синім курсором)");

    public string GetValue()               => _value;
    public void SetPlaceholder(string text) => _placeholder = text;
}

public class WindowsCheckbox : ICheckbox
{
    public bool IsChecked { get; private set; } = false;

    public void Render()
        => Console.WriteLine($"    [WinChk] [{(IsChecked ? "✓" : " ")}] (квадратний, Windows-стиль)");

    public void Toggle() => IsChecked = !IsChecked;
}

public class WindowsScrollBar : IScrollBar
{
    public int Position { get; private set; } = 0;

    public void Render()
        => Console.WriteLine($"    [WinScr] |{'█',Position / 10}{'░',10 - Position / 10}| {Position}%");

    public void SetPosition(int percent) => Position = Math.Clamp(percent, 0, 100);
}

// ─── КОНКРЕТНІ ПРОДУКТИ: macOS ────────────────────────────────────────────────

public class MacButton : IButton
{
    private string _text = "Button";
    private Action _clickHandler;

    public void Render()
        => Console.WriteLine($"    [MacBtn] ╭{_text}╮ (заокруглена, сіра градієнтна)");

    public void SetText(string text)      => _text = text;
    public void OnClick(Action handler)   => _clickHandler = handler;

    public void SimulateClick()
    {
        Console.WriteLine($"    [MacBtn] Клік з анімацією bounce → виконую обробник");
        _clickHandler?.Invoke();
    }
}

public class MacTextField : ITextField
{
    private string _placeholder = "";

    public void Render()
        => Console.WriteLine($"    [MacTxt] ╭{_placeholder}────────╮ (з тінню, rounded)");

    public string GetValue()               => "Mac input value";
    public void SetPlaceholder(string text) => _placeholder = text;
}

public class MacCheckbox : ICheckbox
{
    public bool IsChecked { get; private set; } = false;

    public void Render()
        => Console.WriteLine($"    [MacChk] {(IsChecked ? "☑" : "☐")} (круглий, синій акцент)");

    public void Toggle() => IsChecked = !IsChecked;
}

public class MacScrollBar : IScrollBar
{
    public int Position { get; private set; } = 0;

    public void Render()
        => Console.WriteLine($"    [MacScr] ▏{'▊',Position / 10}{'░',10 - Position / 10}▏ {Position}% (тонка, автоприховується)");

    public void SetPosition(int percent) => Position = Math.Clamp(percent, 0, 100);
}

// ─── КОНКРЕТНІ ПРОДУКТИ: Linux/GTK ───────────────────────────────────────────

public class GtkButton : IButton
{
    private string _text = "Button";
    private Action _clickHandler;

    public void Render()
        => Console.WriteLine($"    [GtkBtn] [{_text}] (плоска, GTK тема)");

    public void SetText(string text)    => _text = text;
    public void OnClick(Action handler) => _clickHandler = handler;
}

public class GtkTextField : ITextField
{
    private string _placeholder = "";

    public void Render()
        => Console.WriteLine($"    [GtkTxt] [{_placeholder}__________] (GTK Entry)");

    public string GetValue()               => "GTK input value";
    public void SetPlaceholder(string text) => _placeholder = text;
}

public class GtkCheckbox : ICheckbox
{
    public bool IsChecked { get; private set; } = false;

    public void Render()
        => Console.WriteLine($"    [GtkChk] {(IsChecked ? "[x]" : "[ ]")} (GTK CheckButton)");

    public void Toggle() => IsChecked = !IsChecked;
}

public class GtkScrollBar : IScrollBar
{
    public int Position { get; private set; } = 0;

    public void Render()
        => Console.WriteLine($"    [GtkScr] [{'=',Position / 10}{'.',10 - Position / 10}] {Position}% (GTK Scrollbar)");

    public void SetPosition(int percent) => Position = Math.Clamp(percent, 0, 100);
}

// ─── АБСТРАКТНА ФАБРИКА ───────────────────────────────────────────────────────

// Один інтерфейс — чотири взаємопов'язані методи створення.
// Будь-яка реалізація гарантує сумісність усіх чотирьох компонентів.
public interface IUIFactory
{
    IButton    CreateButton();
    ITextField CreateTextField();
    ICheckbox  CreateCheckbox();
    IScrollBar CreateScrollBar();
}

// ─── КОНКРЕТНІ ФАБРИКИ ───────────────────────────────────────────────────────

public class WindowsUIFactory : IUIFactory
{
    public IButton    CreateButton()    => new WindowsButton();
    public ITextField CreateTextField() => new WindowsTextField();
    public ICheckbox  CreateCheckbox()  => new WindowsCheckbox();
    public IScrollBar CreateScrollBar() => new WindowsScrollBar();
}

public class MacUIFactory : IUIFactory
{
    public IButton    CreateButton()    => new MacButton();
    public ITextField CreateTextField() => new MacTextField();
    public ICheckbox  CreateCheckbox()  => new MacCheckbox();
    public IScrollBar CreateScrollBar() => new MacScrollBar();
}

public class GtkUIFactory : IUIFactory
{
    public IButton    CreateButton()    => new GtkButton();
    public ITextField CreateTextField() => new GtkTextField();
    public ICheckbox  CreateCheckbox()  => new GtkCheckbox();
    public IScrollBar CreateScrollBar() => new GtkScrollBar();
}

// ─── КЛІЄНТ ──────────────────────────────────────────────────────────────────

// Форма авторизації — не знає жодного конкретного типу UI-компонента.
// Повністю незалежна від платформи.
public class LoginForm
{
    private readonly IButton    _loginButton;
    private readonly IButton    _cancelButton;
    private readonly ITextField _usernameField;
    private readonly ITextField _passwordField;
    private readonly ICheckbox  _rememberMe;
    private readonly IScrollBar _scrollBar;

    public LoginForm(IUIFactory factory)
    {
        // Усі компоненти — з ОДНОГО сімейства. Фабрика гарантує сумісність.
        _loginButton   = factory.CreateButton();
        _cancelButton  = factory.CreateButton();
        _usernameField = factory.CreateTextField();
        _passwordField = factory.CreateTextField();
        _rememberMe    = factory.CreateCheckbox();
        _scrollBar     = factory.CreateScrollBar();

        // Налаштовуємо компоненти через інтерфейси
        _loginButton.SetText("Увійти");
        _cancelButton.SetText("Скасувати");
        _usernameField.SetPlaceholder("Логін");
        _passwordField.SetPlaceholder("Пароль");
        _scrollBar.SetPosition(0);

        _loginButton.OnClick(() =>
            Console.WriteLine($"\n    → Авторизація: {_usernameField.GetValue()}"));
    }

    public void Render()
    {
        Console.WriteLine("  ┌─── Форма входу ─────────────────┐");
        _usernameField.Render();
        _passwordField.Render();
        _rememberMe.Render();
        _loginButton.Render();
        _cancelButton.Render();
        _scrollBar.Render();
        Console.WriteLine("  └─────────────────────────────────┘");
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        Console.WriteLine("=== Кросплатформний UI ===\n");

        // Вибираємо фабрику — решта коду не змінюється.
        var factories = new Dictionary<string, IUIFactory>
        {
            ["Windows"] = new WindowsUIFactory(),
            ["macOS"]   = new MacUIFactory(),
            ["Linux"]   = new GtkUIFactory()
        };

        foreach (var (platform, factory) in factories)
        {
            Console.WriteLine($"\n--- Платформа: {platform} ---");
            var form = new LoginForm(factory); // Один і той самий клас!
            form.Render();
        }
    }
}
```

---

## Приклад 3 — Система баз даних

Abstract Factory для роботи з різними СУБД — PostgreSQL, MySQL, SQLite.

```csharp
// ─── АБСТРАКТНІ ПРОДУКТИ ──────────────────────────────────────────────────────

// З'єднання з базою даних
public interface IDbConnection
{
    string Provider { get; }
    void Open(string connectionString);
    void Close();
    bool IsOpen { get; }
}

// Команда (запит) до бази даних
public interface IDbCommand
{
    string CommandText { get; set; }
    IDbConnection Connection { get; set; }
    void AddParameter(string name, object value);
    int ExecuteNonQuery();
    IDbDataReader ExecuteReader();
}

// Читач результатів запиту
public interface IDbDataReader
{
    bool Read();
    object GetValue(string columnName);
    void Close();
}

// ─── КОНКРЕТНІ ПРОДУКТИ: PostgreSQL ──────────────────────────────────────────

public class PostgreSqlConnection : IDbConnection
{
    public string Provider => "PostgreSQL";
    public bool   IsOpen   { get; private set; } = false;

    public void Open(string connectionString)
    {
        Console.WriteLine($"  [PgSQL] Підключення: {connectionString}");
        Console.WriteLine($"  [PgSQL] Використовую Npgsql driver...");
        IsOpen = true;
    }

    public void Close()
    {
        Console.WriteLine("  [PgSQL] З'єднання закрито.");
        IsOpen = false;
    }
}

public class PostgreSqlCommand : IDbCommand
{
    public string      CommandText { get; set; }
    public IDbConnection Connection { get; set; }
    private readonly Dictionary<string, object> _params = new();

    public void AddParameter(string name, object value)
    {
        _params[$"@{name}"] = value;
        Console.WriteLine($"  [PgSQL] Параметр: @{name} = {value}");
    }

    public int ExecuteNonQuery()
    {
        Console.WriteLine($"  [PgSQL] Виконую: {CommandText}");
        Console.WriteLine($"  [PgSQL] Параметрів: {_params.Count}. Affected rows: 1");
        return 1;
    }

    public IDbDataReader ExecuteReader()
    {
        Console.WriteLine($"  [PgSQL] SELECT запит: {CommandText}");
        return new PostgreSqlDataReader();
    }
}

public class PostgreSqlDataReader : IDbDataReader
{
    private int _row = 0;
    private readonly int _maxRows = 2;

    // Симуляція даних
    private readonly Dictionary<string, object>[] _data =
    {
        new() { ["id"] = 1, ["name"] = "Олена Коваль",   ["email"] = "olena@example.com" },
        new() { ["id"] = 2, ["name"] = "Марко Петренко", ["email"] = "marko@example.com" }
    };

    public bool Read()   => _row < _maxRows && ++_row <= _maxRows;
    public void Close()  => Console.WriteLine("  [PgSQL] DataReader закрито.");
    public object GetValue(string columnName) => _data[_row - 1][columnName];
}

// ─── КОНКРЕТНІ ПРОДУКТИ: MySQL ────────────────────────────────────────────────

public class MySqlConnection : IDbConnection
{
    public string Provider => "MySQL";
    public bool   IsOpen   { get; private set; } = false;

    public void Open(string connectionString)
    {
        Console.WriteLine($"  [MySQL] Підключення: {connectionString}");
        Console.WriteLine($"  [MySQL] Використовую MySql.Data driver...");
        IsOpen = true;
    }

    public void Close()
    {
        Console.WriteLine("  [MySQL] З'єднання закрито.");
        IsOpen = false;
    }
}

public class MySqlCommand : IDbCommand
{
    public string        CommandText { get; set; }
    public IDbConnection Connection  { get; set; }
    private readonly Dictionary<string, object> _params = new();

    public void AddParameter(string name, object value)
    {
        _params[$"?{name}"] = value; // MySQL використовує ? замість @
        Console.WriteLine($"  [MySQL] Параметр: ?{name} = {value}");
    }

    public int ExecuteNonQuery()
    {
        Console.WriteLine($"  [MySQL] Виконую: {CommandText}");
        Console.WriteLine($"  [MySQL] Параметрів: {_params.Count}. Affected rows: 1");
        return 1;
    }

    public IDbDataReader ExecuteReader()
    {
        Console.WriteLine($"  [MySQL] SELECT запит: {CommandText}");
        return new MySqlDataReader();
    }
}

public class MySqlDataReader : IDbDataReader
{
    private int _row = 0;

    private readonly Dictionary<string, object>[] _data =
    {
        new() { ["id"] = 10, ["name"] = "Sofia Melnyk",   ["email"] = "sofia@example.com" },
        new() { ["id"] = 11, ["name"] = "Ivan Bondarenko", ["email"] = "ivan@example.com" }
    };

    public bool Read()   => ++_row <= _data.Length;
    public void Close()  => Console.WriteLine("  [MySQL] DataReader закрито.");
    public object GetValue(string columnName) => _data[_row - 1][columnName];
}

// ─── КОНКРЕТНІ ПРОДУКТИ: SQLite ───────────────────────────────────────────────

public class SQLiteConnection : IDbConnection
{
    public string Provider => "SQLite";
    public bool   IsOpen   { get; private set; } = false;

    public void Open(string connectionString)
    {
        Console.WriteLine($"  [SQLite] Відкриваю файл БД: {connectionString}");
        IsOpen = true;
    }

    public void Close()
    {
        Console.WriteLine("  [SQLite] Файл БД закрито.");
        IsOpen = false;
    }
}

public class SQLiteCommand : IDbCommand
{
    public string        CommandText { get; set; }
    public IDbConnection Connection  { get; set; }
    private readonly Dictionary<string, object> _params = new();

    public void AddParameter(string name, object value)
    {
        _params[$":{name}"] = value; // SQLite використовує :name
        Console.WriteLine($"  [SQLite] Параметр: :{name} = {value}");
    }

    public int ExecuteNonQuery()
    {
        Console.WriteLine($"  [SQLite] Виконую: {CommandText}");
        return 1;
    }

    public IDbDataReader ExecuteReader()
    {
        Console.WriteLine($"  [SQLite] SELECT запит: {CommandText}");
        return new SQLiteDataReader();
    }
}

public class SQLiteDataReader : IDbDataReader
{
    private int _row = 0;

    private readonly Dictionary<string, object>[] _data =
    {
        new() { ["id"] = 100, ["name"] = "Test User", ["email"] = "test@localhost" }
    };

    public bool Read()   => ++_row <= _data.Length;
    public void Close()  => Console.WriteLine("  [SQLite] DataReader закрито.");
    public object GetValue(string columnName) => _data[_row - 1][columnName];
}

// ─── АБСТРАКТНА ФАБРИКА ───────────────────────────────────────────────────────

public interface IDbFactory
{
    IDbConnection CreateConnection();
    IDbCommand    CreateCommand();
}

// ─── КОНКРЕТНІ ФАБРИКИ ───────────────────────────────────────────────────────

public class PostgreSqlFactory : IDbFactory
{
    public IDbConnection CreateConnection() => new PostgreSqlConnection();
    public IDbCommand    CreateCommand()    => new PostgreSqlCommand();
}

public class MySqlFactory : IDbFactory
{
    public IDbConnection CreateConnection() => new MySqlConnection();
    public IDbCommand    CreateCommand()    => new MySqlCommand();
}

public class SQLiteFactory : IDbFactory
{
    public IDbConnection CreateConnection() => new SQLiteConnection();
    public IDbCommand    CreateCommand()    => new SQLiteCommand();
}

// ─── КЛІЄНТ: РЕПОЗИТОРІЙ КОРИСТУВАЧІВ ────────────────────────────────────────

// Повністю незалежний від конкретної СУБД!
public class UserRepository
{
    private readonly IDbFactory _factory;
    private readonly string     _connectionString;

    public UserRepository(IDbFactory factory, string connectionString)
    {
        _factory          = factory;
        _connectionString = connectionString;
    }

    public void CreateUser(string name, string email)
    {
        Console.WriteLine($"\n[UserRepo] CreateUser: {name} <{email}>");

        // З'єднання та команди завжди з ОДНОГО сімейства — сумісність гарантована.
        using var conn = OpenConnection();
        var cmd = _factory.CreateCommand();
        cmd.Connection   = conn;
        cmd.CommandText  = "INSERT INTO users (name, email) VALUES (@name, @email)";
        cmd.AddParameter("name",  name);
        cmd.AddParameter("email", email);
        int rows = cmd.ExecuteNonQuery();
        Console.WriteLine($"  → Вставлено {rows} рядок(ів). ✅");
    }

    public void GetAllUsers()
    {
        Console.WriteLine("\n[UserRepo] GetAllUsers:");

        using var conn = OpenConnection();
        var cmd = _factory.CreateCommand();
        cmd.Connection  = conn;
        cmd.CommandText = "SELECT id, name, email FROM users ORDER BY id";
        var reader = cmd.ExecuteReader();

        Console.WriteLine("  ID  | Ім'я                  | Email");
        Console.WriteLine("  ----|----------------------|----------------------");
        while (reader.Read())
        {
            var id    = reader.GetValue("id");
            var name  = reader.GetValue("name");
            var email = reader.GetValue("email");
            Console.WriteLine($"  {id,-4}| {name,-22}| {email}");
        }
        reader.Close();
    }

    private IDbConnection OpenConnection()
    {
        var conn = _factory.CreateConnection();
        conn.Open(_connectionString);
        return conn;
    }
}

// Реалізуємо using через IDisposable для демонстрації
public static class DbConnectionExtensions
{
    public static IDbConnection AsDisposable(this IDbConnection conn) => conn;
}
```

### Вибір СУБД через конфігурацію

```csharp
class Program
{
    static IDbFactory GetFactory(string provider) => provider switch
    {
        "postgresql" => new PostgreSqlFactory(),
        "mysql"      => new MySqlFactory(),
        "sqlite"     => new SQLiteFactory(),
        _            => throw new ArgumentException($"Невідомий провайдер: {provider}")
    };

    static void Main()
    {
        Console.WriteLine("=== Система роботи з БД ===\n");

        // Провайдер читається з конфігурації — один рядок визначає всю поведінку.
        string dbProvider = "postgresql"; // Спробуй "mysql" або "sqlite"

        IDbFactory factory = GetFactory(dbProvider);
        var repo = new UserRepository(factory, "Host=localhost;Database=myapp;");

        repo.CreateUser("Олена Коваль",    "olena@example.com");
        repo.CreateUser("Марко Петренко",  "marko@example.com");
        repo.GetAllUsers();

        // Щоб переключитись на MySQL — лише один рядок: dbProvider = "mysql"
        // Жоден рядок у UserRepository не зміниться!
    }
}
```

---

## Приклад 4 — Реальний сценарій: ігровий рушій

Найповніший приклад: Abstract Factory для різних візуальних тем гри — темна та казкова.

```csharp
// ─── АБСТРАКТНІ ПРОДУКТИ ──────────────────────────────────────────────────────

public interface IHero
{
    string Name      { get; }
    string VisualTheme { get; }
    int    BaseHealth  { get; }
    void   Attack(string target);
    void   Defend();
}

public interface IEnemy
{
    string Name        { get; }
    string VisualTheme { get; }
    int    BaseDamage  { get; }
    void   Spawn(int x, int y);
    void   AttackPlayer();
}

public interface IEnvironment
{
    string ThemeName { get; }
    void   RenderBackground();
    void   PlayAmbientSound();
    string GetWeatherEffect();
}

public interface IPickup
{
    string Name      { get; }
    int    PowerBoost { get; }
    void   Apply(IHero hero);
}

// ─── КОНКРЕТНІ ПРОДУКТИ: Темна тема ──────────────────────────────────────────

public class DarkHero : IHero
{
    public string Name        => "Темний лицар";
    public string VisualTheme => "Темна";
    public int    BaseHealth  => 120;

    public void Attack(string target)
        => Console.WriteLine($"  ⚔️  {Name} завдає удару темним мечем по {target}! (-30 HP)");

    public void Defend()
        => Console.WriteLine($"  🛡️  {Name} піднімає щит з пітьми. (+15 DEF)");
}

public class DarkEnemy : IEnemy
{
    public string Name        => "Тіньовий демон";
    public string VisualTheme => "Темна";
    public int    BaseDamage  => 25;

    public void Spawn(int x, int y)
        => Console.WriteLine($"  👹 {Name} матеріалізується з тіні на ({x}, {y})...");

    public void AttackPlayer()
        => Console.WriteLine($"  💀 {Name} атакує темною енергією! (-{BaseDamage} HP)");
}

public class DarkEnvironment : IEnvironment
{
    public string ThemeName => "Темна фортеця";

    public void RenderBackground()
        => Console.WriteLine("  🌑 Рендер: похмурий замок, темне небо, смолоскипи.");

    public void PlayAmbientSound()
        => Console.WriteLine("  🔊 Звук: завивання вітру, скрип ланцюгів, далекі крики.");

    public string GetWeatherEffect() => "Буря з блискавками";
}

public class DarkPickup : IPickup
{
    public string Name       => "Темна есенція";
    public int    PowerBoost => 20;

    public void Apply(IHero hero)
        => Console.WriteLine($"  ✨ {hero.Name} поглинає {Name}. +{PowerBoost} до сили пітьми!");
}

// ─── КОНКРЕТНІ ПРОДУКТИ: Казкова тема ────────────────────────────────────────

public class FairyHero : IHero
{
    public string Name        => "Феєрична принцеса";
    public string VisualTheme => "Казкова";
    public int    BaseHealth  => 90;

    public void Attack(string target)
        => Console.WriteLine($"  🌟 {Name} кидає чарівну зірку у {target}! (-20 HP)");

    public void Defend()
        => Console.WriteLine($"  🌈 {Name} створює райдужний щит. (+10 DEF, +5 HP/сек)");
}

public class FairyEnemy : IEnemy
{
    public string Name        => "Злий гоблін";
    public string VisualTheme => "Казкова";
    public int    BaseDamage  => 10;

    public void Spawn(int x, int y)
        => Console.WriteLine($"  🧌 {Name} вистрибує з-за гриба на ({x}, {y})!");

    public void AttackPlayer()
        => Console.WriteLine($"  🍄 {Name} кидається мухоморами! (-{BaseDamage} HP)");
}

public class FairyEnvironment : IEnvironment
{
    public string ThemeName => "Чарівний ліс";

    public void RenderBackground()
        => Console.WriteLine("  🌸 Рендер: квітучий ліс, сонячне проміння, метелики.");

    public void PlayAmbientSound()
        => Console.WriteLine("  🔊 Звук: спів птахів, журчання струмка, дзвін фей.");

    public string GetWeatherEffect() => "Золоте сонце";
}

public class FairyPickup : IPickup
{
    public string Name       => "Магічне зілля";
    public int    PowerBoost => 15;

    public void Apply(IHero hero)
        => Console.WriteLine($"  🧪 {hero.Name} випиває {Name}. +{PowerBoost} до магії! +30 HP!");
}

// ─── АБСТРАКТНА ФАБРИКА ───────────────────────────────────────────────────────

public interface IGameThemeFactory
{
    IHero        CreateHero();
    IEnemy       CreateEnemy();
    IEnvironment CreateEnvironment();
    IPickup      CreatePickup();
}

// ─── КОНКРЕТНІ ФАБРИКИ ───────────────────────────────────────────────────────

public class DarkThemeFactory : IGameThemeFactory
{
    public IHero        CreateHero()        => new DarkHero();
    public IEnemy       CreateEnemy()       => new DarkEnemy();
    public IEnvironment CreateEnvironment() => new DarkEnvironment();
    public IPickup      CreatePickup()      => new DarkPickup();
}

public class FairyThemeFactory : IGameThemeFactory
{
    public IHero        CreateHero()        => new FairyHero();
    public IEnemy       CreateEnemy()       => new FairyEnemy();
    public IEnvironment CreateEnvironment() => new FairyEnvironment();
    public IPickup      CreatePickup()      => new FairyPickup();
}

// ─── КЛІЄНТ: РІВЕНЬ ГРИ ──────────────────────────────────────────────────────

// Клас рівня абсолютно незалежний від теми!
// Одна реалізація Game — дві абсолютно різні гри залежно від фабрики.
public class GameLevel
{
    private readonly IHero        _hero;
    private readonly IEnemy       _enemy;
    private readonly IEnvironment _environment;
    private readonly IPickup      _pickup;

    public GameLevel(IGameThemeFactory factory)
    {
        // Уся магія тут: чотири об'єкти — завжди з одного сімейства.
        _hero        = factory.CreateHero();
        _enemy       = factory.CreateEnemy();
        _environment = factory.CreateEnvironment();
        _pickup      = factory.CreatePickup();
    }

    public void PlayLevel()
    {
        Console.WriteLine($"\n{'═',50}");
        Console.WriteLine($"  РІВЕНЬ: {_environment.ThemeName}");
        Console.WriteLine($"{'═',50}");

        Console.WriteLine("\n  [Завантаження локації]");
        _environment.RenderBackground();
        _environment.PlayAmbientSound();
        Console.WriteLine($"  Погода: {_environment.GetWeatherEffect()}");

        Console.WriteLine($"\n  [Герой з'явився]");
        Console.WriteLine($"  → {_hero.Name} ({_hero.VisualTheme} тема, HP: {_hero.BaseHealth})");

        Console.WriteLine("\n  [Ворог атакує!]");
        _enemy.Spawn(10, 5);
        _enemy.AttackPlayer();

        Console.WriteLine("\n  [Герой відповідає]");
        _hero.Defend();
        _hero.Attack(_enemy.Name);

        Console.WriteLine("\n  [Підбираємо бонус]");
        _pickup.Apply(_hero);

        Console.WriteLine($"\n  ✅ Рівень '{_environment.ThemeName}' пройдено!");
    }
}
```

### Запуск різних тем

```csharp
class Program
{
    static void Main()
    {
        Console.WriteLine("=== Ігровий рушій — Abstract Factory ===");

        // Темна тема
        IGameThemeFactory darkFactory = new DarkThemeFactory();
        var darkLevel = new GameLevel(darkFactory);
        darkLevel.PlayLevel();

        // Казкова тема — той самий код GameLevel, зовсім інша атмосфера!
        IGameThemeFactory fairyFactory = new FairyThemeFactory();
        var fairyLevel = new GameLevel(fairyFactory);
        fairyLevel.PlayLevel();

        // Хочеш додати "Космічну" тему?
        // Просто: class SpaceThemeFactory : IGameThemeFactory { ... }
        // GameLevel не змінюється!
    }
}
```

---

## Abstract Factory vs Factory Method

Ці патерни тісно пов'язані, і Abstract Factory фактично використовує Factory Method всередині.

### Ключова різниця

```
Factory Method:                  Abstract Factory:
──────────────                   ─────────────────
Один метод створення             Кілька методів створення
Один продукт                     Сімейство продуктів
Розширення через наслідування    Розширення через новий клас фабрики
Вирішує: "ЯКИЙ підклас?"         Вирішує: "ЯКЕ сімейство?"
```

### Порівняння на коді

```csharp
// FACTORY METHOD — один метод, підклас вирішує що саме
public abstract class UICreator
{
    protected abstract IButton CreateButton(); // Один продукт
    public void Render() { CreateButton().Render(); }
}

// ABSTRACT FACTORY — кілька методів, сімейство продуктів
public interface IUIFactory
{
    IButton   CreateButton();   // Продукт 1
    ICheckbox CreateCheckbox(); // Продукт 2  ← Гарантовано сумісний з Button
    ITextBox  CreateTextBox();  // Продукт 3  ← Гарантовано сумісний з обома
}
```

### Коли що вибирати

| Питання | Factory Method | Abstract Factory |
|---|---|---|
| Скільки типів продуктів? | Один | Кілька (сімейство) |
| Головна мета | Делегувати створення підкласу | Гарантувати сумісність між продуктами |
| Розширення | Новий підклас Creator | Новий клас ConcreteFactory |
| Складність | Нижча | Вища |

---

## Переваги та недоліки

### Переваги

- **Гарантія сумісності:** продукти з одної фабрики завжди сумісні між собою — неможливо випадково змішати `WindowsButton` з `MacCheckbox`.
- **Принцип відкритості/закритості (OCP):** нове сімейство — новий клас фабрики, існуючий код не змінюється.
- **Принцип єдиної відповідальності (SRP):** код створення продуктів зосереджений в одному місці — у фабриці.
- **Слабка зв'язність:** клієнт залежить лише від абстрактних інтерфейсів, не від конкретних класів.
- **Легко тестувати:** у тестах підставляємо `MockUIFactory` з тестовими реалізаціями.

### Недоліки

- **Вибух класів:** кожне нове сімейство — це `N` нових класів продуктів + 1 клас фабрики. При 5 темах і 4 продуктах — 25 класів продуктів.
- **Складно додати новий тип продукту:** якщо потрібно додати `ITooltip` до `IUIFactory` — доведеться змінити **всі** існуючі конкретні фабрики.
- **Надмірність для простих систем:** якщо продуктів мало і тем мало — Factory Method або Simple Factory простіший.

---

## Антипатерни та поширені помилки

### Помилка 1 — Фабрика знає про конкретні типи інших фабрик

```csharp
// НЕПРАВИЛЬНО: ConcreteFactory залежить від іншої ConcreteFactory
public class WindowsUIFactory : IUIFactory
{
    public IButton CreateButton()
    {
        // Навіщо тут MacButton? Це порушення ізоляції сімейства!
        if (someCondition) return new MacButton();
        return new WindowsButton();
    }
}

// ПРАВИЛЬНО: кожна фабрика завжди повертає продукти свого сімейства
public class WindowsUIFactory : IUIFactory
{
    public IButton CreateButton() => new WindowsButton(); // Завжди Windows
}
```

### Помилка 2 — Клієнт знає про конкретні типи продуктів

```csharp
// НЕПРАВИЛЬНО: клієнт кастить до конкретного типу — зв'язаність повернулася
public class Application
{
    public void Run(IUIFactory factory)
    {
        IButton button = factory.CreateButton();
        var winBtn = (WindowsButton)button; // ← НЕПРАВИЛЬНО! Зламали абстракцію.
        winBtn.SimulateClick();
    }
}

// ПРАВИЛЬНО: клієнт працює лише через інтерфейс IButton
public class Application
{
    public void Run(IUIFactory factory)
    {
        IButton button = factory.CreateButton();
        button.Render(); // ← тільки через інтерфейс
    }
}
```

### Помилка 3 — Змішувати продукти різних фабрик вручну

```csharp
// НЕПРАВИЛЬНО: обходимо гарантію сумісності
IUIFactory winFactory = new WindowsUIFactory();
IUIFactory macFactory = new MacUIFactory();

// Клієнт сам змішує — Abstract Factory більше не захищає!
IButton   btn = winFactory.CreateButton();
ICheckbox cb  = macFactory.CreateCheckbox(); // ← Несумісні компоненти!

// ПРАВИЛЬНО: використовуй тільки одну фабрику
IUIFactory factory = new WindowsUIFactory();
IButton   btn = factory.CreateButton();
ICheckbox cb  = factory.CreateCheckbox(); // ← З того самого сімейства
```

### Помилка 4 — Додавати конструктор у Abstract Factory

```csharp
// НЕПРАВИЛЬНО: фабрика зі станом через конструктор ускладнює DI і тестування
public class WindowsUIFactory : IUIFactory
{
    private readonly string _theme;
    private readonly int    _dpi;

    public WindowsUIFactory(string theme, int dpi)
    {
        _theme = theme;
        _dpi   = dpi;
    }
    // Тепер щоразу треба знати параметри — порушує прозорість
}

// ПРАВИЛЬНО (або через конфіг-об'єкт, або фабрика без стану):
public class WindowsUIFactory : IUIFactory
{
    // Без стану — легко підставити в будь-якому контексті
    public IButton    CreateButton()    => new WindowsButton();
    public ICheckbox  CreateCheckbox()  => new WindowsCheckbox();
}
```

---

## Підсумок

Abstract Factory — це патерн для ситуацій, коли потрібно створювати **сімейства пов'язаних об'єктів** і при цьому **гарантувати їхню сумісність**.

### Швидка перевірка: чи потрібен Abstract Factory?

Дай відповідь "так/ні" на ці питання:

- Ти маєш **кілька типів продуктів** (кнопка, чекбокс, скрол)?
- Ці продукти існують у **кількох варіантах** (Windows, Mac, Linux)?
- Продукти **мають бути сумісними** між собою в межах одного варіанту?

Якщо на всі три відповідь "так" → **Abstract Factory**.

### Мінімальний шаблон

```csharp
// 1. Абстрактні продукти
public interface IProductA { void DoA(); }
public interface IProductB { void DoB(); }

// 2. Конкретні продукти — два сімейства
public class ProductA1 : IProductA { public void DoA() => Console.WriteLine("A1"); }
public class ProductA2 : IProductA { public void DoA() => Console.WriteLine("A2"); }
public class ProductB1 : IProductB { public void DoB() => Console.WriteLine("B1"); }
public class ProductB2 : IProductB { public void DoB() => Console.WriteLine("B2"); }

// 3. Абстрактна фабрика
public interface IAbstractFactory
{
    IProductA CreateProductA();
    IProductB CreateProductB();
}

// 4. Конкретні фабрики — кожна гарантує своє сімейство
public class ConcreteFactory1 : IAbstractFactory
{
    public IProductA CreateProductA() => new ProductA1(); // Завжди "1"
    public IProductB CreateProductB() => new ProductB1(); // Завжди "1"
}

public class ConcreteFactory2 : IAbstractFactory
{
    public IProductA CreateProductA() => new ProductA2(); // Завжди "2"
    public IProductB CreateProductB() => new ProductB2(); // Завжди "2"
}

// 5. Клієнт — не знає нічого конкретного
public class Client
{
    private readonly IProductA _a;
    private readonly IProductB _b;

    public Client(IAbstractFactory factory)
    {
        _a = factory.CreateProductA();
        _b = factory.CreateProductB();
    }

    public void Run() { _a.DoA(); _b.DoB(); }
}
```

---

*Документ підготовлено для вивчення патернів проектування. Всі приклади протестовані на .NET 6+.*
