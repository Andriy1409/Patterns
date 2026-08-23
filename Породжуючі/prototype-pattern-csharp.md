# Патерн Prototype (Прототип) у C#

> **Категорія:** Породжуючий патерн (Creational Pattern)  
> **Мова прикладів:** C# (.NET)

---

## Зміст

1. [Що таке Prototype?](#що-таке-prototype)
2. [Поверхневе vs Глибоке копіювання](#поверхневе-vs-глибоке-копіювання)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Простий (Форма на полотні)](#приклад-1--простий-форма-на-полотні)
5. [Приклад 2 — Просунутий (Конфігурація сервера)](#приклад-2--просунутий-конфігурація-сервера)
6. [Реєстр прототипів (Prototype Registry)](#реєстр-прототипів-prototype-registry)
7. [ICloneable — вбудований інтерфейс C#](#icloneable--вбудований-інтерфейс-c)
8. [Переваги та недоліки](#переваги-та-недоліки)
9. [Підсумок](#підсумок)

---

## Що таке Prototype?

**Prototype** — це патерн, який дозволяє **копіювати існуючі об'єкти**, не прив'язуючись до їхніх конкретних класів.

Замість того, щоб створювати новий об'єкт з нуля (`new`), ми беремо вже існуючий об'єкт і **клонуємо** його. Потім, якщо потрібно, змінюємо лише те, що відрізняється.

### Проблема без Prototype

Уявіть, що у вас є складно налаштований об'єкт:

```csharp
// Щоб зробити схожий об'єкт, доводиться дублювати всю конфігурацію вручну
var original = new ServerConfig();
original.Host = "db.prod.example.com";
original.Port = 5432;
original.MaxConnections = 100;
original.SslEnabled = true;
original.ConnectionTimeout = TimeSpan.FromSeconds(30);
original.Tags.Add("production");
original.Tags.Add("postgres");
// ... ще 20 полів ...

// Хочемо другий сервер — майже такий самий, але з іншим хостом
// Без патерну доводиться переписувати все знову:
var replica = new ServerConfig();
replica.Host = "db-replica.prod.example.com"; // тільки це відрізняється!
replica.Port = 5432;                           // все решта — те саме
replica.MaxConnections = 100;
// ... і так далі ...
```

Це **дублювання коду**, схильне до помилок (легко забути скопіювати одне поле).

### Рішення з Prototype

```csharp
// Клонуємо оригінал — отримуємо точну копію
var replica = original.Clone();

// Змінюємо тільки те, що відрізняється
replica.Host = "db-replica.prod.example.com";
```

Чисто, безпечно, без дублювання.

---

## Поверхневе vs Глибоке копіювання

Це **найважливіша концепція** при роботі з Prototype. Не розуміючи різниці, можна отримати важко помітні баги.

### Поверхневе копіювання (Shallow Copy)

Копіюються значення всіх полів. Але якщо поле — це **посилання на об'єкт**, копіюється саме посилання, а не об'єкт, на який воно вказує.

```csharp
public class Person
{
    public string Name { get; set; }
    public List<string> Hobbies { get; set; }
}

var original = new Person
{
    Name = "Іван",
    Hobbies = new List<string> { "читання", "футбол" }
};

// MemberwiseClone() — вбудований метод C# для shallow copy
var copy = (Person)original.MemberwiseClone();

// Змінюємо ім'я копії — не впливає на оригінал (string — value-like)
copy.Name = "Петро";
Console.WriteLine(original.Name); // "Іван" — ОК

// Додаємо хобі до копії — ЗМІНЮЄ ТАКОЖ ОРИГІНАЛ!
copy.Hobbies.Add("плавання");
Console.WriteLine(original.Hobbies.Count); // 3 — БАГ! Очікували 2
```

**Чому так?** Обидва об'єкти вказують на **один і той самий список** у пам'яті.

```
original ──→ [ Name: "Іван"  | Hobbies ──→ ["читання", "футбол"] ]
                                                    ↑
copy     ──→ [ Name: "Петро" | Hobbies ──────────────┘           ]
```

### Глибоке копіювання (Deep Copy)

Копіюються **всі об'єкти рекурсивно** — кожен вкладений об'єкт також клонується.

```csharp
var copy = new Person
{
    Name = original.Name,
    Hobbies = new List<string>(original.Hobbies) // новий список з тими самими елементами
};

copy.Hobbies.Add("плавання");
Console.WriteLine(original.Hobbies.Count); // 2 — ОК, оригінал не змінився
```

```
original ──→ [ Name: "Іван"  | Hobbies ──→ ["читання", "футбол"]           ]

copy     ──→ [ Name: "Петро" | Hobbies ──→ ["читання", "футбол", "плавання"] ]
```

> **Правило:** Якщо ваш об'єкт містить колекції, вкладені об'єкти або масиви — вам потрібне **глибоке копіювання**.

---

## Структура патерну

```
┌─────────────────────────┐
│   «interface»           │
│   IPrototype<T>         │
│  + Clone() : T          │
└────────────┬────────────┘
             │ implements
    ┌────────┴─────────┐
    │                  │
┌───▼──────────┐  ┌────▼──────────┐
│ConcreteProto │  │ConcreteProto  │
│      A       │  │      B        │
│ + Clone(): A │  │ + Clone(): B  │
└──────────────┘  └───────────────┘

┌─────────────────────────┐
│   Client                │
│                         │
│  proto = existing.Clone │
│  proto.field = newValue │
└─────────────────────────┘
```

Учасники:
- **Prototype (IPrototype)** — інтерфейс з методом `Clone()`
- **ConcretePrototype** — реалізує клонування себе
- **Client** — викликає `Clone()` замість `new`

---

## Приклад 1 — Простий (Форма на полотні)

Уявімо графічний редактор. Користувач малює фігури і хоче дублювати їх.

### Клас `Shape` та його нащадки

```csharp
// Базовий абстрактний клас — визначає контракт Clone()
public abstract class Shape
{
    public int X { get; set; }
    public int Y { get; set; }
    public string Color { get; set; }

    // Захищений конструктор копіювання — для зручності нащадків
    // Копіює спільні поля базового класу
    protected Shape(Shape source)
    {
        X = source.X;
        Y = source.Y;
        Color = source.Color;
    }

    // Порожній конструктор для початкового створення
    protected Shape() { }

    // Абстрактний метод — кожен нащадок реалізує своє клонування
    public abstract Shape Clone();

    public abstract void Draw();
}
```

```csharp
// Прямокутник
public class Rectangle : Shape
{
    public int Width { get; set; }
    public int Height { get; set; }

    public Rectangle() { }

    // Конструктор копіювання: спочатку копіюємо базові поля,
    // потім — специфічні для Rectangle
    private Rectangle(Rectangle source) : base(source)
    {
        Width = source.Width;
        Height = source.Height;
    }

    // Clone повертає новий Rectangle — точну копію поточного
    public override Shape Clone() => new Rectangle(this);

    public override void Draw()
    {
        Console.WriteLine($"Rectangle [{Color}] at ({X},{Y}), size: {Width}x{Height}");
    }
}
```

```csharp
// Коло
public class Circle : Shape
{
    public int Radius { get; set; }

    public Circle() { }

    private Circle(Circle source) : base(source)
    {
        Radius = source.Radius;
    }

    public override Shape Clone() => new Circle(this);

    public override void Draw()
    {
        Console.WriteLine($"Circle [{Color}] at ({X},{Y}), radius: {Radius}");
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        // Створюємо оригінальні фігури
        var rect = new Rectangle { X = 10, Y = 20, Color = "red", Width = 100, Height = 50 };
        var circle = new Circle { X = 50, Y = 50, Color = "blue", Radius = 30 };

        rect.Draw();   // Rectangle [red] at (10,20), size: 100x50
        circle.Draw(); // Circle [blue] at (50,50), radius: 30

        Console.WriteLine("--- Клонуємо і зміщуємо ---");

        // Клонуємо і трохи зміщуємо — як "Ctrl+D" у графічному редакторі
        var rectCopy = rect.Clone();
        rectCopy.X += 20;
        rectCopy.Y += 20;
        rectCopy.Draw(); // Rectangle [red] at (30,40), size: 100x50

        var circleCopy = circle.Clone();
        circleCopy.Color = "green"; // міняємо тільки колір
        circleCopy.Draw(); // Circle [green] at (50,50), radius: 30

        // Оригінали не змінились
        Console.WriteLine("--- Оригінали ---");
        rect.Draw();   // Rectangle [red] at (10,20), size: 100x50
        circle.Draw(); // Circle [blue] at (50,50), radius: 30
    }
}
```

---

## Приклад 2 — Просунутий (Конфігурація сервера)

Тепер реалістичний сценарій: система конфігурацій для кластеру серверів. Сервери мають спільну базову конфігурацію, яка трохи відрізняється між собою.

### Допоміжні класи

```csharp
// Налаштування SSL-сертифіката — окремий об'єкт (вкладений)
public class SslConfig
{
    public string CertificatePath { get; set; }
    public string KeyPath { get; set; }
    public bool VerifyPeer { get; set; }

    // Конструктор копіювання для глибокого клонування
    public SslConfig Clone() => new SslConfig
    {
        CertificatePath = this.CertificatePath,
        KeyPath = this.KeyPath,
        VerifyPeer = this.VerifyPeer
    };

    public override string ToString() =>
        $"SSL(cert={CertificatePath}, verify={VerifyPeer})";
}

// Правило для firewall
public class FirewallRule
{
    public string Direction { get; set; }   // "inbound" / "outbound"
    public int Port { get; set; }
    public string Protocol { get; set; }    // "tcp" / "udp"
    public bool Allow { get; set; }

    public FirewallRule Clone() => new FirewallRule
    {
        Direction = this.Direction,
        Port = this.Port,
        Protocol = this.Protocol,
        Allow = this.Allow
    };

    public override string ToString() =>
        $"{(Allow ? "ALLOW" : "DENY")} {Direction} {Protocol}:{Port}";
}
```

### Основний клас конфігурації

```csharp
public class ServerConfig : ICloneable
{
    // --- Прості поля (value types та strings) ---
    public string NodeName { get; set; }
    public string Host { get; set; }
    public int Port { get; set; }
    public int MaxConnections { get; set; }
    public bool SslEnabled { get; set; }
    public TimeSpan ConnectionTimeout { get; set; }
    public string Environment { get; set; } // "production" / "staging" / "dev"

    // --- Вкладені об'єкти (потребують глибокого копіювання) ---
    public SslConfig Ssl { get; set; }

    // --- Колекції (потребують глибокого копіювання) ---
    public List<string> Tags { get; set; } = new();
    public List<FirewallRule> FirewallRules { get; set; } = new();
    public Dictionary<string, string> EnvironmentVariables { get; set; } = new();

    // Реалізація ICloneable — повертає object (зручний, але менш типізований)
    object ICloneable.Clone() => Clone();

    // Типізований Clone — зручніший у використанні
    public ServerConfig Clone()
    {
        // Крок 1: Копіюємо прості поля через MemberwiseClone
        // Для value types (int, bool, TimeSpan) і strings — це ОК
        var clone = (ServerConfig)this.MemberwiseClone();

        // Крок 2: Вручну глибоко копіюємо всі посилальні типи

        // Копіюємо вкладений об'єкт SslConfig
        clone.Ssl = this.Ssl?.Clone();

        // Копіюємо список тегів — створюємо новий список з тими самими рядками
        // (string в C# immutable, тому shallow copy рядків — безпечна)
        clone.Tags = new List<string>(this.Tags);

        // Копіюємо список правил — кожне правило клонуємо окремо
        // (FirewallRule — mutable об'єкт, тому потрібне глибоке копіювання)
        clone.FirewallRules = this.FirewallRules
            .Select(rule => rule.Clone())
            .ToList();

        // Копіюємо словник змінних середовища
        clone.EnvironmentVariables = new Dictionary<string, string>(this.EnvironmentVariables);

        return clone;
    }

    public override string ToString()
    {
        var sb = new System.Text.StringBuilder();
        sb.AppendLine($"[{NodeName}] {Host}:{Port} ({Environment})");
        sb.AppendLine($"  MaxConn: {MaxConnections} | Timeout: {ConnectionTimeout.TotalSeconds}s | SSL: {SslEnabled}");

        if (Ssl != null)
            sb.AppendLine($"  {Ssl}");

        if (Tags.Any())
            sb.AppendLine($"  Tags: {string.Join(", ", Tags)}");

        if (FirewallRules.Any())
        {
            sb.AppendLine("  Firewall:");
            foreach (var rule in FirewallRules)
                sb.AppendLine($"    {rule}");
        }

        if (EnvironmentVariables.Any())
        {
            sb.AppendLine("  EnvVars:");
            foreach (var kv in EnvironmentVariables)
                sb.AppendLine($"    {kv.Key}={kv.Value}");
        }

        return sb.ToString();
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        // --- Крок 1: Створюємо базовий шаблон конфігурації ---
        var baseConfig = new ServerConfig
        {
            Port = 443,
            MaxConnections = 1000,
            SslEnabled = true,
            ConnectionTimeout = TimeSpan.FromSeconds(30),
            Environment = "production",
            Ssl = new SslConfig
            {
                CertificatePath = "/etc/ssl/certs/prod.crt",
                KeyPath = "/etc/ssl/private/prod.key",
                VerifyPeer = true
            },
            Tags = new List<string> { "production", "postgres", "europe-west" },
            EnvironmentVariables = new Dictionary<string, string>
            {
                ["LOG_LEVEL"] = "warn",
                ["MAX_QUERY_TIME"] = "5000"
            }
        };

        baseConfig.FirewallRules.Add(new FirewallRule
            { Direction = "inbound", Port = 443, Protocol = "tcp", Allow = true });
        baseConfig.FirewallRules.Add(new FirewallRule
            { Direction = "inbound", Port = 22, Protocol = "tcp", Allow = false });

        // --- Крок 2: Клонуємо для першого вузла кластеру ---
        var node1 = baseConfig.Clone();
        node1.NodeName = "db-primary";
        node1.Host = "10.0.1.10";

        // --- Крок 3: Клонуємо для репліки ---
        var node2 = baseConfig.Clone();
        node2.NodeName = "db-replica-1";
        node2.Host = "10.0.1.11";
        node2.MaxConnections = 500;          // репліка може менше
        node2.Tags.Add("read-only");         // додаємо тег тільки репліці
        node2.EnvironmentVariables["LOG_LEVEL"] = "info"; // більше логів на репліці

        // Додаємо правило тільки для репліки
        node2.FirewallRules.Add(new FirewallRule
            { Direction = "inbound", Port = 5432, Protocol = "tcp", Allow = true });

        // --- Крок 4: Ще одна репліка в іншому регіоні ---
        var node3 = baseConfig.Clone();
        node3.NodeName = "db-replica-2";
        node3.Host = "10.0.2.11";
        node3.Tags.Remove("europe-west");
        node3.Tags.Add("us-east");
        node3.Ssl = new SslConfig
        {
            CertificatePath = "/etc/ssl/certs/us-prod.crt",
            KeyPath = "/etc/ssl/private/us-prod.key",
            VerifyPeer = true
        };

        // --- Виводимо результат ---
        Console.WriteLine("=== PRIMARY ===");
        Console.WriteLine(node1);

        Console.WriteLine("=== REPLICA 1 (EU) ===");
        Console.WriteLine(node2);

        Console.WriteLine("=== REPLICA 2 (US) ===");
        Console.WriteLine(node3);

        // --- Доводимо що оригінал не змінився ---
        Console.WriteLine("=== BASE CONFIG (незмінний) ===");
        Console.WriteLine(baseConfig);

        // Перевірка ізоляції: Tags — це різні об'єкти
        Console.WriteLine($"node1.Tags == node2.Tags? {ReferenceEquals(node1.Tags, node2.Tags)}");
        // Виведе: False — чудово, вони незалежні!
    }
}
```

### Очікуваний вивід

```
=== PRIMARY ===
[db-primary] 10.0.1.10:443 (production)
  MaxConn: 1000 | Timeout: 30s | SSL: True
  SSL(cert=/etc/ssl/certs/prod.crt, verify=True)
  Tags: production, postgres, europe-west
  Firewall:
    ALLOW inbound tcp:443
    DENY inbound tcp:22
  EnvVars:
    LOG_LEVEL=warn
    MAX_QUERY_TIME=5000

=== REPLICA 1 (EU) ===
[db-replica-1] 10.0.1.11:443 (production)
  MaxConn: 500 | Timeout: 30s | SSL: True
  SSL(cert=/etc/ssl/certs/prod.crt, verify=True)
  Tags: production, postgres, europe-west, read-only
  Firewall:
    ALLOW inbound tcp:443
    DENY inbound tcp:22
    ALLOW inbound tcp:5432
  EnvVars:
    LOG_LEVEL=info
    MAX_QUERY_TIME=5000

=== BASE CONFIG (незмінний) ===
[] :443 (production)
  MaxConn: 1000 | Timeout: 30s | SSL: True
  Tags: production, postgres, europe-west
  ...
```

---

## Реєстр прототипів (Prototype Registry)

Часто зручно зберігати готові прототипи у центральному сховищі — **реєстрі**. Клієнт просить потрібний тип, отримує клон.

```csharp
// Реєстр зберігає іменовані прототипи і видає їхні копії
public class ServerConfigRegistry
{
    // Словник: ім'я шаблону → прототип
    private readonly Dictionary<string, ServerConfig> _prototypes = new();

    // Реєструємо шаблон
    public void Register(string name, ServerConfig config)
    {
        _prototypes[name] = config;
    }

    // Повертаємо КЛОН — не оригінал!
    // Це гарантує, що клієнт не зіпсує шаблон
    public ServerConfig Get(string name)
    {
        if (!_prototypes.TryGetValue(name, out var prototype))
            throw new KeyNotFoundException($"Прототип '{name}' не знайдено в реєстрі");

        return prototype.Clone(); // завжди клон!
    }

    // Зручний метод: отримати клон і відразу налаштувати через Action
    public ServerConfig Get(string name, Action<ServerConfig> configure)
    {
        var clone = Get(name);
        configure(clone);
        return clone;
    }
}
```

### Використання реєстру

```csharp
// Налаштовуємо реєстр один раз (наприклад, при старті програми)
var registry = new ServerConfigRegistry();

registry.Register("prod-db", new ServerConfig
{
    Port = 5432,
    MaxConnections = 1000,
    SslEnabled = true,
    Environment = "production",
    ConnectionTimeout = TimeSpan.FromSeconds(30),
    Tags = new List<string> { "production" }
});

registry.Register("dev-db", new ServerConfig
{
    Port = 5432,
    MaxConnections = 10,
    SslEnabled = false,
    Environment = "development",
    ConnectionTimeout = TimeSpan.FromSeconds(5),
    Tags = new List<string> { "dev", "local" }
});

// Пізніше — отримуємо налаштований клон в одну строчку
var newProdNode = registry.Get("prod-db", cfg =>
{
    cfg.NodeName = "db-node-5";
    cfg.Host = "10.0.1.15";
});

var devNode = registry.Get("dev-db", cfg =>
{
    cfg.NodeName = "local-db";
    cfg.Host = "localhost";
});
```

---

## ICloneable — вбудований інтерфейс C#

.NET має вбудований інтерфейс `System.ICloneable`:

```csharp
public interface ICloneable
{
    object Clone();
}
```

### Проблема ICloneable

Інтерфейс повертає `object`, що змушує робити приведення типів:

```csharp
var copy = (ServerConfig)original.Clone(); // приведення типу — неелегантно
```

### Краще рішення — Generic Prototype

```csharp
// Типізований інтерфейс — значно зручніший
public interface IPrototype<T>
{
    T Clone();
}

public class ServerConfig : IPrototype<ServerConfig>
{
    public ServerConfig Clone() { /* ... */ }
}

// Тепер без приведення типів:
ServerConfig copy = original.Clone(); // чисто і типобезпечно
```

> **Рекомендація:** У сучасному C# краще використовувати власний типізований інтерфейс, а не стандартний `ICloneable`.

---

## Переваги та недоліки

### Переваги

- **Швидкість** — клонування може бути швидшим за створення об'єкта з нуля (особливо якщо ініціалізація дорога)
- **Незалежність від класу** — клієнт може копіювати об'єкти не знаючи їхнього конкретного типу
- **Уникнення дублювання** — не потрібно повторювати однакові налаштування
- **Гнучкість** — легко отримати трохи відмінний варіант об'єкта

### Недоліки

- **Складне глибоке копіювання** — при циклічних посиланнях або складній ієрархії об'єктів глибоке копіювання стає нетривіальним
- **Приховані залежності** — якщо забути про глибоке копіювання — непомітні баги з shared state
- **Накладні витрати** — для простих об'єктів `new` + ініціалізація може бути простіше і зрозуміліше

---

## Підсумок

| Аспект | Деталь |
|---|---|
| Тип патерну | Породжуючий (Creational) |
| Вирішує проблему | Дорого або незручно створювати об'єкт з нуля |
| Ключова ідея | Копіювати існуючий об'єкт замість `new` |
| Головна складність | Shallow vs Deep copy — завжди думайте про вкладені об'єкти |
| Ключовий метод | `Clone()` |
| Альтернативи | Factory Method, Builder |

### Різниця між схожими патернами

| Патерн | Суть |
|---|---|
| **Prototype** | Копіює існуючий об'єкт |
| **Builder** | Будує новий об'єкт крок за кроком |
| **Factory Method** | Делегує створення нового об'єкта підкласу |

### Коротке правило вибору

```
Є вже налаштований об'єкт, і потрібен схожий?
  ✅ Так → Prototype (Clone)

Потрібно побудувати складний об'єкт з нуля?
  ✅ Так → Builder

Об'єкт вкладений і mutable?
  ✅ Так → Обов'язково Deep Copy у Clone()
```

---

*Документ підготовлено як навчальний матеріал з патернів проєктування на C#.*
