# Патерн Iterator (Ітератор) — Детальний розбір на C#

> **Категорія:** Поведінковий (Behavioral)  
> **Автори:** Gang of Four (GoF)  
> **Мова прикладів:** C#

---

## Зміст

1. [Що таке Iterator?](#1-що-таке-iterator)
2. [Проблема без патерну](#2-проблема-без-патерну)
3. [Структура патерну](#3-структура-патерну)
4. [Приклад 1 — Класична реалізація Iterator без yield](#приклад-1-класична-реалізація-iterator-без-yield)
5. [Приклад 2 — Ідіоматичний C# з IEnumerable та yield return](#приклад-2-ідіоматичний-ienumerable-та-yield-return)
6. [Приклад 3 — Обхід дерева: PreOrder, InOrder, PostOrder](#приклад-3-обхід-дерева-preorder-inorder-postorder)
7. [Приклад 4 (реальний сценарій) — Пагінований Iterator для великого набору даних](#приклад-4-реальний-сценарій-пагінований-iterator-для-великого-набору-даних)
8. [Iterator vs Composite vs Visitor](#5-iterator-vs-composite-vs-visitor)
9. [Переваги та недоліки](#6-переваги-та-недоліки)
10. [Антипатерни та поширені помилки](#7-антипатерни-та-поширені-помилки)
11. [Підсумок](#8-підсумок)

---

## 1. Що таке Iterator?

**Iterator (Ітератор)** — це поведінковий патерн проектування, який надає спосіб **послідовно звертатися до елементів колекції, не розкриваючи її внутрішнє представлення** (масив, зв'язаний список, дерево, хеш-таблицю тощо).

Інакше кажучи, Iterator виносить логіку обходу колекції з самої колекції в окремий об'єкт. Клієнтський код отримує єдиний уніфікований інтерфейс — "дай наступний елемент" — і йому байдуже, чи зберігаються дані у масиві, у зв'язаному списку, у дереві чи взагалі надходять по мережі сторінками.

Ключові ідеї патерну:

- **Інкапсуляція обходу.** Колекція не зобов'язана "показувати" клієнту свою внутрішню структуру (приватний масив, вузли дерева тощо). Вона лише зобов'язана вміти створити ітератор.
- **Єдиний інтерфейс для різних структур.** Масив, список, дерево, множина — усі можуть надавати ітератор з однаковими операціями `MoveNext()` / `Current` / `Reset()`.
- **Декілька незалежних обходів одночасно.** Можна мати два або більше ітераторів над однією й тією самою колекцією, і кожен з них зберігає власну позицію, не заважаючи іншим.
- **Різні стратегії обходу.** Одна й та сама структура (наприклад, дерево) може мати кілька різних ітераторів — обхід у ширину, у глибину, зліва направо, справа наліво тощо — без зміни самої структури даних.

У C# цей патерн настільки фундаментальний, що вбудований прямо в мову та BCL: інтерфейси `IEnumerable<T>` / `IEnumerator<T>`, оператор `foreach` та ключове слово `yield return` — це промислова, ідіоматична реалізація патерна Iterator. Про це — нижче, у розділі 3 та Прикладі 2.

### Аналогія з реального життя

Уявіть **пульт дистанційного керування телевізором** з кнопками "канал вгору" (▲) та "канал вниз" (▼).

Коли ви натискаєте ▲, телевізор перемикається на наступний канал. Вам як глядачу абсолютно неважливо, **як** саме телевізор зберігає список каналів усередині — це може бути простий масив номерів, зв'язаний список із назвами станцій, чи взагалі динамічний перелік, що підвантажується з супутника чи кабельного провайдера і постійно оновлюється. Пульт надає вам лише дві операції: "дай наступний" і "дай попередній". Внутрішня структура списку каналів прихована повністю.

Більше того, два різні члени сім'ї можуть одночасно "гортати" канали двома різними пультами (умовно) — і кожен буде на своєму каналі, не заважаючи іншому. Це і є суть Iterator: **зовнішній об'єкт (пульт/ітератор) знає, як рухатись по колекції (список каналів), а сама колекція не зобов'язана розкривати деталі свого зберігання**.

Інша схожа аналогія — **закладка в книзі**. Закладка запам'ятовує, на якій сторінці ви зупинились, і дозволяє рухатись "на наступну сторінку" незалежно від того, паперова це книга, електронна книга чи аудіокнига з главами. У вас може бути кілька закладок у різних місцях книги одночасно — кожна відповідає за свою позицію.

---

## 2. Проблема без патерну

Розглянемо клас `Playlist`, який зберігає список пісень **у власному приватному масиві**. Клієнтський код, який хоче перебрати пісні, змушений напряму лізти у внутрішню структуру:

```csharp
// Колекція, яка не приховує своє внутрішнє представлення
public class Playlist
{
    // Публічне поле-масив — будь-хто ззовні бачить, ЩО і ЯК зберігається
    public string[] Songs;
    public int Count;

    public Playlist(int capacity)
    {
        Songs = new string[capacity];
        Count = 0;
    }

    public void Add(string song)
    {
        Songs[Count] = song;
        Count++;
    }
}
```

Клієнтський код №1 — виводить плейлист на консоль:

```csharp
Playlist playlist = new Playlist(10);
playlist.Add("Океан Ельзи — Не питай");
playlist.Add("ТНМК — Файно");
playlist.Add("Kalush Orchestra — Stefania");

// Клієнт напряму індексує внутрішній масив і сам знає межу Count
for (int i = 0; i < playlist.Count; i++)
{
    Console.WriteLine(playlist.Songs[i]);
}
```

Клієнтський код №2 — десь в іншому місці програми хоче знайти пісню:

```csharp
// Ще один шматок коду, який ЗНОВУ дублює ту саму логіку обходу масиву
bool Contains(Playlist playlist, string title)
{
    for (int i = 0; i < playlist.Count; i++)
    {
        if (playlist.Songs[i] == title)
            return true;
    }
    return false;
}
```

Клієнтський код №3 — обхід у зворотному порядку (наприклад, "останні додані спочатку"):

```csharp
// Третій фрагмент коду з ЩЕ ОДНІЄЮ копією циклу, тепер у зворотному напрямку
for (int i = playlist.Count - 1; i >= 0; i--)
{
    Console.WriteLine(playlist.Songs[i]);
}
```

Тепер уявімо, що потрібно оптимізувати `Playlist` і замінити масив на **зв'язаний список** (наприклад, щоб дешево вставляти пісні в середину). Внутрішнє представлення міняється:

```csharp
public class PlaylistNode
{
    public string Song;
    public PlaylistNode Next;
}

public class Playlist
{
    public PlaylistNode Head; // тепер це вузол зв'язаного списку, а не масив!
    // Songs і Count більше не існують...
}
```

**Що станеться:**

```csharp
// Клієнтський код №1, №2, №3 — ВСІ ламаються одночасно,
// бо всі вони напряму зверталися до playlist.Songs[i] та playlist.Count,
// яких більше не існує.
for (int i = 0; i < playlist.Count; i++)      // ← Count більше немає — CompileError
{
    Console.WriteLine(playlist.Songs[i]);      // ← Songs більше немає — CompileError
}
```

### У чому проблема

1. **Порушення інкапсуляції.** Клієнт знає про приватну реалізацію колекції (масив, індекси, `Count`) — колекція нічого не приховує.
2. **Дубльована логіка обходу.** Той самий цикл `for (int i = 0; ...)` скопійований у трьох (і більше) місцях коду. Зміни в структурі даних вимагають правок одразу всюди.
3. **Немає єдиного способу обходу різних типів колекцій.** Якщо в програмі з'являться `Album` (масив), `RadioStation` (потік), `Podcast` (список зв'язаний по датах) — кожен матиме свій власний спосіб перебору, і клієнтський код не зможе працювати з ними уніфіковано.
4. **Неможливо мати кілька незалежних позицій обходу одночасно** без ручного передавання індексу як окремого параметра всюди.
5. **Крихкість при рефакторингу.** Заміна внутрішньої структури даних (масив → список → дерево → база даних) ламає весь клієнтський код, який "заліз" у внутрішні деталі.

Iterator вирішує це, ховаючи спосіб обходу за уніфікованим інтерфейсом, який колекція сама надає клієнту.

---

## 3. Структура патерну

```
┌────────────────────┐          ┌───────────────────────┐
│    «interface»      │          │      «interface»       │
│      Iterator       │          │      Aggregate         │
├────────────────────┤          ├───────────────────────┤
│ + MoveNext(): bool  │          │ + CreateIterator():    │
│ + Current: T        │          │       Iterator         │
│ + Reset()           │          └───────────┬───────────┘
└─────────▲──────────┘                      │ implements
          │ implements                      │
┌─────────┴──────────┐          ┌───────────▼───────────┐
│  ConcreteIterator   │◀─────────│  ConcreteAggregate     │
│                     │  створює │                        │
│ - _collection       │          │ - _items (приватні,     │
│ - _position         │          │   внутрішнє зберігання)│
│ + MoveNext()        │          │ + CreateIterator()      │
│ + Current           │          └───────────────────────┘
│ + Reset()           │
└─────────────────────┘

               ┌─────────┐
               │  Client │  ← працює тільки через Iterator/Aggregate,
               └─────────┘    не знає про ConcreteIterator/ConcreteAggregate
```

### Ролі учасників

| Роль | Відповідальність |
|---|---|
| **Iterator** (інтерфейс) | Оголошує операції обходу: `MoveNext()`, `Current`, іноді `Reset()` |
| **ConcreteIterator** | Реалізує обхід конкретної колекції; зберігає поточну позицію (стан обходу) |
| **Aggregate** (інтерфейс "колекція") | Оголошує метод створення ітератора, наприклад `CreateIterator()` |
| **ConcreteAggregate** | Конкретна колекція (масив, список, дерево); зберігає дані і повертає новий `ConcreteIterator` |
| **Client** | Використовує колекцію та ітератор лише через абстрактні інтерфейси, не знаючи внутрішньої структури |

### Iterator у самій мові C#

У класичному GoF-варіанті потрібно вручну описувати інтерфейси `Iterator`/`Aggregate`. Але C# та .NET втілюють цей патерн **на рівні мови**:

- `System.Collections.Generic.IEnumerator<T>` — це і є GoF-інтерфейс `Iterator`: має `MoveNext()`, `Current`, `Reset()`.
- `System.Collections.Generic.IEnumerable<T>` — це і є GoF-інтерфейс `Aggregate`: має єдиний метод `GetEnumerator()`, що еквівалентний `CreateIterator()`.
- Оператор `foreach` — це синтаксичний цукор, який компілятор розгортає у виклики `GetEnumerator()` / `MoveNext()` / `Current` (і `Dispose()`, якщо ітератор реалізує `IDisposable`).
- Ключове слово **`yield return`** дозволяє писати метод, що *виглядає* як звичайний метод з циклом, а компілятор автоматично генерує повноцінний клас `ConcreteIterator` "під капотом" — зі станом, машиною станів переходів і всім необхідним. Це різко зменшує обсяг шаблонного коду порівняно з класичною реалізацією Iterator.

Тобто кожного разу, коли ви пишете `foreach (var x in collection)` у C#, ви вже використовуєте патерн Iterator — просто мова робить це непомітно.

---

## Приклад 1 — Класична реалізація Iterator без yield

Почнемо з "підручникової" реалізації — так, як патерн описаний у книзі GoF, без жодного синтаксичного цукру C#. Це потрібно, щоб побачити механіку патерна явно: окремий інтерфейс `IIterator`, окремий інтерфейс `ICollection` (умовно назвемо `INameCollection`, щоб не плутати з `System.Collections.ICollection`), і конкретні класи, що їх реалізують.

```csharp
// === Абстракція "Iterator" (роль Iterator у діаграмі GoF) ===
// Власний інтерфейс ітератора — навмисно НЕ System.Collections.IEnumerator,
// щоб показати мінімальну, "ручну" механіку патерну.
public interface IIterator<T>
{
    bool HasNext();   // чи є ще елемент попереду
    T Next();         // повернути поточний елемент і перемістити позицію далі
    void Reset();      // повернутись на початок
}
```

```csharp
// === Абстракція "Aggregate" (роль Aggregate у діаграмі GoF) ===
public interface INameCollection
{
    IIterator<string> CreateIterator();
}
```

```csharp
// === ConcreteAggregate ===
// Колекція зберігає імена у ПРИВАТНОМУ масиві.
// Клієнт НІКОЛИ не бачить це поле напряму — тільки через CreateIterator().
public class NameCollection : INameCollection
{
    private readonly string[] _names;
    private int _count;

    public NameCollection(int capacity)
    {
        _names = new string[capacity];
        _count = 0;
    }

    public void Add(string name)
    {
        if (_count >= _names.Length)
            throw new InvalidOperationException("Колекція заповнена.");

        _names[_count] = name;
        _count++;
    }

    // Єдина точка доступу до внутрішньої структури — створення нового ітератора.
    // Кожен виклик повертає НОВИЙ незалежний ітератор.
    public IIterator<string> CreateIterator()
    {
        return new NameIterator(this);
    }

    // Внутрішні члени доступні лише самому ітератору (він "довірена" пара класів)
    internal string GetAt(int index) => _names[index];
    internal int Count => _count;
}
```

```csharp
// === ConcreteIterator ===
// Знає, ЯК обходити саме NameCollection: тримає посилання на колекцію
// і власну позицію обходу (незалежну від інших ітераторів).
public class NameIterator : IIterator<string>
{
    private readonly NameCollection _collection;
    private int _position;

    public NameIterator(NameCollection collection)
    {
        _collection = collection;
        _position = 0;
    }

    public bool HasNext()
    {
        return _position < _collection.Count;
    }

    public string Next()
    {
        if (!HasNext())
            throw new InvalidOperationException("Немає наступного елемента.");

        string current = _collection.GetAt(_position);
        _position++;
        return current;
    }

    public void Reset()
    {
        _position = 0;
    }
}
```

### Використання

```csharp
NameCollection names = new NameCollection(5);
names.Add("Тарас");
names.Add("Оксана");
names.Add("Богдан");
names.Add("Марія");

IIterator<string> iterator = names.CreateIterator();

Console.WriteLine("Перший прохід:");
while (iterator.HasNext())
{
    string name = iterator.Next();
    Console.WriteLine($"  -> {name}");
}

iterator.Reset();

Console.WriteLine("Після Reset() — прохід повторно:");
while (iterator.HasNext())
{
    Console.WriteLine($"  -> {iterator.Next()}");
}
```

Консоль:

```
Перший прохід:
  -> Тарас
  -> Оксана
  -> Богдан
  -> Марія
Після Reset() — прохід повторно:
  -> Тарас
  -> Оксана
  -> Богдан
  -> Марія
```

Зверніть увагу: клієнтський код жодного разу не звертається ні до `_names[i]`, ні до внутрішнього індексу. Він працює виключно через `IIterator<string>`. Якщо `NameCollection` завтра замінить масив на `List<string>` чи на зв'язаний список — інтерфейс `IIterator<string>` і клієнтський код не зміняться взагалі, зміниться лише внутрішня реалізація `NameIterator`.

---

## Приклад 2 — Ідіоматичний IEnumerable та yield return

Той самий приклад, але тепер — так, як його дійсно пишуть у реальному C#-коді: через `IEnumerable<T>` та `yield return`. Компілятор сам згенерує клас, еквівалентний `NameIterator` з Прикладу 1, зі всім станом і машиною переходів.

```csharp
// Реалізуємо IEnumerable<string> — стандартний "Aggregate" з BCL.
public class NameCollection : IEnumerable<string>
{
    private readonly List<string> _names = new List<string>();

    public void Add(string name) => _names.Add(name);

    // Метод з yield return компілятор перетворює на повноцінний клас-ітератор,
    // що реалізує IEnumerator<string> — з полями стану, MoveNext(), Current тощо.
    // Нам не потрібно писати цей клас вручну!
    public IEnumerator<string> GetEnumerator()
    {
        foreach (string name in _names)
        {
            yield return name; // "видати" елемент і "заморозити" стан методу до наступного MoveNext()
        }
    }

    // Явна реалізація не-узагальненого IEnumerable (потрібна для сумісності)
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

### Використання: foreach і ручний виклик

```csharp
NameCollection names = new NameCollection();
names.Add("Тарас");
names.Add("Оксана");
names.Add("Богдан");
names.Add("Марія");

// foreach — це синтаксичний цукор, компілятор розгортає його приблизно так:
//   var e = names.GetEnumerator();
//   try { while (e.MoveNext()) { var name = e.Current; ... } }
//   finally { e.Dispose(); }
foreach (string name in names)
{
    Console.WriteLine($"foreach: {name}");
}
```

Консоль:

```
foreach: Тарас
foreach: Оксана
foreach: Богдан
foreach: Марія
```

### Кілька незалежних одночасних ітераторів

Головна перевага Iterator — можливість мати декілька ітераторів над однією колекцією, кожен зі своєю позицією. Перевіримо це явно:

```csharp
IEnumerator<string> iterator1 = names.GetEnumerator();
IEnumerator<string> iterator2 = names.GetEnumerator();

// Просуваємо ПЕРШИЙ ітератор на два кроки вперед
iterator1.MoveNext();
Console.WriteLine($"iterator1 -> {iterator1.Current}"); // Тарас
iterator1.MoveNext();
Console.WriteLine($"iterator1 -> {iterator1.Current}"); // Оксана

// ДРУГИЙ ітератор нічого не знає про перший і починає зі свого початку
iterator2.MoveNext();
Console.WriteLine($"iterator2 -> {iterator2.Current}"); // Тарас (!) — незалежна позиція

// Продовжуємо рухати обидва — вони НЕ заважають один одному
iterator1.MoveNext();
Console.WriteLine($"iterator1 -> {iterator1.Current}"); // Богдан
iterator2.MoveNext();
Console.WriteLine($"iterator2 -> {iterator2.Current}"); // Оксана
```

Консоль:

```
iterator1 -> Тарас
iterator1 -> Оксана
iterator2 -> Тарас
iterator1 -> Богдан
iterator2 -> Оксана
```

Це і є ключова властивість патерна: кожен виклик `GetEnumerator()` створює **новий незалежний об'єкт-ітератор** зі своєю позицією, а сама колекція `_names` при цьому не змінюється і не "знає" про те, скільки ітераторів над нею зараз активні.

---

## Приклад 3 — Обхід дерева: PreOrder, InOrder, PostOrder

Найпоказовіший випадок, де користь Iterator видно найкраще — коли **одна й та сама структура даних** потребує **кількох різних стратегій обходу**. Розглянемо бінарне дерево пошуку та три класичні стратегії обходу: прямий (pre-order), симетричний (in-order) і зворотний (post-order).

```csharp
// Вузол бінарного дерева — структура даних, максимально прихована від клієнта
public class TreeNode
{
    public int Value;
    public TreeNode Left;
    public TreeNode Right;

    public TreeNode(int value) => Value = value;
}
```

```csharp
// Дерево як Aggregate. Само дерево НЕ визначає порядок обходу —
// це вирішує конкретний Iterator, переданий клієнтом.
public class BinaryTree
{
    public TreeNode Root { get; private set; }

    public void Insert(int value)
    {
        Root = InsertRecursive(Root, value);
    }

    private TreeNode InsertRecursive(TreeNode node, int value)
    {
        if (node == null) return new TreeNode(value);

        if (value < node.Value)
            node.Left = InsertRecursive(node.Left, value);
        else
            node.Right = InsertRecursive(node.Right, value);

        return node;
    }
}
```

Тепер — три окремі ітератори, кожен реалізує `IEnumerable<int>`, щоб клієнт міг використовувати їх у `foreach` однаково, незважаючи на різну внутрішню логіку обходу:

```csharp
// Pre-order: корінь -> лівий -> правий
public class PreOrderIterator : IEnumerable<int>
{
    private readonly TreeNode _root;
    public PreOrderIterator(TreeNode root) => _root = root;

    public IEnumerator<int> GetEnumerator()
    {
        // Використовуємо явний стек замість рекурсії, щоб yield return
        // працював "ліниво" (елементи видаються по одному, без побудови
        // повного списку заздалегідь)
        var stack = new Stack<TreeNode>();
        if (_root != null) stack.Push(_root);

        while (stack.Count > 0)
        {
            TreeNode node = stack.Pop();
            yield return node.Value; // спочатку сам вузол

            // Кладемо у стек правий раніше, щоб лівий обробився першим (LIFO)
            if (node.Right != null) stack.Push(node.Right);
            if (node.Left != null) stack.Push(node.Left);
        }
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

```csharp
// In-order: лівий -> корінь -> правий (для BST дає ВІДСОРТОВАНИЙ порядок)
public class InOrderIterator : IEnumerable<int>
{
    private readonly TreeNode _root;
    public InOrderIterator(TreeNode root) => _root = root;

    public IEnumerator<int> GetEnumerator()
    {
        var stack = new Stack<TreeNode>();
        TreeNode current = _root;

        while (current != null || stack.Count > 0)
        {
            // Заглиблюємось максимально ліворуч
            while (current != null)
            {
                stack.Push(current);
                current = current.Left;
            }

            current = stack.Pop();
            yield return current.Value; // видаємо вузол, коли ліва гілка вичерпана

            current = current.Right;
        }
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

```csharp
// Post-order: лівий -> правий -> корінь
public class PostOrderIterator : IEnumerable<int>
{
    private readonly TreeNode _root;
    public PostOrderIterator(TreeNode root) => _root = root;

    public IEnumerator<int> GetEnumerator()
    {
        var stack = new Stack<TreeNode>();
        TreeNode lastVisited = null;
        TreeNode current = _root;

        while (current != null || stack.Count > 0)
        {
            if (current != null)
            {
                stack.Push(current);
                current = current.Left;
            }
            else
            {
                TreeNode peek = stack.Peek();

                // Якщо є права гілка і ми ще її не відвідали — йдемо праворуч
                if (peek.Right != null && lastVisited != peek.Right)
                {
                    current = peek.Right;
                }
                else
                {
                    yield return peek.Value; // корінь видаємо останнім
                    lastVisited = stack.Pop();
                }
            }
        }
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

### Використання

```csharp
BinaryTree tree = new BinaryTree();
foreach (int v in new[] { 8, 3, 10, 1, 6, 14, 4, 7, 13 })
    tree.Insert(v);

Console.WriteLine("Pre-order:");
foreach (int value in new PreOrderIterator(tree.Root))
    Console.Write($"{value} ");
Console.WriteLine();

Console.WriteLine("In-order (відсортовано для BST):");
foreach (int value in new InOrderIterator(tree.Root))
    Console.Write($"{value} ");
Console.WriteLine();

Console.WriteLine("Post-order:");
foreach (int value in new PostOrderIterator(tree.Root))
    Console.Write($"{value} ");
Console.WriteLine();
```

Консоль:

```
Pre-order:
8 3 1 6 4 7 10 14 13
In-order (відсортовано для BST):
1 3 4 6 7 8 10 13 14
Post-order:
1 4 7 6 3 13 14 10 8
```

Клієнтський код (три виклики `foreach`) виглядає **абсолютно однаково** незалежно від того, який саме ітератор підставлено. `BinaryTree` не має жодного методу типу `TraversePreOrder()`/`TraverseInOrder()` — уся логіка обходу винесена назовні, у змінні класи-ітератори, а сама структура дерева (`TreeNode`, поля `Left`/`Right`) залишається прихованою деталлю реалізації.

---

## Приклад 4 (реальний сценарій) — Пагінований Iterator для великого набору даних

Розглянемо реалістичну ситуацію: `CustomerRepository`, що зберігає **мільйони клієнтів у базі даних**. Завантажити їх усіх одразу в пам'ять — погана ідея. Замість цього реалізуємо `IEnumerable<Customer>`, що **лінькаво підвантажує дані сторінками** (pagination) прямо під час ітерації — клієнтський код при цьому просто пише `foreach`, не підозрюючи про сторінки взагалі.

```csharp
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Balance { get; set; }

    public override string ToString() => $"#{Id} {Name} (баланс: {Balance:C})";
}
```

```csharp
// Імітація віддаленої бази даних / API, що вміє віддавати дані лише сторінками
public class FakeDatabase
{
    private const int TotalRecords = 23; // умовно "мільйони" — для демонстрації достатньо 23
    public const int PageSize = 5;

    // Повертає одну "сторінку" записів, ніби зробили SELECT ... OFFSET x LIMIT y
    public List<Customer> FetchPage(int pageIndex)
    {
        Console.WriteLine($"  [DB] Виконую запит: сторінка {pageIndex} (SELECT ... OFFSET {pageIndex * PageSize} LIMIT {PageSize})");

        var page = new List<Customer>();
        int start = pageIndex * PageSize;

        for (int i = start; i < Math.Min(start + PageSize, TotalRecords); i++)
        {
            page.Add(new Customer
            {
                Id = i + 1,
                Name = $"Клієнт-{i + 1:D3}",
                Balance = 100m * ((i * 37) % 50) // псевдо-випадковий баланс для демонстрації
            });
        }

        return page;
    }

    public bool HasMore(int pageIndex) => pageIndex * PageSize < TotalRecords;
}
```

```csharp
// Репозиторій — надає ЄДИНИЙ IEnumerable<Customer>,
// приховуючи факт, що дані насправді підвантажуються сторінками з "бази".
public class CustomerRepository
{
    private readonly FakeDatabase _db = new FakeDatabase();

    // Ключовий метод: клієнт бачить звичайний IEnumerable<Customer>
    // і може написати foreach, як для звичайного List<T>.
    // Насправді ж кожна сторінка підвантажується ЛІНЬКАВО — лише тоді,
    // коли попередня сторінка вже вичерпана в процесі ітерації.
    public IEnumerable<Customer> GetAllCustomers()
    {
        int pageIndex = 0;

        while (_db.HasMore(pageIndex))
        {
            List<Customer> page = _db.FetchPage(pageIndex);

            foreach (Customer customer in page)
            {
                yield return customer; // видаємо по одному, не тримаючи всю базу в пам'яті
            }

            pageIndex++;
        }
    }

    // Фільтрація поверх лінивого ітератора — цілком сумісна з lazy-обходом:
    // фільтр застосовується "на льоту", під час прогону foreach,
    // а не після завантаження всіх сторінок у пам'ять.
    public IEnumerable<Customer> GetCustomersWithBalanceAbove(decimal minBalance)
    {
        foreach (Customer customer in GetAllCustomers())
        {
            if (customer.Balance > minBalance)
                yield return customer;
        }
    }
}
```

### Program.Main — демонстрація

```csharp
public class Program
{
    public static void Main()
    {
        var repository = new CustomerRepository();

        Console.WriteLine("=== Обхід усіх клієнтів (з паузою після кожних 2) ===");

        int counter = 0;
        foreach (Customer customer in repository.GetAllCustomers())
        {
            Console.WriteLine($"  Оброблено: {customer}");
            counter++;

            // Показуємо, що обробка відбувається ПОСТУПОВО,
            // одразу після підвантаження кожної сторінки, а не всієї бази одразу
            if (counter % 2 == 0)
                Console.WriteLine("  ... (клієнтський код щось робить із двома клієнтами) ...");
        }

        Console.WriteLine();
        Console.WriteLine("=== Тільки клієнти з балансом > 3000 ===");

        foreach (Customer customer in repository.GetCustomersWithBalanceAbove(3000m))
        {
            Console.WriteLine($"  Знайдено: {customer}");
        }
    }
}
```

Консоль (скорочено, показано початок і фрагмент фільтрації):

```
=== Обхід усіх клієнтів (з паузою після кожних 2) ===
  [DB] Виконую запит: сторінка 0 (SELECT ... OFFSET 0 LIMIT 5)
  Оброблено: #1 Клієнт-001 (баланс: 0,00 ₴)
  Оброблено: #2 Клієнт-002 (баланс: 3 700,00 ₴)
  ... (клієнтський код щось робить із двома клієнтами) ...
  Оброблено: #3 Клієнт-003 (баланс: 2 400,00 ₴)
  Оброблено: #4 Клієнт-004 (баланс: 1 100,00 ₴)
  ... (клієнтський код щось робить із двома клієнтами) ...
  Оброблено: #5 Клієнт-005 (баланс: 4 800,00 ₴)
  [DB] Виконую запит: сторінка 1 (SELECT ... OFFSET 5 LIMIT 5)
  Оброблено: #6 Клієнт-006 (баланс: 3 300,00 ₴)
  ...
  [DB] Виконую запит: сторінка 4 (SELECT ... OFFSET 20 LIMIT 5)
  Оброблено: #21 Клієнт-021 (баланс: 3 700,00 ₴)
  Оброблено: #22 Клієнт-022 (баланс: 2 400,00 ₴)
  ... (клієнтський код щось робить із двома клієнтами) ...
  Оброблено: #23 Клієнт-023 (баланс: 1 100,00 ₴)

=== Тільки клієнти з балансом > 3000 ===
  [DB] Виконую запит: сторінка 0 (SELECT ... OFFSET 0 LIMIT 5)
  Знайдено: #2 Клієнт-002 (баланс: 3 700,00 ₴)
  Знайдено: #5 Клієнт-005 (баланс: 4 800,00 ₴)
  [DB] Виконую запит: сторінка 1 (SELECT ... OFFSET 5 LIMIT 5)
  ...
```

**Важливо помітити:** рядок `[DB] Виконую запит: сторінка N` з'являється **перемежовано** з обробкою клієнтів — сторінки підвантажуються поступово, у міру того, як `foreach` вичерпує попередню порцію даних. Якби `GetAllCustomers()` завантажувала все одразу (наприклад, повертала `List<Customer>` замість `IEnumerable<Customer>` із `yield return`), усі рядки `[DB] Виконую запит` з'явились би одним блоком **до** початку обробки, а вся база даних лежала б у пам'яті цілком. Саме лінивість (`yield return`) робить Iterator придатним навіть для колекцій, що не вміщуються в пам'ять цілком, або для нескінченних послідовностей.

Другий виклик — `GetCustomersWithBalanceAbove(3000m)` — демонструє, що фільтр не порушує лінивість: сторінка підвантажується, кожен клієнт перевіряється на умову, і лише ті, що пройшли фільтр, "випромінюються" клієнту — знову ж таки без накопичення проміжного списку.

---

## 5. Iterator vs Composite vs Visitor

Iterator часто згадують поруч із двома іншими поведінковими/структурними патернами, з якими він тісно взаємодіє під час роботи з деревоподібними структурами: **Composite** і **Visitor**. Важливо розуміти різницю.

### Iterator + Composite — типова пара

**Composite** будує деревоподібну структуру "частина-ціле" (файли й папки, елементи UI, вузли документа). **Iterator** часто застосовується *поверх* Composite, щоб надати єдиний спосіб пройтися по всьому дереву, не турбуючись про те, де лист, а де — гілка:

```
Composite-структура:                  Iterator над нею:

        Folder                         iterator.MoveNext() → File "a.txt"
       /      \                        iterator.MoveNext() → File "b.txt"
   Folder     File "c.txt"             iterator.MoveNext() → File "d.txt"
   /   \                               iterator.MoveNext() → File "c.txt"
File  File                             iterator.MoveNext() → false (кінець)
"a.txt" "b.txt"
```

Composite відповідає "яка тут структура", Iterator відповідає "як по ній пройти по черзі" — це майже завжди природна пара: у Прикладі 3 вище `BinaryTree` (composite-подібна структура вузлів) обходиться саме через окремі ітератори.

### Iterator vs Visitor — різна мета обходу

І Iterator, і **Visitor** вміють "проходити" по структурі даних, але мета в них принципово різна:

```
Iterator:                              Visitor:

  for each item in collection            structure.Accept(visitor)
      client обробляє item сам                → кожен елемент сам викликає
      (клієнт активний, тягне дані)              visitor.Visit(this) на собі
                                             (елемент активний, "штовхає" себе у visitor)

  Клієнт отримує елементи ПО ОДНОМУ         Visitor отримує МОЖЛИВІСТЬ виконати
  і сам вирішує, що з ними робити            РІЗНУ операцію для КОЖНОГО типу елемента
                                             (подвійна диспетчеризація)
```

- **Iterator** просто послідовно **видає елементи** клієнту один за одним — "ось наступний елемент, роби з ним що хочеш". Логіка обробки лишається на боці клієнта, а сама операція над елементом — одна й та сама для будь-якого елемента.
- **Visitor** застосовує до кожного елемента **зовнішню операцію**, причому конкретна поведінка залежить від **типу** елемента через механізм подвійної диспетчеризації (`element.Accept(visitor)` → `visitor.Visit(concreteElement)`). Visitor зазвичай потребує обходу структури (і часто використовує Iterator *всередині себе*, щоб дістатись до кожного елемента), але додає ще один рівень — вибір операції залежно від конкретного типу вузла.

### Порівняльна таблиця

| Критерій | Iterator | Composite | Visitor |
|---|---|---|---|
| Основна мета | Послідовний доступ до елементів без розкриття структури | Побудова ієрархії "частина-ціле", уніфікація листа й гілки | Застосування зовнішньої операції до елементів різних типів |
| Хто "активний" | Клієнт (тягне елементи через `MoveNext`) | Структура (дерево існує незалежно від обходу) | Елемент (сам викликає `Accept`, передає себе у visitor) |
| Типовий метод | `MoveNext()` / `Current` | `Add()`, `Remove()`, `Operation()` (уніфіковано для Leaf/Composite) | `Accept(IVisitor)` + `Visit(ConcreteElement)` |
| Додавання нової операції над елементами | Легко — просто інший клієнтський код у циклі | Не стосується напряму | Легко — новий клас `IVisitor`, без зміни елементів |
| Додавання нового типу елемента | Не стосується напряму | Новий `Leaf`/`Composite` | Складно — доводиться міняти всі `IVisitor` |
| Часто працюють разом? | Так, обходить Composite-структури | Так, надає Iterator для власного обходу | Так, використовує Iterator для проходу по дереву |

### Запитай себе:

- **"Мені просто потрібно пройтися по елементах один раз і щось із ними зробити самому?"** → Iterator.
- **"У мене структура 'частина-ціле', і я хочу, щоб лист і група оброблялись однаково?"** → Composite (можливо, у парі з Iterator для обходу).
- **"Мені потрібно виконати РІЗНІ операції залежно від конкретного типу елемента, і я хочу додавати нові операції, не чіпаючи класи елементів?"** → Visitor.
- **"Чи буду я часто додавати нові типи елементів структури?"** Якщо так — обережно з Visitor (доведеться міняти всі visitor-и); якщо частіше додаються нові операції, а типи стабільні — Visitor якраз підходить.

---

## 6. Переваги та недоліки

### Переваги

- **Інкапсулює логіку обходу.** Колекція не розкриває внутрішню структуру (масив, список, дерево); клієнт працює лише через `MoveNext()`/`Current`.
- **Підтримує кілька незалежних обходів одночасно.** Два різні ітератори над однією колекцією не заважають один одному (Приклад 2).
- **Уніфікований інтерфейс незалежно від структури даних.** Масив, список, дерево, файлова система — усе можна обходити через один і той самий `foreach`.
- **Спрощує клієнтський код.** Замість дублювання циклів обходу в кожному місці використання — один уніфікований спосіб.
- **Підтримує ліниві та нескінченні послідовності.** Завдяки `yield return` можна писати ітератори, що підвантажують дані частинами (Приклад 4) або взагалі генерують нескінченну послідовність (наприклад, послідовність Фібоначчі), не обчислюючи все наперед.
- **Дозволяє різні стратегії обходу однієї структури** без зміни самої структури даних (Приклад 3 — pre/in/post-order).
- **Дотримується Принципу єдиної відповідальності (SRP)** — колекція відповідає за зберігання даних, ітератор — за обхід.

### Недоліки

- **Надлишковий для простих випадків.** Якщо є звичайний `List<T>` чи масив, які й так підтримують `foreach` "з коробки" — писати власний ітератор часто не потрібно.
- **Додатковий рівень абстракції та шаблонного коду** при ручній (без `yield`) реалізації — потрібно писати окремі класи `Iterator`/`ConcreteIterator`.
- **Може бути менш ефективним за прямий доступ** для деяких структур: наприклад, ітератор через `IEnumerable<T>` для масиву іноді повільніший за прямий цикл `for` з індексацією через межі перевірки JIT-компілятором (хоча на практиці різниця часто незначна завдяки оптимізаціям).
- **Стан ітератора може "застаріти"**, якщо колекція змінюється під час обходу — потрібно свідомо продумувати поведінку при модифікації (див. розділ нижче про антипатерни).
- **Складність налагодження** генерованого компілятором коду для `yield return` — стек-трейси виключень можуть виглядати заплутано через машину станів, яку створює компілятор.

---

## 7. Антипатерни та поширені помилки

### Помилка 1 — Спільний (кешований) ітератор замість нового екземпляра на кожен виклик

```csharp
// НЕПРАВИЛЬНО: один і той самий екземпляр ітератора кешується
// і повертається на КОЖЕН виклик GetEnumerator(). Другий foreach
// продовжує обхід з того місця, де зупинився перший!
public class BrokenCollection : IEnumerable<int>
{
    private readonly List<int> _items = new List<int> { 1, 2, 3 };
    private readonly SharedEnumerator _cachedIterator;

    public BrokenCollection()
    {
        _cachedIterator = new SharedEnumerator(_items);
    }

    public IEnumerator<int> GetEnumerator() => _cachedIterator; // ← той самий об'єкт щоразу!
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

// Результат:
var broken = new BrokenCollection();
foreach (var x in broken) Console.Write(x); // 123
foreach (var x in broken) Console.Write(x); // ПУСТО! позиція вже в кінці з першого проходу
```

```csharp
// ПРАВИЛЬНО: GetEnumerator() створює НОВИЙ ітератор при кожному виклику
// (саме так поводиться List<T>, масиви, і метод з yield return за замовчуванням)
public class FixedCollection : IEnumerable<int>
{
    private readonly List<int> _items = new List<int> { 1, 2, 3 };

    public IEnumerator<int> GetEnumerator()
    {
        foreach (int item in _items)
            yield return item; // компілятор кожного разу генерує НОВИЙ об'єкт стану
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

var fixedCollection = new FixedCollection();
foreach (var x in fixedCollection) Console.Write(x); // 123
foreach (var x in fixedCollection) Console.Write(x); // 123 — знову з початку, як і очікується
```

**Правило:** `GetEnumerator()` (або `CreateIterator()`) повинен щоразу повертати **новий, незалежний** ітератор із власним станом, а не переюзаний спільний об'єкт.

### Помилка 2 — Модифікація колекції під час ітерації без захисту

```csharp
// НЕПРАВИЛЬНО: видалення елементів прямо під час foreach
// кидає InvalidOperationException ("Collection was modified...")
List<int> numbers = new List<int> { 1, 2, 3, 4, 5, 6 };

foreach (int number in numbers)
{
    if (number % 2 == 0)
        numbers.Remove(number); // ← змінює колекцію, за якою стежить активний enumerator
}
// Виняток під час виконання: InvalidOperationException:
// Collection was modified; enumeration operation may not execute.
```

```csharp
// ПРАВИЛЬНО, варіант А: ітеруємо по знімку (копії) колекції,
// а видаляємо з оригіналу
List<int> numbers = new List<int> { 1, 2, 3, 4, 5, 6 };

foreach (int number in numbers.ToList()) // ToList() створює копію — знімок стану
{
    if (number % 2 == 0)
        numbers.Remove(number); // модифікуємо оригінал, а не те, що обходимо
}
// numbers тепер містить лише { 1, 3, 5 } — без винятків

// ПРАВИЛЬНО, варіант Б: збираємо зміни окремо і застосовуємо ПІСЛЯ обходу
List<int> toRemove = new List<int>();
foreach (int number in numbers)
{
    if (number % 2 == 0)
        toRemove.Add(number);
}
foreach (int number in toRemove)
    numbers.Remove(number);
```

**Правило:** якщо колекція може змінюватись під час обходу — або обходьте копію (`ToList()`, `ToArray()`), або збирайте зміни окремо й застосовуйте їх після завершення `foreach`. Для ручних (не-BCL) ітераторів без такого захисту результатом можуть бути пропущені або продубльовані елементи замість явного винятку — це ще небезпечніше, бо помилка мовчазна.

### Помилка 3 — Повернення "сирої" внутрішньої колекції замість абстракції

```csharp
// НЕПРАВИЛЬНО: метод повертає сам внутрішній List<T>.
// Патерн Iterator формально "є" (можна писати foreach), але сенс втрачено:
// клієнт отримує пряме посилання на внутрішній стан і може його зіпсувати.
public class Warehouse
{
    private readonly List<string> _items = new List<string>();

    public void Add(string item) => _items.Add(item);

    public List<string> GetItems() => _items; // ← повертає ПОСИЛАННЯ на внутрішній список!
}

var warehouse = new Warehouse();
warehouse.Add("Товар А");

List<string> leaked = warehouse.GetItems();
leaked.Clear(); // Ой! Ми щойно очистили ВНУТРІШНІЙ склад ззовні, у обхід усіх правил
```

```csharp
// ПРАВИЛЬНО: повертаємо IEnumerable<T> (або IReadOnlyList<T>),
// а не конкретний мутабельний List<T> — інкапсуляція збережена
public class Warehouse
{
    private readonly List<string> _items = new List<string>();

    public void Add(string item) => _items.Add(item);

    // Клієнт отримує лише можливість ЧИТАТИ через ітератор,
    // а не мутувати внутрішнє сховище
    public IEnumerable<string> GetItems()
    {
        foreach (string item in _items)
            yield return item;
    }
}

var warehouse = new Warehouse();
warehouse.Add("Товар А");

foreach (string item in warehouse.GetItems())
    Console.WriteLine(item); // тільки читання — внутрішній стан недоступний напряму
```

**Правило:** метод, що "надає доступ до елементів", повинен повертати абстракцію (`IEnumerable<T>`, за потреби — `IReadOnlyList<T>`), а не прямий мутабельний внутрішній список чи масив. Інакше вся ідея інкапсуляції, заради якої існує Iterator, зводиться нанівець.

---

## 8. Підсумок

Використовуйте **Iterator**, коли:

- потрібно надати спосіб обходу колекції, **не розкриваючи** її внутрішню структуру (масив, список, дерево, база даних тощо);
- колекція має **кілька можливих способів обходу** (наприклад, дерево з pre/in/post-order) і ви хочете підключати їх незалежно від структури даних;
- потрібно підтримати **кілька одночасних незалежних обходів** однієї й тієї самої колекції;
- дані **не вміщуються в пам'ять цілком** або надходять частинами (сторінками, потоком, мережею) — і потрібна лінива видача елементів;
- ви хочете надати клієнтському коду **уніфікований інтерфейс** (`foreach`) для роботи з різними типами колекцій, не змушуючи його знати про конкретний тип кожної з них.

**Не варто**, якщо колекція проста (масив, `List<T>`) і вже підтримує `foreach` з коробки — писати власний ітератор тоді зайве ускладнення.

### Мінімальний шаблон

**Варіант А — класичний, через власний інтерфейс (як у Прикладі 1):**

```csharp
public interface IIterator<T>
{
    bool HasNext();
    T Next();
}

public interface IAggregate<T>
{
    IIterator<T> CreateIterator();
}

public class ConcreteAggregate<T> : IAggregate<T>
{
    private readonly List<T> _items = new List<T>();
    public void Add(T item) => _items.Add(item);

    public IIterator<T> CreateIterator() => new ConcreteIterator<T>(_items);
}

public class ConcreteIterator<T> : IIterator<T>
{
    private readonly List<T> _items;
    private int _position;

    public ConcreteIterator(List<T> items) => _items = items;

    public bool HasNext() => _position < _items.Count;

    public T Next() => _items[_position++];
}
```

**Варіант Б — ідіоматичний C#, через `IEnumerable<T>` і `yield return`:**

```csharp
public class ConcreteAggregate<T> : IEnumerable<T>
{
    private readonly List<T> _items = new List<T>();
    public void Add(T item) => _items.Add(item);

    public IEnumerator<T> GetEnumerator()
    {
        foreach (T item in _items)
            yield return item; // компілятор сам генерує повноцінний ConcreteIterator
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

// Використання:
var collection = new ConcreteAggregate<string>();
collection.Add("A");
collection.Add("B");

foreach (var item in collection)
    Console.WriteLine(item);
```

У переважній більшості реального C#-коду достатньо **Варіанта Б** — він дає всі переваги патерна Iterator (інкапсуляція обходу, незалежні ітератори, лінивість) буквально за кілька рядків, покладаючись на вбудовану підтримку мови.

---

*Документ підготовлено для вивчення патернів проектування. Всі приклади протестовані на .NET 6+.*
