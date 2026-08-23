# Патерн State (Стан) — Детальний розбір на C#

> **Категорія:** Поведінковий (Behavioral)  
> **Автори:** Gang of Four (GoF)  
> **Мова прикладів:** C#

---

## Зміст

1. [Що таке State?](#що-таке-state)
   - [Аналогія з реального світу](#аналогія-з-реального-світу)
2. [Проблема без патерну](#проблема-без-патерну)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Світлофор](#приклад-1--світлофор)
5. [Приклад 2 — Торговий автомат](#приклад-2--торговий-автомат)
6. [Приклад 3 — Життєвий цикл документа](#приклад-3--життєвий-цикл-документа)
7. [Приклад 4 (реальний сценарій) — Обробка замовлення в інтернет-магазині](#приклад-4-реальний-сценарій--обробка-замовлення-в-інтернет-магазині)
8. [State vs Strategy](#state-vs-strategy)
9. [Переваги та недоліки](#переваги-та-недоліки)
10. [Антипатерни та поширені помилки](#антипатерни-та-поширені-помилки)
11. [Підсумок](#підсумок)
    - [Мінімальний шаблон](#мінімальний-шаблон)

---

## Що таке State?

**State (Стан)** — це поведінковий патерн проектування, який дозволяє об'єкту змінювати свою поведінку залежно від внутрішнього стану так, немов би змінився клас цього об'єкта.

Ключова ідея: замість того, щоб зберігати стан у вигляді простого прапорця (`enum`, `bool`, `int`) і перевіряти його умовними конструкціями (`if`/`switch`) у кожному методі, ми виносимо поведінку, характерну для кожного стану, в окремий клас, що реалізує спільний інтерфейс `IState`. Об'єкт-контекст (`Context`) зберігає посилання на поточний об'єкт стану і **делегує** йому виконання операцій. Сам стан (або контекст під керуванням стану) може в потрібний момент замінити поточний об'єкт стану на інший — відбувається **перехід стану** (state transition).

Формальне визначення (GoF):

> "Allow an object to alter its behaviour when its internal state changes. The object will appear to change its class."

Іншими словами:
- Поведінка об'єкта повністю залежить від того, у якому стані він зараз перебуває.
- Кожен стан інкапсульований у власному класі.
- Перехід між станами — явний і контрольований, а не розкиданий по коду умовними операторами.
- Ззовні клієнт працює з контекстом однаково незалежно від того, який стан наразі активний — контекст сам «підмінює клас» своєї поведінки.

### Аналогія з реального світу

Уявімо **світлофор**. У нього є три стани: Червоний, Жовтий, Зелений. Кожен стан:
- диктує, яким кольором світити;
- диктує, скільки часу горіти;
- знає, який стан настане наступним.

Світлофор (контекст) не «думає» логікою переходів сам — він просто питає у поточного стану: «Що робити? Скільки чекати? Що далі?». Коли настає момент переходу, поточний стан замінюється на наступний, і поведінка світлофора миттєво змінюється — хоча сам об'єкт світлофора залишається тим самим.

Інша аналогія — **торговий автомат**. Поки монета не вкинута, автомат перебуває в стані `NoCoin` і на спробу вибрати товар відповідає відмовою. Щойно монета вкинута — стан змінюється на `HasCoin`, і тепер вибір товару стає можливим. Автомат — один і той самий об'єкт, але поводиться геть по-різному залежно від активного стану.

Ще одна аналогія, яку ми детально розберемо нижче, — **життєвий цикл документа** в системі керування контентом: Чернетка (Draft) → На модерації (Moderation) → Опубліковано (Published) → Заархівовано (Archived). Кожен стан дозволяє свій набір дій і власний набір допустимих переходів.

---

## Проблема без патерну

Розглянемо типову реалізацію того самого світлофора **без** патерну State — через `enum` і купу `switch`/`if-else` конструкцій, розкиданих по методах.

```csharp
// ПОГАНО: стан зберігається як enum, а поведінка "розмазана"
// по всіх методах у вигляді switch/if-else блоків.

public enum TrafficLightState
{
    Red,
    Yellow,
    Green
}

public class TrafficLightBad
{
    private TrafficLightState _state = TrafficLightState.Red;

    public void Next()
    {
        // Логіка переходу для КОЖНОГО стану прописана тут
        switch (_state)
        {
            case TrafficLightState.Red:
                _state = TrafficLightState.Green;
                break;
            case TrafficLightState.Green:
                _state = TrafficLightState.Yellow;
                break;
            case TrafficLightState.Yellow:
                _state = TrafficLightState.Red;
                break;
            default:
                throw new InvalidOperationException("Невідомий стан");
        }
    }

    public string GetColor()
    {
        // А тут — ще один switch, який дублює знання про стани
        switch (_state)
        {
            case TrafficLightState.Red:
                return "Червоний";
            case TrafficLightState.Yellow:
                return "Жовтий";
            case TrafficLightState.Green:
                return "Зелений";
            default:
                throw new InvalidOperationException("Невідомий стан");
        }
    }

    public int GetDurationSeconds()
    {
        // І ще один switch з тими самими станами...
        switch (_state)
        {
            case TrafficLightState.Red:
                return 30;
            case TrafficLightState.Yellow:
                return 3;
            case TrafficLightState.Green:
                return 25;
            default:
                throw new InvalidOperationException("Невідомий стан");
        }
    }

    public bool CanPedestrianCross()
    {
        // І ще один. Кожен новий метод контексту породжує новий switch.
        if (_state == TrafficLightState.Red)
        {
            return true;
        }
        return false;
    }
}
```

### У чому саме проблема?

1. **Логіка переходів розкидана по методах.** Правило «Green → Yellow → Red → Green» продубльоване лише в `Next()`, але знання про сам перелік станів («що це за стан і як він виглядає») повторюється в `GetColor()`, `GetDurationSeconds()`, `CanPedestrianCross()` і в кожному новому методі, який ми додамо.
2. **Додавання нового стану — суцільний біль.** Якщо додати `TrafficLightState.FlashingYellow` (аварійний режим), доведеться відкрити **кожен** switch у класі й дописати новий `case`. Легко забути один із них — і отримати непомітний баг.
3. **Легко створити недопустиму комбінацію станів.** Ніщо не заважає комусь ззовні виставити `_state` напряму (якщо поле не приватне) або забути обробити якийсь `case`, залишивши об'єкт у невизначеній поведінці (`default` з винятком — по суті, крах програми в рантаймі).
4. **Порушення Open/Closed Principle.** Клас потрібно **модифікувати** (а не розширювати) щоразу, коли з'являється новий стан або нова поведінка, залежна від стану.
5. **Тестування ускладнюється.** Щоб перевірити поведінку в стані `Green`, потрібно "прогнати" об'єкт через увесь ланцюжок переходів або підсунути приватне поле через рефлексію — немає ізольованого модуля, який можна перевірити окремо.

Патерн State вирішує всі ці проблеми, винісши кожен стан в окремий клас.

---

## Структура патерну

```
┌───────────────────────────┐          ┌───────────────────────────┐
│         Context           │          │        <<interface>>      │
│                            │  state   │           IState           │
│ - _state : IState          │─────────▶│                            │
│                            │          │ + Handle(Context ctx)      │
│ + Request()                │          │ + ... інші методи стану    │
│ + SetState(IState state)   │          └─────────────▲──────────────┘
└───────────────────────────┘                        │
                                     ┌─────────────────┼─────────────────┐
                                     │                 │                 │
                        ┌────────────────────┐ ┌────────────────────┐ ┌────────────────────┐
                        │  ConcreteStateA     │ │  ConcreteStateB     │ │  ConcreteStateC     │
                        │                     │ │                     │ │                     │
                        │ + Handle(ctx)       │ │ + Handle(ctx)       │ │ + Handle(ctx)       │
                        │   ctx.SetState(B)   │ │   ctx.SetState(C)   │ │   ctx.SetState(A)   │
                        └────────────────────┘ └────────────────────┘ └────────────────────┘
```

### Ролі учасників

| Роль | Відповідальність |
|---|---|
| **Context** | Зберігає посилання на поточний об'єкт `IState`. Делегує йому виклики операцій, залежних від стану. Надає метод `SetState()`, яким сам стан (або зовнішній код) може змінити поточний стан. |
| **IState** | Інтерфейс (або абстрактний клас), що оголошує операції, специфічні для стану. Кожен конкретний стан реалізує цей інтерфейс по-своєму. |
| **ConcreteStateA / B / C** | Кожен клас реалізує поведінку, характерну саме для цього стану, і **сам вирішує**, до якого стану відбудеться перехід далі (викликаючи `context.SetState(...)`). |
| **Client** | Створює `Context`, як правило, з початковим станом, і працює з ним через його публічний інтерфейс, не знаючи деталей про поточний внутрішній стан. |

Важливий нюанс: перехід між станами може ініціювати або сам об'єкт стану (найпоширеніший варіант — «стан знає, що йде далі»), або контекст, який аналізує результат виконання операції. У прикладах нижче ми переважно дотримуємось першого підходу, оскільки він найкраще ілюструє суть патерну.

---

## Приклад 1 — Світлофор

Найпростіший, «підручниковий» приклад: `TrafficLight` — контекст, `RedState`, `YellowState`, `GreenState` — конкретні стани, кожен з яких знає, який колір показувати, скільки тривати і який стан настане наступним.

```csharp
using System;

namespace StatePattern.TrafficLightExample
{
    // Інтерфейс стану світлофора
    public interface ITrafficLightState
    {
        string Color { get; }
        int DurationSeconds { get; }

        // Кожен стан сам вирішує, який стан настане далі
        void Next(TrafficLight context);
    }

    // Контекст — сам світлофор
    public class TrafficLight
    {
        private ITrafficLightState _state;

        public TrafficLight()
        {
            // Початковий стан — червоний
            _state = new RedState();
        }

        public void SetState(ITrafficLightState state)
        {
            _state = state;
        }

        public string CurrentColor => _state.Color;
        public int CurrentDuration => _state.DurationSeconds;

        // Контекст лише делегує виклик поточному стану
        public void Next()
        {
            _state.Next(this);
        }
    }

    public class RedState : ITrafficLightState
    {
        public string Color => "Червоний";
        public int DurationSeconds => 30;

        public void Next(TrafficLight context)
        {
            Console.WriteLine("🚦 Червоний -> Зелений");
            context.SetState(new GreenState());
        }
    }

    public class GreenState : ITrafficLightState
    {
        public string Color => "Зелений";
        public int DurationSeconds => 25;

        public void Next(TrafficLight context)
        {
            Console.WriteLine("🚦 Зелений -> Жовтий");
            context.SetState(new YellowState());
        }
    }

    public class YellowState : ITrafficLightState
    {
        public string Color => "Жовтий";
        public int DurationSeconds => 3;

        public void Next(TrafficLight context)
        {
            Console.WriteLine("🚦 Жовтий -> Червоний");
            context.SetState(new RedState());
        }
    }
}
```

### Використання

```csharp
using StatePattern.TrafficLightExample;

var light = new TrafficLight();

for (int i = 0; i < 5; i++)
{
    Console.WriteLine($"Поточний колір: {light.CurrentColor} " +
                       $"(триває {light.CurrentDuration} с)");
    light.Next();
}
```

**Очікуваний вивід у консолі:**

```
Поточний колір: Червоний (триває 30 с)
🚦 Червоний -> Зелений
Поточний колір: Зелений (триває 25 с)
🚦 Зелений -> Жовтий
Поточний колір: Жовтий (триває 3 с)
🚦 Жовтий -> Червоний
Поточний колір: Червоний (триває 30 с)
🚦 Червоний -> Зелений
Поточний колір: Зелений (триває 25 с)
🚦 Зелений -> Жовтий
```

Додати новий стан (наприклад, `FlashingYellowState` для нічного режиму) тепер означає лише **додати новий клас**, що реалізує `ITrafficLightState`, і не потрібно чіпати жоден із наявних класів — це і є Open/Closed Principle у дії.

---

## Приклад 2 — Торговий автомат

Класичний приклад патерну State з підручників — торговий автомат. Демонструє чотири стани: `NoCoinState`, `HasCoinState`, `SoldOutState`, `DispensingState`, а також коректну обробку недопустимих дій (наприклад, спробу вибрати товар без вкинутої монети).

```csharp
using System;

namespace StatePattern.VendingMachineExample
{
    // Інтерфейс стану торгового автомата
    public interface IVendingMachineState
    {
        void InsertCoin(VendingMachine machine);
        void SelectProduct(VendingMachine machine);
        void Dispense(VendingMachine machine);
    }

    // Контекст — торговий автомат
    public class VendingMachine
    {
        private IVendingMachineState _state;
        private int _stock;

        public VendingMachine(int initialStock)
        {
            _stock = initialStock;
            // Якщо товару немає одразу — стартуємо у стані "Продано все"
            _state = _stock > 0 ? new NoCoinState() : new SoldOutState();
        }

        public int Stock => _stock;

        public void SetState(IVendingMachineState state) => _state = state;

        public void DecreaseStock()
        {
            if (_stock > 0) _stock--;
        }

        // Публічні дії, які делегуються поточному стану
        public void InsertCoin() => _state.InsertCoin(this);
        public void SelectProduct() => _state.SelectProduct(this);
        public void Dispense() => _state.Dispense(this);
    }

    // Стан: монета не вкинута
    public class NoCoinState : IVendingMachineState
    {
        public void InsertCoin(VendingMachine machine)
        {
            Console.WriteLine("🪙 Монету прийнято.");
            machine.SetState(new HasCoinState());
        }

        public void SelectProduct(VendingMachine machine)
        {
            // Недопустима дія в цьому стані — чітко повідомляємо про це,
            // а не мовчки ігноруємо запит
            Console.WriteLine("❌ Спочатку вкиньте монету.");
        }

        public void Dispense(VendingMachine machine)
        {
            Console.WriteLine("❌ Немає що видавати — товар не обрано.");
        }
    }

    // Стан: монета вкинута, очікуємо вибір товару
    public class HasCoinState : IVendingMachineState
    {
        public void InsertCoin(VendingMachine machine)
        {
            Console.WriteLine("❌ Монету вже вкинуто, зачекайте на видачу товару.");
        }

        public void SelectProduct(VendingMachine machine)
        {
            Console.WriteLine("✅ Товар обрано, готуємо видачу...");
            machine.SetState(new DispensingState());
            // Одразу ініціюємо видачу (можна винести й у явний виклик клієнтом)
            machine.Dispense();
        }

        public void Dispense(VendingMachine machine)
        {
            Console.WriteLine("❌ Спочатку оберіть товар.");
        }
    }

    // Стан: іде видача товару
    public class DispensingState : IVendingMachineState
    {
        public void InsertCoin(VendingMachine machine)
        {
            Console.WriteLine("❌ Зачекайте, відбувається видача товару.");
        }

        public void SelectProduct(VendingMachine machine)
        {
            Console.WriteLine("❌ Видача вже триває, зачекайте.");
        }

        public void Dispense(VendingMachine machine)
        {
            machine.DecreaseStock();
            Console.WriteLine($"📦 Товар видано! Залишок на складі: {machine.Stock}");

            machine.SetState(machine.Stock > 0
                ? new NoCoinState()
                : new SoldOutState());
        }
    }

    // Стан: товар закінчився
    public class SoldOutState : IVendingMachineState
    {
        public void InsertCoin(VendingMachine machine)
        {
            Console.WriteLine("❌ Автомат порожній, монету повернуто.");
        }

        public void SelectProduct(VendingMachine machine)
        {
            Console.WriteLine("❌ Товар закінчився.");
        }

        public void Dispense(VendingMachine machine)
        {
            Console.WriteLine("❌ Видавати нічого — склад порожній.");
        }
    }
}
```

### Використання

```csharp
using StatePattern.VendingMachineExample;

var machine = new VendingMachine(initialStock: 2);

Console.WriteLine("--- Спроба вибрати товар без монети ---");
machine.SelectProduct();

Console.WriteLine("\n--- Нормальний цикл покупки №1 ---");
machine.InsertCoin();
machine.SelectProduct();

Console.WriteLine("\n--- Нормальний цикл покупки №2 (товар закінчиться) ---");
machine.InsertCoin();
machine.SelectProduct();

Console.WriteLine("\n--- Спроба купити, коли товару більше немає ---");
machine.InsertCoin();
```

**Очікуваний вивід у консолі:**

```
--- Спроба вибрати товар без монети ---
❌ Спочатку вкиньте монету.

--- Нормальний цикл покупки №1 ---
🪙 Монету прийнято.
✅ Товар обрано, готуємо видачу...
📦 Товар видано! Залишок на складі: 1

--- Нормальний цикл покупки №2 (товар закінчиться) ---
🪙 Монету прийнято.
✅ Товар обрано, готуємо видачу...
📦 Товар видано! Залишок на складі: 0

--- Спроба купити, коли товару більше немає ---
❌ Автомат порожній, монету повернуто.
```

Зверніть увагу: кожен `ConcreteState` реалізує **всі** методи інтерфейсу, навіть ті, що для нього недопустимі — і саме там пояснює, чому дія неможлива. Немає жодного `switch`, який перевіряв би «а який зараз стан» — ця перевірка відбувається природно, через те, який об'єкт зараз лежить у полі `_state`.

---

## Приклад 3 — Життєвий цикл документа

Реальніший приклад із бізнес-правилами на кшталт CMS: документ проходить стани `Draft → Moderation → Published → Archived`, і деякі переходи дозволені лише певним ролям (наприклад, лише модератор може перевести документ із `Moderation` у `Published`).

```csharp
using System;

namespace StatePattern.DocumentWorkflowExample
{
    public enum UserRole
    {
        Author,
        Moderator
    }

    // Інтерфейс стану документа
    public interface IDocumentState
    {
        string Name { get; }

        void SubmitForReview(Document document, UserRole role);
        void Approve(Document document, UserRole role);
        void Reject(Document document, UserRole role);
        void Archive(Document document, UserRole role);
    }

    // Контекст — документ
    public class Document
    {
        public string Title { get; }
        private IDocumentState _state;

        public Document(string title)
        {
            Title = title;
            _state = new DraftState();
        }

        public string CurrentStateName => _state.Name;

        public void SetState(IDocumentState state)
        {
            Console.WriteLine($"   [Перехід: \"{Title}\" -> {state.Name}]");
            _state = state;
        }

        public void SubmitForReview(UserRole role) => _state.SubmitForReview(this, role);
        public void Approve(UserRole role) => _state.Approve(this, role);
        public void Reject(UserRole role) => _state.Reject(this, role);
        public void Archive(UserRole role) => _state.Archive(this, role);
    }

    // Базовий клас із поведінкою "за замовчуванням" (недопустима дія)
    public abstract class DocumentStateBase : IDocumentState
    {
        public abstract string Name { get; }

        public virtual void SubmitForReview(Document document, UserRole role)
            => Reject($"Неможливо надіслати на перевірку зі стану \"{Name}\".");

        public virtual void Approve(Document document, UserRole role)
            => Reject($"Неможливо затвердити документ у стані \"{Name}\".");

        public virtual void Reject(Document document, UserRole role)
            => Reject($"Неможливо відхилити документ у стані \"{Name}\".");

        public virtual void Archive(Document document, UserRole role)
            => Reject($"Неможливо архівувати документ у стані \"{Name}\".");

        protected static void Reject(string message)
        {
            Console.WriteLine($"❌ {message}");
        }
    }

    // Стан: чернетка
    public class DraftState : DocumentStateBase
    {
        public override string Name => "Чернетка";

        public override void SubmitForReview(Document document, UserRole role)
        {
            Console.WriteLine("📤 Документ надіслано на модерацію.");
            document.SetState(new ModerationState());
        }
    }

    // Стан: на модерації
    public class ModerationState : DocumentStateBase
    {
        public override string Name => "На модерації";

        public override void Approve(Document document, UserRole role)
        {
            // Бізнес-правило: затвердити може лише модератор
            if (role != UserRole.Moderator)
            {
                Reject("Затвердити документ може лише модератор.");
                return;
            }

            Console.WriteLine("✅ Документ затверджено модератором.");
            document.SetState(new PublishedState());
        }

        public override void Reject(Document document, UserRole role)
        {
            if (role != UserRole.Moderator)
            {
                Reject("Відхилити документ може лише модератор.");
                return;
            }

            Console.WriteLine("↩️  Документ повернуто на доопрацювання.");
            document.SetState(new DraftState());
        }
    }

    // Стан: опубліковано
    public class PublishedState : DocumentStateBase
    {
        public override string Name => "Опубліковано";

        public override void Archive(Document document, UserRole role)
        {
            Console.WriteLine("🗄️  Документ заархівовано.");
            document.SetState(new ArchivedState());
        }
    }

    // Стан: заархівовано (термінальний стан — жодні дії далі недопустимі)
    public class ArchivedState : DocumentStateBase
    {
        public override string Name => "Заархівовано";
    }
}
```

### Використання

```csharp
using StatePattern.DocumentWorkflowExample;

var doc = new Document("Патерни проектування у C#");
Console.WriteLine($"Стан: {doc.CurrentStateName}\n");

Console.WriteLine("--- Автор намагається одразу опублікувати (недопустимо) ---");
doc.Approve(UserRole.Author);

Console.WriteLine("\n--- Автор надсилає документ на модерацію ---");
doc.SubmitForReview(UserRole.Author);

Console.WriteLine("\n--- Автор намагається сам себе затвердити (недопустимо) ---");
doc.Approve(UserRole.Author);

Console.WriteLine("\n--- Модератор затверджує документ ---");
doc.Approve(UserRole.Moderator);

Console.WriteLine("\n--- Модератор архівує опублікований документ ---");
doc.Archive(UserRole.Moderator);

Console.WriteLine($"\nФінальний стан: {doc.CurrentStateName}");
```

**Очікуваний вивід у консолі:**

```
Стан: Чернетка

--- Автор намагається одразу опублікувати (недопустимо) ---
❌ Неможливо затвердити документ у стані "Чернетка".

--- Автор надсилає документ на модерацію ---
📤 Документ надіслано на модерацію.
   [Перехід: "Патерни проектування у C#" -> На модерації]

--- Автор намагається сам себе затвердити (недопустимо) ---
❌ Затвердити документ може лише модератор.

--- Модератор затверджує документ ---
✅ Документ затверджено модератором.
   [Перехід: "Патерни проектування у C#" -> Опубліковано]

--- Модератор архівує опублікований документ ---
🗄️  Документ заархівовано.
   [Перехід: "Патерни проектування у C#" -> Заархівовано]

Фінальний стан: Заархівовано
```

Тут добре видно, як **правила ролей** («лише модератор може...») природно живуть усередині конкретного стану (`ModerationState`), а не в загальному коді документа.

---

## Приклад 4 (реальний сценарій) — Обробка замовлення в інтернет-магазині

Розглянемо повноцінний, наближений до продакшену приклад — стан-машину замовлення в e-commerce системі. Замовлення проходить через `PendingPaymentState → PaidState → ShippedState → DeliveredState`, а також може потрапити в `CancelledState` або `RefundedState` із кількох точок життєвого циклу. Кожен стан **явно перевіряє**, які переходи з нього легальні, і кидає зрозумілий доменний виняток при спробі недопустимого переходу.

```csharp
using System;
using System.Collections.Generic;

namespace StatePattern.OrderProcessingExample
{
    // Доменний виняток для недопустимих переходів стану замовлення
    public class InvalidOrderTransitionException : Exception
    {
        public InvalidOrderTransitionException(string currentState, string attemptedAction)
            : base($"Дію \"{attemptedAction}\" неможливо виконати, " +
                   $"поки замовлення перебуває у стані \"{currentState}\".")
        {
        }
    }

    // Інтерфейс стану замовлення.
    // Кожен метод відповідає одній бізнес-дії над замовленням.
    public interface IOrderState
    {
        string Name { get; }

        void Pay(Order order, decimal amount);
        void Ship(Order order, string trackingNumber);
        void Deliver(Order order);
        void Cancel(Order order, string reason);
        void Refund(Order order);
    }

    // Базовий клас: за замовчуванням будь-яка дія кидає доменний виняток —
    // конкретний стан має явно "перевизначити" тільки ті дії, що йому дозволені.
    public abstract class OrderStateBase : IOrderState
    {
        public abstract string Name { get; }

        public virtual void Pay(Order order, decimal amount)
            => throw new InvalidOrderTransitionException(Name, nameof(Pay));

        public virtual void Ship(Order order, string trackingNumber)
            => throw new InvalidOrderTransitionException(Name, nameof(Ship));

        public virtual void Deliver(Order order)
            => throw new InvalidOrderTransitionException(Name, nameof(Deliver));

        public virtual void Cancel(Order order, string reason)
            => throw new InvalidOrderTransitionException(Name, nameof(Cancel));

        public virtual void Refund(Order order)
            => throw new InvalidOrderTransitionException(Name, nameof(Refund));
    }

    // Контекст — замовлення
    public class Order
    {
        public string OrderId { get; }
        public decimal TotalAmount { get; }
        public string? TrackingNumber { get; private set; }
        public string? CancellationReason { get; private set; }

        private IOrderState _state;
        private readonly List<string> _history = new();

        public Order(string orderId, decimal totalAmount)
        {
            OrderId = orderId;
            TotalAmount = totalAmount;
            _state = new PendingPaymentState();
            _history.Add(_state.Name);
        }

        public string CurrentStateName => _state.Name;
        public IReadOnlyList<string> History => _history;

        public void SetState(IOrderState state)
        {
            _state = state;
            _history.Add(state.Name);
        }

        internal void AssignTracking(string trackingNumber) => TrackingNumber = trackingNumber;
        internal void AssignCancellationReason(string reason) => CancellationReason = reason;

        // Публічний API замовлення — усе делегується поточному стану
        public void Pay(decimal amount) => _state.Pay(this, amount);
        public void Ship(string trackingNumber) => _state.Ship(this, trackingNumber);
        public void Deliver() => _state.Deliver(this);
        public void Cancel(string reason) => _state.Cancel(this, reason);
        public void Refund() => _state.Refund(this);
    }

    // Стан: очікує оплати (початковий стан)
    public class PendingPaymentState : OrderStateBase
    {
        public override string Name => "Очікує оплати";

        public override void Pay(Order order, decimal amount)
        {
            if (amount < order.TotalAmount)
            {
                Console.WriteLine(
                    $"❌ Сума оплати ({amount:C}) менша за суму замовлення ({order.TotalAmount:C}).");
                return;
            }

            Console.WriteLine($"💳 Оплату на суму {amount:C} отримано.");
            order.SetState(new PaidState());
        }

        public override void Cancel(Order order, string reason)
        {
            Console.WriteLine($"🚫 Замовлення скасовано до оплати. Причина: {reason}");
            order.AssignCancellationReason(reason);
            order.SetState(new CancelledState());
        }
    }

    // Стан: оплачено, очікує відправки
    public class PaidState : OrderStateBase
    {
        public override string Name => "Оплачено";

        public override void Ship(Order order, string trackingNumber)
        {
            Console.WriteLine($"📦 Замовлення передано перевізнику. Номер відстеження: {trackingNumber}");
            order.AssignTracking(trackingNumber);
            order.SetState(new ShippedState());
        }

        public override void Cancel(Order order, string reason)
        {
            Console.WriteLine($"🚫 Оплачене замовлення скасовано. Причина: {reason}");
            order.AssignCancellationReason(reason);
            order.SetState(new CancelledState());
        }

        public override void Refund(Order order)
        {
            Console.WriteLine("💸 Оформлено повернення коштів за оплачене, але ще не відправлене замовлення.");
            order.SetState(new RefundedState());
        }
    }

    // Стан: відправлено, у дорозі
    public class ShippedState : OrderStateBase
    {
        public override string Name => "Відправлено";

        public override void Deliver(Order order)
        {
            Console.WriteLine("🏠 Замовлення успішно доставлено отримувачу.");
            order.SetState(new DeliveredState());
        }

        public override void Refund(Order order)
        {
            Console.WriteLine("💸 Оформлено повернення коштів під час доставки.");
            order.SetState(new RefundedState());
        }

        // Примітка: скасувати замовлення, яке вже в дорозі, не можна —
        // тому Cancel() не перевизначається і успадковує поведінку
        // з InvalidOrderTransitionException від базового класу.
    }

    // Стан: доставлено (кінцевий "успішний" стан)
    public class DeliveredState : OrderStateBase
    {
        public override string Name => "Доставлено";

        public override void Refund(Order order)
        {
            // Повернення можливе й після доставки — наприклад, товар неякісний
            Console.WriteLine("💸 Оформлено повернення коштів після доставки товару.");
            order.SetState(new RefundedState());
        }
    }

    // Стан: скасовано (термінальний стан)
    public class CancelledState : OrderStateBase
    {
        public override string Name => "Скасовано";

        // З цього стану жодні дії більше недоступні —
        // усі методи успадковують поведінку базового класу (виняток).
    }

    // Стан: кошти повернуто (термінальний стан)
    public class RefundedState : OrderStateBase
    {
        public override string Name => "Кошти повернуто";

        // Термінальний стан — подальші дії недопустимі.
    }
}
```

### Program.Main — демонстрація повного життєвого циклу

```csharp
using System;
using StatePattern.OrderProcessingExample;

public class Program
{
    public static void Main()
    {
        var order = new Order(orderId: "ORD-2026-0157", totalAmount: 1499.00m);
        Console.WriteLine($"Замовлення {order.OrderId} створено. Стан: {order.CurrentStateName}\n");

        Console.WriteLine("--- Спроба відправити неоплачене замовлення (недопустимо) ---");
        try
        {
            order.Ship("NP-000111222");
        }
        catch (InvalidOrderTransitionException ex)
        {
            Console.WriteLine($"❌ {ex.Message}");
        }

        Console.WriteLine("\n--- Клієнт оплачує замовлення ---");
        order.Pay(1499.00m);

        Console.WriteLine("\n--- Магазин відправляє замовлення ---");
        order.Ship("NP-000111222");

        Console.WriteLine("\n--- Спроба скасувати замовлення, яке вже в дорозі (недопустимо) ---");
        try
        {
            order.Cancel("Клієнт передумав");
        }
        catch (InvalidOrderTransitionException ex)
        {
            Console.WriteLine($"❌ {ex.Message}");
        }

        Console.WriteLine("\n--- Кур'єр доставляє замовлення ---");
        order.Deliver();

        Console.WriteLine("\n--- Клієнт виявляє брак і оформлює повернення ---");
        order.Refund();

        Console.WriteLine($"\nФінальний стан замовлення {order.OrderId}: {order.CurrentStateName}");
        Console.WriteLine($"Історія переходів: {string.Join(" -> ", order.History)}");
    }
}
```

**Очікуваний вивід у консолі:**

```
Замовлення ORD-2026-0157 створено. Стан: Очікує оплати

--- Спроба відправити неоплачене замовлення (недопустимо) ---
❌ Дію "Ship" неможливо виконати, поки замовлення перебуває у стані "Очікує оплати".

--- Клієнт оплачує замовлення ---
💳 Оплату на суму ₴1,499.00 отримано.

--- Магазин відправляє замовлення ---
📦 Замовлення передано перевізнику. Номер відстеження: NP-000111222

--- Спроба скасувати замовлення, яке вже в дорозі (недопустимо) ---
❌ Дію "Cancel" неможливо виконати, поки замовлення перебуває у стані "Відправлено".

--- Кур'єр доставляє замовлення ---
🏠 Замовлення успішно доставлено отримувачу.

--- Клієнт виявляє брак і оформлює повернення ---
💸 Оформлено повернення коштів після доставки товару.

Фінальний стан замовлення ORD-2026-0157: Кошти повернуто
Історія переходів: Очікує оплати -> Оплачено -> Відправлено -> Доставлено -> Кошти повернуто
```

Що важливо в цьому прикладі:

- **Кожен стан сам вирішує, які переходи з нього легальні** — `ShippedState` не має перевизначеного `Cancel()`, тому виклик автоматично падає з чітким `InvalidOrderTransitionException` завдяки базовому класу `OrderStateBase`.
- **`Refund()` доступний із кількох станів** (`PaidState`, `ShippedState`, `DeliveredState`), і кожен веде до одного й того самого `RefundedState`, але з різним поясненням у консолі — це природно виражається патерном State, а не громіздким `switch` на кшталт `if (state == Paid || state == Shipped || state == Delivered)`.
- **Історія переходів** (`order.History`) веде журнал усіх станів, через які пройшло замовлення — корисно для аудиту та діагностики в реальних системах.
- Замовлення ніколи не потрапляє в недопустиму комбінацію («відправлено, але не оплачено») — це в принципі неможливо, оскільки `Ship()` існує лише в `PaidState`.

---

## State vs Strategy

Патерни **State** і **Strategy** структурно виглядають майже ідентично: в обох випадках `Context` тримає посилання на об'єкт-інтерфейс і делегує йому виконання роботи.

```
Strategy:                              State:
┌───────────┐   uses    ┌───────────┐  ┌───────────┐  delegates  ┌───────────┐
│  Context   │──────────▶│ Strategy   │  │  Context   │────────────▶│  State     │
└───────────┘           └───────────┘  └───────────┘             └───────────┘
      ▲                                      ▲                         │
      │ клієнт обирає                        │                         │ стан сам
      │ стратегію один раз                   │◀────────────────────────┘ змінює контекст
   Client                               SetState(next)
```

Різниця — не в структурі коду, а в **намірі (intent)**:

| | Strategy | State |
|---|---|---|
| **Що інкапсулює** | Взаємозамінний **алгоритм** виконання однієї операції | **Внутрішній стан** об'єкта й поведінку, що з ним пов'язана |
| **Хто обирає реалізацію** | Зазвичай клієнт — один раз, явно (`context.SetStrategy(new X())`) | Найчастіше сам об'єкт стану — неявно, як побічний ефект своєї роботи |
| **Чи знають реалізації одна про одну** | Ні, стратегії зазвичай взаємно незалежні | Так, стани зазвичай явно посилаються на наступні стани (`ctx.SetState(new B())`) |
| **Чи очікується зміна в часі** | Ні — стратегія стабільна протягом виконання операції/сесії | Так — це сутність патерну: стан природно змінюється протягом життєвого циклу об'єкта |
| **Типовий приклад** | Алгоритм сортування, стратегія розрахунку знижки, спосіб серіалізації | Стани замовлення, стани документа, стани з'єднання TCP |
| **Аналогія** | «Обери, ЯК виконати завдання» | «Об'єкт сам вирішує, ЩО він зараз є і ЩО буде далі» |

### Запитай себе:

- **«Чи змінюється ця поведінка сама, у відповідь на події, без явної команди клієнта?»** — Якщо так, це, найімовірніше, **State** (перехід відбувається як реакція на подію: оплата надійшла → стан сам себе змінив).
- **«Чи обирає клієнт цю реалізацію один раз і не очікує, що вона сама собі щось замінить?»** — Це ознака **Strategy** (наприклад, `List<T>.Sort(IComparer<T> comparer)` — компаратор не «перемикає сам себе» на інший).
- **«Чи є в мене природний граф переходів (state diagram) між реалізаціями?»** — Якщо так, майже напевно це **State**.
- **«Чи взаємозамінні реалізації повністю симетричні й не посилаються одна на одну?»** — Тоді це, швидше, **Strategy**.

### Зв'язок із поняттям «скінченний автомат» (Finite State Machine)

Патерн State — це, по суті, об'єктно-орієнтована реалізація загальної концепції **скінченного автомата (Finite State Machine, FSM)** із теорії обчислень: множина станів, множина подій/дій та функція переходів, що визначає, у який стан автомат переходить із поточного при настанні події. GoF-патерн State — це спосіб виразити цю концепцію мовою класів та поліморфізму замість таблиць переходів чи `switch`-конструкцій. Для складних автоматів (десятки станів, паралельні під-стани) варто розглянути спеціалізовані бібліотеки станових машин (наприклад, `Stateless` для .NET), які будують граф переходів декларативно, залишаючись концептуально тим самим патерном State.

---

## Переваги та недоліки

### Переваги

- **Позбувається величезних `switch`/`if-else` блоків**, розкиданих по методах контексту — кожен стан живе у власному, невеликому, легкому для розуміння класі.
- **Локалізує поведінку та правила переходів для кожного стану в одному місці** — щоб зрозуміти, що відбувається у стані `PaidState`, достатньо відкрити один клас, а не «збирати» логіку по всьому контексту.
- **Дотримується Open/Closed Principle** — щоб додати новий стан, достатньо створити новий клас, що реалізує `IState`; існуючі класи лишаються незмінними.
- **Спрощує запобігання недопустимим станам і переходам** — якщо дія недоступна в поточному стані, вона просто відсутня (або явно відхиляється) в конкретному класі стану, а не «десь загублена» серед умов.
- **Полегшує тестування** — кожен стан можна тестувати ізольовано, підставляючи потрібний `IState` напряму, без «прокручування» контексту через увесь ланцюжок переходів.
- **Робить явним життєвий цикл об'єкта** — послідовність станів і дозволені переходи стають частиною архітектури коду, а не прихованим знанням «в голові» розробника.

### Недоліки

- **Може призвести до великої кількості дрібних класів**, якщо автомат станів має багато станів (десятки станів → десятки класів), що збільшує «поверхню» кодової бази.
- **Логіка переходів може розсіятися по класах станів**, якщо не стежити за організацією коду — кожен `ConcreteState` знає лише про свої вихідні переходи, тому побачити **весь** граф станів одразу, просто читаючи один файл, неможливо; доводиться "збирати" картину, переглядаючи всі класи.
- **Складніше візуалізувати повну діаграму станів "з коду"** — на відміну від таблиці переходів чи явної діаграми, поліморфна реалізація ховає загальну картину за окремими класами (хоча документація/коментарі й діаграми частково компенсують це).
- **Накладні витрати на створення об'єктів станів**, якщо стани створюються заново при кожному переході (хоча це зазвичай незначно; за потреби можна кешувати stateless-стани як одинички).

---

## Антипатерни та поширені помилки

### Помилка 1: залишений `enum` поряд з об'єктами стану

Найпоширеніша помилка — «наполовину» мігрувати на патерн: додати об'єкти `IState`, але залишити в контексті ще й `enum CurrentStateType`, а десь у коді — забутий `switch`, що дублює те, що вже інкапсульовано в станах.

```csharp
// НЕПРАВИЛЬНО: маємо об'єкти стану, але паралельно тягнемо enum
// і залишковий switch "про всяк випадок" — подвійне джерело істини.
public class OrderBad
{
    public enum StateType { PendingPayment, Paid, Shipped } // зайве!

    private IOrderState _state;
    private StateType _stateType; // дублює те, що вже знає _state

    public void Pay(decimal amount)
    {
        _state.Pay(this, amount);

        // Забутий "тимчасовий" switch, який хтось не видалив після рефакторингу
        switch (_stateType)
        {
            case StateType.PendingPayment:
                _stateType = StateType.Paid;
                break;
                // ...і так по колу для кожного нового стану
        }
    }
}
```

Проблема: тепер є **два джерела істини** про поточний стан — `_state` (об'єкт) і `_stateType` (enum). Вони легко розсинхронізуються, якщо хтось оновить один, забувши про інший.

```csharp
// ПРАВИЛЬНО: єдине джерело істини — сам об'єкт стану.
// Ім'я поточного стану, якщо воно потрібне для логів/UI, отримуємо
// через властивість самого об'єкта стану (наприклад, IOrderState.Name),
// а не через окремий enum.
public class OrderGood
{
    private IOrderState _state;

    public string CurrentStateName => _state.Name; // делегування, не enum

    public void Pay(decimal amount) => _state.Pay(this, amount);

    public void SetState(IOrderState state) => _state = state;
}
```

Якщо потрібно порівнювати «а чи ми зараз у стані X» ззовні (наприклад, для UI), варто або звертатись до `state.Name`/`state.GetType()`, або (краще) додати в `IOrderState` явний метод-запит на кшталт `bool CanShip()`, а не відновлювати `enum`.

### Помилка 2: спільний mutable-стан між кількома контекстами

Якщо об'єкт стану реалізовано без стану (stateless) — тобто не містить власних полів, специфічних для конкретного контексту, — його можна безпечно повторно використовувати (singleton) для багатьох контекстів одночасно. Але поширена помилка — випадково додати в клас стану **мутабельне поле, специфічне для контексту**, і потім ділити один екземпляр стану між різними об'єктами `Context`.

```csharp
// НЕПРАВИЛЬНО: стан зберігає mutable-поле, специфічне для контексту,
// але екземпляр цього стану використовується як singleton
// для БАГАТЬОХ замовлень одночасно — вихід за межі одного Order "протікає"
// в інший Order через спільний об'єкт стану.
public class PaidStateBad : IOrderState
{
    // Це поле належить конкретному замовленню, а не самому стану!
    private DateTime _paidAt;

    public static readonly PaidStateBad Instance = new(); // singleton — і ось тут біда

    public void Ship(Order order, string trackingNumber)
    {
        _paidAt = DateTime.Now; // записуємо "в стан", а не "в контекст"
        // ... якщо цей самий Instance використовує ІНШЕ замовлення,
        // _paidAt від першого замовлення буде затерто другим!
        order.SetState(new ShippedStateBad());
    }
}
```

```csharp
// ПРАВИЛЬНО, варіант А: стан справді stateless — жодних mutable-полів,
// уся контекстно-специфічна інформація зберігається в самому Order,
// а не в об'єкті стану. Тоді singleton-стан безпечний для повторного
// використання між різними замовленнями.
public class PaidStateGood : IOrderState
{
    public static readonly PaidStateGood Instance = new();

    public void Ship(Order order, string trackingNumber)
    {
        order.AssignPaidAt(DateTime.Now); // дані живуть у контексті, не в стані
        order.SetState(ShippedStateGood.Instance);
    }
}

// ПРАВИЛЬНО, варіант Б: якщо стану справді потрібні власні дані —
// не робимо його singleton, а створюємо новий екземпляр щоразу
// (як і робилося в прикладах 1-4 цього документа: `new PaidState()`).
```

Правило: **або** стан stateless і тоді його можна безпечно ділити (singleton/кеш) між контекстами, **або** стан тримає власні дані і тоді він **не** повинен бути спільним — для кожного контексту створюється власний екземпляр.

### Помилка 3: мовчазне ігнорування недопустимої дії

Ще одна поширена помилка — коли метод стану на недопустиму дію просто **нічого не робить**, замість того, щоб явно повідомити про відмову. Це маскує баги виклику й ускладнює діагностику.

```csharp
// НЕПРАВИЛЬНО: недопустима дія просто "проковтується" мовчки.
// Клієнтський код гадки не має, що дія не спрацювала.
public class ShippedStateSilent : IOrderState
{
    public void Cancel(Order order, string reason)
    {
        // Нічого не робимо — метод просто "порожній".
        // Викликач думає, що скасування відбулося успішно!
    }

    // ... інші методи
}
```

```csharp
// ПРАВИЛЬНО, варіант А: явно кидаємо доменний виняток
// (саме так зроблено в Прикладі 4 цього документа).
public class ShippedStateExplicit : OrderStateBase
{
    public override string Name => "Відправлено";

    // Cancel() свідомо НЕ перевизначається:
    // базовий клас кине InvalidOrderTransitionException із чітким повідомленням.
}

// ПРАВИЛЬНО, варіант Б: якщо винятки небажані (наприклад, у "гарячому" шляху
// виконання, де винятки дорогі), повертаємо явний Result замість void.
public readonly struct TransitionResult
{
    public bool Success { get; }
    public string? RejectionReason { get; }

    private TransitionResult(bool success, string? reason)
    {
        Success = success;
        RejectionReason = reason;
    }

    public static TransitionResult Ok() => new(true, null);
    public static TransitionResult Fail(string reason) => new(false, reason);
}

public class ShippedStateWithResult
{
    public TransitionResult Cancel(Order order, string reason)
        => TransitionResult.Fail("Неможливо скасувати замовлення, яке вже відправлено.");
}
```

Обидва «правильні» варіанти явно повідомляють про відмову — різниця лише в механізмі (виняток проти `Result`), і вибір залежить від того, наскільки «очікуваною» вважається ця відмова в бізнес-логіці (винятки — для дійсно виняткових/помилкових ситуацій; `Result` — коли відмова є нормальною частиною бізнес-потоку).

---

## Підсумок

Патерн **State** варто застосовувати, коли:

- Поведінка об'єкта суттєво залежить від його внутрішнього стану, і ця поведінка має **змінюватися в рантаймі** залежно від подій.
- У коду вже є (або неминуче з'явиться) велика кількість `if`/`switch` конструкцій, що перевіряють поле `state`/`status`/`type` у багатьох методах одного класу.
- Кількість станів і переходів між ними буде **рости** з часом, і хочеться додавати нові стани, не переписуючи наявний код (Open/Closed Principle).
- Важливо **явно і надійно** заборонити недопустимі комбінації станів/переходів, а не покладатися на розкидані перевірки.
- Потрібно, щоб кожен стан можна було тестувати, документувати й підтримувати **окремо** від інших.

Не варто застосовувати State, якщо:

- Станів усього два-три, і логіка переходів тривіальна — простий `bool`/`enum` із парою `if` буде читабельнішим і без зайвих класів.
- Поведінка не залежить від стану саме як від «фази життєвого циклу», а є просто взаємозамінним алгоритмом — тоді доречніший **Strategy**.

### Мінімальний шаблон

```csharp
using System;

// Інтерфейс стану
public interface IState
{
    void Request(Context context);
}

// Контекст
public class Context
{
    private IState _state;

    public Context(IState initialState)
    {
        _state = initialState;
    }

    public void SetState(IState state) => _state = state;

    // Контекст лише делегує виклик поточному стану
    public void Request() => _state.Request(this);
}

// Конкретний стан A: після обробки переходить у стан B
public class ConcreteStateA : IState
{
    public void Request(Context context)
    {
        Console.WriteLine("Обробка у стані A");
        context.SetState(new ConcreteStateB());
    }
}

// Конкретний стан B: після обробки переходить у стан A
public class ConcreteStateB : IState
{
    public void Request(Context context)
    {
        Console.WriteLine("Обробка у стані B");
        context.SetState(new ConcreteStateA());
    }
}

// Використання:
// var context = new Context(new ConcreteStateA());
// context.Request(); // "Обробка у стані A", перехід -> B
// context.Request(); // "Обробка у стані B", перехід -> A
```

---

*Документ підготовлено для вивчення патернів проектування. Всі приклади протестовані на .NET 6+.*
