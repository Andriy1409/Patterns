# Патерн Command (Команда) — Детальний розбір на C#

> **Категорія:** Поведінковий (Behavioral)  
> **Автори:** Gang of Four (GoF)  
> **Мова прикладів:** C#

---

## Зміст

1. [Що таке Command?](#що-таке-command)
   - [Аналогія з реального світу](#аналогія-з-реального-світу)
2. [Проблема без патерну](#проблема-без-патерну)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Пульт керування освітленням](#приклад-1--пульт-керування-освітленням)
5. [Приклад 2 — Undo/Redo в текстовому редакторі](#приклад-2--undoredo-в-текстовому-редакторі)
6. [Приклад 3 — Макрокоманди (Composite Command)](#приклад-3--макрокоманди-composite-command)
7. [Приклад 4 (реальний сценарій) — Черга фонових завдань](#приклад-4-реальний-сценарій--черга-фонових-завдань)
8. [Command vs Strategy vs Memento](#command-vs-strategy-vs-memento)
9. [Переваги та недоліки](#переваги-та-недоліки)
10. [Антипатерни та поширені помилки](#антипатерни-та-поширені-помилки)
11. [Підсумок](#підсумок)

---

## Що таке Command?

**Command (Команда)** — це поведінковий патерн проектування, який перетворює **запит або дію на самостійний об'єкт**, що містить усю інформацію про цей запит.

Замість того, щоб викликати метод напряму (`receiver.DoSomething()`), ми загортаємо цей виклик в окремий об'єкт-команду з єдиним методом `Execute()`. Це перетворення дає нам кілька важливих можливостей:

- **Параметризація** — команду можна передавати як параметр, зберігати в змінній, повертати з методу
- **Черги (queuing)** — команди можна ставити в чергу і виконувати пізніше, в іншому потоці чи навіть на іншій машині
- **Логування** — кожен виклик можна записати в журнал ще до чи після виконання
- **Скасування (undo/redo)** — якщо команда зберігає достатньо інформації, вона може відкотити свою дію

Головна ідея: **дія стає даними**. Те, що раніше було просто викликом методу, тепер — об'єкт, яким можна маніпулювати як будь-яким іншим об'єктом у програмі.

### Аналогія з реального світу

Уявіть **ресторан**. Офіціант підходить до столика, приймає замовлення і записує його на **бланк замовлення** (order slip): "Стіл №5: борщ, котлета по-київськи, компот".

Цей бланк — і є команда:

- Офіціант (**Invoker**, ініціатор) не готує їжу сам — він просто передає бланк далі
- Кухня (**Receiver**, отримувач) отримує бланк і виконує фактичну роботу — готує страви
- Бланк (**Command**, команда) інкапсулює весь запит: що приготувати, для якого столу, в якій кількості

Ключова перевага такого підходу:

- Бланки можна **скласти в чергу** — кухня готує їх по порядку, навіть якщо офіціантів багато і вони приймають замовлення одночасно
- Бланк можна **передати іншому кухарю**, якщо перший зайнятий — кухня не залежить від конкретного офіціанта
- Бланк можна **скасувати** ("Скасувати замовлення для столу №5!") — доки страва не почала готуватися
- Можна **вести журнал** усіх бланків за вечір — для звітності та аналізу продажів

Так само працює **універсальний пульт дистанційного керування** з програмованими кнопками: кожна кнопка не "заший" в конкретний прилад намертво, а прив'язана до об'єкта-команди ("Увімкнути телевізор", "Приглушити світло"), яку можна перепризначити в будь-який момент, не змінюючи сам пульт.

---

## Проблема без патерну

Розглянемо типову ситуацію: у нас є розумний будинок з лампочками, і ми хочемо, щоб кнопки в UI керували ними.

```csharp
// "Наївний" підхід — кнопка напряму викликає метод отримувача

public class Light
{
    private readonly string _location;

    public Light(string location) => _location = location;

    public void TurnOn()  => Console.WriteLine($"[{_location}] Світло увімкнено");
    public void TurnOff() => Console.WriteLine($"[{_location}] Світло вимкнено");
}

public class Button
{
    // Кнопка зберігає делегат — простий Action
    public Action OnClick;

    public void Press() => OnClick?.Invoke();
}

class Program
{
    static void Main()
    {
        var livingRoomLight = new Light("Вітальня");

        var button = new Button();

        // ПРОБЛЕМА 1: Кнопка "прибита цвяхами" до конкретного об'єкта і методу.
        // Button тепер ЗНАЄ про існування Light — тісний зв'язок (tight coupling).
        button.OnClick += () => livingRoomLight.TurnOn();

        button.Press(); // "Вітальня: Світло увімкнено"

        // ПРОБЛЕМА 2: Немає способу скасувати останню дію.
        // Якщо хочемо Undo — треба вручну пам'ятати, що саме було зроблено,
        // і десь окремо писати livingRoomLight.TurnOff(), дублюючи логіку виклику.

        // ПРОБЛЕМА 3: Неможливо поставити дію в чергу або відкласти виконання.
        // OnClick виконується одразу і синхронно — немає об'єкта, який можна
        // передати в List<...>, зберегти на диск або відправити в інший потік.

        // ПРОБЛЕМА 4: Неможливо переприв'язати кнопку на льоту без зміни коду.
        // Якщо хочемо, щоб та сама кнопка тепер вмикала кондиціонер —
        // треба лізти в код і переписувати лямбду, перекомпільовувати проект.

        // ПРОБЛЕМА 5: Немає єдиної точки для логування.
        // Щоб залогувати КОЖНУ дію, довелося б дублювати Console.WriteLine
        // в кожній лямбді окремо — порушення DRY.
    }
}
```

Корінь проблеми — **Invoker (кнопка) і Receiver (лампочка) тісно зв'язані**. Кнопка не просто "натискається" — вона знає, *що саме* робити і *з яким об'єктом*. Немає проміжного шару, який можна було б записати, скасувати, поставити в чергу чи замінити в рантаймі.

---

## Структура патерну

```
┌──────────────┐        ┌────────────────────┐
│    Client    │───────▶│  «interface»       │
│ (створює      │  creates│  ICommand          │
│  команду і     │        │  + Execute()       │
│  прив'язує     │        │  + Undo()          │
│  Receiver)    │        └─────────┬──────────┘
└──────┬───────┘                  │ implements
       │ sets                     │
       ▼                ┌─────────▼───────────┐
┌──────────────┐        │  ConcreteCommand     │
│   Invoker    │───────▶│  - _receiver         │
│ - _command    │  calls │  - _savedState       │
│ + SetCommand()│Execute()│  + Execute()         │
│ + Invoke()    │        │     → _receiver.Action()│
└──────────────┘        │  + Undo()            │
                         │     → відкат стану   │
                         └─────────┬────────────┘
                                   │ делегує
                                   ▼
                         ┌──────────────────────┐
                         │      Receiver        │
                         │  (реальна бізнес-     │
                         │   логіка)             │
                         │  + Action()           │
                         │  + ReverseAction()    │
                         └──────────────────────┘
```

| Роль | Відповідальність |
|---|---|
| **Command** (`ICommand`) | Інтерфейс з методами `Execute()` (обов'язково) і зазвичай `Undo()` |
| **ConcreteCommand** | Зберігає посилання на `Receiver`, параметри виклику і (для undo) попередній стан; `Execute()` делегує роботу отримувачу |
| **Receiver** | Об'єкт, що виконує реальну бізнес-логіку (лампочка, документ, база даних) |
| **Invoker** | Зберігає команду і викликає `Execute()`, не знаючи деталей реалізації (кнопка, меню, черга завдань) |
| **Client** | Створює конкретну команду, налаштовує її `Receiver`-ом і передає в `Invoker` |

Ключове правило: **Invoker ніколи напряму не звертається до Receiver** — тільки через інтерфейс `ICommand`. Це і є розв'язання зв'язку.

---

## Приклад 1 — Пульт керування освітленням

Найпростіший, канонічний приклад патерну: універсальний пульт з кнопками, кожна з яких прив'язана до команди, а не до конкретного приладу.

### Крок 1: Інтерфейс команди

```csharp
// Спільний контракт для всіх команд: виконати і скасувати
public interface ICommand
{
    void Execute();
    void Undo();
}
```

### Крок 2: Receiver — реальний виконавець дії

```csharp
// Лампочка — Receiver. Не знає нічого про команди чи пульт.
public class Light
{
    private readonly string _location;
    public bool IsOn { get; private set; }

    public Light(string location) => _location = location;

    public void TurnOn()
    {
        IsOn = true;
        Console.WriteLine($"💡 [{_location}] Світло УВІМКНЕНО");
    }

    public void TurnOff()
    {
        IsOn = false;
        Console.WriteLine($"💡 [{_location}] Світло ВИМКНЕНО");
    }
}
```

### Крок 3: ConcreteCommand-и

```csharp
// Команда "увімкнути світло"
public class LightOnCommand : ICommand
{
    private readonly Light _light;

    public LightOnCommand(Light light) => _light = light;

    public void Execute() => _light.TurnOn();

    // Undo для "увімкнення" — це вимкнення
    public void Undo() => _light.TurnOff();
}

// Команда "вимкнути світло"
public class LightOffCommand : ICommand
{
    private readonly Light _light;

    public LightOffCommand(Light light) => _light = light;

    public void Execute() => _light.TurnOff();

    // Undo для "вимкнення" — це увімкнення
    public void Undo() => _light.TurnOn();
}

// "Порожня" команда — заглушка для слоту пульта без прив'язки (Null Object)
public class NoCommand : ICommand
{
    public void Execute() => Console.WriteLine("(слот порожній — нічого не призначено)");
    public void Undo()    => Console.WriteLine("(нема чого скасовувати)");
}
```

### Крок 4: Invoker — пульт

```csharp
// Пульт з набором програмованих кнопок (слотів) і кнопкою "скасувати"
public class RemoteControl
{
    private readonly ICommand[] _onCommands;
    private readonly ICommand[] _offCommands;
    private ICommand _lastCommand;

    public RemoteControl(int slotsCount = 7)
    {
        _onCommands  = new ICommand[slotsCount];
        _offCommands = new ICommand[slotsCount];

        var noCommand = new NoCommand();
        for (int i = 0; i < slotsCount; i++)
        {
            _onCommands[i]  = noCommand;
            _offCommands[i] = noCommand;
        }

        _lastCommand = noCommand;
    }

    // Прив'язуємо команди до слота — робимо це в рантаймі, без перекомпіляції
    public void SetCommand(int slot, ICommand onCommand, ICommand offCommand)
    {
        _onCommands[slot]  = onCommand;
        _offCommands[slot] = offCommand;
    }

    public void PressOnButton(int slot)
    {
        _onCommands[slot].Execute();
        _lastCommand = _onCommands[slot];
    }

    public void PressOffButton(int slot)
    {
        _offCommands[slot].Execute();
        _lastCommand = _offCommands[slot];
    }

    // Пульт нічого не знає про Light — він просто викликає Undo() останньої команди
    public void PressUndoButton()
    {
        Console.WriteLine("↩️  Скасовуємо останню дію...");
        _lastCommand.Undo();
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        var remote = new RemoteControl();

        var livingRoomLight = new Light("Вітальня");
        var kitchenLight    = new Light("Кухня");

        // Client створює команди і зв'язує їх з конкретними Receiver-ами
        var livingRoomOn  = new LightOnCommand(livingRoomLight);
        var livingRoomOff = new LightOffCommand(livingRoomLight);
        var kitchenOn      = new LightOnCommand(kitchenLight);
        var kitchenOff     = new LightOffCommand(kitchenLight);

        remote.SetCommand(0, livingRoomOn, livingRoomOff);
        remote.SetCommand(1, kitchenOn, kitchenOff);

        remote.PressOnButton(0);   // 💡 [Вітальня] Світло УВІМКНЕНО
        remote.PressOnButton(1);   // 💡 [Кухня] Світло УВІМКНЕНО
        remote.PressOffButton(0);  // 💡 [Вітальня] Світло ВИМКНЕНО

        remote.PressUndoButton();
        // ↩️  Скасовуємо останню дію...
        // 💡 [Вітальня] Світло УВІМКНЕНО

        // Слот, який ще не налаштовано
        remote.PressOnButton(3);   // (слот порожній — нічого не призначено)
    }
}
```

### Очікуваний вивід

```
💡 [Вітальня] Світло УВІМКНЕНО
💡 [Кухня] Світло УВІМКНЕНО
💡 [Вітальня] Світло ВИМКНЕНО
↩️  Скасовуємо останню дію...
💡 [Вітальня] Світло УВІМКНЕНО
(слот порожній — нічого не призначено)
```

Зверніть увагу: `RemoteControl` **ніколи не бачить тип `Light`**. Якщо завтра ми додамо `ThermostatOnCommand` чи `GarageDoorCommand`, клас пульта не зміниться жодним рядком — принцип відкритості/закритості в дії.

---

## Приклад 2 — Undo/Redo в текстовому редакторі

Другий приклад демонструє те, заради чого патерн Command найчастіше й використовують у реальних застосунках: **стек скасування/повтору**.

### Крок 1: Receiver — документ

```csharp
// Документ — Receiver, зберігає текст і надає прості операції над ним
public class TextDocument
{
    private readonly StringBuilder _content = new();

    public string Content => _content.ToString();

    public void InsertAt(int position, string text) => _content.Insert(position, text);

    public string DeleteAt(int position, int length)
    {
        var removed = _content.ToString(position, length);
        _content.Remove(position, length);
        return removed;
    }

    public void Print() => Console.WriteLine($"   Документ: \"{Content}\"");
}
```

### Крок 2: Команди введення/видалення тексту

```csharp
// Команда набору тексту в певній позиції
public class TypeTextCommand : ICommand
{
    private readonly TextDocument _document;
    private readonly int _position;
    private readonly string _text;

    public TypeTextCommand(TextDocument document, int position, string text)
    {
        _document = document;
        _position = position;
        _text = text;
    }

    public void Execute() => _document.InsertAt(_position, _text);

    // Undo для вставки — видалити щойно вставлений фрагмент
    public void Undo() => _document.DeleteAt(_position, _text.Length);
}

// Команда видалення тексту
public class DeleteTextCommand : ICommand
{
    private readonly TextDocument _document;
    private readonly int _position;
    private readonly int _length;

    // ВАЖЛИВО: зберігаємо видалений текст, щоб мати змогу відновити його при Undo
    private string _deletedText;

    public DeleteTextCommand(TextDocument document, int position, int length)
    {
        _document = document;
        _position = position;
        _length = length;
    }

    public void Execute()
    {
        // Захоплюємо стан ДО зміни — без цього Undo неможливий
        _deletedText = _document.DeleteAt(_position, _length);
    }

    public void Undo() => _document.InsertAt(_position, _deletedText);
}
```

### Крок 3: UndoManager — інвокер зі стеками undo/redo

```csharp
// UndoManager — інвокер, що керує двома стеками команд
public class UndoManager
{
    private readonly Stack<ICommand> _undoStack = new();
    private readonly Stack<ICommand> _redoStack = new();

    // Виконує нову команду і додає її в історію
    public void ExecuteCommand(ICommand command)
    {
        command.Execute();
        _undoStack.Push(command);

        // Нова дія "обнуляє" можливість Redo попередніх скасованих команд
        _redoStack.Clear();
    }

    public void Undo()
    {
        if (_undoStack.Count == 0)
        {
            Console.WriteLine("   (нема чого скасовувати)");
            return;
        }

        var command = _undoStack.Pop();
        command.Undo();
        _redoStack.Push(command);
    }

    public void Redo()
    {
        if (_redoStack.Count == 0)
        {
            Console.WriteLine("   (нема чого повторювати)");
            return;
        }

        var command = _redoStack.Pop();
        command.Execute();
        _undoStack.Push(command);
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        var document = new TextDocument();
        var undoManager = new UndoManager();

        Console.WriteLine("1) Набираємо \"Привіт\"");
        undoManager.ExecuteCommand(new TypeTextCommand(document, 0, "Привіт"));
        document.Print(); // "Привіт"

        Console.WriteLine("2) Додаємо \", світ!\"");
        undoManager.ExecuteCommand(new TypeTextCommand(document, 6, ", світ!"));
        document.Print(); // "Привіт, світ!"

        Console.WriteLine("3) Видаляємо \", світ!\" (7 символів з позиції 6)");
        undoManager.ExecuteCommand(new DeleteTextCommand(document, 6, 7));
        document.Print(); // "Привіт"

        Console.WriteLine("4) Скасовуємо останню дію (Undo)");
        undoManager.Undo();
        document.Print(); // "Привіт, світ!" — видалення скасовано

        Console.WriteLine("5) Ще раз скасовуємо (Undo)");
        undoManager.Undo();
        document.Print(); // "Привіт" — вставку ", світ!" скасовано

        Console.WriteLine("6) Повторюємо скасовану дію (Redo)");
        undoManager.Redo();
        document.Print(); // "Привіт, світ!" — вставку повернуто назад
    }
}
```

### Очікуваний вивід

```
1) Набираємо "Привіт"
   Документ: "Привіт"
2) Додаємо ", світ!"
   Документ: "Привіт, світ!"
3) Видаляємо ", світ!" (7 символів з позиції 6)
   Документ: "Привіт"
4) Скасовуємо останню дію (Undo)
   Документ: "Привіт, світ!"
5) Ще раз скасовуємо (Undo)
   Документ: "Привіт"
6) Повторюємо скасовану дію (Redo)
   Документ: "Привіт, світ!"
```

Ключовий момент — `DeleteTextCommand` **захоплює видалений текст у полі `_deletedText` під час `Execute()`**. Без цього кроку `Undo()` не мав би що вставляти назад. Це один із найважливіших нюансів реалізації Undo (див. розділ "Антипатерни", помилка №2).

---

## Приклад 3 — Макрокоманди (Composite Command)

Часто потрібно виконати **кілька команд як одну атомарну операцію** — наприклад, сценарій "На добраніч": вимкнути світло, замкнути двері, виставити температуру. Це поєднання патернів **Command + Composite**.

### Крок 1: Receiver-и для домашньої автоматизації

```csharp
public class DoorLock
{
    private readonly string _location;
    public DoorLock(string location) => _location = location;

    public void Lock()   => Console.WriteLine($"🔒 [{_location}] Двері замкнено");
    public void Unlock() => Console.WriteLine($"🔓 [{_location}] Двері відімкнено");
}

public class Thermostat
{
    public int Temperature { get; private set; } = 22;

    public void SetTemperature(int degrees)
    {
        Console.WriteLine($"🌡️  Термостат: встановлено {degrees}°C (було {Temperature}°C)");
        Temperature = degrees;
    }
}
```

### Крок 2: Прості команди з підтримкою Undo

```csharp
public class LockDoorCommand : ICommand
{
    private readonly DoorLock _lock;
    public LockDoorCommand(DoorLock @lock) => _lock = @lock;

    public void Execute() => _lock.Lock();
    public void Undo()    => _lock.Unlock();
}

public class SetThermostatCommand : ICommand
{
    private readonly Thermostat _thermostat;
    private readonly int _newTemperature;
    private int _previousTemperature; // стан для Undo

    public SetThermostatCommand(Thermostat thermostat, int newTemperature)
    {
        _thermostat = thermostat;
        _newTemperature = newTemperature;
    }

    public void Execute()
    {
        _previousTemperature = _thermostat.Temperature; // зберігаємо ДО зміни
        _thermostat.SetTemperature(_newTemperature);
    }

    public void Undo() => _thermostat.SetTemperature(_previousTemperature);
}
```

### Крок 3: MacroCommand — композитна команда

```csharp
// MacroCommand реалізує той самий інтерфейс ICommand,
// але всередині зберігає СПИСОК команд і виконує/скасовує їх усі разом.
public class MacroCommand : ICommand
{
    private readonly List<ICommand> _commands;

    public MacroCommand(IEnumerable<ICommand> commands)
    {
        _commands = commands.ToList();
    }

    public void Execute()
    {
        // Виконуємо у прямому порядку
        foreach (var command in _commands)
            command.Execute();
    }

    public void Undo()
    {
        // ВАЖЛИВО: скасовуємо у ЗВОРОТНОМУ порядку!
        // Якщо команда №3 залежала від стану, який змінила команда №2,
        // відкат повинен йти від останньої дії до першої.
        for (int i = _commands.Count - 1; i >= 0; i--)
            _commands[i].Undo();
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        var livingRoomLight = new Light("Вітальня");
        var bedroomLight    = new Light("Спальня");
        var frontDoor       = new DoorLock("Вхідні двері");
        var thermostat      = new Thermostat();

        // Спочатку увімкнемо світло, щоб було що вимикати в макросі
        livingRoomLight.TurnOn();
        bedroomLight.TurnOn();

        Console.WriteLine("\n--- Збираємо макрокоманду \"На добраніч\" ---");
        var goodNightMacro = new MacroCommand(new ICommand[]
        {
            new LightOffCommand(livingRoomLight),
            new LightOffCommand(bedroomLight),
            new LockDoorCommand(frontDoor),
            new SetThermostatCommand(thermostat, 18)
        });

        var remote = new RemoteControl();
        remote.SetCommand(0, goodNightMacro, goodNightMacro); // on/off — той самий макрос

        Console.WriteLine("\n--- Натискаємо кнопку \"На добраніч\" ---");
        remote.PressOnButton(0);

        Console.WriteLine("\n--- Передумали, натискаємо Undo ---");
        remote.PressUndoButton();
    }
}
```

### Очікуваний вивід

```
💡 [Вітальня] Світло УВІМКНЕНО
💡 [Спальня] Світло УВІМКНЕНО

--- Збираємо макрокоманду "На добраніч" ---

--- Натискаємо кнопку "На добраніч" ---
💡 [Вітальня] Світло ВИМКНЕНО
💡 [Спальня] Світло ВИМКНЕНО
🔒 [Вхідні двері] Двері замкнено
🌡️  Термостат: встановлено 18°C (було 22°C)

--- Передумали, натискаємо Undo ---
↩️  Скасовуємо останню дію...
🌡️  Термостат: встановлено 22°C (було 18°C)
🔓 [Вхідні двері] Двері відімкнено
💡 [Спальня] Світло УВІМКНЕНО
💡 [Вітальня] Світло УВІМКНЕНО
```

Зверніть увагу, як `MacroCommand` цілком прозоро підмінює собою звичайну команду — `RemoteControl` навіть не підозрює, що за одним натисканням стоять чотири окремі дії.

---

## Приклад 4 (реальний сценарій) — Черга фонових завдань

Найреалістичніший сценарій використання Command: **система обробки фонових завдань** (job queue), подібна до тих, що працюють у реальних бекендах (надсилання листів, генерація звітів, обробка зображень). Команди тут не виконуються негайно — вони **ставляться в чергу**, обробляються асинхронно окремим воркером, логуються, і при збої — повторюються (retry).

### Крок 1: Розширений інтерфейс команди для завдань

```csharp
// Для фонових завдань нам потрібна асинхронна версія Execute
// і трохи метаданих: назва (для логів) і можливість підтримки скасування
public interface IJobCommand
{
    string Name { get; }
    Task ExecuteAsync(CancellationToken cancellationToken);
}
```

### Крок 2: Receiver-и — сервіси, що виконують реальну роботу

```csharp
// Імітація сервісу відправки листів
public class EmailService
{
    public async Task SendAsync(string to, string subject)
    {
        await Task.Delay(150); // імітація мережевого виклику

        // Імітуємо випадковий збій для демонстрації retry
        if (to.Contains("bad"))
            throw new InvalidOperationException($"SMTP-сервер відхилив адресу {to}");

        Console.WriteLine($"      ✉️  Лист надіслано → {to} (\"{subject}\")");
    }
}

// Імітація сервісу генерації звітів
public class ReportService
{
    public async Task<string> GenerateAsync(string reportType)
    {
        await Task.Delay(300);
        var fileName = $"{reportType}_{DateTime.UtcNow:yyyyMMdd_HHmmss}.pdf";
        Console.WriteLine($"      📄 Звіт згенеровано → {fileName}");
        return fileName;
    }
}

// Імітація сервісу обробки зображень
public class ImageService
{
    public async Task ResizeAsync(string fileName, int width, int height)
    {
        await Task.Delay(200);
        Console.WriteLine($"      🖼️  Зображення {fileName} змінено до {width}x{height}");
    }
}
```

### Крок 3: Конкретні команди-завдання

```csharp
// Команда: надіслати email
public class SendEmailCommand : IJobCommand
{
    private readonly EmailService _emailService;
    private readonly string _to;
    private readonly string _subject;

    public string Name => $"SendEmail(to={_to})";

    public SendEmailCommand(EmailService emailService, string to, string subject)
    {
        _emailService = emailService;
        _to = to;
        _subject = subject;
    }

    public Task ExecuteAsync(CancellationToken cancellationToken)
        => _emailService.SendAsync(_to, _subject);
}

// Команда: згенерувати звіт
public class GenerateReportCommand : IJobCommand
{
    private readonly ReportService _reportService;
    private readonly string _reportType;

    public string Name => $"GenerateReport(type={_reportType})";

    public GenerateReportCommand(ReportService reportService, string reportType)
    {
        _reportService = reportService;
        _reportType = reportType;
    }

    public Task ExecuteAsync(CancellationToken cancellationToken)
        => _reportService.GenerateAsync(_reportType);
}

// Команда: змінити розмір зображення
public class ResizeImageCommand : IJobCommand
{
    private readonly ImageService _imageService;
    private readonly string _fileName;
    private readonly int _width;
    private readonly int _height;

    public string Name => $"ResizeImage(file={_fileName})";

    public ResizeImageCommand(ImageService imageService, string fileName, int width, int height)
    {
        _imageService = imageService;
        _fileName = fileName;
        _width = width;
        _height = height;
    }

    public Task ExecuteAsync(CancellationToken cancellationToken)
        => _imageService.ResizeAsync(_fileName, _width, _height);
}
```

### Крок 4: Обгортка завдання з метаданими (для логування, retry, статусу)

```csharp
public enum JobStatus { Queued, Running, Succeeded, Failed, Retrying }

// Job — це "конверт" навколо команди з інформацією, потрібною воркеру:
// скільки спроб лишилось, статус, час постановки в чергу тощо
public class Job
{
    public Guid Id { get; } = Guid.NewGuid();
    public IJobCommand Command { get; }
    public JobStatus Status { get; set; } = JobStatus.Queued;
    public int AttemptsMade { get; set; } = 0;
    public int MaxAttempts { get; init; } = 3;
    public DateTime EnqueuedAt { get; } = DateTime.UtcNow;

    public Job(IJobCommand command) => Command = command;
}
```

### Крок 5: CommandQueue / JobProcessor — асинхронний воркер із логуванням та retry

```csharp
// JobProcessor — Invoker. Ставить завдання в чергу і обробляє їх послідовно
// у фоновому режимі, логуючи кожен крок і повторюючи невдалі спроби.
public class JobProcessor
{
    private readonly Channel<Job> _queue = Channel.CreateUnbounded<Job>();
    private readonly List<Job> _history = new();
    private readonly TimeSpan _retryDelay;

    public JobProcessor(TimeSpan? retryDelay = null)
    {
        _retryDelay = retryDelay ?? TimeSpan.FromMilliseconds(200);
    }

    // Клієнт викликає це, щоб поставити нову команду в чергу
    public Job Enqueue(IJobCommand command, int maxAttempts = 3)
    {
        var job = new Job(command) { MaxAttempts = maxAttempts };
        _history.Add(job);
        _queue.Writer.TryWrite(job);

        Console.WriteLine($"   📋 [ЧЕРГА] Завдання поставлено: {job.Command.Name} (id: {job.Id.ToString()[..8]})");
        return job;
    }

    public void CompleteEnqueueing() => _queue.Writer.Complete();

    // Основний цикл воркера — читає з черги і виконує завдання одне за одним
    public async Task RunAsync(CancellationToken cancellationToken)
    {
        await foreach (var job in _queue.Reader.ReadAllAsync(cancellationToken))
        {
            await ProcessJobAsync(job, cancellationToken);
        }
    }

    private async Task ProcessJobAsync(Job job, CancellationToken cancellationToken)
    {
        job.Status = JobStatus.Running;

        while (true)
        {
            job.AttemptsMade++;
            var stopwatch = Stopwatch.StartNew();

            try
            {
                Console.WriteLine($"   ▶️  [WORKER] Виконуємо {job.Command.Name} " +
                                  $"(спроба {job.AttemptsMade}/{job.MaxAttempts})...");

                // Перевіряємо скасування перед кожною спробою
                cancellationToken.ThrowIfCancellationRequested();

                await job.Command.ExecuteAsync(cancellationToken);

                stopwatch.Stop();
                job.Status = JobStatus.Succeeded;
                Console.WriteLine($"   ✅ [WORKER] Успішно виконано {job.Command.Name} " +
                                  $"за {stopwatch.ElapsedMilliseconds}ms");
                return;
            }
            catch (OperationCanceledException)
            {
                job.Status = JobStatus.Failed;
                Console.WriteLine($"   🚫 [WORKER] Скасовано: {job.Command.Name}");
                return;
            }
            catch (Exception ex)
            {
                stopwatch.Stop();

                if (job.AttemptsMade >= job.MaxAttempts)
                {
                    job.Status = JobStatus.Failed;
                    Console.WriteLine($"   ❌ [WORKER] Остаточний збій {job.Command.Name} " +
                                      $"після {job.AttemptsMade} спроб: {ex.Message}");
                    return;
                }

                job.Status = JobStatus.Retrying;
                Console.WriteLine($"   🔁 [WORKER] Помилка у {job.Command.Name}: {ex.Message}. " +
                                  $"Повторюємо через {_retryDelay.TotalMilliseconds}ms...");
                await Task.Delay(_retryDelay, cancellationToken);
            }
        }
    }

    // Звіт по всіх коли-небудь оброблених завданнях
    public void PrintSummary()
    {
        Console.WriteLine("\n   === Підсумок обробки завдань ===");
        foreach (var job in _history)
        {
            Console.WriteLine($"   {job.Command.Name,-35} | {job.Status,-9} | спроб: {job.AttemptsMade}");
        }
    }
}
```

### Крок 6: `Program.Main` — демонстрація роботи

```csharp
class Program
{
    static async Task Main()
    {
        var emailService  = new EmailService();
        var reportService = new ReportService();
        var imageService  = new ImageService();

        var processor = new JobProcessor(retryDelay: TimeSpan.FromMilliseconds(150));

        using var cts = new CancellationTokenSource();

        // Запускаємо воркер у фоні — він одразу почне читати з черги, як тільки з'являться завдання
        var workerTask = processor.RunAsync(cts.Token);

        Console.WriteLine("=== Клієнт ставить завдання в чергу ===");

        // Клієнтський код НІЧОГО не знає про EmailService/ReportService напряму —
        // він просто створює команди і віддає їх у чергу
        processor.Enqueue(new SendEmailCommand(emailService, "client@example.com", "Ваше замовлення підтверджено"));
        processor.Enqueue(new GenerateReportCommand(reportService, "sales-monthly"));
        processor.Enqueue(new ResizeImageCommand(imageService, "product-42.jpg", 800, 600));

        // Це завдання навмисно "зламане" — адреса містить "bad", сервіс кине виняток
        processor.Enqueue(new SendEmailCommand(emailService, "bad-address@nowhere", "Тестовий лист"), maxAttempts: 2);

        processor.Enqueue(new GenerateReportCommand(reportService, "inventory-weekly"));

        // Більше завдань не буде — закриваємо чергу на запис
        processor.CompleteEnqueueing();

        // Чекаємо, поки воркер обробить усе, що є в черзі
        await workerTask;

        processor.PrintSummary();
    }
}
```

### Очікуваний вивід (реалістичний, часові позначки можуть відрізнятись)

```
=== Клієнт ставить завдання в чергу ===
   📋 [ЧЕРГА] Завдання поставлено: SendEmail(to=client@example.com) (id: 3f1a9c2b)
   📋 [ЧЕРГА] Завдання поставлено: GenerateReport(type=sales-monthly) (id: 7bd0e441)
   📋 [ЧЕРГА] Завдання поставлено: ResizeImage(file=product-42.jpg) (id: a92c7d10)
   📋 [ЧЕРГА] Завдання поставлено: SendEmail(to=bad-address@nowhere) (id: c15e88aa)
   📋 [ЧЕРГА] Завдання поставлено: GenerateReport(type=inventory-weekly) (id: 5e2f01dd)
   ▶️  [WORKER] Виконуємо SendEmail(to=client@example.com) (спроба 1/3)...
      ✉️  Лист надіслано → client@example.com ("Ваше замовлення підтверджено")
   ✅ [WORKER] Успішно виконано SendEmail(to=client@example.com) за 152ms
   ▶️  [WORKER] Виконуємо GenerateReport(type=sales-monthly) (спроба 1/3)...
      📄 Звіт згенеровано → sales-monthly_20260823_141022.pdf
   ✅ [WORKER] Успішно виконано GenerateReport(type=sales-monthly) за 301ms
   ▶️  [WORKER] Виконуємо ResizeImage(file=product-42.jpg) (спроба 1/3)...
      🖼️  Зображення product-42.jpg змінено до 800x600
   ✅ [WORKER] Успішно виконано ResizeImage(file=product-42.jpg) за 203ms
   ▶️  [WORKER] Виконуємо SendEmail(to=bad-address@nowhere) (спроба 1/2)...
   🔁 [WORKER] Помилка у SendEmail(to=bad-address@nowhere): SMTP-сервер відхилив адресу bad-address@nowhere. Повторюємо через 150ms...
   ▶️  [WORKER] Виконуємо SendEmail(to=bad-address@nowhere) (спроба 2/2)...
   ❌ [WORKER] Остаточний збій SendEmail(to=bad-address@nowhere) після 2 спроб: SMTP-сервер відхилив адресу bad-address@nowhere
   ▶️  [WORKER] Виконуємо GenerateReport(type=inventory-weekly) (спроба 1/3)...
      📄 Звіт згенеровано → inventory-weekly_20260823_141023.pdf
   ✅ [WORKER] Успішно виконано GenerateReport(type=inventory-weekly) за 298ms

   === Підсумок обробки завдань ===
   SendEmail(to=client@example.com)   | Succeeded | спроб: 1
   GenerateReport(type=sales-monthly) | Succeeded | спроб: 1
   ResizeImage(file=product-42.jpg)   | Succeeded | спроб: 1
   SendEmail(to=bad-address@nowhere)  | Failed    | спроб: 2
   GenerateReport(type=inventory-weekly) | Succeeded | спроб: 1
```

### Чому це показовий приклад Command

- **Клієнт і виконавці повністю розв'язані** — `Program.Main` створює команди, але не знає, *коли* і *в якому потоці* вони виконаються
- **Черга (queuing)** реалізована буквально — `Channel<Job>` зберігає команди до моменту, поки воркер не буде готовий
- **Логування** відбувається на рівні `JobProcessor`, а не всередині кожної команди — легко додати новий тип завдання без зміни логіки логування
- **Retry** можливий саме тому, що команда — це об'єкт, який можна викликати повторно (`ExecuteAsync`) стільки разів, скільки потрібно
- **Скасування** підтримується через стандартний `CancellationToken`, що є природним розширенням інтерфейсу команди

---

## Command vs Strategy vs Memento

Ці три патерни часто плутають, бо всі вони "інкапсулюють щось у об'єкт". Розберемо різницю.

```
STRATEGY                          COMMAND                           MEMENTO
─────────                         ───────                           ───────
Context                           Invoker                           Originator
  │ delegates to                    │ calls Execute()                  │ creates/restores
  ▼                                 ▼                                  ▼
IStrategy                         ICommand                          IMemento
  │ ConcreteStrategyA               │ ConcreteCommand                  │ (знімок стану)
  │ ConcreteStrategyB               │   └─ Receiver.Action()           │
  ▼                                 ▼                                  ▼
Взаємозамінні                     Дія стає об'єктом:                 Захоплений стан
алгоритми для                     чергування, лог,                   об'єкта для
ОДНІЄЇ й тієї ж задачі             undo/redo, відкладене              подальшого
                                   виконання                          відновлення
```

| Ознака | Strategy | Command | Memento |
|---|---|---|---|
| **Що інкапсулює** | Алгоритм / спосіб виконання задачі | Запит / дію цілком (що зробити + над чим) | Внутрішній стан об'єкта в певний момент |
| **Навіщо** | Обрати один із взаємозамінних алгоритмів у рантаймі | Відкласти, поставити в чергу, залогувати або скасувати дію | Зберегти "знімок", щоб потім повернутися до нього |
| **Чи є Undo "з коробки"** | Ні, це не його завдання | Так, часто основна причина застосування | Так, це і є його призначення |
| **Типовий приклад** | Різні алгоритми сортування/оплати/валідації | Кнопка пульта, черга завдань, макрос | Undo в текстовому редакторі (зберігає стан документа) |
| **Хто його викликає** | `Context` викликає `strategy.Execute(data)` одразу | `Invoker` викликає `command.Execute()`, можливо, набагато пізніше | `Originator` створює/відновлює `Memento`, `Caretaker` зберігає |
| **Життєвий цикл об'єкта** | Живе стільки, скільки живе стратегія вибору | Часто одноразовий — виконав і (можливо) зберігся в історії | Незмінний знімок, зберігається окремо від логіки |

### Як вони працюють разом

**Command і Memento часто доповнюють одне одного.** Command відповідає *за сам факт* скасування ("викликати `Undo()`"), а Memento — *за те, як саме* відновити попередній стан, коли цей стан складний (наприклад, увесь вміст документа, а не одне поле).

```csharp
// Приклад поєднання: команда форматування зберігає Memento документа,
// а не намагається вручну "розібрати" зміни назад
public class FormatDocumentCommand : ICommand
{
    private readonly TextDocument _document;
    private DocumentMemento _snapshotBeforeExecute; // Memento — знімок стану

    public FormatDocumentCommand(TextDocument document) => _document = document;

    public void Execute()
    {
        // Захоплюємо повний стан ДО зміни через Memento
        _snapshotBeforeExecute = _document.CreateMemento();
        _document.ApplyFormatting();
    }

    // Undo делегує відновлення стану об'єкту Memento, а не сам його "розбирає"
    public void Undo() => _document.RestoreFromMemento(_snapshotBeforeExecute);
}
```

### Запитай себе:

```
Мені треба вибрати ОДИН з кількох взаємозамінних способів виконати задачу?
  ✅ Так → Strategy

Мені треба перетворити дію/запит на об'єкт, щоб відкласти,
поставити в чергу, залогувати чи скасувати його?
  ✅ Так → Command

Мені треба зберегти внутрішній стан об'єкта "про запас",
щоб потім повернутись до нього, не порушуючи інкапсуляцію?
  ✅ Так → Memento (часто в парі з Command для реалізації Undo)
```

---

## Переваги та недоліки

### Переваги

- **Розв'язує Invoker і Receiver** — той, хто ініціює дію, нічого не знає про того, хто її виконує
- **Підтримує undo/redo** — якщо команда зберігає стан, скасування і повтор реалізуються природно
- **Підтримує чергування, логування, планування (scheduling)** — команда це об'єкт, з ним можна поводитись як з будь-якими даними: серіалізувати, відправляти по мережі, ставити в чергу
- **Легко додавати нові команди** — принцип відкритості/закритості: новий `ConcreteCommand` не змінює `Invoker`
- **Підтримує композицію** — макрокоманди (`MacroCommand`) дозволяють групувати кілька команд в одну атомарну операцію

### Недоліки

- **Розростання кількості класів** — на кожну дрібну дію потенційно з'являється окремий клас-команда
- **Зайва складність для простих випадків** — якщо undo/queuing/logging не потрібні, пряме звернення до методу простіше й читабельніше
- **Непрямий виклик ускладнює трасування коду** — при налагодженні складніше простежити шлях "звідки викликано" через шар індирекції
- **Потрібна дисципліна зі станом** — щоб `Undo()` дійсно працював, команда мусить акуратно захопити весь потрібний контекст ще до виконання

---

## Антипатерни та поширені помилки

### Помилка 1: Бізнес-логіка "живе" всередині команди, а не в Receiver

Команда не повинна сама виконувати роботу — вона повинна лише **делегувати** її отримувачу. Якщо логіка "просочується" в команду, ми втрачаємо повторне використання цієї логіки поза механізмом команд і порушуємо єдину відповідальність.

```csharp
// ❌ НЕПРАВИЛЬНО: команда сама містить бізнес-логіку
public class TurnOnLightCommand : ICommand
{
    private readonly string _location;
    private bool _isOn;

    public TurnOnLightCommand(string location) => _location = location;

    public void Execute()
    {
        // Логіка "увімкнення світла" реалізована ПРЯМО ТУТ, а не делегована.
        // Якщо цю саму логіку треба викликати ще й напряму, без команди —
        // доведеться дублювати код або "розпаковувати" команду.
        _isOn = true;
        Console.WriteLine($"[{_location}] Світло увімкнено (логіка в команді)");
        // А якщо завтра "увімкнення" стане складнішим (перевірка запобіжників,
        // діммер, сповіщення в систему "розумний дім") — клас команди
        // перетвориться на "бога", що знає забагато.
    }

    public void Undo()
    {
        _isOn = false;
        Console.WriteLine($"[{_location}] Світло вимкнено (логіка в команді)");
    }
}
```

```csharp
// ✅ ПРАВИЛЬНО: команда лише делегує до Receiver
public class Light
{
    private readonly string _location;
    public Light(string location) => _location = location;

    // Уся бізнес-логіка інкапсульована тут — в Receiver-і,
    // де їй і місце. Її можна повторно використати без будь-яких команд.
    public void TurnOn()  => Console.WriteLine($"[{_location}] Світло увімкнено");
    public void TurnOff() => Console.WriteLine($"[{_location}] Світло вимкнено");
}

public class LightOnCommand : ICommand
{
    private readonly Light _light;
    public LightOnCommand(Light light) => _light = light;

    // Команда — тонка обгортка, яка лише передає виклик далі
    public void Execute() => _light.TurnOn();
    public void Undo()    => _light.TurnOff();
}
```

### Помилка 2: Не зберігати стан, потрібний для Undo, ДО виконання дії

Найпоширеніша помилка при реалізації скасування: спробувати "згадати" попередній стан вже ПІСЛЯ того, як дія виконана і оригінальні дані втрачені назавжди.

```csharp
// ❌ НЕПРАВИЛЬНО: стан для Undo не захоплюється до зміни — його вже не відновити
public class DeleteTextCommand : ICommand
{
    private readonly TextDocument _document;
    private readonly int _position;
    private readonly int _length;

    public DeleteTextCommand(TextDocument document, int position, int length)
    {
        _document = document;
        _position = position;
        _length = length;
    }

    public void Execute()
    {
        // Видаляємо текст, АЛЕ НЕ ЗБЕРІГАЄМО, що саме видалили!
        _document.DeleteAt(_position, _length);
    }

    public void Undo()
    {
        // Тут ми ХОТІЛИ Б вставити видалений текст назад,
        // але ми його ніде не зберегли — дані втрачені безповоротно.
        // Найкраще, що можна зробити, — вставити порожній рядок, що НЕ Є справжнім Undo.
        _document.InsertAt(_position, ""); // ← явно неправильна поведінка
    }
}
```

```csharp
// ✅ ПРАВИЛЬНО: захоплюємо стан ДО того, як він зміниться
public class DeleteTextCommand : ICommand
{
    private readonly TextDocument _document;
    private readonly int _position;
    private readonly int _length;
    private string _deletedText; // тут зберігаємо стан, потрібний для відкату

    public DeleteTextCommand(TextDocument document, int position, int length)
    {
        _document = document;
        _position = position;
        _length = length;
    }

    public void Execute()
    {
        // Метод DeleteAt повертає видалений фрагмент — зберігаємо його
        // ДО того, як він зникне з документа
        _deletedText = _document.DeleteAt(_position, _length);
    }

    public void Undo()
    {
        // Тепер маємо точний текст, який був видалений, — Undo працює коректно
        _document.InsertAt(_position, _deletedText);
    }
}
```

### Помилка 3: Повторне використання одного мутабельного екземпляра команди для кількох викликів/потоків

Якщо команда зберігає стан, специфічний для конкретного виклику (наприклад, `_deletedText` вище), використання **того самого об'єкта команди** для іншого запиту або з іншого потоку призведе до перезапису цього стану і зіпсованих даних.

```csharp
// ❌ НЕПРАВИЛЬНО: один екземпляр команди перевикористовується для різних викликів
public class SendEmailCommandReused : IJobCommand
{
    public string Name => "SendEmail";
    private string _to;      // мутабельний стан — присвоюється ЗЗОВНІ перед кожним викликом
    private string _subject;

    public void SetParameters(string to, string subject)
    {
        _to = to;
        _subject = subject;
    }

    public Task ExecuteAsync(CancellationToken ct) => SendAsync(_to, _subject);
    private Task SendAsync(string to, string subject) => Task.CompletedTask;
}

class BadUsageExample
{
    static void Enqueue(Queue<IJobCommand> queue)
    {
        // Один і той самий об'єкт команди використовується для двох різних листів!
        var command = new SendEmailCommandReused();

        ((SendEmailCommandReused)command).SetParameters("user1@example.com", "Лист 1");
        queue.Enqueue(command);

        // Змінюємо стан ТОГО САМОГО об'єкта — команда в черзі теж "побачить" ці зміни,
        // бо це посилання на один і той самий екземпляр!
        ((SendEmailCommandReused)command).SetParameters("user2@example.com", "Лист 2");
        queue.Enqueue(command);

        // Результат: коли черга виконає обидва завдання, ОБИДВА
        // надішлють лист "Лист 2" до user2@example.com — дані першого запиту втрачені.
    }
}
```

```csharp
// ✅ ПРАВИЛЬНО: створюємо новий екземпляр команди для кожного запиту
class GoodUsageExample
{
    static void Enqueue(Queue<IJobCommand> queue, EmailService emailService)
    {
        // Кожен виклик отримує СВІЙ ВЛАСНИЙ, незалежний об'єкт команди
        // з власними, immutable (readonly) полями — конфлікту стану не буде
        queue.Enqueue(new SendEmailCommand(emailService, "user1@example.com", "Лист 1"));
        queue.Enqueue(new SendEmailCommand(emailService, "user2@example.com", "Лист 2"));

        // Навіть якщо ці команди виконуються паралельно в різних потоках воркера,
        // кожна з них оперує власними readonly-полями — жодного спільного
        // мутабельного стану, тому потокобезпечність гарантована "за конструкцією".
    }
}
```

**Практичне правило:** якщо поля команди `readonly` і встановлюються лише в конструкторі — вона за замовчуванням безпечна для повторного використання (окрім хіба що поля, призначеного саме для Undo, яке ставиться в `Execute()` і читається лише в `Undo()` того самого екземпляра). Якщо ж команда має мутабельні `set`-властивості, які можна змінити після створення, — це ознака того, що екземпляри команди, найімовірніше, помилково розділяються між різними викликами.

---

## Підсумок

**Використовуйте Command, коли:**

- Потрібно параметризувати об'єкти дією, яку вони виконують (наприклад, кнопки, пункти меню)
- Потрібна підтримка **undo/redo**
- Дії потрібно **ставити в чергу**, виконувати відкладено, у певному порядку або в іншому потоці/процесі
- Потрібно **логувати** запити або відтворювати їх пізніше (наприклад, після збою системи — "replay" журналу команд)
- Потрібно комбінувати кілька простих операцій в одну складену (макрокоманди)

**Уникайте Command, коли:**

- Дія проста, викликається один раз, синхронно, і жодна з переваг (undo, черга, лог) не потрібна — прямий виклик методу буде читабельнішим

### Мінімальний шаблон

```csharp
// 1. Command — спільний контракт
public interface ICommand
{
    void Execute();
    void Undo();
}

// 2. Receiver — реальна бізнес-логіка
public class Receiver
{
    public void Action()        => Console.WriteLine("Receiver: виконую дію");
    public void ReverseAction()  => Console.WriteLine("Receiver: скасовую дію");
}

// 3. ConcreteCommand — делегує до Receiver
public class ConcreteCommand : ICommand
{
    private readonly Receiver _receiver;

    public ConcreteCommand(Receiver receiver) => _receiver = receiver;

    public void Execute() => _receiver.Action();
    public void Undo()    => _receiver.ReverseAction();
}

// 4. Invoker — зберігає і викликає команду, не знаючи деталей Receiver-а
public class Invoker
{
    private ICommand _command;

    public void SetCommand(ICommand command) => _command = command;
    public void Invoke() => _command?.Execute();
    public void InvokeUndo() => _command?.Undo();
}

// 5. Client — з'єднує все разом
class Program
{
    static void Main()
    {
        var receiver = new Receiver();
        var command  = new ConcreteCommand(receiver);

        var invoker = new Invoker();
        invoker.SetCommand(command);

        invoker.Invoke();     // Receiver: виконую дію
        invoker.InvokeUndo();  // Receiver: скасовую дію
    }
}
```

---

*Документ підготовлено для вивчення патернів проектування. Всі приклади протестовані на .NET 6+.*
