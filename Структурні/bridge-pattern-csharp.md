# Патерн Bridge (Міст) у C#

> **Категорія:** Структурний патерн (Structural Pattern)  
> **Мова прикладів:** C# (.NET)

---

## Зміст

1. [Що таке Bridge?](#що-таке-bridge)
2. [Проблема: вибух класів при успадкуванні](#проблема-вибух-класів-при-успадкуванні)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Простий (Форми і рендерери)](#приклад-1--простий-форми-і-рендерери)
5. [Приклад 2 — Просунутий (Система сповіщень)](#приклад-2--просунутий-система-сповіщень)
6. [Bridge vs Adapter vs Strategy](#bridge-vs-adapter-vs-strategy)
7. [Переваги та недоліки](#переваги-та-недоліки)
8. [Підсумок](#підсумок)

---

## Що таке Bridge?

**Bridge** — це структурний патерн, який **розділяє абстракцію від реалізації** так, щоб вони могли змінюватись незалежно одна від одної.

Замість того, щоб створювати один великий клас (або ієрархію класів), де абстракція і реалізація переплетені, Bridge виносить реалізацію в окрему ієрархію і з'єднує їх через композицію — **міст**.

> Ключова фраза з GoF: *"prefer composition over inheritance"* — Bridge є прямим втіленням цього принципу.

### Головна ідея одним реченням

```
Є дві незалежні осі змін → розділи їх на дві ієрархії → з'єднай мостом (полем).
```

---

## Проблема: вибух класів при успадкуванні

Уявіть систему UI-компонентів. Є кілька типів елементів і кілька тем оформлення:

```
Без Bridge — всі комбінації через успадкування:

UIComponent
├── Button
│   ├── LightButton
│   └── DarkButton
├── Checkbox
│   ├── LightCheckbox
│   └── DarkCheckbox
└── TextInput
    ├── LightTextInput
    └── DarkTextInput
```

**3 компоненти × 2 теми = 6 класів.** Додаємо третю тему `HighContrast` → вже **9 класів**. Додаємо четвертий компонент → ще **3 класи** для кожної теми.

Формула: **M компонентів × N тем = M×N класів**.

При M=5, N=4 → **20 класів**, більшість з яких — дублювання.

### Рішення з Bridge

```
З Bridge — дві незалежні ієрархії:

Абстракція          Реалізація (міст →)
UIComponent ────────▶ ITheme
├── Button           ├── LightTheme
├── Checkbox         ├── DarkTheme
└── TextInput        └── HighContrastTheme
```

**3 + 3 = 6 класів** замість 9. При M=5, N=4 → **5 + 4 = 9 класів** замість 20. Кожна вісь розширюється незалежно.

---

## Структура патерну

```
      «Абстракція»                    «Реалізація»
  ┌──────────────────┐           ┌──────────────────┐
  │   Abstraction    │           │ «interface»      │
  │                  │           │ IImplementation  │
  │  - _impl ────────┼──────────▶│ + OperationImpl()│
  │  + Operation()   │           └────────┬─────────┘
  └────────┬─────────┘                    │
           │ extends                      │ implements
  ┌────────▼─────────┐      ┌─────────────┴──────────┐
  │ RefinedAbstract. │      │                        │
  │                  │  ┌───▼──────────┐  ┌──────────▼───┐
  │ + Operation()    │  │ ConcreteImpl │  │ ConcreteImpl │
  └──────────────────┘  │      A       │  │      B       │
                        └──────────────┘  └──────────────┘
```

Учасники:
- **Abstraction** — "висока рівнева" логіка. Містить посилання на `IImplementation` (це і є "міст")
- **RefinedAbstraction** — розширення абстракції (необов'язково)
- **IImplementation** — інтерфейс для реалізацій, зазвичай низькорівневий
- **ConcreteImplementation** — конкретні реалізації платформи, формату, протоколу тощо

---

## Приклад 1 — Простий (Форми і рендерери)

Є геометричні фігури (коло, прямокутник) і два рендерери (векторний SVG і растровий PNG). Хочемо комбінувати будь-яку фігуру з будь-яким рендером.

### Крок 1: Реалізація (IRenderer) — вісь "як малювати"

```csharp
// Інтерфейс реалізації — низькорівневі операції малювання
// Це "права" частина мосту
public interface IRenderer
{
    string RendererType { get; }
    void RenderCircle(float x, float y, float radius);
    void RenderRectangle(float x, float y, float width, float height);
}
```

```csharp
// Конкретна реалізація A — векторний SVG рендер
public class SvgRenderer : IRenderer
{
    public string RendererType => "SVG";

    public void RenderCircle(float x, float y, float radius)
    {
        Console.WriteLine($"<circle cx=\"{x}\" cy=\"{y}\" r=\"{radius}\" />");
    }

    public void RenderRectangle(float x, float y, float width, float height)
    {
        Console.WriteLine($"<rect x=\"{x}\" y=\"{y}\" width=\"{width}\" height=\"{height}\" />");
    }
}
```

```csharp
// Конкретна реалізація B — растровий PNG рендер (піксельний)
public class RasterRenderer : IRenderer
{
    public string RendererType => "Raster";

    public void RenderCircle(float x, float y, float radius)
    {
        Console.WriteLine($"[Raster] DrawCircle: center=({x},{y}), r={radius}, pixels≈{(int)(Math.PI * radius * radius)}");
    }

    public void RenderRectangle(float x, float y, float width, float height)
    {
        Console.WriteLine($"[Raster] FillRect: ({x},{y}), {width}×{height}, pixels={(int)(width * height)}");
    }
}
```

### Крок 2: Абстракція (Shape) — вісь "що малювати"

```csharp
// Абстракція — "ліва" частина мосту
// Містить посилання на IRenderer — це і є міст
public abstract class Shape
{
    // Міст: посилання на реалізацію
    protected readonly IRenderer _renderer;

    protected Shape(IRenderer renderer)
    {
        _renderer = renderer;
    }

    // Абстрактний метод — кожна фігура визначає свою логіку
    public abstract void Draw();
    public abstract void Resize(float factor);

    public override string ToString() => $"{GetType().Name} via {_renderer.RendererType}";
}
```

```csharp
// Уточнена абстракція A — Коло
public class Circle : Shape
{
    private float _x, _y, _radius;

    public Circle(float x, float y, float radius, IRenderer renderer)
        : base(renderer)
    {
        _x = x;
        _y = y;
        _radius = radius;
    }

    // Висока логіка: "намалювати коло" — делегує деталі рендеру
    public override void Draw()
    {
        Console.Write($"Drawing Circle ({_x},{_y}) r={_radius} → ");
        _renderer.RenderCircle(_x, _y, _radius);  // ← міст
    }

    public override void Resize(float factor)
    {
        _radius *= factor;
    }
}
```

```csharp
// Уточнена абстракція B — Прямокутник
public class Rectangle : Shape
{
    private float _x, _y, _width, _height;

    public Rectangle(float x, float y, float width, float height, IRenderer renderer)
        : base(renderer)
    {
        _x = x; _y = y; _width = width; _height = height;
    }

    public override void Draw()
    {
        Console.Write($"Drawing Rect ({_x},{_y}) {_width}×{_height} → ");
        _renderer.RenderRectangle(_x, _y, _width, _height); // ← міст
    }

    public override void Resize(float factor)
    {
        _width *= factor;
        _height *= factor;
    }
}
```

### Крок 3: Використання

```csharp
class Program
{
    static void Main()
    {
        // Реалізації — незалежні від фігур
        IRenderer svg    = new SvgRenderer();
        IRenderer raster = new RasterRenderer();

        Console.WriteLine("=== SVG рендер ===");
        Shape circle1 = new Circle(50, 50, 30, svg);
        Shape rect1   = new Rectangle(10, 10, 100, 60, svg);
        circle1.Draw();
        rect1.Draw();

        Console.WriteLine("\n=== Raster рендер ===");
        Shape circle2 = new Circle(50, 50, 30, raster);
        Shape rect2   = new Rectangle(10, 10, 100, 60, raster);
        circle2.Draw();
        rect2.Draw();

        Console.WriteLine("\n=== Змінюємо розмір і малюємо знову ===");
        circle1.Resize(2.0f); // збільшуємо вдвічі
        circle1.Draw();

        Console.WriteLine("\n=== Додаємо новий рендер — OpenGL (не змінюємо фігури!) ===");
        // Нова реалізація — фігури не треба чіпати взагалі
        IRenderer openGl = new OpenGlRenderer();
        Shape circle3 = new Circle(50, 50, 30, openGl);
        circle3.Draw();
    }
}

// Нова реалізація додається незалежно від ієрархії фігур
public class OpenGlRenderer : IRenderer
{
    public string RendererType => "OpenGL";

    public void RenderCircle(float x, float y, float radius)
        => Console.WriteLine($"glDrawCircle({x}f, {y}f, {radius}f);");

    public void RenderRectangle(float x, float y, float width, float height)
        => Console.WriteLine($"glDrawRect({x}f, {y}f, {width}f, {height}f);");
}
```

### Очікуваний вивід

```
=== SVG рендер ===
Drawing Circle (50,50) r=30 → <circle cx="50" cy="50" r="30" />
Drawing Rect (10,10) 100×60 → <rect x="10" y="10" width="100" height="60" />

=== Raster рендер ===
Drawing Circle (50,50) r=30 → [Raster] DrawCircle: center=(50,50), r=30, pixels≈2827
Drawing Rect (10,10) 100×60 → [Raster] FillRect: (10,10), 100×60, pixels=6000

=== Змінюємо розмір і малюємо знову ===
Drawing Circle (50,50) r=60 → <circle cx="50" cy="50" r="60" />

=== Додаємо новий рендер — OpenGL (не змінюємо фігури!) ===
Drawing Circle (50,50) r=30 → glDrawCircle(50f, 50f, 30f);
```

---

## Приклад 2 — Просунутий (Система сповіщень)

Реалістичний сценарій: система сповіщень. Є **типи сповіщень** (Info, Warning, Alert) і **канали доставки** (Email, SMS, Slack, Push). Обидва виміри змінюються незалежно.

### Крок 1: Реалізація — канали доставки (вісь "куди надсилати")

```csharp
// Інтерфейс реалізації — канал доставки повідомлень
public interface IMessageChannel
{
    string ChannelName { get; }
    void Send(string recipient, string subject, string body, MessagePriority priority);
}

// Пріоритет — впливає на поведінку деяких каналів
public enum MessagePriority { Low, Normal, High, Critical }
```

```csharp
// Email-канал
public class EmailChannel : IMessageChannel
{
    private readonly string _smtpServer;

    public string ChannelName => "Email";

    public EmailChannel(string smtpServer = "smtp.example.com")
    {
        _smtpServer = smtpServer;
    }

    public void Send(string recipient, string subject, string body, MessagePriority priority)
    {
        var priorityHeader = priority >= MessagePriority.High ? "[ВАЖЛИВО] " : "";
        Console.WriteLine($"[EMAIL via {_smtpServer}]");
        Console.WriteLine($"  To: {recipient}");
        Console.WriteLine($"  Subject: {priorityHeader}{subject}");
        Console.WriteLine($"  Body: {body}");
    }
}
```

```csharp
// SMS-канал — короткі повідомлення, тільки для High/Critical
public class SmsChannel : IMessageChannel
{
    private readonly string _apiKey;

    public string ChannelName => "SMS";

    public SmsChannel(string apiKey)
    {
        _apiKey = apiKey;
    }

    public void Send(string recipient, string subject, string body, MessagePriority priority)
    {
        // SMS мають обмеження довжини — скорочуємо текст
        var shortBody = body.Length > 100 ? body[..97] + "..." : body;

        // Для низького пріоритету SMS не надсилаємо (дорого)
        if (priority == MessagePriority.Low)
        {
            Console.WriteLine($"[SMS] Пропущено (Low priority): {recipient}");
            return;
        }

        Console.WriteLine($"[SMS] → {recipient}: \"{subject}: {shortBody}\"");
    }
}
```

```csharp
// Slack-канал — підтримує форматування і emoji
public class SlackChannel : IMessageChannel
{
    private readonly string _webhookUrl;
    private readonly string _defaultChannel;

    public string ChannelName => "Slack";

    public SlackChannel(string webhookUrl, string defaultChannel = "#alerts")
    {
        _webhookUrl     = webhookUrl;
        _defaultChannel = defaultChannel;
    }

    public void Send(string recipient, string subject, string body, MessagePriority priority)
    {
        var emoji = priority switch
        {
            MessagePriority.Critical => "🔴",
            MessagePriority.High     => "🟠",
            MessagePriority.Normal   => "🟡",
            _                        => "🟢"
        };

        var channel = recipient.StartsWith("#") ? recipient : _defaultChannel;

        Console.WriteLine($"[SLACK → {channel}]");
        Console.WriteLine($"  {emoji} *{subject}*");
        Console.WriteLine($"  {body}");
        Console.WriteLine($"  _via webhook: {_webhookUrl[..20]}..._");
    }
}
```

### Крок 2: Абстракція — типи сповіщень (вісь "що сповіщати")

```csharp
// Базова абстракція сповіщення — містить міст до каналу доставки
public abstract class Notification
{
    // Міст — посилання на реалізацію (канал)
    protected readonly IMessageChannel _channel;

    // Налаштування сповіщення
    protected string Recipient { get; }
    protected DateTime CreatedAt { get; } = DateTime.UtcNow;

    protected Notification(string recipient, IMessageChannel channel)
    {
        Recipient = recipient;
        _channel  = channel;
    }

    // Шаблонний метод — визначає скелет алгоритму
    public void Notify(string eventTitle, string eventDetails)
    {
        // Кожен підклас додає свою специфіку (тему, форматування, пріоритет)
        var (subject, body, priority) = FormatMessage(eventTitle, eventDetails);

        Console.WriteLine($"\n--- {GetType().Name} via {_channel.ChannelName} ---");
        _channel.Send(Recipient, subject, body, priority);  // ← міст
    }

    // Підкласи визначають: як сформувати повідомлення і який пріоритет
    protected abstract (string subject, string body, MessagePriority priority)
        FormatMessage(string eventTitle, string eventDetails);
}
```

```csharp
// Інформаційне сповіщення — низький пріоритет
public class InfoNotification : Notification
{
    private readonly string _source;

    public InfoNotification(string recipient, IMessageChannel channel, string source = "System")
        : base(recipient, channel)
    {
        _source = source;
    }

    protected override (string, string, MessagePriority) FormatMessage(
        string eventTitle, string eventDetails)
    {
        var subject = $"[INFO] {eventTitle}";
        var body    = $"Джерело: {_source}\nЧас: {CreatedAt:HH:mm:ss UTC}\n\n{eventDetails}";

        return (subject, body, MessagePriority.Low);
    }
}
```

```csharp
// Попередження — середній пріоритет, додає рекомендацію до дії
public class WarningNotification : Notification
{
    private readonly string _recommendation;

    public WarningNotification(string recipient, IMessageChannel channel, string recommendation = "")
        : base(recipient, channel)
    {
        _recommendation = recommendation;
    }

    protected override (string, string, MessagePriority) FormatMessage(
        string eventTitle, string eventDetails)
    {
        var subject = $"⚠️ УВАГА: {eventTitle}";
        var body = new System.Text.StringBuilder()
            .AppendLine(eventDetails)
            .AppendLine()
            .AppendLine($"Час виявлення: {CreatedAt:dd.MM.yyyy HH:mm:ss UTC}");

        if (!string.IsNullOrEmpty(_recommendation))
            body.AppendLine($"Рекомендація: {_recommendation}");

        return (subject, body.ToString(), MessagePriority.Normal);
    }
}
```

```csharp
// Критичний Alert — найвищий пріоритет, вимагає негайної реакції
public class CriticalAlertNotification : Notification
{
    private readonly string _incidentId;
    private readonly string _onCallEngineer;

    public CriticalAlertNotification(
        string recipient,
        IMessageChannel channel,
        string incidentId,
        string onCallEngineer)
        : base(recipient, channel)
    {
        _incidentId     = incidentId;
        _onCallEngineer = onCallEngineer;
    }

    protected override (string, string, MessagePriority) FormatMessage(
        string eventTitle, string eventDetails)
    {
        var subject = $"🚨 КРИТИЧНО #{_incidentId}: {eventTitle}";
        var body = $"""
            ІНЦИДЕНТ: {_incidentId}
            СТАТУС: АКТИВНИЙ
            ЧАС: {CreatedAt:dd.MM.yyyy HH:mm:ss UTC}
            ЧЕРГОВИЙ: {_onCallEngineer}

            ДЕТАЛІ:
            {eventDetails}

            ДІЯ: Негайно перевірте систему!
            """;

        return (subject, body, MessagePriority.Critical);
    }
}
```

### Крок 3: Notification Builder для зручної конфігурації

```csharp
// Допоміжний клас — збирає сповіщення з кількома каналами одночасно
public class MultiChannelNotification
{
    private readonly List<Notification> _notifications = new();

    public MultiChannelNotification Add(Notification notification)
    {
        _notifications.Add(notification);
        return this;
    }

    // Надсилає через всі зареєстровані канали
    public void NotifyAll(string eventTitle, string eventDetails)
    {
        foreach (var notification in _notifications)
            notification.Notify(eventTitle, eventDetails);
    }
}
```

### Крок 4: Використання

```csharp
class Program
{
    static void Main()
    {
        // --- Ініціалізація каналів (реалізацій) ---
        IMessageChannel email = new EmailChannel("smtp.company.com");
        IMessageChannel sms   = new SmsChannel("sms-api-key-xyz");
        IMessageChannel slack = new SlackChannel("https://hooks.slack.com/T123/B456/xyz", "#dev-alerts");

        // ============================================================
        // Сценарій 1: Простий info — тільки email
        // ============================================================
        Console.WriteLine("╔══════════════════════════════════════╗");
        Console.WriteLine("║  Сценарій 1: Деплой завершено        ║");
        Console.WriteLine("╚══════════════════════════════════════╝");

        var deployInfo = new InfoNotification("team@company.com", email, source: "CI/CD Pipeline");
        deployInfo.Notify(
            "Деплой v2.4.1 успішно завершено",
            "Всі 142 тести пройшли. Сервіс доступний на prod-01..prod-03.");

        // ============================================================
        // Сценарій 2: Попередження через Slack
        // ============================================================
        Console.WriteLine("\n╔══════════════════════════════════════╗");
        Console.WriteLine("║  Сценарій 2: Висока завантаженість   ║");
        Console.WriteLine("╚══════════════════════════════════════╝");

        var highLoadWarning = new WarningNotification(
            "#ops-channel",
            slack,
            recommendation: "Розгляньте горизонтальне масштабування або перевірте аномальні запити");

        highLoadWarning.Notify(
            "CPU > 85% на prod-02",
            "Середнє навантаження CPU за останні 15 хвилин: 87%. Пікове: 94%.");

        // ============================================================
        // Сценарій 3: Критичний інцидент — одразу через 3 канали
        // ============================================================
        Console.WriteLine("\n╔══════════════════════════════════════╗");
        Console.WriteLine("║  Сценарій 3: БД недоступна — КРИТИЧНО║");
        Console.WriteLine("╚══════════════════════════════════════╝");

        var multiAlert = new MultiChannelNotification()
            .Add(new CriticalAlertNotification("oncall@company.com",   email, "INC-2024-441", "Іван Петренко"))
            .Add(new CriticalAlertNotification("+380501234567",        sms,   "INC-2024-441", "Іван Петренко"))
            .Add(new CriticalAlertNotification("#incidents",           slack, "INC-2024-441", "@ivan.petrenko"));

        multiAlert.NotifyAll(
            "База даних prod-db-01 не відповідає",
            "Connection timeout після 30s. Остання успішна відповідь: 03:47:12 UTC.\nАвтоматичний failover не спрацював.");

        // ============================================================
        // Демонстрація гнучкості: той самий тип, різний канал
        // ============================================================
        Console.WriteLine("\n╔══════════════════════════════════════╗");
        Console.WriteLine("║  Та сама Warning — але через SMS      ║");
        Console.WriteLine("╚══════════════════════════════════════╝");

        // Просто міняємо реалізацію (канал) — абстракція не змінюється
        var smsWarning = new WarningNotification("+380509876543", sms);
        smsWarning.Notify("Диск заповнений на 90%", "Сервер: prod-storage-02. Вільно: 48GB з 500GB.");
    }
}
```

### Очікуваний вивід

```
╔══════════════════════════════════════╗
║  Сценарій 1: Деплой завершено        ║
╚══════════════════════════════════════╝

--- InfoNotification via Email ---
[EMAIL via smtp.company.com]
  To: team@company.com
  Subject: [INFO] Деплой v2.4.1 успішно завершено
  Body: Джерело: CI/CD Pipeline
Час: 14:22:01 UTC

Всі 142 тести пройшли. Сервіс доступний на prod-01..prod-03.

╔══════════════════════════════════════╗
║  Сценарій 2: Висока завантаженість   ║
╚══════════════════════════════════════╝

--- WarningNotification via Slack ---
[SLACK → #ops-channel]
  🟡 *⚠️ УВАГА: CPU > 85% на prod-02*
  Середнє навантаження CPU за останні 15 хвилин: 87%. Пікове: 94%.
  Рекомендація: Розгляньте горизонтальне масштабування...
  _via webhook: https://hooks.slack.com..._

╔══════════════════════════════════════╗
║  Сценарій 3: БД недоступна — КРИТИЧНО║
╚══════════════════════════════════════╝

--- CriticalAlertNotification via Email ---
[EMAIL via smtp.company.com]
  To: oncall@company.com
  Subject: [ВАЖЛИВО] 🚨 КРИТИЧНО #INC-2024-441: База даних prod-db-01 не відповідає
  Body: ІНЦИДЕНТ: INC-2024-441 ...

--- CriticalAlertNotification via SMS ---
[SMS] → +380501234567: "🚨 КРИТИЧНО #INC-2024-441: База даних prod-db-01 не відповідає..."

--- CriticalAlertNotification via Slack ---
[SLACK → #incidents]
  🔴 *🚨 КРИТИЧНО #INC-2024-441: База даних prod-db-01 не відповідає*
  ...
```

---

## Bridge vs Adapter vs Strategy

Ці патерни часто плутають — всі використовують композицію і делегування.

| Ознака | Bridge | Adapter | Strategy |
|---|---|---|---|
| **Мета** | Розділити абстракцію і реалізацію, дві незалежні осі | Зробити несумісні інтерфейси сумісними | Замінити алгоритм у рантаймі |
| **Коли проєктується** | **Наперед** — при проєктуванні системи | **Після** — коли класи вже несумісні | **Наперед або після** |
| **Ієрархії** | **Дві** паралельні ієрархії | Одна обгортка | Одна ієрархія стратегій |
| **Фокус** | Структура і масштабованість | Сумісність | Поведінка і взаємозамінність |

```csharp
// Bridge: дві осі змін — обидві розширюються незалежно
// "Сповіщення" і "Канал" — дві рівноправні ієрархії
class WarningNotification { IMessageChannel _channel; }  // абстракція тримає реалізацію

// Adapter: несумісні інтерфейси — "перекладаємо" один в інший
// Є ThirdPartyLogger, потрібен ILogger — одна сторона адаптується під іншу
class ThirdPartyLoggerAdapter : ILogger { ThirdPartyLogger _lib; }

// Strategy: один алгоритм замінюється іншим, інтерфейс той самий
// Об'єкт отримує стратегію і використовує її — немає другої ієрархії "типів"
class Sorter { ISortStrategy _strategy; }
```

### Найтонша різниця: Bridge vs Strategy

Bridge і Strategy структурно схожі — обидва використовують поле-інтерфейс. Різниця — у **намірі та масштабі**:

- **Strategy** — замінює **один алгоритм** всередині об'єкта. Є один вимір: "як сортувати", "як рахувати".
- **Bridge** — розділяє **дві повноцінні ієрархії**. Обидва виміри є рівноправними і кожен може мати власну ієрархію підкласів.

---

## Переваги та недоліки

### Переваги

- **Незалежне розширення** — нові фігури не вимагають змін рендерерів, і навпаки
- **Уникнення вибуху класів** — M+N замість M×N
- **Принцип відкритості/закритості** — нові реалізації без зміни абстракцій
- **Приховування деталей реалізації** — клієнт бачить тільки абстракцію
- **Можна змінити реалізацію в рантаймі** — передати інший канал, рендер, провайдер

### Недоліки

- **Складніше проєктування** — потрібно наперед виявити дві незалежні осі змін; якщо вони одна — Bridge зайвий
- **Більше класів** — навіть при M+N, якщо M=2 і N=2, це вже 4 класи замість 4 — не завжди виграш
- **Може бути надмірним** — якщо в реальності реалізація ніколи не змінюється, простіше без Bridge

---

## Підсумок

| Аспект | Деталь |
|---|---|
| Тип патерну | Структурний (Structural) |
| Вирішує проблему | Вибух класів при двох незалежних осях змін |
| Ключова ідея | Дві ієрархії + поле-посилання між ними ("міст") |
| Формула | M+N класів замість M×N |
| Головне питання | "Чи є у мене **дві незалежні осі** змін?" |
| Альтернативи | Adapter (сумісність), Strategy (алгоритм), Decorator (поведінка) |

### Коротке правило вибору

```
Є дві незалежні осі змін (наприклад, "що" і "як")?
  ✅ Кількість комбінацій зростає лінійно M+N → Bridge

Є один несумісний інтерфейс, який треба "перекласти"?
  ✅ Adapter

Треба динамічно замінити один алгоритм?
  ✅ Strategy

Треба додати поведінку, зберігши інтерфейс?
  ✅ Decorator
```

---

*Документ підготовлено як навчальний матеріал з патернів проєктування на C#.*
