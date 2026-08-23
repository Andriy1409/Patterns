# Патерн Builder (Будівельник) у C#

> **Категорія:** Породжуючий патерн (Creational Pattern)  
> **Мова прикладів:** C# (.NET)

---

## Зміст

1. [Що таке Builder?](#що-таке-builder)
2. [Коли використовувати?](#коли-використовувати)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Простий (Бутерброд)](#приклад-1--простий-бутерброд)
5. [Приклад 2 — Просунутий (HTTP-запит)](#приклад-2--просунутий-http-запит)
6. [Fluent Builder vs Classic Builder](#fluent-builder-vs-classic-builder)
7. [Переваги та недоліки](#переваги-та-недоліки)
8. [Підсумок](#підсумок)

---

## Що таке Builder?

**Builder** — це патерн проєктування, який дозволяє конструювати складні об'єкти **крок за кроком**.

Головна ідея: **відокремити процес побудови об'єкта від його представлення**, щоб один і той самий процес побудови міг створювати різні представлення.

### Проблема без Builder

Уявіть клас `Pizza` з десятками параметрів. Звичайний конструктор стає жахливим:

```csharp
// Виклик виглядає незрозуміло — що означає кожен параметр?
var pizza = new Pizza("велика", "тонке", true, false, true, true, false, "томатний", 3);
```

Це так звана **«проблема телескопічного конструктора»** — чим більше параметрів, тим важче читати та підтримувати код.

### Рішення з Builder

```csharp
var pizza = new PizzaBuilder()
    .WithSize("велика")
    .WithCrust("тонке")
    .WithCheese()
    .WithSauce("томатний")
    .Build();
```

Читається як звичайне речення. Зрозуміло, що будується і як.

---

## Коли використовувати?

Використовуйте Builder, коли:

- Об'єкт має **багато параметрів** (особливо необов'язкових)
- Потрібно **покроково** конструювати складний об'єкт
- Один і той самий код будівництва має давати **різні результати**
- Ви хочете уникнути **конструктора з десятками аргументів**
- Об'єкт після створення має бути **незмінним** (immutable)

---

## Структура патерну

```
┌─────────────┐         ┌──────────────────┐
│   Director  │────────▶│  IBuilder        │
│             │         │  + Reset()       │
│ + Construct │         │  + BuildPartA()  │
└─────────────┘         │  + BuildPartB()  │
                        └────────┬─────────┘
                                 │ implements
                    ┌────────────┴────────────┐
                    │                         │
             ┌──────▼──────┐         ┌────────▼──────┐
             │ConcreteBuil.│         │ConcreteBuil.  │
             │     A       │         │     B         │
             └──────┬──────┘         └───────────────┘
                    │ creates
             ┌──────▼──────┐
             │   Product   │
             └─────────────┘
```

Учасники:
- **Builder (IBuilder)** — інтерфейс з методами побудови частин
- **ConcreteBuilder** — конкретна реалізація будівельника
- **Director** — керує порядком побудови (необов'язковий)
- **Product** — складний об'єкт, що будується

---

## Приклад 1 — Простий (Бутерброд)

Цей приклад максимально простий — щоб зрозуміти суть без зайвого шуму.

### Крок 1: Продукт — клас `Sandwich`

```csharp
// Це наш кінцевий об'єкт, який ми будуємо
public class Sandwich
{
    public string Bread { get; set; }       // Тип хліба
    public string Filling { get; set; }     // Начинка
    public bool HasSauce { get; set; }      // Чи є соус
    public bool HasVegetables { get; set; } // Чи є овочі

    // Перевизначаємо ToString(), щоб зручно виводити
    public override string ToString()
    {
        var sauce = HasSauce ? "з соусом" : "без соусу";
        var vegs  = HasVegetables ? "з овочами" : "без овочів";
        return $"Бутерброд: {Bread}, начинка: {Filling}, {sauce}, {vegs}";
    }
}
```

### Крок 2: Будівельник — клас `SandwichBuilder`

```csharp
// Будівельник знає, як крок за кроком зібрати Sandwich
public class SandwichBuilder
{
    // Внутрішній екземпляр продукту, який ми поступово заповнюємо
    private Sandwich _sandwich = new Sandwich();

    // Кожен метод повертає 'this' — сам будівельник.
    // Це дозволяє ланцюжково викликати методи: .WithBread().WithFilling()...
    public SandwichBuilder WithBread(string bread)
    {
        _sandwich.Bread = bread;
        return this; // повертаємо себе для ланцюжка викликів
    }

    public SandwichBuilder WithFilling(string filling)
    {
        _sandwich.Filling = filling;
        return this;
    }

    public SandwichBuilder WithSauce()
    {
        _sandwich.HasSauce = true;
        return this;
    }

    public SandwichBuilder WithVegetables()
    {
        _sandwich.HasVegetables = true;
        return this;
    }

    // Фінальний метод — повертає готовий об'єкт
    // і скидає внутрішній стан для можливого повторного використання
    public Sandwich Build()
    {
        var result = _sandwich;
        _sandwich = new Sandwich(); // скидаємо для наступного використання
        return result;
    }
}
```

### Крок 3: Використання

```csharp
class Program
{
    static void Main()
    {
        var builder = new SandwichBuilder();

        // Будуємо перший бутерброд
        Sandwich clubSandwich = builder
            .WithBread("багет")
            .WithFilling("куряче філе")
            .WithSauce()
            .WithVegetables()
            .Build();

        Console.WriteLine(clubSandwich);
        // Виведе: Бутерброд: багет, начинка: куряче філе, з соусом, з овочами

        // Будуємо другий бутерброд — простий
        Sandwich simpleSandwich = builder
            .WithBread("білий хліб")
            .WithFilling("сир")
            .Build();

        Console.WriteLine(simpleSandwich);
        // Виведе: Бутерброд: білий хліб, начинка: сир, без соусу, без овочів
    }
}
```

### Що тут відбувається?

1. Ми створюємо `SandwichBuilder` — він містить порожній `Sandwich`
2. Кожен виклик методу (`.WithBread(...)`, `.WithFilling(...)`) заповнює одне поле
3. Методи повертають `this` → можна писати ланцюжок
4. `.Build()` повертає готовий об'єкт і скидає стан будівельника

---

## Приклад 2 — Просунутий (HTTP-запит)

Тепер реальніший приклад: будуємо об'єкт `HttpRequest` для відправки HTTP-запитів. Тут є:
- Валідація
- Необов'язкові параметри
- Різні типи тіла запиту
- Director для типових конфігурацій

### Крок 1: Допоміжні типи

```csharp
// Метод HTTP-запиту
public enum HttpMethod
{
    GET,
    POST,
    PUT,
    DELETE,
    PATCH
}

// Тіло запиту — може бути рядком або бінарними даними
public class RequestBody
{
    public string ContentType { get; }
    public string TextContent { get; }
    public byte[] BinaryContent { get; }
    public bool IsEmpty => TextContent == null && BinaryContent == null;

    public RequestBody(string contentType, string text)
    {
        ContentType = contentType;
        TextContent = text;
    }

    public RequestBody(string contentType, byte[] bytes)
    {
        ContentType = contentType;
        BinaryContent = bytes;
    }
}
```

### Крок 2: Продукт — клас `HttpRequest`

```csharp
// Незмінний (immutable) об'єкт запиту
// Всі властивості доступні тільки для читання після створення
public class HttpRequest
{
    public string Url { get; }
    public HttpMethod Method { get; }
    public IReadOnlyDictionary<string, string> Headers { get; }
    public IReadOnlyDictionary<string, string> QueryParams { get; }
    public RequestBody Body { get; }
    public TimeSpan Timeout { get; }
    public bool FollowRedirects { get; }
    public int MaxRetries { get; }

    // Конструктор internal — створювати можна тільки через Builder
    internal HttpRequest(
        string url,
        HttpMethod method,
        Dictionary<string, string> headers,
        Dictionary<string, string> queryParams,
        RequestBody body,
        TimeSpan timeout,
        bool followRedirects,
        int maxRetries)
    {
        Url = url;
        Method = method;
        Headers = headers.AsReadOnly();
        QueryParams = queryParams.AsReadOnly();
        Body = body;
        Timeout = timeout;
        FollowRedirects = followRedirects;
        MaxRetries = maxRetries;
    }

    public override string ToString()
    {
        var sb = new System.Text.StringBuilder();
        sb.AppendLine($"{Method} {Url}");

        foreach (var header in Headers)
            sb.AppendLine($"  Header: {header.Key}: {header.Value}");

        if (QueryParams.Any())
        {
            sb.AppendLine("  Query params:");
            foreach (var param in QueryParams)
                sb.AppendLine($"    {param.Key}={param.Value}");
        }

        if (Body != null && !Body.IsEmpty)
            sb.AppendLine($"  Body [{Body.ContentType}]: {Body.TextContent ?? "<binary>"}");

        sb.AppendLine($"  Timeout: {Timeout.TotalSeconds}s | Retries: {MaxRetries} | Redirects: {FollowRedirects}");
        return sb.ToString();
    }
}
```

### Крок 3: Інтерфейс будівельника

```csharp
// Інтерфейс визначає контракт — що вміє будь-який будівельник HTTP-запитів
public interface IHttpRequestBuilder
{
    IHttpRequestBuilder WithUrl(string url);
    IHttpRequestBuilder WithMethod(HttpMethod method);
    IHttpRequestBuilder WithHeader(string key, string value);
    IHttpRequestBuilder WithQueryParam(string key, string value);
    IHttpRequestBuilder WithJsonBody(string json);
    IHttpRequestBuilder WithFormBody(Dictionary<string, string> fields);
    IHttpRequestBuilder WithTimeout(TimeSpan timeout);
    IHttpRequestBuilder WithRetries(int count);
    IHttpRequestBuilder WithFollowRedirects(bool follow);
    HttpRequest Build(); // Повертає готовий продукт
}
```

### Крок 4: Конкретний будівельник

```csharp
public class HttpRequestBuilder : IHttpRequestBuilder
{
    // Внутрішні поля — накопичуємо налаштування
    private string _url;
    private HttpMethod _method = HttpMethod.GET; // значення за замовчуванням
    private readonly Dictionary<string, string> _headers = new();
    private readonly Dictionary<string, string> _queryParams = new();
    private RequestBody _body;
    private TimeSpan _timeout = TimeSpan.FromSeconds(30); // за замовчуванням 30 сек
    private bool _followRedirects = true;
    private int _maxRetries = 0;

    public IHttpRequestBuilder WithUrl(string url)
    {
        // Базова валідація — не приймаємо порожній URL
        if (string.IsNullOrWhiteSpace(url))
            throw new ArgumentException("URL не може бути порожнім", nameof(url));

        _url = url;
        return this;
    }

    public IHttpRequestBuilder WithMethod(HttpMethod method)
    {
        _method = method;
        return this;
    }

    public IHttpRequestBuilder WithHeader(string key, string value)
    {
        if (string.IsNullOrWhiteSpace(key))
            throw new ArgumentException("Назва заголовку не може бути порожньою");

        _headers[key] = value; // якщо ключ вже є — перезаписуємо
        return this;
    }

    public IHttpRequestBuilder WithQueryParam(string key, string value)
    {
        _queryParams[key] = value;
        return this;
    }

    public IHttpRequestBuilder WithJsonBody(string json)
    {
        // Автоматично встановлюємо правильний Content-Type
        _body = new RequestBody("application/json", json);

        // Для JSON-тіла метод зазвичай POST або PUT
        if (_method == HttpMethod.GET)
            _method = HttpMethod.POST;

        return this;
    }

    public IHttpRequestBuilder WithFormBody(Dictionary<string, string> fields)
    {
        // Перетворюємо словник у рядок x-www-form-urlencoded
        var encoded = string.Join("&",
            fields.Select(f => $"{Uri.EscapeDataString(f.Key)}={Uri.EscapeDataString(f.Value)}"));

        _body = new RequestBody("application/x-www-form-urlencoded", encoded);

        if (_method == HttpMethod.GET)
            _method = HttpMethod.POST;

        return this;
    }

    public IHttpRequestBuilder WithTimeout(TimeSpan timeout)
    {
        if (timeout <= TimeSpan.Zero)
            throw new ArgumentException("Таймаут має бути більше нуля");

        _timeout = timeout;
        return this;
    }

    public IHttpRequestBuilder WithRetries(int count)
    {
        if (count < 0)
            throw new ArgumentException("Кількість повторів не може бути від'ємною");

        _maxRetries = count;
        return this;
    }

    public IHttpRequestBuilder WithFollowRedirects(bool follow)
    {
        _followRedirects = follow;
        return this;
    }

    // Метод Build — фінальний крок
    public HttpRequest Build()
    {
        // Перевіряємо обов'язкові поля перед побудовою
        if (string.IsNullOrWhiteSpace(_url))
            throw new InvalidOperationException("URL є обов'язковим полем. Викличте WithUrl() перед Build().");

        // Створюємо незмінний об'єкт
        return new HttpRequest(
            url:            _url,
            method:         _method,
            headers:        new Dictionary<string, string>(_headers),   // копія!
            queryParams:    new Dictionary<string, string>(_queryParams), // копія!
            body:           _body,
            timeout:        _timeout,
            followRedirects: _followRedirects,
            maxRetries:     _maxRetries
        );
    }

    // Скидаємо стан будівельника для повторного використання
    public HttpRequestBuilder Reset()
    {
        _url = null;
        _method = HttpMethod.GET;
        _headers.Clear();
        _queryParams.Clear();
        _body = null;
        _timeout = TimeSpan.FromSeconds(30);
        _followRedirects = true;
        _maxRetries = 0;
        return this;
    }
}
```

### Крок 5: Director — типові конфігурації

```csharp
// Director не будує сам — він знає ПОРЯДОК і НАБІР кроків для типових сценаріїв
// Це необов'язкова частина патерну, але дуже корисна для повторюваних конфігурацій
public class ApiRequestDirector
{
    private readonly IHttpRequestBuilder _builder;

    public ApiRequestDirector(IHttpRequestBuilder builder)
    {
        _builder = builder;
    }

    // Стандартний GET-запит до REST API з авторизацією
    public HttpRequest CreateAuthenticatedGetRequest(string url, string token)
    {
        return _builder
            .WithUrl(url)
            .WithMethod(HttpMethod.GET)
            .WithHeader("Authorization", $"Bearer {token}")
            .WithHeader("Accept", "application/json")
            .WithHeader("X-Api-Version", "2")
            .WithTimeout(TimeSpan.FromSeconds(10))
            .Build();
    }

    // POST-запит з JSON і повторними спробами
    public HttpRequest CreateRetryablePostRequest(string url, string jsonBody, string token)
    {
        return _builder
            .WithUrl(url)
            .WithHeader("Authorization", $"Bearer {token}")
            .WithHeader("Accept", "application/json")
            .WithJsonBody(jsonBody)
            .WithTimeout(TimeSpan.FromSeconds(30))
            .WithRetries(3)             // до 3 повторних спроб при помилці
            .WithFollowRedirects(false) // не слідувати редиректам для API
            .Build();
    }

    // Запит для завантаження файлу — довгий таймаут
    public HttpRequest CreateFileUploadRequest(string url, byte[] fileData, string contentType)
    {
        var body = new RequestBody(contentType, fileData);

        return _builder
            .WithUrl(url)
            .WithMethod(HttpMethod.POST)
            .WithHeader("Content-Type", contentType)
            .WithTimeout(TimeSpan.FromMinutes(5)) // файли можуть завантажуватись довго
            .WithRetries(1)
            .Build();
    }
}
```

### Крок 6: Використання

```csharp
class Program
{
    static void Main()
    {
        var builder = new HttpRequestBuilder();

        // --- Варіант 1: Будуємо вручну ---
        HttpRequest searchRequest = builder
            .WithUrl("https://api.example.com/search")
            .WithMethod(HttpMethod.GET)
            .WithHeader("Accept", "application/json")
            .WithQueryParam("q", "builder pattern")
            .WithQueryParam("lang", "csharp")
            .WithTimeout(TimeSpan.FromSeconds(15))
            .Build();

        Console.WriteLine("=== GET-запит ===");
        Console.WriteLine(searchRequest);

        // Скидаємо будівельник і будуємо ще один запит
        builder.Reset();

        HttpRequest createRequest = builder
            .WithUrl("https://api.example.com/articles")
            .WithJsonBody(@"{ ""title"": ""Builder Pattern"", ""lang"": ""csharp"" }")
            .WithHeader("Authorization", "Bearer my-secret-token")
            .WithRetries(2)
            .Build();

        Console.WriteLine("=== POST-запит ===");
        Console.WriteLine(createRequest);

        // --- Варіант 2: Через Director ---
        var director = new ApiRequestDirector(new HttpRequestBuilder());

        HttpRequest authRequest = director.CreateAuthenticatedGetRequest(
            url: "https://api.example.com/profile",
            token: "eyJhbGciOiJIUzI1NiJ9..."
        );

        Console.WriteLine("=== Запит через Director ===");
        Console.WriteLine(authRequest);

        // --- Варіант 3: Форма ---
        HttpRequest loginRequest = new HttpRequestBuilder()
            .WithUrl("https://example.com/login")
            .WithFormBody(new Dictionary<string, string>
            {
                ["username"] = "john_doe",
                ["password"] = "secret123"
            })
            .WithFollowRedirects(true)
            .Build();

        Console.WriteLine("=== Форма (login) ===");
        Console.WriteLine(loginRequest);
    }
}
```

### Очікуваний вивід

```
=== GET-запит ===
GET https://api.example.com/search
  Header: Accept: application/json
  Query params:
    q=builder pattern
    lang=csharp
  Timeout: 15s | Retries: 0 | Redirects: True

=== POST-запит ===
POST https://api.example.com/articles
  Header: Authorization: Bearer my-secret-token
  Body [application/json]: { "title": "Builder Pattern", "lang": "csharp" }
  Timeout: 30s | Retries: 2 | Redirects: True

=== Запит через Director ===
GET https://api.example.com/profile
  Header: Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
  Header: Accept: application/json
  Header: X-Api-Version: 2
  Timeout: 10s | Retries: 0 | Redirects: True

=== Форма (login) ===
POST https://example.com/login
  Body [application/x-www-form-urlencoded]: username=john_doe&password=secret123
  Timeout: 30s | Retries: 0 | Redirects: True
```

---

## Fluent Builder vs Classic Builder

Існує два основних варіанти реалізації:

### Classic Builder (з Director)

```csharp
// Розділення: Director знає ЩО будувати, Builder знає ЯК
var builder = new HouseBuilder();
var director = new Director(builder);

director.BuildSmallHouse();
var house = builder.GetResult();
```

Плюси: Чіткий поділ відповідальності, легко змінити Director.  
Мінуси: Більше класів, складніша структура.

### Fluent Builder (ланцюжок викликів)

```csharp
// Все в одному місці, методи повертають this
var house = new HouseBuilder()
    .WithFoundation("бетон")
    .WithWalls(3)
    .WithRoof("черепиця")
    .Build();
```

Плюси: Зручний API, читабельний код, менше класів.  
Мінуси: Builder знає і порядок, і деталі побудови.

> **Порада:** У сучасному C# Fluent Builder набагато популярніший. Classic Builder з Director корисний, коли є кілька чітко відмінних конфігурацій (наприклад, у тестах або фабриках).

---

## Переваги та недоліки

### Переваги

- **Читабельний код** — видно, що і як налаштовується
- **Гнучкість** — легко додавати нові параметри без зміни клієнтського коду
- **Валідація** — можна перевірити коректність у методі `Build()`
- **Immutability** — готовий об'єкт можна зробити незмінним
- **Повторне використання** — один Builder може будувати різні варіанти об'єкта

### Недоліки

- **Більше коду** — потрібно написати додатковий клас Builder
- **Дублювання полів** — поля є і в Builder, і в Product
- **Зайвий** для простих об'єктів — якщо у вас 2–3 параметри, Builder надмірний

---

## Підсумок

| Аспект | Деталь |
|---|---|
| Тип патерну | Породжуючий (Creational) |
| Вирішує проблему | Телескопічний конструктор, складна побудова |
| Ключова ідея | Крок за кроком + відокремлення побудови від представлення |
| Сигнатура | Методи повертають `this`, фінальний метод `.Build()` |
| Альтернативи | Factory Method, Abstract Factory, Prototype |

### Коротке правило вибору

```
Об'єкт має > 4 параметрів або складну логіку побудови?
  ✅ Так → Builder
  ❌ Ні  → Звичайний конструктор або object initializer
```

---

*Документ підготовлено як навчальний матеріал з патернів проєктування на C#.*
