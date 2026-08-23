# Патерн Factory Method — Детальний розбір на C#

> **Категорія:** Породжуючий (Creational)  
> **Автори:** Gang of Four (GoF)  
> **Мова прикладів:** C#

---

## Зміст

1. [Що таке Factory Method?](#що-таке-factory-method)
2. [Проблема без патерну](#проблема-без-патерну)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Найпростіший Factory Method](#приклад-1--найпростіший-factory-method)
5. [Приклад 2 — Транспортна компанія](#приклад-2--транспортна-компанія)
6. [Приклад 3 — Система сповіщень](#приклад-3--система-сповіщень)
7. [Приклад 4 — Реальний сценарій: парсери документів](#приклад-4--реальний-сценарій-парсери-документів)
8. [Factory Method vs Simple Factory vs Abstract Factory](#factory-method-vs-simple-factory-vs-abstract-factory)
9. [Переваги та недоліки](#переваги-та-недоліки)
10. [Антипатерни та поширені помилки](#антипатерни-та-поширені-помилки)
11. [Підсумок](#підсумок)

---

## Що таке Factory Method?

**Factory Method** — це патерн проектування, який визначає **інтерфейс для створення об'єктів**, але дозволяє **підкласам вирішувати**, який саме клас інстанціювати.

Іншими словами: базовий клас каже _"я створю якийсь об'єкт"_, а підкласи кажуть _"а ось який саме"_.

### Аналогія з реального світу

Уяви мережу ресторанів швидкого харчування. Є загальна процедура: **прийняти замовлення → приготувати їжу → подати клієнту**. Але кожен ресторан (_McDonald's_, _KFC_, _Burger King_) готує **свою власну їжу** за своїми рецептами.

- Базовий клас: `Restaurant` — знає **як** обробити замовлення.
- Підкласи: `McDonalds`, `KFC` — знають **що** саме готувати.
- Factory Method: `CreateMeal()` — визначений у базовому класі, перевизначений у кожному підкласі.

---

## Проблема без патерну

Розглянемо типову проблему, яку вирішує Factory Method.

### Код без патерну — жорсткі залежності

```csharp
// Є кілька типів кнопок
public class WindowsButton
{
    public void Render() => Console.WriteLine("Рендер кнопки у стилі Windows");
    public void Click()  => Console.WriteLine("Windows кнопка натиснута");
}

public class MacButton
{
    public void Render() => Console.WriteLine("Рендер кнопки у стилі macOS");
    public void Click()  => Console.WriteLine("Mac кнопка натиснута");
}

// Клас UI знає про ВСІ конкретні типи кнопок — жорстка залежність!
public class UI
{
    private string _os;

    public UI(string os)
    {
        _os = os;
    }

    public void RenderButton()
    {
        // Щоразу треба додавати нову гілку if/else при додаванні нового типу
        if (_os == "Windows")
        {
            var btn = new WindowsButton(); // ← Жорстка залежність від конкретного класу
            btn.Render();
        }
        else if (_os == "Mac")
        {
            var btn = new MacButton();     // ← Ще одна жорстка залежність
            btn.Render();
        }
        // А якщо з'явиться Linux? Треба лізти в цей метод і змінювати код!
    }
}
```

### Що тут погано?

- `UI` знає про `WindowsButton` і `MacButton` — **порушення принципу відкритості/закритості (OCP)**.
- Щоб додати `LinuxButton` — потрібно **змінювати** вже написаний код `UI`.
- `if/else` розростається з кожним новим типом.
- Код **важко тестувати** — не можна підставити тестову кнопку.

### Саме це і вирішує Factory Method.

---

## Структура патерну

```
┌──────────────────────────────────────────────────┐
│              <<abstract>>                        │
│                 Creator                          │
│  ─────────────────────────────────────────────   │
│  + SomeOperation()           ← використовує      │
│  # CreateProduct() : IProduct ← Factory Method  │
└──────────────────────┬───────────────────────────┘
                       │ наслідує
          ┌────────────┴────────────┐
          ▼                        ▼
┌─────────────────┐      ┌─────────────────┐
│  ConcreteCreatorA│      │ ConcreteCreatorB│
│  ─────────────  │      │  ─────────────  │
│  # CreateProduct│      │  # CreateProduct│
│    → ProductA   │      │    → ProductB   │
└────────┬────────┘      └────────┬────────┘
         │ створює                │ створює
         ▼                        ▼
┌─────────────────┐      ┌─────────────────┐
│   ProductA      │      │   ProductB      │
│  implements     │      │  implements     │
│  IProduct       │      │  IProduct       │
└─────────────────┘      └─────────────────┘
         │                        │
         └────────────┬───────────┘
                      ▼
             ┌─────────────────┐
             │   <<interface>> │
             │    IProduct     │
             │  ─────────────  │
             │  + DoStuff()    │
             └─────────────────┘
```

### Чотири ключові ролі

| Роль | Що робить | Приклад |
|---|---|---|
| `IProduct` | Інтерфейс для створюваних об'єктів | `IButton` |
| `ConcreteProduct` | Конкретна реалізація продукту | `WindowsButton`, `MacButton` |
| `Creator` | Базовий клас з factory method | `Dialog` |
| `ConcreteCreator` | Підклас, що перевизначає factory method | `WindowsDialog`, `MacDialog` |

---

## Приклад 1 — Найпростіший Factory Method

Починаємо з мінімального прикладу, щоб зрозуміти суть.

```csharp
// ─── ПРОДУКТ ─────────────────────────────────────────────────────────────────

// Інтерфейс продукту — що вміє кожна кнопка.
public interface IButton
{
    void Render();
    void Click();
}

// Конкретний продукт A
public class WindowsButton : IButton
{
    public void Render() => Console.WriteLine("[Windows] Відмальовую прямокутну кнопку");
    public void Click()  => Console.WriteLine("[Windows] Клік з характерним звуком Windows");
}

// Конкретний продукт B
public class MacButton : IButton
{
    public void Render() => Console.WriteLine("[Mac] Відмальовую округлу кнопку у стилі macOS");
    public void Click()  => Console.WriteLine("[Mac] Клік з плавною анімацією macOS");
}

// ─── CREATOR ─────────────────────────────────────────────────────────────────

// Абстрактний Creator — знає як ВИКОРИСТОВУВАТИ кнопку, але не знає яку саме.
public abstract class Dialog
{
    // ★ Це і є Factory Method — абстрактний метод, який підкласи мусять перевизначити.
    // Він повертає IButton, не конкретний тип — це ключова ідея.
    protected abstract IButton CreateButton();

    // Бізнес-логіка використовує продукт через інтерфейс IButton.
    // Цей метод НЕ знає, яка саме кнопка буде створена — його це не цікавить.
    public void Render()
    {
        Console.WriteLine("Dialog: починаю рендер вікна...");

        // Викликаємо Factory Method — отримуємо кнопку потрібного типу.
        IButton button = CreateButton();

        // Працюємо через інтерфейс — незалежно від конкретного типу.
        button.Render();

        Console.WriteLine("Dialog: вікно відмальовано.");
    }

    public void HandleClick()
    {
        IButton button = CreateButton();
        button.Click();
    }
}

// ─── CONCRETE CREATORS ───────────────────────────────────────────────────────

// Конкретний Creator для Windows — перевизначає Factory Method.
public class WindowsDialog : Dialog
{
    // Підклас вирішує: створювати WindowsButton.
    protected override IButton CreateButton()
    {
        Console.WriteLine("WindowsDialog: створюю WindowsButton");
        return new WindowsButton();
    }
}

// Конкретний Creator для Mac.
public class MacDialog : Dialog
{
    // Підклас вирішує: створювати MacButton.
    protected override IButton CreateButton()
    {
        Console.WriteLine("MacDialog: створюю MacButton");
        return new MacButton();
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        // Визначаємо тип ОС один раз при старті.
        string os = "Windows"; // У реальному коді: RuntimeInformation.IsOSPlatform(...)

        // Вибираємо потрібний Creator — далі код НЕ знає про конкретні типи кнопок.
        Dialog dialog = os == "Windows"
            ? new WindowsDialog()
            : new MacDialog();

        // Викликаємо бізнес-метод — він сам знає як створити потрібну кнопку.
        dialog.Render();
        Console.WriteLine();
        dialog.HandleClick();
    }
}

// Виведе:
// Dialog: починаю рендер вікна...
// WindowsDialog: створюю WindowsButton
// [Windows] Відмальовую прямокутну кнопку
// Dialog: вікно відмальовано.
//
// WindowsDialog: створюю WindowsButton
// [Windows] Клік з характерним звуком Windows
```

### Ключовий момент

Щоб додати `LinuxButton` — не потрібно чіпати `Dialog` або `WindowsDialog`. Просто:

```csharp
public class LinuxButton : IButton
{
    public void Render() => Console.WriteLine("[Linux] Відмальовую кнопку у стилі GTK");
    public void Click()  => Console.WriteLine("[Linux] Клік з анімацією Gnome");
}

public class LinuxDialog : Dialog
{
    protected override IButton CreateButton() => new LinuxButton();
}
```

Два нові класи — і все. Старий код не змінюється. **Принцип відкритості/закритості (OCP) виконано.**

---

## Приклад 2 — Транспортна компанія

Більш розгорнутий приклад з реальнішою бізнес-логікою — логістична система.

```csharp
// ─── ПРОДУКТИ ─────────────────────────────────────────────────────────────────

public interface ITransport
{
    string Name { get; }
    int MaxCargoKg { get; }
    void Deliver(string destination, int cargoKg);
}

public class Truck : ITransport
{
    public string Name => "Вантажівка";
    public int MaxCargoKg => 20_000;

    public void Deliver(string destination, int cargoKg)
    {
        Console.WriteLine($"  🚛 {Name}: везу {cargoKg} кг до '{destination}' по автошляху.");
        Console.WriteLine($"     Прибуття через: ~{cargoKg / 500 + 2} год.");
    }
}

public class Ship : ITransport
{
    public string Name => "Вантажне судно";
    public int MaxCargoKg => 500_000;

    public void Deliver(string destination, int cargoKg)
    {
        Console.WriteLine($"  🚢 {Name}: везу {cargoKg} кг до '{destination}' морем.");
        Console.WriteLine($"     Прибуття через: ~{cargoKg / 10000 + 24} год.");
    }
}

public class Drone : ITransport
{
    public string Name => "Дрон";
    public int MaxCargoKg => 30;

    public void Deliver(string destination, int cargoKg)
    {
        Console.WriteLine($"  🚁 {Name}: доставляю {cargoKg} кг до '{destination}' повітрям.");
        Console.WriteLine($"     Прибуття через: ~{cargoKg / 5 + 1} год.");
    }
}

// ─── CREATOR ─────────────────────────────────────────────────────────────────

public abstract class LogisticsCompany
{
    public string CompanyName { get; }

    protected LogisticsCompany(string name)
    {
        CompanyName = name;
    }

    // Factory Method — підкласи вирішують який транспорт створювати.
    protected abstract ITransport CreateTransport(int cargoKg);

    // Бізнес-логіка: планування та виконання доставки.
    // Весь цей код залишається незмінним незалежно від типу транспорту.
    public void PlanDelivery(string destination, int cargoKg)
    {
        Console.WriteLine($"\n[{CompanyName}] Замовлення: {cargoKg} кг → {destination}");

        // Перевірка обмежень
        ITransport transport = CreateTransport(cargoKg);

        if (cargoKg > transport.MaxCargoKg)
        {
            Console.WriteLine($"  ⚠️  Вантаж {cargoKg} кг перевищує ліміт {transport.MaxCargoKg} кг для {transport.Name}!");
            Console.WriteLine($"  ↳  Розбиваємо на {Math.Ceiling((double)cargoKg / transport.MaxCargoKg)} рейси.");
            int trips = (int)Math.Ceiling((double)cargoKg / transport.MaxCargoKg);
            for (int i = 1; i <= trips; i++)
            {
                int portion = Math.Min(transport.MaxCargoKg, cargoKg - (i - 1) * transport.MaxCargoKg);
                Console.Write($"  Рейс {i}/{trips}: ");
                transport.Deliver(destination, portion);
            }
        }
        else
        {
            transport.Deliver(destination, cargoKg);
        }

        Console.WriteLine($"  ✅ Доставка заплановано компанією '{CompanyName}'.");
    }
}

// ─── CONCRETE CREATORS ───────────────────────────────────────────────────────

// Наземна логістика — завжди використовує вантажівки.
public class RoadLogistics : LogisticsCompany
{
    public RoadLogistics() : base("UkrTruck Express") { }

    protected override ITransport CreateTransport(int cargoKg)
    {
        Console.WriteLine("  → Вибираємо наземний транспорт: Вантажівка");
        return new Truck();
    }
}

// Морська логістика — завжди використовує судна.
public class SeaLogistics : LogisticsCompany
{
    public SeaLogistics() : base("Black Sea Cargo") { }

    protected override ITransport CreateTransport(int cargoKg)
    {
        Console.WriteLine("  → Вибираємо морський транспорт: Судно");
        return new Ship();
    }
}

// Розумна логістика — вибирає транспорт залежно від розміру вантажу.
public class SmartLogistics : LogisticsCompany
{
    public SmartLogistics() : base("SmartDeliver AI") { }

    protected override ITransport CreateTransport(int cargoKg)
    {
        // Factory Method з внутрішньою логікою вибору — дуже гнучко!
        if (cargoKg <= 25)
        {
            Console.WriteLine("  → SmartAI: малий вантаж, вибираємо Дрон");
            return new Drone();
        }
        else if (cargoKg <= 15_000)
        {
            Console.WriteLine("  → SmartAI: середній вантаж, вибираємо Вантажівку");
            return new Truck();
        }
        else
        {
            Console.WriteLine("  → SmartAI: великий вантаж, вибираємо Судно");
            return new Ship();
        }
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        Console.WriteLine("=== Логістична система ===");

        var companies = new List<LogisticsCompany>
        {
            new RoadLogistics(),
            new SeaLogistics(),
            new SmartLogistics()
        };

        // Один і той самий виклик PlanDelivery — різна поведінка залежно від компанії.
        foreach (var company in companies)
        {
            company.PlanDelivery("Львів", 20);       // малий вантаж
            company.PlanDelivery("Одеса", 8_000);    // середній вантаж
            company.PlanDelivery("Стамбул", 80_000); // великий вантаж
        }
    }
}
```

### Очікуваний результат (фрагмент для SmartLogistics)

```
[SmartDeliver AI] Замовлення: 20 кг → Львів
  → SmartAI: малий вантаж, вибираємо Дрон
  🚁 Дрон: доставляю 20 кг до 'Львів' повітрям.
     Прибуття через: ~5 год.
  ✅ Доставка заплановано компанією 'SmartDeliver AI'.

[SmartDeliver AI] Замовлення: 8000 кг → Одеса
  → SmartAI: середній вантаж, вибираємо Вантажівку
  🚛 Вантажівка: везу 8000 кг до 'Одеса' по автошляху.
     Прибуття через: ~18 год.
  ✅ Доставка заплановано компанією 'SmartDeliver AI'.
```

---

## Приклад 3 — Система сповіщень

Цей приклад демонструє Factory Method у поєднанні з параметризацією — коли creator отримує дані через конструктор.

```csharp
// ─── ПРОДУКТИ ─────────────────────────────────────────────────────────────────

public interface INotification
{
    string Channel { get; }
    bool Send(string recipient, string message);
}

public class EmailNotification : INotification
{
    private readonly string _smtpHost;
    private readonly int _smtpPort;

    public EmailNotification(string smtpHost, int smtpPort)
    {
        _smtpHost = smtpHost;
        _smtpPort = smtpPort;
    }

    public string Channel => "Email";

    public bool Send(string recipient, string message)
    {
        // У реальному коді тут був би SmtpClient
        Console.WriteLine($"  📧 Email через {_smtpHost}:{_smtpPort}");
        Console.WriteLine($"     Кому: {recipient}");
        Console.WriteLine($"     Повідомлення: {message}");
        return true;
    }
}

public class SmsNotification : INotification
{
    private readonly string _apiKey;
    private readonly string _senderPhone;

    public SmsNotification(string apiKey, string senderPhone)
    {
        _apiKey = apiKey;
        _senderPhone = senderPhone;
    }

    public string Channel => "SMS";

    public bool Send(string recipient, string message)
    {
        // У реальному коді тут був би HTTP запит до SMS API
        Console.WriteLine($"  📱 SMS від {_senderPhone} (API key: {_apiKey[..4]}...)");
        Console.WriteLine($"     Кому: {recipient}");
        Console.WriteLine($"     Текст: {message[..Math.Min(160, message.Length)]}");
        return true;
    }
}

public class PushNotification : INotification
{
    private readonly string _fcmToken;

    public PushNotification(string fcmToken)
    {
        _fcmToken = fcmToken;
    }

    public string Channel => "Push";

    public bool Send(string recipient, string message)
    {
        Console.WriteLine($"  🔔 Push через FCM (token: {_fcmToken[..8]}...)");
        Console.WriteLine($"     Пристрій: {recipient}");
        Console.WriteLine($"     Сповіщення: {message}");
        return true;
    }
}

// ─── CREATOR ─────────────────────────────────────────────────────────────────

public abstract class NotificationSender
{
    // Factory Method — підкласи знають як створити конкретний тип сповіщення.
    protected abstract INotification CreateNotification();

    // Бізнес-логіка відправки — однакова для всіх каналів.
    public void Notify(string recipient, string message, bool urgent = false)
    {
        var notification = CreateNotification();

        Console.WriteLine($"\n[{notification.Channel}] Відправляю сповіщення...");

        if (urgent)
        {
            message = $"🚨 ТЕРМІНОВО: {message}";
            Console.WriteLine("  ⚡ Позначено як термінове!");
        }

        bool success = notification.Send(recipient, message);

        Console.WriteLine(success
            ? $"  ✅ Сповіщення успішно відправлено через {notification.Channel}."
            : $"  ❌ Помилка відправки через {notification.Channel}.");
    }

    // Масова розсилка — також використовує Factory Method внутрішньо.
    public void BroadcastNotify(IEnumerable<string> recipients, string message)
    {
        Console.WriteLine($"\n[Broadcast] Масова розсилка: {message}");
        int count = 0;
        foreach (var recipient in recipients)
        {
            var notification = CreateNotification();
            notification.Send(recipient, message);
            count++;
        }
        Console.WriteLine($"  ✅ Надіслано {count} сповіщень.");
    }
}

// ─── CONCRETE CREATORS ───────────────────────────────────────────────────────

// Creator налаштовується через конструктор — параметри передаються у продукт.
public class EmailNotificationSender : NotificationSender
{
    private readonly string _smtpHost;
    private readonly int _smtpPort;

    public EmailNotificationSender(string smtpHost, int smtpPort)
    {
        _smtpHost = smtpHost;
        _smtpPort = smtpPort;
    }

    protected override INotification CreateNotification()
        => new EmailNotification(_smtpHost, _smtpPort);
}

public class SmsNotificationSender : NotificationSender
{
    private readonly string _apiKey;
    private readonly string _senderPhone;

    public SmsNotificationSender(string apiKey, string senderPhone)
    {
        _apiKey = apiKey;
        _senderPhone = senderPhone;
    }

    protected override INotification CreateNotification()
        => new SmsNotification(_apiKey, _senderPhone);
}

public class PushNotificationSender : NotificationSender
{
    private readonly string _fcmToken;

    public PushNotificationSender(string fcmToken)
    {
        _fcmToken = fcmToken;
    }

    protected override INotification CreateNotification()
        => new PushNotification(_fcmToken);
}
```

### Конфігурація через налаштування (реалістичне використання)

```csharp
// Симуляція читання конфігурації (наприклад, з appsettings.json)
static NotificationSender CreateSenderFromConfig(string channel)
{
    return channel switch
    {
        "email" => new EmailNotificationSender("smtp.gmail.com", 587),
        "sms"   => new SmsNotificationSender("sk_live_abc123xyz", "+380441234567"),
        "push"  => new PushNotificationSender("AAAAabc123:APA91bXyz..."),
        _       => throw new ArgumentException($"Невідомий канал: {channel}")
    };
}

class Program
{
    static void Main()
    {
        // Тип відправника визначається один раз (з конфігу, середовища тощо).
        // Далі код не знає і не дбає про деталі реалізації.
        var sender = CreateSenderFromConfig("email");

        sender.Notify("user@example.com", "Ваше замовлення підтверджено!");
        sender.Notify("user@example.com", "Сервер недоступний!", urgent: true);

        sender.BroadcastNotify(
            new[] { "a@b.com", "c@d.com", "e@f.com" },
            "Планові роботи 15.01 з 02:00 до 04:00"
        );
    }
}
```

---

## Приклад 4 — Реальний сценарій: парсери документів

Повноцінний приклад, готовий до адаптації у реальному проекті.

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;

// ─── МОДЕЛЬ ДАНИХ ─────────────────────────────────────────────────────────────

public class ParsedDocument
{
    public string Title      { get; set; }
    public string Author     { get; set; }
    public DateTime ParsedAt { get; set; }
    public List<string> Sections { get; set; } = new();
    public Dictionary<string, string> Metadata { get; set; } = new();

    public override string ToString()
    {
        var sb = new StringBuilder();
        sb.AppendLine($"  Назва:   {Title}");
        sb.AppendLine($"  Автор:   {Author}");
        sb.AppendLine($"  Дата:    {ParsedAt:dd.MM.yyyy HH:mm}");
        sb.AppendLine($"  Розділів: {Sections.Count}");

        if (Metadata.Any())
        {
            sb.AppendLine("  Метадані:");
            foreach (var (key, value) in Metadata)
                sb.AppendLine($"    {key}: {value}");
        }
        return sb.ToString();
    }
}

// ─── ПРОДУКТИ ─────────────────────────────────────────────────────────────────

public interface IDocumentParser
{
    string SupportedFormat { get; }
    bool CanParse(string filePath);
    ParsedDocument Parse(string content);
}

// Парсер Markdown файлів (.md)
public class MarkdownParser : IDocumentParser
{
    public string SupportedFormat => "Markdown";

    public bool CanParse(string filePath)
        => filePath.EndsWith(".md", StringComparison.OrdinalIgnoreCase);

    public ParsedDocument Parse(string content)
    {
        var lines = content.Split('\n');
        var doc = new ParsedDocument
        {
            ParsedAt = DateTime.Now,
            Metadata = { ["format"] = "Markdown" }
        };

        foreach (var line in lines)
        {
            var trimmed = line.Trim();

            // Перший заголовок H1 — це назва документа
            if (trimmed.StartsWith("# ") && doc.Title == null)
            {
                doc.Title = trimmed[2..];
                continue;
            }

            // Заголовок H2 — розділ
            if (trimmed.StartsWith("## "))
            {
                doc.Sections.Add(trimmed[3..]);
                continue;
            }

            // Метадані у форматі <!-- author: Ім'я -->
            if (trimmed.StartsWith("<!-- author:"))
            {
                doc.Author = trimmed.Replace("<!-- author:", "").Replace("-->", "").Trim();
            }
        }

        doc.Title  ??= "Без назви";
        doc.Author ??= "Невідомий автор";
        doc.Metadata["sections_count"] = doc.Sections.Count.ToString();
        return doc;
    }
}

// Парсер CSV файлів (.csv)
public class CsvParser : IDocumentParser
{
    private readonly char _delimiter;

    public CsvParser(char delimiter = ',')
    {
        _delimiter = delimiter;
    }

    public string SupportedFormat => "CSV";

    public bool CanParse(string filePath)
        => filePath.EndsWith(".csv", StringComparison.OrdinalIgnoreCase);

    public ParsedDocument Parse(string content)
    {
        var lines = content.Split('\n', StringSplitOptions.RemoveEmptyEntries);
        var doc = new ParsedDocument
        {
            ParsedAt = DateTime.Now,
            Metadata = { ["format"] = "CSV", ["delimiter"] = _delimiter.ToString() }
        };

        if (lines.Length == 0) return doc;

        // Перший рядок — заголовки стовпців (вони ж "розділи" у нашій моделі)
        var headers = lines[0].Split(_delimiter);
        doc.Title = $"CSV таблиця ({headers.Length} колонок)";
        doc.Author = "Система";
        doc.Sections.AddRange(headers.Select(h => h.Trim()));

        doc.Metadata["rows"]    = (lines.Length - 1).ToString();
        doc.Metadata["columns"] = headers.Length.ToString();
        return doc;
    }
}

// Парсер JSON файлів (.json)
public class JsonParser : IDocumentParser
{
    public string SupportedFormat => "JSON";

    public bool CanParse(string filePath)
        => filePath.EndsWith(".json", StringComparison.OrdinalIgnoreCase);

    public ParsedDocument Parse(string content)
    {
        // Спрощений парсер для демонстрації (у реальному коді — System.Text.Json)
        var doc = new ParsedDocument
        {
            ParsedAt = DateTime.Now,
            Metadata = { ["format"] = "JSON" }
        };

        // Витягуємо прості значення ключів за наявності
        doc.Title  = ExtractJsonValue(content, "title")  ?? "JSON документ";
        doc.Author = ExtractJsonValue(content, "author") ?? "Невідомий";

        // Рахуємо верхньорівневі ключі як "розділи"
        var depth = 0;
        foreach (var ch in content)
        {
            if (ch == '{') depth++;
            if (ch == '}') depth--;
            if (ch == '"' && depth == 1) doc.Sections.Add("ключ");
        }

        doc.Metadata["approx_keys"] = (doc.Sections.Count / 2).ToString();
        doc.Sections.Clear();
        doc.Sections.Add("Корінь JSON об'єкту");
        return doc;
    }

    private static string ExtractJsonValue(string json, string key)
    {
        var search = $"\"{key}\"";
        var idx = json.IndexOf(search, StringComparison.OrdinalIgnoreCase);
        if (idx < 0) return null;

        var colonIdx = json.IndexOf(':', idx);
        if (colonIdx < 0) return null;

        var start = json.IndexOf('"', colonIdx) + 1;
        var end   = json.IndexOf('"', start);
        return (start > 0 && end > start) ? json[start..end] : null;
    }
}

// ─── CREATOR ─────────────────────────────────────────────────────────────────

public abstract class DocumentProcessor
{
    // Factory Method
    protected abstract IDocumentParser CreateParser();

    // Повна бізнес-логіка обробки — незалежна від типу файлу.
    public void ProcessDocument(string filePath, string content)
    {
        var parser = CreateParser();

        Console.WriteLine($"\n[{parser.SupportedFormat}] Обробка: {filePath}");

        if (!parser.CanParse(filePath))
        {
            Console.WriteLine($"  ⚠️  Парсер {parser.SupportedFormat} не підтримує цей формат!");
            return;
        }

        try
        {
            var doc = parser.Parse(content);
            Console.WriteLine("  ✅ Парсинг успішний:");
            Console.Write(doc.ToString());
            PostProcess(doc);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"  ❌ Помилка парсингу: {ex.Message}");
        }
    }

    // Хук — підкласи можуть перевизначити для додаткової обробки.
    protected virtual void PostProcess(ParsedDocument doc)
    {
        Console.WriteLine($"  → Документ передано до сховища.");
    }
}

// ─── CONCRETE CREATORS ───────────────────────────────────────────────────────

public class MarkdownProcessor : DocumentProcessor
{
    protected override IDocumentParser CreateParser() => new MarkdownParser();

    protected override void PostProcess(ParsedDocument doc)
    {
        Console.WriteLine($"  → Markdown: генерую HTML-превʼю для '{doc.Title}'...");
        base.PostProcess(doc);
    }
}

public class CsvProcessor : DocumentProcessor
{
    private readonly char _delimiter;

    public CsvProcessor(char delimiter = ',') => _delimiter = delimiter;

    protected override IDocumentParser CreateParser() => new CsvParser(_delimiter);

    protected override void PostProcess(ParsedDocument doc)
    {
        Console.WriteLine($"  → CSV: імпортую {doc.Metadata["rows"]} рядків до бази даних...");
        base.PostProcess(doc);
    }
}

public class JsonProcessor : DocumentProcessor
{
    protected override IDocumentParser CreateParser() => new JsonParser();

    protected override void PostProcess(ParsedDocument doc)
    {
        Console.WriteLine($"  → JSON: валідую схему документу...");
        base.PostProcess(doc);
    }
}
```

### Фабрика для автоматичного вибору процесора

```csharp
// Додатково: Simple Factory для зручного вибору потрібного процесора.
// (Це не Factory Method — це Simple Factory, але вони часто йдуть разом)
public static class DocumentProcessorFactory
{
    public static DocumentProcessor GetProcessor(string filePath)
    {
        var ext = System.IO.Path.GetExtension(filePath).ToLower();

        return ext switch
        {
            ".md"   => new MarkdownProcessor(),
            ".csv"  => new CsvProcessor(';'), // Наприклад, з крапкою з комою
            ".json" => new JsonProcessor(),
            _       => throw new NotSupportedException($"Формат '{ext}' не підтримується.")
        };
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        Console.WriteLine("=== Система обробки документів ===");

        // Тестові дані
        var files = new[]
        {
            (
                path: "report.md",
                content: "# Звіт за квартал\n<!-- author: Іван Петренко -->\n## Доходи\n## Витрати\n## Прибуток"
            ),
            (
                path: "data.csv",
                content: "Ім'я;Вік;Місто\nОлена;28;Київ\nМарко;35;Львів\nСофія;22;Харків"
            ),
            (
                path: "config.json",
                content: "{\"title\": \"Конфігурація\", \"author\": \"DevTeam\", \"version\": \"1.0\"}"
            )
        };

        foreach (var (path, content) in files)
        {
            // Автоматичний вибір процесора за розширенням файлу
            var processor = DocumentProcessorFactory.GetProcessor(path);
            processor.ProcessDocument(path, content);
        }
    }
}
```

---

## Factory Method vs Simple Factory vs Abstract Factory

Ці три поняття часто плутають. Ось чітке розмежування.

### Simple Factory (не GoF патерн)

Це просто статичний метод або клас, що повертає різні об'єкти залежно від параметра. **Не є патерном GoF**, але широко використовується.

```csharp
// Simple Factory — НЕ Factory Method
public static class ButtonFactory
{
    public static IButton Create(string os)
    {
        return os switch
        {
            "Windows" => new WindowsButton(),
            "Mac"     => new MacButton(),
            _         => throw new ArgumentException($"Невідомий OS: {os}")
        };
    }
}

// Використання
IButton btn = ButtonFactory.Create("Windows");
```

**Мінус:** щоб додати новий тип — потрібно змінювати `ButtonFactory`. Порушує OCP.

### Factory Method (GoF)

Метод у базовому класі, який перевизначається підкласами. Розширення — через новий підклас, не зміну існуючого коду.

```csharp
// Factory Method — через наслідування
public abstract class Dialog
{
    protected abstract IButton CreateButton(); // ← Factory Method
}

public class LinuxDialog : Dialog
{
    protected override IButton CreateButton() => new LinuxButton(); // Новий підклас
}
```

**Плюс:** додаємо новий тип — просто новий підклас. Старий код не змінюється.

### Abstract Factory (GoF)

Інтерфейс для створення **сімейств пов'язаних об'єктів**. Factory Method — один метод, Abstract Factory — кілька пов'язаних методів.

```csharp
// Abstract Factory — ціле сімейство об'єктів
public interface IUIFactory
{
    IButton CreateButton();    // ← Factory Method #1
    ICheckbox CreateCheckbox(); // ← Factory Method #2
    ITextInput CreateTextInput(); // ← Factory Method #3
}

public class WindowsUIFactory : IUIFactory
{
    public IButton CreateButton()       => new WindowsButton();
    public ICheckbox CreateCheckbox()   => new WindowsCheckbox();
    public ITextInput CreateTextInput() => new WindowsTextInput();
}
```

### Коли що використовувати

| Ситуація | Патерн |
|---|---|
| Потрібно створити один тип об'єктів, підкласи вирішують який саме | **Factory Method** |
| Простий вибір з кількох типів в одному місці | **Simple Factory** |
| Потрібно гарантувати сумісність між кількома типами об'єктів | **Abstract Factory** |

---

## Переваги та недоліки

### Переваги

- **Принцип відкритості/закритості (OCP):** нові типи продуктів — нові підкласи, без зміни існуючого коду.
- **Принцип єдиної відповідальності (SRP):** код створення продукту відокремлено від коду його використання.
- **Слабка зв'язність:** Creator залежить від інтерфейсу `IProduct`, а не від конкретних класів.
- **Легко тестувати:** у тестах можна підставити TestDialog з перевизначеним `CreateButton()`.

### Недоліки

- **Більше класів:** для кожного нового типу продукту потрібен новий підклас Creator. Кількість класів зростає.
- **Складніша ієрархія:** при глибокому наслідуванні код стає важким для розуміння.
- **Надмірність для простих випадків:** якщо продукт один і ніколи не зміниться — Simple Factory простіший.

---

## Антипатерни та поширені помилки

### Помилка 1 — Не використовувати інтерфейс для продукту

```csharp
// НЕПРАВИЛЬНО: повертає конкретний тип, а не інтерфейс.
public abstract class Dialog
{
    protected abstract WindowsButton CreateButton(); // ← WindowsButton, не IButton!
}

// Тепер підклас MacDialog НЕ МОЖЕ повернути MacButton — він змушений повертати WindowsButton.
// Вся ідея патерну зламана.

// ПРАВИЛЬНО:
protected abstract IButton CreateButton(); // ← IButton
```

### Помилка 2 — Логіка вибору в базовому класі

```csharp
// НЕПРАВИЛЬНО: базовий клас сам вирішує що створювати.
public class Dialog
{
    private string _os;

    // Це вже не Factory Method — це Simple Factory всередині базового класу.
    // Щоб додати новий тип — треба змінювати базовий клас.
    protected IButton CreateButton()
    {
        if (_os == "Windows") return new WindowsButton();
        if (_os == "Mac")     return new MacButton();
        throw new Exception("Невідомий OS");
    }
}

// ПРАВИЛЬНО: логіка виноситься в підкласи.
public abstract class Dialog
{
    protected abstract IButton CreateButton(); // Базовий не знає — підкласи вирішують.
}
```

### Помилка 3 — Factory Method без інтерфейсу Creator

```csharp
// НЕПРАВИЛЬНО: нема абстракції для Creator.
// Клієнтський код залежить від конкретного WindowsDialog.
WindowsDialog dialog = new WindowsDialog();
dialog.Render();

// ПРАВИЛЬНО: клієнт використовує базовий тип.
Dialog dialog = new WindowsDialog(); // ← Тип Dialog
dialog.Render();
// Тепер можна замінити на MacDialog без зміни клієнтського коду.
```

### Помилка 4 — Забути що Factory Method може мати реалізацію за замовчуванням

```csharp
// Factory Method не обов'язково абстрактний!
// Базовий клас може надати реалізацію за замовчуванням.
public class Dialog
{
    // virtual (не abstract) — підкласи МОЖУТЬ перевизначити, але не зобов'язані.
    protected virtual IButton CreateButton()
    {
        return new DefaultButton(); // Реалізація за замовчуванням
    }
}

public class SpecialDialog : Dialog
{
    // Перевизначаємо тільки якщо потрібна інша кнопка.
    protected override IButton CreateButton() => new SpecialButton();
}

public class OrdinaryDialog : Dialog
{
    // Нічого не перевизначаємо — використовується DefaultButton.
}
```

---

## Підсумок

Factory Method — це патерн для ситуацій, коли клас **знає що робити** з об'єктом, але **не знає який саме** об'єкт потрібно створити.

### Швидка схема прийняття рішення

```
Тобі потрібно створювати об'єкти, і...
│
├─► тип визначається підкласом
│   → Factory Method
│
├─► тип визначається параметром в одному місці
│   → Simple Factory
│
└─► потрібне ціле сімейство пов'язаних об'єктів
    → Abstract Factory
```

### Мінімальний шаблон для C#

```csharp
// 1. Інтерфейс продукту
public interface IProduct
{
    void Operation();
}

// 2. Конкретні продукти
public class ConcreteProductA : IProduct
{
    public void Operation() => Console.WriteLine("Продукт A");
}

public class ConcreteProductB : IProduct
{
    public void Operation() => Console.WriteLine("Продукт B");
}

// 3. Абстрактний Creator з Factory Method
public abstract class Creator
{
    protected abstract IProduct CreateProduct(); // Factory Method

    public void SomeOperation()
    {
        var product = CreateProduct(); // Використовуємо без знання конкретного типу
        product.Operation();
    }
}

// 4. Конкретні Creator — перевизначають Factory Method
public sealed class CreatorA : Creator
{
    protected override IProduct CreateProduct() => new ConcreteProductA();
}

public sealed class CreatorB : Creator
{
    protected override IProduct CreateProduct() => new ConcreteProductB();
}
```

---

*Документ підготовлено для вивчення патернів проектування. Всі приклади протестовані на .NET 6+.*
