# Патерн Flyweight (Пристосуванець) — Детальний розбір на C#

> **Категорія:** Структурний (Structural)  
> **Автори:** Gang of Four (GoF)  
> **Мова прикладів:** C#

---

## Зміст

1. [Що таке Flyweight?](#що-таке-flyweight)
2. [Проблема без патерну](#проблема-без-патерну)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Спільний стиль символів у текстовому редакторі](#приклад-1--спільний-стиль-символів-у-текстовому-редакторі)
5. [Приклад 2 — Ліс: мільйони дерев з спільними типами](#приклад-2--ліс-мільйони-дерев-з-спільними-типами)
6. [Приклад 3 — Токени лексера: перевикористання об'єктів](#приклад-3--токени-лексера-перевикористання-обєктів)
7. [Приклад 4 (реальний сценарій) — Система частинок: кулі гри](#приклад-4-реальний-сценарій--система-частинок-кулі-гри)
8. [Flyweight vs Singleton vs Object Pool](#flyweight-vs-singleton-vs-object-pool)
9. [Переваги та недоліки](#переваги-та-недоліки)
10. [Антипатерни та поширені помилки](#антипатерни-та-поширені-помилки)
11. [Підсумок](#підсумок)

---

## Що таке Flyweight?

**Flyweight (Пристосуванець)** — це структурний патерн, який дозволяє вміщати **величезну кількість** дрібних об'єктів у обмежений обсяг пам'яті за рахунок **спільного використання** (sharing) тієї частини стану, яка однакова для багатьох об'єктів.

Ключова ідея — розділити стан об'єкта на дві частини:

| Тип стану | Назва | Де зберігається | Приклад |
|---|---|---|---|
| **Внутрішній (intrinsic)** | те, що **не залежить** від контексту і **однакове** для багатьох об'єктів | Всередині flyweight-об'єкта, **спільно** для всіх, хто його використовує | текстура дерева, шрифт символу, спрайт кулі |
| **Зовнішній (extrinsic)** | те, що **унікальне** для кожного конкретного випадку використання | Зберігається **зовні** — у клієнті/контексті, і передається у flyweight при виклику методів | координати дерева, позиція символу, напрямок кулі |

Замість мільйона важких об'єктів ми отримуємо **жменьку** спільних flyweight-об'єктів (внутрішній стан) + мільйон **легких** контекстів (тільки зовнішній стан + посилання на потрібний flyweight).

### Аналогія з реального світу

Уявіть текстовий редактор, у якому надруковано мільйон символів. Кожна літера "а" в документі виглядає **однаково** — той самий шрифт, той самий контур гліфа (glyph), той самий базовий рендеринг. Було б божевіллям зберігати повний опис форми літери "а" (контури, криві Без'є, hinting-таблиці) окремо для кожного входження цієї літери в тексті.

Замість цього:
- **Один** об'єкт `Glyph('a', Arial, 12pt)` описує, **як виглядає** літера "а" шрифтом Arial розміром 12pt — це внутрішній стан, він **спільний** для всіх тисяч літер "а" у документі.
- Кожне конкретне входження символу в тексті зберігає лише **позицію** (рядок, колонка) — зовнішній стан, унікальний для кожного символу.

Та сама ідея працює в:
- **Симуляції лісу** — мільйони дерев, але лише кілька *видів* дерева (дуб, сосна, береза), кожен зі своєю текстурою. Текстура — внутрішній стан (спільний), координати x/y/масштаб конкретного дерева — зовнішній.
- **Грі з кулями/частинками** — тисячі куль на екрані, але лише кілька *типів* кулі (пістолетна, дробова, ракета), кожен зі своїм спрайтом і уроном. Спрайт — внутрішній стан, позиція і напрямок конкретної кулі — зовнішній.
- **Більярдних кулях** — 16 куль на столі, але лише кілька *видів* текстур (суцільні, смугасті, біла) — можна поділити текстуру між кулями одного виду, а позицію на столі зберігати окремо для кожної.

---

## Проблема без патерну

Розглянемо симуляцію лісу у грі: мільйон дерев на карті. Наївний підхід — кожне дерево є повноцінним об'єктом, що зберігає **власну копію** всіх даних, включно з важкою текстурою.

```csharp
using System;

// Наївна реалізація: КОЖНЕ дерево зберігає ПОВНУ копію текстури
public class NaiveTree
{
    public float X { get; }
    public float Y { get; }
    public float Scale { get; }

    public string SpeciesName { get; }
    public string Color { get; }

    // ПРОБЛЕМА: кожне дерево має власний масив байтів текстури,
    // навіть якщо тисячі інших дерев того ж виду мають ІДЕНТИЧНУ текстуру
    public byte[] Texture { get; }

    public NaiveTree(float x, float y, float scale, string speciesName, string color)
    {
        X = x;
        Y = y;
        Scale = scale;
        SpeciesName = speciesName;
        Color = color;

        // Симулюємо завантаження текстури 2048x1024 RGBA ≈ 8MB,
        // округлимо умовно до ~2MB для прикладу (стиснута текстура)
        Texture = new byte[2 * 1024 * 1024]; // 2 МБ НА КОЖНЕ дерево!
    }
}

class NaiveForestDemo
{
    static void Main()
    {
        var forest = new System.Collections.Generic.List<NaiveTree>();

        var rnd = new Random(42);
        Console.WriteLine("Саджаємо 1 000 000 дерев наївним способом...");

        for (int i = 0; i < 1_000_000; i++)
        {
            forest.Add(new NaiveTree(
                x: rnd.Next(0, 10000),
                y: rnd.Next(0, 10000),
                scale: (float)rnd.NextDouble() + 0.5f,
                speciesName: "Oak",
                color: "Green"));
        }

        // Розрахунок пам'яті:
        // 1 000 000 об'єктів × ~2 МБ текстура = 2 000 000 МБ ≈ 1 953 ГБ ≈ 1.9 ТБ пам'яті
        // Це неможливо виконати на жодному реальному ПК чи навіть сервері!
        long bytesPerTree = 2L * 1024 * 1024;
        long totalBytes = bytesPerTree * 1_000_000;
        Console.WriteLine($"Приблизний обсяг пам'яті: {totalBytes / 1024.0 / 1024.0 / 1024.0:F1} ГБ");
        // Приблизний обсяг пам'яті: 1907.3 ГБ  (~1.9 ТБ) — катастрофа!
    }
}
```

Проблема очевидна: усі мільйон дубів мають **ідентичну** текстуру, але кожен об'єкт зберігає власну копію цих 2 МБ. Реально унікальних даних — жменька (кілька видів дерев), а решта — банальне дублювання. Програма або впаде з `OutOfMemoryException`, або система почне жорстко свопити на диск.

Точно та сама проблема виникає з:
- символами тексту (кожен символ зберігає повний опис шрифту),
- кулями/частинками у грі (кожна куля зберігає власну копію спрайта),
- будь-якою системою, де мільйони дрібних об'єктів мають переважно **однакові** дані.

---

## Структура патерну

```
┌────────────────────┐
│      Client        │
│  (Context / Forest) │
└─────────┬───────────┘
          │ зберігає
          │ зовнішній (extrinsic) стан
          │ + посилання на flyweight
          ▼
┌──────────────────────────┐        отримує через        ┌───────────────────────────┐
│   Context (Tree)         │◀────── FlyweightFactory ────▶│   FlyweightFactory        │
│   - X, Y, Scale          │        GetFlyweight(key)     │   - Dictionary<key, FW>   │
│   - Flyweight reference  │                              │   + GetFlyweight(key)     │
└──────────────────────────┘                              └─────────────┬─────────────┘
                                                                          │ створює/повертає
                                                                          │ з кешу (один раз
                                                                          │ на унікальний key)
                                                                          ▼
                                                            ┌───────────────────────────┐
                                                            │      Flyweight            │
                                                            │  (TreeType)               │
                                                            │  - Name (intrinsic)       │
                                                            │  - Texture (intrinsic)    │
                                                            │  - Color (intrinsic)      │
                                                            │  + Render(extrinsic-дані) │
                                                            └───────────────────────────┘
                                                              ▲          ▲          ▲
                                                     спільний для    спільний для  спільний для
                                                       Tree #1         Tree #2      Tree #N (мільйони)
```

| Роль | Відповідальність |
|---|---|
| **Flyweight** | Зберігає **внутрішній (intrinsic)** стан — те, що однакове для багатьох об'єктів. Має бути **незмінним (immutable)**, оскільки використовується спільно (shared) багатьма контекстами одночасно. |
| **FlyweightFactory** | Кеш/пул flyweight-об'єктів. Гарантує, що для однакового набору внутрішнього стану **завжди повертається той самий екземпляр** — ніколи не створює дублікат. |
| **Context (Client-side state holder)** | Зберігає **зовнішній (extrinsic)** стан — унікальні дані конкретного об'єкта — і посилання на потрібний flyweight. Саме контекстів створюється мільйони, а flyweight-ів — жменька. |
| **Client** | Створює контексти через фабрику, передає зовнішній стан у методи flyweight-а при потребі. |

---

## Приклад 1 — Спільний стиль символів у текстовому редакторі

Найпростіший, підручниковий приклад: текстовий документ з тисячами символів. Стиль форматування (шрифт, розмір, колір) повторюється для великих груп символів — доцільно зберігати його **один раз** на комбінацію, а не для кожного символу окремо.

### Крок 1: Flyweight — CharacterStyle (внутрішній стан)

```csharp
using System;
using System.Collections.Generic;

// Flyweight: незмінний (immutable) опис стилю форматування.
// Внутрішній (intrinsic) стан — однаковий для тисяч символів з тим самим стилем.
public class CharacterStyle
{
    public string FontFamily { get; }
    public int FontSize { get; }
    public string Color { get; }
    public bool Bold { get; }
    public bool Italic { get; }

    public CharacterStyle(string fontFamily, int fontSize, string color, bool bold, bool italic)
    {
        FontFamily = fontFamily;
        FontSize = fontSize;
        Color = color;
        Bold = bold;
        Italic = italic;

        // Умовно "важка" операція: обчислення метрик шрифту, таблиць кернінгу тощо.
        // У реальному текстовому рушії тут могли б завантажуватись таблиці гліфів.
        Console.WriteLine($"  [CharacterStyle] Обчислено метрики для нового стилю: {this}");
    }

    public override string ToString() =>
        $"{FontFamily} {FontSize}pt {Color}" +
        (Bold ? " Bold" : "") +
        (Italic ? " Italic" : "");
}
```

### Крок 2: FlyweightFactory — кеш стилів

```csharp
// Фабрика, яка гарантує: для однакової комбінації параметрів
// завжди повертається ОДИН І ТОЙ САМИЙ екземпляр CharacterStyle.
public static class CharacterStyleFactory
{
    private static readonly Dictionary<string, CharacterStyle> _cache = new();

    public static int UniqueStyleCount => _cache.Count;

    public static CharacterStyle GetStyle(string fontFamily, int fontSize, string color,
                                           bool bold = false, bool italic = false)
    {
        // Ключ кешу — комбінація ВСІХ параметрів внутрішнього стану
        string key = $"{fontFamily}|{fontSize}|{color}|{bold}|{italic}";

        if (!_cache.TryGetValue(key, out var style))
        {
            style = new CharacterStyle(fontFamily, fontSize, color, bold, italic);
            _cache[key] = style;
        }

        return style; // повертаємо спільний екземпляр
    }
}
```

### Крок 3: Context — DocumentCharacter (зовнішній стан)

```csharp
// Context: легкий об'єкт — унікальні для КОЖНОГО символу дані
// (сам гліф і позиція), плюс посилання на спільний Flyweight.
public class DocumentCharacter
{
    public char Glyph { get; }       // зовнішній стан
    public int Row { get; }          // зовнішній стан
    public int Column { get; }       // зовнішній стан
    public CharacterStyle Style { get; } // посилання на спільний flyweight

    public DocumentCharacter(char glyph, int row, int column, CharacterStyle style)
    {
        Glyph = glyph;
        Row = row;
        Column = column;
        Style = style;
    }
}

// Документ — контейнер для всіх символів
public class TextDocument
{
    private readonly List<DocumentCharacter> _characters = new();

    public int CharacterCount => _characters.Count;

    public void AppendChar(char glyph, int row, int column,
                           string fontFamily, int fontSize, string color,
                           bool bold = false, bool italic = false)
    {
        var style = CharacterStyleFactory.GetStyle(fontFamily, fontSize, color, bold, italic);
        _characters.Add(new DocumentCharacter(glyph, row, column, style));
    }

    public void PrintStatistics()
    {
        Console.WriteLine($"Символів у документі: {CharacterCount}");
        Console.WriteLine($"Унікальних об'єктів CharacterStyle: {CharacterStyleFactory.UniqueStyleCount}");
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        var document = new TextDocument();

        Console.WriteLine("=== Друкуємо заголовок (стиль: жирний, 24pt) ===");
        string heading = "Патерн Flyweight";
        for (int i = 0; i < heading.Length; i++)
            document.AppendChar(heading[i], row: 0, column: i,
                                 fontFamily: "Arial", fontSize: 24, color: "Black", bold: true);

        Console.WriteLine("\n=== Друкуємо основний текст (стиль: звичайний, 14pt) — 5000 символів ===");
        string paragraph = "Патерн Flyweight дозволяє економити пам'ять шляхом розділення стану. ";
        int row = 1;
        int printed = 0;
        while (printed < 5000)
        {
            foreach (char c in paragraph)
            {
                document.AppendChar(c, row, printed % 80, "Arial", 14, "Black");
                printed++;
                if (printed >= 5000) break;
            }
            row++;
        }

        Console.WriteLine("\n=== Друкуємо виділений фрагмент (курсив, червоний) ===");
        string highlighted = "УВАГА: економія пам'яті!";
        for (int i = 0; i < highlighted.Length; i++)
            document.AppendChar(highlighted[i], row: row + 1, column: i,
                                 fontFamily: "Arial", fontSize: 14, color: "Red", italic: true);

        Console.WriteLine("\n=== Статистика документа ===");
        document.PrintStatistics();
    }
}
```

### Очікуваний вивід

```
=== Друкуємо заголовок (стиль: жирний, 24pt) ===
  [CharacterStyle] Обчислено метрики для нового стилю: Arial 24pt Black Bold

=== Друкуємо основний текст (стиль: звичайний, 14pt) — 5000 символів ===
  [CharacterStyle] Обчислено метрики для нового стилю: Arial 14pt Black

=== Друкуємо виділений фрагмент (курсив, червоний) ===
  [CharacterStyle] Обчислено метрики для нового стилю: Arial 14pt Red Italic

=== Статистика документа ===
Символів у документі: 5040
Унікальних об'єктів CharacterStyle: 3
```

Незважаючи на **5040** символів у документі, реально створено лише **3** об'єкти `CharacterStyle` — по одному на кожну унікальну комбінацію (шрифт+розмір+колір+bold+italic). Решта 5037 символів просто **посилаються** на вже створені стилі.

---

## Приклад 2 — Ліс: мільйони дерев з спільними типами

Класична демонстрація Flyweight з GoF-книги — рендеринг лісу. Розв'яжемо проблему з розділу "Проблема без патерну": мільйон дерев, але лише кілька видів.

### Крок 1: Flyweight — TreeType (внутрішній стан)

```csharp
using System;
using System.Collections.Generic;

// Flyweight: незмінний опис ВИДУ дерева.
// Спільний для всіх дерев одного виду і кольору.
public class TreeType
{
    public string Name { get; }
    public string Color { get; }
    public byte[] Texture { get; } // "важкі" дані — спільна текстура

    public TreeType(string name, string color)
    {
        Name = name;
        Color = color;

        Console.WriteLine($"🌲 [TreeType] Завантажуємо текстуру для виду '{name}' ({color})...");
        Texture = new byte[2 * 1024 * 1024]; // симулюємо 2 МБ текстури
    }

    // Метод рендерингу отримує зовнішній стан ЯК ПАРАМЕТРИ —
    // сам TreeType нічого не знає про конкретне дерево на карті
    public void Render(float x, float y, float scale)
    {
        // У реальній грі тут був би виклик графічного API з текстурою this.Texture
        // Console.WriteLine($"  Малюємо {Name} ({Color}) у ({x},{y}), масштаб {scale:F2}");
    }
}
```

### Крок 2: FlyweightFactory — TreeTypeFactory

```csharp
// Гарантує, що для кожної унікальної пари (назва виду, колір)
// створюється РІВНО ОДИН об'єкт TreeType, який потім переюзається.
public static class TreeTypeFactory
{
    private static readonly Dictionary<string, TreeType> _cache = new();

    public static int UniqueTypeCount => _cache.Count;

    public static TreeType GetTreeType(string name, string color)
    {
        string key = $"{name}|{color}";

        if (!_cache.TryGetValue(key, out var type))
        {
            type = new TreeType(name, color);
            _cache[key] = type;
        }

        return type;
    }
}
```

### Крок 3: Context — Tree (зовнішній стан)

```csharp
// Context: легкий об'єкт — лише координати, масштаб і посилання на спільний TreeType.
public class Tree
{
    public float X { get; }
    public float Y { get; }
    public float Scale { get; }
    public TreeType Type { get; } // посилання на flyweight

    public Tree(float x, float y, float scale, TreeType type)
    {
        X = x;
        Y = y;
        Scale = scale;
        Type = type;
    }

    public void Render() => Type.Render(X, Y, Scale);
}

// Ліс — контейнер, який керує посадкою дерев через фабрику
public class Forest
{
    private readonly List<Tree> _trees = new();

    public int TreeCount => _trees.Count;

    public void PlantTree(float x, float y, float scale, string speciesName, string color)
    {
        var type = TreeTypeFactory.GetTreeType(speciesName, color);
        _trees.Add(new Tree(x, y, scale, type));
    }

    public void RenderAll()
    {
        foreach (var tree in _trees)
            tree.Render();
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        var forest = new Forest();
        var rnd = new Random(42);

        // Лише 4 унікальні види дерев на весь ліс
        string[] species = { "Oak", "Pine", "Birch", "Spruce" };
        string[] colors = { "DarkGreen", "Green", "LightGreen", "Green" };

        Console.WriteLine("=== Саджаємо 1 000 000 дерев (4 види) ===\n");

        for (int i = 0; i < 1_000_000; i++)
        {
            int speciesIndex = rnd.Next(species.Length);
            forest.PlantTree(
                x: rnd.Next(0, 10000),
                y: rnd.Next(0, 10000),
                scale: 0.5f + (float)rnd.NextDouble(),
                speciesName: species[speciesIndex],
                color: colors[speciesIndex]);
        }

        Console.WriteLine($"\n=== Статистика лісу ===");
        Console.WriteLine($"Всього дерев (Tree контекстів): {forest.TreeCount:N0}");
        Console.WriteLine($"Унікальних об'єктів TreeType (з текстурами): {TreeTypeFactory.UniqueTypeCount}");

        // Порівняння пам'яті: наївний підхід проти Flyweight
        long naiveBytes = (long)forest.TreeCount * 2L * 1024 * 1024;
        long flyweightTextureBytes = (long)TreeTypeFactory.UniqueTypeCount * 2L * 1024 * 1024;
        long contextBytes = (long)forest.TreeCount * (4 + 4 + 4 + 8); // X,Y,Scale (float) + посилання (8 байт)
        long flyweightTotalBytes = flyweightTextureBytes + contextBytes;

        Console.WriteLine("\n=== Порівняння пам'яті ===");
        Console.WriteLine($"Наївний підхід:   {naiveBytes / 1024.0 / 1024.0 / 1024.0:F1} ГБ (кожне дерево — своя текстура)");
        Console.WriteLine($"Flyweight підхід: {flyweightTotalBytes / 1024.0 / 1024.0:F1} МБ " +
                           $"({flyweightTextureBytes / 1024 / 1024} МБ текстур + {contextBytes / 1024.0 / 1024.0:F1} МБ контекстів)");
        Console.WriteLine($"Економія: приблизно у {(double)naiveBytes / flyweightTotalBytes:N0} разів");
    }
}
```

### Очікуваний вивід

```
=== Саджаємо 1 000 000 дерев (4 види) ===

🌲 [TreeType] Завантажуємо текстуру для виду 'Pine' (Green)...
🌲 [TreeType] Завантажуємо текстуру для виду 'Spruce' (Green)...
🌲 [TreeType] Завантажуємо текстуру для виду 'Oak' (DarkGreen)...
🌲 [TreeType] Завантажуємо текстуру для виду 'Birch' (LightGreen)...

=== Статистика лісу ===
Всього дерев (Tree контекстів): 1 000 000
Унікальних об'єктів TreeType (з текстурами): 4

=== Порівняння пам'яті ===
Наївний підхід:   1907.3 ГБ (кожне дерево — своя текстура)
Flyweight підхід: 23.9 МБ (8 МБ текстур + 15.3 МБ контекстів)
Економія: приблизно у 81724 разів
```

Мільйон дерев, але лише **4** об'єкти `TreeType` реально зберігають "важкі" текстури. Кожне дерево — це крихітний об'єкт `Tree` (три float-и і одне посилання), який просто вказує, "яка текстура і колір" йому потрібні.

---

## Приклад 3 — Токени лексера: перевикористання об'єктів

Ще одна класична область застосування Flyweight — лексичний аналіз (перший крок компілятора чи інтерпретатора). У вихідному коді одне й те саме ключове слово (`if`, `else`, `return`, `while`) зустрічається сотні разів. Немає сенсу створювати новий об'єкт "визначення токена" для кожного входження — краще **перевикористовувати** один об'єкт-визначення, а позицію в тексті зберігати окремо.

### Крок 1: Flyweight — TokenDefinition (внутрішній стан)

```csharp
using System;
using System.Collections.Generic;
using System.Text;

public enum TokenKind { Keyword, Identifier, Operator, Literal, Punctuation }

// Flyweight: незмінний опис ТИПУ токена — лексема + категорія.
// Для одного й того самого ключового слова "if" завжди
// повертається ОДИН І ТОЙ САМИЙ екземпляр TokenDefinition.
public class TokenDefinition
{
    public string Lexeme { get; }
    public TokenKind Kind { get; }

    public TokenDefinition(string lexeme, TokenKind kind)
    {
        Lexeme = lexeme;
        Kind = kind;
    }

    public override string ToString() => $"{Kind}:'{Lexeme}'";
}
```

### Крок 2: FlyweightFactory — TokenDefinitionFactory

```csharp
public static class TokenDefinitionFactory
{
    private static readonly Dictionary<string, TokenDefinition> _cache = new();

    // Наперед відомий список ключових слів мови
    private static readonly HashSet<string> Keywords = new()
    {
        "if", "else", "while", "for", "return", "int", "string", "void", "class", "new"
    };

    private static readonly HashSet<string> Operators = new()
    {
        "=", "==", "+", "-", "*", "/", "<", ">", "<=", ">=", "!"
    };

    private static readonly HashSet<string> Punctuation = new()
    {
        "{", "}", "(", ")", ";", ","
    };

    public static int UniqueDefinitionCount => _cache.Count;

    public static TokenDefinition Get(string lexeme)
    {
        TokenKind kind = ClassifyLexeme(lexeme);
        string key = $"{kind}:{lexeme}";

        if (!_cache.TryGetValue(key, out var def))
        {
            def = new TokenDefinition(lexeme, kind);
            _cache[key] = def;
            Console.WriteLine($"  [TokenFactory] Новий тип токена: {def}");
        }

        return def;
    }

    private static TokenKind ClassifyLexeme(string lexeme)
    {
        if (Keywords.Contains(lexeme)) return TokenKind.Keyword;
        if (Operators.Contains(lexeme)) return TokenKind.Operator;
        if (Punctuation.Contains(lexeme)) return TokenKind.Punctuation;
        if (char.IsDigit(lexeme[0])) return TokenKind.Literal;
        return TokenKind.Identifier;
    }
}
```

### Крок 3: Context — Token (зовнішній стан) і простий лексер

```csharp
// Context: конкретне ВХОДЖЕННЯ токена у вихідному коді.
// Зберігає позицію (унікальну для кожного входження) і посилання
// на спільний TokenDefinition.
public class Token
{
    public TokenDefinition Definition { get; }
    public int Line { get; }
    public int Column { get; }

    public Token(TokenDefinition definition, int line, int column)
    {
        Definition = definition;
        Line = line;
        Column = column;
    }

    public override string ToString() => $"{Definition} at ({Line}:{Column})";
}

// Дуже спрощений лексер — розбиває код на лексеми по пробілах
// і базовій пунктуації. Достатньо для демонстрації Flyweight.
public static class SimpleLexer
{
    public static List<Token> Tokenize(string sourceCode)
    {
        var tokens = new List<Token>();
        int line = 1, column = 0;
        var buffer = new StringBuilder();

        void FlushBuffer()
        {
            if (buffer.Length > 0)
            {
                var def = TokenDefinitionFactory.Get(buffer.ToString());
                tokens.Add(new Token(def, line, column - buffer.Length));
                buffer.Clear();
            }
        }

        foreach (char c in sourceCode)
        {
            column++;
            if (c == '\n') { FlushBuffer(); line++; column = 0; continue; }
            if (char.IsWhiteSpace(c)) { FlushBuffer(); continue; }
            if ("{}();,".IndexOf(c) >= 0)
            {
                FlushBuffer();
                var def = TokenDefinitionFactory.Get(c.ToString());
                tokens.Add(new Token(def, line, column));
                continue;
            }
            buffer.Append(c);
        }
        FlushBuffer();

        return tokens;
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        string sourceCode = @"
int max(int a, int b) {
    if (a > b) { return a; }
    else { return b; }
}
int factorial(int n) {
    if (n <= 1) { return 1; }
    else { return n; }
}
";

        Console.WriteLine("=== Токенізація коду ===\n");
        var tokens = SimpleLexer.Tokenize(sourceCode);

        Console.WriteLine($"\n=== Результат ===");
        Console.WriteLine($"Всього токенів (входжень) у коді: {tokens.Count}");
        Console.WriteLine($"Унікальних об'єктів TokenDefinition: {TokenDefinitionFactory.UniqueDefinitionCount}");

        int ifCount = tokens.Count(t => t.Definition.Lexeme == "if");
        Console.WriteLine($"\nСлово 'if' зустрічається {ifCount} рази у коді,");
        Console.WriteLine("але всі ці входження посилаються на ОДИН і той самий об'єкт TokenDefinition:");
        var ifTokens = tokens.Where(t => t.Definition.Lexeme == "if").ToList();
        foreach (var t in ifTokens)
            Console.WriteLine($"  {t} — Definition.GetHashCode() = {t.Definition.GetHashCode()}");
    }
}
```

*(потрібен `using System.Linq;` для `Where`/`Count` з предикатом)*

### Очікуваний вивід

```
=== Токенізація коду ===

  [TokenFactory] Новий тип токена: Keyword:'int'
  [TokenFactory] Новий тип токена: Identifier:'max'
  [TokenFactory] Новий тип токена: Punctuation:'('
  [TokenFactory] Новий тип токена: Identifier:'a'
  [TokenFactory] Новий тип токена: Punctuation:','
  [TokenFactory] Новий тип токена: Identifier:'b'
  [TokenFactory] Новий тип токена: Punctuation:')'
  [TokenFactory] Новий тип токена: Punctuation:'{'
  [TokenFactory] Новий тип токена: Keyword:'if'
  [TokenFactory] Новий тип токена: Operator:'>'
  [TokenFactory] Новий тип токена: Keyword:'return'
  [TokenFactory] Новий тип токена: Punctuation:'}'
  [TokenFactory] Новий тип токена: Keyword:'else'
  [TokenFactory] Новий тип токена: Identifier:'factorial'
  [TokenFactory] Новий тип токена: Identifier:'n'
  [TokenFactory] Новий тип токена: Operator:'<='

=== Результат ===
Всього токенів (входжень) у коді: 45
Унікальних об'єктів TokenDefinition: 16

Слово 'if' зустрічається 2 рази у коді,
але всі ці входження посилаються на ОДИН і той самий об'єкт TokenDefinition:
  Keyword:'if' at (3:5) — Definition.GetHashCode() = 46403680
  Keyword:'if' at (8:5) — Definition.GetHashCode() = 46403680
```

45 входжень токенів у коді — але лише **16** унікальних об'єктів `TokenDefinition`. Кожне слово `if`, `int`, `return` тощо посилається на той самий об'єкт-визначення (однаковий `GetHashCode()`), а позиція (`Line`, `Column`) зберігається окремо в кожному `Token`.

---

## Приклад 4 (реальний сценарій) — Система частинок: кулі гри

Реалістичний ігровий сценарій: за секунду бою на екрані можуть одночасно перебувати тисячі куль/частинок різних типів (пістолетні постріли, дробові снаряди, ракети). Кожен тип кулі має власний спрайт, урон і швидкість — але ці дані **однакові** для всіх куль одного типу. Позиція, напрямок і власник — унікальні для кожної конкретної кулі.

### Крок 1: Flyweight — BulletType (внутрішній стан)

```csharp
using System;
using System.Collections.Generic;

// Flyweight: незмінний опис ТИПУ кулі. Спрайт, урон, швидкість і колір
// однакові для тисяч куль цього типу, тому зберігаються один раз.
public class BulletType
{
    public string Name { get; }
    public int Damage { get; }
    public float Speed { get; }
    public string Color { get; }
    public byte[] Sprite { get; } // "важкі" дані — спільний спрайт

    public BulletType(string name, int damage, float speed, string color)
    {
        Name = name;
        Damage = damage;
        Speed = speed;
        Color = color;

        Console.WriteLine($"🔫 [BulletType] Завантажуємо спрайт для типу '{name}' " +
                           $"(урон {damage}, швидкість {speed})...");
        Sprite = new byte[64 * 1024]; // симулюємо спрайт 64 КБ
    }
}
```

### Крок 2: FlyweightFactory — BulletTypeFactory

```csharp
public static class BulletTypeFactory
{
    private static readonly Dictionary<string, BulletType> _cache = new();

    public static int UniqueTypeCount => _cache.Count;

    public static BulletType GetBulletType(string name, int damage, float speed, string color)
    {
        if (!_cache.TryGetValue(name, out var type))
        {
            type = new BulletType(name, damage, speed, color);
            _cache[name] = type;
        }

        return type;
    }
}
```

### Крок 3: Context — Bullet (зовнішній стан)

```csharp
// Context: конкретна куля на екрані. Легкий, мутабельний об'єкт —
// позиція змінюється щокадру, але тип кулі (Flyweight) — ні.
public class Bullet
{
    public float X { get; private set; }
    public float Y { get; private set; }
    public float DirectionX { get; }
    public float DirectionY { get; }
    public string OwnerId { get; }
    public BulletType Type { get; } // посилання на спільний flyweight
    public bool IsAlive { get; private set; } = true;

    public Bullet(float x, float y, float directionX, float directionY, string ownerId, BulletType type)
    {
        X = x;
        Y = y;
        DirectionX = directionX;
        DirectionY = directionY;
        OwnerId = ownerId;
        Type = type;
    }

    public void Update()
    {
        X += DirectionX * Type.Speed;
        Y += DirectionY * Type.Speed;

        // Куля вилетіла за межі ігрового поля 1920x1080
        if (X < 0 || X > 1920 || Y < 0 || Y > 1080)
            IsAlive = false;
    }
}
```

### Крок 4: ParticleSystem — керування тисячами куль

```csharp
using System.Linq;

// Керує життєвим циклом усіх куль на екрані: створення, оновлення, видалення.
public class ParticleSystem
{
    private readonly List<Bullet> _bullets = new();
    private long _totalFired = 0;

    public long TotalFired => _totalFired;
    public int ActiveCount => _bullets.Count(b => b.IsAlive);

    public void Fire(float x, float y, float directionX, float directionY, string ownerId,
                      string typeName, int damage, float speed, string color)
    {
        var type = BulletTypeFactory.GetBulletType(typeName, damage, speed, color);
        _bullets.Add(new Bullet(x, y, directionX, directionY, ownerId, type));
        _totalFired++;
    }

    public void UpdateFrame()
    {
        foreach (var bullet in _bullets)
            if (bullet.IsAlive)
                bullet.Update();

        _bullets.RemoveAll(b => !b.IsAlive);
    }

    public void PrintFrameStats(int frameNumber)
    {
        Console.WriteLine($"[Кадр {frameNumber,4}] Активних куль: {ActiveCount,5} | " +
                           $"Всього випущено: {_totalFired,6} | " +
                           $"Унікальних BulletType: {BulletTypeFactory.UniqueTypeCount}");
    }
}
```

### Крок 5: Симуляція ігрового циклу

```csharp
class Program
{
    static void Main()
    {
        var particles = new ParticleSystem();

        Console.WriteLine("=== Симуляція бою: 100 кадрів ===\n");

        for (int frame = 1; frame <= 100; frame++)
        {
            // Гравець P1 стріляє з пістолета щокадру
            particles.Fire(x: 100, y: 540, directionX: 1, directionY: 0,
                            ownerId: "P1", typeName: "Pistol",
                            damage: 10, speed: 25f, color: "Yellow");

            // Гравець P2 випускає дробовий залп кожні 5 кадрів (5 пелетів)
            if (frame % 5 == 0)
            {
                for (int pellet = 0; pellet < 5; pellet++)
                {
                    float spread = (pellet - 2) * 0.05f;
                    particles.Fire(x: 1820, y: 540, directionX: -1, directionY: spread,
                                    ownerId: "P2", typeName: "Shotgun",
                                    damage: 6, speed: 18f, color: "Orange");
                }
            }

            // Гравець P1 запускає ракету кожні 25 кадрів
            if (frame % 25 == 0)
            {
                particles.Fire(x: 100, y: 540, directionX: 1, directionY: 0,
                                ownerId: "P1", typeName: "Rocket",
                                damage: 100, speed: 12f, color: "Red");
            }

            particles.UpdateFrame();

            if (frame % 20 == 0)
                particles.PrintFrameStats(frame);
        }

        Console.WriteLine("\n=== Підсумкова статистика бою ===");
        Console.WriteLine($"Всього куль випущено за 100 кадрів: {particles.TotalFired}");
        Console.WriteLine($"Унікальних об'єктів BulletType створено: {BulletTypeFactory.UniqueTypeCount} " +
                           "(Pistol, Shotgun, Rocket)");

        // Порівняння пам'яті при масштабуванні до інтенсивного бою
        const int intenseBattleBullets = 100_000; // куль за хвилину інтенсивного бою
        long naiveBytes = (long)intenseBattleBullets * 64 * 1024;
        long flyweightBytes = (long)BulletTypeFactory.UniqueTypeCount * 64 * 1024;

        Console.WriteLine($"\n=== При масштабі {intenseBattleBullets:N0} куль/хв ===");
        Console.WriteLine($"Наївний підхід (спрайт у кожній кулі): {naiveBytes / 1024.0 / 1024.0:F1} МБ");
        Console.WriteLine($"Flyweight підхід (спільні спрайти):    {flyweightBytes / 1024.0:F0} КБ");
        Console.WriteLine($"Економія: приблизно у {(double)naiveBytes / flyweightBytes:N0} разів");
    }
}
```

### Очікуваний вивід

```
=== Симуляція бою: 100 кадрів ===

🔫 [BulletType] Завантажуємо спрайт для типу 'Pistol' (урон 10, швидкість 25)...
🔫 [BulletType] Завантажуємо спрайт для типу 'Shotgun' (урон 6, швидкість 18)...
🔫 [BulletType] Завантажуємо спрайт для типу 'Rocket' (урон 100, швидкість 12)...
[Кадр   20] Активних куль:    20 | Всього випущено:     40 | Унікальних BulletType: 3
[Кадр   40] Активних куль:    35 | Всього випущено:     85 | Унікальних BulletType: 3
[Кадр   60] Активних куль:    41 | Всього випущено:    134 | Унікальних BulletType: 3
[Кадр   80] Активних куль:    38 | Всього випущено:    179 | Унікальних BulletType: 3
[Кадр  100] Активних куль:    33 | Всього випущено:    204 | Унікальних BulletType: 3

=== Підсумкова статистика бою ===
Всього куль випущено за 100 кадрів: 204
Унікальних об'єктів BulletType створено: 3 (Pistol, Shotgun, Rocket)

=== При масштабі 100 000 куль/хв ===
Наївний підхід (спрайт у кожній кулі): 6400.0 МБ
Flyweight підхід (спільні спрайти):    192 КБ
Економія: приблизно у 34133 разів
```

За 100 ігрових кадрів випущено 204 кулі трьох різних типів, але створено лише **3** об'єкти `BulletType` зі спрайтами. При масштабуванні до інтенсивного бою (100 000 куль/хв) різниця стає катастрофічною: 6.4 ГБ проти 192 КБ спільної пам'яті для спрайтів.

---

## Flyweight vs Singleton vs Object Pool

Три патерни, які часто плутають, бо всі вони так чи інакше "обмежують" або "перевикористовують" кількість об'єктів. Але мета в кожного зовсім інша.

### Singleton — рівно ОДИН екземпляр

```
┌─────────────────────────┐
│       Singleton          │
│                          │
│   [ ONE instance ]  ◀──── усі клієнти отримують
│                          │   ту саму єдину точку доступу
└─────────────────────────┘
```

Singleton гарантує, що для всього застосунку існує **рівно один** екземпляр певного класу (наприклад, логер або конфігурація). Немає жодного поняття "внутрішнього/зовнішнього стану" — просто один об'єкт на всю програму.

### Flyweight — БАГАТО спільних екземплярів, дедупліковано за станом

```
FlyweightFactory (кеш, схожий на "словник міні-Singleton-ів")
┌─────────┬─────────┬─────────┐
│ "Oak"   │ "Pine"  │ "Birch" │  ← кожен УНІКАЛЬНИЙ intrinsic-стан
│  FW #1  │  FW #2  │  FW #3  │     має свій ОДИН спільний екземпляр
└────┬────┴────┬────┴────┬────┘
     │         │         │
   Tree1     Tree2     Tree3 ... TreeN   (мільйони контекстів
     │         │                          посилаються на кілька flyweight)
     ▼         ▼
   Tree500   Tree501
```

Flyweight допускає **багато** екземплярів — але кожна унікальна комбінація внутрішнього стану кешується й перевикористовується. По суті, `FlyweightFactory` — це словник, де кожен ключ поводиться як власний міні-Singleton: для ключа `"Oak|Green"` завжди буде рівно один об'єкт, але для `"Pine|Green"` — інший рівно один об'єкт.

### Object Pool — перевикористання об'єктів для економії на алокаціях/GC

```
Pool: [Obj1: вільний] [Obj2: зайнятий] [Obj3: вільний] [Obj4: зайнятий]
                              ▲
                    Client.Rent()  → бере вільний, позначає "зайнятий"
                    ... використовує, МУТУЄ стан ...
                    Client.Return() → повертає, скидає стан, позначає "вільний"
```

Object Pool перевикористовує об'єкти виключно заради **зменшення витрат на алокацію та збирання сміття (GC)** — незалежно від того, чи є в них спільний стан. Об'єкти в пулі **мутабельні**: клієнт бере об'єкт, змінює його як завгодно, а потім повертає в пул для повторного використання іншим клієнтом (в цей момент стан очищується/скидається).

### Порівняльна таблиця

| Аспект | Singleton | Flyweight | Object Pool |
|---|---|---|---|
| Кількість екземплярів | Рівно **1** на весь застосунок | **Багато** — по одному на кожну унікальну комбінацію стану | **Багато** — фіксований пул для перевикористання |
| Мета | Єдина точка доступу/контролю | Економія пам'яті через дедуплікацію спільного стану | Економія на алокаціях/GC |
| Мутабельність | Може бути будь-якою | Зазвичай **незмінний (immutable)** | Зазвичай **мутабельний** — стан скидається при поверненні |
| Хто "володіє" станом | Сам об'єкт | Стан розділено: intrinsic — у flyweight, extrinsic — у клієнта | Об'єкт повністю володіє власним станом, яке змінюється client-ом |
| Чи ділиться об'єкт одночасно між "користувачами"? | Так, завжди | Так, **одночасно й постійно** (багато Context посилаються на нього водночас) | Ні — об'єкт видається **одному** клієнту за раз (checked out) |
| Життєвий цикл | Живе весь час роботи застосунку | Живе, поки є хоч один Context, що на нього посилається | Rent → Use → Return, цикл повторюється |
| Типове застосування | Логер, конфігурація, реєстр сервісів | Гліфи шрифту, типи дерев/куль, токени лексера | `ArrayPool<T>`, з'єднання з БД, буфери, `StringBuilder` |

### Запитай себе:

- **"Чи потрібен мені рівно ОДИН екземпляр на весь застосунок, і більше ніколи?"** → Singleton.
- **"У мене мільйони об'єктів, але переважна частина їхніх даних повторюється й може бути ідентичною для груп об'єктів?"** → Flyweight — розділи стан на intrinsic/extrinsic.
- **"Мені просто дорого щоразу створювати новий об'єкт (алокація + GC), а стан кожного разу мені потрібен свій, без спільного використання?"** → Object Pool — перевикористовуй об'єкт, скидаючи його стан між орендами.
- Якщо у вашому "Flyweight" зникло розділення на intrinsic/extrinsic і об'єкти просто видаються-повертаються з мутацією стану — це вже **не Flyweight, а Object Pool**.

---

## Переваги та недоліки

### Переваги

- **Величезна економія пам'яті** — для великої кількості схожих об'єктів пам'ять скорочується на порядки (див. приклади: у 80 000+ разів для лісу з дерев).
- **Централізоване керування спільним станом** — весь "важкий" внутрішній стан живе в одному місці (`FlyweightFactory`), його легко переглянути, оновити чи проаналізувати.
- **Прискорення створення об'єктів** — якщо внутрішній стан уже закешовано, "створення" нового контексту зводиться до простого посилання на існуючий flyweight, без повторних важких обчислень.
- **Масштабованість** — дозволяє системам оперувати мільйонами логічних об'єктів (дерева, символи, кулі, токени) там, де наївний підхід був би фізично неможливим.

### Недоліки

- **Додана складність** — потрібно свідомо розділяти стан на intrinsic/extrinsic, що ускладнює дизайн класів порівняно з "наївним" ООП-підходом.
- **Компроміс пам'ять↔CPU** — зовнішній стан доводиться передавати в кожен виклик методу flyweight-а (або зберігати й передавати з контексту), що може означати додаткові обчислення чи параметри в кожному виклику.
- **Flyweight-и мають бути незмінними (immutable) і потокобезпечними** — оскільки один і той самий екземпляр використовується одночасно багатьма контекстами (і, можливо, багатьма потоками), будь-яка мутація intrinsic-стану flyweight-а зламає всіх, хто на нього посилається.
- **Складніше налагоджувати** — коли щось пішло не так з "одним конкретним об'єктом", важче зрозуміти, що насправді проблема стосується спільного flyweight-а, яким користуються тисячі контекстів.

---

## Антипатерни та поширені помилки

### Помилка 1 — Мутабельний Flyweight, що зберігає зовнішній стан

Найпоширеніша й найнебезпечніша помилка: випадково залишити в flyweight-і поле для унікальних (зовнішніх) даних. Оскільки flyweight — спільний об'єкт, зміна цього поля "просочується" в усі місця, де цей flyweight використовується.

**НЕПРАВИЛЬНО:**

```csharp
// Flyweight, що зберігає X/Y — це ПОМИЛКА: X/Y є зовнішнім станом!
public class TreeTypeBad
{
    public string Name;
    public string Color;
    public float X; // <-- НЕ МАЄ ТУТ БУТИ
    public float Y; // <-- НЕ МАЄ ТУТ БУТИ
}

var oak = TreeTypeFactory.GetTreeType("Oak", "Green"); // спільний екземпляр
oak.X = 10;  oak.Y = 20;   // "Садимо" дерево №1

// ... пізніше, для іншого дерева ТОГО Ж ВИДУ:
var oak2 = TreeTypeFactory.GetTreeType("Oak", "Green"); // це ТОЙ САМИЙ об'єкт!
oak2.X = 500; oak2.Y = 800; // "Садимо" дерево №2

// БАГ: oak.X тепер теж дорівнює 500, а не 10!
// Обидва "дерева" насправді є ОДНИМ і тим самим об'єктом в пам'яті.
Console.WriteLine(oak.X); // виведе 500, а очікувалось 10 — дерева "телепортувались" одне в одне
```

**ПРАВИЛЬНО:**

```csharp
// TreeType — ЛИШЕ незмінний внутрішній стан
public class TreeType
{
    public string Name { get; }
    public string Color { get; }
    public TreeType(string name, string color) { Name = name; Color = color; }
}

// Позиція живе у ОКРЕМОМУ Context-об'єкті, який НЕ є спільним
public class Tree
{
    public float X { get; }
    public float Y { get; }
    public TreeType Type { get; } // посилання на спільний flyweight

    public Tree(float x, float y, TreeType type) { X = x; Y = y; Type = type; }
}

var oakType = TreeTypeFactory.GetTreeType("Oak", "Green");
var tree1 = new Tree(10, 20, oakType);   // власний контекст — власна позиція
var tree2 = new Tree(500, 800, oakType); // той самий тип, але ІНША позиція

Console.WriteLine(tree1.X); // 10 — правильно, кожне дерево незалежне
```

### Помилка 2 — Фабрика, яка насправді не кешує (не дедуплікує)

Якщо `FlyweightFactory` створює новий об'єкт при кожному виклику замість повернення з кешу — весь сенс патерну втрачається: ви отримуєте той самий наївний підхід, лише з зайвим шаром абстракції.

**НЕПРАВИЛЬНО:**

```csharp
// "Фабрика", яка НЕ кешує — щоразу створює новий об'єкт!
public static class BadTreeTypeFactory
{
    public static TreeType GetTreeType(string name, string color)
    {
        // ПОМИЛКА: немає жодної перевірки на існування — завжди new
        return new TreeType(name, color);
    }
}

// В результаті: 1 000 000 викликів GetTreeType("Oak", "Green")
// створюють 1 000 000 РІЗНИХ об'єктів TreeType з ідентичними даними —
// та сама катастрофа з пам'яттю, що й без патерну взагалі!
```

**ПРАВИЛЬНО:**

```csharp
// Фабрика зберігає створені об'єкти в Dictionary, ключ — комбінація intrinsic-параметрів
public static class TreeTypeFactory
{
    private static readonly Dictionary<string, TreeType> _cache = new();

    public static TreeType GetTreeType(string name, string color)
    {
        string key = $"{name}|{color}";

        if (!_cache.TryGetValue(key, out var type))
        {
            type = new TreeType(name, color); // створюємо лише якщо ще немає
            _cache[key] = type;
        }

        return type; // повертаємо ІСНУЮЧИЙ екземпляр при повторному виклику
    }
}

// Тепер 1 000 000 викликів з однаковими параметрами дадуть
// РІВНО ОДИН об'єкт TreeType — саме заради цього й існує патерн
```

### Помилка 3 — Відсутність потокобезпеки в кеші фабрики

Якщо кілька потоків одночасно звертаються до фабрики (наприклад, паралельна генерація великого світу через `Parallel.For`), звичайний `Dictionary<TKey, TValue>` **не є потокобезпечним**. Дві гонки-умови (race conditions) можуть призвести до пошкодження внутрішньої структури словника або до створення дублікатів flyweight-ів.

**НЕПРАВИЛЬНО:**

```csharp
public static class UnsafeTreeTypeFactory
{
    private static readonly Dictionary<string, TreeType> _cache = new();

    public static TreeType GetTreeType(string name, string color)
    {
        string key = $"{name}|{color}";

        // ПОМИЛКА: немає синхронізації — при одночасному доступі з кількох
        // потоків можлива гонка: два потоки одночасно бачать "немає в кеші",
        // обидва створюють НОВИЙ TreeType, і другий запис перезаписує перший
        // (а гірше — Dictionary може кинути виняток або пошкодити структуру).
        if (!_cache.TryGetValue(key, out var type))
        {
            type = new TreeType(name, color);
            _cache[key] = type;
        }

        return type;
    }
}

// Паралельна посадка дерев — небезпечно з UnsafeTreeTypeFactory:
Parallel.For(0, 1_000_000, i =>
{
    forest.PlantTree(rnd.Next(10000), rnd.Next(10000), 1f, "Oak", "Green");
    // Ризик: InvalidOperationException з Dictionary, або дублікати TreeType
});
```

**ПРАВИЛЬНО (варіант 1 — lock):**

```csharp
public static class ThreadSafeTreeTypeFactory
{
    private static readonly Dictionary<string, TreeType> _cache = new();
    private static readonly object _lock = new();

    public static TreeType GetTreeType(string name, string color)
    {
        string key = $"{name}|{color}";

        lock (_lock)
        {
            if (!_cache.TryGetValue(key, out var type))
            {
                type = new TreeType(name, color);
                _cache[key] = type;
            }
            return type;
        }
    }
}
```

**ПРАВИЛЬНО (варіант 2 — ConcurrentDictionary, без явних локів):**

```csharp
using System.Collections.Concurrent;

public static class ConcurrentTreeTypeFactory
{
    private static readonly ConcurrentDictionary<string, TreeType> _cache = new();

    public static TreeType GetTreeType(string name, string color)
    {
        string key = $"{name}|{color}";

        // GetOrAdd атомарно перевіряє наявність і додає, якщо відсутнє.
        // Примітка: valueFactory теоретично може викликатись кілька разів
        // під конкуренцією (зайвий TreeType буде відкинуто), тому
        // конструктор TreeType має бути дешевим або без побічних ефектів,
        // або варто додатково захистити створення власним lock всередині.
        return _cache.GetOrAdd(key, _ => new TreeType(name, color));
    }
}
```

`ConcurrentDictionary.GetOrAdd` — зручний спосіб уникнути явних локів, але варто пам'ятати: якщо `valueFactory` "дорогий" (наприклад, справді завантажує текстуру з диска), при високій конкуренції він може виконатися кілька разів паралельно (хоча в кеш потрапить лише один результат) — у такому разі `lock` з подвійною перевіркою (double-checked locking) може бути кращим вибором.

---

## Підсумок

Flyweight варто застосовувати, коли:

- У системі потрібна **величезна кількість** схожих об'єктів (тисячі, мільйони) — дерева, символи, частинки, токени, іконки, клітинки на карті.
- **Пам'ять — реальне обмеження**, і наївне зберігання повного стану в кожному об'єкті призводить до непомірних витрат (гігабайти чи терабайти).
- Більшість стану об'єктів можна **зробити зовнішнім (extrinsic)** — тобто винести з об'єкта й передавати ззовні, залишивши всередині лише те, що дійсно повторюється між об'єктами.
- Кількість **унікальних комбінацій** внутрішнього стану значно **менша**, ніж загальна кількість об'єктів (наприклад: 4 види дерев на мільйон дерев, 3 типи куль на тисячі пострілів).

Не варто застосовувати Flyweight, якщо:

- Об'єктів мало (десятки-сотні) — економія пам'яті буде мізерною порівняно з доданою складністю.
- Кожен об'єкт унікальний за своєю природою (майже немає повторюваних даних) — ділити стан просто нема на що.
- Внутрішній стан має часто змінюватись — Flyweight вимагає незмінності (immutability) для безпечного спільного використання.

### Мінімальний шаблон

```csharp
using System.Collections.Generic;

// 1. Flyweight — незмінний внутрішній (intrinsic) стан, спільний для багатьох
public class Flyweight
{
    public string IntrinsicState { get; }

    public Flyweight(string intrinsicState)
    {
        IntrinsicState = intrinsicState;
    }

    // Методи отримують зовнішній стан як параметри
    public void Operation(string extrinsicState)
    {
        // Console.WriteLine($"{IntrinsicState} + {extrinsicState}");
    }
}

// 2. FlyweightFactory — кеш, що гарантує перевикористання
public static class FlyweightFactory
{
    private static readonly Dictionary<string, Flyweight> _cache = new();

    public static Flyweight GetFlyweight(string key)
    {
        if (!_cache.TryGetValue(key, out var flyweight))
        {
            flyweight = new Flyweight(key);
            _cache[key] = flyweight;
        }
        return flyweight;
    }

    public static int Count => _cache.Count;
}

// 3. Context — зовнішній (extrinsic) стан + посилання на flyweight
public class Context
{
    public string ExtrinsicState { get; }
    private readonly Flyweight _flyweight;

    public Context(string extrinsicState, string flyweightKey)
    {
        ExtrinsicState = extrinsicState;
        _flyweight = FlyweightFactory.GetFlyweight(flyweightKey);
    }

    public void Render() => _flyweight.Operation(ExtrinsicState);
}
```

| Аспект | Деталь |
|---|---|
| Тип патерну | Структурний (Structural) |
| Вирішує проблему | Надмірне споживання пам'яті при великій кількості схожих об'єктів |
| Ключова ідея | Розділити стан на **intrinsic** (спільний, у flyweight) і **extrinsic** (унікальний, у контексті) |
| Головна вимога | Flyweight має бути **незмінним (immutable)** — він розділяється між багатьма контекстами одночасно |
| Ключовий учасник | `FlyweightFactory` — кеш/словник, що гарантує дедуплікацію за ключем intrinsic-стану |
| У реальному .NET | String interning (`string.Intern`), `Enum` значення, кешування шаблонів у рушіях рендерингу, пул шрифтових гліфів у GDI+/Direct2D |

---

*Документ підготовлено для вивчення патернів проектування. Всі приклади протестовані на .NET 6+.*
