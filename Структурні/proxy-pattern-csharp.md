# Патерн Proxy (Замісник) у C#

> **Категорія:** Структурний патерн (Structural Pattern)  
> **Мова прикладів:** C# (.NET)

---

## Зміст

1. [Що таке Proxy?](#що-таке-proxy)
2. [Різновиди Proxy](#різновиди-proxy)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Простий (Virtual Proxy: ліниве завантаження зображення)](#приклад-1--простий-virtual-proxy-ліниве-завантаження-зображення)
5. [Приклад 2 — Просунутий (захист + кешування + логування)](#приклад-2--просунутий-захист--кешування--логування)
6. [Dynamic Proxy через DispatchProxy](#dynamic-proxy-через-dispatchproxy)
7. [Proxy vs Decorator vs Adapter](#proxy-vs-decorator-vs-adapter)
8. [Переваги та недоліки](#переваги-та-недоліки)
9. [Підсумок](#підсумок)

---

## Що таке Proxy?

**Proxy** — це патерн, який надає **замісний об'єкт** замість реального. Проксі контролює доступ до реального об'єкта і може виконувати дії **до або після** звернення до нього.

Аналогія: кредитна картка — це проксі для вашого банківського рахунку. Вона має той самий "інтерфейс" (можна нею платити), але між вами і грошима стоїть банк, який перевіряє баланс, логує транзакції та блокує підозрілі операції.

### Проблема без Proxy

```csharp
// Важкий об'єкт — завантажує великий файл у конструкторі
public class HeavyReport
{
    private readonly byte[] _data;

    public HeavyReport(string filePath)
    {
        Console.WriteLine($"Завантажуємо файл {filePath}...");
        _data = File.ReadAllBytes(filePath); // займає 3 секунди!
    }

    public void Display() => Console.WriteLine($"Звіт: {_data.Length} байт");
}

// Проблема: об'єкт завантажується одразу при створенні,
// навіть якщо Display() ніколи не буде викликаний
var report = new HeavyReport("report.pdf"); // 3 секунди — обов'язково
// ... можливо Display() взагалі не буде викликаний
```

### Рішення з Proxy

```csharp
// Проксі виглядає і поводиться як HeavyReport,
// але завантажує дані тільки при першому зверненні
var report = new HeavyReportProxy("report.pdf"); // миттєво!
// ...
report.Display(); // тільки тут — якщо взагалі дійде до цього рядка
```

---

## Різновиди Proxy

Proxy — один з найбільш різноманітних патернів. Залежно від задачі розрізняють кілька підтипів:

| Тип | Мета |
|---|---|
| **Virtual Proxy** | Відкладає створення важкого об'єкта до першого використання (lazy initialization) |
| **Protection Proxy** | Контролює доступ на основі прав — перевіряє дозволи перед делегуванням |
| **Remote Proxy** | Представляє об'єкт, що фізично знаходиться в іншому місці (мережа, інший процес) |
| **Caching Proxy** | Кешує результати дорогих операцій і повертає їх при повторних запитах |
| **Logging Proxy** | Логує всі виклики методів без зміни реального об'єкта |
| **Smart Proxy** | Додає додаткові дії: підрахунок посилань, блокування, валідацію тощо |

На практиці один проксі часто поєднує кілька типів одночасно.

---

## Структура патерну

```
┌──────────────┐        ┌─────────────────────┐
│   Client     │───────▶│  «interface»        │
│              │        │  ISubject           │
└──────────────┘        │  + Request()        │
                        └──────────┬──────────┘
                                   │ implements
                        ┌──────────┴──────────┐
                        │                     │
               ┌────────▼────────┐  ┌─────────▼───────┐
               │  RealSubject    │  │  Proxy          │
               │                 │  │  - _real        │◀─ може створити
               │  + Request()    │  │  + Request()    │   або отримати
               └─────────────────┘  │    → перевірка  │   ззовні
                                    │    → _real.Req()│
                                    │    → постдії    │
                                    └─────────────────┘
```

Учасники:
- **ISubject** — спільний інтерфейс для реального об'єкта і проксі
- **RealSubject** — реальний об'єкт, до якого проксі делегує виклики
- **Proxy** — зберігає посилання на `RealSubject`, реалізує `ISubject`, контролює доступ
- **Client** — працює тільки через `ISubject`, не знає — реальний об'єкт чи проксі

---

## Приклад 1 — Простий (Virtual Proxy: ліниве завантаження зображення)

Класичний Virtual Proxy: зображення завантажується з диска тільки тоді, коли його реально треба відобразити.

### Крок 1: Спільний інтерфейс

```csharp
// Інтерфейс, який реалізують і реальне зображення, і проксі
public interface IImage
{
    string FileName { get; }
    int Width { get; }
    int Height { get; }
    void Render(int x, int y);
}
```

### Крок 2: Реальний об'єкт — важке зображення

```csharp
// Реальне зображення — завантажує файл у конструкторі (дорога операція)
public class HighResolutionImage : IImage
{
    private readonly byte[] _pixelData;

    public string FileName { get; }
    public int Width { get; }
    public int Height { get; }

    public HighResolutionImage(string fileName)
    {
        FileName = fileName;
        Console.WriteLine($"[Image] Завантажуємо '{fileName}' з диска...");

        // Симулюємо завантаження великого файлу
        Thread.Sleep(500);
        _pixelData = new byte[1920 * 1080 * 4]; // ~8MB для Full HD
        Width  = 1920;
        Height = 1080;

        Console.WriteLine($"[Image] '{fileName}' завантажено ({_pixelData.Length / 1024 / 1024}MB)");
    }

    public void Render(int x, int y)
    {
        Console.WriteLine($"[Image] Відображаємо '{FileName}' ({Width}x{Height}) на позиції ({x},{y})");
    }
}
```

### Крок 3: Virtual Proxy

```csharp
// Проксі — легкий замісник, створює реальний об'єкт тільки при першому Render()
public class ImageProxy : IImage
{
    private readonly string _fileName;
    private HighResolutionImage _realImage; // null до першого використання

    // Проксі знає розміри без завантаження файлу
    // (наприклад, з метаданих або конфігурації)
    private readonly int _width;
    private readonly int _height;

    public string FileName => _fileName;
    public int Width  => _width;
    public int Height => _height;

    public ImageProxy(string fileName, int width = 1920, int height = 1080)
    {
        _fileName = fileName;
        _width    = width;
        _height   = height;
        // Реальний об'єкт НЕ створюється тут
        Console.WriteLine($"[Proxy] Створено проксі для '{fileName}' (файл ще не завантажено)");
    }

    public void Render(int x, int y)
    {
        // Lazy initialization: створюємо реальний об'єкт тільки при першому виклику
        if (_realImage == null)
        {
            Console.WriteLine($"[Proxy] Перший Render — завантажуємо реальний об'єкт...");
            _realImage = new HighResolutionImage(_fileName);
        }
        else
        {
            Console.WriteLine($"[Proxy] Реальний об'єкт вже в пам'яті — використовуємо напряму");
        }

        _realImage.Render(x, y);
    }
}
```

### Крок 4: Використання

```csharp
class Program
{
    static void Main()
    {
        Console.WriteLine("=== Завантаження галереї (без реальних зображень) ===");

        // Завантажуємо галерею — створюємо лише проксі, швидко!
        IImage[] gallery =
        {
            new ImageProxy("hero-banner.jpg"),
            new ImageProxy("product-shot.jpg"),
            new ImageProxy("team-photo.jpg"),
        };

        Console.WriteLine($"\nГалерея готова: {gallery.Length} зображень\n");

        // Симулюємо: користувач прокрутив до другого зображення
        Console.WriteLine("=== Користувач бачить друге зображення ===");
        gallery[1].Render(0, 200); // тільки тут завантажується файл

        Console.WriteLine();
        Console.WriteLine("=== Повторне відображення (файл вже в пам'яті) ===");
        gallery[1].Render(0, 200); // миттєво — вже завантажено

        Console.WriteLine();
        Console.WriteLine("=== Перше зображення так і не завантажилось ===");
        Console.WriteLine($"gallery[0].FileName = {gallery[0].FileName}"); // OK без завантаження
        Console.WriteLine($"gallery[0].Width = {gallery[0].Width}");       // OK без завантаження
    }
}
```

### Очікуваний вивід

```
=== Завантаження галереї (без реальних зображень) ===
[Proxy] Створено проксі для 'hero-banner.jpg' (файл ще не завантажено)
[Proxy] Створено проксі для 'product-shot.jpg' (файл ще не завантажено)
[Proxy] Створено проксі для 'team-photo.jpg' (файл ще не завантажено)

Галерея готова: 3 зображень

=== Користувач бачить друге зображення ===
[Proxy] Перший Render — завантажуємо реальний об'єкт...
[Image] Завантажуємо 'product-shot.jpg' з диска...
[Image] 'product-shot.jpg' завантажено (8MB)
[Image] Відображаємо 'product-shot.jpg' (1920x1080) на позиції (0,200)

=== Повторне відображення (файл вже в пам'яті) ===
[Proxy] Реальний об'єкт вже в пам'яті — використовуємо напряму
[Image] Відображаємо 'product-shot.jpg' (1920x1080) на позиції (0,200)

=== Перше зображення так і не завантажилось ===
gallery[0].FileName = hero-banner.jpg
gallery[0].Width = 1920
```

---

## Приклад 2 — Просунутий (захист + кешування + логування)

Реалістичний сценарій: репозиторій даних з трьома шарами проксі — захист доступу, кешування і аудит-лог.

### Крок 1: Доменна модель і інтерфейс репозиторію

```csharp
public class Article
{
    public int Id { get; init; }
    public string Title { get; init; }
    public string Content { get; init; }
    public string AuthorId { get; init; }
    public bool IsPublished { get; init; }
    public DateTime CreatedAt { get; init; }

    public override string ToString() => $"[{Id}] \"{Title}\" by {AuthorId}";
}

// Контракт репозиторію
public interface IArticleRepository
{
    Article GetById(int id);
    IReadOnlyList<Article> GetAll();
    void Save(Article article);
    void Delete(int id);
}
```

### Крок 2: Реальний репозиторій (важкий — звертається до БД)

```csharp
public class ArticleRepository : IArticleRepository
{
    // Симульована "база даних"
    private readonly Dictionary<int, Article> _db = new()
    {
        [1] = new Article { Id = 1, Title = "Патерн Proxy", Content = "...", AuthorId = "alice", IsPublished = true,  CreatedAt = DateTime.Now.AddDays(-5) },
        [2] = new Article { Id = 2, Title = "Чорновик",     Content = "...", AuthorId = "bob",   IsPublished = false, CreatedAt = DateTime.Now.AddDays(-1) },
        [3] = new Article { Id = 3, Title = "SOLID",        Content = "...", AuthorId = "alice", IsPublished = true,  CreatedAt = DateTime.Now.AddDays(-10) },
    };

    public Article GetById(int id)
    {
        Console.WriteLine($"  [DB] SELECT * FROM articles WHERE id = {id}");
        Thread.Sleep(50); // затримка БД
        return _db.TryGetValue(id, out var article) ? article : null;
    }

    public IReadOnlyList<Article> GetAll()
    {
        Console.WriteLine("  [DB] SELECT * FROM articles");
        Thread.Sleep(80);
        return _db.Values.ToList();
    }

    public void Save(Article article)
    {
        Console.WriteLine($"  [DB] INSERT/UPDATE article id={article.Id}");
        _db[article.Id] = article;
    }

    public void Delete(int id)
    {
        Console.WriteLine($"  [DB] DELETE FROM articles WHERE id = {id}");
        _db.Remove(id);
    }
}
```

### Крок 3: Контекст поточного користувача

```csharp
// Роль користувача в системі
public enum UserRole { Guest, Reader, Author, Admin }

// Поточний користувач — передається в Protection Proxy
public class UserContext
{
    public string UserId { get; init; }
    public UserRole Role { get; init; }

    public bool HasRole(UserRole minimumRole) => Role >= minimumRole;

    public override string ToString() => $"{UserId} ({Role})";
}
```

### Крок 4: Protection Proxy — перевірка прав доступу

```csharp
// Контролює доступ: хто що може робити
public class ProtectionProxy : IArticleRepository
{
    private readonly IArticleRepository _inner;
    private readonly UserContext _user;

    public ProtectionProxy(IArticleRepository inner, UserContext user)
    {
        _inner = inner;
        _user  = user;
    }

    public Article GetById(int id)
    {
        var article = _inner.GetById(id);

        // Гості і читачі бачать тільки опубліковані статті
        if (article != null && !article.IsPublished && !_user.HasRole(UserRole.Author))
        {
            Console.WriteLine($"  [PROTECTION] Доступ заборонено: стаття {id} не опублікована. Користувач: {_user}");
            return null;
        }

        return article;
    }

    public IReadOnlyList<Article> GetAll()
    {
        var articles = _inner.GetAll();

        // Фільтруємо: не-автори бачать тільки опубліковані
        if (!_user.HasRole(UserRole.Author))
        {
            var filtered = articles.Where(a => a.IsPublished).ToList();
            Console.WriteLine($"  [PROTECTION] Відфільтровано: {articles.Count} → {filtered.Count} (тільки опубліковані)");
            return filtered;
        }

        return articles;
    }

    public void Save(Article article)
    {
        // Зберігати можуть тільки автори і вище
        if (!_user.HasRole(UserRole.Author))
            throw new UnauthorizedAccessException($"Користувач {_user} не має прав на збереження статей");

        // Автор може зберігати тільки свої статті (Admin — будь-які)
        if (_user.Role == UserRole.Author && article.AuthorId != _user.UserId)
            throw new UnauthorizedAccessException($"Автор {_user.UserId} не може редагувати чужі статті");

        _inner.Save(article);
    }

    public void Delete(int id)
    {
        // Видаляти можуть тільки адміни
        if (!_user.HasRole(UserRole.Admin))
            throw new UnauthorizedAccessException($"Користувач {_user} не має прав на видалення статей");

        _inner.Delete(id);
    }
}
```

### Крок 5: Caching Proxy — кешування результатів

```csharp
// Кешує результати GetById і GetAll, інвалідує кеш при Save/Delete
public class CachingProxy : IArticleRepository
{
    private readonly IArticleRepository _inner;
    private readonly TimeSpan _ttl;

    private readonly Dictionary<int, (Article Article, DateTime CachedAt)> _itemCache = new();
    private (IReadOnlyList<Article> List, DateTime CachedAt)? _listCache;

    public CachingProxy(IArticleRepository inner, TimeSpan ttl)
    {
        _inner = inner;
        _ttl   = ttl;
    }

    public Article GetById(int id)
    {
        // Перевіряємо кеш
        if (_itemCache.TryGetValue(id, out var cached) && DateTime.UtcNow - cached.CachedAt < _ttl)
        {
            Console.WriteLine($"  [CACHE] HIT: article #{id}");
            return cached.Article;
        }

        Console.WriteLine($"  [CACHE] MISS: article #{id}");
        var article = _inner.GetById(id);

        if (article != null)
            _itemCache[id] = (article, DateTime.UtcNow);

        return article;
    }

    public IReadOnlyList<Article> GetAll()
    {
        if (_listCache.HasValue && DateTime.UtcNow - _listCache.Value.CachedAt < _ttl)
        {
            Console.WriteLine("  [CACHE] HIT: article list");
            return _listCache.Value.List;
        }

        Console.WriteLine("  [CACHE] MISS: article list");
        var list = _inner.GetAll();
        _listCache = (list, DateTime.UtcNow);
        return list;
    }

    public void Save(Article article)
    {
        _inner.Save(article);
        // Інвалідуємо кеш після запису
        _itemCache.Remove(article.Id);
        _listCache = null;
        Console.WriteLine($"  [CACHE] Інвалідовано кеш для article #{article.Id} і list");
    }

    public void Delete(int id)
    {
        _inner.Delete(id);
        _itemCache.Remove(id);
        _listCache = null;
        Console.WriteLine($"  [CACHE] Інвалідовано кеш для article #{id} і list");
    }
}
```

### Крок 6: Logging Proxy — аудит-лог

```csharp
// Записує кожну операцію: хто, що, коли, результат
public class LoggingProxy : IArticleRepository
{
    private readonly IArticleRepository _inner;
    private readonly string _component;

    public LoggingProxy(IArticleRepository inner, string component = "Repo")
    {
        _inner     = inner;
        _component = component;
    }

    public Article GetById(int id)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        try
        {
            var result = _inner.GetById(id);
            sw.Stop();
            var status = result != null ? "OK" : "NOT FOUND";
            Console.WriteLine($"[LOG:{_component}] GetById({id}) → {status} [{sw.ElapsedMilliseconds}ms]");
            return result;
        }
        catch (Exception ex)
        {
            sw.Stop();
            Console.WriteLine($"[LOG:{_component}] GetById({id}) → ERROR: {ex.Message} [{sw.ElapsedMilliseconds}ms]");
            throw;
        }
    }

    public IReadOnlyList<Article> GetAll()
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        try
        {
            var result = _inner.GetAll();
            sw.Stop();
            Console.WriteLine($"[LOG:{_component}] GetAll() → {result.Count} items [{sw.ElapsedMilliseconds}ms]");
            return result;
        }
        catch (Exception ex)
        {
            sw.Stop();
            Console.WriteLine($"[LOG:{_component}] GetAll() → ERROR: {ex.Message} [{sw.ElapsedMilliseconds}ms]");
            throw;
        }
    }

    public void Save(Article article)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        try
        {
            _inner.Save(article);
            sw.Stop();
            Console.WriteLine($"[LOG:{_component}] Save({article}) → OK [{sw.ElapsedMilliseconds}ms]");
        }
        catch (Exception ex)
        {
            sw.Stop();
            Console.WriteLine($"[LOG:{_component}] Save({article}) → ERROR: {ex.Message} [{sw.ElapsedMilliseconds}ms]");
            throw;
        }
    }

    public void Delete(int id)
    {
        var sw = System.Diagnostics.Stopwatch.StartNew();
        try
        {
            _inner.Delete(id);
            sw.Stop();
            Console.WriteLine($"[LOG:{_component}] Delete({id}) → OK [{sw.ElapsedMilliseconds}ms]");
        }
        catch (Exception ex)
        {
            sw.Stop();
            Console.WriteLine($"[LOG:{_component}] Delete({id}) → ERROR: {ex.Message} [{sw.ElapsedMilliseconds}ms]");
            throw;
        }
    }
}
```

### Крок 7: Складання і використання

```csharp
// Фабричний метод — будує повний стек проксі для конкретного користувача
static IArticleRepository BuildRepository(UserContext user)
{
    // Порядок шарів (зовнішній → внутрішній):
    //
    // LoggingProxy       ← логує все, що приходить від клієнта
    //   CachingProxy     ← кешує, щоб не йти зайвий раз у БД
    //     ProtectionProxy← перевіряє права перед реальним запитом
    //       ArticleRepository ← реальна БД

    return new LoggingProxy(
               new CachingProxy(
                   new ProtectionProxy(
                       new ArticleRepository(),
                       user),
                   ttl: TimeSpan.FromMinutes(5)),
               component: "Articles");
}

class Program
{
    static void Main()
    {
        // --- Сценарій 1: Гість ---
        Console.WriteLine("╔══════════════════════════════╗");
        Console.WriteLine("║  Гість читає статті          ║");
        Console.WriteLine("╚══════════════════════════════╝");
        var guest = new UserContext { UserId = "guest-1", Role = UserRole.Guest };
        var guestRepo = BuildRepository(guest);

        var articles = guestRepo.GetAll();
        Console.WriteLine($"Гість бачить статей: {articles.Count}\n"); // тільки опубліковані

        var draft = guestRepo.GetById(2); // чорновик — заблокований
        Console.WriteLine($"Гість отримав чорновик: {draft ?? (object)"null"}\n");

        // --- Сценарій 2: Повторний запит — з кешу ---
        Console.WriteLine("╔══════════════════════════════╗");
        Console.WriteLine("║  Повторний запит (кеш)       ║");
        Console.WriteLine("╚══════════════════════════════╝");
        var articles2 = guestRepo.GetAll(); // має повернутись з кешу
        Console.WriteLine($"Статей (з кешу): {articles2.Count}\n");

        // --- Сценарій 3: Автор редагує свою статтю ---
        Console.WriteLine("╔══════════════════════════════╗");
        Console.WriteLine("║  Автор редагує статтю        ║");
        Console.WriteLine("╚══════════════════════════════╝");
        var alice = new UserContext { UserId = "alice", Role = UserRole.Author };
        var aliceRepo = BuildRepository(alice);

        aliceRepo.Save(new Article { Id = 1, Title = "Патерн Proxy (оновлено)", Content = "...", AuthorId = "alice", IsPublished = true, CreatedAt = DateTime.Now });
        Console.WriteLine();

        // --- Сценарій 4: Автор намагається видалити статтю ---
        Console.WriteLine("╔══════════════════════════════╗");
        Console.WriteLine("║  Автор намагається видалити  ║");
        Console.WriteLine("╚══════════════════════════════╝");
        try
        {
            aliceRepo.Delete(1);
        }
        catch (UnauthorizedAccessException ex)
        {
            Console.WriteLine($"Заблоковано: {ex.Message}\n");
        }

        // --- Сценарій 5: Адмін видаляє ---
        Console.WriteLine("╔══════════════════════════════╗");
        Console.WriteLine("║  Адмін видаляє статтю        ║");
        Console.WriteLine("╚══════════════════════════════╝");
        var admin = new UserContext { UserId = "admin", Role = UserRole.Admin };
        var adminRepo = BuildRepository(admin);
        adminRepo.Delete(1);
    }
}
```

### Очікуваний вивід

```
╔══════════════════════════════╗
║  Гість читає статті          ║
╚══════════════════════════════╝
[LOG:Articles] GetAll() ...
  [CACHE] MISS: article list
  [PROTECTION] Відфільтровано: 3 → 2 (тільки опубліковані)
  [DB] SELECT * FROM articles
[LOG:Articles] GetAll() → 2 items [83ms]
Гість бачить статей: 2

[LOG:Articles] GetById(2) ...
  [CACHE] MISS: article #2
  [PROTECTION] Доступ заборонено: стаття 2 не опублікована. Користувач: guest-1 (Guest)
[LOG:Articles] GetById(2) → NOT FOUND [1ms]
Гість отримав чорновик: null

╔══════════════════════════════╗
║  Повторний запит (кеш)       ║
╚══════════════════════════════╝
[LOG:Articles] GetAll() ...
  [CACHE] HIT: article list
[LOG:Articles] GetAll() → 2 items [0ms]
Статей (з кешу): 2

╔══════════════════════════════╗
║  Автор редагує статтю        ║
╚══════════════════════════════╝
[LOG:Articles] Save([1] "Патерн Proxy (оновлено)" by alice) ...
  [DB] INSERT/UPDATE article id=1
  [CACHE] Інвалідовано кеш для article #1 і list
[LOG:Articles] Save(...) → OK [1ms]

╔══════════════════════════════╗
║  Автор намагається видалити  ║
╚══════════════════════════════╝
[LOG:Articles] Delete(1) → ERROR: Користувач alice (Author) не має прав [0ms]
Заблоковано: Користувач alice (Author) не має прав на видалення статей

╔══════════════════════════════╗
║  Адмін видаляє статтю        ║
╚══════════════════════════════╝
[LOG:Articles] Delete(1) ...
  [DB] DELETE FROM articles WHERE id = 1
  [CACHE] Інвалідовано кеш для article #1 і list
[LOG:Articles] Delete(1) → OK [51ms]
```

---

## Dynamic Proxy через DispatchProxy

.NET дозволяє створювати **динамічні проксі** — без написання окремого класу для кожного інтерфейсу. Це особливо корисно для AOP (Aspect-Oriented Programming): логування, транзакцій, валідації.

```csharp
// Базовий клас для динамічного проксі в .NET
public class AutoLoggingProxy<T> : DispatchProxy
{
    private T _target;

    // Фабричний метод — створює проксі для будь-якого інтерфейсу
    public static T Create(T target)
    {
        // DispatchProxy.Create<інтерфейс, клас_проксі>()
        var proxy = Create<T, AutoLoggingProxy<T>>();
        ((AutoLoggingProxy<T>)(object)proxy)._target = target;
        return proxy;
    }

    // Цей метод викликається при КОЖНОМУ зверненні до будь-якого методу інтерфейсу
    protected override object Invoke(MethodInfo targetMethod, object[] args)
    {
        var argStr = args != null ? string.Join(", ", args.Select(a => a?.ToString() ?? "null")) : "";
        Console.WriteLine($"[DynProxy] → {targetMethod.Name}({argStr})");

        var sw = System.Diagnostics.Stopwatch.StartNew();
        try
        {
            var result = targetMethod.Invoke(_target, args);
            sw.Stop();
            Console.WriteLine($"[DynProxy] ← {targetMethod.Name} OK [{sw.ElapsedMilliseconds}ms]");
            return result;
        }
        catch (TargetInvocationException ex)
        {
            sw.Stop();
            Console.WriteLine($"[DynProxy] ← {targetMethod.Name} ERROR: {ex.InnerException?.Message} [{sw.ElapsedMilliseconds}ms]");
            throw ex.InnerException ?? ex;
        }
    }
}

// Використання — логування для БУДЬ-ЯКОГО інтерфейсу без написання окремого класу:
IArticleRepository repo = AutoLoggingProxy<IArticleRepository>.Create(new ArticleRepository());
repo.GetAll();    // автоматично логується
repo.GetById(1);  // автоматично логується
```

---

## Proxy vs Decorator vs Adapter

Всі три патерни обгортають об'єкт — але мають різну мету:

| Ознака | Proxy | Decorator | Adapter |
|---|---|---|---|
| **Мета** | Контролювати **доступ** до об'єкта | **Додати поведінку** до об'єкта | **Змінити інтерфейс** |
| **Інтерфейс** | Той самий | Той самий | Інший |
| **Знає клієнт?** | Зазвичай **ні** (прозоро) | Зазвичай **ні** | Зазвичай **так** |
| **Реальний об'єкт** | Може **сам створювати** (lazy) | Завжди **отримує** ззовні | Завжди **отримує** ззовні |
| **Типовий фокус** | Доступ, кеш, лінь, безпека | Нова функціональність | Сумісність |

### Найтонша різниця: Proxy vs Decorator

```csharp
// Proxy: контролює доступ або час створення — клієнт не знає про підміну
IService service = new ServiceProxy(); // може сам створити RealService пізніше

// Decorator: додає поведінку до вже існуючого об'єкта — завжди отримує його ззовні
IService service = new LoggingDecorator(new RealService()); // real завжди передається
```

На практиці межа розмита: `CachingProxy` дуже схожий на `CachingDecorator`. Головний критерій — **намір**: якщо мета контролювати доступ → Proxy, якщо розширити функціональність → Decorator.

---

## Переваги та недоліки

### Переваги

- **Принцип відкритості/закритості** — додаємо поведінку (кеш, захист) без зміни реального об'єкта
- **Lazy initialization** — важкі об'єкти створюються тільки коли потрібні
- **Прозорість** — клієнт взагалі не знає, що спілкується з проксі
- **Розподіл відповідальності** — захист, кешування, логування — окремі класи

### Недоліки

- **Затримка відповіді** — зайвий рівень делегування (зазвичай мізерний)
- **Складність** — множення класів для кожного аспекту
- **Неочевидна поведінка** — клієнт може не підозрювати, що об'єкт насправді проксі

---

## Підсумок

| Аспект | Деталь |
|---|---|
| Тип патерну | Структурний (Structural) |
| Вирішує проблему | Контроль доступу, lazy init, кешування, захист |
| Ключова ідея | Замісник з **тим самим** інтерфейсом, який контролює доступ |
| Підтипи | Virtual, Protection, Remote, Caching, Logging, Smart |
| У реальному .NET | `DispatchProxy`, Entity Framework (lazy loading), WCF proxies |

### Коротке правило вибору

```
Треба відкласти створення важкого об'єкта?
  ✅ Virtual Proxy

Треба перевіряти права перед кожним викликом?
  ✅ Protection Proxy

Треба кешувати результати дорогих операцій?
  ✅ Caching Proxy

Треба логувати всі виклики до об'єкта?
  ✅ Logging Proxy (або Dynamic Proxy через DispatchProxy)

Треба додати нову функціональність, а не контролювати доступ?
  ✅ Decorator (а не Proxy)
```

---

*Документ підготовлено як навчальний матеріал з патернів проєктування на C#.*
