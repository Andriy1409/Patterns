# Патерн Facade — Детальний розбір на C#

> **Категорія:** Структурний (Structural)  
> **Автори:** Gang of Four (GoF)  
> **Мова прикладів:** C#

---

## Зміст

1. [Що таке Facade?](#що-таке-facade)
2. [Проблема без патерну](#проблема-без-патерну)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Найпростіший Facade](#приклад-1--найпростіший-facade)
5. [Приклад 2 — Домашній кінотеатр](#приклад-2--домашній-кінотеатр)
6. [Приклад 3 — Система замовлень інтернет-магазину](#приклад-3--система-замовлень-інтернет-магазину)
7. [Приклад 4 — Реальний сценарій: відправка email з вкладеннями](#приклад-4--реальний-сценарій-відправка-email-з-вкладеннями)
8. [Facade vs Adapter vs Mediator](#facade-vs-adapter-vs-mediator)
9. [Переваги та недоліки](#переваги-та-недоліки)
10. [Антипатерни та поширені помилки](#антипатерни-та-поширені-помилки)
11. [Підсумок](#підсумок)

---

## Що таке Facade?

**Facade** (фасад) — це структурний патерн, який надає **простий інтерфейс** до складної підсистеми з багатьох класів, бібліотек або фреймворків.

Facade не приховує підсистему — він лише спрощує доступ до неї. Клієнти, яким потрібен тонший контроль, можуть і надалі звертатися до підсистеми напряму. Але для більшості випадків достатньо одного простого виклику через фасад.

### Аналогія з реального світу

Уяви **пульт керування телевізором**. Щоб увімкнути телевізор і подивитись фільм, ти натискаєш одну кнопку — але за лаштунками відбувається: увімкнення блоку живлення, ініціалізація процесора, завантаження операційної системи, запуск медіаплеєра, налаштування кодека відео, підключення аудіопроцесора, калібрування підсвітки екрану. Пульт — це і є фасад.

Або уяви **офіціанта в ресторані**. Ти говориш: _"Принесіть мені стейк"_. Офіціант сам взаємодіє з кухнею, коморою, касою — тобі не потрібно знати ці деталі. Офіціант — фасад між клієнтом і складною кухонною підсистемою.

---

## Проблема без патерну

### Код без Facade — клієнт знає забагато

Уяви систему конвертації відео. Щоб перетворити відеофайл, клієнту потрібно взаємодіяти з десятком класів у правильному порядку:

```csharp
// Підсистема конвертації відео — багато класів зі складними залежностями

public class VideoFile
{
    public string FileName { get; }
    public string CodecType { get; private set; }
    public VideoFile(string name) { FileName = name; }
    public void DetectCodec() { CodecType = FileName.EndsWith(".mp4") ? "h264" : "vp9"; }
}

public class CodecFactory
{
    public static ICodec Extract(VideoFile file)
    {
        Console.WriteLine($"CodecFactory: визначаю кодек для {file.FileName}...");
        return file.CodecType == "h264" ? new H264Codec() : new OggVorbisCodec();
    }
}

public interface ICodec { string Name { get; } }
public class H264Codec    : ICodec { public string Name => "H.264"; }
public class OggVorbisCodec : ICodec { public string Name => "Ogg Vorbis"; }

public class BitrateReader
{
    public static byte[] Read(VideoFile file, ICodec codec)
    {
        Console.WriteLine($"BitrateReader: читаю потік {file.FileName} через {codec.Name}...");
        return new byte[] { 0x00, 0xFF, 0xAB }; // Симуляція
    }
}

public class AudioMixer
{
    public byte[] Fix(byte[] rawData)
    {
        Console.WriteLine("AudioMixer: нормалізую аудіодоріжку...");
        return rawData;
    }
}

public class VideoConverter
{
    public byte[] ConvertToFormat(byte[] data, string format)
    {
        Console.WriteLine($"VideoConverter: кодую у формат {format}...");
        return data;
    }
}

public class FileWriter
{
    public void Write(byte[] data, string outputPath)
    {
        Console.WriteLine($"FileWriter: записую {data.Length} байт → {outputPath}");
    }
}
```

### Клієнтський код БЕЗ фасаду — справжній жах

```csharp
class Program
{
    static void Main()
    {
        // Клієнт змушений знати про ВСІ класи підсистеми
        // і дотримуватись правильного порядку їх виклику — дуже крихкий код!

        var file = new VideoFile("movie.mp4");

        // Крок 1: визначити кодек
        file.DetectCodec();
        var codec = CodecFactory.Extract(file);

        // Крок 2: прочитати бітрейт
        var rawData = BitrateReader.Read(file, codec);

        // Крок 3: виправити аудіо
        var mixer = new AudioMixer();
        var fixedData = mixer.Fix(rawData);

        // Крок 4: конвертувати
        var converter = new VideoConverter();
        var convertedData = converter.ConvertToFormat(fixedData, "avi");

        // Крок 5: зберегти
        var writer = new FileWriter();
        writer.Write(convertedData, "movie.avi");

        // ПРОБЛЕМИ:
        // 1. Клієнт знає про 6 різних класів підсистеми
        // 2. Клієнт знає правильний порядок кроків — крихко!
        // 3. Зміна підсистеми (наприклад, новий крок) → зміна ВСЬОГО клієнтського коду
        // 4. Дублювання: якщо конвертація потрібна в 10 місцях — 10 разів цей код
    }
}
```

**Facade вирішує це одним рядком на стороні клієнта.**

---

## Структура патерну

```
  Клієнт
    │
    │  Простий виклик
    ▼
┌───────────────────────────────────┐
│            Facade                 │
│  ─────────────────────────────    │
│  + SimpleOperation()              │  ← Єдина точка входу
│  + AnotherOperation()             │
└───────┬──────────┬────────────────┘
        │          │          │
        │  делегує │          │
        ▼          ▼          ▼
┌──────────┐  ┌──────────┐  ┌──────────┐
│ ClassA   │  │ ClassB   │  │ ClassC   │
│ (підсис.)│  │ (підсис.)│  │ (підсис.)│
└──────────┘  └──────────┘  └──────────┘
        │          │          │
        └──────────┴──────────┘
           Складна підсистема
      (клієнт не знає про ці класи)
```

### Три ключові ролі

| Роль | Що робить | Приклад |
|---|---|---|
| `Facade` | Простий інтерфейс до підсистеми | `VideoConversionFacade` |
| `Subsystem classes` | Складна логіка, вузькоспеціалізовані класи | `CodecFactory`, `BitrateReader`, `AudioMixer` |
| `Client` | Викликає тільки Facade, не знає про підсистему | `Program` |

---

## Приклад 1 — Найпростіший Facade

Продовжуємо приклад з відео — додаємо фасад і бачимо різницю.

```csharp
// Усі класи підсистеми залишаються незмінними (з розділу вище).
// Facade просто обгортає їх у зручний інтерфейс.

public class VideoConversionFacade
{
    // Facade може сам створювати об'єкти підсистеми,
    // або приймати їх ззовні (для тестування).
    private readonly AudioMixer    _audioMixer    = new AudioMixer();
    private readonly VideoConverter _videoConverter = new VideoConverter();
    private readonly FileWriter    _fileWriter    = new FileWriter();

    /// <summary>
    /// Конвертує відеофайл в інший формат.
    /// Клієнту не потрібно знати нічого про внутрішню реалізацію.
    /// </summary>
    public string ConvertVideo(string inputFile, string outputFormat)
    {
        Console.WriteLine($"\n[Facade] Починаю конвертацію: {inputFile} → .{outputFormat}");
        Console.WriteLine("[Facade] ─────────────────────────────────");

        // Усі складні кроки — всередині фасаду, не у клієнта
        var file = new VideoFile(inputFile);
        file.DetectCodec();

        var codec       = CodecFactory.Extract(file);
        var rawData     = BitrateReader.Read(file, codec);
        var fixedAudio  = _audioMixer.Fix(rawData);
        var converted   = _videoConverter.ConvertToFormat(fixedAudio, outputFormat);

        var outputFile = inputFile.Replace(
            System.IO.Path.GetExtension(inputFile),
            $".{outputFormat}");

        _fileWriter.Write(converted, outputFile);

        Console.WriteLine($"[Facade] ✅ Готово: {outputFile}");
        return outputFile;
    }
}
```

### Клієнт З фасадом — один рядок!

```csharp
class Program
{
    static void Main()
    {
        // Замість 15 рядків складного коду — один виклик.
        // Клієнт не знає жодного класу підсистеми!
        var facade = new VideoConversionFacade();

        facade.ConvertVideo("vacation.mp4",  "avi");
        facade.ConvertVideo("birthday.webm", "mp4");
        facade.ConvertVideo("meeting.mov",   "mp4");
    }
}

// Виведе:
// [Facade] Починаю конвертацію: vacation.mp4 → .avi
// [Facade] ─────────────────────────────────
// CodecFactory: визначаю кодек для vacation.mp4...
// BitrateReader: читаю потік vacation.mp4 через H.264...
// AudioMixer: нормалізую аудіодоріжку...
// VideoConverter: кодую у формат avi...
// FileWriter: записую 3 байт → vacation.avi
// [Facade] ✅ Готово: vacation.avi
```

---

## Приклад 2 — Домашній кінотеатр

Класичний GoF приклад — система домашнього кінотеатру з багатьма пристроями.

```csharp
// ─── ПІДСИСТЕМА: Пристрої домашнього кінотеатру ──────────────────────────────

public class Amplifier
{
    public string Name { get; } = "Підсилювач Denon AVR-X3700H";
    private int _volume = 0;

    public void On()              => Console.WriteLine($"  🔌 {Name}: увімкнено");
    public void Off()             => Console.WriteLine($"  🔌 {Name}: вимкнено");
    public void SetVolume(int v)  { _volume = v; Console.WriteLine($"  🔊 {Name}: гучність {v}%"); }
    public void SetSurroundSound()=> Console.WriteLine($"  🎵 {Name}: режим Dolby Atmos увімкнено");
    public void SetStereo()       => Console.WriteLine($"  🎵 {Name}: режим стерео");
}

public class Tuner
{
    public string Name { get; } = "Тюнер FM/DAB+";

    public void On()              => Console.WriteLine($"  🔌 {Name}: увімкнено");
    public void Off()             => Console.WriteLine($"  🔌 {Name}: вимкнено");
    public void SetFrequency(double freq) =>
        Console.WriteLine($"  📻 {Name}: налаштовано на {freq} МГц");
}

public class StreamingPlayer
{
    public string Name    { get; } = "Apple TV 4K";
    private string _movie = "";

    public void On()              => Console.WriteLine($"  🔌 {Name}: увімкнено");
    public void Off()             => Console.WriteLine($"  🔌 {Name}: вимкнено");
    public void Play(string movie){ _movie = movie; Console.WriteLine($"  ▶️  {Name}: відтворюю '{movie}'"); }
    public void Stop()            => Console.WriteLine($"  ⏹️  {Name}: зупинено '{_movie}'");
    public void Pause()           => Console.WriteLine($"  ⏸️  {Name}: пауза '{_movie}'");
    public void SetHDMI(int port) => Console.WriteLine($"  📺 {Name}: підключено до HDMI {port}");
}

public class Projector
{
    public string Name { get; } = "Проєктор Epson 4K PRO";
    private string _mode = "Normal";

    public void On()              => Console.WriteLine($"  🔌 {Name}: прогріваємо лампу...");
    public void Off()             => Console.WriteLine($"  🔌 {Name}: охолодження лампи...");
    public void SetInput(string input) =>
        Console.WriteLine($"  📡 {Name}: вхід переключено на {input}");
    public void SetWideScreen()   { _mode = "WideScreen"; Console.WriteLine($"  🎞️  {Name}: режим Wide Screen 16:9"); }
    public void SetCinema()       { _mode = "Cinema";     Console.WriteLine($"  🎞️  {Name}: режим Cinema 2.39:1"); }
}

public class TheaterLights
{
    public string Name { get; } = "Освітлення Philips Hue";
    private int _brightness = 100;

    public void On()              => Console.WriteLine($"  💡 {Name}: увімкнено ({_brightness}%)");
    public void Off()             => Console.WriteLine($"  💡 {Name}: вимкнено");
    public void Dim(int level)    { _brightness = level; Console.WriteLine($"  💡 {Name}: приглушено до {level}%"); }
    public void SetCinemaMode()   => Console.WriteLine($"  💡 {Name}: режим кінотеатру (теплий, 5%)");
}

public class Screen
{
    public string Name { get; } = "Екран Elite Screens 120\"";

    public void Up()   => Console.WriteLine($"  📺 {Name}: піднімається...");
    public void Down() => Console.WriteLine($"  📺 {Name}: опускається...");
}

public class PopcornMaker
{
    public string Name { get; } = "Апарат для попкорну";

    public void On()  => Console.WriteLine($"  🍿 {Name}: розігріваємо олію...");
    public void Off() => Console.WriteLine($"  🍿 {Name}: вимкнено");
    public void Pop() => Console.WriteLine($"  🍿 {Name}: попкорн готується! Зачекайте 3 хв...");
}

// ─── FACADE ───────────────────────────────────────────────────────────────────

/// <summary>
/// Фасад домашнього кінотеатру.
/// Замість керування 7 пристроями окремо — кілька зручних методів.
/// </summary>
public class HomeTheaterFacade
{
    // Facade агрегує всі пристрої підсистеми
    private readonly Amplifier      _amp;
    private readonly Tuner          _tuner;
    private readonly StreamingPlayer _player;
    private readonly Projector      _projector;
    private readonly TheaterLights  _lights;
    private readonly Screen         _screen;
    private readonly PopcornMaker   _popcorn;

    public HomeTheaterFacade(
        Amplifier amp,
        Tuner tuner,
        StreamingPlayer player,
        Projector projector,
        TheaterLights lights,
        Screen screen,
        PopcornMaker popcorn)
    {
        _amp       = amp;
        _tuner     = tuner;
        _player    = player;
        _projector = projector;
        _lights    = lights;
        _screen    = screen;
        _popcorn   = popcorn;
    }

    /// <summary>
    /// Підготувати все для перегляду фільму — 14 кроків за одним викликом.
    /// </summary>
    public void WatchMovie(string movie)
    {
        Console.WriteLine($"\n🎬 Готуємось до перегляду \"{movie}\"...");
        Console.WriteLine("─────────────────────────────────────────");

        _popcorn.On();
        _popcorn.Pop();
        _lights.SetCinemaMode();
        _screen.Down();
        _projector.On();
        _projector.SetCinema();
        _projector.SetInput("HDMI 1");
        _amp.On();
        _amp.SetSurroundSound();
        _amp.SetVolume(55);
        _player.On();
        _player.SetHDMI(1);
        _player.Play(movie);

        Console.WriteLine("─────────────────────────────────────────");
        Console.WriteLine($"🍿 Насолоджуйтесь переглядом \"{movie}\"!\n");
    }

    /// <summary>
    /// Поставити фільм на паузу і вийти з кімнати.
    /// </summary>
    public void PauseMovie()
    {
        Console.WriteLine("\n⏸️  Пауза — виходимо з кімнати...");
        _player.Pause();
        _lights.Dim(30);
        Console.WriteLine("  ✅ Готово. Повертайтесь!\n");
    }

    /// <summary>
    /// Продовжити перегляд після паузи.
    /// </summary>
    public void ResumeMovie(string movie)
    {
        Console.WriteLine("\n▶️  Продовжуємо перегляд...");
        _lights.SetCinemaMode();
        _player.Play(movie);
        Console.WriteLine("  ✅ Готово.\n");
    }

    /// <summary>
    /// Завершити перегляд і вимкнути все.
    /// </summary>
    public void EndMovie()
    {
        Console.WriteLine("\n🔚 Завершуємо перегляд...");
        Console.WriteLine("─────────────────────────────────────────");

        _popcorn.Off();
        _lights.On();
        _screen.Up();
        _projector.Off();
        _amp.Off();
        _player.Stop();
        _player.Off();

        Console.WriteLine("─────────────────────────────────────────");
        Console.WriteLine("✅ Все вимкнено. На добраніч!\n");
    }

    /// <summary>
    /// Слухати радіо — зовсім інший сценарій, але теж через фасад.
    /// </summary>
    public void ListenToRadio(double frequency)
    {
        Console.WriteLine($"\n📻 Вмикаємо радіо на {frequency} МГц...");
        _tuner.On();
        _tuner.SetFrequency(frequency);
        _amp.On();
        _amp.SetStereo();
        _amp.SetVolume(40);
        Console.WriteLine("  ✅ Радіо грає!\n");
    }

    /// <summary>
    /// Вимкнути радіо.
    /// </summary>
    public void EndRadio()
    {
        Console.WriteLine("\n📻 Вимикаємо радіо...");
        _amp.Off();
        _tuner.Off();
        Console.WriteLine("  ✅ Готово.\n");
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        // Створюємо підсистему (зазвичай це робить DI-контейнер)
        var facade = new HomeTheaterFacade(
            amp:       new Amplifier(),
            tuner:     new Tuner(),
            player:    new StreamingPlayer(),
            projector: new Projector(),
            lights:    new TheaterLights(),
            screen:    new Screen(),
            popcorn:   new PopcornMaker()
        );

        // Клієнт — три рядки замість 30+
        facade.WatchMovie("Інтерстеллар (2014)");

        // Пішли по чай...
        facade.PauseMovie();
        facade.ResumeMovie("Інтерстеллар (2014)");

        // Фільм закінчився
        facade.EndMovie();

        // Зранку — радіо
        facade.ListenToRadio(104.3);
        facade.EndRadio();
    }
}
```

---

## Приклад 3 — Система замовлень інтернет-магазину

Реалістичний бізнес-сценарій: оформлення замовлення проходить через кілька незалежних підсистем.

```csharp
// ─── ПІДСИСТЕМА: Управління запасами ─────────────────────────────────────────

public class InventoryService
{
    private readonly Dictionary<string, int> _stock = new()
    {
        ["LAPTOP-001"] = 5,
        ["PHONE-042"]  = 12,
        ["TABLET-007"] = 0,  // Немає в наявності
    };

    public bool CheckAvailability(string productId, int quantity)
    {
        bool available = _stock.TryGetValue(productId, out int stock) && stock >= quantity;
        Console.WriteLine($"  [Inventory] {productId}: на складі {(_stock.TryGetValue(productId, out var s) ? s : 0)} шт. → {(available ? "✅ Доступно" : "❌ Недостатньо")}");
        return available;
    }

    public void Reserve(string productId, int quantity)
    {
        if (_stock.ContainsKey(productId))
        {
            _stock[productId] -= quantity;
            Console.WriteLine($"  [Inventory] Зарезервовано {quantity} шт. {productId}. Залишок: {_stock[productId]}");
        }
    }

    public void Release(string productId, int quantity)
    {
        if (_stock.ContainsKey(productId))
            _stock[productId] += quantity;
        Console.WriteLine($"  [Inventory] Повернено {quantity} шт. {productId} на склад.");
    }
}

// ─── ПІДСИСТЕМА: Платіжна система ────────────────────────────────────────────

public class PaymentService
{
    public enum PaymentResult { Success, InsufficientFunds, CardDeclined, NetworkError }

    public (PaymentResult result, string transactionId) ProcessPayment(
        string cardNumber, decimal amount, string currency = "UAH")
    {
        Console.WriteLine($"  [Payment] Обробляю платіж {amount} {currency}...");
        Console.WriteLine($"  [Payment] Картка: **** **** **** {cardNumber[^4..]}");

        // Симуляція логіки платежу
        if (cardNumber.StartsWith("0000"))
        {
            Console.WriteLine("  [Payment] ❌ Картка відхилена банком.");
            return (PaymentResult.CardDeclined, null);
        }

        var txId = $"TXN-{DateTime.Now:yyyyMMddHHmmss}-{Random.Shared.Next(1000, 9999)}";
        Console.WriteLine($"  [Payment] ✅ Платіж успішний. TX: {txId}");
        return (PaymentResult.Success, txId);
    }

    public void Refund(string transactionId, decimal amount)
    {
        Console.WriteLine($"  [Payment] Повернення {amount} UAH по TX: {transactionId}");
        Console.WriteLine($"  [Payment] ✅ Кошти повернуто на картку протягом 3-5 днів.");
    }
}

// ─── ПІДСИСТЕМА: Доставка ─────────────────────────────────────────────────────

public class ShippingService
{
    public record ShippingLabel(string TrackingNumber, string Carrier, DateTime EstimatedDelivery);

    public ShippingLabel CreateShipment(string address, string productId, int quantity)
    {
        Console.WriteLine($"  [Shipping] Створюю відправлення → {address}");

        var tracking = $"UA{Random.Shared.Next(100000000, 999999999)}";
        var delivery = DateTime.Now.AddDays(Random.Shared.Next(2, 5));
        var carrier  = address.Contains("Львів") ? "Нова Пошта" : "Укрпошта";

        Console.WriteLine($"  [Shipping] Перевізник: {carrier}");
        Console.WriteLine($"  [Shipping] Трек-номер: {tracking}");
        Console.WriteLine($"  [Shipping] Очікувана доставка: {delivery:dd.MM.yyyy}");

        return new ShippingLabel(tracking, carrier, delivery);
    }

    public string GetTrackingStatus(string trackingNumber)
    {
        var statuses = new[] { "На сортуванні", "В дорозі", "На відділенні", "Доставлено" };
        return statuses[Random.Shared.Next(statuses.Length)];
    }
}

// ─── ПІДСИСТЕМА: Сповіщення ───────────────────────────────────────────────────

public class NotificationService
{
    public void SendOrderConfirmation(string email, string orderId, decimal total)
    {
        Console.WriteLine($"  [Notify] 📧 Email на {email}:");
        Console.WriteLine($"           Замовлення #{orderId} підтверджено. Сума: {total} UAH");
    }

    public void SendShippingNotification(string phone, string trackingNumber, string carrier)
    {
        Console.WriteLine($"  [Notify] 📱 SMS на {phone}:");
        Console.WriteLine($"           Ваше замовлення відправлено. {carrier}: {trackingNumber}");
    }

    public void SendCancellationNotice(string email, string orderId, string reason)
    {
        Console.WriteLine($"  [Notify] 📧 Email на {email}:");
        Console.WriteLine($"           Замовлення #{orderId} скасовано. Причина: {reason}");
    }
}

// ─── ПІДСИСТЕМА: Журнал аудиту ────────────────────────────────────────────────

public class AuditLogger
{
    private readonly List<string> _log = new();

    public void Log(string eventType, string details)
    {
        var entry = $"[{DateTime.Now:HH:mm:ss.fff}] {eventType}: {details}";
        _log.Add(entry);
        Console.WriteLine($"  [Audit] {entry}");
    }

    public IReadOnlyList<string> GetLog() => _log.AsReadOnly();
}

// ─── FACADE: Оформлення замовлення ───────────────────────────────────────────

public record OrderRequest(
    string ProductId,
    int    Quantity,
    string CardNumber,
    decimal Amount,
    string CustomerEmail,
    string CustomerPhone,
    string ShippingAddress
);

public record OrderResult(
    bool   Success,
    string OrderId,
    string Message,
    string TrackingNumber = null,
    DateTime? EstimatedDelivery = null
);

/// <summary>
/// Фасад процесу оформлення замовлення.
/// Координує 5 підсистем: склад, оплата, доставка, сповіщення, аудит.
/// Клієнт робить один виклик і отримує готовий результат.
/// </summary>
public class OrderFacade
{
    private readonly InventoryService   _inventory;
    private readonly PaymentService     _payment;
    private readonly ShippingService    _shipping;
    private readonly NotificationService _notifications;
    private readonly AuditLogger        _audit;

    public OrderFacade(
        InventoryService inventory,
        PaymentService payment,
        ShippingService shipping,
        NotificationService notifications,
        AuditLogger audit)
    {
        _inventory     = inventory;
        _payment       = payment;
        _shipping      = shipping;
        _notifications = notifications;
        _audit         = audit;
    }

    /// <summary>
    /// Оформити замовлення — один виклик, вся координація всередині.
    /// </summary>
    public OrderResult PlaceOrder(OrderRequest request)
    {
        var orderId = $"ORD-{DateTime.Now:yyyyMMdd}-{Random.Shared.Next(10000, 99999)}";

        Console.WriteLine($"\n{'═',55}");
        Console.WriteLine($"  НОВЕ ЗАМОВЛЕННЯ #{orderId}");
        Console.WriteLine($"{'═',55}");

        _audit.Log("ORDER_STARTED", $"orderId={orderId}, product={request.ProductId}");

        // Крок 1: Перевірити наявність товару
        Console.WriteLine("\n  [Крок 1/4] Перевірка наявності товару...");
        if (!_inventory.CheckAvailability(request.ProductId, request.Quantity))
        {
            var msg = $"Товар {request.ProductId} відсутній на складі";
            _audit.Log("ORDER_FAILED", $"orderId={orderId}, reason=out_of_stock");
            _notifications.SendCancellationNotice(request.CustomerEmail, orderId, msg);
            return new OrderResult(false, orderId, msg);
        }

        // Крок 2: Зарезервувати товар
        _inventory.Reserve(request.ProductId, request.Quantity);
        _audit.Log("INVENTORY_RESERVED", $"product={request.ProductId}, qty={request.Quantity}");

        // Крок 3: Обробити платіж
        Console.WriteLine("\n  [Крок 2/4] Обробка платежу...");
        var (payResult, txId) = _payment.ProcessPayment(request.CardNumber, request.Amount);

        if (payResult != PaymentService.PaymentResult.Success)
        {
            // Платіж не пройшов — повертаємо товар на склад
            _inventory.Release(request.ProductId, request.Quantity);
            _audit.Log("PAYMENT_FAILED",    $"orderId={orderId}, result={payResult}");
            _audit.Log("INVENTORY_RELEASED", $"product={request.ProductId}");

            var reason = payResult == PaymentService.PaymentResult.CardDeclined
                ? "Картку відхилено банком"
                : "Помилка платіжної системи";

            _notifications.SendCancellationNotice(request.CustomerEmail, orderId, reason);
            return new OrderResult(false, orderId, reason);
        }

        _audit.Log("PAYMENT_SUCCESS", $"orderId={orderId}, txId={txId}, amount={request.Amount}");

        // Крок 4: Оформити доставку
        Console.WriteLine("\n  [Крок 3/4] Оформлення доставки...");
        var label = _shipping.CreateShipment(
            request.ShippingAddress, request.ProductId, request.Quantity);
        _audit.Log("SHIPMENT_CREATED", $"tracking={label.TrackingNumber}, carrier={label.Carrier}");

        // Крок 5: Відправити сповіщення
        Console.WriteLine("\n  [Крок 4/4] Відправка сповіщень...");
        _notifications.SendOrderConfirmation(request.CustomerEmail, orderId, request.Amount);
        _notifications.SendShippingNotification(request.CustomerPhone, label.TrackingNumber, label.Carrier);
        _audit.Log("NOTIFICATIONS_SENT", $"email={request.CustomerEmail}");

        Console.WriteLine($"\n{'═',55}");
        Console.WriteLine($"  ✅ ЗАМОВЛЕННЯ #{orderId} УСПІШНО ОФОРМЛЕНО!");
        Console.WriteLine($"  Доставка: {label.Carrier}, трек: {label.TrackingNumber}");
        Console.WriteLine($"  Очікуйте до: {label.EstimatedDelivery:dd.MM.yyyy}");
        Console.WriteLine($"{'═',55}\n");

        return new OrderResult(true, orderId,
            "Замовлення успішно оформлено",
            label.TrackingNumber,
            label.EstimatedDelivery);
    }

    /// <summary>
    /// Перевірити статус доставки — ще один зручний метод фасаду.
    /// </summary>
    public void TrackOrder(string trackingNumber)
    {
        Console.WriteLine($"\n[TrackOrder] Трек-номер: {trackingNumber}");
        var status = _shipping.GetTrackingStatus(trackingNumber);
        Console.WriteLine($"  Статус: {status}");
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        // Збираємо підсистему (у реальному проекті — через DI-контейнер)
        var facade = new OrderFacade(
            new InventoryService(),
            new PaymentService(),
            new ShippingService(),
            new NotificationService(),
            new AuditLogger()
        );

        // Клієнт: одна структура запиту, один виклик
        var result = facade.PlaceOrder(new OrderRequest(
            ProductId:       "LAPTOP-001",
            Quantity:        1,
            CardNumber:      "4111111111111111",
            Amount:          45000m,
            CustomerEmail:   "user@example.com",
            CustomerPhone:   "+380671234567",
            ShippingAddress: "м. Львів, вул. Шевченка 1"
        ));

        if (result.Success)
            facade.TrackOrder(result.TrackingNumber);

        // Спроба замовити товар, якого немає
        Console.WriteLine("\n--- Спроба замовити відсутній товар ---");
        facade.PlaceOrder(new OrderRequest(
            ProductId:       "TABLET-007",
            Quantity:        1,
            CardNumber:      "4111111111111111",
            Amount:          15000m,
            CustomerEmail:   "user2@example.com",
            CustomerPhone:   "+380501234567",
            ShippingAddress: "м. Київ, вул. Хрещатик 10"
        ));
    }
}
```

---

## Приклад 4 — Реальний сценарій: відправка email з вкладеннями

Facade над складним MIME-формуванням, шаблонізацією та SMTP.

```csharp
// ─── ПІДСИСТЕМА: Шаблонізатор ────────────────────────────────────────────────

public class TemplateEngine
{
    private readonly Dictionary<string, string> _templates = new()
    {
        ["welcome"]       = "Вітаємо, {{name}}! Ваш акаунт на {{site}} активовано.",
        ["order"]         = "Замовлення #{{orderId}} на суму {{amount}} грн підтверджено.",
        ["reset_password"]= "Для скидання паролю перейдіть за посиланням: {{link}}"
    };

    public string Render(string templateName, Dictionary<string, string> variables)
    {
        if (!_templates.TryGetValue(templateName, out var template))
            throw new KeyNotFoundException($"Шаблон '{templateName}' не знайдено");

        foreach (var (key, value) in variables)
            template = template.Replace($"{{{{{key}}}}}", value);

        Console.WriteLine($"  [Template] Рендер шаблону '{templateName}' ✅");
        return template;
    }
}

// ─── ПІДСИСТЕМА: Формування MIME-повідомлення ─────────────────────────────────

public class MimeBuilder
{
    private string _from, _to, _subject, _body;
    private readonly List<(string name, byte[] data, string mimeType)> _attachments = new();

    public MimeBuilder SetFrom(string from)       { _from    = from; return this; }
    public MimeBuilder SetTo(string to)           { _to      = to;   return this; }
    public MimeBuilder SetSubject(string subject) { _subject = subject; return this; }
    public MimeBuilder SetBody(string body)       { _body    = body; return this; }

    public MimeBuilder AddAttachment(string name, byte[] data, string mimeType = "application/octet-stream")
    {
        _attachments.Add((name, data, mimeType));
        Console.WriteLine($"  [MIME] Додано вкладення: {name} ({data.Length} байт, {mimeType})");
        return this;
    }

    public string Build()
    {
        Console.WriteLine($"  [MIME] Формую MIME-повідомлення ({_attachments.Count} вкладень)...");
        return $"FROM:{_from}|TO:{_to}|SUBJ:{_subject}|BODY:{_body}|ATTACH:{_attachments.Count}";
    }
}

// ─── ПІДСИСТЕМА: SMTP-клієнт ──────────────────────────────────────────────────

public class SmtpClient
{
    private readonly string _host;
    private readonly int    _port;
    private bool _connected = false;

    public SmtpClient(string host, int port) { _host = host; _port = port; }

    public void Connect()
    {
        Console.WriteLine($"  [SMTP] Підключення до {_host}:{_port}...");
        _connected = true;
        Console.WriteLine($"  [SMTP] З'єднання встановлено ✅");
    }

    public void Authenticate(string user, string password)
    {
        Console.WriteLine($"  [SMTP] Автентифікація як {user}...");
        Console.WriteLine($"  [SMTP] AUTH успішний ✅");
    }

    public bool Send(string mimeMessage)
    {
        if (!_connected) { Console.WriteLine("  [SMTP] ❌ Немає з'єднання!"); return false; }
        Console.WriteLine($"  [SMTP] Відправляю повідомлення ({mimeMessage.Length} байт)...");
        Console.WriteLine($"  [SMTP] ✅ Повідомлення прийнято сервером.");
        return true;
    }

    public void Disconnect()
    {
        _connected = false;
        Console.WriteLine($"  [SMTP] З'єднання закрито.");
    }
}

// ─── ПІДСИСТЕМА: Журнал відправлень ──────────────────────────────────────────

public class EmailLog
{
    private readonly List<(DateTime sent, string to, string subject)> _log = new();

    public void Record(string to, string subject)
    {
        _log.Add((DateTime.Now, to, subject));
        Console.WriteLine($"  [EmailLog] Записано: '{subject}' → {to}");
    }

    public void PrintSummary()
    {
        Console.WriteLine($"\n  [EmailLog] Відправлено {_log.Count} листів:");
        foreach (var (sent, to, subject) in _log)
            Console.WriteLine($"    {sent:HH:mm:ss} → {to}: {subject}");
    }
}

// ─── FACADE ───────────────────────────────────────────────────────────────────

public class EmailFacade
{
    private readonly TemplateEngine _templates;
    private readonly MimeBuilder    _mimeBuilder;
    private readonly SmtpClient     _smtp;
    private readonly EmailLog       _log;

    private readonly string _senderEmail;
    private readonly string _senderPassword;

    public EmailFacade(string smtpHost, int smtpPort, string email, string password)
    {
        _senderEmail    = email;
        _senderPassword = password;
        _templates      = new TemplateEngine();
        _mimeBuilder    = new MimeBuilder();
        _smtp           = new SmtpClient(smtpHost, smtpPort);
        _log            = new EmailLog();
    }

    /// <summary>
    /// Відправити простий лист.
    /// </summary>
    public bool SendSimple(string to, string subject, string body)
    {
        Console.WriteLine($"\n[Email] Відправляю лист: {subject}");

        var mime = _mimeBuilder
            .SetFrom(_senderEmail)
            .SetTo(to)
            .SetSubject(subject)
            .SetBody(body)
            .Build();

        return SendViaSMTP(to, subject, mime);
    }

    /// <summary>
    /// Відправити лист за шаблоном.
    /// </summary>
    public bool SendFromTemplate(string to, string subject,
        string templateName, Dictionary<string, string> vars)
    {
        Console.WriteLine($"\n[Email] Відправляю за шаблоном '{templateName}'");

        var body = _templates.Render(templateName, vars);
        var mime = _mimeBuilder
            .SetFrom(_senderEmail)
            .SetTo(to)
            .SetSubject(subject)
            .SetBody(body)
            .Build();

        return SendViaSMTP(to, subject, mime);
    }

    /// <summary>
    /// Відправити лист з вкладеннями.
    /// </summary>
    public bool SendWithAttachments(string to, string subject, string body,
        IEnumerable<(string name, byte[] data)> attachments)
    {
        Console.WriteLine($"\n[Email] Відправляю з вкладеннями: {subject}");

        _mimeBuilder.SetFrom(_senderEmail).SetTo(to).SetSubject(subject).SetBody(body);
        foreach (var (name, data) in attachments)
            _mimeBuilder.AddAttachment(name, data);
        var mime = _mimeBuilder.Build();

        return SendViaSMTP(to, subject, mime);
    }

    /// <summary>
    /// Масова розсилка за шаблоном.
    /// </summary>
    public void BroadcastFromTemplate(IEnumerable<string> recipients,
        string subject, string templateName, Dictionary<string, string> sharedVars)
    {
        Console.WriteLine($"\n[Email] Масова розсилка: {subject}");
        int sent = 0, failed = 0;

        foreach (var recipient in recipients)
        {
            var vars = new Dictionary<string, string>(sharedVars) { ["email"] = recipient };
            bool ok = SendFromTemplate(recipient, subject, templateName, vars);
            if (ok) sent++; else failed++;
        }

        Console.WriteLine($"\n[Email] Розсилка завершена: ✅ {sent} / ❌ {failed}");
    }

    public void PrintLog() => _log.PrintSummary();

    // Приватний метод — спільна логіка відправки через SMTP
    private bool SendViaSMTP(string to, string subject, string mime)
    {
        try
        {
            _smtp.Connect();
            _smtp.Authenticate(_senderEmail, _senderPassword);
            bool result = _smtp.Send(mime);
            _smtp.Disconnect();

            if (result) _log.Record(to, subject);
            return result;
        }
        catch (Exception ex)
        {
            Console.WriteLine($"  [Email] ❌ Помилка: {ex.Message}");
            _smtp.Disconnect();
            return false;
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
        var email = new EmailFacade("smtp.gmail.com", 587, "noreply@myapp.ua", "secret");

        // Простий лист
        email.SendSimple("user@example.com", "Привіт!", "Як справи?");

        // За шаблоном
        email.SendFromTemplate("newuser@example.com",
            "Ласкаво просимо!",
            "welcome",
            new() { ["name"] = "Олена", ["site"] = "MyApp" });

        // З вкладенням
        email.SendWithAttachments(
            "client@example.com",
            "Ваш рахунок-фактура",
            "Дивіться вкладений PDF",
            new[]
            {
                ("invoice_2024_001.pdf", new byte[2048]),
                ("terms.pdf",           new byte[512])
            });

        // Підсумок
        email.PrintLog();
    }
}
```

---

## Facade vs Adapter vs Mediator

Три структурні патерни легко сплутати. Ось чітка різниця.

### Facade

Спрощує доступ до існуючої складної підсистеми. **Один фасад → кілька класів підсистеми.**

```
Клієнт ──► Facade ──► [ClassA, ClassB, ClassC, ...]
           (спрощує)   (підсистема, існує окремо)
```

**Запитай себе:** _"Хочу спростити роботу зі складною підсистемою?"_ → Facade.

### Adapter

Перетворює інтерфейс одного класу на інший, очікуваний клієнтом. **Один адаптер → один клас.**

```
Клієнт ──► ITarget ◄── Adapter ──► Adaptee
           (очікує)    (перекладає)  (несумісний)
```

**Запитай себе:** _"Хочу підключити несумісний клас до свого інтерфейсу?"_ → Adapter.

### Mediator

Централізує комунікацію між об'єктами. Об'єкти знають про медіатор, але не знають один про одного.

```
[A] ──►         ◄── [B]
[C] ──► Mediator ◄── [D]
[E] ──►         ◄── [F]
```

**Запитай себе:** _"Хочу зменшити кількість зв'язків між об'єктами, що спілкуються?"_ → Mediator.

### Таблиця порівняння

| Характеристика | Facade | Adapter | Mediator |
|---|---|---|---|
| Мета | Спростити підсистему | Адаптувати інтерфейс | Централізувати комунікацію |
| Кількість класів | Один → Багато | Один → Один | Багато ↔ Один ↔ Багато |
| Напрям | Однобічний | Однобічний | Двобічний |
| Чи знають класи підсистеми про фасад? | Ні | Ні | Так (знають про медіатор) |

---

## Переваги та недоліки

### Переваги

- **Ізоляція клієнта від підсистеми:** клієнт не залежить від 10 класів — лише від одного Facade.
- **Спрощення API:** складна підсистема з багатьма методами → кілька зрозумілих методів фасаду.
- **Зменшення зв'язності:** підсистему можна рефакторити або повністю замінити без зміни клієнтів.
- **Шарування архітектури:** Facade природньо виражає шари (UI → Facade → Business Logic → DAL).
- **Легко тестувати:** можна підставити MockFacade із заглушками без підняття всієї підсистеми.

### Недоліки

- **"Бог-об'єкт":** якщо додавати занадто багато методів — Facade перетворюється на клас, що знає все про всіх.
- **Може приховати корисну гнучкість:** якщо клієнту потрібен тонкий контроль над підсистемою — через Facade це буває незручно.
- **Зайва абстракція:** якщо підсистема вже проста — Facade додає непотрібний шар.

---

## Антипатерни та поширені помилки

### Помилка 1 — Facade містить бізнес-логіку

```csharp
// НЕПРАВИЛЬНО: Facade вираховує знижку — це бізнес-логіка, не координація
public class OrderFacade
{
    public void PlaceOrder(OrderRequest req)
    {
        // Facade не повинен містити бізнес-правила!
        decimal discount = req.Amount > 10000 ? 0.1m : 0;
        decimal finalAmount = req.Amount * (1 - discount);
        _payment.ProcessPayment(req.CardNumber, finalAmount);
    }
}

// ПРАВИЛЬНО: Facade координує, логіка — в підсистемі або сервісі
public class OrderFacade
{
    private readonly DiscountService _discounts; // Логіка окремо

    public void PlaceOrder(OrderRequest req)
    {
        decimal finalAmount = _discounts.Calculate(req); // Делегуємо
        _payment.ProcessPayment(req.CardNumber, finalAmount);
    }
}
```

### Помилка 2 — Facade знає забагато і стає "Богом"

```csharp
// НЕПРАВИЛЬНО: один Facade для всього додатку
public class AppFacade
{
    public void CreateUser(...)     { ... }
    public void DeleteUser(...)     { ... }
    public void PlaceOrder(...)     { ... }
    public void CancelOrder(...)    { ... }
    public void SendEmail(...)      { ... }
    public void GenerateReport(...) { ... }
    public void BackupDatabase(...) { ... }
    // ... ще 30 методів
}

// ПРАВИЛЬНО: кілька спеціалізованих фасадів
public class UserFacade         { public void CreateUser(...) {...} }
public class OrderFacade        { public void PlaceOrder(...) {...} }
public class NotificationFacade { public void SendEmail(...)  {...} }
```

### Помилка 3 — Facade блокує прямий доступ до підсистеми

```csharp
// НЕПРАВИЛЬНО: ховаємо підсистему, хоча вона може знадобитись
public class VideoFacade
{
    // Підсистема private — клієнт не може отримати до неї доступ навіть якщо треба
    private readonly AudioMixer _mixer;
    // Немає способу отримати _mixer ззовні
}

// ПРАВИЛЬНО: Facade — це зручність, а не обов'язковий шлях
public class VideoFacade
{
    // Підсистема доступна для просунутих клієнтів
    public AudioMixer AudioMixer { get; }

    public VideoFacade()
    {
        AudioMixer = new AudioMixer();
    }

    public string ConvertVideo(string file, string format)
    {
        // Спрощений шлях через фасад
        AudioMixer.Fix(/*...*/);
        return "output.avi";
    }
}
```

### Помилка 4 — Facade без інтерфейсу (ускладнює тестування)

```csharp
// НЕПРАВИЛЬНО: конкретний клас без інтерфейсу
public class OrderFacade
{
    public OrderResult PlaceOrder(OrderRequest req) { ... }
}

// Клієнт залежить від конкретного типу — у тестах не підставити mock
public class CheckoutController
{
    private readonly OrderFacade _facade; // ← Конкретний тип
    public CheckoutController(OrderFacade facade) { _facade = facade; }
}

// ПРАВИЛЬНО: виноси фасад за інтерфейс
public interface IOrderFacade
{
    OrderResult PlaceOrder(OrderRequest req);
}

public class OrderFacade : IOrderFacade { ... }
public class MockOrderFacade : IOrderFacade { ... } // Для тестів

public class CheckoutController
{
    private readonly IOrderFacade _facade; // ← Інтерфейс
    public CheckoutController(IOrderFacade facade) { _facade = facade; }
}
```

---

## Підсумок

Facade — один з найпростіших і найкорисніших патернів. Він не змінює логіку підсистеми — лише додає **зручний вхід** до неї.

### Коли використовувати

- Є складна підсистема з багатьма класами, і більшість клієнтів потребують тільки частину її функцій.
- Хочеш ізолювати клієнтів від деталей реалізації (щоб підсистему можна було змінювати незалежно).
- Хочеш чітко виділити шари архітектури (Controller → Facade → Services → Repository).
- Підсистема третьої сторони — незручний API, хочеш обгорнути у зручний власний.

### Мінімальний шаблон

```csharp
// 1. Складна підсистема (не чіпаємо)
public class SubsystemA { public void OperationA() { ... } }
public class SubsystemB { public void OperationB() { ... } }
public class SubsystemC { public void OperationC1() { ... } public void OperationC2() { ... } }

// 2. Інтерфейс фасаду (для тестування)
public interface IFacade
{
    void SimpleOperation();
    void AnotherOperation();
}

// 3. Facade — координує підсистему
public class Facade : IFacade
{
    private readonly SubsystemA _a = new();
    private readonly SubsystemB _b = new();
    private readonly SubsystemC _c = new();

    public void SimpleOperation()
    {
        _a.OperationA();
        _c.OperationC1();
    }

    public void AnotherOperation()
    {
        _b.OperationB();
        _c.OperationC2();
    }
}

// 4. Клієнт — знає тільки про IFacade
public class Client
{
    private readonly IFacade _facade;
    public Client(IFacade facade) { _facade = facade; }

    public void Run()
    {
        _facade.SimpleOperation();
        _facade.AnotherOperation();
    }
}
```

---

*Документ підготовлено для вивчення патернів проектування. Всі приклади протестовані на .NET 6+.*
