# Патерн Decorator (Декоратор) у C#

> **Категорія:** Структурний патерн (Structural Pattern)  
> **Мова прикладів:** C# (.NET)

---

## Зміст

1. [Що таке Decorator?](#що-таке-decorator)
2. [Проблема успадкування](#проблема-успадкування)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Простий (Кава з додатками)](#приклад-1--простий-кава-з-додатками)
5. [Приклад 2 — Просунутий (HTTP-middleware конвеєр)](#приклад-2--просунутий-http-middleware-конвеєр)
6. [Декоратори в реальному .NET](#декоратори-в-реальному-net)
7. [Decorator vs Inheritance vs Adapter](#decorator-vs-inheritance-vs-adapter)
8. [Переваги та недоліки](#переваги-та-недоліки)
9. [Підсумок](#підсумок)

---

## Що таке Decorator?

**Decorator** — це патерн, який дозволяє **динамічно додавати нову поведінку об'єкту**, обгортаючи його в інший об'єкт з **тим самим інтерфейсом**.

Ключова особливість: і оригінальний об'єкт, і кожен декоратор реалізують **один і той самий інтерфейс**. Клієнт не знає — він спілкується з оригіналом чи з декоратором.

Декоратори можна **накладати один на одного** як шари цибулі або матрьошки:

```
┌─────────────────────────────────┐
│  LoggingDecorator               │
│  ┌───────────────────────────┐  │
│  │  CachingDecorator         │  │
│  │  ┌─────────────────────┐  │  │
│  │  │  RetryDecorator      │  │  │
│  │  │  ┌───────────────┐   │  │  │
│  │  │  │  RealService  │   │  │  │
│  │  │  └───────────────┘   │  │  │
│  │  └─────────────────────┘  │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

Виклик іде від зовнішнього шару до внутрішнього, і кожен шар може додати поведінку **до** або **після** реального виклику.

---

## Проблема успадкування

Чому не просто створити підклас для кожної комбінації поведінки?

Уявіть текстовий потік з різними можливостями: шифрування, стиснення, буферизація.

```
Stream
├── EncryptedStream
├── CompressedStream
├── BufferedStream
├── EncryptedCompressedStream        ← вже дублювання
├── EncryptedBufferedStream          ← ще більше
├── CompressedBufferedStream         ← ...
└── EncryptedCompressedBufferedStream ← комбінаторний вибух!
```

При **3 можливостях** вже потрібно **7 підкласів**. При 4 — 15. При N — 2ⁿ − 1.

Це **комбінаторний вибух** успадкування. Decorator вирішує це елегантно: кожна можливість — окремий декоратор, їх можна комбінувати довільно в рантаймі.

```csharp
// Замість 7 класів — 3 декоратори, необмежені комбінації
IStream stream = new EncryptedDecorator(
                     new CompressedDecorator(
                         new BufferedDecorator(
                             new FileStream("data.bin"))));
```

---

## Структура патерну

```
┌─────────────────────────┐
│   «interface»           │
│   IComponent            │
│   + Operation()         │
└────────────┬────────────┘
             │
    ┌────────┴──────────┐
    │                   │
┌───▼──────────┐  ┌─────▼───────────────┐
│  Concrete    │  │  BaseDecorator      │
│  Component   │  │  - _inner: IComponent│
│              │  │  + Operation()      │
│ + Operation()│  │    → _inner.Op()    │
└──────────────┘  └──────────┬──────────┘
                             │ extends
                   ┌─────────┴──────────┐
                   │                    │
           ┌───────▼──────┐   ┌─────────▼────┐
           │ DecoratorA   │   │ DecoratorB   │
           │ + Operation()│   │ + Operation()│
           │   → extra    │   │   → extra    │
           │   → _inner   │   │   → _inner   │
           └──────────────┘   └──────────────┘
```

Учасники:
- **IComponent** — спільний інтерфейс для компонента і всіх декораторів
- **ConcreteComponent** — реальний об'єкт, базова реалізація
- **BaseDecorator** — абстрактний декоратор, зберігає посилання на `IComponent`
- **ConcreteDecorator** — конкретний декоратор, додає специфічну поведінку
- **Client** — працює через `IComponent`, не знає про декоратори

---

## Приклад 1 — Простий (Кава з додатками)

Класичний приклад для розуміння механіки: замовлення кави з різними додатками. Ціна і опис залежать від комбінації.

### Крок 1: Інтерфейс напою

```csharp
// Спільний інтерфейс для кави і всіх її варіацій
public interface IBeverage
{
    string GetDescription();
    decimal GetCost();
}
```

### Крок 2: Базові напої (ConcreteComponent)

```csharp
// Простий еспресо — базова реалізація
public class Espresso : IBeverage
{
    public string GetDescription() => "Еспресо";
    public decimal GetCost() => 35.00m;
}

// Американо
public class Americano : IBeverage
{
    public string GetDescription() => "Американо";
    public decimal GetCost() => 40.00m;
}

// Латте
public class Latte : IBeverage
{
    public string GetDescription() => "Латте";
    public decimal GetCost() => 55.00m;
}
```

### Крок 3: Базовий декоратор

```csharp
// Абстрактний декоратор — реалізує IBeverage і зберігає вкладений напій
// За замовчуванням просто делегує виклики — підкласи додають своє
public abstract class BeverageDecorator : IBeverage
{
    // Захищене поле — вкладений напій (може бути базовий або інший декоратор)
    protected readonly IBeverage _beverage;

    protected BeverageDecorator(IBeverage beverage)
    {
        _beverage = beverage;
    }

    // За замовчуванням — делегуємо до вкладеного об'єкта
    // Підкласи перевизначають і розширюють цю поведінку
    public virtual string GetDescription() => _beverage.GetDescription();
    public virtual decimal GetCost() => _beverage.GetCost();
}
```

### Крок 4: Конкретні декоратори (додатки до кави)

```csharp
// Молоко
public class MilkDecorator : BeverageDecorator
{
    public MilkDecorator(IBeverage beverage) : base(beverage) { }

    // Розширюємо опис і ціну — делегуємо до вкладеного, потім додаємо своє
    public override string GetDescription()
        => _beverage.GetDescription() + ", молоко";

    public override decimal GetCost()
        => _beverage.GetCost() + 8.00m;
}

// Карамель
public class CaramelDecorator : BeverageDecorator
{
    public CaramelDecorator(IBeverage beverage) : base(beverage) { }

    public override string GetDescription()
        => _beverage.GetDescription() + ", карамель";

    public override decimal GetCost()
        => _beverage.GetCost() + 12.00m;
}

// Ванільний сироп
public class VanillaDecorator : BeverageDecorator
{
    public VanillaDecorator(IBeverage beverage) : base(beverage) { }

    public override string GetDescription()
        => _beverage.GetDescription() + ", ваніль";

    public override decimal GetCost()
        => _beverage.GetCost() + 10.00m;
}

// Додаткова порція еспресо
public class ExtraShotDecorator : BeverageDecorator
{
    public ExtraShotDecorator(IBeverage beverage) : base(beverage) { }

    public override string GetDescription()
        => _beverage.GetDescription() + ", подвійний шот";

    public override decimal GetCost()
        => _beverage.GetCost() + 20.00m;
}
```

### Крок 5: Використання

```csharp
class Program
{
    static void Main()
    {
        // --- Простий еспресо ---
        IBeverage beverage = new Espresso();
        PrintOrder(beverage);
        // Еспресо | 35.00 грн

        // --- Латте з карамеллю ---
        IBeverage caramelLatte = new CaramelDecorator(new Latte());
        PrintOrder(caramelLatte);
        // Латте, карамель | 67.00 грн

        // --- Складний напій: американо з молоком, ваніллю і подвійним шотом ---
        IBeverage customDrink = new ExtraShotDecorator(
                                    new VanillaDecorator(
                                        new MilkDecorator(
                                            new Americano())));
        PrintOrder(customDrink);
        // Американо, молоко, ваніль, подвійний шот | 78.00 грн

        // --- Два молока (декоратор можна застосувати двічі!) ---
        IBeverage doubleMilkEspresso = new MilkDecorator(
                                           new MilkDecorator(
                                               new Espresso()));
        PrintOrder(doubleMilkEspresso);
        // Еспресо, молоко, молоко | 51.00 грн
    }

    static void PrintOrder(IBeverage b)
        => Console.WriteLine($"{b.GetDescription()} | {b.GetCost():F2} грн");
}
```

### Що відбувається при виклику `GetCost()`?

Розгорнемо `customDrink.GetCost()` по кроках:

```
ExtraShotDecorator.GetCost()
  └── VanillaDecorator.GetCost() + 20
        └── MilkDecorator.GetCost() + 10
              └── Americano.GetCost() + 8
                    └── 40
              = 48
        = 58
  = 68
= 78  ← фінальна ціна
```

Кожен шар додає своє і делегує далі — чисто і прозоро.

---

## Приклад 2 — Просунутий (HTTP-middleware конвеєр)

Реалістичний сценарій: конвеєр обробки HTTP-запитів. Кожен декоратор — окремий "middleware": логування, авторизація, кешування, повторні спроби.

### Крок 1: Доменні типи

```csharp
// HTTP-запит у нашій системі
public class HttpRequest
{
    public string Method { get; init; }    // GET, POST ...
    public string Url { get; init; }
    public Dictionary<string, string> Headers { get; init; } = new();
    public string Body { get; init; }

    public override string ToString() => $"{Method} {Url}";
}

// HTTP-відповідь
public class HttpResponse
{
    public int StatusCode { get; init; }
    public string Body { get; init; }
    public Dictionary<string, string> Headers { get; init; } = new();
    public bool IsSuccess => StatusCode >= 200 && StatusCode < 300;

    public override string ToString() => $"HTTP {StatusCode}: {Body?[..Math.Min(50, Body?.Length ?? 0)]}...";
}
```

### Крок 2: Цільовий інтерфейс

```csharp
// Єдиний інтерфейс — і реальний клієнт, і всі декоратори його реалізують
public interface IHttpClient
{
    Task<HttpResponse> SendAsync(HttpRequest request);
}
```

### Крок 3: Реальний HTTP-клієнт (ConcreteComponent)

```csharp
// Справжня реалізація — відправляє HTTP-запит
public class RealHttpClient : IHttpClient
{
    public async Task<HttpResponse> SendAsync(HttpRequest request)
    {
        // Симулюємо реальний мережевий запит
        Console.WriteLine($"  [HTTP] Відправляємо {request}");
        await Task.Delay(100); // затримка мережі

        // Симулюємо відповідь сервера
        return new HttpResponse
        {
            StatusCode = 200,
            Body = $"{{\"result\": \"ok\", \"url\": \"{request.Url}\"}}",
            Headers = new() { ["Content-Type"] = "application/json" }
        };
    }
}
```

### Крок 4: Базовий декоратор

```csharp
// Базовий декоратор — зберігає вкладений клієнт і делегує виклики
public abstract class HttpClientDecorator : IHttpClient
{
    protected readonly IHttpClient _inner;

    protected HttpClientDecorator(IHttpClient inner)
    {
        _inner = inner;
    }

    // За замовчуванням — просто делегуємо
    public virtual Task<HttpResponse> SendAsync(HttpRequest request)
        => _inner.SendAsync(request);
}
```

### Крок 5: Декоратор логування

```csharp
// Логує кожен запит і відповідь з часом виконання
public class LoggingDecorator : HttpClientDecorator
{
    private readonly string _name;

    public LoggingDecorator(IHttpClient inner, string name = "HTTP") : base(inner)
    {
        _name = name;
    }

    public override async Task<HttpResponse> SendAsync(HttpRequest request)
    {
        var stopwatch = System.Diagnostics.Stopwatch.StartNew();

        // Логуємо ДО запиту
        Console.WriteLine($"[{_name}:LOG] → {request.Method} {request.Url}");
        if (request.Headers.Any())
            foreach (var h in request.Headers)
                Console.WriteLine($"[{_name}:LOG]   Header: {h.Key}: {h.Value}");

        try
        {
            // Делегуємо до вкладеного клієнта
            var response = await _inner.SendAsync(request);

            stopwatch.Stop();
            // Логуємо ПІСЛЯ запиту
            Console.WriteLine($"[{_name}:LOG] ← {response.StatusCode} за {stopwatch.ElapsedMilliseconds}ms");

            return response;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();
            Console.WriteLine($"[{_name}:LOG] ✗ Помилка за {stopwatch.ElapsedMilliseconds}ms: {ex.Message}");
            throw;
        }
    }
}
```

### Крок 6: Декоратор авторизації

```csharp
// Автоматично додає Authorization-заголовок до кожного запиту
public class AuthorizationDecorator : HttpClientDecorator
{
    private readonly Func<string> _tokenProvider; // фабрика токенів — може оновлювати токен

    public AuthorizationDecorator(IHttpClient inner, Func<string> tokenProvider)
        : base(inner)
    {
        _tokenProvider = tokenProvider;
    }

    public override Task<HttpResponse> SendAsync(HttpRequest request)
    {
        // Отримуємо актуальний токен (може бути з кешу або оновлений)
        var token = _tokenProvider();

        // Створюємо новий запит з доданим заголовком авторизації
        // (не мутуємо оригінальний запит — immutable підхід)
        var authorizedRequest = new HttpRequest
        {
            Method = request.Method,
            Url    = request.Url,
            Body   = request.Body,
            Headers = new Dictionary<string, string>(request.Headers)
            {
                ["Authorization"] = $"Bearer {token}"
            }
        };

        Console.WriteLine($"[AUTH] Додаємо токен: {token[..10]}...");

        return _inner.SendAsync(authorizedRequest);
    }
}
```

### Крок 7: Декоратор кешування

```csharp
// Кешує відповіді GET-запитів на певний час
public class CachingDecorator : HttpClientDecorator
{
    private readonly TimeSpan _ttl;

    // Кеш: ключ (URL) → (відповідь, час отримання)
    private readonly Dictionary<string, (HttpResponse Response, DateTime CachedAt)> _cache = new();

    public CachingDecorator(IHttpClient inner, TimeSpan ttl) : base(inner)
    {
        _ttl = ttl;
    }

    public override async Task<HttpResponse> SendAsync(HttpRequest request)
    {
        // Кешуємо тільки GET-запити (POST/PUT/DELETE змінюють стан)
        if (request.Method != "GET")
            return await _inner.SendAsync(request);

        var cacheKey = request.Url;

        // Перевіряємо кеш
        if (_cache.TryGetValue(cacheKey, out var cached))
        {
            var age = DateTime.UtcNow - cached.CachedAt;
            if (age < _ttl)
            {
                Console.WriteLine($"[CACHE] Повертаємо з кешу (вік: {age.TotalSeconds:F1}s): {cacheKey}");
                return cached.Response;
            }
            // Кеш застарів — видаляємо
            _cache.Remove(cacheKey);
            Console.WriteLine($"[CACHE] Кеш застарів для: {cacheKey}");
        }

        // Кешу немає — виконуємо реальний запит
        var response = await _inner.SendAsync(request);

        // Зберігаємо в кеш тільки успішні відповіді
        if (response.IsSuccess)
        {
            _cache[cacheKey] = (response, DateTime.UtcNow);
            Console.WriteLine($"[CACHE] Збережено в кеш: {cacheKey}");
        }

        return response;
    }
}
```

### Крок 8: Декоратор повторних спроб

```csharp
// Автоматично повторює запит при помилці (з exponential backoff)
public class RetryDecorator : HttpClientDecorator
{
    private readonly int _maxRetries;
    private readonly TimeSpan _initialDelay;

    public RetryDecorator(IHttpClient inner, int maxRetries = 3, TimeSpan? initialDelay = null)
        : base(inner)
    {
        _maxRetries = maxRetries;
        _initialDelay = initialDelay ?? TimeSpan.FromMilliseconds(200);
    }

    public override async Task<HttpResponse> SendAsync(HttpRequest request)
    {
        Exception lastException = null;

        for (int attempt = 1; attempt <= _maxRetries + 1; attempt++)
        {
            try
            {
                var response = await _inner.SendAsync(request);

                // Якщо сервер повернув 5xx — теж повторюємо
                if (response.StatusCode >= 500 && attempt <= _maxRetries)
                {
                    Console.WriteLine($"[RETRY] Сервер повернув {response.StatusCode}, спроба {attempt}/{_maxRetries + 1}");
                    await DelayWithBackoff(attempt);
                    continue;
                }

                if (attempt > 1)
                    Console.WriteLine($"[RETRY] Успіх на спробі {attempt}");

                return response;
            }
            catch (Exception ex) when (attempt <= _maxRetries)
            {
                lastException = ex;
                Console.WriteLine($"[RETRY] Помилка на спробі {attempt}: {ex.Message}. Повторюємо...");
                await DelayWithBackoff(attempt);
            }
        }

        throw new Exception($"Всі {_maxRetries + 1} спроби вичерпано", lastException);
    }

    // Exponential backoff: 200ms, 400ms, 800ms...
    private Task DelayWithBackoff(int attempt)
    {
        var delay = TimeSpan.FromMilliseconds(_initialDelay.TotalMilliseconds * Math.Pow(2, attempt - 1));
        Console.WriteLine($"[RETRY] Чекаємо {delay.TotalMilliseconds}ms перед наступною спробою...");
        return Task.Delay(delay);
    }
}
```

### Крок 9: Складання конвеєру і використання

```csharp
class Program
{
    static async Task Main()
    {
        // --- Збираємо конвеєр декораторів ---
        // Порядок важливий: зовнішній → внутрішній
        //
        // LoggingDecorator     ← найзовнішній, бачить все
        //   AuthDecorator      ← додає токен перед передачею далі
        //     CachingDecorator ← кешує вже авторизовані запити
        //       RetryDecorator ← повторює при помилці
        //         RealClient   ← реальний HTTP

        IHttpClient client =
            new LoggingDecorator(
                new AuthorizationDecorator(
                    new CachingDecorator(
                        new RetryDecorator(
                            new RealHttpClient(),
                            maxRetries: 2),
                        ttl: TimeSpan.FromMinutes(5)),
                    tokenProvider: () => "eyJhbGciOiJIUzI1NiJ9.abc123"),
                name: "API");

        var request = new HttpRequest
        {
            Method = "GET",
            Url    = "https://api.example.com/users/42"
        };

        Console.WriteLine("=== Перший запит (іде до сервера) ===");
        var response1 = await client.SendAsync(request);
        Console.WriteLine($"Відповідь: {response1}\n");

        Console.WriteLine("=== Другий запит (той самий URL — з кешу) ===");
        var response2 = await client.SendAsync(request);
        Console.WriteLine($"Відповідь: {response2}\n");

        Console.WriteLine("=== POST-запит (не кешується) ===");
        var postRequest = new HttpRequest
        {
            Method = "POST",
            Url    = "https://api.example.com/users",
            Body   = "{\"name\": \"Іван\"}"
        };
        var response3 = await client.SendAsync(postRequest);
        Console.WriteLine($"Відповідь: {response3}");
    }
}
```

### Очікуваний вивід

```
=== Перший запит (іде до сервера) ===
[API:LOG] → GET https://api.example.com/users/42
[AUTH] Додаємо токен: eyJhbGciO...
[CACHE] Кешу немає для: https://api.example.com/users/42
  [HTTP] Відправляємо GET https://api.example.com/users/42
[CACHE] Збережено в кеш: https://api.example.com/users/42
[API:LOG] ← 200 за 103ms
Відповідь: HTTP 200: {"result": "ok", "url": "https://api.exam...

=== Другий запит (той самий URL — з кешу) ===
[API:LOG] → GET https://api.example.com/users/42
[AUTH] Додаємо токен: eyJhbGciO...
[CACHE] Повертаємо з кешу (вік: 0.1s): https://api.example.com/users/42
[API:LOG] ← 200 за 1ms
Відповідь: HTTP 200: {"result": "ok", "url": "https://api.exam...

=== POST-запит (не кешується) ===
[API:LOG] → POST https://api.example.com/users
[AUTH] Додаємо токен: eyJhbGciO...
  [HTTP] Відправляємо POST https://api.example.com/users
[API:LOG] ← 200 за 101ms
Відповідь: HTTP 200: {"result": "ok", ...
```

### Гнучкість: легко змінити конвеєр

```csharp
// Без кешування і без retry — для dev-середовища
IHttpClient devClient =
    new LoggingDecorator(
        new AuthorizationDecorator(
            new RealHttpClient(),
            tokenProvider: () => GetDevToken()),
        name: "DEV");

// Тільки retry без авторизації — для публічного API
IHttpClient publicClient =
    new RetryDecorator(
        new RealHttpClient(),
        maxRetries: 3);
```

---

## Декоратори в реальному .NET

Decorator — один з найпоширеніших патернів у самому .NET. Ось де він зустрічається:

### System.IO.Stream

```csharp
// Stream — класичний приклад декораторів у .NET
// Кожен клас обгортає інший Stream і додає поведінку

Stream stream = new FileStream("data.bin", FileMode.Open);   // базовий
stream        = new BufferedStream(stream, bufferSize: 4096); // буферизація
stream        = new GZipStream(stream, CompressionMode.Decompress); // розпакування
// Читаємо — прозоро проходить через всі шари
```

### ASP.NET Core Middleware

```csharp
// Middleware в ASP.NET Core — це і є Decorator для обробника запитів
app.UseExceptionHandler();   // декоратор: перехоплює помилки
app.UseHttpsRedirection();   // декоратор: редирект на HTTPS
app.UseAuthentication();     // декоратор: автентифікація
app.UseAuthorization();      // декоратор: авторизація
app.UseEndpoints(...);       // реальний обробник
```

### ILogger з фільтрами

```csharp
// LoggerFactory дозволяє декорувати логери провайдерами і фільтрами
services.AddLogging(builder =>
{
    builder.AddConsole();         // декоратор: виводить в консоль
    builder.AddFilter("MyApp", LogLevel.Debug); // декоратор: фільтрує
});
```

---

## Decorator vs Inheritance vs Adapter

| Ознака | Decorator | Inheritance | Adapter |
|---|---|---|---|
| **Мета** | Додати поведінку динамічно | Розширити клас статично | Змінити інтерфейс |
| **Інтерфейс** | **Той самий** що й оригінал | Успадкований / розширений | **Інший** — перетворює |
| **Комбінування** | Довільне в рантаймі | Фіксоване при компіляції | Не застосовно |
| **Зв'язування** | Слабке (через інтерфейс) | Тісне (з батьківським) | Слабке |
| **Коли** | Гнучкі комбінації поведінки | Чітка ієрархія типів | Сумісність API |

```csharp
// Decorator: той самий інтерфейс, нова поведінка — прозоро для клієнта
class LoggingDecorator : IService { private IService _inner; ... }

// Adapter: РІЗНІ інтерфейси — перетворює один в інший
class ThirdPartyAdapter : IService { private ThirdPartyLib _lib; ... }

// Inheritance: розширення типу — статично, при компіляції
class ExtendedService : BaseService { override void Method() { ... } }
```

---

## Переваги та недоліки

### Переваги

- **Принцип єдиної відповідальності** — кожен декоратор робить одну річ
- **Принцип відкритості/закритості** — додаємо поведінку без зміни існуючих класів
- **Гнучкість у рантаймі** — конфігурація конвеєру вирішується динамічно
- **Уникнення вибуху класів** — 4 декоратори замість 15 підкласів
- **Повторне використання** — декоратори незалежні й легко переставляються

### Недоліки

- **Порядок має значення** — неправильний порядок декораторів може призвести до помилок
- **Складне налагодження** — стек викликів через кілька декораторів важче читати
- **Багато дрібних об'єктів** — кожен шар — окремий об'єкт у пам'яті
- **Складність конфігурації** — глибока вкладеність конструкторів погано читається (вирішується через Builder або DI-контейнер)

---

## Підсумок

| Аспект | Деталь |
|---|---|
| Тип патерну | Структурний (Structural) |
| Вирішує проблему | Комбінаторний вибух при успадкуванні, статичне розширення |
| Ключова ідея | Обгортка з **тим самим** інтерфейсом, яка додає поведінку |
| Ключова властивість | Декоратори можна **накладати** один на одного |
| У реальному .NET | `Stream`, ASP.NET Middleware, `ILogger` |
| Альтернативи | Inheritance (статично), Strategy (змінює алгоритм), Adapter (змінює інтерфейс) |

### Коротке правило вибору

```
Потрібно додати поведінку до об'єкта, зберігши інтерфейс?
  ✅ Так → Decorator

Поведінок кілька і їх треба комбінувати довільно?
  ✅ Точно Decorator (уникаємо вибуху підкласів)

Потрібно змінити інтерфейс?
  ✅ Ні — тоді Adapter

Один алгоритм треба замінити іншим?
  ✅ Ні — тоді Strategy
```

---

*Документ підготовлено як навчальний матеріал з патернів проєктування на C#.*
