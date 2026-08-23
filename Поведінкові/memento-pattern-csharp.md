# Патерн Memento (Знімок) — Детальний розбір на C#

> **Категорія:** Поведінковий (Behavioral)  
> **Автори:** Gang of Four (GoF)  
> **Мова прикладів:** C#

---

## Зміст

1. [Що таке Memento?](#що-таке-memento)
2. [Проблема без патерну](#проблема-без-патерну)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Текстовий редактор з Undo](#приклад-1--текстовий-редактор-з-undo)
5. [Приклад 2 — Undo та Redo](#приклад-2--undo-та-redo)
6. [Приклад 3 — Збереження гри (Save/Load)](#приклад-3--збереження-гри-saveload)
7. [Приклад 4 (реальний сценарій) — Графічний редактор з історією змін](#приклад-4-реальний-сценарій--графічний-редактор-з-історією-змін)
8. [Memento vs Command vs Prototype](#memento-vs-command-vs-prototype)
9. [Переваги та недоліки](#переваги-та-недоліки)
10. [Антипатерни та поширені помилки](#антипатерни-та-поширені-помилки)
11. [Підсумок](#підсумок)

---

## Що таке Memento?

**Memento (Знімок)** — це поведінковий патерн проектування, який дозволяє зберігати та відновлювати попередній внутрішній стан об'єкта, **не порушуючи його інкапсуляцію**.

Головна ідея: об'єкт (його називають **Originator**, "джерело") сам вирішує, яку частину свого внутрішнього стану зробити знімком (**Memento**), і сам вміє відновлюватися з цього знімка. Зовнішній код, який зберігає історію знімків (**Caretaker**, "доглядач"), ніколи не заглядає всередину Memento — він лише зберігає його, передає назад і не знає, що там усередині.

Це і є ключова відмінність Memento від "наївного" підходу — зберегти стан можна без того, щоб оголошувати всі приватні поля публічними.

Патерн лежить в основі:

- функції **Undo/Redo** в текстових та графічних редакторах;
- **збережень (save game)** у відеоіграх;
- **транзакцій та rollback** у базах даних та бізнес-логіці;
- **checkpoint'ів** у довгих обчисленнях чи workflow.

### Аналогія з реального світу

Уявімо три ситуації, які насправді є одним і тим самим патерном:

1. **Збереження гри.** Коли ви натискаєте "Зберегти гру", гра створює файл збереження — знімок усього поточного стану (рівень, здоров'я, інвентар, позиція). Ви граєте далі, можливо, помираєте чи робите щось невдало — і завантажуєте той самий save-файл, повертаючись рівно в той момент. Сам файл збереження — це "чорна скринька": ви не редагуєте його вручну, за це відповідає гра.

2. **Ctrl+Z у текстовому редакторі.** Кожна суттєва зміна документа може супроводжуватись прихованим знімком попереднього стану. Натискання Ctrl+Z не "вгадує", як скасувати зміну — воно просто підставляє назад раніше збережений знімок.

3. **Точка збереження (checkpoint) у транзакції бази даних.** `SAVEPOINT` у SQL фіксує стан транзакції, а `ROLLBACK TO SAVEPOINT` повертає базу даних до цього стану, якщо щось пішло не так — без того, щоб зовнішній код знав про внутрішню структуру сторінок і логів бази даних.

У всіх трьох випадках є три ролі: **об'єкт, чий стан зберігається** (гра, документ, транзакція), **непрозорий знімок** (файл збереження, прихована копія тексту, savepoint) та **той, хто керує колекцією знімків** (меню "Завантажити гру", стек Undo, менеджер транзакцій).

---

## Проблема без патерну

Найпростіший (і найгірший) спосіб реалізувати Undo — просто зробити всі приватні поля об'єкта публічними, щоб зовнішній код міг їх зчитувати й записувати напряму.

```csharp
// ПОГАНО: TextDocument змушений розкривати всю свою внутрішню структуру,
// щоб зовнішній код міг "запам'ятовувати" і "відновлювати" стан.
public class TextDocument
{
    // Було б набагато краще зробити ці поля приватними,
    // але тоді History не зможе прочитати чи записати їх напряму.
    public string Content { get; set; }
    public int CursorPosition { get; set; }
    public string FontName { get; set; }
    public int FontSize { get; set; }

    public void Type(string text)
    {
        Content = Content.Insert(CursorPosition, text);
        CursorPosition += text.Length;
    }
}

// "Історія" зберігає стан документа, звертаючись до публічних полів напряму.
public class BadHistory
{
    // Кожен запис - це "ручна" копія всіх полів документа.
    private class Snapshot
    {
        public string Content;
        public int CursorPosition;
        public string FontName;
        public int FontSize;
    }

    private readonly Stack<Snapshot> _snapshots = new();

    public void Save(TextDocument document)
    {
        _snapshots.Push(new Snapshot
        {
            Content = document.Content,
            CursorPosition = document.CursorPosition,
            FontName = document.FontName,
            FontSize = document.FontSize
        });
    }

    public void Undo(TextDocument document)
    {
        if (_snapshots.Count == 0) return;

        var snapshot = _snapshots.Pop();
        // Зовнішній клас напряму записує стан всередину документа -
        // це і є порушення інкапсуляції.
        document.Content = snapshot.Content;
        document.CursorPosition = snapshot.CursorPosition;
        document.FontName = snapshot.FontName;
        document.FontSize = snapshot.FontSize;
    }
}
```

**У чому проблема:**

1. **Грубе порушення інкапсуляції.** `TextDocument` змушений робити ВСІ свої внутрішні поля публічними, хоча зовнішньому коду (крім `BadHistory`) вони не мають бути видимі взагалі. Будь-хто в програмі тепер може написати `document.CursorPosition = -999;` і зламати інваріанти об'єкта.

2. **Крихка взаємозалежність (tight coupling).** `BadHistory` повинен точно знати, з яких полів складається `TextDocument`. Щойно в `TextDocument` з'явиться нове поле (наприклад, `IsBold`), потрібно буде синхронно правити і `BadHistory`, інакше Undo буде відновлювати стан частково і неправильно.

3. **Немає єдиної точки контролю за створенням знімка.** Логіка "що саме входить у знімок" розмазана по зовнішньому класу, замість того щоб бути інкапсульованою всередині самого `TextDocument`, який єдиний насправді знає, що є критичним для його стану.

4. **Масштабується погано.** Якщо в проєкті є 10 різних класів, для яких потрібен Undo, доведеться писати 10 однотипних "History"-класів, кожен з яких напряму лізе у внутрішні публічні поля свого об'єкта — і кожна така зміна є потенційним джерелом багів.

Патерн Memento вирішує це, дозволяючи `TextDocument` самому створювати непрозорий об'єкт-знімок і самому ж його розшифровувати, тоді як "історія" лише зберігає посилання на цей знімок, не знаючи його вмісту.

---

## Структура патерну

```
┌────────────────────────────┐
│         Originator          │
│  (об'єкт, чий стан          │
│   зберігається)             │
├──────────────────────────────┤
│ - state: SomeState           │
├──────────────────────────────┤
│ + CreateMemento(): Memento    │──────┐
│ + RestoreMemento(m: Memento)  │      │ створює
└──────────────────────────────┘      ▼
                              ┌────────────────────┐
                              │      Memento         │
                              │  (непрозорий знімок) │
                              ├────────────────────┤
                              │ - state: SomeState    │ <-- видно лише Originator-у
                              ├────────────────────┤
                              │ (немає публічного      │
                              │  доступу до state)      │
                              └────────────────────┘
                                        ▲
                                        │ зберігає / повертає
                                        │ (не заглядаючи всередину)
                              ┌────────────────────┐
                              │      Caretaker        │
                              │  (історія / стек змін) │
                              ├────────────────────┤
                              │ - history: List<Memento>│
                              ├────────────────────┤
                              │ + Backup()             │
                              │ + Undo()                │
                              └────────────────────┘
```

### Таблиця ролей

| Роль | Відповідальність |
|---|---|
| **Originator** | Об'єкт, чий внутрішній стан потрібно зберігати та відновлювати. Єдиний, хто вміє створювати `Memento` (`CreateMemento()`) і відновлюватися з нього (`RestoreMemento()`). Знає структуру свого стану. |
| **Memento** | Непрозорий об'єкт-знімок стану Originator-а на певний момент часу. Зазвичай реалізується як **приватний вкладений клас** Originator-а, тому доступ до вмісту мають лише методи самого Originator-а. |
| **Caretaker** | Зберігає історію знімків (наприклад, у стеку чи списку) і керує тим, *коли* робити знімок і *коли* відновлюватися. Ніколи не читає й не змінює вміст Memento — для нього це "чорна скринька". |
| **Client** | Код, який ініціює операції над Originator-ом і викликає Caretaker для Undo/Redo/Save/Load. |

Ключовий принцип: **Caretaker зберігає Memento, але не має доступу до його внутрішнього стану.** Це досягається в C# зазвичай через вкладений приватний клас або через `internal`-модифікатори доступу, обмежені збіркою.

---

## Приклад 1 — Текстовий редактор з Undo

Найпростіший приклад: `TextDocument` — Originator, вкладений приватний клас `Memento` — знімок, `History` — Caretaker.

```csharp
using System;
using System.Collections.Generic;

// ===================== ORIGINATOR =====================
public class TextDocument
{
    private string _content = string.Empty;

    public string Content => _content;

    public void Type(string text)
    {
        _content += text;
        Console.WriteLine($"[Документ] Додано текст: \"{text}\" -> \"{_content}\"");
    }

    public void DeleteLastWord()
    {
        int lastSpace = _content.TrimEnd().LastIndexOf(' ');
        _content = lastSpace >= 0 ? _content[..(lastSpace + 1)] : string.Empty;
        Console.WriteLine($"[Документ] Видалено останнє слово -> \"{_content}\"");
    }

    // Створює знімок поточного стану.
    // Повертає Memento - непрозорий об'єкт, чий вміст видно лише зсередини TextDocument.
    public Memento CreateMemento()
    {
        return new Memento(_content);
    }

    // Відновлює стан з переданого знімка.
    public void RestoreMemento(Memento memento)
    {
        _content = memento.GetContent();
        Console.WriteLine($"[Документ] Відновлено стан -> \"{_content}\"");
    }

    // Memento - приватний вкладений клас.
    // Його конструктор і метод GetContent() доступні лише коду всередині TextDocument,
    // тому Caretaker (History) не може прочитати чи змінити вміст напряму.
    public class Memento
    {
        private readonly string _savedContent;

        // internal - доступ обмежено збіркою; можна зробити private
        // і винести History як вкладений клас, якщо потрібна повна ізоляція.
        internal Memento(string content)
        {
            _savedContent = content;
        }

        internal string GetContent() => _savedContent;
    }
}

// ===================== CARETAKER =====================
public class History
{
    private readonly Stack<TextDocument.Memento> _snapshots = new();

    // Зробити знімок поточного стану документа й покласти його в історію.
    public void Save(TextDocument document)
    {
        _snapshots.Push(document.CreateMemento());
        Console.WriteLine("[Історія] Знімок збережено.");
    }

    // Відкотити документ до попереднього знімка.
    public void Undo(TextDocument document)
    {
        if (_snapshots.Count == 0)
        {
            Console.WriteLine("[Історія] Немає збережених станів для скасування.");
            return;
        }

        var memento = _snapshots.Pop();
        document.RestoreMemento(memento);
    }
}
```

### Використання

```csharp
var document = new TextDocument();
var history = new History();

history.Save(document);           // знімок порожнього документа
document.Type("Привіт, ");

history.Save(document);
document.Type("світ!");

Console.WriteLine($"Поточний текст: \"{document.Content}\"");

history.Undo(document);           // повернення до "Привіт, "
history.Undo(document);           // повернення до ""
history.Undo(document);           // історія порожня
```

**Очікуваний вивід:**

```
[Історія] Знімок збережено.
[Документ] Додано текст: "Привіт, " -> "Привіт, "
[Історія] Знімок збережено.
[Документ] Додано текст: "світ!" -> "Привіт, світ!"
Поточний текст: "Привіт, світ!"
[Документ] Відновлено стан -> "Привіт, "
[Документ] Відновлено стан -> ""
[Історія] Немає збережених станів для скасування.
```

Зверніть увагу: `History` ніде не звертається до `_savedContent` напряму — вона лише передає об'єкт `Memento` туди й назад.

---

## Приклад 2 — Undo та Redo

Розширимо перший приклад: додамо повноцінний `HistoryManager` з **двома стеками** — стеком Undo та стеком Redo. Це класична реалізація Undo/Redo, яку використовують текстові й графічні редактори.

Логіка:

- При кожній зміні поточний стан (перед зміною) кладеться в **стек Undo**.
- При виклику `Undo()` поточний стан переміщується в **стек Redo**, а зі стека Undo дістається попередній стан.
- При виклику `Redo()` відбувається зворотна операція.
- Будь-яка **нова** зміна після Undo очищає стек Redo (стандартна поведінка редакторів — "гілка" історії, яку скасували, більше не доступна для Redo).

```csharp
using System;
using System.Collections.Generic;

// ===================== ORIGINATOR =====================
public class TextDocument
{
    private string _content = string.Empty;

    public string Content => _content;

    public void Type(string text) => _content += text;

    public void DeleteLastWord()
    {
        int lastSpace = _content.TrimEnd().LastIndexOf(' ');
        _content = lastSpace >= 0 ? _content[..(lastSpace + 1)] : string.Empty;
    }

    public Memento CreateMemento() => new Memento(_content);

    public void RestoreMemento(Memento memento) => _content = memento.GetContent();

    public class Memento
    {
        private readonly string _savedContent;
        internal Memento(string content) => _savedContent = content;
        internal string GetContent() => _savedContent;
    }
}

// ===================== CARETAKER з Undo/Redo =====================
public class HistoryManager
{
    private readonly Stack<TextDocument.Memento> _undoStack = new();
    private readonly Stack<TextDocument.Memento> _redoStack = new();

    // Викликається ПЕРЕД тим, як внести зміну в документ.
    public void RecordState(TextDocument document)
    {
        _undoStack.Push(document.CreateMemento());

        // Нова зміна "обриває" гілку redo - її більше не можна повернути.
        if (_redoStack.Count > 0)
        {
            _redoStack.Clear();
            Console.WriteLine("[Історія] Стек Redo очищено через нову зміну.");
        }
    }

    public void Undo(TextDocument document)
    {
        if (_undoStack.Count == 0)
        {
            Console.WriteLine("[Історія] ↩️ Немає що скасовувати.");
            return;
        }

        // Поточний стан документа зберігаємо в Redo перед відкатом.
        _redoStack.Push(document.CreateMemento());

        var previous = _undoStack.Pop();
        document.RestoreMemento(previous);
        Console.WriteLine($"[Історія] ↩️ Undo -> \"{document.Content}\"");
    }

    public void Redo(TextDocument document)
    {
        if (_redoStack.Count == 0)
        {
            Console.WriteLine("[Історія] ⤴️ Немає що повторювати.");
            return;
        }

        _undoStack.Push(document.CreateMemento());

        var next = _redoStack.Pop();
        document.RestoreMemento(next);
        Console.WriteLine($"[Історія] ⤴️ Redo -> \"{document.Content}\"");
    }
}
```

### Використання

```csharp
var document = new TextDocument();
var history = new HistoryManager();

history.RecordState(document);
document.Type("Дизайн");
Console.WriteLine($"Стан 1: \"{document.Content}\"");

history.RecordState(document);
document.Type(" патернів");
Console.WriteLine($"Стан 2: \"{document.Content}\"");

history.RecordState(document);
document.Type(" - це круто!");
Console.WriteLine($"Стан 3: \"{document.Content}\"");

Console.WriteLine("--- Скасовуємо двічі ---");
history.Undo(document);
history.Undo(document);

Console.WriteLine("--- Повторюємо один раз ---");
history.Redo(document);

Console.WriteLine("--- Вносимо нову зміну (гілка Redo обривається) ---");
history.RecordState(document);
document.Type(" Дуже!");
Console.WriteLine($"Стан: \"{document.Content}\"");

Console.WriteLine("--- Спроба Redo після нової зміни ---");
history.Redo(document);
```

**Очікуваний вивід:**

```
Стан 1: "Дизайн"
Стан 2: "Дизайн патернів"
Стан 3: "Дизайн патернів - це круто!"
--- Скасовуємо двічі ---
[Історія] ↩️ Undo -> "Дизайн патернів"
[Історія] ↩️ Undo -> "Дизайн"
--- Повторюємо один раз ---
[Історія] ⤴️ Redo -> "Дизайн патернів"
--- Вносимо нову зміну (гілка Redo обривається) ---
[Історія] Стек Redo очищено через нову зміну.
Стан: "Дизайн патернів Дуже!"
--- Спроба Redo після нової зміни ---
[Історія] ⤴️ Немає що повторювати.
```

---

## Приклад 3 — Збереження гри (Save/Load)

Тепер застосуємо Memento до сценарію "справжнього" збереження гри: `GameCharacter` (Originator) має здоров'я, рівень, інвентар та позицію. `SaveSlotManager` (Caretaker) керує кількома **іменованими слотами збереження**, як у більшості RPG-ігор.

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

// ===================== ORIGINATOR =====================
public class GameCharacter
{
    public string Name { get; }
    public int Health { get; private set; }
    public int Level { get; private set; }
    public List<string> Inventory { get; private set; } = new();
    public (int X, int Y) Position { get; private set; }

    public GameCharacter(string name)
    {
        Name = name;
        Health = 100;
        Level = 1;
        Position = (0, 0);
    }

    public void TakeDamage(int amount)
    {
        Health = Math.Max(0, Health - amount);
        Console.WriteLine($"[{Name}] Отримано {amount} шкоди. Здоров'я: {Health}");
    }

    public void LevelUp()
    {
        Level++;
        Health = 100;
        Console.WriteLine($"[{Name}] Новий рівень: {Level}! Здоров'я відновлено.");
    }

    public void PickUpItem(string item)
    {
        Inventory.Add(item);
        Console.WriteLine($"[{Name}] Підібрано предмет: {item}");
    }

    public void MoveTo(int x, int y)
    {
        Position = (x, y);
        Console.WriteLine($"[{Name}] Переміщено до ({x}, {y})");
    }

    // Створює знімок поточного стану персонажа.
    public GameMemento SaveGame()
    {
        Console.WriteLine($"[{Name}] 💾 Стан збережено (рівень {Level}, здоров'я {Health}).");
        // Важливо: список Inventory копіюється (new List<string>(Inventory)),
        // інакше Memento зберігав би посилання на той самий список,
        // і майбутні зміни інвентарю "протікали" б у вже збережений знімок.
        return new GameMemento(Health, Level, new List<string>(Inventory), Position);
    }

    // Відновлює стан персонажа зі знімка.
    public void LoadGame(GameMemento memento)
    {
        Health = memento.Health;
        Level = memento.Level;
        Inventory = new List<string>(memento.Inventory); // знову копія, а не посилання
        Position = memento.Position;
        Console.WriteLine($"[{Name}] 📂 Стан завантажено (рівень {Level}, здоров'я {Health}).");
    }

    // ===================== MEMENTO =====================
    // Вкладений клас з приватними полями. Публічні лише для читання
    // всередині GameCharacter (internal), назовні недоступні для запису.
    public class GameMemento
    {
        internal int Health { get; }
        internal int Level { get; }
        internal List<string> Inventory { get; }
        internal (int X, int Y) Position { get; }
        internal DateTime SavedAt { get; }

        internal GameMemento(int health, int level, List<string> inventory, (int, int) position)
        {
            Health = health;
            Level = level;
            Inventory = inventory;
            Position = position;
            SavedAt = DateTime.Now;
        }
    }
}

// ===================== CARETAKER =====================
// Керує іменованими слотами збереження - як меню "Save Game" в RPG.
public class SaveSlotManager
{
    private readonly Dictionary<string, GameCharacter.GameMemento> _slots = new();

    public void SaveToSlot(string slotName, GameCharacter character)
    {
        _slots[slotName] = character.SaveGame();
        Console.WriteLine($"[Слоти] Записано в слот \"{slotName}\".");
    }

    public void LoadFromSlot(string slotName, GameCharacter character)
    {
        if (!_slots.TryGetValue(slotName, out var memento))
        {
            Console.WriteLine($"[Слоти] ❌ Слот \"{slotName}\" не знайдено.");
            return;
        }

        character.LoadGame(memento);
        Console.WriteLine($"[Слоти] Завантажено зі слота \"{slotName}\".");
    }

    public void ListSlots()
    {
        Console.WriteLine("[Слоти] Доступні збереження:");
        foreach (var slot in _slots)
        {
            Console.WriteLine($"  - \"{slot.Key}\"");
        }
    }
}
```

### Використання

```csharp
var hero = new GameCharacter("Данило");
var saveSlots = new SaveSlotManager();

hero.PickUpItem("Меч новачка");
hero.MoveTo(5, 3);
saveSlots.SaveToSlot("slot1_start", hero);

hero.TakeDamage(40);
hero.LevelUp();
hero.PickUpItem("Щит дракона");
hero.MoveTo(20, 15);
saveSlots.SaveToSlot("slot2_before_boss", hero);

hero.TakeDamage(90);   // персонаж майже загинув у бою з босом
Console.WriteLine($"Поточне здоров'я: {hero.Health}");

Console.WriteLine("--- Завантажуємо збереження перед боєм з босом ---");
saveSlots.LoadFromSlot("slot2_before_boss", hero);
Console.WriteLine($"Здоров'я після завантаження: {hero.Health}, рівень: {hero.Level}, " +
                   $"інвентар: [{string.Join(", ", hero.Inventory)}], позиція: {hero.Position}");

saveSlots.ListSlots();
```

**Очікуваний вивід:**

```
[Данило] Підібрано предмет: Меч новачка
[Данило] Переміщено до (5, 3)
[Данило] 💾 Стан збережено (рівень 1, здоров'я 100).
[Слоти] Записано в слот "slot1_start".
[Данило] Отримано 40 шкоди. Здоров'я: 60
[Данило] Новий рівень: 2! Здоров'я відновлено.
[Данило] Підібрано предмет: Щит дракона
[Данило] Переміщено до (20, 15)
[Данило] 💾 Стан збережено (рівень 2, здоров'я 100).
[Слоти] Записано в слот "slot2_before_boss".
[Данило] Отримано 90 шкоди. Здоров'я: 10
Поточне здоров'я: 10
--- Завантажуємо збереження перед боєм з босом ---
[Данило] 📂 Стан завантажено (рівень 2, здоров'я 100).
[Слоти] Завантажено зі слота "slot2_before_boss".
Здоров'я після завантаження: 100, рівень: 2, інвентар: [Меч новачка, Щит дракона], позиція: (20, 15)
[Слоти] Доступні збереження:
  - "slot1_start"
  - "slot2_before_boss"
```

---

## Приклад 4 (реальний сценарій) — Графічний редактор з історією змін

Розглянемо реалістичний сценарій, схожий на спрощений Figma: `Canvas` (Originator) містить список фігур (`Shape`) з позицією, розміром і кольором. Перед кожною деструктивною операцією (переміщення, видалення, зміна кольору, зміна розміру) робиться знімок поточного стану полотна. `HistoryManager` (Caretaker) підтримує багаторівневий Undo/Redo **з обмеженням розміру історії** — коли історія переповнюється, найстаріші знімки відкидаються, щоб не витрачати пам'ять безмежно.

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

// ===================== ДОПОМІЖНІ ТИПИ =====================
public class Shape
{
    public Guid Id { get; }
    public string Type { get; set; }       // "Rectangle", "Circle", "Text" тощо
    public int X { get; set; }
    public int Y { get; set; }
    public int Width { get; set; }
    public int Height { get; set; }
    public string Color { get; set; }

    public Shape(string type, int x, int y, int width, int height, string color)
    {
        Id = Guid.NewGuid();
        Type = type;
        X = x;
        Y = y;
        Width = width;
        Height = height;
        Color = color;
    }

    // Глибока копія фігури - потрібна, щоб знімки Canvas
    // не ділили один і той самий об'єкт Shape між собою.
    public Shape DeepClone()
    {
        return new Shape(Type, X, Y, Width, Height, Color);
    }

    public override string ToString() =>
        $"{Type}#{Id.ToString()[..4]} [{Color}] at ({X},{Y}) size {Width}x{Height}";
}

// ===================== ORIGINATOR =====================
public class Canvas
{
    private List<Shape> _shapes = new();

    public IReadOnlyList<Shape> Shapes => _shapes;

    public Shape AddShape(string type, int x, int y, int width, int height, string color)
    {
        var shape = new Shape(type, x, y, width, height, color);
        _shapes.Add(shape);
        Console.WriteLine($"[Canvas] Додано фігуру: {shape}");
        return shape;
    }

    public void MoveShape(Guid id, int newX, int newY)
    {
        var shape = _shapes.FirstOrDefault(s => s.Id == id);
        if (shape == null) return;
        shape.X = newX;
        shape.Y = newY;
        Console.WriteLine($"[Canvas] Переміщено {shape.Type}#{id.ToString()[..4]} -> ({newX}, {newY})");
    }

    public void RecolorShape(Guid id, string newColor)
    {
        var shape = _shapes.FirstOrDefault(s => s.Id == id);
        if (shape == null) return;
        shape.Color = newColor;
        Console.WriteLine($"[Canvas] Змінено колір {shape.Type}#{id.ToString()[..4]} -> {newColor}");
    }

    public void ResizeShape(Guid id, int width, int height)
    {
        var shape = _shapes.FirstOrDefault(s => s.Id == id);
        if (shape == null) return;
        shape.Width = width;
        shape.Height = height;
        Console.WriteLine($"[Canvas] Змінено розмір {shape.Type}#{id.ToString()[..4]} -> {width}x{height}");
    }

    public void DeleteShape(Guid id)
    {
        var shape = _shapes.FirstOrDefault(s => s.Id == id);
        if (shape == null) return;
        _shapes.Remove(shape);
        Console.WriteLine($"[Canvas] Видалено {shape.Type}#{id.ToString()[..4]}");
    }

    public void PrintState(string label)
    {
        Console.WriteLine($"--- Стан полотна ({label}): {_shapes.Count} фігур(и) ---");
        foreach (var s in _shapes)
            Console.WriteLine($"    {s}");
    }

    // Створює повний знімок поточного набору фігур.
    // КРИТИЧНО ВАЖЛИВО: кожна фігура клонується глибоко (DeepClone),
    // інакше знімок міститиме посилання на ті самі об'єкти Shape,
    // які продовжують змінюватися після створення знімка.
    public CanvasMemento CreateMemento()
    {
        var clonedShapes = _shapes.Select(s => s.DeepClone()).ToList();
        return new CanvasMemento(clonedShapes);
    }

    public void RestoreMemento(CanvasMemento memento)
    {
        _shapes = memento.Shapes.Select(s => s.DeepClone()).ToList();
    }

    // ===================== MEMENTO =====================
    public class CanvasMemento
    {
        internal List<Shape> Shapes { get; }
        internal DateTime CreatedAt { get; }

        internal CanvasMemento(List<Shape> shapes)
        {
            Shapes = shapes;
            CreatedAt = DateTime.Now;
        }
    }
}

// ===================== CARETAKER з обмеженням розміру історії =====================
public class HistoryManager
{
    private readonly LinkedList<Canvas.CanvasMemento> _undoHistory = new();
    private readonly Stack<Canvas.CanvasMemento> _redoStack = new();
    private readonly int _maxHistorySize;

    public HistoryManager(int maxHistorySize = 5)
    {
        _maxHistorySize = maxHistorySize;
    }

    // Викликається ПЕРЕД будь-якою деструктивною операцією.
    public void SnapshotBeforeChange(Canvas canvas)
    {
        _undoHistory.AddLast(canvas.CreateMemento());
        _redoStack.Clear(); // нова зміна обриває гілку Redo

        // Якщо історія переповнена - викидаємо найстаріший знімок.
        if (_undoHistory.Count > _maxHistorySize)
        {
            _undoHistory.RemoveFirst();
            Console.WriteLine($"[Історія] Ліміт ({_maxHistorySize}) досягнуто, найстаріший знімок відкинуто.");
        }
    }

    public void Undo(Canvas canvas)
    {
        if (_undoHistory.Count == 0)
        {
            Console.WriteLine("[Історія] ↩️ Немає що скасовувати.");
            return;
        }

        // Поточний стан переносимо в Redo перед відкатом.
        _redoStack.Push(canvas.CreateMemento());

        var last = _undoHistory.Last!.Value;
        _undoHistory.RemoveLast();
        canvas.RestoreMemento(last);
        Console.WriteLine("[Історія] ↩️ Виконано Undo.");
    }

    public void Redo(Canvas canvas)
    {
        if (_redoStack.Count == 0)
        {
            Console.WriteLine("[Історія] ⤴️ Немає що повторювати.");
            return;
        }

        _undoHistory.AddLast(canvas.CreateMemento());

        var next = _redoStack.Pop();
        canvas.RestoreMemento(next);
        Console.WriteLine("[Історія] ⤴️ Виконано Redo.");
    }

    public int UndoAvailable => _undoHistory.Count;
    public int RedoAvailable => _redoStack.Count;
}

// ===================== ВИКОРИСТАННЯ =====================
public class Program
{
    public static void Main()
    {
        var canvas = new Canvas();
        var history = new HistoryManager(maxHistorySize: 3);

        var rect = canvas.AddShape("Rectangle", 10, 10, 100, 50, "Червоний");
        var circle = canvas.AddShape("Circle", 200, 100, 60, 60, "Синій");
        canvas.PrintState("початковий");

        Console.WriteLine();
        Console.WriteLine("=== Переміщення прямокутника ===");
        history.SnapshotBeforeChange(canvas);
        canvas.MoveShape(rect.Id, 50, 50);
        canvas.PrintState("після переміщення");

        Console.WriteLine();
        Console.WriteLine("=== Зміна кольору кола ===");
        history.SnapshotBeforeChange(canvas);
        canvas.RecolorShape(circle.Id, "Зелений");
        canvas.PrintState("після зміни кольору");

        Console.WriteLine();
        Console.WriteLine("=== Зміна розміру прямокутника ===");
        history.SnapshotBeforeChange(canvas);
        canvas.ResizeShape(rect.Id, 150, 80);
        canvas.PrintState("після зміни розміру");

        Console.WriteLine();
        Console.WriteLine("=== Видалення кола (4-та деструктивна операція - історія переповниться) ===");
        history.SnapshotBeforeChange(canvas);
        canvas.DeleteShape(circle.Id);
        canvas.PrintState("після видалення");

        Console.WriteLine();
        Console.WriteLine($"Доступно Undo: {history.UndoAvailable}, Redo: {history.RedoAvailable}");

        Console.WriteLine();
        Console.WriteLine("=== Скасовуємо видалення кола ===");
        history.Undo(canvas);
        canvas.PrintState("після Undo #1");

        Console.WriteLine();
        Console.WriteLine("=== Скасовуємо зміну розміру ===");
        history.Undo(canvas);
        canvas.PrintState("після Undo #2");

        Console.WriteLine();
        Console.WriteLine("=== Повторюємо зміну розміру (Redo) ===");
        history.Redo(canvas);
        canvas.PrintState("після Redo #1");
    }
}
```

### Використання (пояснення сценарію)

Програма виконує чотири деструктивні операції поспіль (переміщення, зміна кольору, зміна розміру, видалення) при ліміті історії в **3 знімки**. Це означає, що на четвертій операції найстаріший знімок (стан "до переміщення прямокутника") буде відкинутий, і повний Undo "в самий початок" стане неможливим — лише в межах останніх трьох кроків.

**Очікуваний вивід:**

```
[Canvas] Додано фігуру: Rectangle#a1b2 [Червоний] at (10,10) size 100x50
[Canvas] Додано фігуру: Circle#c3d4 [Синій] at (200,100) size 60x60
--- Стан полотна (початковий): 2 фігур(и) ---
    Rectangle#a1b2 [Червоний] at (10,10) size 100x50
    Circle#c3d4 [Синій] at (200,100) size 60x60

=== Переміщення прямокутника ===
[Canvas] Переміщено Rectangle#a1b2 -> (50, 50)
--- Стан полотна (після переміщення): 2 фігур(и) ---
    Rectangle#a1b2 [Червоний] at (50,50) size 100x50
    Circle#c3d4 [Синій] at (200,100) size 60x60

=== Зміна кольору кола ===
[Canvas] Змінено колір Circle#c3d4 -> Зелений
--- Стан полотна (після зміни кольору): 2 фігур(и) ---
    Rectangle#a1b2 [Червоний] at (50,50) size 100x50
    Circle#c3d4 [Зелений] at (200,100) size 60x60

=== Зміна розміру прямокутника ===
[Canvas] Змінено розмір Rectangle#a1b2 -> 150x80
--- Стан полотна (після зміни розміру): 2 фігур(и) ---
    Rectangle#a1b2 [Червоний] at (50,50) size 150x80
    Circle#c3d4 [Зелений] at (200,100) size 60x60

=== Видалення кола (4-та деструктивна операція - історія переповниться) ===
[Історія] Ліміт (3) досягнуто, найстаріший знімок відкинуто.
[Canvas] Видалено Circle#c3d4
--- Стан полотна (після видалення): 1 фігур(и) ---
    Rectangle#a1b2 [Червоний] at (50,50) size 150x80

Доступно Undo: 3, Redo: 0

=== Скасовуємо видалення кола ===
[Історія] ↩️ Виконано Undo.
--- Стан полотна (після Undo #1): 2 фігур(и) ---
    Rectangle#a1b2 [Червоний] at (50,50) size 150x80
    Circle#c3d4 [Зелений] at (200,100) size 60x60

=== Скасовуємо зміну розміру ===
[Історія] ↩️ Виконано Undo.
--- Стан полотна (після Undo #2): 2 фігур(и) ---
    Rectangle#a1b2 [Червоний] at (50,50) size 100x50
    Circle#c3d4 [Зелений] at (200,100) size 60x60

=== Повторюємо зміну розміру (Redo) ===
[Історія] ⤴️ Виконано Redo.
--- Стан полотна (після Redo #1): 2 фігур(и) ---
    Rectangle#a1b2 [Червоний] at (50,50) size 150x80
    Circle#c3d4 [Зелений] at (200,100) size 60x60
```

Зверніть увагу: кольори кольору `Rectangle` після Undo #1 і Undo #2 залишаються **червоними**, оскільки колір прямокутника ніколи не змінювався в цьому сценарії — змінювались лише його позиція та розмір. Це підтверджує, що знімки коректно захоплюють *весь* стан полотна, а не лише змінену властивість.

---

## Memento vs Command vs Prototype

На перший погляд ці три патерни можуть здатися схожими, оскільки всі вони так чи інакше пов'язані зі "збереженням стану" чи "копіюванням". Розберемось у відмінностях.

### Memento + Command (типова комбінація)

Дуже часто Command і Memento використовуються **разом**: конкретна команда (наприклад, `MoveShapeCommand`) перед виконанням операції створює Memento стану Originator-а, щоб потім мати можливість реалізувати власний метод `Undo()`.

```
┌───────────────────┐        створює       ┌───────────┐
│ MoveShapeCommand    │ ───────────────────▶ │  Memento    │
│  + Execute()          │                     └───────────┘
│  + Undo() ◀── використовує Memento, щоб відкотити зміну
└───────────────────┘
```

Тобто **Command відповідає "що зробити"**, а **Memento відповідає "як повернути стан назад"**. Команда інкапсулює дію та, за потреби, свій знімок стану для відкату саме цієї дії.

### Memento vs Prototype

Обидва патерни передбачають копіювання стану об'єкта, але мета зовсім різна:

```
Prototype:                              Memento:
┌──────────┐   Clone()   ┌──────────┐   ┌──────────┐  CreateMemento()  ┌──────────┐
│ Original    │ ──────────▶ │  Copy       │   │ Originator │ ────────────────▶ │  Memento   │
│ (продовжує   │            │ (новий,       │   │ (той самий  │                    │ (для того ж │
│  жити далі)  │            │ незалежний   │   │  об'єкт)    │ ◀──────────────── │  об'єкта,   │
└──────────┘            │  об'єкт)     │   └──────────┘   RestoreMemento() │  щоб      │
                                └──────────┘                                       │  повернутись│
                                                                                    │  назад)     │
                                                                                    └──────────┘
```

- **Prototype** створює **нову, незалежну копію** об'єкта, яку далі використовують як самостійну сутність (наприклад, клонуємо шаблон ворога, щоб заспавнити ще одного ворога в грі — і обидва відтепер живуть незалежно одне від одного).
- **Memento** створює знімок стану **того самого** об'єкта, який пізніше буде використано для **повернення цього ж об'єкта** до попереднього стану. Знімок сам по собі — не повноцінна "робоча" копія об'єкта, а часто лише пасивний контейнер даних, не призначений для самостійного використання як `TextDocument` чи `Canvas`.

### Таблиця порівняння

| Критерій | Memento | Command | Prototype |
|---|---|---|---|
| **Мета** | Зберегти/відновити стан об'єкта | Інкапсулювати дію/запит як об'єкт | Створити нову незалежну копію об'єкта |
| **Результат операції** | Непрозорий знімок стану | Об'єкт-команда з `Execute()`/`Undo()` | Повноцінний новий екземпляр об'єкта |
| **Хто "власник" результату** | Originator (лише він читає вміст) | Invoker / історія команд | Клієнтський код (нова незалежна сутність) |
| **Типове застосування** | Undo/Redo стану, save/load, rollback | Черги задач, макроси, Undo дій, логування операцій | Клонування складних об'єктів замість `new` + ручного налаштування |
| **Чи використовуються разом** | Так — усередині `Command.Undo()` | Так — зберігає Memento для відкату | Рідко комбінується з Memento напряму |

### Запитай себе:

- *"Я хочу повернутися до попереднього стану цього самого об'єкта?"* → Memento.
- *"Я хочу представити дію як об'єкт, який можна виконати, поставити в чергу чи скасувати?"* → Command (можливо, у парі з Memento для реалізації Undo).
- *"Мені потрібна нова, самостійна копія об'єкта для подальшого незалежного використання?"* → Prototype.

---

## Переваги та недоліки

### Переваги

- **Зберігає інкапсуляцію.** Внутрішній стан об'єкта залишається прихованим — лише сам Originator знає, як створити й прочитати свій Memento.
- **Спрощує Originator.** Логіка збереження історії (скільки зберігати, коли очищати, скільки версій тримати) винесена в окремий клас Caretaker, а не змішана з бізнес-логікою Originator-а.
- **Дає чистий механізм Undo/Redo/rollback.** Немає потреби вручну "відкочувати" кожну операцію протилежною дією — досить відновити попередній знімок.
- **Знімки незалежні один від одного** (за умови правильного глибокого копіювання), тому можна безпечно зберігати довгу історію станів без побічних ефектів між ними.
- **Легко розширюється.** Додавання нового поля в Originator не вимагає зміни Caretaker — досить оновити логіку `CreateMemento()`/`RestoreMemento()` всередині самого Originator-а.

### Недоліки

- **Витрати пам'яті та CPU.** Якщо стан Originator-а великий (наприклад, весь граф об'єктів складної сцени), а знімки робляться часто, це може швидко з'їсти багато пам'яті — особливо якщо кожен знімок вимагає глибокого копіювання.
- **Caretaker повинен ретельно керувати життєвим циклом знімків.** Без обмеження розміру історії (як у Прикладі 4) чи стратегії видалення старих знімків легко отримати необмежене зростання пам'яті протягом довгої сесії роботи користувача.
- **Складність глибокого копіювання.** Якщо стан містить посилання на інші мутабельні об'єкти (списки, вкладені об'єкти), потрібно уважно реалізовувати глибоке, а не поверхневе копіювання — інакше знімок буде "протікати" і змінюватись разом з оригіналом.
- **Не завжди очевидно, що саме входить у знімок.** Якщо Originator має і "суттєвий" стан (потрібно зберігати), і "похідний"/кешований стан (не варто зберігати), розробнику потрібно свідомо розділяти ці категорії.

---

## Антипатерни та поширені помилки

### Помилка 1 — Публічний доступ до внутрішнього стану Memento

Якщо зробити поля чи властивості Memento публічними й доступними на запис, зовнішній код (Caretaker чи навіть звичайний клієнтський код) отримує можливість "залізти всередину" знімка і зіпсувати його — а це повністю знецінює сенс патерну.

```csharp
// НЕПРАВИЛЬНО: усі поля Memento публічні й доступні для запису.
public class Memento
{
    public string Content { get; set; }   // будь-хто може змінити!
}

public class History
{
    public void CorruptHistory(Memento memento)
    {
        // Caretaker (чи будь-який інший код) може напряму зіпсувати збережений стан.
        memento.Content = "ЗІПСОВАНО";
    }
}
```

```csharp
// ПРАВИЛЬНО: Memento - приватний вкладений клас Originator-а.
// Доступ до вмісту мають лише методи самого TextDocument.
public class TextDocument
{
    private string _content;

    public Memento CreateMemento() => new Memento(_content);

    public void RestoreMemento(Memento memento) => _content = memento.GetContent();

    // Вкладений клас: конструктор і гетер - internal чи private,
    // тому ззовні (з Caretaker) вміст недоступний ні для читання, ні для запису.
    public class Memento
    {
        private readonly string _savedContent;
        internal Memento(string content) => _savedContent = content;
        internal string GetContent() => _savedContent;
    }
}
```

### Помилка 2 — Зберігання посилання замість глибокої копії

Якщо `CreateMemento()` просто зберігає посилання на існуючий мутабельний об'єкт (наприклад, `List<T>`), а не його копію, то будь-яка подальша зміна оригіналу "непомітно" міняє і вже збережений знімок — Undo перестає працювати коректно.

```csharp
// НЕПРАВИЛЬНО: Memento зберігає те саме посилання на список,
// що й Originator. Зміни оригінального списку "просочуються" в знімок.
public class GameCharacter
{
    private List<string> _inventory = new();

    public GameMemento SaveGame()
    {
        // BUG: не копія, а те саме посилання!
        return new GameMemento(_inventory);
    }

    public void PickUpItem(string item) => _inventory.Add(item);

    public class GameMemento
    {
        internal List<string> Inventory { get; }
        internal GameMemento(List<string> inventory) => Inventory = inventory;
    }
}

// Наслідок:
// var memento = character.SaveGame();
// character.PickUpItem("Новий меч");
// memento.Inventory - тепер ТЕЖ містить "Новий меч", хоча це "старий" знімок!
```

```csharp
// ПРАВИЛЬНО: при створенні знімка робимо копію колекції (і, за потреби,
// глибоку копію кожного елемента, якщо елементи теж мутабельні).
public class GameCharacter
{
    private List<string> _inventory = new();

    public GameMemento SaveGame()
    {
        // Копіюємо список у новий об'єкт.
        return new GameMemento(new List<string>(_inventory));
    }

    public void LoadGame(GameMemento memento)
    {
        // І при відновленні теж копіюємо, а не привласнюємо посилання напряму,
        // інакше подальші зміни інвентарю персонажа будуть змінювати сам Memento.
        _inventory = new List<string>(memento.Inventory);
    }

    public void PickUpItem(string item) => _inventory.Add(item);

    public class GameMemento
    {
        internal List<string> Inventory { get; }
        internal GameMemento(List<string> inventory) => Inventory = inventory;
    }
}
```

### Помилка 3 — Необмежене зростання історії

Якщо Caretaker зберігає кожен знімок назавжди, без будь-якого обмеження, довга сесія роботи (наприклад, багатогодинне редагування великого документа чи складної сцени) призведе до постійного зростання споживання пам'яті.

```csharp
// НЕПРАВИЛЬНО: історія росте необмежено протягом усієї сесії роботи.
public class History
{
    private readonly List<TextDocument.Memento> _snapshots = new();

    public void Save(TextDocument document)
    {
        // Жодного обмеження розміру - через кілька годин роботи
        // тут можуть накопичитися тисячі знімків.
        _snapshots.Add(document.CreateMemento());
    }
}
```

```csharp
// ПРАВИЛЬНО: обмежуємо розмір історії та відкидаємо найстаріші знімки.
public class History
{
    private readonly LinkedList<TextDocument.Memento> _snapshots = new();
    private readonly int _maxSize;

    public History(int maxSize = 50)
    {
        _maxSize = maxSize;
    }

    public void Save(TextDocument document)
    {
        _snapshots.AddLast(document.CreateMemento());

        if (_snapshots.Count > _maxSize)
        {
            _snapshots.RemoveFirst(); // відкидаємо найстаріший знімок
        }
    }
}
```

---

## Підсумок

**Використовуйте Memento, коли:**

- потрібна функція **Undo/Redo** для дій користувача (текстові й графічні редактори, IDE);
- потрібно реалізувати **збереження/завантаження** стану (ігри, форми з чернетками, довгі майстри налаштувань);
- потрібен **rollback/checkpoint** у транзакціях чи довгих процесах обробки даних;
- важливо **зберегти інкапсуляцію** об'єкта, чий стан зберігається, і не розкривати його внутрішню структуру зовнішньому коду;
- логіку "коли зберігати" і "коли відновлювати" варто відокремити від бізнес-логіки самого об'єкта (розділення відповідальностей між Originator і Caretaker).

**Будьте обережні, якщо:**

- стан об'єкта дуже великий, а знімки робляться дуже часто — розгляньте інкрементальні знімки (зберігати лише "дельту" змін) замість повних копій;
- об'єкт містить складні мутабельні вкладені структури — переконайтесь, що копіювання дійсно глибоке;
- історія повинна жити довго — обов'язково обмежте її розмір чи додайте стратегію витіснення старих знімків (LRU, TTL тощо).

### Мінімальний шаблон

```csharp
// ORIGINATOR
public class Originator
{
    private string _state;

    public void SetState(string state) => _state = state;
    public string GetState() => _state;

    public Memento CreateMemento() => new Memento(_state);

    public void RestoreMemento(Memento memento) => _state = memento.GetState();

    // MEMENTO - вкладений клас, доступ до вмісту обмежений
    public class Memento
    {
        private readonly string _state;
        internal Memento(string state) => _state = state;
        internal string GetState() => _state;
    }
}

// CARETAKER
public class Caretaker
{
    private readonly Stack<Originator.Memento> _history = new();

    public void Backup(Originator originator) => _history.Push(originator.CreateMemento());

    public void Undo(Originator originator)
    {
        if (_history.Count == 0) return;
        originator.RestoreMemento(_history.Pop());
    }
}
```

---

*Документ підготовлено для вивчення патернів проектування. Всі приклади протестовані на .NET 6+.*
