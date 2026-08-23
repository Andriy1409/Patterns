# Патерн Chain of Responsibility (Ланцюжок обов'язків) — Детальний розбір на C#

> **Категорія:** Поведінковий (Behavioral)  
> **Автори:** Gang of Four (GoF)  
> **Мова прикладів:** C#

---

## Зміст

1. [Що таке Chain of Responsibility?](#що-таке-chain-of-responsibility)
2. [Проблема без патерну](#проблема-без-патерну)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Технічна підтримка (ескалація заявок)](#приклад-1--технічна-підтримка-ескалація-заявок)
5. [Приклад 2 — Middleware-конвеєр обробки запитів](#приклад-2--middleware-конвеєр-обробки-запитів)
6. [Приклад 3 — Ланцюжок затвердження витрат](#приклад-3--ланцюжок-затвердження-витрат)
7. [Приклад 4 — Реальний сценарій: маршрутизація звернень підтримки](#приклад-4--реальний-сценарій-маршрутизація-звернень-підтримки)
8. [Chain of Responsibility vs Decorator vs Command](#chain-of-responsibility-vs-decorator-vs-command)
9. [Переваги та недоліки](#переваги-та-недоліки)
10. [Антипатерни та поширені помилки](#антипатерни-та-поширені-помилки)
11. [Підсумок](#підсумок)

---

## Що таке Chain of Responsibility?

**Chain of Responsibility (Ланцюжок обов'язків)** — це поведінковий патерн, який дозволяє передавати запит послідовно через **ланцюжок обробників**. Кожен обробник отримує запит і вирішує: **обробити його самостійно** чи **передати далі** наступному обробнику в ланцюжку.

Головна ідея — **розірвати зв'язок** між відправником запиту і конкретним отримувачем. Відправник просто кидає запит у ланцюжок і не знає (і не повинен знати), хто саме його обробить — перший обробник, останній, чи взагалі кілька одразу.

### Ключова відмінність від звичайного виклику методу

```
Звичайний виклик:           Chain of Responsibility:

Client                       Client
  │                            │
  │ знає точно, кого викликати │ не знає, хто обробить
  ▼                            ▼
Receiver                    Handler A ──?── Handler B ──?── Handler C
                             (сам вирішує, чи обробляти, чи передати далі)
```

Кожен обробник у ланцюжку:
1. Отримує запит.
2. Перевіряє: **"чи можу я його обробити?"**
3. Якщо так — обробляє (і, залежно від реалізації, або зупиняє ланцюжок, або все одно передає далі).
4. Якщо ні — передає запит **наступному** обробнику в ланцюжку.

Якщо жоден обробник не впорався — запит або "губиться" (погана практика), або потрапляє до термінального обробника за замовчуванням.

### Аналогія з реального світу

Уяви типову компанію зі структурою підпорядкування, де співробітник хоче отримати дозвіл на відрядження вартістю 15 000 грн:

```
Співробітник подає заявку на 15 000 грн
        │
        ▼
┌───────────────────┐   ліміт 5 000  →  не може затвердити, передає далі
│   Тімлід           │
└─────────┬──────────┘
          │ передає далі
          ▼
┌───────────────────┐   ліміт 20 000 →  МОЖЕ затвердити!
│   Керівник відділу │
└─────────┬──────────┘
          │ затверджено, ланцюжок зупинено
          ▼
      ✅ Заявку затверджено
```

Співробітнику **не потрібно знати**, хто саме має повноваження затвердити його заявку. Він просто подає її "вгору по ланцюжку", а система сама знаходить того, хто може прийняти рішення.

Такий самий принцип працює:
- У **службі підтримки** — заявка ескалюється від оператора першої лінії (L1) до другої (L2) і третьої (L3), поки хтось не матиме достатньо кваліфікації її вирішити.
- У **middleware-конвеєрах** веб-фреймворків (ASP.NET Core, Express.js) — HTTP-запит проходить через ланцюжок обробників (автентифікація, логування, лімітування), кожен з яких може обробити запит сам або передати його далі.
- У **фільтрах електронної пошти** — лист проходить через ланцюжок правил (спам-фільтр, фільтр за відправником, фільтр за темою), поки якийсь фільтр не "забере" його у відповідну папку.

---

## Проблема без патерну

Розглянемо систему обробки заявок у службі підтримки без застосування патерну. Найпростіший (і найгірший) спосіб — написати один величезний метод з ланцюжком `if-else`, що перевіряє все підряд.

```csharp
// ПОГАНИЙ ПІДХІД: один метод "знає" про всі рівні підтримки і всі правила одразу.
public class SupportTicketProcessor
{
    public void ProcessTicket(int difficulty, string description)
    {
        // Проблема 1: метод повинен знати ВСІ можливі рівні підтримки
        // і ВСІ умови, за яких кожен з них спрацьовує.
        if (difficulty <= 3)
        {
            Console.WriteLine($"[L1] Оператор першої лінії обробляє: {description}");
            // логіка L1...
        }
        else if (difficulty <= 6)
        {
            Console.WriteLine($"[L2] Технічний спеціаліст обробляє: {description}");
            // логіка L2...
        }
        else if (difficulty <= 9)
        {
            Console.WriteLine($"[L3] Інженер обробляє: {description}");
            // логіка L3...
        }
        else
        {
            Console.WriteLine($"[CTO] Критична проблема, ескалюємо на CTO: {description}");
            // логіка CTO...
        }

        // Проблема 2: якщо завтра з'явиться новий рівень підтримки
        // (наприклад, "чергування вихідного дня"), доведеться
        // ЗМІНЮВАТИ цей метод — порушення Open/Closed Principle.

        // Проблема 3: щоб ЗМІНИТИ порядок перевірки (наприклад,
        // спочатку перевіряти VIP-клієнтів, а вже потім складність),
        // потрібно переписувати структуру if-else вручну.

        // Проблема 4: неможливо динамічно зібрати ІНШИЙ ланцюжок
        // для іншого сценарію (наприклад, тестового середовища
        // без CTO) — все зашито в одному методі.

        // Проблема 5: метод стає дедалі більшим з кожним новим правилом —
        // класичний "God Method", який важко тестувати і підтримувати.
    }
}
```

**Проблеми такого підходу:**

- **Порушення Open/Closed Principle** — щоб додати новий рівень обробки, потрібно редагувати існуючий метод, а не додавати новий код поруч.
- **Неможливо змінити порядок обробників** без переписування логіки вручну.
- **Один метод знає забагато** — він одночасно відповідає і за маршрутизацію, і за бізнес-логіку кожного рівня.
- **Складно тестувати** — неможливо протестувати "рівень L2" ізольовано, він схований усередині великого методу.
- **Неможлива повторна конфігурація** — не можна на льоту зібрати інший ланцюжок обробників (наприклад, без певного рівня) для іншого середовища чи клієнта.

Chain of Responsibility вирішує все це, розбиваючи логіку на окремі, незалежні обробники, які самі вирішують — обробляти запит чи передати далі.

---

## Структура патерну

```
┌────────────────────────────┐
│      «abstract»            │
│      Handler                │
├────────────────────────────┤
│ # _next : Handler           │
├────────────────────────────┤
│ + SetNext(handler): Handler │
│ + Handle(request)            │◄──── Client викликає перший обробник
└──────────────┬──────────────┘        у ланцюжку
               │ extends
   ┌───────────┼────────────────┐
   │           │                │
┌──▼──────┐ ┌──▼──────┐   ┌─────▼─────┐
│ Concrete │ │ Concrete │   │ Concrete  │
│ HandlerA │ │ HandlerB │   │ HandlerC  │
├──────────┤ ├──────────┤   ├───────────┤
│Handle():  │ │Handle(): │   │Handle():  │
│ якщо можу │ │ якщо можу│   │ якщо можу │
│  → обробити│ │ → обробити│  │ → обробити│
│ інакше    │ │ інакше    │  │ інакше    │
│  → _next  │ │  → _next  │  │  → _next  │
│   .Handle()│ │  .Handle()│  │  .Handle()│
└──────────┘ └──────────┘   └───────────┘

Ланцюжок:  HandlerA → HandlerB → HandlerC → (null / термінальний обробник)
```

### Учасники

| Роль | Відповідальність |
|---|---|
| **Handler** (абстрактний) | Оголошує метод `Handle(request)` і зберігає посилання на наступний обробник (`_next`). Часто має метод `SetNext()` для побудови ланцюжка. |
| **ConcreteHandlerA/B/C** | Конкретні обробники. Кожен вирішує самостійно: обробити запит чи передати далі через `_next.Handle(request)`. |
| **Client** | Формує ланцюжок (з'єднує обробники між собою) і надсилає запит **першому** обробнику в ланцюжку. Клієнту байдуже, хто саме в результаті обробить запит. |

### Дві типові реалізації методу `Handle`

```
1) "Хтось один обробляє і зупиняє ланцюжок" (типово для ескалації):

   Handle(request):
       if (можу обробити) { обробити; return; }
       else { _next?.Handle(request); }

2) "Кожен обробник щось додає і завжди передає далі" (типово для middleware):

   Handle(request):
       зробити щось ДО (наприклад, перевірка/логування)
       if (потрібно зупинити) { return; }   // короткий обрив (short-circuit)
       _next?.Handle(request);
       зробити щось ПІСЛЯ (опційно)
```

Обидва варіанти — це Chain of Responsibility. Різниця лише в тому, скільки обробників фактично щось "роблять" із запитом.

---

## Приклад 1 — Технічна підтримка (ескалація заявок)

Найпростіший приклад для розуміння механіки: заявки в службу підтримки ескалюються від першої лінії до третьої залежно від складності.

### Крок 1: Модель заявки

```csharp
// Заявка в службу підтримки
public class SupportTicket
{
    public int Id { get; }
    public string Description { get; }

    // Складність від 1 (тривіальна) до 10 (критична)
    public int Difficulty { get; }

    public SupportTicket(int id, string description, int difficulty)
    {
        Id = id;
        Description = description;
        Difficulty = difficulty;
    }
}
```

### Крок 2: Абстрактний обробник

```csharp
// Базовий абстрактний обробник ланцюжка
public abstract class SupportHandler
{
    // Посилання на наступний обробник у ланцюжку
    private SupportHandler _next;

    // Дозволяє клієнту (або будівельнику ланцюжка) з'єднати обробники між собою.
    // Повертає переданий обробник — це дозволяє робити "текучий" (fluent) виклик:
    // handlerA.SetNext(handlerB).SetNext(handlerC);
    public SupportHandler SetNext(SupportHandler next)
    {
        _next = next;
        return next;
    }

    // Шаблонний метод: перевіряє, чи може ЦЕЙ обробник впоратись,
    // інакше передає запит далі по ланцюжку.
    public void Handle(SupportTicket ticket)
    {
        if (CanHandle(ticket))
        {
            Process(ticket);
        }
        else if (_next != null)
        {
            Console.WriteLine($"  [{GetType().Name}] Не можу обробити заявку #{ticket.Id}, передаю далі...");
            _next.Handle(ticket);
        }
        else
        {
            // Термінальна ланка — ніхто в ланцюжку не впорався
            Console.WriteLine($"  [!] Заявку #{ticket.Id} НЕ ОБРОБЛЕНО — ланцюжок вичерпано.");
        }
    }

    // Кожен конкретний обробник визначає, які заявки він може обробити
    protected abstract bool CanHandle(SupportTicket ticket);

    // І як саме він їх обробляє
    protected abstract void Process(SupportTicket ticket);
}
```

### Крок 3: Конкретні обробники

```csharp
// Перша лінія: тривіальні питання (складність 1-3)
public class Level1Support : SupportHandler
{
    protected override bool CanHandle(SupportTicket ticket) => ticket.Difficulty <= 3;

    protected override void Process(SupportTicket ticket)
        => Console.WriteLine($"[L1 Support] ✅ Заявку #{ticket.Id} вирішено: \"{ticket.Description}\"");
}

// Друга лінія: технічні питання середньої складності (4-6)
public class Level2Support : SupportHandler
{
    protected override bool CanHandle(SupportTicket ticket) => ticket.Difficulty <= 6;

    protected override void Process(SupportTicket ticket)
        => Console.WriteLine($"[L2 Support] ✅ Заявку #{ticket.Id} вирішено технічним спеціалістом: \"{ticket.Description}\"");
}

// Третя лінія: складні інженерні питання (7-9)
public class Level3Support : SupportHandler
{
    protected override bool CanHandle(SupportTicket ticket) => ticket.Difficulty <= 9;

    protected override void Process(SupportTicket ticket)
        => Console.WriteLine($"[L3 Support] ✅ Заявку #{ticket.Id} вирішено інженером: \"{ticket.Description}\"");
}

// Найвищий рівень: критичні проблеми (10) — ескалація на керівника
public class ManagerEscalation : SupportHandler
{
    protected override bool CanHandle(SupportTicket ticket) => ticket.Difficulty >= 10;

    protected override void Process(SupportTicket ticket)
        => Console.WriteLine($"[Manager] 🔥 Критична заявка #{ticket.Id} передана керівнику: \"{ticket.Description}\"");
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        // Будуємо ланцюжок: L1 → L2 → L3 → Manager
        var level1 = new Level1Support();
        var level2 = new Level2Support();
        var level3 = new Level3Support();
        var manager = new ManagerEscalation();

        level1.SetNext(level2).SetNext(level3).SetNext(manager);

        var tickets = new List<SupportTicket>
        {
            new(1, "Не працює кнопка \"Зберегти\"", difficulty: 2),
            new(2, "Помилка синхронізації даних", difficulty: 5),
            new(3, "Витік пам'яті у продакшн-сервісі", difficulty: 8),
            new(4, "Повний збій платіжної системи", difficulty: 10),
        };

        foreach (var ticket in tickets)
        {
            Console.WriteLine($"\n--- Нова заявка #{ticket.Id} (складність {ticket.Difficulty}) ---");
            level1.Handle(ticket); // завжди надсилаємо ПЕРШОМУ обробнику в ланцюжку
        }
    }
}
```

### Очікуваний вивід

```
--- Нова заявка #1 (складність 2) ---
[L1 Support] ✅ Заявку #1 вирішено: "Не працює кнопка "Зберегти""

--- Нова заявка #2 (складність 5) ---
  [Level1Support] Не можу обробити заявку #2, передаю далі...
[L2 Support] ✅ Заявку #2 вирішено технічним спеціалістом: "Помилка синхронізації даних"

--- Нова заявка #3 (складність 8) ---
  [Level1Support] Не можу обробити заявку #3, передаю далі...
  [Level2Support] Не можу обробити заявку #3, передаю далі...
[L3 Support] ✅ Заявку #3 вирішено інженером: "Витік пам'яті у продакшн-сервісі"

--- Нова заявка #4 (складність 10) ---
  [Level1Support] Не можу обробити заявку #4, передаю далі...
  [Level2Support] Не можу обробити заявку #4, передаю далі...
  [Level3Support] Не можу обробити заявку #4, передаю далі...
[Manager] 🔥 Критична заявка #4 передана керівнику: "Повний збій платіжної системи"
```

Клієнтський код (`Main`) завжди звертається лише до `level1` — першого обробника. Він **не знає і не має знати**, хто саме врешті-решт вирішить заявку.

---

## Приклад 2 — Middleware-конвеєр обробки запитів

Другий класичний випадок застосування Chain of Responsibility — конвеєр обробки HTTP-подібних запитів, як у ASP.NET Core. На відміну від Прикладу 1 (де запит обробляє **рівно один** обробник), тут кожен middleware **завжди щось робить** (перевіряє, логує), а потім або зупиняє конвеєр, або передає керування далі.

### Крок 1: Модель запиту та контексту

```csharp
// Вхідний HTTP-подібний запит
public class HttpRequest
{
    public string Path { get; init; }
    public string Method { get; init; } = "GET";
    public string ApiKey { get; init; }
    public string ClientIp { get; init; }
}

// Контекст обробки — сюди middleware записують результат і статус
public class RequestContext
{
    public HttpRequest Request { get; }
    public int StatusCode { get; set; } = 200;
    public string ResponseBody { get; set; }

    // Прапорець: чи потрібно зупинити конвеєр (короткий обрив)
    public bool ShortCircuited { get; private set; }

    public RequestContext(HttpRequest request) => Request = request;

    public void ShortCircuit(int statusCode, string body)
    {
        StatusCode = statusCode;
        ResponseBody = body;
        ShortCircuited = true;
    }
}
```

### Крок 2: Абстрактний middleware

```csharp
// Базовий клас для всіх middleware у конвеєрі
public abstract class Middleware
{
    private Middleware _next;

    public Middleware SetNext(Middleware next)
    {
        _next = next;
        return next;
    }

    // Публічний метод, який викликає клієнт (або попередній middleware)
    public void Invoke(RequestContext context)
    {
        HandleRequest(context);

        // Якщо поточний middleware НЕ зупинив конвеєр — передаємо далі.
        // Якщо _next немає (кінець ланцюжка) — конвеєр завершується.
        if (!context.ShortCircuited)
        {
            _next?.Invoke(context);
        }
    }

    protected abstract void HandleRequest(RequestContext context);
}
```

### Крок 3: Конкретні middleware

```csharp
// Перевірка автентифікації — перший рубіж
public class AuthenticationMiddleware : Middleware
{
    private readonly HashSet<string> _validApiKeys;

    public AuthenticationMiddleware(IEnumerable<string> validApiKeys)
        => _validApiKeys = new HashSet<string>(validApiKeys);

    protected override void HandleRequest(RequestContext context)
    {
        Console.WriteLine("[Auth] Перевірка API-ключа...");

        if (string.IsNullOrEmpty(context.Request.ApiKey) ||
            !_validApiKeys.Contains(context.Request.ApiKey))
        {
            Console.WriteLine("[Auth] ❌ Невалідний або відсутній API-ключ — зупиняємо конвеєр");
            context.ShortCircuit(401, "Unauthorized: невалідний API-ключ");
            return; // не передаємо далі — конвеєр зупинено
        }

        Console.WriteLine("[Auth] ✅ Автентифікація пройдена");
    }
}

// Логування — фіксує кожен запит незалежно від результату
public class LoggingMiddleware : Middleware
{
    protected override void HandleRequest(RequestContext context)
    {
        Console.WriteLine($"[Logging] → {context.Request.Method} {context.Request.Path} " +
                           $"від {context.Request.ClientIp}");
    }
}

// Обмеження частоти запитів (rate limiting)
public class RateLimitMiddleware : Middleware
{
    // Спрощений лічильник запитів по IP (у реальному коді — з вікном часу)
    private readonly Dictionary<string, int> _requestCounts = new();
    private readonly int _maxRequestsPerClient;

    public RateLimitMiddleware(int maxRequestsPerClient)
        => _maxRequestsPerClient = maxRequestsPerClient;

    protected override void HandleRequest(RequestContext context)
    {
        var ip = context.Request.ClientIp;
        _requestCounts.TryGetValue(ip, out var count);
        count++;
        _requestCounts[ip] = count;

        Console.WriteLine($"[RateLimit] Запит #{count} від {ip} (ліміт {_maxRequestsPerClient})");

        if (count > _maxRequestsPerClient)
        {
            Console.WriteLine("[RateLimit] ❌ Перевищено ліміт запитів — зупиняємо конвеєр");
            context.ShortCircuit(429, "Too Many Requests");
            return;
        }
    }
}

// Фінальний обробник — власне бізнес-логіка
public class RequestHandler : Middleware
{
    protected override void HandleRequest(RequestContext context)
    {
        Console.WriteLine($"[Handler] Обробляємо бізнес-логіку для {context.Request.Path}");
        context.StatusCode = 200;
        context.ResponseBody = $"{{\"status\": \"ok\", \"path\": \"{context.Request.Path}\"}}";
    }
}
```

### Крок 4: Складання конвеєра і використання

```csharp
class Program
{
    static void Main()
    {
        // Будуємо конвеєр: Auth → Logging → RateLimit → RequestHandler
        var auth = new AuthenticationMiddleware(validApiKeys: new[] { "secret-key-123" });
        var logging = new LoggingMiddleware();
        var rateLimit = new RateLimitMiddleware(maxRequestsPerClient: 2);
        var handler = new RequestHandler();

        auth.SetNext(logging).SetNext(rateLimit).SetNext(handler);

        // --- Запит 1: валідний ключ, перший запит з цього IP ---
        Console.WriteLine("=== Запит 1 (валідний, перший) ===");
        var ctx1 = new RequestContext(new HttpRequest
        {
            Path = "/api/users", ApiKey = "secret-key-123", ClientIp = "192.168.1.10"
        });
        auth.Invoke(ctx1);
        Console.WriteLine($"Результат: {ctx1.StatusCode} — {ctx1.ResponseBody}\n");

        // --- Запит 2: невалідний ключ ---
        Console.WriteLine("=== Запит 2 (невалідний ключ) ===");
        var ctx2 = new RequestContext(new HttpRequest
        {
            Path = "/api/orders", ApiKey = "wrong-key", ClientIp = "192.168.1.11"
        });
        auth.Invoke(ctx2);
        Console.WriteLine($"Результат: {ctx2.StatusCode} — {ctx2.ResponseBody}\n");

        // --- Запити 3 і 4 з того самого IP, що й запит 1: перевищення ліміту ---
        Console.WriteLine("=== Запит 3 (той самий IP, 2-й запит) ===");
        var ctx3 = new RequestContext(new HttpRequest
        {
            Path = "/api/users", ApiKey = "secret-key-123", ClientIp = "192.168.1.10"
        });
        auth.Invoke(ctx3);
        Console.WriteLine($"Результат: {ctx3.StatusCode} — {ctx3.ResponseBody}\n");

        Console.WriteLine("=== Запит 4 (той самий IP, 3-й запит — перевищено ліміт) ===");
        var ctx4 = new RequestContext(new HttpRequest
        {
            Path = "/api/users", ApiKey = "secret-key-123", ClientIp = "192.168.1.10"
        });
        auth.Invoke(ctx4);
        Console.WriteLine($"Результат: {ctx4.StatusCode} — {ctx4.ResponseBody}");
    }
}
```

### Очікуваний вивід

```
=== Запит 1 (валідний, перший) ===
[Auth] Перевірка API-ключа...
[Auth] ✅ Автентифікація пройдена
[Logging] → GET /api/users від 192.168.1.10
[RateLimit] Запит #1 від 192.168.1.10 (ліміт 2)
[Handler] Обробляємо бізнес-логіку для /api/users
Результат: 200 — {"status": "ok", "path": "/api/users"}

=== Запит 2 (невалідний ключ) ===
[Auth] Перевірка API-ключа...
[Auth] ❌ Невалідний або відсутній API-ключ — зупиняємо конвеєр
Результат: 401 — Unauthorized: невалідний API-ключ

=== Запит 3 (той самий IP, 2-й запит) ===
[Auth] Перевірка API-ключа...
[Auth] ✅ Автентифікація пройдена
[Logging] → GET /api/users від 192.168.1.10
[RateLimit] Запит #2 від 192.168.1.10 (ліміт 2)
[Handler] Обробляємо бізнес-логіку для /api/users
Результат: 200 — {"status": "ok", "path": "/api/users"}

=== Запит 4 (той самий IP, 3-й запит — перевищено ліміт) ===
[Auth] Перевірка API-ключа...
[Auth] ✅ Автентифікація пройдена
[Logging] → GET /api/users від 192.168.1.10
[RateLimit] Запит #3 від 192.168.1.10 (ліміт 2)
[RateLimit] ❌ Перевищено ліміт запитів — зупиняємо конвеєр
Результат: 429 — Too Many Requests
```

Зауваж різницю з Прикладом 1: тут **кілька** обробників (Auth, Logging, RateLimit) щось роблять із запитом послідовно — і лише останній (`RequestHandler`) виконує "основну" роботу, якщо жоден не зупинив конвеєр раніше.

---

## Приклад 3 — Ланцюжок затвердження витрат

Третій приклад — класичний бізнес-кейс: заявка на витрати проходить ланцюжок посадових осіб, кожна з яких має свій ліміт повноважень.

### Крок 1: Модель заявки

```csharp
public class ExpenseRequest
{
    public string Requester { get; }
    public decimal Amount { get; }
    public string Purpose { get; }

    public ExpenseRequest(string requester, decimal amount, string purpose)
    {
        Requester = requester;
        Amount = amount;
        Purpose = purpose;
    }
}
```

### Крок 2: Абстрактний затверджувач

```csharp
public abstract class Approver
{
    protected string Name { get; }
    protected decimal ApprovalLimit { get; }
    private Approver _next;

    protected Approver(string name, decimal approvalLimit)
    {
        Name = name;
        ApprovalLimit = approvalLimit;
    }

    public Approver SetNext(Approver next)
    {
        _next = next;
        return next;
    }

    public void ProcessRequest(ExpenseRequest request)
    {
        if (request.Amount <= ApprovalLimit)
        {
            Console.WriteLine($"✅ {Name} затвердив {request.Amount:C} на \"{request.Purpose}\" " +
                               $"(ліміт {ApprovalLimit:C}) — заявник: {request.Requester}");
            return;
        }

        if (_next != null)
        {
            Console.WriteLine($"↑ {Name} не може затвердити {request.Amount:C} " +
                               $"(ліміт лише {ApprovalLimit:C}) — ескалюю до наступного рівня");
            _next.ProcessRequest(request);
        }
        else
        {
            Console.WriteLine($"❌ Заявку на {request.Amount:C} НЕ затверджено — " +
                               "перевищує повноваження всіх рівнів у компанії");
        }
    }
}
```

### Крок 3: Конкретні затверджувачі

```csharp
public class TeamLeadApprover : Approver
{
    public TeamLeadApprover() : base("Тімлід", approvalLimit: 5_000m) { }
}

public class ManagerApprover : Approver
{
    public ManagerApprover() : base("Керівник відділу", approvalLimit: 20_000m) { }
}

public class DirectorApprover : Approver
{
    public DirectorApprover() : base("Директор", approvalLimit: 100_000m) { }
}

public class CeoApprover : Approver
{
    // CEO може затвердити будь-яку суму в межах компанії
    public CeoApprover() : base("CEO", approvalLimit: decimal.MaxValue) { }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        var teamLead = new TeamLeadApprover();
        var manager  = new ManagerApprover();
        var director = new DirectorApprover();
        var ceo      = new CeoApprover();

        teamLead.SetNext(manager).SetNext(director).SetNext(ceo);

        var requests = new List<ExpenseRequest>
        {
            new("Олена (розробник)", 3_500m, "Ліцензія на IDE"),
            new("Максим (тімлід)", 15_000m, "Тімбілдинг для команди"),
            new("Ірина (керівник відділу)", 75_000m, "Нові ноутбуки для команди"),
            new("Андрій (директор)", 500_000m, "Придбання серверного обладнання"),
        };

        foreach (var request in requests)
        {
            Console.WriteLine($"\n--- Заявка від {request.Requester}: {request.Amount:C} ---");
            teamLead.ProcessRequest(request);
        }
    }
}
```

### Очікуваний вивід

```
--- Заявка від Олена (розробник): ₴3 500,00 ---
✅ Тімлід затвердив ₴3 500,00 на "Ліцензія на IDE" (ліміт ₴5 000,00) — заявник: Олена (розробник)

--- Заявка від Максим (тімлід): ₴15 000,00 ---
↑ Тімлід не може затвердити ₴15 000,00 (ліміт лише ₴5 000,00) — ескалюю до наступного рівня
✅ Керівник відділу затвердив ₴15 000,00 на "Тімбілдинг для команди" (ліміт ₴20 000,00) — заявник: Максим (тімлід)

--- Заявка від Ірина (керівник відділу): ₴75 000,00 ---
↑ Тімлід не може затвердити ₴75 000,00 (ліміт лише ₴5 000,00) — ескалюю до наступного рівня
↑ Керівник відділу не може затвердити ₴75 000,00 (ліміт лише ₴20 000,00) — ескалюю до наступного рівня
✅ Директор затвердив ₴75 000,00 на "Нові ноутбуки для команди" (ліміт ₴100 000,00) — заявник: Ірина (керівник відділу)

--- Заявка від Андрій (директор): ₴500 000,00 ---
↑ Тімлід не може затвердити ₴500 000,00 (ліміт лише ₴5 000,00) — ескалюю до наступного рівня
↑ Керівник відділу не може затвердити ₴500 000,00 (ліміт лише ₴20 000,00) — ескалюю до наступного рівня
↑ Директор не може затвердити ₴500 000,00 (ліміт лише ₴100 000,00) — ескалюю до наступного рівня
✅ CEO затвердив ₴500 000,00 на "Придбання серверного обладнання" (ліміт ₴79 228 162 514 264 337 593 543 950 335,00) — заявник: Андрій (директор)
```

> Зверни увагу: у реальному коді для CEO варто використати не `decimal.MaxValue`, а окрему логіку "без ліміту" (наприклад, `bool CanApproveAnyAmount`), щоб уникнути потворного форматування у виводі. Тут значення залишено навмисно, щоб показати граничний випадок.

---

## Приклад 4 — Реальний сценарій: маршрутизація звернень підтримки

У попередніх прикладах ланцюжок будувався "вручну" прямо в `Main`. У реальних системах ланцюжок зазвичай **конфігурується один раз при старті додатку** через спеціальний будівельник (builder), а кожен обробник не просто передає запит далі, а **збагачує** об'єкт додатковою інформацією і веде **журнал шляху**, яким пройшов запит.

Розглянемо систему класифікації та маршрутизації вхідних звернень клієнтів служби підтримки.

### Крок 1: Доменна модель — квиток підтримки

```csharp
// Квиток служби підтримки, що проходить через ланцюжок обробників
public class SupportTicket
{
    public int Id { get; }
    public string CustomerEmail { get; }
    public bool IsPriorityCustomer { get; }
    public string Subject { get; }
    public string Body { get; }

    // Обробники додають сюди мітки під час проходження ланцюжка
    public List<string> Tags { get; } = new();

    // Куди врешті-решт направлено квиток (визначається обробниками)
    public string RoutedTo { get; private set; } = "Невизначено";

    // Журнал шляху — кожен обробник, через який пройшов квиток, лишає запис
    public List<string> ProcessingLog { get; } = new();

    // Прапорець: чи потрібно зупинити подальшу обробку (наприклад, спам)
    public bool IsBlocked { get; private set; }

    public SupportTicket(int id, string customerEmail, bool isPriorityCustomer,
                          string subject, string body)
    {
        Id = id;
        CustomerEmail = customerEmail;
        IsPriorityCustomer = isPriorityCustomer;
        Subject = subject;
        Body = body;
    }

    public void RouteTo(string queue) => RoutedTo = queue;

    public void Block(string reason)
    {
        IsBlocked = true;
        RoutedTo = $"Заблоковано ({reason})";
    }

    public void Log(string entry) => ProcessingLog.Add(entry);

    public void PrintSummary()
    {
        Console.WriteLine($"\n=== Квиток #{Id} від {CustomerEmail} ===");
        Console.WriteLine($"Тема: \"{Subject}\"");
        Console.WriteLine($"Мітки: {(Tags.Count > 0 ? string.Join(", ", Tags) : "немає")}");
        Console.WriteLine($"Направлено до: {RoutedTo}");
        Console.WriteLine("Шлях обробки:");
        foreach (var entry in ProcessingLog)
            Console.WriteLine($"  → {entry}");
    }
}
```

### Крок 2: Абстрактний обробник ланцюжка

```csharp
// Базовий клас для всіх обробників маршрутизації квитків.
// На відміну від Прикладу 1, тут обробники не "конкурують" за право
// обробити квиток одноосібно — вони можуть і АНОТУВАТИ (додати мітку),
// і ПЕРЕНАПРАВИТИ (зупинити ланцюжок), залежно від логіки.
public abstract class TicketHandler
{
    // Ім'я обробника — для журналювання
    protected abstract string HandlerName { get; }

    private TicketHandler _next;

    public TicketHandler SetNext(TicketHandler next)
    {
        _next = next;
        return next;
    }

    public void Handle(SupportTicket ticket)
    {
        // Кожен обробник виконує свою перевірку/анотацію
        var stopChain = Process(ticket);

        if (stopChain)
        {
            ticket.Log($"{HandlerName}: ланцюжок зупинено на цьому кроці");
            return;
        }

        if (_next != null)
        {
            _next.Handle(ticket);
        }
        else
        {
            // Кінець ланцюжка — якщо ніхто не направив квиток, це помилка конфігурації
            if (ticket.RoutedTo == "Невизначено")
            {
                ticket.Log($"{HandlerName}: КІНЕЦЬ ЛАНЦЮЖКА — квиток лишився не направленим!");
                ticket.RouteTo("Немаршрутизовано (потребує ручного втручання)");
            }
        }
    }

    // Повертає true, якщо ланцюжок потрібно зупинити (квиток вже "вирішено" цим кроком)
    protected abstract bool Process(SupportTicket ticket);
}
```

### Крок 3: Конкретні обробники

```csharp
// 1. Фільтр спаму — перший рубіж, перевіряє на явні ознаки спаму
public class SpamFilterHandler : TicketHandler
{
    protected override string HandlerName => "SpamFilter";

    private static readonly string[] SpamKeywords =
        { "виграш", "безкоштовно 100%", "негайно натисніть", "криптовалюта x100" };

    protected override bool Process(SupportTicket ticket)
    {
        var text = (ticket.Subject + " " + ticket.Body).ToLowerInvariant();
        var isSpam = SpamKeywords.Any(keyword => text.Contains(keyword));

        if (isSpam)
        {
            ticket.Tags.Add("spam");
            ticket.Block("виявлено ознаки спаму");
            ticket.Log($"{HandlerName}: заблоковано як спам ❌");
            return true; // зупиняємо ланцюжок — далі спам не йде
        }

        ticket.Log($"{HandlerName}: спаму не виявлено ✅");
        return false; // продовжуємо ланцюжок
    }
}

// 2. Пріоритетні клієнти — VIP-квитки одразу йдуть у окрему чергу
public class PriorityCustomerHandler : TicketHandler
{
    protected override string HandlerName => "PriorityCustomer";

    protected override bool Process(SupportTicket ticket)
    {
        if (ticket.IsPriorityCustomer)
        {
            ticket.Tags.Add("vip");
            ticket.RouteTo("Пріоритетна черга (VIP)");
            ticket.Log($"{HandlerName}: клієнт має VIP-статус — направлено у пріоритетну чергу 📩");
            return true; // VIP-квитки не проходять через звичайну маршрутизацію за ключовими словами
        }

        ticket.Log($"{HandlerName}: клієнт не має пріоритетного статусу, продовжуємо");
        return false;
    }
}

// 3. Маршрутизація за ключовими словами у темі/тексті звернення
public class KeywordRoutingHandler : TicketHandler
{
    protected override string HandlerName => "KeywordRouting";

    // Мапа: ключове слово → черга, куди направляти
    private static readonly Dictionary<string, string> RoutingRules = new(StringComparer.OrdinalIgnoreCase)
    {
        ["оплат"]    = "Черга білінгу",
        ["рахунок"]  = "Черга білінгу",
        ["помилка"]  = "Технічна черга",
        ["не працює"] = "Технічна черга",
        ["купити"]   = "Черга продажів",
        ["тариф"]    = "Черга продажів",
    };

    protected override bool Process(SupportTicket ticket)
    {
        var text = ticket.Subject + " " + ticket.Body;

        foreach (var rule in RoutingRules)
        {
            if (text.Contains(rule.Key, StringComparison.OrdinalIgnoreCase))
            {
                ticket.Tags.Add($"keyword:{rule.Key.Trim()}");
                ticket.RouteTo(rule.Value);
                ticket.Log($"{HandlerName}: знайдено ключове слово \"{rule.Key}\" → {rule.Value} 🔀");
                return true; // маршрут визначено, зупиняємо ланцюжок
            }
        }

        ticket.Log($"{HandlerName}: жодне ключове слово не збіглося, продовжуємо");
        return false;
    }
}

// 4. Обробник за замовчуванням — термінальна ланка ланцюжка
public class DefaultQueueHandler : TicketHandler
{
    protected override string HandlerName => "DefaultQueue";

    protected override bool Process(SupportTicket ticket)
    {
        ticket.Tags.Add("general");
        ticket.RouteTo("Загальна черга");
        ticket.Log($"{HandlerName}: жоден спеціалізований обробник не спрацював — загальна черга");
        return true; // це завжди останній крок, що фактично щось вирішує
    }
}
```

### Крок 4: Будівельник ланцюжка (централізована конфігурація)

```csharp
// Централізоване місце побудови ланцюжка обробників.
// Замінює розкидані по коду виклики SetNext() одним чітким методом,
// який легко змінити, розширити чи покрити тестами.
public static class SupportChainBuilder
{
    public static TicketHandler BuildDefaultChain()
    {
        var spamFilter       = new SpamFilterHandler();
        var priorityCustomer = new PriorityCustomerHandler();
        var keywordRouting   = new KeywordRoutingHandler();
        var defaultQueue     = new DefaultQueueHandler();

        // Порядок навмисно такий:
        // 1) спочатку відсіюємо спам,
        // 2) потім перевіряємо VIP-статус (він має пріоритет над ключовими словами),
        // 3) потім намагаємось розпізнати тему звернення,
        // 4) і лише якщо нічого не спрацювало — загальна черга.
        spamFilter
            .SetNext(priorityCustomer)
            .SetNext(keywordRouting)
            .SetNext(defaultQueue);

        return spamFilter; // повертаємо ГОЛОВУ ланцюжка
    }
}
```

### Крок 5: Демонстрація — `Program.Main`

```csharp
class Program
{
    static void Main()
    {
        // Ланцюжок будується ОДИН раз при старті застосунку
        TicketHandler routingChain = SupportChainBuilder.BuildDefaultChain();

        var tickets = new List<SupportTicket>
        {
            new(1, "ivan.petrenko@mail.com", isPriorityCustomer: false,
                subject: "Не працює вхід в акаунт",
                body: "Після оновлення додатку не можу увійти, видає помилку."),

            new(2, "vip.client@bigcorp.com", isPriorityCustomer: true,
                subject: "Питання щодо контракту",
                body: "Потрібно обговорити умови продовження контракту на наступний рік."),

            new(3, "maria.k@example.com", isPriorityCustomer: false,
                subject: "Не можу оплатити підписку",
                body: "Картка відхиляється при спробі оплатити рахунок за місяць."),

            new(4, "spammer123@fake.com", isPriorityCustomer: false,
                subject: "Ви виграш!!! Натисніть негайно натисніть",
                body: "Отримайте безкоштовно 100% бонус у криптовалюта x100!"),

            new(5, "novyi.user@example.com", isPriorityCustomer: false,
                subject: "Загальне питання",
                body: "Просто хотів дізнатись про ваш сервіс більше."),
        };

        foreach (var ticket in tickets)
        {
            routingChain.Handle(ticket);
            ticket.PrintSummary();
        }

        // --- Статистика по чергах ---
        Console.WriteLine("\n=== Підсумкова статистика маршрутизації ===");
        var byQueue = tickets.GroupBy(t => t.RoutedTo);
        foreach (var group in byQueue)
            Console.WriteLine($"{group.Key}: {group.Count()} квиток(ів)");
    }
}
```

### Очікуваний вивід

```
=== Квиток #1 від ivan.petrenko@mail.com ===
Тема: "Не працює вхід в акаунт"
Мітки: keyword:не працює
Направлено до: Технічна черга
Шлях обробки:
  → SpamFilter: спаму не виявлено ✅
  → PriorityCustomer: клієнт не має пріоритетного статусу, продовжуємо
  → KeywordRouting: знайдено ключове слово "не працює" → Технічна черга 🔀

=== Квиток #2 від vip.client@bigcorp.com ===
Тема: "Питання щодо контракту"
Мітки: vip
Направлено до: Пріоритетна черга (VIP)
Шлях обробки:
  → SpamFilter: спаму не виявлено ✅
  → PriorityCustomer: клієнт має VIP-статус — направлено у пріоритетну чергу 📩

=== Квиток #3 від maria.k@example.com ===
Тема: "Не можу оплатити підписку"
Мітки: keyword:оплат
Направлено до: Черга білінгу
Шлях обробки:
  → SpamFilter: спаму не виявлено ✅
  → PriorityCustomer: клієнт не має пріоритетного статусу, продовжуємо
  → KeywordRouting: знайдено ключове слово "оплат" → Черга білінгу 🔀

=== Квиток #4 від spammer123@fake.com ===
Тема: "Ви виграш!!! Натисніть негайно натисніть"
Мітки: spam
Направлено до: Заблоковано (виявлено ознаки спаму)
Шлях обробки:
  → SpamFilter: заблоковано як спам ❌

=== Квиток #5 від novyi.user@example.com ===
Тема: "Загальне питання"
Мітки: general
Направлено до: Загальна черга
Шлях обробки:
  → SpamFilter: спаму не виявлено ✅
  → PriorityCustomer: клієнт не має пріоритетного статусу, продовжуємо
  → KeywordRouting: жодне ключове слово не збіглося, продовжуємо
  → DefaultQueue: жоден спеціалізований обробник не спрацював — загальна черга

=== Підсумкова статистика маршрутизації ===
Технічна черга: 1 квиток(ів)
Пріоритетна черга (VIP): 1 квиток(ів)
Черга білінгу: 1 квиток(ів)
Заблоковано (виявлено ознаки спаму): 1 квиток(ів)
Загальна черга: 1 квиток(ів)
```

### Чому цей приклад "реальний"

- **Ланцюжок конфігурується централізовано** (`SupportChainBuilder`), а не розкидається по коду — легко змінити порядок чи додати новий обробник.
- **Обробники не обов'язково "конкурують"** — деякі анотують і пропускають далі (`SpamFilterHandler`, коли спаму немає), інші зупиняють ланцюжок (`PriorityCustomerHandler` для VIP).
- **Кожен крок журналюється** (`ProcessingLog`) — коли щось піде не так, легко зрозуміти, через яких обробників пройшов конкретний квиток і чому він опинився саме в тій черзі.
- **Є явний термінальний обробник** (`DefaultQueueHandler`), тому жоден квиток ніколи не "губиться" мовчки.

---

## Chain of Responsibility vs Decorator vs Command

Ці три патерни часто плутають, бо всі вони так чи інакше працюють з "ланцюжками" об'єктів або обгортками. Розберемо принципову різницю.

### Chain of Responsibility vs Decorator

```
Decorator: КОЖЕН шар ЗАВЖДИ щось додає до результату

  LoggingDecorator ──▶ CachingDecorator ──▶ RealService
   (завжди логує)        (завжди кешує)       (завжди виконує)


Chain of Responsibility: кожен обробник САМ ВИРІШУЄ — обробити чи передати

  HandlerA ──?── HandlerB ──?── HandlerC
  (можливо жоден, можливо один, можливо кілька спрацюють)
```

| Ознака | Chain of Responsibility | Decorator |
|---|---|---|
| **Мета** | Знайти того, хто обробить запит | Додати поведінку до об'єкта |
| **Скільки ланок "спрацює"** | Невідомо заздалегідь — 0, 1 або кілька | **Завжди всі** ланки в ланцюжку |
| **Чи може ланка "відмовитись"** | Так — це і є суть патерну | Ні — декоратор завжди виконує свою частину |
| **Результат** | Хтось один (типово) обробляє запит | Результат накопичується через усі шари |
| **Тип зв'язку** | Логічний вибір "хто відповідальний" | Композиція поведінки |

### Chain of Responsibility vs Command

```
Command: інкапсулює запит як ОБ'ЄКТ (з методом Execute())

  ICommand command = new SaveFileCommand(document);
  command.Execute();   // клієнт не знає, ЩО саме робить команда


CoR + Command разом: обробник у ланцюжку МОЖЕ виконувати команду

  Handler.Handle(request):
      if (можу обробити)
          var command = CommandFactory.CreateFor(request);
          command.Execute();   // ← делегуємо виконання об'єкту-команді
      else
          _next.Handle(request);
```

| Ознака | Chain of Responsibility | Command |
|---|---|---|
| **Мета** | Передати запит по ланцюжку обробників | Інкапсулювати дію (і її параметри) в об'єкт |
| **Що є "головним" об'єктом** | Обробник (Handler) | Команда (Command) |
| **Підтримка undo/redo** | Не властиво патерну | Природно підтримується (зберігай історію команд) |
| **Часте поєднання** | Обробник у ланцюжку створює і виконує Command | Command може викликатись з різних джерел (меню, гарячі клавіші, макроси) |

### Запитай себе

```
Чи потрібно, щоб КОЖЕН елемент у ланцюжку щось додав до результату?
  ✅ Так → Decorator
  ❌ Ні, лише хтось один (чи кілька) мають відповідати за обробку → Chain of Responsibility

Чи потрібно зберігати запит як ОБ'ЄКТ для відкладеного виконання, черги або undo/redo?
  ✅ Так → Command (можна поєднати з CoR: обробник створює і виконує команду)
  ❌ Ні, просто потрібно знайти "відповідального" → Chain of Responsibility

Чи невідомо заздалегідь, СКІЛЬКИ обробників і В ЯКОМУ порядку спрацюють?
  ✅ Так, порядок і кількість гнучкі — Chain of Responsibility
```

---

## Переваги та недоліки

### Переваги

- **Слабке зв'язування (loose coupling)** — відправник запиту не знає, який саме обробник у результаті його опрацює.
- **Легко додавати, видаляти чи переставляти обробники** — досить змінити виклики `SetNext()`, не чіпаючи логіку самих обробників.
- **Принцип єдиної відповідальності** — кожен обробник відповідає лише за одну умову/правило, а не за весь процес прийняття рішення.
- **Принцип відкритості/закритості** — новий обробник додається як новий клас, без зміни існуючих.
- **Гнучка конфігурація для різних сценаріїв** — можна зібрати різні ланцюжки для різних середовищ (наприклад, спрощений ланцюжок для тестів, без rate-limiting).

### Недоліки

- **Немає гарантії обробки** — якщо жоден обробник не підходить і немає термінальної ланки, запит може "загубитися" мовчки.
- **Складніше налагоджувати** — щоб зрозуміти, хто саме обробив (чи не обробив) запит, потрібно простежити весь ланцюжок; без журналювання це може бути непросто.
- **Продуктивність при довгому ланцюжку** — кожен запит може проходити через багато обробників, перш ніж знайдеться "відповідальний", що додає накладні витрати.
- **Порядок обробників критично важливий** — неправильний порядок (наприклад, VIP-перевірка після маршрутизації за ключовими словами) може призвести до логічних помилок, які важко помітити одразу.

---

## Антипатерни та поширені помилки

### Помилка 1 — Забутий "кінець ланцюжка": запити губляться мовчки

Якщо жоден обробник не спрацював, а термінальної ланки немає — запит просто зникає, і про це ніхто не дізнається.

```csharp
// НЕПРАВИЛЬНО: якщо жоден обробник не підійшов — нічого не відбувається,
// і про пропущений запит НІХТО НЕ ДІЗНАЄТЬСЯ.
public abstract class BadHandler
{
    private BadHandler _next;
    public BadHandler SetNext(BadHandler next) { _next = next; return next; }

    public void Handle(SupportTicket ticket)
    {
        if (CanHandle(ticket))
            Process(ticket);
        else
            _next?.Handle(ticket); // якщо _next == null — запит просто "провалюється в нікуди"
    }

    protected abstract bool CanHandle(SupportTicket ticket);
    protected abstract void Process(SupportTicket ticket);
}
```

```csharp
// ПРАВИЛЬНО: додаємо термінальний обробник за замовчуванням,
// який гарантовано обробляє БУДЬ-ЩО, що дійшло до кінця ланцюжка.
public class DefaultTerminalHandler : SupportHandler
{
    protected override bool CanHandle(SupportTicket ticket) => true; // приймає все

    protected override void Process(SupportTicket ticket)
    {
        // Мінімум — логуємо і/або кидаємо виняток, залежно від вимог системи.
        Console.WriteLine($"[!] Заявку #{ticket.Id} не вдалося обробити жодним " +
                           "спеціалізованим обробником — потребує ручного втручання");
        // Альтернатива для критичних систем:
        // throw new InvalidOperationException($"Заявку #{ticket.Id} неможливо обробити");
    }
}

// При побудові ланцюжка ЗАВЖДИ додаємо термінальну ланку останньою:
level1.SetNext(level2).SetNext(level3).SetNext(new DefaultTerminalHandler());
```

### Помилка 2 — "Роздутий" обробник, що змішує кілька відповідальностей

Обробник, який одночасно перевіряє автентифікацію, логує, обмежує швидкість і виконує бізнес-логіку — це прихований "God Method" всередині одного класу ланцюжка.

```csharp
// НЕПРАВИЛЬНО: один обробник робить забагато — важко тестувати і змінювати окремо.
public class BloatedMiddleware : Middleware
{
    protected override void HandleRequest(RequestContext context)
    {
        // Автентифікація
        if (context.Request.ApiKey != "secret-key-123")
        {
            context.ShortCircuit(401, "Unauthorized");
            return;
        }

        // Логування
        Console.WriteLine($"→ {context.Request.Method} {context.Request.Path}");

        // Rate limiting
        // ... ще 20 рядків логіки лічильника запитів ...

        // Бізнес-логіка
        context.ResponseBody = "{ \"status\": \"ok\" }";

        // Якщо потрібно змінити ЛИШЕ правило автентифікації —
        // доведеться редагувати клас, що відповідає за ВСЕ одразу.
    }
}
```

```csharp
// ПРАВИЛЬНО: кожна відповідальність — окремий, сфокусований обробник у ланцюжку
// (як у Прикладі 2): AuthenticationMiddleware, LoggingMiddleware,
// RateLimitMiddleware, RequestHandler — кожен тестується і змінюється незалежно.
auth.SetNext(logging).SetNext(rateLimit).SetNext(handler);
```

### Помилка 3 — Побудова ланцюжка розкидана по коду вручну

Виклики `SetNext()` "де попало" (в різних методах, класах, навіть у циклах на кожен запит) роблять конфігурацію ланцюжка непрозорою і схильною до помилок при зміні порядку.

```csharp
// НЕПРАВИЛЬНО: побудова ланцюжка розкидана по різних методах і файлах.
// В одному місці:
var a = new SpamFilterHandler();
var b = new PriorityCustomerHandler();
a.SetNext(b);

// ...десь в іншому файлі, можливо через кілька місяців...
var c = new KeywordRoutingHandler();
b.SetNext(c); // легко забути, легко переплутати порядок, важко знайти всі місця

// ...і ще десь третє...
var d = new DefaultQueueHandler();
c.SetNext(d);
```

```csharp
// ПРАВИЛЬНО: побудова ланцюжка централізована в одному будівельнику/фабричному методі
// (як SupportChainBuilder у Прикладі 4). Один погляд на метод — і весь порядок зрозумілий,
// його легко змінити чи покрити unit-тестом.
public static class SupportChainBuilder
{
    public static TicketHandler BuildDefaultChain()
    {
        var spamFilter       = new SpamFilterHandler();
        var priorityCustomer = new PriorityCustomerHandler();
        var keywordRouting   = new KeywordRoutingHandler();
        var defaultQueue     = new DefaultQueueHandler();

        spamFilter.SetNext(priorityCustomer).SetNext(keywordRouting).SetNext(defaultQueue);

        return spamFilter;
    }
}
```

---

## Підсумок

**Використовуй Chain of Responsibility, коли:**

- Кілька об'єктів можуть обробити запит, і конкретний обробник заздалегідь невідомий — його треба визначити динамічно.
- Потрібно надіслати запит одному з кількох обробників **без явної вказівки**, якому саме.
- Набір обробників і їхній порядок повинні **змінюватись у рантаймі** або конфігуруватись під різні сценарії.
- Хочеться уникнути одного величезного `if-else`/`switch`, що знає про всі можливі випадки одразу.
- Потрібен конвеєр обробки (middleware) — автентифікація, логування, валідація, бізнес-логіка — де кожен крок може зупинити подальшу обробку.

**Уникай Chain of Responsibility, коли:**

- Заздалегідь точно відомо, який єдиний обробник має обробити запит — тоді простий прямий виклик буде зрозумілішим.
- Кожен елемент ланцюжка **завжди** повинен щось зробити (у такому разі це, найімовірніше, Decorator).
- Ланцюжок буде дуже довгим, а продуктивність критична — варто розглянути маршрутизацію за словником/картою замість послідовного проходу.

### Мінімальний шаблон

```csharp
// Базовий абстрактний обробник
public abstract class Handler
{
    private Handler _next;

    public Handler SetNext(Handler next)
    {
        _next = next;
        return next;
    }

    public virtual void Handle(object request)
    {
        if (_next != null)
            _next.Handle(request);
        // Якщо _next == null — це кінець ланцюжка;
        // у реальному коді тут варто мати термінальний обробник.
    }
}

// Конкретний обробник
public class ConcreteHandler : Handler
{
    public override void Handle(object request)
    {
        if (CanHandle(request))
        {
            // Обробляємо запит
            Console.WriteLine("Оброблено ConcreteHandler");
            return; // не передаємо далі — обробку завершено
        }

        // Не можемо обробити — передаємо далі по ланцюжку
        base.Handle(request);
    }

    private bool CanHandle(object request) => true; // умова обробки
}

// Використання
var handler1 = new ConcreteHandler();
var handler2 = new ConcreteHandler();
handler1.SetNext(handler2);

handler1.Handle(new object());
```

---

*Документ підготовлено для вивчення патернів проектування. Всі приклади протестовані на .NET 6+.*
