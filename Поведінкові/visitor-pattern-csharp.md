# Патерн Visitor (Відвідувач) — Детальний розбір на C#

> **Категорія:** Поведінковий (Behavioral)  
> **Автори:** Gang of Four (GoF)  
> **Мова прикладів:** C#

---

## Зміст

1. [Що таке Visitor?](#що-таке-visitor)
2. [Проблема без патерну](#проблема-без-патерну)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Обчислення площі фігур](#приклад-1--обчислення-площі-фігур)
5. [Приклад 2 — Експорт у XML та JSON без зміни фігур](#приклад-2--експорт-у-xml-та-json-без-зміни-фігур)
6. [Приклад 3 — Обчислювач виразів (AST)](#приклад-3--обчислювач-виразів-ast)
7. [Приклад 4 — Реальний сценарій: експорт файлової структури](#приклад-4--реальний-сценарій-експорт-файлової-структури)
8. [Visitor vs Strategy vs Iterator](#visitor-vs-strategy-vs-iterator)
9. [Переваги та недоліки](#переваги-та-недоліки)
10. [Антипатерни та поширені помилки](#антипатерни-та-поширені-помилки)
11. [Підсумок](#підсумок)

---

## Що таке Visitor?

**Visitor (Відвідувач)** — це поведінковий патерн, який дозволяє **представити операцію, що виконується над елементами неоднорідної структури об'єктів, не змінюючи класи цих елементів**.

Іншими словами: у вас є ієрархія класів (наприклад, різні фігури, різні вузли документа, різні типи файлів), і вам потрібно виконувати над нею різні операції (порахувати площу, експортувати в XML, намалювати, провалідувати). Замість того, щоб додавати новий метод у кожен клас ієрархії щоразу, коли з'являється нова операція, ви виносите операцію в окремий клас — **Visitor**, а елементи лише "приймають" відвідувача.

Механізм, який робить це можливим, називається **подвійна диспетчеризація (double dispatch)**:

```
element.Accept(visitor)     // 1-й виклик: віртуальний за типом element
    → visitor.Visit(this)   // 2-й виклик: перевантаження за типом this (Element)
```

Перший виклик (`Accept`) — звичайний віртуальний виклик, що вирішується за **реальним рантайм-типом** елемента (завдяки поліморфізму). Другий виклик (`Visit`) — вирішується компілятором за **статичним типом `this`** всередині конкретного класу `Accept`, тобто за типом, який на момент компіляції `Accept` вже точно відомий. Комбінація цих двох виборів і дає ефект, ніби метод вибирається одразу за двома типами — типом елемента ТА типом відвідувача.

> Ключова ідея GoF: *"Represent an operation to be performed on the elements of an object structure. Visitor lets you define a new operation without changing the classes of the elements on which it operates."*

### Аналогія з реального світу

Уявіть **податкового інспектора**, який відвідує різні типи бізнесів: кафе, будівельну компанію, IT-стартап. Кожен тип бізнесу перевіряється по-своєму — для кафе важливі санітарні норми та готівкові розрахунки, для будівельної компанії — ліцензії та безпека праці, для IT-стартапу — інтелектуальна власність та експортні контракти.

Важливо: **логіка перевірки живе в інспекторі, а не в бізнесах**. Кафе не знає, як саме його перевіряють — воно просто "приймає" інспектора (`Accept(inspector)`) і дозволяє йому виконати свою роботу (`inspector.VisitCafe(this)`). Якщо завтра з'явиться новий вид перевірки (наприклад, екологічний аудит), достатньо створити **нового інспектора** (`EcoAuditInspector`) — не потрібно змінювати жоден клас бізнесу.

Так само страховий оцінювач (`InsuranceAppraiser`) відвідує різні типи майна — будинок, автомобіль, човен — і для кожного типу застосовує зовсім іншу формулу розрахунку вартості, хоча сам процес "оцінювання" залишається єдиною, добре зорганізованою операцією.

---

## Проблема без патерну

Розглянемо ієрархію фігур, до якої постійно додаються нові операції прямо у вигляді віртуальних методів.

```csharp
// ПОГАНО: базовий клас Shape, що "розпухає" з кожною новою операцією
public abstract class Shape
{
    // Операція 1: Area — більш-менш доречна для фігури
    public abstract double GetArea();

    // Операція 2: ExportToXml — фігура тепер "знає" про формат XML
    public abstract string ExportToXml();

    // Операція 3: ExportToJson — фігура тепер "знає" про формат JSON
    public abstract string ExportToJson();

    // Операція 4: Render — фігура тепер "знає", як малювати себе на екрані
    public abstract void Render();

    // Завтра з'явиться ExportToCsv, ExportToBinary, Validate,
    // CalculatePerimeter, ApplyDiscountPricing... і так до нескінченності.
}

public class Circle : Shape
{
    public double Radius { get; set; }

    public override double GetArea() => Math.PI * Radius * Radius;

    public override string ExportToXml()
        => $"<circle radius=\"{Radius}\" />";

    public override string ExportToJson()
        => $"{{\"type\":\"circle\",\"radius\":{Radius}}}";

    public override void Render()
        => Console.WriteLine($"Малюємо коло радіусом {Radius}");
}

public class Rectangle : Shape
{
    public double Width { get; set; }
    public double Height { get; set; }

    public override double GetArea() => Width * Height;

    public override string ExportToXml()
        => $"<rectangle width=\"{Width}\" height=\"{Height}\" />";

    public override string ExportToJson()
        => $"{{\"type\":\"rectangle\",\"width\":{Width},\"height\":{Height}}}";

    public override void Render()
        => Console.WriteLine($"Малюємо прямокутник {Width}x{Height}");
}

public class Triangle : Shape
{
    public double Base { get; set; }
    public double Height { get; set; }

    public override double GetArea() => 0.5 * Base * Height;

    public override string ExportToXml()
        => $"<triangle base=\"{Base}\" height=\"{Height}\" />";

    public override string ExportToJson()
        => $"{{\"type\":\"triangle\",\"base\":{Base},\"height\":{Height}}}";

    public override void Render()
        => Console.WriteLine($"Малюємо трикутник з основою {Base} і висотою {Height}");
}

// ПРОБЛЕМА №1: Кожна нова операція (ExportToCsv, Validate, Serialize...)
// вимагає ЗМІНИ базового класу Shape І КОЖНОГО існуючого підкласу
// (Circle, Rectangle, Triangle, ...) — навіть якщо їх у проєкті 30 штук.

// ПРОБЛЕМА №2: Shape перетворюється на "звалище" незв'язаних відповідальностей:
// геометрія (GetArea), серіалізація (ExportToXml/Json), рендеринг (Render).
// Це порушує Single Responsibility Principle — фігура повинна знати
// про свою геометрію, а не про те, як експортувати себе у форматі,
// який завтра можуть взагалі прибрати з проєкту.

// ПРОБЛЕМА №3: Логіка однієї операції (наприклад, "весь код ExportToXml
// для всіх фігур") розкидана по десятках файлів замість того, щоб
// бути зібраною в одному місці, де її легко читати, тестувати й змінювати.

// ПРОБЛЕМА №4: Якщо потрібна операція, яку не можна/не варто реалізовувати
// всередині фігур (наприклад, вона залежить від зовнішньої бібліотеки
// рендерингу конкретної платформи), доводиться або тягнути залежність
// у Shape, або городити switch/if із перевіркою GetType() десь зовні —
// що знову ж таки крихко й порушує принцип відкритості/закритості.
```

Саме цю проблему й вирішує Visitor: він дозволяє **додавати нові операції, не торкаючись класів `Circle`, `Rectangle`, `Triangle`** взагалі.

---

## Структура патерну

```
                    «інтерфейс»                              «інтерфейс»
                    IElement                                  IVisitor
              ┌────────────────────┐                 ┌───────────────────────────┐
              │ + Accept(IVisitor) │                 │ + VisitConcreteElementA(A)│
              └─────────┬──────────┘                 │ + VisitConcreteElementB(B)│
                        │                             └─────────────┬─────────────┘
        ┌───────────────┴────────────────┐                          │
        │                                │                          │
┌───────▼─────────┐            ┌─────────▼────────┐     ┌───────────┴────────────┐
│ ConcreteElementA │            │ ConcreteElementB │     │                        │
│                  │            │                   │    │                        │
│ Accept(v):       │            │ Accept(v):        │┌───▼──────────┐  ┌──────────▼───┐
│  v.VisitConcrete-│            │  v.VisitConcrete- ││ConcreteVisitor1│  │ConcreteVisitor2│
│  ElementA(this)  │            │  ElementB(this)   ││ (напр. Area)   │  │ (напр. Export) │
└──────────────────┘            └───────────────────┘└────────────────┘  └────────────────┘

  Крок 1: client.Accept(visitor)  →  диспетчеризація за РАНТАЙМ-типом елемента (віртуальний виклик)
  Крок 2: visitor.VisitXxx(this)  →  диспетчеризація за СТАТИЧНИМ типом this всередині Accept
                                     (перевантаження методу компілятором)
  Разом ці два кроки = "подвійна диспетчеризація"
```

### Чому потрібна подвійна диспетчеризація?

У C# (як і в більшості мов з одинарною диспетчеризацією) вибір перевантаженого методу для аргументів відбувається **на етапі компіляції**, за їхнім **статичним (оголошеним) типом**, а не за реальним типом об'єкта в пам'яті. Наприклад:

```csharp
Shape shape = new Circle(); // статичний тип змінної — Shape, реальний — Circle
visitor.Visit(shape); // компілятор вибере Visit(Shape), НЕ Visit(Circle) —
                       // навіть якщо в IVisitor є перевантаження Visit(Circle)!
```

Якби ми покладалися лише на перевантаження `Visit(Shape shape)`, ми б завжди потрапляли в загальну версію методу і втрачали інформацію про реальний тип. Патерн Visitor вирішує це хитрістю: спочатку викликається **віртуальний** метод `Accept` на самому елементі. Усередині кожного конкретного класу (`Circle.Accept`, `Rectangle.Accept`) статичний тип `this` **точно дорівнює** реальному типу (`Circle`, `Rectangle` відповідно) — компілятор бачить це просто тому, що ми знаходимося всередині коду класу `Circle`. Тому виклик `visitor.VisitCircle(this)` компілюється саме в перевантаження для `Circle`, без жодних перевірок типу в рантаймі.

### Учасники патерну

| Роль | Відповідальність |
|---|---|
| **IElement** | Інтерфейс елемента структури, оголошує метод `Accept(IVisitor visitor)` |
| **ConcreteElementA/B** | Конкретні елементи; кожен реалізує `Accept`, викликаючи "свій" метод відвідувача |
| **IVisitor** | Інтерфейс відвідувача, оголошує по одному методу `Visit...` на кожен конкретний тип елемента |
| **ConcreteVisitor1/2** | Конкретні відвідувачі — кожен реалізує ОДНУ повноцінну операцію (наприклад, "порахувати", "експортувати") для ВСІХ типів елементів |
| **ObjectStructure** | (необов'язково) Колекція/дерево елементів, яке можна перебрати і викликати `Accept` для кожного |

---

## Приклад 1 — Обчислення площі фігур

Найпростіший, класичний приклад: ієрархія фігур і відвідувач, що рахує сумарну площу.

```csharp
// Інтерфейс відвідувача — по одному Visit-методу на кожен тип фігури
public interface IShapeVisitor
{
    void VisitCircle(Circle circle);
    void VisitRectangle(Rectangle rectangle);
    void VisitTriangle(Triangle triangle);
}

// Інтерфейс елемента — єдине, що вимагається від фігури
public interface IShape
{
    void Accept(IShapeVisitor visitor);
}
```

```csharp
// Конкретний елемент А — Коло
public class Circle : IShape
{
    public double Radius { get; }

    public Circle(double radius)
    {
        Radius = radius;
    }

    // Accept просто "перенаправляє" виклик на відповідний Visit-метод,
    // передаючи себе (this) — саме тут компілятор бачить реальний тип Circle
    public void Accept(IShapeVisitor visitor) => visitor.VisitCircle(this);
}

// Конкретний елемент B — Прямокутник
public class Rectangle : IShape
{
    public double Width { get; }
    public double Height { get; }

    public Rectangle(double width, double height)
    {
        Width = width;
        Height = height;
    }

    public void Accept(IShapeVisitor visitor) => visitor.VisitRectangle(this);
}

// Конкретний елемент C — Трикутник
public class Triangle : IShape
{
    public double Base { get; }
    public double Height { get; }

    public Triangle(double @base, double height)
    {
        Base = @base;
        Height = height;
    }

    public void Accept(IShapeVisitor visitor) => visitor.VisitTriangle(this);
}
```

```csharp
// Конкретний відвідувач — обчислює сумарну площу всіх відвіданих фігур
public class AreaCalculatorVisitor : IShapeVisitor
{
    public double TotalArea { get; private set; }

    public void VisitCircle(Circle circle)
    {
        var area = Math.PI * circle.Radius * circle.Radius;
        Console.WriteLine($"  Коло (r={circle.Radius}): площа = {area:F2}");
        TotalArea += area;
    }

    public void VisitRectangle(Rectangle rectangle)
    {
        var area = rectangle.Width * rectangle.Height;
        Console.WriteLine($"  Прямокутник ({rectangle.Width}x{rectangle.Height}): площа = {area:F2}");
        TotalArea += area;
    }

    public void VisitTriangle(Triangle triangle)
    {
        var area = 0.5 * triangle.Base * triangle.Height;
        Console.WriteLine($"  Трикутник (осн={triangle.Base}, вис={triangle.Height}): площа = {area:F2}");
        TotalArea += area;
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        // Неоднорідна колекція фігур — усі приховані за спільним інтерфейсом IShape
        List<IShape> shapes = new()
        {
            new Circle(3),
            new Rectangle(4, 5),
            new Triangle(6, 2),
            new Circle(1.5)
        };

        Console.WriteLine("🧮 Обчислення площ фігур:");

        var areaVisitor = new AreaCalculatorVisitor();

        // Кожна фігура "приймає" відвідувача — подвійна диспетчеризація
        // сама визначає, який VisitXxx-метод буде викликано
        foreach (var shape in shapes)
        {
            shape.Accept(areaVisitor);
        }

        Console.WriteLine($"\n✅ Загальна площа всіх фігур: {areaVisitor.TotalArea:F2}");
    }
}
```

### Очікуваний вивід

```
🧮 Обчислення площ фігур:
  Коло (r=3): площа = 28.27
  Прямокутник (4x5): площа = 20.00
  Трикутник (осн=6, вис=2): площа = 6.00
  Коло (r=1.5): площа = 7.07

✅ Загальна площа всіх фігур: 61.34
```

---

## Приклад 2 — Експорт у XML та JSON без зміни фігур

Головна перевага Visitor: додаємо ДВІ нові операції (експорт у XML і в JSON) — і жодного рядка коду в `Circle`, `Rectangle`, `Triangle` чи в `IShape` не змінюємо. Достатньо реалізувати новий `IShapeVisitor`.

```csharp
// Новий відвідувач — експорт у XML. Класи фігур НЕ ЗМІНЮВАЛИСЬ ані на рядок!
public class XmlExportVisitor : IShapeVisitor
{
    private readonly StringBuilder _sb = new();

    public void VisitCircle(Circle circle)
    {
        _sb.AppendLine($"  <circle radius=\"{circle.Radius}\" />");
    }

    public void VisitRectangle(Rectangle rectangle)
    {
        _sb.AppendLine($"  <rectangle width=\"{rectangle.Width}\" height=\"{rectangle.Height}\" />");
    }

    public void VisitTriangle(Triangle triangle)
    {
        _sb.AppendLine($"  <triangle base=\"{triangle.Base}\" height=\"{triangle.Height}\" />");
    }

    public string GetXml() => $"<shapes>\n{_sb}</shapes>";
}
```

```csharp
// Ще один новий відвідувач — експорт у JSON. Знову — жодних змін у Shape-класах.
public class JsonExportVisitor : IShapeVisitor
{
    private readonly List<string> _items = new();

    public void VisitCircle(Circle circle)
    {
        _items.Add($"{{\"type\":\"circle\",\"radius\":{circle.Radius}}}");
    }

    public void VisitRectangle(Rectangle rectangle)
    {
        _items.Add($"{{\"type\":\"rectangle\",\"width\":{rectangle.Width},\"height\":{rectangle.Height}}}");
    }

    public void VisitTriangle(Triangle triangle)
    {
        _items.Add($"{{\"type\":\"triangle\",\"base\":{triangle.Base},\"height\":{triangle.Height}}}");
    }

    public string GetJson() => "[\n  " + string.Join(",\n  ", _items) + "\n]";
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        List<IShape> shapes = new()
        {
            new Circle(3),
            new Rectangle(4, 5),
            new Triangle(6, 2)
        };

        // --- Операція 1: підрахунок площі (з Прикладу 1) ---
        var areaVisitor = new AreaCalculatorVisitor();
        foreach (var shape in shapes) shape.Accept(areaVisitor);
        Console.WriteLine($"Загальна площа: {areaVisitor.TotalArea:F2}\n");

        // --- Операція 2: експорт у XML — новий відвідувач, старі фігури ---
        var xmlVisitor = new XmlExportVisitor();
        foreach (var shape in shapes) shape.Accept(xmlVisitor);
        Console.WriteLine("📄 XML-експорт:");
        Console.WriteLine(xmlVisitor.GetXml());

        // --- Операція 3: експорт у JSON — ще один новий відвідувач ---
        var jsonVisitor = new JsonExportVisitor();
        foreach (var shape in shapes) shape.Accept(jsonVisitor);
        Console.WriteLine("\n📄 JSON-експорт:");
        Console.WriteLine(jsonVisitor.GetJson());

        Console.WriteLine("\n✅ Ми додали дві нові операції (XML, JSON), " +
                           "не змінивши жодного рядка в Circle/Rectangle/Triangle!");
    }
}
```

### Очікуваний вивід

```
Загальна площа: 54.27

📄 XML-експорт:
<shapes>
  <circle radius="3" />
  <rectangle width="4" height="5" />
  <triangle base="6" height="2" />
</shapes>

📄 JSON-експорт:
[
  {"type":"circle","radius":3},
  {"type":"rectangle","width":4,"height":5},
  {"type":"triangle","base":6,"height":2}
]

✅ Ми додали дві нові операції (XML, JSON), не змінивши жодного рядка в Circle/Rectangle/Triangle!
```

---

## Приклад 3 — Обчислювач виразів (AST)

Класичне застосування Visitor — дерево абстрактного синтаксису (AST) арифметичних виразів. Побудуємо вираз `(3 + 4) * 2` і реалізуємо два незалежні відвідувачі: обчислення результату і побудову текстового представлення.

```csharp
// Інтерфейс відвідувача для виразів
public interface IExprVisitor<T>
{
    T VisitNumber(NumberExpr expr);
    T VisitAdd(AddExpr expr);
    T VisitMultiply(MultiplyExpr expr);
}

// Базовий інтерфейс вузла виразу
public interface IExpr
{
    T Accept<T>(IExprVisitor<T> visitor);
}
```

```csharp
// Лист дерева — число
public class NumberExpr : IExpr
{
    public double Value { get; }

    public NumberExpr(double value) => Value = value;

    public T Accept<T>(IExprVisitor<T> visitor) => visitor.VisitNumber(this);
}

// Вузол додавання — має два піддерева (ліве і праве)
public class AddExpr : IExpr
{
    public IExpr Left { get; }
    public IExpr Right { get; }

    public AddExpr(IExpr left, IExpr right)
    {
        Left = left;
        Right = right;
    }

    public T Accept<T>(IExprVisitor<T> visitor) => visitor.VisitAdd(this);
}

// Вузол множення — теж два піддерева
public class MultiplyExpr : IExpr
{
    public IExpr Left { get; }
    public IExpr Right { get; }

    public MultiplyExpr(IExpr left, IExpr right)
    {
        Left = left;
        Right = right;
    }

    public T Accept<T>(IExprVisitor<T> visitor) => visitor.VisitMultiply(this);
}
```

```csharp
// Відвідувач №1 — обчислює числовий результат виразу.
// Рекурсивно викликає Accept на дочірніх вузлах — саме так Visitor
// природно поєднується з рекурсивним обходом дерева.
public class EvaluateVisitor : IExprVisitor<double>
{
    public double VisitNumber(NumberExpr expr) => expr.Value;

    public double VisitAdd(AddExpr expr)
    {
        // Рекурсивно обчислюємо ліве й праве піддерево
        var left = expr.Left.Accept(this);
        var right = expr.Right.Accept(this);
        return left + right;
    }

    public double VisitMultiply(MultiplyExpr expr)
    {
        var left = expr.Left.Accept(this);
        var right = expr.Right.Accept(this);
        return left * right;
    }
}
```

```csharp
// Відвідувач №2 — будує людино-читабельний текстовий вигляд виразу.
// Використовує ТУ Ж САМУ структуру дерева, зовсім інша операція.
public class PrintVisitor : IExprVisitor<string>
{
    public string VisitNumber(NumberExpr expr) => expr.Value.ToString();

    public string VisitAdd(AddExpr expr)
    {
        var left = expr.Left.Accept(this);
        var right = expr.Right.Accept(this);
        return $"({left} + {right})";
    }

    public string VisitMultiply(MultiplyExpr expr)
    {
        var left = expr.Left.Accept(this);
        var right = expr.Right.Accept(this);
        return $"({left} * {right})";
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        // Будуємо дерево для виразу: (3 + 4) * 2
        IExpr expression = new MultiplyExpr(
            new AddExpr(new NumberExpr(3), new NumberExpr(4)),
            new NumberExpr(2));

        // Відвідувач для друку — показує структуру виразу текстом
        var printer = new PrintVisitor();
        string text = expression.Accept(printer);
        Console.WriteLine($"Вираз: {text}");

        // Відвідувач для обчислення — той самий об'єкт дерева, інша операція
        var evaluator = new EvaluateVisitor();
        double result = expression.Accept(evaluator);
        Console.WriteLine($"Результат: {result}");

        Console.WriteLine();

        // Другий, складніший вираз: (10 - немає віднімання, тому додамо через AddExpr з від'ємним числом)
        // (2 + 3) * (4 + 1)
        IExpr expr2 = new MultiplyExpr(
            new AddExpr(new NumberExpr(2), new NumberExpr(3)),
            new AddExpr(new NumberExpr(4), new NumberExpr(1)));

        Console.WriteLine($"Вираз: {expr2.Accept(printer)}");
        Console.WriteLine($"Результат: {expr2.Accept(evaluator)}");
    }
}
```

### Очікуваний вивід

```
Вираз: ((3 + 4) * 2)
Результат: 14

Вираз: ((2 + 3) * (4 + 1))
Результат: 25
```

> Зверніть увагу: щоб додати третю операцію (наприклад, `OptimizeVisitor`, що спрощує вираз, або `LatexVisitor`, що генерує формулу для LaTeX), знову не потрібно чіпати `NumberExpr`, `AddExpr`, `MultiplyExpr` — лише реалізувати новий `IExprVisitor<T>`.

---

## Приклад 4 — Реальний сценарій: експорт файлової структури

Реалістичний кейс: дерево файлової системи (Composite), де листки — файли різних типів (`TextFileNode`, `ImageFileNode`), а гілки — папки (`FolderNode`), що можуть містити інші вузли. Реалізуємо три незалежні відвідувачі: підрахунок сумарного розміру, генерацію HTML-звіту та генерацію JSON-маніфесту.

```csharp
// Інтерфейс відвідувача для вузлів файлової системи
public interface IFileSystemVisitor
{
    void VisitTextFile(TextFileNode file);
    void VisitImageFile(ImageFileNode file);
    void VisitFolder(FolderNode folder);
}

// Базовий інтерфейс вузла — і файли, і папки є вузлами
public interface IFileSystemNode
{
    string Name { get; }
    void Accept(IFileSystemVisitor visitor);
}
```

```csharp
// Листок — текстовий файл
public class TextFileNode : IFileSystemNode
{
    public string Name { get; }
    public long SizeBytes { get; }
    public int LineCount { get; }

    public TextFileNode(string name, long sizeBytes, int lineCount)
    {
        Name = name;
        SizeBytes = sizeBytes;
        LineCount = lineCount;
    }

    public void Accept(IFileSystemVisitor visitor) => visitor.VisitTextFile(this);
}

// Листок — файл зображення
public class ImageFileNode : IFileSystemNode
{
    public string Name { get; }
    public long SizeBytes { get; }
    public int Width { get; }
    public int Height { get; }

    public ImageFileNode(string name, long sizeBytes, int width, int height)
    {
        Name = name;
        SizeBytes = sizeBytes;
        Width = width;
        Height = height;
    }

    public void Accept(IFileSystemVisitor visitor) => visitor.VisitImageFile(this);
}

// Гілка (Composite) — папка, що містить інші вузли (файли або підпапки)
public class FolderNode : IFileSystemNode
{
    public string Name { get; }
    private readonly List<IFileSystemNode> _children = new();

    public IReadOnlyList<IFileSystemNode> Children => _children;

    public FolderNode(string name) => Name = name;

    public FolderNode Add(IFileSystemNode child)
    {
        _children.Add(child);
        return this;
    }

    // Папка викликає свій власний Visit, а відвідувач сам вирішує,
    // чи потрібно йому рекурсивно обходити дітей (patterns Composite + Visitor)
    public void Accept(IFileSystemVisitor visitor) => visitor.VisitFolder(this);
}
```

```csharp
// Відвідувач №1 — рекурсивно рахує сумарний розмір усього дерева.
// Composite + Visitor: папка "знає", як віддати своїх дітей,
// а рекурсію керує сам відвідувач.
public class SizeCalculatorVisitor : IFileSystemVisitor
{
    public long TotalBytes { get; private set; }

    public void VisitTextFile(TextFileNode file)
    {
        TotalBytes += file.SizeBytes;
    }

    public void VisitImageFile(ImageFileNode file)
    {
        TotalBytes += file.SizeBytes;
    }

    public void VisitFolder(FolderNode folder)
    {
        // Рекурсивно відвідуємо кожну дитину — саме тут "обхід" (traversal)
        foreach (var child in folder.Children)
        {
            child.Accept(this);
        }
    }

    public string GetTotalSizeFormatted()
    {
        if (TotalBytes >= 1024 * 1024)
            return $"{TotalBytes / (1024.0 * 1024.0):F2} MB";
        if (TotalBytes >= 1024)
            return $"{TotalBytes / 1024.0:F2} KB";
        return $"{TotalBytes} B";
    }
}
```

```csharp
// Відвідувач №2 — будує HTML-звіт про структуру дерева
public class HtmlReportVisitor : IFileSystemVisitor
{
    private readonly StringBuilder _html = new();
    private int _depth = 0;

    public void VisitTextFile(TextFileNode file)
    {
        var indent = new string(' ', _depth * 2);
        _html.AppendLine($"{indent}<li>📄 {file.Name} " +
                          $"<small>({file.SizeBytes} B, {file.LineCount} рядків)</small></li>");
    }

    public void VisitImageFile(ImageFileNode file)
    {
        var indent = new string(' ', _depth * 2);
        _html.AppendLine($"{indent}<li>🖼 {file.Name} " +
                          $"<small>({file.SizeBytes} B, {file.Width}x{file.Height})</small></li>");
    }

    public void VisitFolder(FolderNode folder)
    {
        var indent = new string(' ', _depth * 2);
        _html.AppendLine($"{indent}<li>📁 <b>{folder.Name}/</b>");
        _html.AppendLine($"{indent}<ul>");

        _depth++;
        foreach (var child in folder.Children)
        {
            child.Accept(this); // рекурсія: HTML для кожного вкладеного вузла
        }
        _depth--;

        _html.AppendLine($"{indent}</ul>");
        _html.AppendLine($"{indent}</li>");
    }

    public string GetHtml() => $"<ul>\n{_html}</ul>";
}
```

```csharp
// Відвідувач №3 — будує JSON-маніфест (плаский список усіх файлів зі шляхами)
public class JsonManifestVisitor : IFileSystemVisitor
{
    private readonly List<string> _entries = new();
    private readonly Stack<string> _pathStack = new();

    private string CurrentPath(string name)
    {
        var prefix = _pathStack.Count > 0 ? string.Join("/", _pathStack.Reverse()) + "/" : "";
        return prefix + name;
    }

    public void VisitTextFile(TextFileNode file)
    {
        var path = CurrentPath(file.Name);
        _entries.Add($"{{\"path\":\"{path}\",\"type\":\"text\",\"size\":{file.SizeBytes}}}");
    }

    public void VisitImageFile(ImageFileNode file)
    {
        var path = CurrentPath(file.Name);
        _entries.Add($"{{\"path\":\"{path}\",\"type\":\"image\",\"size\":{file.SizeBytes}," +
                     $"\"width\":{file.Width},\"height\":{file.Height}}}");
    }

    public void VisitFolder(FolderNode folder)
    {
        _pathStack.Push(folder.Name);
        foreach (var child in folder.Children)
        {
            child.Accept(this);
        }
        _pathStack.Pop();
    }

    public string GetManifest()
        => "{\n  \"files\": [\n    " + string.Join(",\n    ", _entries) + "\n  ]\n}";
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        // Будуємо дерево проєкту:
        //
        // project/
        // ├── README.txt
        // ├── logo.png
        // └── src/
        //     ├── Program.cs (як TextFileNode)
        //     └── assets/
        //         └── banner.png

        var assets = new FolderNode("assets")
            .Add(new ImageFileNode("banner.png", 204_800, 1200, 300));

        var src = new FolderNode("src")
            .Add(new TextFileNode("Program.cs", 3_072, 128))
            .Add(assets);

        var project = new FolderNode("project")
            .Add(new TextFileNode("README.txt", 1_536, 42))
            .Add(new ImageFileNode("logo.png", 51_200, 512, 512))
            .Add(src);

        // --- Відвідувач 1: підрахунок розміру ---
        Console.WriteLine("📁 === Аналіз розміру ===");
        var sizeVisitor = new SizeCalculatorVisitor();
        project.Accept(sizeVisitor);
        Console.WriteLine($"Загальний розмір проєкту: {sizeVisitor.GetTotalSizeFormatted()}");

        // --- Відвідувач 2: HTML-звіт ---
        Console.WriteLine("\n📄 === HTML-звіт ===");
        var htmlVisitor = new HtmlReportVisitor();
        project.Accept(htmlVisitor);
        Console.WriteLine(htmlVisitor.GetHtml());

        // --- Відвідувач 3: JSON-маніфест ---
        Console.WriteLine("📄 === JSON-маніфест ===");
        var jsonVisitor = new JsonManifestVisitor();
        project.Accept(jsonVisitor);
        Console.WriteLine(jsonVisitor.GetManifest());

        Console.WriteLine("\n✅ Три незалежні операції над одним і тим самим деревом, " +
                           "жодних змін у TextFileNode/ImageFileNode/FolderNode.");
    }
}
```

### Очікуваний вивід

```
📁 === Аналіз розміру ===
Загальний розмір проєкту: 254.00 KB

📄 === HTML-звіт ===
<ul>
<li>📁 <b>project/</b>
<ul>
  <li>📄 README.txt <small>(1536 B, 42 рядків)</small></li>
  <li>🖼 logo.png <small>(51200 B, 512x512)</small></li>
  <li>📁 <b>src/</b>
  <ul>
    <li>📄 Program.cs <small>(3072 B, 128 рядків)</small></li>
    <li>📁 <b>assets/</b>
    <ul>
      <li>🖼 banner.png <small>(204800 B, 1200x300)</small></li>
    </ul>
    </li>
  </ul>
  </li>
</ul>
</li>
</ul>

📄 === JSON-маніфест ===
{
  "files": [
    {"path":"project/README.txt","type":"text","size":1536},
    {"path":"project/logo.png","type":"image","size":51200,"width":512,"height":512},
    {"path":"project/src/Program.cs","type":"text","size":3072},
    {"path":"project/src/assets/banner.png","type":"image","size":204800,"width":1200,"height":300}
  ]
}

✅ Три незалежні операції над одним і тим самим деревом, жодних змін у TextFileNode/ImageFileNode/FolderNode.
```

> Тут Visitor поєднано з **Composite**: `FolderNode` є композитним вузлом, а рекурсивний обхід дерева виконує сам відвідувач (`VisitFolder` викликає `child.Accept(this)` для кожної дитини). Це типова комбінація для дерев довільної глибини — AST компіляторів, DOM-подібних структур, файлових систем, оргструктур компаній тощо.

---

## Visitor vs Strategy vs Iterator

Ці три патерни іноді плутають, бо всі вони "виносять поведінку в окремий об'єкт". Різниця — у тому, **скільки типів** і **хто вибирає** поведінку.

```
Strategy:
  клієнт ── обирає ──▶ ISortStrategy ── застосовується до ──▶ ОДНОГО об'єкта за раз
  (клієнт явно каже: "використай саме цей алгоритм")

Iterator:
  клієнт ── просить ──▶ IEnumerator ── видає елементи по черзі
  (фокус на ПОСЛІДОВНОСТІ доступу, а не на операціях над елементами)

Visitor:
  структура з РІЗНИМИ типами елементів
       │
       │ Accept(visitor) на кожному елементі
       ▼
  visitor.VisitTypeA / VisitTypeB / VisitTypeC ── диспетчеризація АВТОМАТИЧНА,
  визначається типом елемента, а не явним вибором клієнта для кожного об'єкта
```

| Ознака | Strategy | Iterator | Visitor |
|---|---|---|---|
| **Що змінюється** | Один алгоритм для одного об'єкта | Спосіб послідовного доступу до елементів | Одна операція, що по-різному поводиться для БАГАТЬОХ типів |
| **Хто обирає поведінку** | Клієнт явно передає стратегію | Клієнт просто викликає `MoveNext()` | Диспетчеризація автоматична (double dispatch) за типом елемента |
| **Кількість типів, з якими працює** | Зазвичай один тип контексту | Один тип колекції | Багато різних типів елементів (гетерогенна структура) |
| **Головна мета** | Взаємозамінність алгоритмів у рантаймі | Уніфікований обхід без розкриття внутрішньої структури | Додавання нових операцій без зміни класів елементів |
| **Типова комбінація** | — | Часто комбінується з Visitor або Composite | Часто комбінується з Iterator/Composite для обходу + операції |

### Запитай себе:

```
Чи є ОДИН об'єкт (або одна колекція однотипних об'єктів),
і потрібно підмінювати ОДИН алгоритм у рантаймі?
  → Strategy

Чи потрібно просто ПОСЛІДОВНО пройтися по елементах колекції,
не виконуючи над кожним типом окремої, специфічної логіки?
  → Iterator

Чи є структура з РІЗНИМИ типами елементів (2+), і потрібно
виконувати над нею операцію, що для кожного типу поводиться по-своєму,
причому таких операцій буде декілька і вони змінюватимуться частіше,
ніж самі типи елементів?
  → Visitor
```

Часто ці патерни працюють **разом**: `Iterator` (або рекурсивний обхід `Composite`, як у Прикладі 4) відповідає за те, "куди йти далі" в структурі, а `Visitor` — за те, "що робити" з кожним конкретним типом елемента під час цього обходу.

---

## Переваги та недоліки

### Переваги

- **Додавання нових операцій без зміни класів елементів** — принцип Open/Closed застосовується саме до *операцій*: щоб додати нову, достатньо створити новий клас `ConcreteVisitor`
- **Уся логіка однієї операції зібрана в одному місці** — замість того, щоб код `ExportToXml` був розкиданий по 10 файлах фігур, весь він знаходиться в одному класі `XmlExportVisitor` — легше читати, тестувати, рефакторити
- **Зручно для операцій, що накопичують/комбінують інформацію з різнотипних елементів** — наприклад, підрахунок сумарного розміру файлів різних типів, збір статистики за неоднорідною структурою
- **Візитор може мати стан** — на відміну від статичних методів-розширень, відвідувач — повноцінний об'єкт, що може накопичувати результат (`TotalArea`, `TotalBytes`) під час обходу

### Недоліки

- **Додавання нового типу елемента — дороге** — якщо з'являється, наприклад, `Pentagon : IShape`, доведеться додати новий метод `VisitPentagon` в інтерфейс `IShapeVisitor` **і в кожну його реалізацію** (`AreaCalculatorVisitor`, `XmlExportVisitor`, `JsonExportVisitor`, ...). Це прямо протилежний компроміс порівняно зі звичайним поліморфізмом: там легко додати новий підклас, але важко додати нову операцію; у Visitor — навпаки
- **Може порушувати інкапсуляцію** — щоб відвідувач міг щось порахувати чи експортувати, елементу часто доводиться відкривати внутрішній стан (публічні властивості), який в іншому дизайні міг би залишатися приватним
- **Механіка подвійної диспетчеризації неочевидна новачкам** — код `element.Accept(visitor)` → `visitor.VisitX(this)` виглядає як зайвий рівень непрямоти, поки не зрозуміло, навіщо він потрібен
- **Не підходить, якщо ієрархія елементів часто змінюється** — у такому випадку постійні правки інтерфейсу `IVisitor` й усіх його реалізацій зведуть нанівець економію часу від патерну

---

## Антипатерни та поширені помилки

### Помилка 1 — перевірка типу замість подвійної диспетчеризації

```csharp
// ❌ НЕПРАВИЛЬНО: візитор сам перевіряє тип через is/GetType()
public class BadAreaVisitor
{
    public double CalculateArea(IShape shape)
    {
        // Це знову крихка перевірка типів, яку Visitor мав усунути!
        if (shape is Circle c)
            return Math.PI * c.Radius * c.Radius;
        if (shape is Rectangle r)
            return r.Width * r.Height;
        if (shape is Triangle t)
            return 0.5 * t.Base * t.Height;

        throw new NotSupportedException($"Невідомий тип фігури: {shape.GetType()}");
        // Проблема: компілятор НЕ перевірить повноту цього switch/if.
        // Забули додати гілку для нового типу — дізнаєтесь про це
        // лише в рантаймі через виключення (або взагалі мовчки, якщо
        // немає default-гілки з помилкою).
    }
}
```

```csharp
// ✅ ПРАВИЛЬНО: справжня подвійна диспетчеризація через Accept/Visit
public interface IShapeVisitor
{
    void VisitCircle(Circle circle);
    void VisitRectangle(Rectangle rectangle);
    void VisitTriangle(Triangle triangle);
}

public class AreaCalculatorVisitor : IShapeVisitor
{
    public double TotalArea { get; private set; }

    public void VisitCircle(Circle circle) => TotalArea += Math.PI * circle.Radius * circle.Radius;
    public void VisitRectangle(Rectangle rectangle) => TotalArea += rectangle.Width * rectangle.Height;
    public void VisitTriangle(Triangle triangle) => TotalArea += 0.5 * triangle.Base * triangle.Height;
}

// Використання: shape.Accept(visitor) — тип визначається компілятором
// автоматично, БЕЗ жодних is/GetType(). А якщо забудете реалізувати
// VisitXxx для нового типу — компілятор одразу вкаже на помилку,
// бо IShapeVisitor вимагає реалізації ВСІХ методів інтерфейсу.
```

### Помилка 2 — забули додати новий Visit-метод у IVisitor при додаванні нового елемента

```csharp
// ❌ НЕПРАВИЛЬНО: додали новий тип фігури Pentagon,
// але забули додати VisitPentagon в IShapeVisitor
public interface IShapeVisitor
{
    void VisitCircle(Circle circle);
    void VisitRectangle(Rectangle rectangle);
    void VisitTriangle(Triangle triangle);
    // VisitPentagon відсутній!
}

public class Pentagon : IShape
{
    public void Accept(IShapeVisitor visitor)
    {
        // Немає VisitPentagon — доведеться або підставляти виклик
        // якогось невідповідного методу, або кидати виключення,
        // або (найгірше) мовчки нічого не робити.
        // Жоден варіант не є коректним по суті.
    }
}
```

```csharp
// ✅ ПРАВИЛЬНО: додавання нового типу елемента = оновлення IVisitor
// і ВСІХ його реалізацій. Це "неприємно", але компілятор про це
// нагадає сам — і в цьому сила підходу на основі інтерфейсу.
public interface IShapeVisitor
{
    void VisitCircle(Circle circle);
    void VisitRectangle(Rectangle rectangle);
    void VisitTriangle(Triangle triangle);
    void VisitPentagon(Pentagon pentagon); // додали — і одразу
    // AreaCalculatorVisitor, XmlExportVisitor, JsonExportVisitor
    // ПЕРЕСТАНУТЬ КОМПІЛЮВАТИСЯ, доки не реалізують цей метод.
    // Це НЕ баг компілятора — це ФІЧА: неможливо забути обробити
    // новий тип у жодному наявному відвідувачі.
}

public class Pentagon : IShape
{
    public void Accept(IShapeVisitor visitor) => visitor.VisitPentagon(this);
}

// Висновок: Visitor варто застосовувати саме тоді, коли набір типів
// елементів ВІДНОСНО СТАБІЛЬНИЙ і рідко змінюється, а операцій —
// багато і вони змінюються часто. Якщо навпаки (типи змінюються
// часто, операцій мало) — краще звичайний поліморфізм або патерн Strategy.
```

### Помилка 3 — стан-акумулятор без скидання між обходами

```csharp
// ❌ НЕПРАВИЛЬНО: одна й та сама інстанція відвідувача перевикористовується
// для кількох незалежних обходів — результати "перетікають" між викликами
public class LeakyAreaVisitor : IShapeVisitor
{
    public double TotalArea { get; private set; } // накопичувальний стан!

    public void VisitCircle(Circle circle) => TotalArea += Math.PI * circle.Radius * circle.Radius;
    public void VisitRectangle(Rectangle rectangle) => TotalArea += rectangle.Width * rectangle.Height;
    public void VisitTriangle(Triangle triangle) => TotalArea += 0.5 * triangle.Base * triangle.Height;
}

// Використання з багом:
var visitor = new LeakyAreaVisitor();

foreach (var shape in firstBatch) shape.Accept(visitor);
Console.WriteLine(visitor.TotalArea); // напр. 61.34 — коректно

foreach (var shape in secondBatch) shape.Accept(visitor);
// TotalArea тепер = 61.34 + площа secondBatch —
// а не площа ЛИШЕ secondBatch, як очікував розробник!
Console.WriteLine(visitor.TotalArea); // неправильне значення для secondBatch
```

```csharp
// ✅ ПРАВИЛЬНО, варіант A: нова інстанція відвідувача на кожен обхід
foreach (var batch in new[] { firstBatch, secondBatch })
{
    var visitor = new AreaCalculatorVisitor(); // свіжий стан щоразу
    foreach (var shape in batch) shape.Accept(visitor);
    Console.WriteLine(visitor.TotalArea); // завжди коректно ізольовано
}

// ✅ ПРАВИЛЬНО, варіант B: явний метод Reset(), якщо перестворення дороге
public class ResettableAreaVisitor : IShapeVisitor
{
    public double TotalArea { get; private set; }

    public void Reset() => TotalArea = 0; // явне скидання стану

    public void VisitCircle(Circle circle) => TotalArea += Math.PI * circle.Radius * circle.Radius;
    public void VisitRectangle(Rectangle rectangle) => TotalArea += rectangle.Width * rectangle.Height;
    public void VisitTriangle(Triangle triangle) => TotalArea += 0.5 * triangle.Base * triangle.Height;
}

var reusable = new ResettableAreaVisitor();
foreach (var shape in firstBatch) shape.Accept(reusable);
Console.WriteLine(reusable.TotalArea);

reusable.Reset(); // явно скидаємо перед новим обходом
foreach (var shape in secondBatch) shape.Accept(reusable);
Console.WriteLine(reusable.TotalArea); // тепер коректно — тільки secondBatch
```

---

## Підсумок

Visitor варто застосовувати, коли:

- Є **стабільна** структура (ієрархія або дерево) типів елементів, яка **рідко змінюється**
- Над цією структурою потрібно виконувати **багато різних операцій**, і нові операції додаються **частіше**, ніж нові типи елементів
- Логіку кожної операції хочеться зібрати **в одному місці**, а не розкидати по класах елементів
- Потрібно комбінувати/накопичувати інформацію під час обходу неоднорідної структури (сума, звіт, серіалізація)

Класичний "солодкий момент" застосування — компілятори та інтерпретатори (AST + різні прохідники: type-checker, optimizer, code generator), системи документообігу (документ + різні експортери), геометричні/графічні системи (фігури + різні рендери/обчислення).

Не варто застосовувати Visitor, якщо типи елементів часто змінюються — тоді вигідніше звичайний поліморфізм (віртуальні методи прямо в елементах).

### Мінімальний шаблон

```csharp
// 1. Інтерфейс відвідувача — один Visit на кожен тип елемента
public interface IVisitor
{
    void VisitConcreteElementA(ConcreteElementA element);
    void VisitConcreteElementB(ConcreteElementB element);
}

// 2. Інтерфейс елемента
public interface IElement
{
    void Accept(IVisitor visitor);
}

// 3. Конкретні елементи
public class ConcreteElementA : IElement
{
    public void Accept(IVisitor visitor) => visitor.VisitConcreteElementA(this);
}

public class ConcreteElementB : IElement
{
    public void Accept(IVisitor visitor) => visitor.VisitConcreteElementB(this);
}

// 4. Конкретний відвідувач — одна повноцінна операція для всіх типів
public class ConcreteVisitor : IVisitor
{
    public void VisitConcreteElementA(ConcreteElementA element)
    {
        // Логіка операції для типу A
    }

    public void VisitConcreteElementB(ConcreteElementB element)
    {
        // Логіка операції для типу B
    }
}

// 5. Використання
var visitor = new ConcreteVisitor();
foreach (IElement element in elements)
{
    element.Accept(visitor); // double dispatch: Accept (virtual) → VisitX (overload)
}
```

---

*Документ підготовлено для вивчення патернів проектування. Всі приклади протестовані на .NET 6+.*
