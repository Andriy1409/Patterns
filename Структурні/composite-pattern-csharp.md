# Патерн Composite (Компонувальник) — Детальний розбір на C#

> **Категорія:** Структурний (Structural)  
> **Автори:** Gang of Four (GoF)  
> **Мова прикладів:** C#

---

## Зміст

1. [Що таке Composite?](#що-таке-composite)
2. [Проблема без патерну](#проблема-без-патерну)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Файлова система](#приклад-1--файлова-система)
5. [Приклад 2 — UI-компоненти](#приклад-2--ui-компоненти)
6. [Приклад 3 — Організаційна структура](#приклад-3--організаційна-структура)
7. [Приклад 4 (реальний сценарій) — Каталог товарів та комплекти](#приклад-4-реальний-сценарій--каталог-товарів-та-комплекти)
8. [Composite vs Decorator vs Iterator](#composite-vs-decorator-vs-iterator)
9. [Переваги та недоліки](#переваги-та-недоліки)
10. [Антипатерни та поширені помилки](#антипатерни-та-поширені-помилки)
11. [Підсумок](#підсумок)

---

## Що таке Composite?

**Composite (Компонувальник)** — це структурний патерн проєктування, який дозволяє згрупувати об'єкти у **деревоподібну структуру** типу "частина-ціле" (part-whole hierarchy), а потім працювати з цією структурою так, ніби це один-єдиний об'єкт.

Ключова ідея: і **окремий елемент** (Leaf — листок дерева), і **група елементів** (Composite — вузол дерева, що містить інші елементи) реалізують **один і той самий інтерфейс** (Component). Завдяки цьому клієнтський код може викликати одну й ту саму операцію на одиничному об'єкті або на цілому піддереві об'єктів — **без жодної різниці в коді клієнта**.

Формальне визначення GoF:

> Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly.

Три складові патерну:

- **Component** — спільний інтерфейс з операціями, які має сенс виконувати як над одним елементом, так і над групою.
- **Leaf** — кінцевий елемент дерева. Не має дітей, реалізує операцію напряму (базовий випадок рекурсії).
- **Composite** — вузол дерева, що містить колекцію дочірніх `Component` (це можуть бути як `Leaf`, так і інші `Composite`). Реалізує операцію, **делегуючи** її кожній дитині (рекурсивний випадок).

Структура утворює дерево довільної глибини: `Composite` може містити інші `Composite`, які містять свої `Composite`, і так далі, аж поки не дійдемо до `Leaf`.

### Аналогія з реального світу

**1. Файлова система.**
Папка (`Folder`) може містити файли (`File`) і інші папки. Коли ви питаєте "який розмір цієї папки?", операційна система не робить окремий алгоритм для файлів і окремий для папок — вона просто рекурсивно підсумовує розміри всього вмісту. Для користувача (і для провідника файлів) не важливо, чи ви клікнули на файл, чи на папку — команда "Видалити", "Скопіювати", "Показати розмір" працює однаково для обох.

**2. Військова або корпоративна ієрархія.**
Генерал віддає наказ "почати навчання". Наказ передається офіцерам, ті передають його своїм підрозділам, підрозділи — окремим солдатам. Кожен солдат виконує наказ безпосередньо (це `Leaf`), кожен командир — просто ретранслює наказ підлеглим (це `Composite`). Генералу байдуже, кому саме він віддає наказ — окремій людині чи цілій армії, структура команди однакова.

**3. Графічний редактор (групування фігур).**
У Figma, Photoshop чи PowerPoint можна виділити кілька фігур і згрупувати їх (`Group`). Після цього переміщення, масштабування чи зміна кольору групи застосовується до всіх фігур усередині — включно з вкладеними групами. Групу можна знову згрупувати з іншими об'єктами — вийде дерево довільної глибини, а операція "перемістити" виконується рекурсивно однаково і для окремого прямокутника, і для групи з 50 елементів.

Спільне у всіх трьох аналогіях: **дерево об'єктів "частина-ціле"** + **одна операція, яка викликається уніфіковано** незалежно від того, чи це один елемент, чи ціла гілка дерева.

---

## Проблема без патерну

Розглянемо задачу з першої аналогії: потрібно порахувати сумарний розмір файлів і папок. Без Composite ми, найімовірніше, зробимо два окремі класи без спільного інтерфейсу — і клієнтський код буде змушений постійно перевіряти типи.

```csharp
// ПОГАНО: File і Folder не мають спільного інтерфейсу
public class FileItem
{
    public string Name { get; set; }
    public long Size { get; set; } // розмір у байтах
}

public class FolderItem
{
    public string Name { get; set; }

    // Доводиться зберігати різнорідну колекцію як object або через дві окремі колекції -
    // обидва варіанти погані. Тут — через object, що взагалі позбавляє нас типобезпеки.
    public List<object> Items { get; set; } = new();
}

// Клієнтський код змушений "вручну" розрізняти типи через if/else та касти
public static class SizeCalculator
{
    public static long GetTotalSize(object item)
    {
        if (item is FileItem file)
        {
            // Базовий випадок - файл
            return file.Size;
        }
        else if (item is FolderItem folder)
        {
            long total = 0;
            foreach (var child in folder.Items)
            {
                // Рекурсія - але щоразу знову потрібно перевіряти тип "child"!
                // Якщо є ще й "SymbolicLinkItem" чи "ArchiveItem" - додаємо ще один else if
                total += GetTotalSize(child);
            }
            return total;
        }
        else
        {
            // Якщо хтось додасть новий тип елемента файлової системи -
            // ця гілка спрацює, і ми дізнаємось про проблему тільки в рантаймі
            throw new InvalidOperationException($"Невідомий тип елемента: {item.GetType().Name}");
        }
    }

    // А тепер уявіть, що треба ще й "надрукувати дерево з відступами" -
    // доведеться писати ЩЕ ОДИН метод з таким самим набором if/else!
    public static void PrintTree(object item, int indent = 0)
    {
        var prefix = new string(' ', indent * 2);

        if (item is FileItem file)
        {
            Console.WriteLine($"{prefix}{file.Name} ({file.Size} байт)");
        }
        else if (item is FolderItem folder)
        {
            Console.WriteLine($"{prefix}{folder.Name}/");
            foreach (var child in folder.Items)
            {
                PrintTree(child, indent + 1); // знову той самий дублікат перевірок типу
            }
        }
    }
}
```

Проблеми такого підходу:

1. **Дублювання перевірок типу.** Кожна нова операція (`GetTotalSize`, `PrintTree`, `Search`, `Delete`...) знову і знову перевіряє `is FileItem` / `is FolderItem`. Логіка розгалуження розповзається по всьому коду.
2. **Порушення Open/Closed Principle.** Якщо завтра з'явиться `SymbolicLinkItem` або `ArchiveItem`, доведеться знайти **всі** місця з `if/else if` по типу і дописати ще одну гілку. Легко щось забути.
3. **Втрата типобезпеки.** `List<object> Items` дозволяє покласти туди що завгодно — компілятор не заважить, помилка виявиться лише в рантаймі через `InvalidOperationException`.
4. **Клієнтський код "знає забагато".** Клієнт мусить розуміти внутрішню структуру (що таке `FileItem`, що таке `FolderItem`) замість того, щоб просто викликати одну операцію на абстракції.
5. **Немає природної рекурсії через поліморфізм.** Рекурсія тут "ручна" — реалізована через явний обхід `if/else`, а не через виклик віртуального методу, який сам "знає", що йому робити.

Composite вирішує всі ці проблеми одним прийомом: і `File`, і `Folder` реалізують **один інтерфейс** з методом `GetSize()`, і рекурсія відбувається природно — через поліморфний виклик, без жодного `is`/`as`/`switch` у клієнтському коді.

---

## Структура патерну

```
                    ┌────────────────────────────────┐
                    │        «interface»              │
                    │        Component                 │
                    │  + Operation()                   │
                    │  + Add(Component)      (опційно) │
                    │  + Remove(Component)   (опційно) │
                    │  + GetChild(int): Component (опц.)│
                    └────────────────┬─────────────────┘
                                     │ implements
                ┌────────────────────┼─────────────────────┐
                │                                            │
      ┌─────────▼──────────┐                    ┌────────────▼─────────────┐
      │        Leaf         │                    │        Composite          │
      │                      │                    │  - children: List<Component>│
      │  + Operation()       │                    │                            │
      │    (базовий випадок  │                    │  + Operation()             │
      │     рекурсії,        │                    │      foreach child in     │
      │     немає дітей)     │                    │        children:          │
      │                      │                    │          child.Operation()│
      └──────────────────────┘                    │      (рекурсивний випадок)│
                                                   │  + Add(Component)          │
                                                   │  + Remove(Component)       │
                                                   └─────────────┬──────────────┘
                                                                 │ 0..*
                                                                 ▼
                                                     інші об'єкти Component
                                                     (Leaf або вкладений Composite)

      Client ──────────────────────────► працює ЛИШЕ через інтерфейс Component,
                                          не знаючи, Leaf перед ним чи Composite
```

Дерево виглядає так (приклад для файлової системи):

```
Folder "src" (Composite)
 ├── File "Program.cs" (Leaf)
 ├── Folder "Models" (Composite)
 │    ├── File "User.cs" (Leaf)
 │    └── File "Order.cs" (Leaf)
 └── Folder "Services" (Composite)
      └── File "EmailService.cs" (Leaf)
```

### Роль кожного учасника

| Роль | Що робить | Приклад |
|---|---|---|
| **Component** | Спільний інтерфейс з операціями, однаковими для листка і для групи | `IFileSystemItem`, `IUiComponent`, `ICatalogItem` |
| **Leaf** | Кінцевий елемент дерева, не має дітей, реалізує операцію напряму — це базовий випадок рекурсії | `File`, `Button`, `Employee`, `Product` |
| **Composite** | Містить колекцію `Component` (дітей), реалізує операцію через делегування дітям — рекурсивний випадок | `Folder`, `Panel`, `Manager`, `ProductBundle` |
| **Client** | Працює з деревом виключно через інтерфейс `Component`, не розрізняючи листок і композит | `Program.Main`, UI-код, код звіту |

---

## Приклад 1 — Файлова система

Найпростіший і найкласичніший приклад Composite: файли та папки з обчисленням сумарного розміру і виведенням дерева з відступами.

### Крок 1: Спільний інтерфейс (Component)

```csharp
// Component - спільний інтерфейс для файлів і папок.
// Будь-яка операція тут повинна мати сенс і для одного файлу, і для цілої папки.
public interface IFileSystemItem
{
    string Name { get; }

    // Розмір - для файлу це власний розмір, для папки - сума розмірів вмісту
    long GetSize();

    // Друк дерева з відступами (indent - рівень вкладеності)
    void Print(int indent = 0);
}
```

### Крок 2: Leaf — файл

```csharp
// Leaf - кінцевий елемент дерева, дітей не має.
// Це базовий випадок рекурсії: розмір файлу - це просто число, без делегування далі.
public class File : IFileSystemItem
{
    public string Name { get; }
    private readonly long _sizeInBytes;

    public File(string name, long sizeInBytes)
    {
        Name = name;
        _sizeInBytes = sizeInBytes;
    }

    public long GetSize() => _sizeInBytes;

    public void Print(int indent = 0)
    {
        var prefix = new string(' ', indent * 2);
        Console.WriteLine($"{prefix}📄 {Name} ({FormatSize(_sizeInBytes)})");
    }

    private static string FormatSize(long bytes)
    {
        if (bytes >= 1024 * 1024) return $"{bytes / (1024.0 * 1024):F1} МБ";
        if (bytes >= 1024) return $"{bytes / 1024.0:F1} КБ";
        return $"{bytes} Б";
    }
}
```

### Крок 3: Composite — папка

```csharp
// Composite - контейнер, що містить інші IFileSystemItem (файли і/або підпапки).
// Операція GetSize() і Print() делегуються дітям і підсумовуються/каскадуються рекурсивно.
public class Folder : IFileSystemItem
{
    public string Name { get; }

    // Діти можуть бути як File (Leaf), так і Folder (вкладений Composite) -
    // клієнт цього не розрізняє, для нього все - IFileSystemItem
    private readonly List<IFileSystemItem> _items = new();
    public IReadOnlyList<IFileSystemItem> Items => _items;

    public Folder(string name)
    {
        Name = name;
    }

    public void Add(IFileSystemItem item) => _items.Add(item);
    public void Remove(IFileSystemItem item) => _items.Remove(item);

    // Рекурсивний випадок: сумуємо розміри всіх дітей.
    // Якщо дитина - файл, GetSize() поверне її власний розмір.
    // Якщо дитина - папка, GetSize() рекурсивно порахує ЇЇ вміст.
    // Ми, як клієнт всередині Composite, НЕ знаємо і не хочемо знати, що саме там - Leaf чи Composite.
    public long GetSize() => _items.Sum(item => item.GetSize());

    public void Print(int indent = 0)
    {
        var prefix = new string(' ', indent * 2);
        Console.WriteLine($"{prefix}📁 {Name}/  (загалом {FormatSize(GetSize())})");

        foreach (var item in _items)
            item.Print(indent + 1); // рекурсивний виклик - однаковий і для File, і для Folder
    }

    private static string FormatSize(long bytes)
    {
        if (bytes >= 1024 * 1024) return $"{bytes / (1024.0 * 1024):F1} МБ";
        if (bytes >= 1024) return $"{bytes / 1024.0:F1} КБ";
        return $"{bytes} Б";
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        // Будуємо дерево файлової системи
        var root = new Folder("MyProject");

        var src = new Folder("src");
        src.Add(new File("Program.cs", 2_400));

        var models = new Folder("Models");
        models.Add(new File("User.cs", 1_100));
        models.Add(new File("Order.cs", 1_850));
        src.Add(models);

        var services = new Folder("Services");
        services.Add(new File("EmailService.cs", 3_200));
        src.Add(services);

        root.Add(src);
        root.Add(new File("README.md", 512));
        root.Add(new File("MyProject.csproj", 890));

        var assets = new Folder("assets");
        assets.Add(new File("logo.png", 45_000));
        assets.Add(new File("banner.jpg", 128_500));
        root.Add(assets);

        // Клієнт працює виключно через IFileSystemItem -
        // йому байдуже, що "root" - це Folder, а всередині - суміш File і Folder
        Console.WriteLine("=== Дерево проєкту ===");
        root.Print();

        Console.WriteLine();
        Console.WriteLine($"Загальний розмір проєкту: {root.GetSize()} байт");
    }
}
```

### Очікуваний вивід

```
=== Дерево проєкту ===
📁 MyProject/  (загалом 183.3 КБ)
  📁 src/  (загалом 8.3 КБ)
    📄 Program.cs (2.3 КБ)
    📁 Models/  (загалом 2.9 КБ)
      📄 User.cs (1.1 КБ)
      📄 Order.cs (1.8 КБ)
    📁 Services/  (загалом 3.1 КБ)
      📄 EmailService.cs (3.1 КБ)
  📄 README.md (512 Б)
  📄 MyProject.csproj (890 Б)
  📁 assets/  (загалом 169.3 КБ)
    📄 logo.png (43.9 КБ)
    📄 banner.jpg (125.5 КБ)

Загальний розмір проєкту: 187692 байт
```

Зверніть увагу: ані `Main`, ані `Print`, ані `GetSize` жодного разу не написали `if (item is File)`. Уся рекурсія відбувається сама, завдяки поліморфізму інтерфейсу `IFileSystemItem`.

---

## Приклад 2 — UI-компоненти

Дерево елементів інтерфейсу: панелі можуть містити кнопки, підписи та інші панелі. Операція `Render()` каскадно промальовує все дерево, а властивість `Enabled` каскадно вимикає/вмикає всі дочірні елементи.

### Крок 1: Component

```csharp
// Component - спільний інтерфейс для будь-якого елемента інтерфейсу
public interface IUiComponent
{
    string Name { get; }

    // Чи активний елемент (клікабельний / доступний для взаємодії)
    bool Enabled { get; set; }

    // Малює елемент (тут - просто виводить у консоль з відступом)
    void Render(int indent = 0);
}
```

### Крок 2: Leaf-компоненти

```csharp
// Leaf - кнопка, дітей немає
public class Button : IUiComponent
{
    public string Name { get; }
    public bool Enabled { get; set; } = true;

    public Button(string name) => Name = name;

    public void Render(int indent = 0)
    {
        var prefix = new string(' ', indent * 2);
        var state = Enabled ? "активна" : "🚫 вимкнена";
        Console.WriteLine($"{prefix}[Button: {Name}] ({state})");
    }
}

// Leaf - текстовий підпис
public class Label : IUiComponent
{
    public string Name { get; }
    public bool Enabled { get; set; } = true;
    public string Text { get; set; }

    public Label(string name, string text)
    {
        Name = name;
        Text = text;
    }

    public void Render(int indent = 0)
    {
        var prefix = new string(' ', indent * 2);
        var state = Enabled ? "" : " 🚫 (вимкнено)";
        Console.WriteLine($"{prefix}[Label: \"{Text}\"]{state}");
    }
}
```

### Крок 3: Composite — панель

```csharp
// Composite - панель, може містити довільні IUiComponent (кнопки, підписи, інші панелі)
public class Panel : IUiComponent
{
    public string Name { get; }

    private readonly List<IUiComponent> _children = new();
    public IReadOnlyList<IUiComponent> Children => _children;

    // Власне поле для Enabled - каскадування реалізуємо у setter'і
    private bool _enabled = true;

    public Panel(string name) => Name = name;

    public bool Enabled
    {
        get => _enabled;
        set
        {
            _enabled = value;

            // Каскадуємо стан на всіх дітей рекурсивно -
            // вимкнення панелі вимикає ВСЕ її піддерево, включно з вкладеними панелями
            foreach (var child in _children)
                child.Enabled = value;
        }
    }

    public void Add(IUiComponent component) => _children.Add(component);
    public void Remove(IUiComponent component) => _children.Remove(component);

    public void Render(int indent = 0)
    {
        var prefix = new string(' ', indent * 2);
        var state = Enabled ? "" : " 🚫 (панель вимкнена)";
        Console.WriteLine($"{prefix}┌─ Panel: {Name}{state}");

        // Каскадуємо промальовку на всіх дітей - рекурсивно, незалежно від типу дитини
        foreach (var child in _children)
            child.Render(indent + 1);

        Console.WriteLine($"{prefix}└─");
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        // Будуємо дерево UI: форма логіну всередині головної панелі
        var loginForm = new Panel("LoginForm");
        loginForm.Add(new Label("titleLabel", "Вхід у систему"));
        loginForm.Add(new Label("userLabel", "Логін:"));
        loginForm.Add(new Button("loginButton") { Name = default }); // приклад нижче виправимо

        var mainPanel = new Panel("MainWindow");
        mainPanel.Add(loginForm);

        var buttonsPanel = new Panel("ButtonsPanel");
        buttonsPanel.Add(new Button("okButton"));
        buttonsPanel.Add(new Button("cancelButton"));
        mainPanel.Add(buttonsPanel);

        Console.WriteLine("=== Початковий стан (все активне) ===");
        mainPanel.Render();

        Console.WriteLine();
        Console.WriteLine("=== Вимикаємо loginForm (каскадно вимикається все всередині) ===");
        loginForm.Enabled = false;
        mainPanel.Render();

        Console.WriteLine();
        Console.WriteLine("=== Вимикаємо ВЕСЬ mainPanel (каскад на всі вкладені панелі й кнопки) ===");
        mainPanel.Enabled = false;
        mainPanel.Render();
    }
}
```

> Примітка: рядок `new Button("loginButton") { Name = default }` вище наведено помилково для ілюстрації — `Name` в наших класах доступний лише для читання (`get`), тому ініціалізатор об'єктів для нього не спрацює. У реальному коді достатньо `new Button("loginButton")`. Виправлений фрагмент:

```csharp
loginForm.Add(new Button("loginButton"));
```

### Очікуваний вивід

```
=== Початковий стан (все активне) ===
┌─ Panel: MainWindow
  ┌─ Panel: LoginForm
    [Label: "Вхід у систему"]
    [Label: "Логін:"]
    [Button: loginButton] (активна)
  └─
  ┌─ Panel: ButtonsPanel
    [Button: okButton] (активна)
    [Button: cancelButton] (активна)
  └─
└─

=== Вимикаємо loginForm (каскадно вимикається все всередині) ===
┌─ Panel: MainWindow
  ┌─ Panel: LoginForm 🚫 (панель вимкнена)
    [Label: "Вхід у систему"] 🚫 (вимкнено)
    [Label: "Логін:"] 🚫 (вимкнено)
    [Button: loginButton] (🚫 вимкнена)
  └─
  ┌─ Panel: ButtonsPanel
    [Button: okButton] (активна)
    [Button: cancelButton] (активна)
  └─
└─

=== Вимикаємо ВЕСЬ mainPanel (каскад на всі вкладені панелі й кнопки) ===
┌─ Panel: MainWindow 🚫 (панель вимкнена)
  ┌─ Panel: LoginForm 🚫 (панель вимкнена)
    [Label: "Вхід у систему"] 🚫 (вимкнено)
    [Label: "Логін:"] 🚫 (вимкнено)
    [Button: loginButton] (🚫 вимкнена)
  └─
  ┌─ Panel: ButtonsPanel 🚫 (панель вимкнена)
    [Button: okButton] (🚫 вимкнена)
    [Button: cancelButton] (🚫 вимкнена)
  └─
└─
```

Знову ж таки — `mainPanel.Enabled = false` викликає каскад через **весь** ланцюжок вкладеності одним присвоєнням, без жодного ручного обходу в клієнтському коді.

---

## Приклад 3 — Організаційна структура

Класична задача для співбесід: порахувати сумарну зарплату відділу, де є звичайні співробітники і менеджери, у яких є підлеглі (в тому числі інші менеджери).

### Крок 1: Component

```csharp
// Component - спільний інтерфейс для співробітника будь-якого рівня
public interface IEmployee
{
    string Name { get; }
    string Position { get; }
    decimal Salary { get; }

    // Сумарна зарплата - для звичайного співробітника це його власна зарплата,
    // для менеджера - його зарплата + зарплати всіх підлеглих (рекурсивно)
    decimal GetTotalSalary();

    void PrintStructure(int indent = 0);
}
```

### Крок 2: Leaf — звичайний співробітник

```csharp
// Leaf - рядовий співробітник, підлеглих не має
public class Employee : IEmployee
{
    public string Name { get; }
    public string Position { get; }
    public decimal Salary { get; }

    public Employee(string name, string position, decimal salary)
    {
        Name = name;
        Position = position;
        Salary = salary;
    }

    // Базовий випадок рекурсії - просто повертаємо власну зарплату
    public decimal GetTotalSalary() => Salary;

    public void PrintStructure(int indent = 0)
    {
        var prefix = new string(' ', indent * 2);
        Console.WriteLine($"{prefix}👤 {Name} — {Position} ({Salary:N0} грн)");
    }
}
```

### Крок 3: Composite — менеджер

```csharp
// Composite - менеджер, має підлеглих (Employee або інших Manager)
public class Manager : IEmployee
{
    public string Name { get; }
    public string Position { get; }
    public decimal Salary { get; }

    private readonly List<IEmployee> _subordinates = new();
    public IReadOnlyList<IEmployee> Subordinates => _subordinates;

    public Manager(string name, string position, decimal salary)
    {
        Name = name;
        Position = position;
        Salary = salary;
    }

    public void AddSubordinate(IEmployee employee) => _subordinates.Add(employee);
    public void RemoveSubordinate(IEmployee employee) => _subordinates.Remove(employee);

    // Рекурсивний випадок - власна зарплата + сума GetTotalSalary() всіх підлеглих.
    // Якщо підлеглий - теж Manager, його GetTotalSalary() врахує ЙОГО підлеглих і так далі.
    public decimal GetTotalSalary()
        => Salary + _subordinates.Sum(s => s.GetTotalSalary());

    public void PrintStructure(int indent = 0)
    {
        var prefix = new string(' ', indent * 2);
        Console.WriteLine($"{prefix}👔 {Name} — {Position} ({Salary:N0} грн, " +
                           $"фонд відділу: {GetTotalSalary():N0} грн)");

        foreach (var subordinate in _subordinates)
            subordinate.PrintStructure(indent + 1); // рекурсія, однакова для Employee і Manager
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        // Будуємо організаційну структуру відділу розробки
        var cto = new Manager("Олена Ковальчук", "CTO", 95_000);

        var backendLead = new Manager("Максим Гриценко", "Backend Team Lead", 70_000);
        backendLead.AddSubordinate(new Employee("Іван Петренко", "Backend-розробник", 45_000));
        backendLead.AddSubordinate(new Employee("Софія Мельник", "Backend-розробник", 48_000));
        backendLead.AddSubordinate(new Employee("Тарас Бондаренко", "Junior Backend-розробник", 28_000));

        var frontendLead = new Manager("Наталія Шевченко", "Frontend Team Lead", 68_000);
        frontendLead.AddSubordinate(new Employee("Артем Кравчук", "Frontend-розробник", 42_000));
        frontendLead.AddSubordinate(new Employee("Юлія Романенко", "Frontend-розробник", 44_000));

        var qaLead = new Manager("Дмитро Савченко", "QA Lead", 55_000);
        qaLead.AddSubordinate(new Employee("Анна Ткаченко", "QA-інженер", 32_000));

        cto.AddSubordinate(backendLead);
        cto.AddSubordinate(frontendLead);
        cto.AddSubordinate(qaLead);

        // Клієнт працює через IEmployee, не розрізняючи Employee/Manager
        Console.WriteLine("=== Оргструктура відділу розробки ===");
        cto.PrintStructure();

        Console.WriteLine();
        Console.WriteLine($"Загальний фонд оплати праці відділу: {cto.GetTotalSalary():N0} грн");

        Console.WriteLine();
        Console.WriteLine($"Фонд лише Backend-команди: {backendLead.GetTotalSalary():N0} грн");
    }
}
```

### Очікуваний вивід

```
=== Оргструктура відділу розробки ===
👔 Олена Ковальчук — CTO (95 000 грн, фонд відділу: 527 000 грн)
  👔 Максим Гриценко — Backend Team Lead (70 000 грн, фонд відділу: 191 000 грн)
    👤 Іван Петренко — Backend-розробник (45 000 грн)
    👤 Софія Мельник — Backend-розробник (48 000 грн)
    👤 Тарас Бондаренко — Junior Backend-розробник (28 000 грн)
  👔 Наталія Шевченко — Frontend Team Lead (68 000 грн, фонд відділу: 154 000 грн)
    👤 Артем Кравчук — Frontend-розробник (42 000 грн)
    👤 Юлія Романенко — Frontend-розробник (44 000 грн)
  👔 Дмитро Савченко — QA Lead (55 000 грн, фонд відділу: 87 000 грн)
    👤 Анна Ткаченко — QA-інженер (32 000 грн)

Загальний фонд оплати праці відділу: 527 000 грн
Фонд лише Backend-команди: 191 000 грн
```

Якщо завтра з'явиться `TeamLead`, який одночасно і сам пише код, і має підлеглих — досить зробити ще один клас, що реалізує `IEmployee` (найімовірніше, ще один `Composite`). Жодного клієнтського коду змінювати не потрібно.

---

## Приклад 4 (реальний сценарій) — Каталог товарів та комплекти

Реалістичний сценарій e-commerce: інтернет-магазин комп'ютерної периферії продає окремі товари, а також **комплекти (комбо/кити)**, які можуть містити як окремі товари, так і **інші комплекти** (вкладені комбо). Потрібно порахувати сумарну ціну зі знижкою, сумарну вагу для розрахунку доставки, а також перевірити, чи весь комплект в наявності на складі.

### Крок 1: Component

```csharp
// Component - спільний інтерфейс для товару і для комплекту товарів
public interface ICatalogItem
{
    string Name { get; }

    // Сумарна ціна: для товару - його власна ціна, для комплекту - сума цін вмісту зі знижкою
    decimal GetTotalPrice();

    // Сумарна вага в кілограмах - потрібна для розрахунку вартості доставки
    double GetTotalWeight();

    // Чи доступний елемент повністю (товар в наявності / всі товари комплекту в наявності)
    bool IsAvailable();

    // Список назв недоступних товарів (для детального звіту користувачу)
    IEnumerable<string> GetUnavailableItems();

    void PrintDetails(int indent = 0);
}
```

### Крок 2: Leaf — окремий товар

```csharp
// Leaf - окремий товар на складі, дітей немає
public class Product : ICatalogItem
{
    public string Name { get; }
    public decimal Price { get; }
    public double Weight { get; }      // кг
    public int StockQuantity { get; set; }

    public Product(string name, decimal price, double weight, int stockQuantity)
    {
        Name = name;
        Price = price;
        Weight = weight;
        StockQuantity = stockQuantity;
    }

    public decimal GetTotalPrice() => Price;
    public double GetTotalWeight() => Weight;
    public bool IsAvailable() => StockQuantity > 0;

    public IEnumerable<string> GetUnavailableItems()
        => IsAvailable() ? Enumerable.Empty<string>() : new[] { Name };

    public void PrintDetails(int indent = 0)
    {
        var prefix = new string(' ', indent * 2);
        var status = IsAvailable() ? "✅" : "❌ немає в наявності";
        Console.WriteLine($"{prefix}📦 {Name} — {Price:F2} грн, {Weight:F2} кг {status}");
    }
}
```

### Крок 3: Composite — комплект товарів (Kit / Bundle)

```csharp
// Composite - комплект (комбо), може містити товари і/або вкладені комплекти.
// Знижка застосовується уніфіковано до всієї суми вмісту - незалежно від того,
// що там всередині: прості товари чи вкладені комбо.
public class ProductBundle : ICatalogItem
{
    public string Name { get; }

    // Знижка у частках (0.10 = 10%)
    public decimal DiscountPercent { get; }

    private readonly List<ICatalogItem> _items = new();
    public IReadOnlyList<ICatalogItem> Items => _items;

    public ProductBundle(string name, decimal discountPercent = 0m)
    {
        Name = name;
        DiscountPercent = discountPercent;
    }

    public void Add(ICatalogItem item)
    {
        // Захист від додавання самого себе (пряме циклічне посилання) -
        // детальніше про цю проблему в розділі "Антипатерни"
        if (ReferenceEquals(item, this))
            throw new InvalidOperationException(
                $"Не можна додати комплект \"{Name}\" сам до себе.");

        // Захист від додавання предка як нащадка (непряма циклічна залежність)
        if (item is ProductBundle bundleItem && bundleItem.Contains(this))
            throw new InvalidOperationException(
                $"Циклічне посилання: \"{item.Name}\" вже містить \"{Name}\" у своєму піддереві.");

        _items.Add(item);
    }

    public void Remove(ICatalogItem item) => _items.Remove(item);

    // Допоміжний рекурсивний пошук - чи міститься elem десь у піддереві цього комплекту
    public bool Contains(ICatalogItem elem)
    {
        if (ReferenceEquals(this, elem)) return true;
        foreach (var item in _items)
        {
            if (ReferenceEquals(item, elem)) return true;
            if (item is ProductBundle nested && nested.Contains(elem)) return true;
        }
        return false;
    }

    public decimal GetTotalPrice()
    {
        // Рекурсивно підсумовуємо ціни всіх елементів (товарів і вкладених комплектів)
        decimal subtotal = _items.Sum(i => i.GetTotalPrice());

        // Знижка комплекту застосовується один раз до підсумкової суми -
        // це і є "правило знижки", що діє уніфіковано через Composite,
        // не турбуючись про те, з чого саме складається сума
        return Math.Round(subtotal * (1 - DiscountPercent), 2);
    }

    public double GetTotalWeight() => _items.Sum(i => i.GetTotalWeight());

    // Комплект доступний повністю, лише якщо ВСІ елементи в наявності (рекурсивно)
    public bool IsAvailable() => _items.All(i => i.IsAvailable());

    public IEnumerable<string> GetUnavailableItems()
        => _items.SelectMany(i => i.GetUnavailableItems());

    public void PrintDetails(int indent = 0)
    {
        var prefix = new string(' ', indent * 2);
        Console.WriteLine($"{prefix}🎁 {Name} (знижка {DiscountPercent:P0})");

        foreach (var item in _items)
            item.PrintDetails(indent + 1); // рекурсія - Product чи вкладений ProductBundle
    }
}
```

### Крок 4: Допоміжний сервіс перевірки замовлення

```csharp
// Простий сервіс, що працює виключно через ICatalogItem -
// йому абсолютно байдуже, чи це один товар, чи величезний вкладений комплект
public static class OrderValidator
{
    public static void ValidateAndPrint(ICatalogItem item)
    {
        Console.WriteLine($"Загальна ціна: {item.GetTotalPrice():N2} грн");
        Console.WriteLine($"Загальна вага: {item.GetTotalWeight():F2} кг");

        if (item.IsAvailable())
        {
            Console.WriteLine("✅ Все в наявності — замовлення можна оформити.");
        }
        else
        {
            Console.WriteLine("❌ Недоступні товари:");
            foreach (var name in item.GetUnavailableItems())
                Console.WriteLine($"   - {name}");
        }
    }
}
```

### Використання (`Program.Main`)

```csharp
class Program
{
    static void Main()
    {
        // --- Окремі товари каталогу ---
        var mouse    = new Product("Ігрова миша Logitech G Pro", 1899.00m, 0.10, stockQuantity: 12);
        var keyboard = new Product("Механічна клавіатура Keychron K2", 3299.00m, 0.85, stockQuantity: 5);
        var monitor  = new Product("Монітор Dell 27\" 144Hz", 12999.00m, 4.20, stockQuantity: 3);
        var headset  = new Product("Гарнітура HyperX Cloud II", 2499.00m, 0.35, stockQuantity: 0); // немає на складі!

        Console.WriteLine("=== КАТАЛОГ ТОВАРІВ ===");
        foreach (var p in new[] { mouse, keyboard, monitor, headset })
            p.PrintDetails();

        // --- Комплект 1: клавіатура + миша, знижка 5% ---
        var comboKeyboardMouse = new ProductBundle("Комбо \"Клавіатура + Миша\"", discountPercent: 0.05m);
        comboKeyboardMouse.Add(keyboard);
        comboKeyboardMouse.Add(mouse);

        Console.WriteLine();
        Console.WriteLine("=== КОМПЛЕКТ: Клавіатура + Миша (знижка 5%) ===");
        comboKeyboardMouse.PrintDetails();
        Console.WriteLine();
        OrderValidator.ValidateAndPrint(comboKeyboardMouse);

        // --- Комплект 2: "Геймерський стартовий набір" - вкладає в себе Комплект 1! ---
        // Composite дозволяє це зробити абсолютно природно - ProductBundle сам є ICatalogItem
        var starterKit = new ProductBundle("Геймерський стартовий набір", discountPercent: 0.15m);
        starterKit.Add(monitor);
        starterKit.Add(comboKeyboardMouse); // вкладений комплект - Composite всередині Composite
        starterKit.Add(headset);            // товар, якого немає на складі

        Console.WriteLine();
        Console.WriteLine("=== ГЕЙМЕРСЬКИЙ СТАРТОВИЙ НАБІР (знижка 15%) ===");
        starterKit.PrintDetails();
        Console.WriteLine();
        OrderValidator.ValidateAndPrint(starterKit);

        // --- Поповнюємо склад гарнітури і перевіряємо ще раз ---
        Console.WriteLine();
        Console.WriteLine("--- Поповнюємо склад гарнітури (+10 шт.) ---");
        headset.StockQuantity += 10;
        OrderValidator.ValidateAndPrint(starterKit);
    }
}
```

### Очікуваний вивід

```
=== КАТАЛОГ ТОВАРІВ ===
📦 Ігрова миша Logitech G Pro — 1899.00 грн, 0.10 кг ✅
📦 Механічна клавіатура Keychron K2 — 3299.00 грн, 0.85 кг ✅
📦 Монітор Dell 27" 144Hz — 12999.00 грн, 4.20 кг ✅
📦 Гарнітура HyperX Cloud II — 2499.00 грн, 0.35 кг ❌ немає в наявності

=== КОМПЛЕКТ: Клавіатура + Миша (знижка 5%) ===
🎁 Комбо "Клавіатура + Миша" (знижка 5%)
  📦 Механічна клавіатура Keychron K2 — 3299.00 грн, 0.85 кг ✅
  📦 Ігрова миша Logitech G Pro — 1899.00 грн, 0.10 кг ✅

Загальна ціна: 4 938.10 грн
Загальна вага: 0.95 кг
✅ Все в наявності — замовлення можна оформити.

=== ГЕЙМЕРСЬКИЙ СТАРТОВИЙ НАБІР (знижка 15%) ===
🎁 Геймерський стартовий набір (знижка 15%)
  📦 Монітор Dell 27" 144Hz — 12999.00 грн, 4.20 кг ✅
  🎁 Комбо "Клавіатура + Миша" (знижка 5%)
    📦 Механічна клавіатура Keychron K2 — 3299.00 грн, 0.85 кг ✅
    📦 Ігрова миша Logitech G Pro — 1899.00 грн, 0.10 кг ✅
  📦 Гарнітура HyperX Cloud II — 2499.00 грн, 0.35 кг ❌ немає в наявності

Загальна ціна: 17 370.69 грн
Загальна вага: 5.50 кг
❌ Недоступні товари:
   - Гарнітура HyperX Cloud II

--- Поповнюємо склад гарнітури (+10 шт.) ---
Загальна ціна: 17 370.69 грн
Загальна вага: 5.50 кг
✅ Все в наявності — замовлення можна оформити.
```

Зверніть увагу на ключові моменти цього прикладу:

- `starterKit` містить у собі `comboKeyboardMouse`, який сам є `Composite`. Це **Composite всередині Composite** — саме заради такої довільної вкладеності й існує патерн.
- `GetTotalPrice()` для `starterKit` бере вже **зі знижкою порахований** підсумок `comboKeyboardMouse.GetTotalPrice()` (4938.10 грн) і застосовує ще одну знижку (15%) до всієї суми разом з монітором і гарнітурою — знижки коректно накопичуються рекурсивно, без спеціального коду для "комплексних" комплектів.
- `IsAvailable()` і `GetUnavailableItems()` рекурсивно "пробиваються" через усе дерево — `OrderValidator` не написав жодного рядка, специфічного саме для `ProductBundle`.
- Метод `Contains()` і перевірки в `Add()` захищають від циклічних посилань — детальніше про це в розділі помилок нижче.

---

## Composite vs Decorator vs Iterator

Composite, Decorator та Iterator часто згадуються разом, бо всі три активно використовують **рекурсивну композицію об'єктів через спільний інтерфейс** — і тому їх легко переплутати. Розберемо різницю.

### Composite: дерево "один-або-багато"

```
                Component
               /    |     \
            Leaf  Leaf   Composite
                            /    \
                         Leaf   Leaf
```

Composite моделює **дерево довільної форми**: у вузла може бути 0, 1 або багато дітей, і будь-яка дитина сама може бути піддеревом. Мета — **уніфікувати** роботу з одиничним об'єктом і групою об'єктів під одним інтерфейсом.

**Запитай себе:** "Чи є в мене структура 'частина-ціле', де я хочу викликати ОДНУ операцію однаково і для окремого елемента, і для цілої групи/дерева?" → Так → **Composite**.

### Decorator: лінійний ланцюжок "рівно один"

```
Decorator3( Decorator2( Decorator1( RealObject ) ) )

┌────────────┐
│ Decorator3 │
│ ┌────────┐ │
│ │Decorat2│ │
│ │┌──────┐│ │
│ ││Decor1││ │
│ ││┌────┐││ │
│ │││Real│││ │
│ ││└────┘││ │
│ │└──────┘│ │
│ └────────┘ │
└────────────┘
```

Decorator обгортає **рівно один** об'єкт і додає йому **одну додаткову відповідальність**, зберігаючи той самий інтерфейс. Це не дерево, а лінійний ланцюжок обгорток. Мета — **розширити поведінку** одного об'єкта динамічно, не чіпаючи його клас.

**Запитай себе:** "Чи я хочу додати ОДНУ нову поведінку ОДНОМУ об'єкту, зберігши його інтерфейс, і, можливо, комбінувати декілька таких поведінок у рантаймі?" → Так → **Decorator**.

### Iterator: спосіб обходу, а не структура

```
foreach (var item in tree)   // Iterator ховає деталі обходу дерева
{
    // клієнт бачить лише плаский потік елементів,
    // не знаючи, що насправді це рекурсивний обхід Composite-дерева
}
```

Iterator — це не структура даних, а **спосіб послідовно обійти** елементи колекції чи дерева, не розкриваючи її внутрішню реалізацію (масив це, зв'язний список чи Composite-дерево — байдуже). Iterator часто застосовують **разом** із Composite: щоб надати клієнту простий `foreach` замість того, щоб він сам писав рекурсивний обхід дерева.

**Запитай себе:** "Чи мені потрібно послідовно обійти елементи структури (в тому числі дерева Composite), не розкриваючи, як саме вона влаштована всередині?" → Так → **Iterator**.

### Порівняльна таблиця

| Ознака | Composite | Decorator | Iterator |
|---|---|---|---|
| **Мета** | Уніфікувати "частину" і "ціле" | Додати поведінку одному об'єкту | Обійти структуру без розкриття деталей |
| **Форма** | Дерево (0..N дітей) | Лінійний ланцюжок (рівно 1 вкладений) | Не структура даних — алгоритм обходу |
| **Кількість вкладених об'єктів** | 0 або багато | Рівно один | Не застосовно |
| **Що спільного з іншими** | Спільний інтерфейс Leaf/Composite | Спільний інтерфейс з обгорнутим об'єктом | Спільний інтерфейс ітератора (`MoveNext`/`Current`) |
| **Типовий приклад** | Файлова система, UI-дерево, BOM | `Stream`-обгортки, ASP.NET Middleware | `IEnumerator<T>`, `foreach` |
| **Як комбінуються** | Вузол дерева можна обгорнути декоратором | Декоратор можна застосувати до Leaf і Composite однаково (в обох той самий інтерфейс!) | Ітератор часто обходить саме Composite-дерево |

Практичний приклад комбінування: якщо `IFileSystemItem` (з Прикладу 1) — це `Component`, то ніщо не заважає створити `LoggingFileSystemItemDecorator : IFileSystemItem`, який обгортає **будь-який** `IFileSystemItem` — байдуже, `File` це чи цілий `Folder` — і логує кожен виклик `GetSize()`. Це можливо саме тому, що Composite і Decorator розділяють один прийом: "той самий інтерфейс для обгортки і для вмісту" — але Composite робить це для **дерева**, а Decorator — для **одного ланцюжка**.

---

## Переваги та недоліки

### Переваги

- **Уніфікована обробка Leaf і Composite** — клієнтський код працює через один інтерфейс, не розрізняючи одиничний об'єкт і групу.
- **Відкритість/закритість (OCP)** — новий тип листка чи композита додається без зміни існуючого клієнтського коду.
- **Спрощення клієнтського коду** — зникають нескінченні `if (item is X) else if (item is Y)`.
- **Природна рекурсія** — операції над деревом (сума, пошук, друк) виражаються рекурсивно й лаконічно завдяки поліморфізму.
- **Легко будувати складні структури з простих об'єктів** — довільна глибина вкладеності "з коробки", без додаткового коду.
- **Легко додавати нові операції** — достатньо додати метод в інтерфейс `Component` і реалізувати його в `Leaf` та `Composite` (хоча це й вимагає зміни самого інтерфейсу — див. недоліки).

### Недоліки

- **Дизайн може стати надто загальним** — важко обмежити, які саме типи дітей дозволені для конкретного `Composite` (наприклад, хочеться дозволити `Folder` містити тільки `File` та `Folder`, але заборонити щось інше — інтерфейс `Component` цього не гарантує на рівні компілятора).
- **Trade-off типобезпеки** — якщо методи керування дітьми (`Add`/`Remove`/`GetChild`) винесені в спільний інтерфейс `Component` заради "прозорості", `Leaf` змушений або підтримувати їх штучно, або кидати виключення в рантаймі (детальніше — у наступному розділі).
- **Складніше гарантувати інваріанти дерева** — захист від циклічних посилань, обмеження глибини вкладеності, унікальність дітей — усе це лягає на плечі розробника `Composite`, а не забезпечується патерном "з коробки".
- **Може бути надлишковим для не ієрархічних даних** — якщо структура насправді плоска (список без вкладеності), Composite додає зайву абстракцію без реальної користі.
- **Дебагінг рекурсії** — при глибокій вкладеності стек викликів під час обходу дерева може бути довгим і важким для читання, а помилка в одному вузлі може "розмножитись" по всьому дереву.

---

## Антипатерни та поширені помилки

### Помилка 1: де розміщувати Add/Remove — "прозорість" проти "безпеки"

Це справжня дилема, яку описують самі автори GoF: чи виносити методи керування дітьми (`Add`, `Remove`, `GetChild`) у спільний інтерфейс `Component`, чи лишати їх тільки в `Composite`?

**Варіант "прозорий" (transparent)** — методи в `Component`, клієнт може викликати `Add` на будь-чому, не перевіряючи тип:

```csharp
// НЕПРАВИЛЬНО (точніше - "прозорий", але небезпечний варіант)
public interface IComponent
{
    void Operation();
    void Add(IComponent child);
    void Remove(IComponent child);
    IComponent GetChild(int index);
}

public class Leaf : IComponent
{
    public void Operation() => Console.WriteLine("Leaf: виконую операцію");

    // Leaf ЗМУШЕНИЙ реалізувати ці методи, хоча дітей у нього ніколи не буде.
    // Єдиний вихід - кидати виключення в рантаймі
    public void Add(IComponent child)
        => throw new NotSupportedException("Leaf не може мати дітей!");

    public void Remove(IComponent child)
        => throw new NotSupportedException("Leaf не може мати дітей!");

    public IComponent GetChild(int index)
        => throw new NotSupportedException("Leaf не має дітей!");
}
```

Проблема: клієнт може написати `leaf.Add(something)`, і код **скомпілюється**, але впаде в рантаймі з `NotSupportedException`. Компілятор не захищає від цієї помилки — це і є плата за "прозорість" (однаковий інтерфейс для всіх).

**Варіант "безпечний" (safe)** — методи керування дітьми лишаються тільки в `Composite`:

```csharp
// ПРАВИЛЬНО (точніше - "безпечний" варіант)
public interface IComponent
{
    void Operation(); // тільки та операція, що дійсно спільна для Leaf і Composite
}

public class Leaf : IComponent
{
    public void Operation() => Console.WriteLine("Leaf: виконую операцію");
    // Leaf не змушений реалізовувати Add/Remove - їх просто немає в контракті
}

public class Composite : IComponent
{
    private readonly List<IComponent> _children = new();

    // Add/Remove/GetChild - специфічні для Composite, оголошені лише тут
    public void Add(IComponent child) => _children.Add(child);
    public void Remove(IComponent child) => _children.Remove(child);
    public IComponent GetChild(int index) => _children[index];

    public void Operation()
    {
        foreach (var child in _children)
            child.Operation();
    }
}

// Клієнту, який будує дерево, доведеться один раз перевірити тип -
// але ЛИШЕ у момент побудови дерева, а не при кожному виклику Operation()
if (node is Composite composite)
{
    composite.Add(new Leaf());
}
```

**Висновок:** жоден варіант не є "правильним" абсолютно — це усвідомлений компроміс GoF між **прозорістю** (однаковий інтерфейс для всього, простіше використовувати, але помилки виявляються в рантаймі) і **безпекою типів** (компілятор захищає від виклику `Add` на `Leaf`, але клієнту, який будує дерево, іноді доведеться перевіряти тип). На практиці "безпечний" варіант обирають частіше, коли дерево будується один раз у конкретному, контрольованому місці коду (як у наших прикладах вище — `Folder.Add`, `ProductBundle.Add` доступні лише на конкретних класах).

### Помилка 2: домішування посилання на батька в рекурсивну операцію

Іноді вузлу дерева потрібне посилання на батька (наприклад, для навігації "піднятись на рівень вище" в UI). Небезпека в тому, щоб випадково використати це посилання **всередині** рекурсивної операції, що йде "вниз" по дереву — це призводить до нескінченної рекурсії.

```csharp
// НЕПРАВИЛЬНО
public class Folder : IFileSystemItem
{
    public Folder Parent { get; set; }
    private readonly List<IFileSystemItem> _items = new();

    public long GetSize()
    {
        long total = _items.Sum(i => i.GetSize());

        // ПОМИЛКА: хтось "для повноти картини" вирішив врахувати і батька -
        // але GetSize() батька знову порахує ВСІХ своїх дітей, включно з ЦІЄЮ папкою!
        if (Parent != null)
            total += Parent.GetSize(); // ЗАЦИКЛЕННЯ - StackOverflowException

        return total;
    }
}
```

Що відбувається: `child.GetSize()` викликає `Parent.GetSize()`, а `Parent.GetSize()` знову підсумовує своїх дітей — включно з `child`, викликаючи знову `child.GetSize()`. Кожен виклик породжує ще один — гарантований `StackOverflowException` (а якщо пощастить не впасти одразу — то щонайменше катастрофічно неправильна сума).

```csharp
// ПРАВИЛЬНО
public class Folder : IFileSystemItem
{
    // Посилання на батька можна зберігати - це нормально для навігації,
    // наприклад, для методу "піднятись на верхній рівень" чи "видалити себе з батька"
    public Folder Parent { get; internal set; }

    private readonly List<IFileSystemItem> _items = new();

    // Рекурсивна операція йде ЛИШЕ вниз по дереву - до дітей.
    // Parent тут узагалі не згадується
    public long GetSize() => _items.Sum(i => i.GetSize());

    // А ось окрема, НЕ рекурсивна операція цілком може використовувати Parent -
    // наприклад, видалення самого себе з батьківської папки
    public void RemoveFromParent() => Parent?.Remove(this);
}
```

**Правило:** посилання на батька — це навігаційна деталь, яку можна використовувати в операціях, що явно рухаються "вгору" (наприклад, `RemoveFromParent`, `GetPath`), але його **ніколи** не можна змішувати з операціями, що рекурсивно обходять дерево "вниз" (`GetSize`, `Render`, `PrintStructure`) — інакше рекурсія піде в обидва боки одночасно і зациклиться.

### Помилка 3: відсутність захисту від циклічних посилань

Якщо `Composite` дозволяє додати як дитину будь-який `Component`, ніщо (крім явної перевірки) не завадить додати сам композит у себе — напряму або через кілька рівнів вкладеності.

```csharp
// НЕПРАВИЛЬНО - Add() не перевіряє нічого
public class Folder : IFileSystemItem
{
    private readonly List<IFileSystemItem> _items = new();
    public void Add(IFileSystemItem item) => _items.Add(item);
    public long GetSize() => _items.Sum(i => i.GetSize());
    // ...
}

var root = new Folder("root");
var sub  = new Folder("sub");

root.Add(sub);
sub.Add(root);          // ЦИКЛ: root містить sub, а sub тепер містить root!

root.GetSize();          // StackOverflowException - нескінченна рекурсія:
                          // root -> sub -> root -> sub -> ...
```

```csharp
// ПРАВИЛЬНО - Add() захищається від прямого і непрямого циклу
public class Folder : IFileSystemItem
{
    private readonly List<IFileSystemItem> _items = new();

    public void Add(IFileSystemItem item)
    {
        // Захист від прямого циклу: додавання самого себе
        if (ReferenceEquals(item, this))
            throw new InvalidOperationException("Папка не може містити саму себе.");

        // Захист від непрямого циклу: додавання предка як нащадка
        if (item is Folder folder && folder.Contains(this))
            throw new InvalidOperationException(
                $"Циклічне посилання: \"{item.Name}\" вже містить \"{Name}\" у своєму піддереві.");

        _items.Add(item);
    }

    // Рекурсивна перевірка: чи міститься elem десь у піддереві цієї папки
    public bool Contains(IFileSystemItem elem)
    {
        if (ReferenceEquals(this, elem)) return true;
        return _items.Any(i => ReferenceEquals(i, elem)) ||
               _items.OfType<Folder>().Any(f => f.Contains(elem));
    }

    public long GetSize() => _items.Sum(i => i.GetSize());
}
```

Саме такий захист ми вже застосували в `ProductBundle.Add()` у Прикладі 4 — це не надлишкова обережність, а необхідна умова коректності будь-якого дерева, яке будується динамічно (а не один раз статично прописується в коді).

---

## Підсумок

Composite варто застосовувати, коли:

- у предметній області природно існує структура **"частина-ціле"**, яку зручно представити деревом (файлова система, UI, оргструктура, каталог товарів / специфікація виробу — BOM);
- клієнтський код має **однаково** працювати з окремим об'єктом і з групою об'єктів, не розрізняючи їх;
- структура може мати **довільну глибину вкладеності**, і кількість "дітей" наперед невідома;
- потрібно уникнути розповзання перевірок типу (`if (item is X) else if (item is Y)`) по всій кодовій базі;
- операції над деревом природно виражаються **рекурсивно** (сума, пошук, валідація, промальовка, серіалізація).

### Мінімальний шаблон

```csharp
// Component - спільний інтерфейс для листка і для контейнера
public interface IComponent
{
    void Operation();
}

// Leaf - кінцевий елемент, дітей не має (базовий випадок рекурсії)
public class Leaf : IComponent
{
    public string Name { get; }
    public Leaf(string name) => Name = name;

    public void Operation()
        => Console.WriteLine($"Leaf {Name}: виконую операцію");
}

// Composite - контейнер, що містить інші IComponent (Leaf або вкладений Composite)
public class Composite : IComponent
{
    private readonly List<IComponent> _children = new();

    public void Add(IComponent component) => _children.Add(component);
    public void Remove(IComponent component) => _children.Remove(component);

    public void Operation()
    {
        Console.WriteLine("Composite: делегую дітям");

        // Рекурсивний випадок - для кожної дитини викликаємо ту саму операцію,
        // не знаючи (і не переймаючись), Leaf це чи вкладений Composite
        foreach (var child in _children)
            child.Operation();
    }
}

// Client
var tree = new Composite();
tree.Add(new Leaf("A"));

var subtree = new Composite();
subtree.Add(new Leaf("B"));
subtree.Add(new Leaf("C"));
tree.Add(subtree);

tree.Operation(); // однаково спрацює незалежно від глибини й форми дерева
```

---

*Документ підготовлено для вивчення патернів проектування. Всі приклади протестовані на .NET 6+.*
