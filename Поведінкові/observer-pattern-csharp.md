# Патерн Observer (Спостерігач) — Детальний розбір на C#

> **Категорія:** Поведінковий (Behavioral)  
> **Автори:** Gang of Four (GoF)  
> **Мова прикладів:** C#

---

## Зміст

1. [Що таке Observer?](#що-таке-observer)
   - [Аналогія з реального світу](#аналогія-з-реального-світу)
2. [Проблема без патерну](#проблема-без-патерну)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Класична реалізація через інтерфейси](#приклад-1--класична-реалізація-через-інтерфейси)
5. [Приклад 2 — Ідіоматичний C# через event/EventHandler](#приклад-2--ідіоматичний-c-через-eventeventhandler)
6. [Приклад 3 — INotifyPropertyChanged для MVVM](#приклад-3--inotifypropertychanged-для-mvvm)
7. [Приклад 4 (реальний сценарій) — Система сповіщень про статус замовлення](#приклад-4-реальний-сценарій--система-сповіщень-про-статус-замовлення)
8. [Observer vs Mediator vs Pub-Sub](#observer-vs-mediator-vs-pub-sub)
9. [Переваги та недоліки](#переваги-та-недоліки)
10. [Антипатерни та поширені помилки](#антипатерни-та-поширені-помилки)
11. [Підсумок](#підсумок)
    - [Мінімальний шаблон](#мінімальний-шаблон)

---

## Що таке Observer?

**Observer (Спостерігач)** — це поведінковий патерн проектування, який визначає залежність типу "один до багатьох" (one-to-many) між об'єктами таким чином, що коли один об'єкт (**Subject**, або **Observable**) змінює свій стан, усі залежні від нього об'єкти (**Observers**) автоматично сповіщаються та оновлюються.

Ключова ідея: **Subject не повинен знати нічого конкретного про своїх Observer-ів**, окрім того, що вони реалізують певний інтерфейс сповіщення. Це дозволяє додавати, видаляти чи змінювати спостерігачів у будь-який момент часу, не торкаючись коду самого Subject-а.

Формальне визначення GoF:

> Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.

Головні учасники:

- **Subject (Observable)** — зберігає список спостерігачів, надає методи для підписки/відписки та сповіщення.
- **Observer** — визначає інтерфейс оновлення (`Update`), який викликається Subject-ом.
- **ConcreteSubject** — конкретний об'єкт зі станом, зміна якого цікавить спостерігачів.
- **ConcreteObserver** — конкретна реалізація реакції на зміну стану.

### Аналогія з реального світу

Уявіть **підписку на журнал** (або на YouTube-канал).

- Видавництво (Subject) випускає новий номер журналу.
- Воно **не знає особисто** кожного підписника — воно просто має список адрес підписки.
- Коли виходить новий номер, видавництво розсилає його **всім**, хто підписаний, незалежно від того, хто ці люди і що вони робитимуть з журналом.
- Ви можете **підписатися** сьогодні і **відписатися** завтра — видавництву не потрібно змінювати процес друку чи розсилки через це.
- Різні підписники реагують по-різному: хтось читає одразу, хтось відкладає, хтось пересилає другові — видавництву це байдуже.

Так само й у коді: **Subject випускає подію "стан змінився"**, а кожен **Observer сам вирішує, що робити** з цією інформацією. Точно так само працює кнопка "дзвіночок" 🔔 на YouTube — канал (Subject) публікує нове відео, і всі, хто натиснув "Підписатися" та ввімкнув сповіщення (Observers), отримують push-повідомлення, при цьому автор відео не веде окремого списку "хто саме" підписаний — це відповідальність платформи (реалізації патерну).

---

## Проблема без патерну

Розглянемо метеостанцію, яка отримує нові дані (температура, вологість) і повинна оновлювати кілька дисплеїв: телефонний застосунок, телевізійну панель та веб-сайт.

"Наївна" реалізація без патерну виглядає так:

```csharp
// ПОГАНИЙ ПІДХІД: WeatherStation напряму знає про КОНКРЕТНІ класи дисплеїв
public class PhoneDisplay
{
    public void Update(float temperature, float humidity)
    {
        Console.WriteLine($"[Телефон] Температура: {temperature}°C, Вологість: {humidity}%");
    }
}

public class TVDisplay
{
    public void Update(float temperature, float humidity)
    {
        Console.WriteLine($"[Телевізор] Зараз: {temperature}°C, вологість {humidity}%");
    }
}

public class WebDisplay
{
    public void Update(float temperature, float humidity)
    {
        Console.WriteLine($"[Веб-сайт] T={temperature}°C, H={humidity}%");
    }
}

public class WeatherStation
{
    // Проблема №1: WeatherStation ЖОРСТКО прив'язана до конкретних класів дисплеїв.
    // Вона має посилання на кожен конкретний тип, а не на абстракцію.
    private readonly PhoneDisplay _phoneDisplay = new();
    private readonly TVDisplay _tvDisplay = new();
    private readonly WebDisplay _webDisplay = new();

    private float _temperature;
    private float _humidity;

    public void SetMeasurements(float temperature, float humidity)
    {
        _temperature = temperature;
        _humidity = humidity;

        // Проблема №2: кожного разу, коли з'являються нові дані,
        // ми ВРУЧНУ викликаємо кожен дисплей поіменно.
        _phoneDisplay.Update(_temperature, _humidity);
        _tvDisplay.Update(_temperature, _humidity);
        _webDisplay.Update(_temperature, _humidity);

        // Проблема №3: щоб додати новий дисплей (наприклад, SmartWatchDisplay),
        // потрібно ЗМІНЮВАТИ КОД WeatherStation — додавати нове поле
        // і новий рядок виклику. Це порушує принцип Open/Closed
        // (клас має бути відкритим для розширення, але закритим для модифікації).

        // Проблема №4: неможливо динамічно "відписати" дисплей під час роботи
        // програми (наприклад, користувач закрив вкладку веб-сайту) —
        // немає механізму видалення підписника.

        // Проблема №5: WeatherStation відповідає одразу за ДВІ речі —
        // за зберігання даних погоди І за логіку сповіщення конкретних
        // споживачів. Це порушує принцип єдиної відповідальності (SRP).
    }
}
```

Усі ці проблеми вирішує патерн Observer: Subject працює лише з **абстракцією** `IObserver`, а не з конкретними класами, і підтримує **динамічний** список підписників.

---

## Структура патерну

```
┌────────────────────────────┐          ┌───────────────────────┐
│   <<interface>> Subject     │          │ <<interface>> Observer │
│   (Observable)               │          │                        │
├──────────────────────────────┤          ├────────────────────────┤
│ + Subscribe(observer)        │◇────────>│ + Update(data)          │
│ + Unsubscribe(observer)      │  0..*    └───────────▲────────────┘
│ + Notify()                   │                      │
└───────────────▲──────────────┘                      │ implements
                │ implements                          │
                │                          ┌───────────┴────────────┐
   ┌────────────┴─────────────┐            │    ConcreteObserverA    │
   │     ConcreteSubject       │            ├─────────────────────────┤
   ├────────────────────────────┤           │ + Update(data)          │
   │ - state                   │            │   // власна реакція     │
   │ + SetState(newState)      │            └─────────────────────────┘
   │   { state = newState;     │            ┌─────────────────────────┐
   │     Notify(); }           │            │    ConcreteObserverB    │
   │ - List<Observer> observers │            ├─────────────────────────┤
   └────────────────────────────┘           │ + Update(data)          │
                                              └─────────────────────────┘
```

**Потік взаємодії:**

1. `ConcreteObserverA` та `ConcreteObserverB` викликають `Subject.Subscribe(this)`, щоб зареєструватися.
2. Коли стан `ConcreteSubject` змінюється (`SetState`), він викликає внутрішній метод `Notify()`.
3. `Notify()` проходить по всьому списку зареєстрованих спостерігачів і викликає у кожного `Update(data)`.
4. Кожен `ConcreteObserver` сам вирішує, що робити з отриманими даними.

### Таблиця ролей

| Роль | Призначення | У прикладах нижче |
|---|---|---|
| **Subject / Observable** | Інтерфейс з методами `Subscribe`, `Unsubscribe`, `Notify` | `IWeatherSubject`, або `IObservable<T>` |
| **ConcreteSubject** | Зберігає стан, який цікавить спостерігачів; викликає сповіщення при зміні | `WeatherStation`, `Order` |
| **Observer** | Інтерфейс з методом `Update`, який реалізують усі підписники | `IWeatherObserver`, або `IObserver<T>` |
| **ConcreteObserver** | Конкретна реакція на зміну стану | `PhoneDisplay`, `EmailNotifierObserver` |

### Ідіоматичні реалізації в C#

C# має **вбудовану мовну підтримку** для патерну Observer, тому в реальному коді "класичну" версію з ручним списком спостерігачів використовують рідко:

1. **`event` + делегати** (`EventHandler`, `EventHandler<T>`, або власний `delegate`) — компілятор сам генерує механізм підписки (`+=`) / відписки (`-=`) та безпечного виклику. Це основний, найпоширеніший спосіб реалізації Observer у .NET.
2. **`IObservable<T>` / `IObserver<T>`** (простір імен `System`) — формалізований варіант патерну зі стандарту .NET, що лежить в основі **Reactive Extensions (Rx.NET)**. `IObservable<T>.Subscribe(IObserver<T> observer)` повертає `IDisposable`, виклик `Dispose()` якого і є відпискою — елегантне вирішення проблеми "забули відписатися" (див. розділ про антипатерни).
3. **`INotifyPropertyChanged`** — спеціалізований варіант Observer для сповіщення про зміну властивостей одного об'єкта, основа біндингу даних у WPF, MAUI, Avalonia, Blazor (патерн MVVM).

Нижче розглянемо всі ці варіанти на практиці.

---

## Приклад 1 — Класична реалізація через інтерфейси

Реалізуємо метеостанцію "по книзі GoF": власні інтерфейси `IWeatherObserver` та `IWeatherSubject`, ручний список підписників.

```csharp
using System;
using System.Collections.Generic;

// ==================== Observer ====================

// Інтерфейс спостерігача: усі, хто хоче отримувати оновлення погоди,
// повинні реалізувати цей метод.
public interface IWeatherObserver
{
    void Update(float temperature, float humidity, float pressure);
}

// ==================== Subject ====================

// Інтерфейс суб'єкта: визначає контракт підписки/відписки/сповіщення.
public interface IWeatherSubject
{
    void Subscribe(IWeatherObserver observer);
    void Unsubscribe(IWeatherObserver observer);
    void Notify();
}

// ==================== ConcreteSubject ====================

public class WeatherStation : IWeatherSubject
{
    // Список спостерігачів — Subject знає ЛИШЕ про абстракцію IWeatherObserver,
    // а не про конкретні класи дисплеїв.
    private readonly List<IWeatherObserver> _observers = new();

    private float _temperature;
    private float _humidity;
    private float _pressure;

    public void Subscribe(IWeatherObserver observer)
    {
        if (_observers.Contains(observer))
        {
            Console.WriteLine($"[WeatherStation] {observer.GetType().Name} вже підписаний.");
            return;
        }

        _observers.Add(observer);
        Console.WriteLine($"[WeatherStation] {observer.GetType().Name} підписався на оновлення.");
    }

    public void Unsubscribe(IWeatherObserver observer)
    {
        if (_observers.Remove(observer))
        {
            Console.WriteLine($"[WeatherStation] {observer.GetType().Name} відписався від оновлень.");
        }
    }

    public void Notify()
    {
        // Проходимо копією списку на випадок, якщо якийсь Observer
        // вирішить відписатися прямо всередині свого Update().
        foreach (var observer in new List<IWeatherObserver>(_observers))
        {
            observer.Update(_temperature, _humidity, _pressure);
        }
    }

    // Головний бізнес-метод: отримання нових вимірювань з датчиків.
    public void SetMeasurements(float temperature, float humidity, float pressure)
    {
        _temperature = temperature;
        _humidity = humidity;
        _pressure = pressure;

        Console.WriteLine($"\n[WeatherStation] Нові дані: {temperature}°C, {humidity}%, {pressure} гПа");
        Notify();
    }
}

// ==================== ConcreteObservers ====================

public class PhoneDisplay : IWeatherObserver
{
    public void Update(float temperature, float humidity, float pressure)
    {
        Console.WriteLine($"  📱 [Телефон] Зараз {temperature}°C, вологість {humidity}%");
    }
}

public class TVDisplay : IWeatherObserver
{
    public void Update(float temperature, float humidity, float pressure)
    {
        Console.WriteLine($"  📺 [Телевізор] Погода: {temperature}°C / вологість {humidity}% / тиск {pressure} гПа");
    }
}

public class WebDisplay : IWeatherObserver
{
    public void Update(float temperature, float humidity, float pressure)
    {
        Console.WriteLine($"  🌐 [Веб-сайт] T={temperature}°C, H={humidity}%, P={pressure} гПа");
    }
}
```

### Використання

```csharp
public static class Program1
{
    public static void Main()
    {
        var station = new WeatherStation();

        var phone = new PhoneDisplay();
        var tv = new TVDisplay();
        var web = new WebDisplay();

        // Підписуємо всіх трьох спостерігачів.
        station.Subscribe(phone);
        station.Subscribe(tv);
        station.Subscribe(web);

        station.SetMeasurements(22.5f, 65f, 1013f);

        // Веб-сайт "закрився" — відписуємо його.
        station.Unsubscribe(web);

        station.SetMeasurements(24.0f, 58f, 1011f);
    }
}
```

**Очікуваний консольний вивід:**

```
[WeatherStation] PhoneDisplay підписався на оновлення.
[WeatherStation] TVDisplay підписався на оновлення.
[WeatherStation] WebDisplay підписався на оновлення.

[WeatherStation] Нові дані: 22.5°C, 65%, 1013 гПа
  📱 [Телефон] Зараз 22.5°C, вологість 65%
  📺 [Телевізор] Погода: 22.5°C / вологість 65% / тиск 1013 гПа
  🌐 [Веб-сайт] T=22.5°C, H=65%, P=1013 гПа
[WeatherStation] WebDisplay відписався від оновлень.

[WeatherStation] Нові дані: 24°C, 58%, 1011 гПа
  📱 [Телефон] Зараз 24°C, вологість 58%
  📺 [Телевізор] Погода: 24°C / вологість 58% / тиск 1011 гПа
```

Зверніть увагу: після `Unsubscribe(web)` веб-дисплей більше не отримує оновлень, а `WeatherStation` при цьому взагалі не змінювалась — весь механізм підписки повністю динамічний.

---

## Приклад 2 — Ідіоматичний C# через event/EventHandler

Тепер реалізуємо **той самий приклад**, але скориставшись вбудованим механізмом `event` — це те, як Observer реалізують у "бойовому" C#-коді 95% часу.

```csharp
using System;

// ==================== EventArgs ====================

// Замість того, щоб передавати три окремі float-параметри,
// інкапсулюємо дані події в незмінний (immutable) клас-снапшот.
public class WeatherChangedEventArgs : EventArgs
{
    public float Temperature { get; }
    public float Humidity { get; }
    public float Pressure { get; }

    public WeatherChangedEventArgs(float temperature, float humidity, float pressure)
    {
        Temperature = temperature;
        Humidity = humidity;
        Pressure = pressure;
    }
}

// ==================== ConcreteSubject ====================

public class WeatherStationEvents
{
    // Це і є весь механізм Subject-а: компілятор C# сам згенерує
    // приховане поле-делегат, метод add/remove (підписка/відписка)
    // та безпечну семантику виклику. Жодного List<Observer> писати не треба.
    public event EventHandler<WeatherChangedEventArgs>? WeatherChanged;

    public void SetMeasurements(float temperature, float humidity, float pressure)
    {
        Console.WriteLine($"\n[WeatherStation] Нові дані: {temperature}°C, {humidity}%, {pressure} гПа");

        // "?.Invoke" — стандартна ідіома безпечного виклику події:
        // якщо жоден спостерігач не підписаний, WeatherChanged == null,
        // і виклику просто не станеться (без NullReferenceException).
        WeatherChanged?.Invoke(this, new WeatherChangedEventArgs(temperature, humidity, pressure));
    }
}

// ==================== "Observers" — тепер це прості методи ====================

public class PhoneDisplayEvents
{
    // Підписник більше не зобов'язаний реалізовувати спеціальний інтерфейс —
    // достатньо мати метод із сигнатурою EventHandler<WeatherChangedEventArgs>.
    public void OnWeatherChanged(object? sender, WeatherChangedEventArgs e)
    {
        Console.WriteLine($"  📱 [Телефон] Зараз {e.Temperature}°C, вологість {e.Humidity}%");
    }
}

public class TVDisplayEvents
{
    public void OnWeatherChanged(object? sender, WeatherChangedEventArgs e)
    {
        Console.WriteLine($"  📺 [Телевізор] Погода: {e.Temperature}°C / тиск {e.Pressure} гПа");
    }
}
```

### Використання

```csharp
public static class Program2
{
    public static void Main()
    {
        var station = new WeatherStationEvents();
        var phone = new PhoneDisplayEvents();
        var tv = new TVDisplayEvents();

        // Підписка — це просто оператор "+=" над подією.
        station.WeatherChanged += phone.OnWeatherChanged;
        station.WeatherChanged += tv.OnWeatherChanged;

        // Можна підписати навіть анонімний лямбда-обробник без окремого класу.
        station.WeatherChanged += (sender, e) =>
            Console.WriteLine($"  🌐 [Веб-сайт, lambda] T={e.Temperature}°C");

        station.SetMeasurements(22.5f, 65f, 1013f);

        // Відписка — оператор "-=".
        station.WeatherChanged -= tv.OnWeatherChanged;

        station.SetMeasurements(24.0f, 58f, 1011f);
    }
}
```

**Очікуваний консольний вивід:**

```
[WeatherStation] Нові дані: 22.5°C, 65%, 1013 гПа
  📱 [Телефон] Зараз 22.5°C, вологість 65%
  📺 [Телевізор] Погода: 22.5°C / тиск 1013 гПа
  🌐 [Веб-сайт, lambda] T=22.5°C

[WeatherStation] Нові дані: 24°C, 58%, 1011 гПа
  📱 [Телефон] Зараз 24°C, вологість 58%
  🌐 [Веб-сайт, lambda] T=24°C
```

### Порівняння з Прикладом 1

| | Класичний Observer (Приклад 1) | `event`-версія (Приклад 2) |
|---|---|---|
| Потрібен окремий інтерфейс `IObserver`? | Так | Ні — достатньо сигнатури делегата |
| Ручний `List<IObserver>` | Так, пишемо самі | Ні, генерується компілятором |
| Підписка / відписка | `Subscribe()` / `Unsubscribe()` | `+=` / `-=` |
| Безпека виклику при 0 підписниках | Потрібна власна перевірка | `?.Invoke(...)` — вбудована ідіома |
| Підписка лямбдою "на льоту" | Незручно (потрібен клас-обгортка) | Природно |
| Рядків коду | Більше | Менше |
| Коли обирати | Навчальні приклади, крос-платформний код без C#, особливі вимоги (напр. пріоритети виклику, кастомний список) | 95% реальних задач у .NET |

---

## Приклад 3 — INotifyPropertyChanged для MVVM

`INotifyPropertyChanged` — це стандартизований варіант Observer із простору імен `System.ComponentModel`, який .NET-фреймворки (WPF, MAUI, Blazor, Avalonia) використовують для **прив'язки даних (data binding)** у патерні MVVM. Об'єкт-модель (ViewModel) сповіщає підписників (найчастіше — рушій біндингу UI) про зміну конкретної властивості.

```csharp
using System;
using System.ComponentModel;
using System.Runtime.CompilerServices;

// ==================== ConcreteSubject (ViewModel) ====================

public class UserProfile : INotifyPropertyChanged
{
    // Стандартна подія з інтерфейсу INotifyPropertyChanged.
    public event PropertyChangedEventHandler? PropertyChanged;

    private string _displayName = string.Empty;
    private string _status = "офлайн";

    public string DisplayName
    {
        get => _displayName;
        set
        {
            if (_displayName == value) return; // не сповіщаємо, якщо значення не змінилось
            _displayName = value;
            OnPropertyChanged();
        }
    }

    public string Status
    {
        get => _status;
        set
        {
            if (_status == value) return;
            _status = value;
            OnPropertyChanged();
        }
    }

    // [CallerMemberName] автоматично підставляє ім'я властивості,
    // з якої був викликаний метод — не треба писати nameof() вручну щоразу.
    protected void OnPropertyChanged([CallerMemberName] string? propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}

// ==================== ConcreteObservers ("UI") ====================

public class ProfileHeaderUI
{
    public ProfileHeaderUI(UserProfile profile)
    {
        profile.PropertyChanged += OnProfileChanged;
    }

    private void OnProfileChanged(object? sender, PropertyChangedEventArgs e)
    {
        // Реагуємо ЛИШЕ на конкретну властивість — типовий патерн для UI.
        if (e.PropertyName == nameof(UserProfile.DisplayName))
        {
            var profile = (UserProfile)sender!;
            Console.WriteLine($"  🖼️ [Header UI] Оновлюю заголовок сторінки на: \"{profile.DisplayName}\"");
        }
    }
}

public class StatusIndicatorUI
{
    public StatusIndicatorUI(UserProfile profile)
    {
        profile.PropertyChanged += OnProfileChanged;
    }

    private void OnProfileChanged(object? sender, PropertyChangedEventArgs e)
    {
        if (e.PropertyName == nameof(UserProfile.Status))
        {
            var profile = (UserProfile)sender!;
            var icon = profile.Status == "онлайн" ? "🟢" : "⚪";
            Console.WriteLine($"  {icon} [Status Indicator] Новий статус: {profile.Status}");
        }
    }
}
```

### Використання

```csharp
public static class Program3
{
    public static void Main()
    {
        var profile = new UserProfile();

        // Обидва UI-компоненти підписуються в конструкторі —
        // типово для реального MVVM-коду (View підписується на ViewModel).
        var header = new ProfileHeaderUI(profile);
        var statusIndicator = new StatusIndicatorUI(profile);

        Console.WriteLine("Змінюємо ім'я користувача:");
        profile.DisplayName = "Олена Коваль";

        Console.WriteLine("\nЗмінюємо статус:");
        profile.Status = "онлайн";

        Console.WriteLine("\nЗмінюємо і ім'я, і статус одночасно:");
        profile.DisplayName = "Олена К.";
        profile.Status = "офлайн";

        Console.WriteLine("\nПовторне присвоєння того ж значення (сповіщення НЕ буде):");
        profile.Status = "офлайн";
    }
}
```

**Очікуваний консольний вивід:**

```
Змінюємо ім'я користувача:
  🖼️ [Header UI] Оновлюю заголовок сторінки на: "Олена Коваль"

Змінюємо статус:
  🟢 [Status Indicator] Новий статус: онлайн

Змінюємо і ім'я, і статус одночасно:
  🖼️ [Header UI] Оновлюю заголовок сторінки на: "Олена К."
  ⚪ [Status Indicator] Новий статус: офлайн

Повторне присвоєння того ж значення (сповіщення НЕ буде):
```

Важливий момент: обидва спостерігачі підписані на **одну й ту саму подію** `PropertyChanged`, але кожен фільтрує повідомлення за `e.PropertyName` і реагує лише на "свою" властивість — так реалізовано вибіркову реакцію без окремих подій для кожного поля.

---

## Приклад 4 (реальний сценарій) — Система сповіщень про статус замовлення

Розглянемо реалістичний e-commerce сценарій: замовлення (`Order`) проходить через статуси **Created → Paid → Shipped → Delivered**. На кожну зміну статусу повинні незалежно реагувати кілька сервісів:

- `EmailNotifierObserver` — надсилає email клієнту.
- `SmsNotifierObserver` — надсилає SMS.
- `WarehouseObserver` — коли замовлення оплачене, починає пакування на складі.
- `AnalyticsObserver` — фіксує метрики для бізнес-аналітики.

Також продемонструємо дві "підводні каміння" реального коду:
1. Підписку/відписку спостерігача **під час виконання** (наприклад, SMS вимкнули в налаштуваннях користувача).
2. Ситуацію, коли **один спостерігач кидає виняток**, а решта повинні однаково отримати сповіщення (типова помилка — див. розділ про антипатерни; тут показано, як зробити правильно).

```csharp
using System;
using System.Collections.Generic;

// ==================== Допоміжні типи ====================

public enum OrderStatus
{
    Created,
    Paid,
    Shipped,
    Delivered
}

// Незмінний знімок даних події — спостерігачі отримують ТІЛЬКИ читання,
// а не посилання на внутрішній mutable-стан Order.
public sealed class OrderStatusChangedEventArgs : EventArgs
{
    public string OrderId { get; }
    public OrderStatus OldStatus { get; }
    public OrderStatus NewStatus { get; }
    public DateTime ChangedAtUtc { get; }

    public OrderStatusChangedEventArgs(string orderId, OrderStatus oldStatus, OrderStatus newStatus)
    {
        OrderId = orderId;
        OldStatus = oldStatus;
        NewStatus = newStatus;
        ChangedAtUtc = DateTime.UtcNow;
    }
}

// Інтерфейс спостерігача для цього домену — трохи багатший за просту
// сигнатуру EventHandler, бо у нас декілька видів сповіщень (гнучкіше
// для розширення в реальному проєкті: логування, метрики виклику тощо).
public interface IOrderObserver
{
    string Name { get; }
    void OnOrderStatusChanged(OrderStatusChangedEventArgs e);
}

// ==================== ConcreteSubject ====================

public class Order
{
    private readonly List<IOrderObserver> _observers = new();

    public string Id { get; }
    public OrderStatus Status { get; private set; } = OrderStatus.Created;

    public Order(string id)
    {
        Id = id;
    }

    public void Subscribe(IOrderObserver observer)
    {
        _observers.Add(observer);
        Console.WriteLine($"[Order {Id}] Спостерігач '{observer.Name}' підписався.");
    }

    public void Unsubscribe(IOrderObserver observer)
    {
        if (_observers.Remove(observer))
        {
            Console.WriteLine($"[Order {Id}] Спостерігач '{observer.Name}' відписався.");
        }
    }

    public void ChangeStatus(OrderStatus newStatus)
    {
        var oldStatus = Status;
        Status = newStatus;

        Console.WriteLine($"\n[Order {Id}] Статус змінено: {oldStatus} → {newStatus}");

        var eventArgs = new OrderStatusChangedEventArgs(Id, oldStatus, newStatus);
        Notify(eventArgs);
    }

    private void Notify(OrderStatusChangedEventArgs e)
    {
        // Копіюємо список ПЕРЕД ітерацією: якщо один зі спостерігачів
        // відпишеться (або підпише когось нового) прямо у своєму
        // OnOrderStatusChanged, це не призведе до InvalidOperationException
        // ("Collection was modified") і не спотворить поточний прохід.
        foreach (var observer in new List<IOrderObserver>(_observers))
        {
            try
            {
                observer.OnOrderStatusChanged(e);
            }
            catch (Exception ex)
            {
                // КЛЮЧОВИЙ МОМЕНТ: якщо один спостерігач впаде з винятком,
                // це НЕ повинно завадити сповіщенню решти спостерігачів.
                // Логуємо помилку і продовжуємо цикл.
                Console.WriteLine($"  ⚠️ Спостерігач '{observer.Name}' кинув виняток: {ex.Message}. Продовжуємо сповіщення інших.");
            }
        }
    }
}

// ==================== ConcreteObservers ====================

public class EmailNotifierObserver : IOrderObserver
{
    public string Name => "EmailNotifier";

    public void OnOrderStatusChanged(OrderStatusChangedEventArgs e)
    {
        Console.WriteLine($"  📧 [Email] Клієнту надіслано лист: замовлення {e.OrderId} тепер '{e.NewStatus}'.");
    }
}

public class SmsNotifierObserver : IOrderObserver
{
    public string Name => "SmsNotifier";

    public void OnOrderStatusChanged(OrderStatusChangedEventArgs e)
    {
        // Навмисно імітуємо збій зовнішнього SMS-провайдера при статусі Shipped,
        // щоб продемонструвати стійкість Notify() до винятків одного спостерігача.
        if (e.NewStatus == OrderStatus.Shipped)
        {
            throw new InvalidOperationException("SMS-провайдер тимчасово недоступний (timeout)");
        }

        Console.WriteLine($"  📱 [SMS] Відправлено SMS: замовлення {e.OrderId} — '{e.NewStatus}'.");
    }
}

public class WarehouseObserver : IOrderObserver
{
    public string Name => "Warehouse";

    public void OnOrderStatusChanged(OrderStatusChangedEventArgs e)
    {
        // Складу цікавий лише один конкретний перехід статусу.
        if (e.NewStatus == OrderStatus.Paid)
        {
            Console.WriteLine($"  📦 [Склад] Замовлення {e.OrderId} оплачено — розпочинаємо пакування.");
        }
    }
}

public class AnalyticsObserver : IOrderObserver
{
    public string Name => "Analytics";
    private readonly Dictionary<OrderStatus, int> _transitionCounts = new();

    public void OnOrderStatusChanged(OrderStatusChangedEventArgs e)
    {
        _transitionCounts.TryGetValue(e.NewStatus, out var count);
        _transitionCounts[e.NewStatus] = count + 1;

        Console.WriteLine($"  📊 [Analytics] Зафіксовано перехід у '{e.NewStatus}' (усього таких переходів: {_transitionCounts[e.NewStatus]}).");
    }
}
```

### Використання

```csharp
public static class Program4
{
    public static void Main()
    {
        var order = new Order("ORD-1042");

        var email = new EmailNotifierObserver();
        var sms = new SmsNotifierObserver();
        var warehouse = new WarehouseObserver();
        var analytics = new AnalyticsObserver();

        order.Subscribe(email);
        order.Subscribe(sms);
        order.Subscribe(warehouse);
        order.Subscribe(analytics);

        // Крок 1: замовлення оплачене — Warehouse почне пакування.
        order.ChangeStatus(OrderStatus.Paid);

        // Крок 2: замовлення відправлене — SMS-провайдер "падає" з винятком,
        // але Email, Warehouse (не реагує на цей статус) та Analytics
        // все одно повинні коректно відпрацювати.
        order.ChangeStatus(OrderStatus.Shipped);

        // Клієнт вимкнув SMS-сповіщення в налаштуваннях профілю —
        // демонструємо динамічну відписку під час роботи програми.
        order.Unsubscribe(sms);

        // Крок 3: замовлення доставлене.
        order.ChangeStatus(OrderStatus.Delivered);
    }
}
```

**Очікуваний консольний вивід:**

```
[Order ORD-1042] Спостерігач 'EmailNotifier' підписався.
[Order ORD-1042] Спостерігач 'SmsNotifier' підписався.
[Order ORD-1042] Спостерігач 'Warehouse' підписався.
[Order ORD-1042] Спостерігач 'Analytics' підписався.

[Order ORD-1042] Статус змінено: Created → Paid
  📧 [Email] Клієнту надіслано лист: замовлення ORD-1042 тепер 'Paid'.
  📱 [SMS] Відправлено SMS: замовлення ORD-1042 — 'Paid'.
  📦 [Склад] Замовлення ORD-1042 оплачено — розпочинаємо пакування.
  📊 [Analytics] Зафіксовано перехід у 'Paid' (усього таких переходів: 1).

[Order ORD-1042] Статус змінено: Paid → Shipped
  📧 [Email] Клієнту надіслано лист: замовлення ORD-1042 тепер 'Shipped'.
  ⚠️ Спостерігач 'SmsNotifier' кинув виняток: SMS-провайдер тимчасово недоступний (timeout). Продовжуємо сповіщення інших.
  📊 [Analytics] Зафіксовано перехід у 'Shipped' (усього таких переходів: 1).
[Order ORD-1042] Спостерігач 'SmsNotifier' відписався.

[Order ORD-1042] Статус змінено: Shipped → Delivered
  📧 [Email] Клієнту надіслано лист: замовлення ORD-1042 тепер 'Delivered'.
  📊 [Analytics] Зафіксовано перехід у 'Delivered' (усього таких переходів: 1).
```

Зверніть увагу на два важливі моменти:

1. Коли `SmsNotifierObserver` кидає виняток на кроці `Shipped`, **`WarehouseObserver` та `AnalyticsObserver` все одно отримують сповіщення** — завдяки `try/catch` всередині циклу `Notify()`, а не навколо нього.
2. `WarehouseObserver` взагалі не виводить нічого при `Shipped` та `Delivered` — це нормально, він **сам вирішує**, на які саме статуси реагувати. Subject про цю логіку нічого не знає.

---

## Observer vs Mediator vs Pub-Sub

Ці три підходи часто плутають, бо всі вони про "сповіщення" та "зменшення зв'язності", але суть різна.

### Observer — однонаправлена трансляція

```
Subject ──notify──▶ Observer A
   │
   └──────notify──▶ Observer B
```

Subject **напряму тримає посилання** на своїх Observer-ів (список підписників) і викликає їх методи. Subject **не знає і не повинен знати**, що саме Observer робить у відповідь — це "запусти і забудь" (fire-and-forget) сповіщення в один бік.

### Mediator — центральний координатор двосторонньої взаємодії

```
Colleague A ──▶ Mediator ◀── Colleague B
                   │  ▲
                   ▼  │
              Colleague C
```

**Mediator** централізує складну логіку **взаємної** координації між об'єктами (Colleagues), які **знають про сам Mediator** (кожен колега тримає посилання на медіатора) і звертаються до нього, коли їм потрібно щось зробити або сповістити інших. Медіатор вирішує, кого і як сповістити, часто з нетривіальною логікою ("якщо А зробив X, то Б треба заблокувати, а В — оновити"). Це вирішує проблему "все спілкується з усім" (M×M зв'язків перетворюються на M зв'язків із медіатором).

### Класичний Pub/Sub (через брокер повідомлень) — повна відв'язаність

```
Publisher ──▶ [ Message Broker / Topic ] ──▶ Subscriber A
                       │
                       └──────────────────▶ Subscriber B
```

У класичному Pub/Sub (наприклад, RabbitMQ, Kafka, Azure Service Bus) видавець (**Publisher**) **навіть не має посилання** на підписників — і підписники не мають посилання на видавця. Між ними стоїть **брокер** (проміжна інфраструктура, часто окремий процес чи навіть окремий сервер), який приймає повідомлення від видавця та розподіляє їх підписникам певної теми (topic/queue). Видавець і підписник можуть навіть **не працювати одночасно** (повідомлення чекає в черзі).

### Порівняльна таблиця

| Критерій | Observer | Mediator | Класичний Pub/Sub (брокер) |
|---|---|---|---|
| Хто кого знає | Subject знає список Observer-ів напряму | Colleague-и знають про Mediator (а він — про них) | Publisher і Subscriber не знають одне одного |
| Напрямок взаємодії | Одностороннє сповіщення | Двостороння координація | Одностороннє сповіщення через посередника |
| Проміжна інфраструктура | Не потрібна (виклик "у процесі") | Не потрібна (об'єкт у тій же програмі) | Обов'язкова (брокер, черга, топік) |
| Типовий масштаб | Клас/модуль всередині одного застосунку | Клас/модуль всередині одного застосунку | Розподілена система, кілька сервісів/процесів |
| Асинхронність, персистентність | Зазвичай синхронний виклик у пам'яті | Зазвичай синхронний виклик у пам'яті | Часто асинхронно, з чергою, можливою затримкою |
| Мета | Розсилка "стан змінився" усім зацікавленим | Позбутися зв'язків "кожен з кожним" | Повна незалежність систем/сервісів одна від одної |

### Запитай себе:

- **"Чи Subject напряму тримає список тих, кого сповіщає?"** — так → це Observer (чи його різновид).
- **"Чи є об'єкт, який знає ПРО ВСІХ учасників і вирішує складну логіку взаємодії між ними (а не просто розсилає одне повідомлення всім однаково)?"** — так → це Mediator.
- **"Чи видавець взагалі не знає, хто підписники, і між ними стоїть окрема інфраструктура (черга, топік, брокер), можливо навіть інший процес чи сервер?"** — так → це класичний Pub/Sub.
- Простий орієнтир: **Observer — це Pub/Sub у мініатюрі, в межах одного процесу, без брокера.** Різниця між ними — ступінь відв'язаності, а не інша ідея.

---

## Переваги та недоліки

### Переваги

- **Слабка зв'язність (loose coupling)** — Subject знає лише про абстракцію `Observer`/`event`, а не про конкретні класи спостерігачів.
- **Динамічна підписка/відписка** — спостерігачів можна додавати й видаляти в будь-який момент виконання програми, без перезапуску чи перекомпіляції.
- **Широкомовне сповіщення (broadcast)** — один виклик `Notify()`/один `Invoke()` дістає одразу всіх зацікавлених сторін.
- **Відповідність Open/Closed Principle** — щоб додати новий тип реакції на подію, достатньо створити новий клас-спостерігач і підписати його; код Subject-а лишається незмінним.
- **Розділення відповідальності** — Subject відповідає лише за власний стан і факт сповіщення, а кожен Observer — за власну логіку реакції.

### Недоліки

- **Порядок сповіщення непередбачуваний/неконтрольований за замовчуванням** — стандартна реалізація не гарантує і не дає легко керувати тим, у якому порядку викликаються спостерігачі (зазвичай — порядок підписки, але покладатися на це небезпечно).
- **"Повільний" або зациклений спостерігач сповільнює або (без обробки помилок) ламає весь ланцюжок** — якщо `Notify()` викликає спостерігачів синхронно і один з них виконується довго чи кидає необроблений виняток, це може заблокувати або перервати сповіщення решти (див. Приклад 4 та розділ про антипатерни).
- **Ризик витоків пам'яті ("lapsed listener")** — якщо Observer підписався на довгоживучий Subject, але забув відписатися, Subject тримає на нього посилання й не дає збирачу сміття звільнити пам'ять, навіть коли Observer логічно вже "не потрібен".
- **Каскади непередбачуваних оновлень** — якщо Observer у відповідь на сповіщення сам змінює якийсь спільний стан, це може викликати ланцюгову реакцію нових сповіщень, яку важко відстежити й налагодити (особливо при циклічних залежностях між Subject-ами).

---

## Антипатерни та поширені помилки

### Помилка 1: забули відписатися → витік пам'яті ("lapsed listener")

Короткоживучий об'єкт (наприклад, UI-контрол, який користувач може закрити) підписується на довгоживучий Subject (наприклад, глобальний сервіс, singleton), але ніколи не відписується. Subject продовжує тримати на нього посилання — GC не може звільнити пам'ять навіть після того, як контрол логічно вже не потрібен.

```csharp
// ==================== НЕПРАВИЛЬНО ====================

public class NotificationCenter
{
    // Живе стільки ж, скільки й весь застосунок (singleton-подібний сервіс).
    public event EventHandler<string>? MessageReceived;

    public void Publish(string message) => MessageReceived?.Invoke(this, message);
}

public class ToastPopup
{
    // ToastPopup — короткоживучий об'єкт: створюється й знищується
    // щоразу, коли показується спливаюче повідомлення.
    public ToastPopup(NotificationCenter center)
    {
        center.MessageReceived += OnMessageReceived; // підписались...

        // ...але немає ЖОДНОГО методу Dispose/Close, який би викликав
        // "-=". Коли користувач закриє ToastPopup, сам ОБ'ЄКТ логічно
        // повинен зникнути, але NotificationCenter продовжує тримати
        // на нього посилання через делегат MessageReceived —
        // GC не звільнить пам'ять ToastPopup, доки NotificationCenter живий!
    }

    private void OnMessageReceived(object? sender, string message)
    {
        Console.WriteLine($"Toast: {message}");
    }
}
```

```csharp
// ==================== ПРАВИЛЬНО (варіант 1: явна відписка через IDisposable) ====================

public class ToastPopup : IDisposable
{
    private readonly NotificationCenter _center;

    public ToastPopup(NotificationCenter center)
    {
        _center = center;
        _center.MessageReceived += OnMessageReceived;
    }

    private void OnMessageReceived(object? sender, string message)
    {
        Console.WriteLine($"Toast: {message}");
    }

    // Реалізуємо IDisposable, щоб гарантовано відписатися при знищенні.
    public void Dispose()
    {
        _center.MessageReceived -= OnMessageReceived;
    }
}

// Використання: `using var toast = new ToastPopup(center);`
// — компілятор гарантує виклик Dispose() (а отже, і "-="), коли toast
// виходить з області видимості.
```

```csharp
// ==================== ПРАВИЛЬНО (варіант 2: слабке посилання, коли явну відписку забезпечити важко) ====================

using System;

public class WeakEventSubscription<TEventArgs>
{
    // Зберігаємо СЛАБКЕ посилання на отримувача — воно не заважає GC
    // зібрати об'єкт, навіть якщо той "забув" відписатися.
    private readonly WeakReference<Action<TEventArgs>> _weakHandler;

    public WeakEventSubscription(Action<TEventArgs> handler)
    {
        _weakHandler = new WeakReference<Action<TEventArgs>>(handler);
    }

    public bool TryInvoke(TEventArgs args)
    {
        if (_weakHandler.TryGetTarget(out var handler))
        {
            handler(args);
            return true; // отримувач ще живий
        }

        return false; // отримувач вже зібраний GC — підписку можна видалити зі списку
    }
}

// Ідея: Subject періодично (або перед кожним Notify) прибирає зі свого
// списку ті підписки, чий TryInvoke повернув false. Це "самоочищувана"
// підписка — типовий підхід у бібліотеках, де гарантувати виклик
// Dispose/Unsubscribe з боку клієнтського коду неможливо.
// Примітка: у більшості звичайних застосунків простіше й надійніше
// користуватись Варіантом 1 (явний IDisposable) — слабкі посилання
// додають складності і застосовуються, коли явна відписка справді
// нездійсненна (наприклад, у деяких сценаріях плагінів).
```

### Помилка 2: виняток в одному спостерігачі перериває сповіщення решти

```csharp
// ==================== НЕПРАВИЛЬНО ====================

public class BadSubject
{
    private readonly List<Action<string>> _observers = new();

    public void Subscribe(Action<string> observer) => _observers.Add(observer);

    public void Notify(string data)
    {
        // Якщо ХОЧ ОДИН обробник кине виняток, foreach негайно перерветься,
        // і всі спостерігачі, що йдуть далі у списку, НЕ отримають сповіщення
        // взагалі — навіть не дізнаються, що подія відбулась!
        foreach (var observer in _observers)
        {
            observer(data); // немає try/catch
        }
    }
}
```

```csharp
// ==================== ПРАВИЛЬНО ====================

public class GoodSubject
{
    private readonly List<Action<string>> _observers = new();

    public void Subscribe(Action<string> observer) => _observers.Add(observer);

    public void Notify(string data)
    {
        foreach (var observer in new List<Action<string>>(_observers))
        {
            try
            {
                // try/catch ВСЕРЕДИНІ циклу (навколо КОЖНОГО виклику),
                // а не навколо всього циклу — інакше перший же виняток
                // все одно перерве обробку решти спостерігачів.
                observer(data);
            }
            catch (Exception ex)
            {
                // Логуємо і продовжуємо — один "поганий" спостерігач
                // не повинен впливати на решту.
                Console.WriteLine($"Помилка в обробнику: {ex.Message}");
            }
        }
    }
}
```

### Помилка 3: Subject передає повний mutable-стан замість незмінного знімка

```csharp
// ==================== НЕПРАВИЛЬНО ====================

public class MutableOrderState
{
    // Публічні сеттери роблять об'єкт повністю mutable.
    public string Status { get; set; } = "Created";
    public decimal Total { get; set; }
}

public class BadOrder
{
    // Subject напряму віддає посилання на СВІЙ ВНУТРІШНІЙ стан.
    public MutableOrderState State { get; } = new();

    public event Action<MutableOrderState>? Changed;

    public void MarkAsPaid()
    {
        State.Status = "Paid";
        Changed?.Invoke(State); // передаємо посилання на mutable-об'єкт!
    }
}

public class SneakyObserver
{
    public void React(MutableOrderState state)
    {
        // Спостерігач може (навмисно чи випадково) ЗМІНИТИ стан,
        // який належить Subject-у! Це ламає інкапсуляцію: інші
        // спостерігачі побачать змінений іншим спостерігачем стан,
        // а Subject навіть не дізнається, що хтось "втрутився".
        state.Status = "Cancelled"; // 💥 непередбачувана побічна дія
    }
}
```

```csharp
// ==================== ПРАВИЛЬНО ====================

// Незмінний (immutable) record — знімок стану на момент події.
// Спостерігач фізично не може змінити те, що бачить Subject,
// бо у record немає сеттерів (init-only властивості).
public sealed record OrderStateSnapshot(string Status, decimal Total, DateTime CapturedAtUtc);

public class GoodOrder
{
    private string _status = "Created";
    private decimal _total;

    public event Action<OrderStateSnapshot>? Changed;

    public void MarkAsPaid()
    {
        _status = "Paid";

        // Передаємо НЕЗМІННИЙ знімок — а не посилання на внутрішній стан.
        Changed?.Invoke(new OrderStateSnapshot(_status, _total, DateTime.UtcNow));
    }
}

public class SafeObserver
{
    public void React(OrderStateSnapshot snapshot)
    {
        // snapshot.Status = "Cancelled"; // ← ТАК НЕ СКОМПІЛЮЄТЬСЯ:
        // init-only властивість record-а не можна змінити ззовні.
        Console.WriteLine($"Отримано знімок стану: {snapshot.Status}, сума {snapshot.Total}");
    }
}
```

---

## Підсумок

Патерн **Observer** варто застосовувати, коли:

- Зміна стану одного об'єкта повинна автоматично відображатись у довільній (наперед невідомій) кількості інших об'єктів.
- Ви хочете, щоб Subject і Observer-и були **слабко зв'язані** — Subject не повинен знати конкретні класи своїх підписників.
- Кількість і склад підписників можуть **змінюватись динамічно** під час роботи програми.
- Потрібна реалізація патернів **MVVM/MVC**, **event-driven архітектури** в межах одного процесу, реакції UI на зміну даних моделі.

Не варто застосовувати (або варто застосовувати обережно), коли:

- Потрібна **гарантія порядку** виконання реакцій або **транзакційність** (усі повинні виконатись успішно, або жодна) — Observer цього не гарантує "з коробки".
- Спостерігачів і видавців багато, і вони належать **різним сервісам/процесам** — тоді краще розглянути повноцінний message broker (Pub/Sub-інфраструктуру), а не Observer у пам'яті.
- Взаємодія між об'єктами **двостороння й складна** (об'єкти повинні координувати дії одне з одним, а не просто отримувати сповіщення) — тоді підійде **Mediator**.

### Мінімальний шаблон

**Класичний, інтерфейсний Observer:**

```csharp
public interface IObserver<TData>
{
    void Update(TData data);
}

public interface ISubject<TData>
{
    void Subscribe(IObserver<TData> observer);
    void Unsubscribe(IObserver<TData> observer);
}

public class Subject<TData> : ISubject<TData>
{
    private readonly List<IObserver<TData>> _observers = new();

    public void Subscribe(IObserver<TData> observer) => _observers.Add(observer);
    public void Unsubscribe(IObserver<TData> observer) => _observers.Remove(observer);

    protected void Notify(TData data)
    {
        foreach (var observer in new List<IObserver<TData>>(_observers))
        {
            try { observer.Update(data); }
            catch (Exception ex) { Console.WriteLine($"Помилка спостерігача: {ex.Message}"); }
        }
    }
}
```

**Ідіоматичний C# (event-based) Observer:**

```csharp
public class Subject
{
    public event EventHandler<EventArgs>? Changed;

    protected void OnChanged(EventArgs e) => Changed?.Invoke(this, e);
}

// Підписка:  subject.Changed += (sender, e) => { /* реакція */ };
// Відписка:  subject.Changed -= handlerReference;
```

---

*Документ підготовлено для вивчення патернів проектування. Всі приклади протестовані на .NET 6+.*
