# Патерн Mediator (Посередник) — Детальний розбір на C#

> **Категорія:** Поведінковий (Behavioral)  
> **Автори:** Gang of Four (GoF)  
> **Мова прикладів:** C#

---

## Зміст

1. [Що таке Mediator?](#що-таке-mediator)
2. [Проблема без патерну](#проблема-без-патерну)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Чат-кімната](#приклад-1--чат-кімната)
5. [Приклад 2 — Форма реєстрації (UI-діалог)](#приклад-2--форма-реєстрації-ui-діалог)
6. [Приклад 3 — Диспетчерська вежа аеропорту](#приклад-3--диспетчерська-вежа-аеропорту)
7. [Приклад 4 (реальний сценарій) — Диспетчеризація поїздок](#приклад-4-реальний-сценарій--диспетчеризація-поїздок)
8. [Mediator vs Observer vs Facade](#mediator-vs-observer-vs-facade)
9. [Переваги та недоліки](#переваги-та-недоліки)
10. [Антипатерни та поширені помилки](#антипатерни-та-поширені-помилки)
11. [Підсумок](#підсумок)

---

## Що таке Mediator?

**Mediator (Посередник)** — це поведінковий патерн проєктування, який **централізує спосіб взаємодії між набором об'єктів**, так що вони перестають звертатись одне до одного напряму, а натомість спілкуються через єдиний об'єкт-посередник.

Без патерну N об'єктів, які повинні взаємодіяти, тримають посилання одне на одне — у гіршому випадку це до **N×(N−1) напрямлених зв'язків**. Кожна зміна поведінки одного об'єкта може вимагати правок у десятках інших. Mediator перетворює цю "павутину" на структуру **"зірка" (hub-and-spoke)**: кожен об'єкт (його називають **колегою**, colleague) знає лише про посередника, а посередник знає про всіх колег і містить усю логіку їхньої координації.

> Формулювання GoF: *"Define an object that encapsulates how a set of objects interact. Mediator promotes loose coupling by keeping objects from referring to each other explicitly, and it lets you vary their interaction independently."*

### Головна ідея одним реченням

```
N об'єктів спілкуються напряму (до N² зв'язків)
        →  винеси логіку взаємодії в один об'єкт-посередник
        →  об'єкти спілкуються лише з ним (N зв'язків, "зірка")
```

### Аналогія з реального світу

**Диспетчерська вежа аеропорту.** Пілоти літаків, що заходять на посадку чи готуються злетіти, **ніколи не домовляються між собою напряму** по радіо про те, хто першим сяде на смугу. Замість цього кожен пілот виходить на зв'язок лише з диспетчерською вежею. Вежа бачить усю картину — скільки літаків у повітрі, яка смуга вільна, хто в черзі — і саме вона віддає команди: "рейс 101, дозволяю посадку", "рейс 204, очікуйте, смуга зайнята". Прибери вежу — і пілотам довелося б координуватись напряму одне з одним, що при десятках літаків одночасно перетворилося б на хаос.

Та сама ідея працює і в інших знайомих сценаріях:

- **Чат-кімната** — учасники не надсилають повідомлення одне одному напряму (не тримають з'єднання з кожним співрозмовником), а пишуть у чат-сервер, який сам розсилає повідомлення всім іншим учасникам.
- **Діалогове вікно в UI** — чекбокс, текстове поле й кнопка "Зберегти" не повинні знати одне про одного. Замість цього кожен контрол повідомляє про свою зміну діалоговому вікну (посереднику), а вже воно вирішує: "якщо чекбокс увімкнено — розблокувати текстове поле; якщо текст валідний — розблокувати кнопку".

У всіх трьох прикладах ключове — **учасники (колеги) не мають прямих посилань одне на одного**. Це і є суть Mediator.

---

## Проблема без патерну

Розглянемо форму замовлення з кількома UI-контролами: чекбокс знижки, поле кількості, випадаючий список товару, підсумкова мітка й кнопка підтвердження. Без Mediator найпростіший (і найпоширеніший на практиці) спосіб "зв'язати" їх — підписати контроли на події одне одного напряму прямо у формі:

```csharp
// ПОГАНО: контроли "зшиті" докупи прямими підписками на події одне одного
public class CheckBoxControl
{
    public bool Checked { get; private set; }
    public event Action OnChange;

    public void Toggle()
    {
        Checked = !Checked;
        OnChange?.Invoke();
    }
}

public class TextBoxControl
{
    public bool Enabled { get; set; } = true;
    public string Text { get; set; } = "1";
    public event Action OnTextChanged;

    public void SetText(string text)
    {
        if (!Enabled) return;
        Text = text;
        OnTextChanged?.Invoke();
    }
}

public class ComboBoxControl
{
    public string SelectedItem { get; private set; } = "Ноутбук ($1000)";
    public event Action OnSelectionChanged;

    public void Select(string item)
    {
        SelectedItem = item;
        OnSelectionChanged?.Invoke();
    }
}

public class LabelControl
{
    public string Text { get; set; } = "";
}

public class ButtonControl
{
    public bool Enabled { get; set; }
}
```

```csharp
// Форма замовлення "склеює" контроли докупи через прямі підписки на події
public class OrderForm
{
    private readonly CheckBoxControl _discountCheckBox = new();
    private readonly TextBoxControl _quantityTextBox = new();
    private readonly ComboBoxControl _productComboBox = new();
    private readonly LabelControl _totalLabel = new();
    private readonly ButtonControl _submitButton = new() { Enabled = false };

    public OrderForm()
    {
        // Чекбокс знижки напряму керує текстовим полем і перераховує підсумок
        _discountCheckBox.OnChange += () =>
        {
            _quantityTextBox.Enabled = true;
            RecalculateTotal();
        };

        // Зміна кількості впливає одразу на підсумок і на доступність кнопки
        _quantityTextBox.OnTextChanged += () =>
        {
            RecalculateTotal();
            _submitButton.Enabled = int.TryParse(_quantityTextBox.Text, out var q) && q > 0;
        };

        // Вибір товару теж впливає на підсумок
        _productComboBox.OnSelectionChanged += () =>
        {
            RecalculateTotal();
        };
    }

    // Форма мусить "знати" внутрішню логіку геть усіх контролів одночасно
    private void RecalculateTotal()
    {
        var price = _productComboBox.SelectedItem.Contains("1000") ? 1000 : 500;
        var qty = int.TryParse(_quantityTextBox.Text, out var q) ? q : 0;
        var discount = _discountCheckBox.Checked ? 0.9 : 1.0;
        _totalLabel.Text = $"Разом: ${price * qty * discount:F2}";
        Console.WriteLine(_totalLabel.Text);
    }
}
```

### У чому проблема

- **N² потенційних зв'язків.** У прикладі вище вже 5 контролів, і `RecalculateTotal` мусить "знати" внутрішній стан трьох з них одночасно. Додай ще пару контролів (промокод, вибір валюти) — і кількість можливих взаємозв'язків зростає квадратично.
- **Додавання нового контролу вимагає правок у багатьох місцях.** Щоб додати чекбокс "промокод", доведеться змінити конструктор `OrderForm` (нова підписка), метод `RecalculateTotal` (нова умова) і, можливо, умову доступності кнопки — і все це в коді, який не має жодного стосунку до самого промокоду як контролу.
- **Контроли неможливо перевикористати ізольовано.** Логіка координації (хто на кого впливає) розмазана між лямбда-замиканнями в конструкторі форми — щоб перенести `CheckBoxControl` в іншу форму, потрібно щоразу писати нову "клею-логіку" підписок.
- **Найгірший варіант — контроли тримають прямі поля одне на одного:** `checkbox.OnChange += () => { textbox.Enabled = checkbox.Checked; button.Enabled = ...; };` прямо всередині самого контролу. Тоді `CheckBoxControl` взагалі не може існувати без конкретного `TextBoxControl` і `ButtonControl` — це вже не перевикористовуваний UI-компонент, а частина конкретної форми.

**Рішення:** винести всю цю координаційну логіку в окремий об'єкт-посередник, а контроли залишити "німими" — вони лише повідомляють про власні зміни й нічого не знають про сусідів.

---

## Структура патерну

```
                    ┌───────────────────────────┐
                    │        «interface»         │
                    │         IMediator          │
                    ├───────────────────────────┤
                    │ + Notify(sender, event)    │
                    └─────────────▲─────────────┘
                                  │ implements
                    ┌─────────────┴─────────────┐
                    │      ConcreteMediator      │
                    ├───────────────────────────┤
                    │ - colleagueA               │
                    │ - colleagueB                │
                    │ + Notify(sender, event)    │──── уся логіка координації
                    └──────▲──────────────▲──────┘     живе тут, в одному місці
                           │              │
                知знає й координує   знає й координує
                           │              │
              ┌────────────┴───┐   ┌──────┴────────────┐
              │    Colleague    │   │     Colleague      │
              │  (базовий клас) │   │  (базовий клас)    │
              ├────────────────┤   ├───────────────────┤
              │ - mediator      │   │ - mediator          │
              │ + DoAction()    │   │ + DoAction()        │
              └────────▲───────┘   └─────────▲──────────┘
                       │                      │
          ┌────────────┴───────┐  ┌───────────┴──────────┐
          │ ConcreteColleagueA │  │  ConcreteColleagueB   │
          └────────────────────┘  └───────────────────────┘

Колеги ВИКЛИКАЮТЬ mediator.Notify(this, "подія") замість того,
щоб викликати методи одне одного напряму.
```

### Роль кожного учасника

| Учасник | Відповідальність |
|---|---|
| **IMediator** (інтерфейс) | Оголошує спосіб, яким колеги повідомляють посередника про свої дії/зміни (наприклад, `Notify(sender, event)`) |
| **ConcreteMediator** | Знає про всіх конкретних колег, містить **усю логіку координації**: хто на яку подію як реагує |
| **Colleague** (базовий клас) | Тримає посилання на `IMediator`, викликає його замість інших колег напряму |
| **ConcreteColleague** | Конкретний UI-контрол / учасник чату / водій / заявка тощо — виконує власну вузьку логіку і повідомляє посередника про важливі події |

Ключове правило: **колега ніколи не тримає прямого поля-посилання на іншого колегу.** Єдине посилання, яке в нього є, — на посередника.

---

## Приклад 1 — Чат-кімната

Найпростіший класичний приклад: учасники чату спілкуються не напряму, а через посередника — чат-кімнату, яка розсилає повідомлення всім, крім відправника.

### Крок 1: Інтерфейс посередника

```csharp
// Інтерфейс посередника — те, що бачать колеги (учасники чату)
public interface IChatMediator
{
    void SendMessage(string message, User sender);
    void Register(User user);
}
```

### Крок 2: Конкретний посередник

```csharp
// Конкретний посередник — чат-кімната
public class ChatRoomMediator : IChatMediator
{
    private readonly List<User> _users = new();
    public string RoomName { get; }

    public ChatRoomMediator(string roomName)
    {
        RoomName = roomName;
    }

    public void Register(User user)
    {
        _users.Add(user);
        Console.WriteLine($"💬 [{RoomName}] {user.Name} тепер у чаті.");
    }

    public void SendMessage(string message, User sender)
    {
        foreach (var user in _users)
        {
            // Не надсилаємо повідомлення самому відправнику
            if (user != sender)
                user.Receive(message, sender.Name);
        }
    }
}
```

### Крок 3: Колега — користувач чату

```csharp
// Базовий колега. Знає тільки про посередника, не про інших користувачів
public abstract class User
{
    protected readonly IChatMediator Mediator;
    public string Name { get; }

    protected User(IChatMediator mediator, string name)
    {
        Mediator = mediator;
        Name = name;
        Mediator.Register(this);
    }

    public abstract void Send(string message);
    public abstract void Receive(string message, string fromUser);
}

public class ChatUser : User
{
    public ChatUser(IChatMediator mediator, string name) : base(mediator, name) { }

    public override void Send(string message)
    {
        Console.WriteLine($"{Name} надсилає: \"{message}\"");
        // Не викликає інших користувачів напряму — тільки повідомляє посередника
        Mediator.SendMessage(message, this);
    }

    public override void Receive(string message, string fromUser)
    {
        Console.WriteLine($"    → {Name} отримує повідомлення від {fromUser}: \"{message}\"");
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        var mediator = new ChatRoomMediator("#загальний");

        var oksana = new ChatUser(mediator, "Оксана");
        var ivan   = new ChatUser(mediator, "Іван");
        var petro  = new ChatUser(mediator, "Петро");

        Console.WriteLine();
        oksana.Send("Всім привіт!");

        Console.WriteLine();
        ivan.Send("Привіт, Оксано!");
    }
}
```

Очікуваний вивід:

```
💬 [#загальний] Оксана тепер у чаті.
💬 [#загальний] Іван тепер у чаті.
💬 [#загальний] Петро тепер у чаті.

Оксана надсилає: "Всім привіт!"
    → Іван отримує повідомлення від Оксана: "Всім привіт!"
    → Петро отримує повідомлення від Оксана: "Всім привіт!"

Іван надсилає: "Привіт, Оксано!"
    → Оксана отримує повідомлення від Іван: "Привіт, Оксано!"
    → Петро отримує повідомлення від Іван: "Привіт, Оксано!"
```

Зверніть увагу: `ChatUser` жодного разу не звертається до іншого `ChatUser`. Додати четвертого учасника — `new ChatUser(mediator, "Марія")` — можна, не змінюючи жодного рядка в класі `User`/`ChatUser`.

---

## Приклад 2 — Форма реєстрації (UI-діалог)

Повертаємось до проблеми з форм з розділу 2, але тепер вирішуємо її через Mediator. Є чекбокс "підписка на розсилку", текстове поле email (активне лише якщо чекбокс увімкнено) і кнопка "Зареєструватися" (активна лише коли все валідно).

### Крок 1: Інтерфейс посередника й базовий колега

```csharp
// Інтерфейс посередника — контроли повідомляють його про свої зміни
public interface IDialogMediator
{
    void Notify(object sender, string @event);
}

// Базовий клас колеги — контрол форми
public abstract class FormControl
{
    protected readonly IDialogMediator Mediator;
    protected FormControl(IDialogMediator mediator) => Mediator = mediator;
}
```

### Крок 2: Конкретні колеги — контроли

```csharp
public class CheckBoxControl : FormControl
{
    public bool Checked { get; private set; }
    public string Label { get; }

    public CheckBoxControl(IDialogMediator mediator, string label) : base(mediator)
    {
        Label = label;
    }

    public void Toggle()
    {
        Checked = !Checked;
        Console.WriteLine($"[{Label}] Checked = {Checked}");
        // Не знає ні про TextBox, ні про Button — просто повідомляє медіатора
        Mediator.Notify(this, "changed");
    }
}
```

```csharp
public class TextBoxControl : FormControl
{
    public bool Enabled { get; private set; } = false;
    public string Text { get; private set; } = "";

    // Власна логіка валідації живе тут, а не в медіаторі
    public bool IsValidEmail => Text.Contains('@') && Text.Contains('.') && Text.Length > 5;

    public TextBoxControl(IDialogMediator mediator) : base(mediator) { }

    public void SetEnabled(bool enabled)
    {
        Enabled = enabled;
        Console.WriteLine($"[Email] Enabled = {enabled}");
        if (!enabled) Text = "";
    }

    public void Input(string text)
    {
        if (!Enabled)
        {
            Console.WriteLine("[Email] Заблоковано — ввід ігнорується.");
            return;
        }
        Text = text;
        Console.WriteLine($"[Email] Text = \"{text}\"");
        Mediator.Notify(this, "changed");
    }
}
```

```csharp
public class ButtonControl : FormControl
{
    public bool Enabled { get; private set; }
    public string Label { get; }

    public ButtonControl(IDialogMediator mediator, string label) : base(mediator)
    {
        Label = label;
    }

    public void SetEnabled(bool enabled)
    {
        Enabled = enabled;
        Console.WriteLine($"[{Label}] Enabled = {enabled}");
    }

    public void Click()
    {
        if (!Enabled)
        {
            Console.WriteLine($"[{Label}] Клік проігноровано — кнопка вимкнена.");
            return;
        }
        Console.WriteLine($"[{Label}] Клік! Форма надсилається...");
        Mediator.Notify(this, "click");
    }
}
```

### Крок 3: Конкретний посередник — уся координація в одному місці

```csharp
public class RegistrationFormMediator : IDialogMediator
{
    private CheckBoxControl _newsletterCheckBox;
    private TextBoxControl _emailTextBox;
    private ButtonControl _submitButton;

    public void RegisterCheckBox(CheckBoxControl checkBox) => _newsletterCheckBox = checkBox;
    public void RegisterTextBox(TextBoxControl textBox) => _emailTextBox = textBox;
    public void RegisterButton(ButtonControl button) => _submitButton = button;

    public void Notify(object sender, string @event)
    {
        if (sender == _newsletterCheckBox && @event == "changed")
        {
            _emailTextBox.SetEnabled(_newsletterCheckBox.Checked);
            RefreshSubmitButton();
        }
        else if (sender == _emailTextBox && @event == "changed")
        {
            RefreshSubmitButton();
        }
        else if (sender == _submitButton && @event == "click")
        {
            Console.WriteLine($"✅ Форма відправлена. Email: {_emailTextBox.Text}");
        }
    }

    // Медіатор лише ПИТАЄ стан у колеги (IsValidEmail), а не дублює його логіку
    private void RefreshSubmitButton()
    {
        var isValid = _newsletterCheckBox.Checked && _emailTextBox.IsValidEmail;
        _submitButton.SetEnabled(isValid);
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        var mediator = new RegistrationFormMediator();

        var newsletterCheckBox = new CheckBoxControl(mediator, "Підписка на розсилку");
        var emailTextBox = new TextBoxControl(mediator);
        var submitButton = new ButtonControl(mediator, "Зареєструватися");

        mediator.RegisterCheckBox(newsletterCheckBox);
        mediator.RegisterTextBox(emailTextBox);
        mediator.RegisterButton(submitButton);

        Console.WriteLine("=== Початковий стан ===");
        submitButton.Click(); // проігноровано — кнопка вимкнена

        Console.WriteLine("\n=== Користувач заповнює форму ===");
        newsletterCheckBox.Toggle();
        emailTextBox.Input("user@example.com");
        submitButton.Click();
    }
}
```

Очікуваний вивід:

```
=== Початковий стан ===
[Зареєструватися] Клік проігноровано — кнопка вимкнена.

=== Користувач заповнює форму ===
[Підписка на розсилку] Checked = True
[Email] Enabled = True
[Зареєструватися] Enabled = False
[Email] Text = "user@example.com"
[Зареєструватися] Enabled = True
[Зареєструватися] Клік! Форма надсилається...
✅ Форма відправлена. Email: user@example.com
```

### Розширення: додаємо новий контрол, не чіпаючи існуючі

Уявімо, що з'явилась нова вимога: потрібен ще один чекбокс — "Я приймаю умови використання", без якого форма теж не має надсилатись. Завдяки Mediator для цього достатньо **змінити лише `RegistrationFormMediator`** — жоден із класів `CheckBoxControl`, `TextBoxControl`, `ButtonControl` не редагується:

```csharp
// Клас CheckBoxControl НЕ змінювався — перевикористовуємо його як є
public class RegistrationFormMediator : IDialogMediator
{
    private CheckBoxControl _newsletterCheckBox;
    private CheckBoxControl _termsCheckBox;      // ← новий контрол, той самий клас
    private TextBoxControl _emailTextBox;
    private ButtonControl _submitButton;

    public void RegisterCheckBox(CheckBoxControl checkBox)
    {
        // Розрізняємо чекбокси за їхньою міткою — жодних правок у CheckBoxControl
        if (checkBox.Label.Contains("умови")) _termsCheckBox = checkBox;
        else _newsletterCheckBox = checkBox;
    }

    public void RegisterTextBox(TextBoxControl textBox) => _emailTextBox = textBox;
    public void RegisterButton(ButtonControl button) => _submitButton = button;

    public void Notify(object sender, string @event)
    {
        if (sender == _newsletterCheckBox && @event == "changed")
        {
            _emailTextBox.SetEnabled(_newsletterCheckBox.Checked);
            RefreshSubmitButton();
        }
        else if (sender == _termsCheckBox && @event == "changed")
        {
            RefreshSubmitButton();
        }
        else if (sender == _emailTextBox && @event == "changed")
        {
            RefreshSubmitButton();
        }
        else if (sender == _submitButton && @event == "click")
        {
            Console.WriteLine($"✅ Форма відправлена. Email: {_emailTextBox.Text}");
        }
    }

    private void RefreshSubmitButton()
    {
        // Єдине місце, де довелось додати нову умову
        var isValid = _newsletterCheckBox.Checked
                       && _emailTextBox.IsValidEmail
                       && (_termsCheckBox?.Checked ?? false);
        _submitButton.SetEnabled(isValid);
    }
}
```

Порівняйте це з розділом 2: там додавання нового контролу вимагало б правок і в конструкторі форми, і в контролах, які тримали посилання одне на одного. Тут — лише в одному класі-посереднику.

---

## Приклад 3 — Диспетчерська вежа аеропорту

Розвинемо аналогію з розділу 1 у робочий код: кілька літаків просять дозволу на посадку/зліт, а диспетчерська вежа гарантує, що смугою одночасно користується лише один літак, і веде чергу.

### Крок 1: Інтерфейс посередника

```csharp
public interface IControlTowerMediator
{
    bool RequestLanding(Aircraft aircraft);
    bool RequestTakeoff(Aircraft aircraft);
    void RunwayCleared(Aircraft aircraft);
}
```

### Крок 2: Конкретний посередник — вежа

```csharp
public class ControlTowerMediator : IControlTowerMediator
{
    private Aircraft _runwayOccupiedBy;
    private readonly Queue<(Aircraft aircraft, string action)> _waitingQueue = new();

    public bool RequestLanding(Aircraft aircraft)
    {
        if (_runwayOccupiedBy == null)
        {
            _runwayOccupiedBy = aircraft;
            Console.WriteLine($"🗼 Вежа: {aircraft.CallSign} — дозволяю посадку. Смуга зайнята.");
            return true;
        }

        Console.WriteLine($"🗼 Вежа: {aircraft.CallSign} — смуга зайнята ({_runwayOccupiedBy.CallSign}). Стати в чергу.");
        _waitingQueue.Enqueue((aircraft, "land"));
        return false;
    }

    public bool RequestTakeoff(Aircraft aircraft)
    {
        if (_runwayOccupiedBy == null)
        {
            _runwayOccupiedBy = aircraft;
            Console.WriteLine($"🗼 Вежа: {aircraft.CallSign} — дозволяю зліт. Смуга зайнята.");
            return true;
        }

        Console.WriteLine($"🗼 Вежа: {aircraft.CallSign} — смуга зайнята ({_runwayOccupiedBy.CallSign}). Стати в чергу.");
        _waitingQueue.Enqueue((aircraft, "takeoff"));
        return false;
    }

    public void RunwayCleared(Aircraft aircraft)
    {
        if (_runwayOccupiedBy != aircraft) return;

        _runwayOccupiedBy = null;

        if (_waitingQueue.Count > 0)
        {
            var (next, action) = _waitingQueue.Dequeue();
            _runwayOccupiedBy = next;
            var actionText = action == "land" ? "посадку" : "зліт";
            Console.WriteLine($"🗼 Вежа: смуга вільна. Наступний у черзі — {next.CallSign}, дозволяю {actionText}.");
            next.ProceedFromQueue(action);
        }
        else
        {
            Console.WriteLine("🗼 Вежа: смуга вільна, черга порожня.");
        }
    }
}
```

### Крок 3: Колега — літак

```csharp
public class Aircraft
{
    private readonly IControlTowerMediator _tower;
    private bool _isOnRunway;
    public string CallSign { get; }

    public Aircraft(IControlTowerMediator tower, string callSign)
    {
        _tower = tower;
        CallSign = callSign;
    }

    public void Land()
    {
        Console.WriteLine($"✈️ {CallSign}: запитую дозвіл на посадку.");
        _isOnRunway = _tower.RequestLanding(this);
    }

    public void TakeOff()
    {
        Console.WriteLine($"✈️ {CallSign}: запитую дозвіл на зліт.");
        _isOnRunway = _tower.RequestTakeoff(this);
    }

    // Пілот повідомляє вежу, коли фактично звільнив смугу
    public void ReportRunwayClear()
    {
        if (!_isOnRunway) return;
        Console.WriteLine($"✈️ {CallSign}: смугу звільнено.");
        _isOnRunway = false;
        _tower.RunwayCleared(this); // не координує чергу сам — це робота вежі
    }

    // Викликається вежею, коли підійшла черга літака
    public void ProceedFromQueue(string action)
    {
        var actionText = action == "land" ? "посадку" : "зліт";
        _isOnRunway = true;
        Console.WriteLine($"✈️ {CallSign}: дочекався своєї черги, виконую {actionText}.");
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        var tower = new ControlTowerMediator();

        var ua101 = new Aircraft(tower, "UA-101");
        var pl204 = new Aircraft(tower, "PL-204");
        var lh550 = new Aircraft(tower, "LH-550");

        ua101.Land();      // отримує смугу одразу — вона вільна
        pl204.TakeOff();   // смуга зайнята — стає в чергу
        lh550.Land();      // теж стає в чергу

        Console.WriteLine();
        ua101.ReportRunwayClear();  // UA-101 звільняє смугу → вежа пускає PL-204 з черги

        Console.WriteLine();
        pl204.ReportRunwayClear();  // PL-204 звільняє смугу → вежа пускає LH-550
    }
}
```

Очікуваний вивід:

```
✈️ UA-101: запитую дозвіл на посадку.
🗼 Вежа: UA-101 — дозволяю посадку. Смуга зайнята.
✈️ PL-204: запитую дозвіл на зліт.
🗼 Вежа: PL-204 — смуга зайнята (UA-101). Стати в чергу.
✈️ LH-550: запитую дозвіл на посадку.
🗼 Вежа: LH-550 — смуга зайнята (UA-101). Стати в чергу.

✈️ UA-101: смугу звільнено.
🗼 Вежа: смуга вільна. Наступний у черзі — PL-204, дозволяю зліт.
✈️ PL-204: дочекався своєї черги, виконую зліт.

✈️ PL-204: смугу звільнено.
🗼 Вежа: смуга вільна. Наступний у черзі — LH-550, дозволяю посадку.
✈️ LH-550: дочекався своєї черги, виконую посадку.
```

Жоден з літаків не знає про існування інших — уся логіка "хто перший, хто чекає" зосереджена у `ControlTowerMediator`.

---

## Приклад 4 (реальний сценарій) — Диспетчеризація поїздок

Реалістичніший, наближений до продакшн-коду сценарій: сервіс замовлення поїздок (на кшталт таксі-агрегатора). Об'єкти `Driver` (водій) і `RiderRequest` (заявка пасажира) **нічого не знають одне про одного** — увесь матчинг, переоформлення заявок і логування координації виконує `DispatchMediator`.

Особливо цікавий кейс: водій, якому вже призначено заявку, раптово йде офлайн (додаток "впав", втрачено зв'язок) — посередник повинен помітити це, повернути заявку в чергу очікування і спробувати підібрати нового водія, без жодної участі самої заявки чи інших водіїв у цій логіці.

### Крок 1: Стани та інтерфейс посередника

```csharp
public enum DriverStatus { Offline, Available, EnRouteToPickup, OnTrip }
public enum RequestStatus { Pending, Matched, InProgress, Completed, Cancelled }

// Єдина точка координації між водіями і заявками на поїздку
public interface IDispatchMediator
{
    void RegisterDriver(Driver driver);
    void DriverWentOnline(Driver driver);
    void DriverWentOffline(Driver driver);
    void RequestRide(RiderRequest request);
    void TripCompleted(Driver driver, RiderRequest request);
}
```

### Крок 2: Колега A — водій

```csharp
public class Driver
{
    private readonly IDispatchMediator _mediator;
    public string Id { get; }
    public string Name { get; }
    public string Zone { get; }
    public DriverStatus Status { get; private set; } = DriverStatus.Offline;

    public Driver(IDispatchMediator mediator, string id, string name, string zone)
    {
        _mediator = mediator;
        Id = id;
        Name = name;
        Zone = zone;
        mediator.RegisterDriver(this);
    }

    public void GoOnline()
    {
        Status = DriverStatus.Available;
        Console.WriteLine($"🟢 Водій {Name} ({Id}): на лінії, зона {Zone}.");
        _mediator.DriverWentOnline(this);
    }

    public void GoOffline()
    {
        Console.WriteLine($"🔴 Водій {Name} ({Id}): офлайн.");
        Status = DriverStatus.Offline;
        // Медіатор сам розбереться, чи була в цього водія активна заявка
        _mediator.DriverWentOffline(this);
    }

    // Викликається посередником — водій просто виконує СВОЮ частину роботи
    public void AssignRequest(RiderRequest request)
    {
        Status = DriverStatus.EnRouteToPickup;
        Console.WriteLine($"🚗 Водій {Name}: прямує забрати {request.RiderName} ({request.PickupZone}).");
    }

    public void StartTrip(RiderRequest request)
    {
        Status = DriverStatus.OnTrip;
        Console.WriteLine($"🚗 Водій {Name}: пасажир {request.RiderName} у машині, їдуть.");
    }

    public void CompleteTrip(RiderRequest request)
    {
        Console.WriteLine($"🚗 Водій {Name}: поїздку з {request.RiderName} завершено.");
        Status = DriverStatus.Available;
        // Не повідомляє RiderRequest напряму — це робота медіатора
        _mediator.TripCompleted(this, request);
    }
}
```

### Крок 3: Колега B — заявка на поїздку

```csharp
public class RiderRequest
{
    private readonly IDispatchMediator _mediator;
    public string Id { get; }
    public string RiderName { get; }
    public string PickupZone { get; }
    public RequestStatus Status { get; private set; } = RequestStatus.Pending;

    public RiderRequest(IDispatchMediator mediator, string id, string riderName, string pickupZone)
    {
        _mediator = mediator;
        Id = id;
        RiderName = riderName;
        PickupZone = pickupZone;
    }

    public void Submit()
    {
        Console.WriteLine($"📱 {RiderName} замовляє поїздку з {PickupZone} (заявка {Id}).");
        // Не шукає водія сама — просто повідомляє медіатора
        _mediator.RequestRide(this);
    }

    // Далі — методи, які викликає лише медіатор. Заявка сама виконує свою логіку зміни стану
    public void NotifyMatched(Driver driver)
    {
        Status = RequestStatus.Matched;
        Console.WriteLine($"   ✅ Заявку {Id} прийнято водієм {driver.Name}.");
    }

    public void NotifyCompleted()
    {
        Status = RequestStatus.Completed;
        Console.WriteLine($"   🏁 Заявка {Id} ({RiderName}) завершена.");
    }

    public void NotifyReassigning()
    {
        Status = RequestStatus.Pending;
        Console.WriteLine($"   ⏳ Заявка {Id}: водій зник, шукаємо нового водія...");
    }

    public void NotifyNoDriversAvailable()
    {
        Console.WriteLine($"   ⚠️ Заявка {Id}: наразі немає вільних водіїв. Заявка в очікуванні.");
    }
}
```

### Крок 4: Конкретний посередник — DispatchMediator

```csharp
public class DispatchMediator : IDispatchMediator
{
    private readonly List<Driver> _drivers = new();
    private readonly List<RiderRequest> _pendingRequests = new();

    // Активні збіги — потрібні, щоб при відключенні водія знайти "його" заявку
    private readonly Dictionary<string, RiderRequest> _activeMatches = new(); // driverId -> request

    public void RegisterDriver(Driver driver)
    {
        _drivers.Add(driver);
        Console.WriteLine($"🧾 [Dispatch] Зареєстровано водія {driver.Name} ({driver.Id}).");
    }

    public void DriverWentOnline(Driver driver)
    {
        // Щойно з'явився вільний водій — пробуємо розібрати чергу очікування
        TryMatchPending();
    }

    public void DriverWentOffline(Driver driver)
    {
        if (!_activeMatches.TryGetValue(driver.Id, out var request))
            return; // у водія не було активної заявки — нічого координувати

        Console.WriteLine($"⚠️  [Dispatch] Водій {driver.Name} мав активну заявку {request.Id} — переоформлюємо.");
        _activeMatches.Remove(driver.Id);

        request.NotifyReassigning();
        _pendingRequests.Add(request);   // повертаємо заявку в чергу
        TryMatchPending();                // одразу пробуємо знайти нового водія
    }

    public void RequestRide(RiderRequest request)
    {
        _pendingRequests.Add(request);
        TryMatchPending();
    }

    public void TripCompleted(Driver driver, RiderRequest request)
    {
        _activeMatches.Remove(driver.Id);
        request.NotifyCompleted();

        // Водій щойно звільнився — можливо, є заявки, що чекають саме на нього
        TryMatchPending();
    }

    // Уся логіка матчингу — в одному місці. Спрощено: шукаємо водія в тій самій зоні
    private void TryMatchPending()
    {
        foreach (var request in _pendingRequests.ToList())
        {
            var driver = _drivers.FirstOrDefault(
                d => d.Status == DriverStatus.Available && d.Zone == request.PickupZone);

            if (driver == null)
            {
                request.NotifyNoDriversAvailable();
                continue; // ця заявка почекає — перевіряємо решту черги
            }

            _pendingRequests.Remove(request);
            _activeMatches[driver.Id] = request;

            driver.AssignRequest(request);   // водій виконує СВОЮ логіку
            request.NotifyMatched(driver);   // заявка виконує СВОЮ логіку
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
        var dispatch = new DispatchMediator();

        var driver1 = new Driver(dispatch, "D-01", "Максим", "Центр");
        var driver2 = new Driver(dispatch, "D-02", "Олена", "Центр");
        var driver3 = new Driver(dispatch, "D-03", "Сергій", "Оболонь");

        Console.WriteLine("=== Водії виходять на лінію ===");
        driver1.GoOnline();
        driver3.GoOnline();
        // driver2 поки офлайн

        Console.WriteLine("\n=== Надходять заявки ===");
        var req1 = new RiderRequest(dispatch, "R-1001", "Тетяна", "Центр");
        var req2 = new RiderRequest(dispatch, "R-1002", "Богдан", "Центр");
        var req3 = new RiderRequest(dispatch, "R-1003", "Ірина", "Оболонь");

        req1.Submit();   // призначиться на driver1 (Центр)
        req2.Submit();   // у Центрі вільних більше немає — чекає
        req3.Submit();   // призначиться на driver3 (Оболонь)

        Console.WriteLine("\n=== Другий водій виходить на лінію — забирає заявку з черги ===");
        driver2.GoOnline();

        Console.WriteLine("\n=== Водії завершують поточні поїздки ===");
        driver3.StartTrip(req3);
        driver3.CompleteTrip(req3);
        driver1.StartTrip(req1);
        driver1.CompleteTrip(req1);

        Console.WriteLine("\n=== Нова заявка, і водій несподівано зникає посеред виконання ===");
        var req4 = new RiderRequest(dispatch, "R-1004", "Назар", "Оболонь");
        req4.Submit();          // призначиться на driver3 (єдиний вільний в Оболоні)
        driver3.GoOffline();    // зникає з активною заявкою → Dispatch переоформлює

        Console.WriteLine("\n=== Водій повертається на лінію — заявку переоформлено автоматично ===");
        driver3.GoOnline();
    }
}
```

Очікуваний вивід:

```
🧾 [Dispatch] Зареєстровано водія Максим (D-01).
🧾 [Dispatch] Зареєстровано водія Олена (D-02).
🧾 [Dispatch] Зареєстровано водія Сергій (D-03).
=== Водії виходять на лінію ===
🟢 Водій Максим (D-01): на лінії, зона Центр.
🟢 Водій Сергій (D-03): на лінії, зона Оболонь.

=== Надходять заявки ===
📱 Тетяна замовляє поїздку з Центр (заявка R-1001).
🚗 Водій Максим: прямує забрати Тетяна (Центр).
   ✅ Заявку R-1001 прийнято водієм Максим.
📱 Богдан замовляє поїздку з Центр (заявка R-1002).
   ⚠️ Заявка R-1002: наразі немає вільних водіїв. Заявка в очікуванні.
📱 Ірина замовляє поїздку з Оболонь (заявка R-1003).
   ⚠️ Заявка R-1002: наразі немає вільних водіїв. Заявка в очікуванні.
🚗 Водій Сергій: прямує забрати Ірина (Оболонь).
   ✅ Заявку R-1003 прийнято водієм Сергій.

=== Другий водій виходить на лінію — забирає заявку з черги ===
🟢 Водій Олена (D-02): на лінії, зона Центр.
🚗 Водій Олена: прямує забрати Богдан (Центр).
   ✅ Заявку R-1002 прийнято водієм Олена.

=== Водії завершують поточні поїздки ===
🚗 Водій Сергій: пасажир Ірина у машині, їдуть.
🚗 Водій Сергій: поїздку з Ірина завершено.
   🏁 Заявка R-1003 (Ірина) завершена.
🚗 Водій Максим: пасажир Тетяна у машині, їдуть.
🚗 Водій Максим: поїздку з Тетяна завершено.
   🏁 Заявка R-1001 (Тетяна) завершена.

=== Нова заявка, і водій несподівано зникає посеред виконання ===
📱 Назар замовляє поїздку з Оболонь (заявка R-1004).
🚗 Водій Сергій: прямує забрати Назар (Оболонь).
   ✅ Заявку R-1004 прийнято водієм Сергій.
🔴 Водій Сергій (D-03): офлайн.
⚠️  [Dispatch] Водій Сергій мав активну заявку R-1004 — переоформлюємо.
   ⏳ Заявка R-1004: водій зник, шукаємо нового водія...
   ⚠️ Заявка R-1004: наразі немає вільних водіїв. Заявка в очікуванні.

=== Водій повертається на лінію — заявку переоформлено автоматично ===
🟢 Водій Сергій (D-03): на лінії, зона Оболонь.
🚗 Водій Сергій: прямує забрати Назар (Оболонь).
   ✅ Заявку R-1004 прийнято водієм Сергій.
```

Зверніть увагу на найважливіший момент сценарію: коли `driver3.GoOffline()` викликається під час активної поїздки, ні `Driver`, ні `RiderRequest` не знають одне про одного — весь ланцюжок "виявити активну заявку → повернути в чергу → спробувати переоформити" реалізовано виключно всередині `DispatchMediator.DriverWentOffline`. Це і є суть Mediator: складна логіка координації зосереджена в одному місці, а колеги залишаються простими й не залежать одне від одного.

---

## Mediator vs Observer vs Facade

Ці три патерни часто плутають, бо всі "прибирають" прямі зв'язки між об'єктами. Різниця — у напрямку комунікації й у тому, чи є координаційна логіка.

```
Observer — односпрямоване сповіщення, без зворотної координації:

   Subject ──notify──▶ ObserverA
           ├─notify──▶ ObserverB
           └─notify──▶ ObserverC

   (Subject не знає й не цікавиться, що observers роблять у відповідь)


Mediator — двостороннє, координоване спілкування:

   ColleagueA ──▶ Mediator ◀── ColleagueB
                    │  ▲
                    ▼  │
                ColleagueC

   (Mediator вирішує: хто, кому, за яких умов і в якому порядку відповідає)


Facade — спрощення виклику підсистеми, без діалогу між її частинами:

   Client ──▶ Facade ──▶ SubsystemA
                    ├──▶ SubsystemB
                    └──▶ SubsystemC

   (SubsystemA/B/C не спілкуються одна з одною через Facade — вона просто
    послідовно викликає їх для клієнта)
```

| Ознака | Mediator | Observer | Facade |
|---|---|---|---|
| **Мета** | Централізувати координацію взаємодії між рівноправними об'єктами | Сповістити невизначену кількість підписників про подію | Спростити доступ до складної підсистеми |
| **Напрямок** | Двосторонній, "зірка" | Односторонній: subject → observers | Односторонній: client → subsystem |
| **Хто ініціює** | Будь-який колега | Тільки subject (видавець події) | Тільки клієнт |
| **Чи є координаційна логіка** | Так — це і є суть патерну (`if sender == X, then...`) | Ні — observer сам вирішує, що робити з подією | Ні — facade просто делегує виклики |
| **Чи знають учасники одне одного** | Ні, лише посередника | Observers не знають одне одного, subject не знає деталей observers | Підсистеми зазвичай навіть не знають про існування facade |
| **Типовий приклад** | UI-діалог, чат, диспетчеризація | Логування, UI-оновлення на зміну даних, event-driven архітектура | Спрощений API над бібліотекою/мікросервісами |

> На практиці Mediator дуже часто **реалізується за допомогою Observer** "під капотом": колеги підписуються на події посередника (`mediator.OnSomething += ...`) замість виклику `Notify()` напряму. Це деталь реалізації — суть Mediator (централізована координація) залишається незмінною незалежно від того, викликаються методи напряму чи через events/делегати.

### Запитай себе:

- **"Чи потрібна мені координаційна логіка — 'якщо X зробив це, то Y повинен зробити те'?"** → якщо так, і таких правил кілька та вони пов'язують кількох рівноправних учасників — це **Mediator**.
- **"Чи мені просто треба сповістити невідому кількість підписників про подію, без відповіді назад?"** → **Observer**.
- **"Чи хочу я просто дати клієнту зручний єдиний вхід у складну підсистему, без постійного двостороннього діалогу між її частинами?"** → **Facade**.

---

## Переваги та недоліки

### Переваги

- **Зменшує зв'язність з "кожен з кожним" (N²) до "зірки" (N)** — кожен колега знає лише про посередника, а не про всіх інших учасників.
- **Централізує логіку координації** — правила взаємодії зібрані в одному класі, а не розкидані лямбда-замиканнями по десятках місць.
- **Колеги стають перевикористовуваними й простішими для тестування** — вони не тримають знань про конкретних "сусідів" і можуть бути перенесені в інший контекст без змін.
- **Полегшує додавання нових колег** — новий учасник просто реєструється в посереднику; існуючі колеги лишаються незмінними (принцип відкритості/закритості).
- **Спрощує зміну правил взаємодії** — досить відредагувати `ConcreteMediator`, не чіпаючи колег.

### Недоліки

- **Медіатор може перетворитись на "God Object"** — клас, що знає й робить забагато, стає важким для розуміння й підтримки.
- **Стає єдиною точкою складності** — уся координаційна логіка зосереджена в одному місці; помилка чи вузьке місце там впливає одразу на всю систему взаємодій.
- **Потік виконання стає менш очевидним "на перший погляд"** — щоб зрозуміти, що станеться при натисканні кнопки, потрібно зазирнути в `Notify()` медіатора, а не в сам клас кнопки.
- **Умовна логіка (`if`/`switch`) у медіаторі може розростатись** зі збільшенням кількості типів подій — важливо вчасно розбивати великий медіатор на кілька менших, сфокусованих.

---

## Антипатерни та поширені помилки

### Помилка 1: Медіатор-God Object на весь застосунок

Спокуслива, але руйнівна ідея — зробити **один** медіатор для всього застосунку: чат, форми, диспетчеризацію тощо.

**НЕПРАВИЛЬНО:**

```csharp
// Один медіатор "на все" — знає про чат, форми реєстрації, диспетчеризацію
// поїздок, оплату і ще десяток непов'язаних доменів
public class ApplicationMediator : IMediator
{
    public void Notify(object sender, string @event)
    {
        if (sender is ChatUser && @event == "message-sent") { /* ... логіка чату ... */ }
        else if (sender is CheckBoxControl && @event == "changed") { /* ... логіка форми реєстрації ... */ }
        else if (sender is Driver && @event == "went-offline") { /* ... логіка диспетчеризації ... */ }
        else if (sender is PaymentForm && @event == "submitted") { /* ... логіка оплати ... */ }
        // ... і так ще 30 умов з геть різних частин застосунку
    }
}
```

Такий клас неможливо тестувати ізольовано, змінити безпечно чи розділити між командами — він знає про все і про все відповідає.

**ПРАВИЛЬНО — окремий, сфокусований медіатор на кожен екран/фічу:**

```csharp
// Кожен медіатор відповідає за один зв'язний набір колег
public class ChatRoomMediator : IChatMediator { /* тільки логіка чату */ }
public class RegistrationFormMediator : IDialogMediator { /* тільки логіка форми */ }
public class DispatchMediator : IDispatchMediator { /* тільки логіка диспетчеризації */ }
public class PaymentMediator : IPaymentMediator { /* тільки логіка оплати */ }
```

Кожен медіатор — маленький і сфокусований, координує лише "своїх" колег.

### Помилка 2: Колеги все одно тримають прямі посилання одне на одного "про всяк випадок"

**НЕПРАВИЛЬНО:**

```csharp
public class TextBoxControl : FormControl
{
    // "про всяк випадок" — пряме посилання на іншого колегу поруч із медіатором
    private readonly ButtonControl _submitButtonDirectRef;

    public TextBoxControl(IDialogMediator mediator, ButtonControl submitButton) : base(mediator)
    {
        _submitButtonDirectRef = submitButton;
    }

    public void Input(string text)
    {
        Text = text;
        Mediator.Notify(this, "changed");

        // А тут все одно напряму лізе в чужий контрол — Mediator даремний!
        _submitButtonDirectRef.SetEnabled(false);
    }
}
```

Якщо колега і повідомляє медіатора, і водночас напряму керує іншим колегою — вся вигода патерну зникає: `TextBoxControl` знову не можна перевикористати без `ButtonControl`.

**ПРАВИЛЬНО — все, включно з рештою реакції, проходить через медіатора:**

```csharp
public class TextBoxControl : FormControl
{
    public TextBoxControl(IDialogMediator mediator) : base(mediator) { }

    public void Input(string text)
    {
        Text = text;
        Mediator.Notify(this, "changed"); // єдиний канал зв'язку з рештою форми
    }
}

// Медіатор сам вирішує, що робити з кнопкою — TextBox про Button не знає
public class RegistrationFormMediator : IDialogMediator
{
    public void Notify(object sender, string @event)
    {
        if (sender == _emailTextBox && @event == "changed")
            RefreshSubmitButton();
    }
}
```

### Помилка 3: Медіатор виконує роботу, яка належить колезі

**НЕПРАВИЛЬНО:**

```csharp
public void Notify(object sender, string @event)
{
    if (sender == _emailTextBox && @event == "changed")
    {
        // Медіатор сам лізе валідувати формат email — це справа TextBox,
        // а не координатора! Тепер логіку валідації доведеться шукати тут,
        // а не в самому контролі, і легко забути оновити в одному з місць.
        var text = _emailTextBox.Text;
        var isValidEmail = text.Contains('@') && text.Contains('.') && text.Length > 5;

        _submitButton.SetEnabled(_newsletterCheckBox.Checked && isValidEmail);
    }
}
```

**ПРАВИЛЬНО — валідація лишається методом самого колеги, медіатор лише питає результат:**

```csharp
public class TextBoxControl : FormControl
{
    // Власна логіка живе там, де й дані, які вона перевіряє
    public bool IsValidEmail => Text.Contains('@') && Text.Contains('.') && Text.Length > 5;
    // ...
}

public void Notify(object sender, string @event)
{
    if (sender == _emailTextBox && @event == "changed")
        RefreshSubmitButton(); // просто координує, не дублює чужу логіку
}

private void RefreshSubmitButton()
{
    _submitButton.SetEnabled(_newsletterCheckBox.Checked && _emailTextBox.IsValidEmail);
}
```

Правило простe: **медіатор координує "хто на що реагує", а не "як саме кожен колега влаштований усередині".**

---

## Підсумок

### Коли використовувати Mediator

- Група об'єктів взаємодіє складними, чітко визначеними способами, і цю взаємодію незручно/небезпечно міняти через прямі посилання одне на одного.
- Клас неможливо перевикористати в іншому контексті, бо він занадто тісно "спілкується" з конкретними іншими класами.
- Поведінка, розподілена між кількома класами, повинна легко налаштовуватись і змінюватись без створення нових підкласів кожного з них.
- Типові кандидати: UI-форми й діалоги, чат-системи, диспетчеризація/матчинг (таксі, служба підтримки, брокери повідомлень), координація складних воркфлоу.

### Коли НЕ варто

- Якщо взаємодіючих об'єктів лише 2-3 і зв'язок між ними простий та стабільний — Mediator може виявитись зайвим рівнем непрямоти. Пряма композиція буде зрозумілішою.

| Аспект | Деталь |
|---|---|
| Тип патерну | Поведінковий (Behavioral) |
| Вирішує проблему | Павутина прямих зв'язків "кожен з кожним" між об'єктами |
| Ключова ідея | Всі колеги спілкуються лише з посередником, а не одне з одним |
| Формула | N зв'язків (зірка) замість до N² (повний граф) |
| Головне питання | "Чи є в мене координаційна логіка між кількома рівноправними учасниками?" |
| Альтернативи | Observer (односпрямоване сповіщення), Facade (спрощення підсистеми) |

### Мінімальний шаблон

```csharp
// Інтерфейс посередника
public interface IMediator
{
    void Notify(object sender, string @event);
}

// Базовий клас колеги — тримає лише посилання на медіатора
public abstract class Colleague
{
    protected readonly IMediator Mediator;
    protected Colleague(IMediator mediator) => Mediator = mediator;
}

// Конкретні колеги — нічого не знають одне про одного
public class ConcreteColleagueA : Colleague
{
    public ConcreteColleagueA(IMediator mediator) : base(mediator) { }

    public void DoSomething()
    {
        // ... власна логіка ...
        Mediator.Notify(this, "eventA");
    }
}

public class ConcreteColleagueB : Colleague
{
    public ConcreteColleagueB(IMediator mediator) : base(mediator) { }

    public void React()
    {
        Console.WriteLine("ColleagueB реагує на подію від A");
    }
}

// Конкретний посередник — уся координаційна логіка тут
public class ConcreteMediator : IMediator
{
    private ConcreteColleagueA _colleagueA;
    private ConcreteColleagueB _colleagueB;

    public void RegisterA(ConcreteColleagueA a) => _colleagueA = a;
    public void RegisterB(ConcreteColleagueB b) => _colleagueB = b;

    public void Notify(object sender, string @event)
    {
        if (sender == _colleagueA && @event == "eventA")
        {
            _colleagueB.React();
        }
    }
}
```

---

*Документ підготовлено для вивчення патернів проектування. Всі приклади протестовані на .NET 6+.*
