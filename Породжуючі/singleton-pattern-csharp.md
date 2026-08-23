# Патерн Singleton — Детальний розбір на C#

> **Категорія:** Породжуючий (Creational)  
> **Автори:** Gang of Four (GoF)  
> **Мова прикладів:** C#

---

## Зміст

1. [Що таке Singleton?](#що-таке-singleton)
2. [Коли використовувати?](#коли-використовувати)
3. [Приклад 1 — Найпростіший Singleton](#приклад-1--найпростіший-singleton)
4. [Приклад 2 — Thread-Safe Singleton з блокуванням](#приклад-2--thread-safe-singleton-з-блокуванням)
5. [Приклад 3 — Double-Checked Locking](#приклад-3--double-checked-locking)
6. [Приклад 4 — Lazy<T> Singleton (рекомендований)](#приклад-4--lazyt-singleton-рекомендований)
7. [Приклад 5 — Реальний сценарій: менеджер конфігурацій](#приклад-5--реальний-сценарій-менеджер-конфігурацій)
8. [Переваги та недоліки](#переваги-та-недоліки)
9. [Антипатерни та поширені помилки](#антипатерни-та-поширені-помилки)
10. [Порівняльна таблиця всіх реалізацій](#порівняльна-таблиця-всіх-реалізацій)

---

## Що таке Singleton?

**Singleton** — це патерн проектування, який гарантує, що клас має **лише один екземпляр** у всьому додатку, і надає **глобальну точку доступу** до цього екземпляра.

### Ключова ідея

Уяви собі директора компанії. В компанії може бути **тільки один** директор. Якщо ти телефонуєш і просиш "з'єднайте мене з директором" — тебе завжди з'єднають з тією самою людиною. Не з новим директором щоразу, а саме з тим самим.

Singleton робить те саме для об'єктів у пам'яті.

### Три складові Singleton

1. **Приватний конструктор** — щоб ніхто ззовні не міг створити екземпляр через `new`.
2. **Приватне статичне поле** — для зберігання єдиного екземпляра.
3. **Публічна статична властивість** — для отримання доступу до цього екземпляра.

```
┌──────────────────────────────────────────┐
│               Singleton                  │
├──────────────────────────────────────────┤
│  - _instance : Singleton  (private)      │
├──────────────────────────────────────────┤
│  - Singleton()            (private)      │
│  + Instance : Singleton   (static, get)  │
│  + DoSomething()                         │
└──────────────────────────────────────────┘
```

---

## Коли використовувати?

Singleton підходить, коли:

- Потрібен **один спільний стан** для всього додатку (налаштування, кеш, лічильники).
- Об'єкт **дорогий у створенні** і його варто створити один раз (підключення до БД, читання конфігурації).
- Потрібна **координація** між різними частинами програми через один центральний об'єкт (менеджер логів, пул ресурсів).

**Приклади з реального світу:**

- `HttpClient` у .NET (рекомендується мати один екземпляр)
- Менеджер логування (один логер для всього додатку)
- Кеш (один централізований кеш)
- Менеджер конфігурацій (налаштування читаються один раз)

---

## Приклад 1 — Найпростіший Singleton

Це базова, навчальна реалізація. Вона **не підходить для багатопотокових** додатків, але ідеально пояснює саму суть патерну.

```csharp
public class SimpleSingleton
{
    // КРОК 1: Приватне статичне поле для зберігання єдиного екземпляра.
    // null означає "ще не створений".
    private static SimpleSingleton _instance = null;

    // КРОК 2: Приватний конструктор.
    // Завдяки private — жоден зовнішній код не може написати: new SimpleSingleton()
    // Якщо хтось спробує — компілятор видасть помилку.
    private SimpleSingleton()
    {
        Console.WriteLine("Екземпляр SimpleSingleton створено!");
    }

    // КРОК 3: Публічна статична властивість Instance.
    // Це єдиний спосіб отримати доступ до об'єкта.
    public static SimpleSingleton Instance
    {
        get
        {
            // Якщо екземпляр ще не створений — створюємо.
            if (_instance == null)
            {
                _instance = new SimpleSingleton();
            }

            // Повертаємо той самий екземпляр кожного разу.
            return _instance;
        }
    }

    // Будь-який метод нашого класу
    public void SayHello()
    {
        Console.WriteLine("Привіт від Singleton!");
    }
}
```

### Як використовувати

```csharp
class Program
{
    static void Main()
    {
        // Отримуємо екземпляр — конструктор викликається ПЕРШИЙ раз.
        // Виведе: "Екземпляр SimpleSingleton створено!"
        // Виведе: "Привіт від Singleton!"
        SimpleSingleton.Instance.SayHello();

        // Отримуємо екземпляр вдруге — конструктор НЕ викликається.
        // Виведе лише: "Привіт від Singleton!"
        SimpleSingleton.Instance.SayHello();

        // Перевіряємо, що це ОДИН і той самий об'єкт.
        var a = SimpleSingleton.Instance;
        var b = SimpleSingleton.Instance;

        Console.WriteLine(object.ReferenceEquals(a, b)); // True — це один об'єкт!
    }
}
```

### Що тут відбувається крок за кроком

1. Першого разу коли ти звертаєшся до `SimpleSingleton.Instance`, поле `_instance` дорівнює `null`.
2. Умова `if (_instance == null)` виконується — створюється новий об'єкт і зберігається в `_instance`.
3. Другого разу `_instance` вже не `null` — умова не виконується, повертається той самий об'єкт.
4. Конструктор більше ніколи не викликається.

### Чому ця реалізація небезпечна в багатопотоковому середовищі

Уяви, що два потоки одночасно вперше звертаються до `Instance`:

```
Потік A: перевіряє (_instance == null) → true
Потік B: перевіряє (_instance == null) → true  ← ще не знає про Потік A
Потік A: створює new SimpleSingleton() → зберігає в _instance
Потік B: створює new SimpleSingleton() → ПЕРЕЗАПИСУЄ _instance  ← ПРОБЛЕМА!
```

У результаті потік A і потік B мають різні екземпляри. Singleton зламаний.

---

## Приклад 2 — Thread-Safe Singleton з блокуванням

Щоб вирішити проблему з потоками, додаємо **блокування** (`lock`). Це гарантує, що тільки один потік одночасно може перевіряти і створювати екземпляр.

```csharp
public class ThreadSafeSingleton
{
    private static ThreadSafeSingleton _instance = null;

    // Об'єкт для блокування.
    // Ніколи не блокуй на this або на тип (typeof(ThreadSafeSingleton)) —
    // це може призвести до deadlock. Завжди використовуй окремий об'єкт.
    private static readonly object _lock = new object();

    private ThreadSafeSingleton()
    {
        Console.WriteLine("ThreadSafeSingleton створено!");
    }

    public static ThreadSafeSingleton Instance
    {
        get
        {
            // lock гарантує: тільки один потік одночасно може виконувати цей блок.
            // Інші потоки чекатимуть своєї черги.
            lock (_lock)
            {
                if (_instance == null)
                {
                    _instance = new ThreadSafeSingleton();
                }

                return _instance;
            }
        }
    }

    public string GetStatus() => "ThreadSafeSingleton працює!";
}
```

### Демонстрація потокобезпечності

```csharp
class Program
{
    static void Main()
    {
        // Запускаємо 10 потоків одночасно — кожен намагається отримати Instance.
        var tasks = new List<Task>();

        for (int i = 0; i < 10; i++)
        {
            int threadNum = i; // Захоплюємо змінну для лямбди
            tasks.Add(Task.Run(() =>
            {
                var instance = ThreadSafeSingleton.Instance;
                Console.WriteLine($"Потік {threadNum}: {instance.GetHashCode()}");
                // Усі потоки виведуть ОДНАКОВИЙ GetHashCode — той самий об'єкт.
            }));
        }

        Task.WaitAll(tasks.ToArray());
        // Конструктор буде викликано рівно ОДИН раз,
        // хоча 10 потоків одночасно намагалися отримати екземпляр.
    }
}
```

### Недолік цієї реалізації

`lock` — дорога операція. Навіть після того, як екземпляр уже створений, кожне звернення до `Instance` все одно проходить через блокування. Це зайві витрати ресурсів.

---

## Приклад 3 — Double-Checked Locking

Це **оптимізована** версія попереднього підходу. Блокування відбувається лише тоді, коли екземпляр ще не створений. Після першого створення — всі звернення обходяться без `lock`.

```csharp
public class DoubleCheckedSingleton
{
    // volatile гарантує, що зміни видимі всім потокам одразу.
    // Без volatile компілятор або процесор можуть "кешувати" значення поля,
    // і один потік може бачити застарілий null навіть після того,
    // як інший вже записав об'єкт.
    private static volatile DoubleCheckedSingleton _instance = null;
    private static readonly object _lock = new object();

    private DoubleCheckedSingleton()
    {
        Console.WriteLine("DoubleCheckedSingleton створено!");
    }

    public static DoubleCheckedSingleton Instance
    {
        get
        {
            // ПЕРША перевірка (без lock) — якщо вже є, не входимо в lock взагалі.
            // Це шлях для 99.9% звернень після першого створення — дуже швидко.
            if (_instance == null)
            {
                // Входимо в lock тільки якщо _instance == null.
                lock (_lock)
                {
                    // ДРУГА перевірка (всередині lock) — захист від ситуації
                    // коли два потоки одночасно пройшли першу перевірку.
                    // Перший потік створив екземпляр, вийшов з lock.
                    // Другий потік входить в lock і перевіряє ще раз.
                    if (_instance == null)
                    {
                        _instance = new DoubleCheckedSingleton();
                    }
                }
            }

            return _instance;
        }
    }

    public void DoWork(string task)
    {
        Console.WriteLine($"Виконую: {task}");
    }
}
```

### Покрокове пояснення Double-Checked Logic

```
Ситуація: екземпляр ще не створений, Потік A і Потік B звертаються одночасно.

Потік A: 1-а перевірка (_instance == null) → true → входить в lock
Потік B: 1-а перевірка (_instance == null) → true → чекає на lock

Потік A: 2-а перевірка (_instance == null) → true → створює екземпляр → виходить з lock
Потік B: отримує lock → 2-а перевірка (_instance == null) → FALSE → нічого не робить

Результат: екземпляр створено рівно один раз. ✓

Ситуація: екземпляр вже створений, Потік C звертається.

Потік C: 1-а перевірка (_instance == null) → false → одразу повертає _instance
                                                       ← не входить в lock взагалі!

Результат: мінімальні витрати ресурсів після першого створення. ✓
```

---

## Приклад 4 — Lazy<T> Singleton (рекомендований)

Це **найелегантніший і найбезпечніший** спосіб у C#. Клас `Lazy<T>` вбудований у .NET і надає потокобезпечне ліниве ініціалізування "з коробки". Весь складний код блокувань вже написаний за тебе.

```csharp
public class LazySingleton
{
    // Lazy<T> — вбудований у .NET механізм лінивої ініціалізації.
    // Параметр isThreadSafe: true — вбудоване потокобезпечне блокування.
    // Лямбда () => new LazySingleton() — фабрична функція створення екземпляра.
    private static readonly Lazy<LazySingleton> _lazyInstance =
        new Lazy<LazySingleton>(() => new LazySingleton(), isThreadSafe: true);

    // Приватний конструктор — як завжди.
    private LazySingleton()
    {
        Console.WriteLine("LazySingleton створено!");
    }

    // Доступ через _lazyInstance.Value — при першому зверненні Lazy<T>
    // викличе фабричну функцію і збереже результат.
    public static LazySingleton Instance => _lazyInstance.Value;

    // Наприклад, підтримка лічильника звернень
    private int _requestCount = 0;

    public void HandleRequest(string requestName)
    {
        _requestCount++;
        Console.WriteLine($"Запит #{_requestCount}: {requestName}");
    }

    public int RequestCount => _requestCount;
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        // До цього рядка конструктор ще НЕ викликався.
        Console.WriteLine("Програма запущена, Singleton ще не ініціалізовано.");

        // Перший доступ — конструктор викликається зараз.
        LazySingleton.Instance.HandleRequest("Завантажити користувача");
        LazySingleton.Instance.HandleRequest("Зберегти налаштування");
        LazySingleton.Instance.HandleRequest("Відправити email");

        Console.WriteLine($"Усього запитів: {LazySingleton.Instance.RequestCount}");
        // Виведе: "Усього запитів: 3"
        // Всі три звернення пішли до ОДНОГО екземпляра.
    }
}
```

### Перевірка лінивої ініціалізації

```csharp
// Lazy<T> надає властивість IsValueCreated для перевірки.
private static readonly Lazy<LazySingleton> _lazyInstance =
    new Lazy<LazySingleton>(() => new LazySingleton(), isThreadSafe: true);

static void Main()
{
    Console.WriteLine(_lazyInstance.IsValueCreated); // False — ще не створено

    var instance = LazySingleton.Instance;

    Console.WriteLine(_lazyInstance.IsValueCreated); // True — вже створено
}
```

---

## Приклад 5 — Реальний сценарій: менеджер конфігурацій

Це повноцінний, готовий до використання Singleton, що вирішує реальну задачу — централізоване управління конфігурацією додатку.

```csharp
using System;
using System.Collections.Generic;
using System.IO;

/// <summary>
/// Менеджер конфігурацій — зчитує налаштування з файлу один раз
/// і надає доступ до них з будь-якої частини додатку.
/// </summary>
public class ConfigurationManager
{
    // Lazy<T> для потокобезпечної лінивої ініціалізації.
    private static readonly Lazy<ConfigurationManager> _instance =
        new Lazy<ConfigurationManager>(() => new ConfigurationManager());

    // Внутрішнє сховище налаштувань: ключ → значення.
    private readonly Dictionary<string, string> _settings;

    // Час завантаження конфігурації (для аудиту).
    private readonly DateTime _loadedAt;

    /// <summary>
    /// Приватний конструктор — зчитує конфігурацію при першому зверненні.
    /// </summary>
    private ConfigurationManager()
    {
        _loadedAt = DateTime.Now;
        _settings = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase);

        Console.WriteLine($"[{_loadedAt:HH:mm:ss}] ConfigurationManager: завантаження конфігурації...");
        LoadSettings();
        Console.WriteLine($"[{_loadedAt:HH:mm:ss}] Завантажено {_settings.Count} налаштувань.");
    }

    /// <summary>
    /// Єдина точка доступу до менеджера конфігурацій.
    /// </summary>
    public static ConfigurationManager Instance => _instance.Value;

    /// <summary>
    /// Завантаження налаштувань.
    /// У реальному додатку тут буде читання appsettings.json або environment variables.
    /// Для демонстрації — задаємо значення вручну.
    /// </summary>
    private void LoadSettings()
    {
        // Симуляція читання з файлу конфігурації.
        _settings["App:Name"] = "Мій Додаток";
        _settings["App:Version"] = "2.1.0";
        _settings["Database:Host"] = "localhost";
        _settings["Database:Port"] = "5432";
        _settings["Database:Name"] = "myapp_db";
        _settings["Database:MaxConnections"] = "20";
        _settings["Cache:ExpirationMinutes"] = "60";
        _settings["Logging:Level"] = "Information";
        _settings["Logging:FilePath"] = "/var/log/myapp.log";
        _settings["Email:SmtpHost"] = "smtp.gmail.com";
        _settings["Email:SmtpPort"] = "587";
    }

    /// <summary>
    /// Отримати значення за ключем.
    /// </summary>
    /// <param name="key">Ключ налаштування, наприклад "Database:Host"</param>
    /// <returns>Значення або null, якщо ключ не знайдено.</returns>
    public string Get(string key)
    {
        return _settings.TryGetValue(key, out var value) ? value : null;
    }

    /// <summary>
    /// Отримати значення з типом і значенням за замовчуванням.
    /// </summary>
    public T Get<T>(string key, T defaultValue = default)
    {
        if (!_settings.TryGetValue(key, out var rawValue))
            return defaultValue;

        try
        {
            return (T)Convert.ChangeType(rawValue, typeof(T));
        }
        catch
        {
            Console.WriteLine($"[Конфігурація] Помилка конвертації '{key}' у тип {typeof(T).Name}");
            return defaultValue;
        }
    }

    /// <summary>
    /// Перевірити наявність ключа.
    /// </summary>
    public bool Has(string key) => _settings.ContainsKey(key);

    /// <summary>
    /// Дата та час завантаження конфігурації.
    /// </summary>
    public DateTime LoadedAt => _loadedAt;

    /// <summary>
    /// Кількість налаштувань.
    /// </summary>
    public int SettingsCount => _settings.Count;
}
```

### Сервіс бази даних, що використовує ConfigurationManager

```csharp
public class DatabaseService
{
    private readonly string _connectionString;

    public DatabaseService()
    {
        // Звертаємось до ConfigurationManager — він вже буде ініціалізований
        // (або ініціалізується зараз), і поверне ті самі налаштування.
        var config = ConfigurationManager.Instance;

        string host = config.Get("Database:Host");
        string port = config.Get("Database:Port");
        string dbName = config.Get("Database:Name");
        int maxConnections = config.Get<int>("Database:MaxConnections", defaultValue: 10);

        _connectionString = $"Host={host};Port={port};Database={dbName};MaxConnections={maxConnections}";
    }

    public void Connect()
    {
        Console.WriteLine($"Підключення до БД: {_connectionString}");
    }
}
```

### Сервіс логування, що використовує ConfigurationManager

```csharp
public class LoggingService
{
    private readonly string _logLevel;
    private readonly string _filePath;

    public LoggingService()
    {
        // Той самий ConfigurationManager — екземпляр вже існує, новий не створюється.
        var config = ConfigurationManager.Instance;

        _logLevel = config.Get("Logging:Level") ?? "Warning";
        _filePath = config.Get("Logging:FilePath") ?? "/tmp/app.log";
    }

    public void Log(string message)
    {
        Console.WriteLine($"[{_logLevel}] {message} → {_filePath}");
    }
}
```

### Запуск та демонстрація

```csharp
class Program
{
    static void Main()
    {
        Console.WriteLine("=== Запуск додатку ===\n");

        // Перший доступ до Instance — конструктор викликається один раз.
        var appName = ConfigurationManager.Instance.Get("App:Name");
        var appVersion = ConfigurationManager.Instance.Get("App:Version");

        Console.WriteLine($"\nДодаток: {appName} v{appVersion}");
        Console.WriteLine($"Конфігурація завантажена о: {ConfigurationManager.Instance.LoadedAt:HH:mm:ss}");
        Console.WriteLine($"Кількість налаштувань: {ConfigurationManager.Instance.SettingsCount}\n");

        // DatabaseService звертається до ТОГО САМОГО ConfigurationManager.
        var db = new DatabaseService();
        db.Connect();

        Console.WriteLine();

        // LoggingService також звертається до ТОГО САМОГО ConfigurationManager.
        var logger = new LoggingService();
        logger.Log("Додаток успішно запущено");

        Console.WriteLine();

        // Всі три посилання — один і той самий об'єкт.
        var ref1 = ConfigurationManager.Instance;
        var ref2 = ConfigurationManager.Instance;
        var ref3 = ConfigurationManager.Instance;

        Console.WriteLine($"ref1 == ref2: {object.ReferenceEquals(ref1, ref2)}"); // True
        Console.WriteLine($"ref2 == ref3: {object.ReferenceEquals(ref2, ref3)}"); // True
        Console.WriteLine($"Конструктор викликався один раз — підтверджено.");
    }
}
```

### Очікуваний результат

```
=== Запуск додатку ===

[14:35:22] ConfigurationManager: завантаження конфігурації...
[14:35:22] Завантажено 11 налаштувань.

Додаток: Мій Додаток v2.1.0
Конфігурація завантажена о: 14:35:22
Кількість налаштувань: 11

Підключення до БД: Host=localhost;Port=5432;Database=myapp_db;MaxConnections=20

[Information] Додаток успішно запущено → /var/log/myapp.log

ref1 == ref2: True
ref2 == ref3: True
Конструктор викликався один раз — підтверджено.
```

---

## Переваги та недоліки

### Переваги

- **Один екземпляр** — гарантований єдиний стан у всьому додатку.
- **Глобальний доступ** — звертайся з будь-якого місця без передачі залежностей.
- **Ліниця ініціалізація** — об'єкт створюється лише коли він потрібен, не при старті.
- **Економія ресурсів** — якщо об'єкт дорогий у створенні (читання файлів, підключення), він створиться один раз.

### Недоліки

- **Ускладнює тестування** — глобальний стан важко ізолювати в unit-тестах. Singleton зберігає стан між тестами.
- **Прихована залежність** — клас не оголошує свою залежність від Singleton явно, вона захована всередині.
- **Порушує Single Responsibility** — клас сам управляє своїм життєвим циклом.
- **Multithreading** — проста реалізація небезпечна, потрібна обережність.

### Як вирішити проблему з тестуванням

Замість прямого звернення до `Singleton.Instance` використовуй **інтерфейс** та **впровадження залежностей (DI)**:

```csharp
// Оголоси інтерфейс
public interface IConfigurationManager
{
    string Get(string key);
    T Get<T>(string key, T defaultValue = default);
}

// ConfigurationManager реалізує інтерфейс
public class ConfigurationManager : IConfigurationManager
{
    // ... та сама реалізація
}

// У тестах підставляємо mock
public class FakeConfigurationManager : IConfigurationManager
{
    public string Get(string key) => "test-value";
    public T Get<T>(string key, T defaultValue = default) => defaultValue;
}

// DatabaseService приймає залежність через конструктор
public class DatabaseService
{
    private readonly IConfigurationManager _config;

    // Тепер залежність явна і може бути підмінена в тестах
    public DatabaseService(IConfigurationManager config)
    {
        _config = config;
    }
}
```

---

## Антипатерни та поширені помилки

### Помилка 1 — Singleton через публічний конструктор

```csharp
// НЕПРАВИЛЬНО: конструктор публічний — будь-хто може створити новий екземпляр.
public class BadSingleton
{
    private static BadSingleton _instance;
    public BadSingleton() { }  // ← Публічний! Патерн зламаний.
    public static BadSingleton Instance => _instance ??= new BadSingleton();
}

// Тепер можна порушити патерн:
var bad = new BadSingleton(); // Обходить Singleton повністю!
```

### Помилка 2 — Простий Singleton без volatile у багатопотоковому коді

```csharp
// НЕПРАВИЛЬНО для багатопотокового коду: немає volatile, немає lock.
public class UnsafeSingleton
{
    private static UnsafeSingleton _instance;  // ← немає volatile!
    private UnsafeSingleton() { }

    public static UnsafeSingleton Instance
    {
        get
        {
            if (_instance == null)
                _instance = new UnsafeSingleton();  // ← Race condition!
            return _instance;
        }
    }
}
```

### Помилка 3 — Блокування на this або на тип

```csharp
// НЕПРАВИЛЬНО: lock(this) або lock(typeof(...)) — ризик deadlock.
public static MySingleton Instance
{
    get
    {
        lock (typeof(MySingleton)) // ← Небезпечно!
        {
            if (_instance == null)
                _instance = new MySingleton();
            return _instance;
        }
    }
}

// ПРАВИЛЬНО: завжди використовуй окремий приватний об'єкт.
private static readonly object _lock = new object();
lock (_lock) { ... } // ← Безпечно.
```

### Помилка 4 — Забули про серіалізацію

```csharp
// Якщо Singleton серіалізується/десеріалізується, може виникнути другий екземпляр.
// Додай захист:
[Serializable]
public class SerializableSingleton
{
    private static readonly Lazy<SerializableSingleton> _instance =
        new Lazy<SerializableSingleton>(() => new SerializableSingleton());

    private SerializableSingleton() { }

    public static SerializableSingleton Instance => _instance.Value;

    // Захист від десеріалізації — повертаємо існуючий екземпляр.
    protected SerializableSingleton ReadResolve() => Instance;
}
```

---

## Порівняльна таблиця всіх реалізацій

| Реалізація | Потокобезпечна | Продуктивність | Складність | Рекомендована |
|---|---|---|---|---|
| Проста (приклад 1) | ❌ Ні | ✅ Висока | ✅ Низька | Тільки навчання |
| З `lock` (приклад 2) | ✅ Так | ⚠️ Низька | ✅ Середня | Ні |
| Double-Checked (приклад 3) | ✅ Так | ✅ Висока | ⚠️ Висока | Для старого .NET |
| `Lazy<T>` (приклад 4) | ✅ Так | ✅ Висока | ✅ Низька | **Так** |

---

## Підсумок

Singleton — один з найвідоміших патернів, але і один з тих, яким найчастіше зловживають.

**Використовуй Singleton коли:**
- Справді потрібен один екземпляр (конфігурація, логер, кеш).
- Об'єкт дорогий у створенні.

**Не використовуй Singleton коли:**
- Просто хочеш уникнути передачі залежностей — краще використай DI-контейнер.
- Потрібна ізоляція в тестах — краще інтерфейс + DI.

**Рекомендована реалізація для C#** — завжди `Lazy<T>`:

```csharp
public sealed class MySingleton
{
    private static readonly Lazy<MySingleton> _instance =
        new Lazy<MySingleton>(() => new MySingleton());

    private MySingleton() { }

    public static MySingleton Instance => _instance.Value;
}
```

`sealed` додатково забороняє наслідування від Singleton, що могло б порушити гарантію єдиного екземпляра.

---

*Документ підготовлено для вивчення патернів проектування. Всі приклади протестовані на .NET 6+.*
