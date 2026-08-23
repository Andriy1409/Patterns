# Патерн Adapter (Адаптер) у C#

> **Категорія:** Структурний патерн (Structural Pattern)  
> **Мова прикладів:** C# (.NET)

---

## Зміст

1. [Що таке Adapter?](#що-таке-adapter)
2. [Структура патерну](#структура-патерну)
3. [Object Adapter vs Class Adapter](#object-adapter-vs-class-adapter)
4. [Приклад 1 — Простий (Різні формати логування)](#приклад-1--простий-різні-формати-логування)
5. [Приклад 2 — Просунутий (Платіжні системи)](#приклад-2--просунутий-платіжні-системи)
6. [Two-Way Adapter (двосторонній)](#two-way-adapter-двосторонній)
7. [Adapter vs Facade vs Decorator](#adapter-vs-facade-vs-decorator)
8. [Переваги та недоліки](#переваги-та-недоліки)
9. [Підсумок](#підсумок)

---

## Що таке Adapter?

**Adapter** — це структурний патерн, який дозволяє об'єктам з **несумісними інтерфейсами** працювати разом.

Аналогія з реального життя: у вас є ноутбук з USB-C, а в готелі — розетка на 220V з євровилкою. Щоб зарядитись, ви використовуєте **адаптер** — він не змінює ані ваш кабель, ані розетку в стіні. Він просто "перекладає" між ними.

У коді те саме: є **існуючий клас** з одним інтерфейсом і **клієнт**, який очікує інший інтерфейс. Adapter стоїть між ними і перетворює виклики.

### Проблема без Adapter

```csharp
// Ваш код очікує такий інтерфейс логера:
public interface ILogger
{
    void LogInfo(string message);
    void LogError(string message, Exception ex);
}

// Але стороння бібліотека (NLog, Serilog, log4net) має зовсім інший API:
public class ThirdPartyLogger
{
    public void WriteEntry(LogLevel level, string text, Exception exception = null) { }
    public void Flush() { }
}

// Проблема: не можна просто передати ThirdPartyLogger туди, де очікується ILogger
// Вони несумісні за інтерфейсом
```

### Рішення з Adapter

```csharp
// Адаптер "обгортає" сторонній логер і реалізує потрібний інтерфейс
public class ThirdPartyLoggerAdapter : ILogger
{
    private readonly ThirdPartyLogger _logger;

    public ThirdPartyLoggerAdapter(ThirdPartyLogger logger)
    {
        _logger = logger;
    }

    public void LogInfo(string message)
        => _logger.WriteEntry(LogLevel.Info, message); // "перекладаємо" виклик

    public void LogError(string message, Exception ex)
        => _logger.WriteEntry(LogLevel.Error, message, ex);
}

// Тепер клієнт може працювати з ThirdPartyLogger через звичний ILogger
ILogger logger = new ThirdPartyLoggerAdapter(new ThirdPartyLogger());
logger.LogInfo("Все працює!"); // клієнт навіть не знає, що під капотом — стороння бібліотека
```

---

## Структура патерну

```
┌──────────────┐        ┌────────────────────┐
│   Client     │───────▶│  «interface»       │
│              │        │  ITarget           │
└──────────────┘        │  + Request()       │
                        └─────────┬──────────┘
                                  │ implements
                        ┌─────────▼──────────┐
                        │   Adapter          │
                        │                   │
                        │  - adaptee        │◀──── обгортає
                        │  + Request()      │
                        └────────────────────┘
                                  │ делегує
                        ┌─────────▼──────────┐
                        │   Adaptee          │
                        │ (існуючий клас /   │
                        │  стороння бібл.)   │
                        │  + SpecificMethod()│
                        └────────────────────┘
```

Учасники:
- **ITarget** — інтерфейс, який очікує клієнт
- **Adaptee** — існуючий клас з несумісним інтерфейсом (часто — стороння бібліотека)
- **Adapter** — реалізує `ITarget`, всередині делегує виклики до `Adaptee`
- **Client** — працює тільки з `ITarget`, не знає про `Adaptee`

---

## Object Adapter vs Class Adapter

У C# є два способи реалізувати адаптер.

### Object Adapter — через композицію (рекомендований)

Адаптер **містить** екземпляр Adaptee як поле.

```csharp
public class ObjectAdapter : ITarget
{
    private readonly Adaptee _adaptee; // композиція — "має"

    public ObjectAdapter(Adaptee adaptee)
    {
        _adaptee = adaptee;
    }

    public void Request()
    {
        _adaptee.SpecificRequest(); // делегуємо
    }
}
```

**Переваги:** Можна адаптувати будь-який об'єкт Adaptee (навіть підклас). Слабке зв'язування.

### Class Adapter — через множинне успадкування

Адаптер **успадковує** і Adaptee, і реалізує ITarget. У C# це можливо, оскільки можна успадкувати клас і реалізовувати інтерфейс одночасно.

```csharp
public class ClassAdapter : Adaptee, ITarget // успадкування + інтерфейс
{
    public void Request()
    {
        this.SpecificRequest(); // викликаємо метод батьківського класу
    }
}
```

**Недолік:** Щільне зв'язування з конкретним класом. Не можна адаптувати підкласи. У C# немає множинного успадкування класів, тому цей підхід обмежений.

> **Висновок:** У C# майже завжди використовуйте **Object Adapter** (через композицію). Це гнучкіше та правильніше.

---

## Приклад 1 — Простий (Різні формати логування)

Є система з власним інтерфейсом логера. Потрібно підключити три різних логери: консольний, файловий та сторонній — не змінюючи клієнтський код.

### Крок 1: Цільовий інтерфейс (те, що очікує система)

```csharp
// Це інтерфейс, який знає і використовує вся система
public interface IAppLogger
{
    void Info(string message);
    void Warning(string message);
    void Error(string message, Exception exception = null);
}
```

### Крок 2: Перший Adaptee — консольний логер зі старим API

```csharp
// Старий клас, який не можна змінити (наприклад, є в legacy-коді)
public class LegacyConsoleLogger
{
    // Зовсім інший API: один метод з числовим рівнем
    public void Log(int severity, string text)
    {
        var level = severity switch
        {
            0 => "INFO",
            1 => "WARN",
            2 => "ERROR",
            _ => "UNKNOWN"
        };
        Console.WriteLine($"[{DateTime.Now:HH:mm:ss}] [{level}] {text}");
    }
}
```

### Крок 3: Адаптер для LegacyConsoleLogger

```csharp
public class LegacyConsoleLoggerAdapter : IAppLogger
{
    private readonly LegacyConsoleLogger _logger;

    public LegacyConsoleLoggerAdapter(LegacyConsoleLogger logger)
    {
        _logger = logger;
    }

    // Перетворюємо зручний виклик Info() → Log(0, ...)
    public void Info(string message)    => _logger.Log(0, message);
    public void Warning(string message) => _logger.Log(1, message);

    public void Error(string message, Exception exception = null)
    {
        var text = exception != null
            ? $"{message} | Exception: {exception.Message}"
            : message;
        _logger.Log(2, text);
    }
}
```

### Крок 4: Другий Adaptee — сторонній JSON-логер

```csharp
// Уявна стороння бібліотека — зовсім інший стиль API
public class JsonFileLogger
{
    private readonly string _filePath;

    public JsonFileLogger(string filePath)
    {
        _filePath = filePath;
    }

    // API: приймає словник полів, записує JSON
    public void Write(Dictionary<string, object> logEntry)
    {
        // Серіалізуємо вручну для простоти прикладу
        var parts = logEntry.Select(kv => $"\"{kv.Key}\": \"{kv.Value}\"");
        var json = "{ " + string.Join(", ", parts) + " }";
        Console.WriteLine($"[JSON→{_filePath}] {json}");
    }
}
```

### Крок 5: Адаптер для JsonFileLogger

```csharp
public class JsonFileLoggerAdapter : IAppLogger
{
    private readonly JsonFileLogger _logger;

    public JsonFileLoggerAdapter(JsonFileLogger logger)
    {
        _logger = logger;
    }

    public void Info(string message)
        => _logger.Write(CreateEntry("INFO", message));

    public void Warning(string message)
        => _logger.Write(CreateEntry("WARNING", message));

    public void Error(string message, Exception exception = null)
    {
        var entry = CreateEntry("ERROR", message);
        if (exception != null)
        {
            entry["exceptionType"] = exception.GetType().Name;
            entry["exceptionMessage"] = exception.Message;
        }
        _logger.Write(entry);
    }

    // Приватний хелпер — будує стандартний словник запису
    private static Dictionary<string, object> CreateEntry(string level, string message)
        => new()
        {
            ["timestamp"] = DateTime.UtcNow.ToString("o"),
            ["level"]     = level,
            ["message"]   = message
        };
}
```

### Крок 6: Клієнтський код — не знає про конкретні логери

```csharp
// Сервіс, який використовує тільки IAppLogger
public class OrderService
{
    private readonly IAppLogger _logger;

    // Отримує логер через DI — не знає, що це за реалізація
    public OrderService(IAppLogger logger)
    {
        _logger = logger;
    }

    public void PlaceOrder(string orderId, decimal amount)
    {
        _logger.Info($"Починаємо обробку замовлення {orderId}");

        try
        {
            if (amount <= 0)
                throw new ArgumentException("Сума не може бути від'ємною");

            // ... логіка замовлення ...

            _logger.Info($"Замовлення {orderId} на суму {amount:C} успішно оброблено");
        }
        catch (Exception ex)
        {
            _logger.Error($"Помилка при обробці замовлення {orderId}", ex);
        }
    }
}

class Program
{
    static void Main()
    {
        // Варіант 1: використовуємо legacy-консольний логер
        IAppLogger consoleLogger = new LegacyConsoleLoggerAdapter(new LegacyConsoleLogger());
        var service1 = new OrderService(consoleLogger);
        service1.PlaceOrder("ORD-001", 299.99m);

        Console.WriteLine();

        // Варіант 2: використовуємо JSON-логер — той самий сервіс, інший логер
        IAppLogger jsonLogger = new JsonFileLoggerAdapter(new JsonFileLogger("/var/log/app.json"));
        var service2 = new OrderService(jsonLogger);
        service2.PlaceOrder("ORD-002", -10m); // спровокуємо помилку
    }
}
```

### Очікуваний вивід

```
[14:32:01] [INFO] Починаємо обробку замовлення ORD-001
[14:32:01] [INFO] Замовлення ORD-001 на суму ₴299.99 успішно оброблено

[JSON→/var/log/app.json] { "timestamp": "...", "level": "INFO", "message": "Починаємо обробку замовлення ORD-002" }
[JSON→/var/log/app.json] { "timestamp": "...", "level": "ERROR", "message": "Помилка при обробці ORD-002", "exceptionType": "ArgumentException", "exceptionMessage": "Сума не може бути від'ємною" }
```

---

## Приклад 2 — Просунутий (Платіжні системи)

Реалістичний сценарій: платіжна система для інтернет-магазину. Потрібно підтримувати кілька платіжних провайдерів (Stripe, LiqPay, PayPal), кожен з яких має зовсім різний SDK.

### Крок 1: Доменні типи системи

```csharp
// Результат будь-якого платежу у нашій системі
public class PaymentResult
{
    public bool Success { get; init; }
    public string TransactionId { get; init; }
    public string ProviderName { get; init; }
    public decimal Amount { get; init; }
    public string Currency { get; init; }
    public string ErrorMessage { get; init; }
    public DateTime ProcessedAt { get; init; } = DateTime.UtcNow;

    public override string ToString()
    {
        if (Success)
            return $"✓ [{ProviderName}] TxID: {TransactionId} | {Amount} {Currency} | {ProcessedAt:HH:mm:ss}";
        return $"✗ [{ProviderName}] Помилка: {ErrorMessage}";
    }
}

// Запит на оплату у нашій системі
public class PaymentRequest
{
    public string OrderId { get; init; }
    public decimal Amount { get; init; }
    public string Currency { get; init; }        // "UAH", "USD", "EUR"
    public string CardNumber { get; init; }
    public string CardExpiry { get; init; }      // "MM/YY"
    public string Cvv { get; init; }
    public string CardholderName { get; init; }
}
```

### Крок 2: Цільовий інтерфейс платіжного провайдера

```csharp
// Єдиний інтерфейс, який знає наша система
public interface IPaymentProvider
{
    string ProviderName { get; }
    PaymentResult Charge(PaymentRequest request);
    PaymentResult Refund(string transactionId, decimal amount);
    bool IsAvailable();
}
```

### Крок 3: Перший Adaptee — умовний Stripe SDK

```csharp
// Уявний Stripe SDK — зовсім інший стиль API
// (в реальності Stripe.net має схожу, але власну структуру)
public class StripePaymentGateway
{
    private readonly string _secretKey;

    public StripePaymentGateway(string secretKey)
    {
        _secretKey = secretKey;
    }

    // Stripe працює в центах (або мінімальних одиницях валюти)
    public StripeChargeResponse CreateCharge(
        long amountInCents,
        string currency,
        string cardToken,
        string description)
    {
        Console.WriteLine($"[Stripe SDK] CreateCharge: {amountInCents} cents, {currency}");
        // Симулюємо успішний запит
        return new StripeChargeResponse
        {
            Id = $"ch_{Guid.NewGuid():N}",
            Status = "succeeded",
            AmountCaptured = amountInCents,
            Currency = currency.ToLower()
        };
    }

    public StripeRefundResponse CreateRefund(string chargeId, long amountInCents)
    {
        Console.WriteLine($"[Stripe SDK] CreateRefund: chargeId={chargeId}, {amountInCents} cents");
        return new StripeRefundResponse { Id = $"re_{Guid.NewGuid():N}", Status = "succeeded" };
    }

    public bool Ping() => true; // перевірка з'єднання
}

// DTO Stripe
public class StripeChargeResponse
{
    public string Id { get; set; }
    public string Status { get; set; }
    public long AmountCaptured { get; set; }
    public string Currency { get; set; }
    public string FailureMessage { get; set; }
}

public class StripeRefundResponse
{
    public string Id { get; set; }
    public string Status { get; set; }
}
```

### Крок 4: Адаптер для Stripe

```csharp
public class StripeAdapter : IPaymentProvider
{
    private readonly StripePaymentGateway _stripe;

    public string ProviderName => "Stripe";

    public StripeAdapter(StripePaymentGateway stripe)
    {
        _stripe = stripe;
    }

    public PaymentResult Charge(PaymentRequest request)
    {
        try
        {
            // Адаптуємо: наш Amount (decimal) → Stripe amountInCents (long)
            var amountInCents = (long)(request.Amount * 100);

            // Stripe очікує токен картки, а не сирі дані
            // У реальності тут була б токенізація через Stripe.js
            var cardToken = TokenizeCard(request.CardNumber, request.CardExpiry, request.Cvv);

            var response = _stripe.CreateCharge(
                amountInCents: amountInCents,
                currency: request.Currency.ToLower(),
                cardToken: cardToken,
                description: $"Order {request.OrderId}"
            );

            // Адаптуємо відповідь Stripe → наш PaymentResult
            if (response.Status == "succeeded")
            {
                return new PaymentResult
                {
                    Success = true,
                    TransactionId = response.Id,
                    ProviderName = ProviderName,
                    Amount = response.AmountCaptured / 100m, // центи → гривні
                    Currency = response.Currency.ToUpper()
                };
            }

            return new PaymentResult
            {
                Success = false,
                ProviderName = ProviderName,
                ErrorMessage = response.FailureMessage ?? "Невідома помилка"
            };
        }
        catch (Exception ex)
        {
            return new PaymentResult
            {
                Success = false,
                ProviderName = ProviderName,
                ErrorMessage = $"Stripe error: {ex.Message}"
            };
        }
    }

    public PaymentResult Refund(string transactionId, decimal amount)
    {
        var amountInCents = (long)(amount * 100);
        var response = _stripe.CreateRefund(transactionId, amountInCents);

        return new PaymentResult
        {
            Success = response.Status == "succeeded",
            TransactionId = response.Id,
            ProviderName = ProviderName,
            Amount = amount,
            ErrorMessage = response.Status != "succeeded" ? "Refund failed" : null
        };
    }

    public bool IsAvailable() => _stripe.Ping();

    // Приватний хелпер — симулює токенізацію картки
    private static string TokenizeCard(string number, string expiry, string cvv)
        => $"tok_{number[^4..]}"; // в реальності — запит до Stripe API
}
```

### Крок 5: Другий Adaptee — умовний LiqPay SDK

```csharp
// LiqPay має зовсім інший підхід — XML-орієнтований, з підписом запиту
public class LiqPayClient
{
    private readonly string _publicKey;
    private readonly string _privateKey;

    public LiqPayClient(string publicKey, string privateKey)
    {
        _publicKey = publicKey;
        _privateKey = privateKey;
    }

    // LiqPay приймає XML-документ і повертає XML
    public LiqPayResponse ProcessPayment(string xmlRequest, string signature)
    {
        Console.WriteLine($"[LiqPay SDK] ProcessPayment з підписом {signature[..8]}...");
        return new LiqPayResponse
        {
            PaymentId = new Random().Next(100000, 999999).ToString(),
            Status = "success",
            ErrCode = null
        };
    }

    public LiqPayResponse ProcessRefund(string paymentId, decimal amount, string signature)
    {
        Console.WriteLine($"[LiqPay SDK] ProcessRefund: paymentId={paymentId}");
        return new LiqPayResponse { PaymentId = paymentId, Status = "success" };
    }

    public bool CheckStatus() => true;
}

public class LiqPayResponse
{
    public string PaymentId { get; set; }
    public string Status { get; set; }  // "success" / "failure" / "error"
    public string ErrCode { get; set; }
    public string ErrDescription { get; set; }
}
```

### Крок 6: Адаптер для LiqPay

```csharp
public class LiqPayAdapter : IPaymentProvider
{
    private readonly LiqPayClient _liqPay;
    private readonly string _privateKey;

    public string ProviderName => "LiqPay";

    public LiqPayAdapter(LiqPayClient liqPay, string privateKey)
    {
        _liqPay = liqPay;
        _privateKey = privateKey;
    }

    public PaymentResult Charge(PaymentRequest request)
    {
        try
        {
            // Адаптуємо: будуємо XML-запит у форматі LiqPay
            var xml = BuildXmlRequest(request);
            var signature = SignRequest(xml);

            var response = _liqPay.ProcessPayment(xml, signature);

            return MapToResult(response, request.Amount, request.Currency);
        }
        catch (Exception ex)
        {
            return new PaymentResult
            {
                Success = false,
                ProviderName = ProviderName,
                ErrorMessage = $"LiqPay error: {ex.Message}"
            };
        }
    }

    public PaymentResult Refund(string transactionId, decimal amount)
    {
        var signature = SignRequest(transactionId);
        var response = _liqPay.ProcessRefund(transactionId, amount, signature);
        return MapToResult(response, amount, "UAH");
    }

    public bool IsAvailable() => _liqPay.CheckStatus();

    // Приватні хелпери — деталі адаптації від нашого формату до LiqPay
    private static string BuildXmlRequest(PaymentRequest req)
        => $"<payment><order_id>{req.OrderId}</order_id><amount>{req.Amount}</amount><currency>{req.Currency}</currency></payment>";

    private string SignRequest(string data)
    {
        // Спрощено — реальний підпис є SHA1 хешем
        var combined = _privateKey + data + _privateKey;
        return Convert.ToBase64String(System.Text.Encoding.UTF8.GetBytes(combined))[..16];
    }

    private PaymentResult MapToResult(LiqPayResponse response, decimal amount, string currency)
        => new()
        {
            Success = response.Status == "success",
            TransactionId = response.PaymentId,
            ProviderName = ProviderName,
            Amount = amount,
            Currency = currency,
            ErrorMessage = response.Status != "success"
                ? $"[{response.ErrCode}] {response.ErrDescription}"
                : null
        };
}
```

### Крок 7: Клієнтський код — платіжна служба

```csharp
// Служба, яка оркеструє платежі — знає тільки IPaymentProvider
public class PaymentService
{
    private readonly IReadOnlyList<IPaymentProvider> _providers;

    public PaymentService(IEnumerable<IPaymentProvider> providers)
    {
        _providers = providers.ToList();
    }

    // Спробувати всі доступні провайдери, повернути перший успішний
    public PaymentResult ProcessWithFallback(PaymentRequest request)
    {
        foreach (var provider in _providers)
        {
            if (!provider.IsAvailable())
            {
                Console.WriteLine($"[PaymentService] {provider.ProviderName} недоступний, пропускаємо");
                continue;
            }

            Console.WriteLine($"[PaymentService] Спробуємо {provider.ProviderName}...");
            var result = provider.Charge(request);

            if (result.Success)
                return result;

            Console.WriteLine($"[PaymentService] {provider.ProviderName} відмовив: {result.ErrorMessage}");
        }

        return new PaymentResult
        {
            Success = false,
            ErrorMessage = "Всі платіжні провайдери недоступні або відмовили"
        };
    }
}

class Program
{
    static void Main()
    {
        // Збираємо всі адаптери разом — клієнт не знає про конкретні SDK
        var providers = new IPaymentProvider[]
        {
            new StripeAdapter(new StripePaymentGateway("sk_test_...")),
            new LiqPayAdapter(new LiqPayClient("pub_key", "priv_key"), "priv_key")
        };

        var paymentService = new PaymentService(providers);

        var request = new PaymentRequest
        {
            OrderId     = "ORDER-42",
            Amount      = 1500.00m,
            Currency    = "UAH",
            CardNumber  = "4111111111111111",
            CardExpiry  = "12/26",
            Cvv         = "123",
            CardholderName = "JOHN DOE"
        };

        Console.WriteLine("=== Обробка платежу ===");
        var result = paymentService.ProcessWithFallback(request);
        Console.WriteLine($"\nРезультат: {result}");
    }
}
```

### Очікуваний вивід

```
=== Обробка платежу ===
[PaymentService] Спробуємо Stripe...
[Stripe SDK] CreateCharge: 150000 cents, uah

Результат: ✓ [Stripe] TxID: ch_3a7f2b1c... | 1500 UAH | 14:45:12
```

---

## Two-Way Adapter (двосторонній)

Іноді потрібно, щоб адаптер працював в обидві сторони — адаптував і A→B, і B→A.

```csharp
public interface IMetricSystem { double GetDistanceKm(); }
public interface IImperialSystem { double GetDistanceMiles(); }

// Двосторонній адаптер реалізує обидва інтерфейси
public class DistanceAdapter : IMetricSystem, IImperialSystem
{
    private double _km;

    public DistanceAdapter(double km) { _km = km; }

    // IMetricSystem → повертає кілометри
    public double GetDistanceKm() => _km;

    // IImperialSystem → конвертує і повертає милі
    public double GetDistanceMiles() => _km * 0.621371;
}

// Використання:
var adapter = new DistanceAdapter(100);
IMetricSystem metric = adapter;
IImperialSystem imperial = adapter;

Console.WriteLine($"{metric.GetDistanceKm()} км = {imperial.GetDistanceMiles():F2} миль");
// 100 км = 62.14 миль
```

---

## Adapter vs Facade vs Decorator

Ці три патерни схожі — всі "обгортають" інший об'єкт. Важливо розуміти різницю:

| Патерн | Мета | Інтерфейс | Кількість об'єктів |
|---|---|---|---|
| **Adapter** | Зробити несумісні інтерфейси сумісними | **Інший** (перетворює) | 1 Adaptee |
| **Facade** | Спростити складний підсистемний API | **Новий, простіший** (приховує) | Багато об'єктів |
| **Decorator** | Додати поведінку без зміни інтерфейсу | **Той самий** (розширює) | 1 об'єкт |

```csharp
// Adapter: ITarget ≠ Adaptee API — перетворюємо
class Adapter : ITarget { private Adaptee _a; /* ... */ }

// Facade: спрощений фасад над підсистемою
class Facade { private SubA _a; private SubB _b; private SubC _c; /* ... */ }

// Decorator: той самий інтерфейс, нова поведінка поверх
class LoggingDecorator : ITarget { private ITarget _inner; /* ... */ }
```

---

## Переваги та недоліки

### Переваги

- **Принцип відкритості/закритості** — додаємо нових провайдерів без зміни існуючого коду
- **Повторне використання** — можна використовувати несумісні класи без їхньої зміни
- **Ізоляція** — деталі стороннього API приховані в адаптері
- **Тестованість** — легко замінити адаптер на mock у тестах

### Недоліки

- **Зайвий клас** — для кожного Adaptee потрібен окремий Adapter
- **Накладні витрати** — зайвий рівень делегування (зазвичай незначний)
- **Ускладнення коду** — якщо Adaptee і ITarget дуже різні, адаптер може стати складним

---

## Підсумок

| Аспект | Деталь |
|---|---|
| Тип патерну | Структурний (Structural) |
| Вирішує проблему | Несумісні інтерфейси між класами |
| Ключова ідея | Обгортка, яка "перекладає" виклики |
| Реалізація в C# | Композиція (Object Adapter) — рекомендовано |
| Альтернативи | Facade (спрощення), Decorator (розширення) |

### Коротке правило вибору

```
Є клас з потрібною функціональністю, але не тим інтерфейсом?
  ✅ Так → Adapter

Хочете спростити складний API з багатьох класів?
  ✅ Так → Facade

Хочете додати поведінку, зберігши інтерфейс?
  ✅ Так → Decorator
```

---

*Документ підготовлено як навчальний матеріал з патернів проєктування на C#.*
