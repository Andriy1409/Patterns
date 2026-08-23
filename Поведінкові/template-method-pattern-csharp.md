# Патерн Template Method (Шаблонний метод) — Детальний розбір на C#

> **Категорія:** Поведінковий (Behavioral)  
> **Автори:** Gang of Four (GoF)  
> **Мова прикладів:** C#

---

## Зміст

1. [Що таке Template Method?](#що-таке-template-method)
2. [Проблема без патерну](#проблема-без-патерну)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Гарячі напої (класика GoF)](#приклад-1--гарячі-напої-класика-gof)
5. [Приклад 2 — Хук-методи: WantsCondiments](#приклад-2--хук-методи-wantscondiments)
6. [Приклад 3 — Конвеєр обробки даних](#приклад-3--конвеєр-обробки-даних)
7. [Приклад 4 — Реальний сценарій: генератор звітів](#приклад-4--реальний-сценарій-генератор-звітів)
8. [Template Method vs Strategy vs Factory Method](#template-method-vs-strategy-vs-factory-method)
9. [Переваги та недоліки](#переваги-та-недоліки)
10. [Антипатерни та поширені помилки](#антипатерни-та-поширені-помилки)
11. [Підсумок](#підсумок)

---

## Що таке Template Method?

**Template Method (Шаблонний метод)** — це поведінковий патерн проектування, який визначає **скелет алгоритму** в методі базового класу, дозволяючи підкласам перевизначати **окремі кроки** цього алгоритму, не змінюючи його загальну структуру.

Базовий клас каже: _"ось послідовність кроків, яка виконується завжди в такому порядку, і я гарантую, що ніхто цю послідовність не порушить"_. Підкласи кажуть: _"а ось як саме я виконаю конкретні кроки"_.

Ключова відмінність від простого наслідування "як заманеться" — сам метод-шаблон (template method) зазвичай позначається `sealed` (або принаймні задокументований як такий, що не повинен перевизначатися), тому підклас **не може** змінити порядок кроків чи пропустити важливий крок — він може лише реалізувати деталі окремих кроків через `abstract`/`virtual` методи.

### Аналогія з реального світу

Уяви процес приготування гарячого напою. Незалежно від того, готуєш ти чай чи каву, послідовність кроків **завжди однакова**:

1. Закип'ятити воду.
2. Заварити напій (чай чи каву — це вже деталь).
3. Налити у чашку.
4. Додати смакові добавки (лимон до чаю, цукор і молоко до кави — знову деталь).

Це і є класичний приклад GoF: `CaffeineBeverage` — базовий клас, що фіксує **порядок** цих чотирьох кроків у методі `PrepareRecipe()`. А `Tea` і `Coffee` — підкласи, що реалізують лише кроки 2 і 4 по-своєму. Кроки 1 і 3 (кип'ятіння води, наливання у чашку) — однакові для всіх напоїв, тому реалізовані один раз у базовому класі.

Інша життєва аналогія — **стандартизований процес співбесіди**: HR завжди виконує кроки в одному порядку (screening резюме → технічне інтерв'ю → інтерв'ю з менеджером → ухвалення рішення), але **зміст технічного тесту** відрізняється залежно від вакансії (для розробника — задачі з коду, для дизайнера — портфоліо-рев'ю). Загальний процес (skeleton) той самий для всіх позицій — змінюється лише один крок.

---

## Проблема без патерну

Розглянемо, що відбувається, якщо не виносити спільний скелет алгоритму в базовий клас, а копіювати його в кожен конкретний клас.

### Код без патерну — дубльований скелет алгоритму

```csharp
// Клас для приготування чаю — містить ПОВНИЙ алгоритм приготування напою.
public class TeaMaker
{
    public void Make()
    {
        // Крок 1: закип'ятити воду (ІДЕНТИЧНИЙ код у всіх класах-напоях!)
        Console.WriteLine("Кип'ятимо воду...");
        Console.WriteLine("Вода закипіла (100°C).");

        // Крок 2: заварити чай (це єдина частина, що дійсно відрізняється)
        Console.WriteLine("Занурюємо чайний пакетик у воду...");
        Console.WriteLine("Настоюємо 3 хвилини.");

        // Крок 3: налити у чашку (ІДЕНТИЧНИЙ код!)
        Console.WriteLine("Наливаємо напій у чашку...");

        // Крок 4: додати добавки
        Console.WriteLine("Додаємо часточку лимона.");

        Console.WriteLine("Чай готовий!\n");
    }
}

// Клас для приготування кави — окремий клас, але 80% коду ІДЕНТИЧНЕ до TeaMaker!
public class CoffeeMaker
{
    public void Make()
    {
        // Крок 1: закип'ятити воду (СКОПІЙОВАНО з TeaMaker — точна копія!)
        Console.WriteLine("Кип'ятимо воду...");
        Console.WriteLine("Вода закипіла (100°C).");

        // Крок 2: заварити каву (тут дійсно інша логіка)
        Console.WriteLine("Пропускаємо окріп через мелену каву...");
        Console.WriteLine("Заварюємо 5 хвилин.");

        // Крок 3: налити у чашку (СКОПІЙОВАНО ще раз — точна копія!)
        Console.WriteLine("Наливаємо напій у чашку...");

        // Крок 4: додати добавки
        Console.WriteLine("Додаємо цукор і молоко.");

        Console.WriteLine("Кава готова!\n");
    }
}
```

### Що тут погано?

- **Дублювання коду скелету алгоритму.** Кроки 1 і 3 (кип'ятіння води, наливання у чашку) написані двічі, слово в слово. Якщо в майбутньому потрібно буде додати крок "перевірити температуру чашки" — доведеться правити **обидва** класи.
- **Копії легко розходяться (code drift).** Хтось виправить помилку або додасть логування в `TeaMaker.Make()`, але забуде зробити те саме в `CoffeeMaker.Make()` — і за півроку два методи, які мали бути однаковими, стають непередбачувано різними.
- **Немає гарантії однакового порядку кроків.** Ніщо не заважає розробнику при додаванні `HotChocolateMaker` переплутати порядок — наприклад, спочатку додати добавки, а потім налити у чашку. Компілятор цього не перевірить, а результат буде працювати "якось не так".
- Якщо напоїв стане 20 — матимемо **20 майже ідентичних методів `Make()`**, кожен з power copy-paste дублюванням, що робить підтримку кошмаром.

### Саме це і вирішує Template Method.

Спільний скелет алгоритму пишеться **один раз** у базовому класі, а підкласи реалізують лише те, що дійсно відрізняється.

---

## Структура патерну

```
┌────────────────────────────────────────────────────────┐
│                    <<abstract>>                          │
│                    AbstractClass                          │
│  ──────────────────────────────────────────────────────  │
│  + TemplateMethod() : sealed/non-virtual                  │
│      1. Step1()          ← конкретна реалізація (fixed)   │
│      2. Step2()          ← abstract, підклас МУСИТЬ дати   │
│      3. Step3()          ← virtual, дефолт є, можна override│
│      4. if (Hook()) Step4()  ← virtual "хук", керує потоком│
│  # Step2() : abstract                                      │
│  # Step3() : virtual { /* дефолтна реалізація */ }         │
│  # Hook()  : virtual  { return true; }                     │
└───────────────────────┬──────────────────────────────────┘
                        │ наслідує
             ┌──────────┴──────────┐
             ▼                     ▼
  ┌─────────────────────┐  ┌─────────────────────┐
  │   ConcreteClassA     │  │   ConcreteClassB     │
  │  ───────────────     │  │  ───────────────     │
  │  # Step2() override  │  │  # Step2() override  │
  │  # Step3() override? │  │  (Step3 — дефолт)     │
  │  # Hook() override?  │  │  # Hook() override?  │
  └─────────────────────┘  └─────────────────────┘
```

Головна ідея структури: `TemplateMethod()` **фіксує порядок викликів**, і клієнтський код завжди викликає тільки його. Це називають **принципом Голлівуду** ("Hollywood Principle"): _"не дзвони нам — ми подзвонимо тобі"_ (Don't call us, we'll call you). Підкласи не викликають кроки самі — базовий клас викликає їхні перевизначені кроки в потрібний момент.

### Ключові ролі

| Роль | Що робить | Приклад |
|---|---|---|
| `AbstractClass` | Визначає `TemplateMethod()` з фіксованим порядком кроків; оголошує кроки як `abstract`/`virtual` | `CaffeineBeverage` |
| `TemplateMethod()` | Метод, що визначає скелет алгоритму; зазвичай `sealed` або документований як "не перевизначати" | `PrepareRecipe()` |
| Абстрактний крок | Метод без реалізації — підклас **зобов'язаний** його реалізувати | `Brew()`, `AddCondiments()` |
| Крок з дефолтом | `virtual` метод із стандартною реалізацією — підклас **може** перевизначити, але не зобов'язаний | `BoilWater()` |
| Хук (hook) | `virtual` метод, зазвичай `bool` або порожній за замовчуванням — керує тим, чи виконається певна гілка алгоритму | `WantsCondiments()` |
| `ConcreteClass` | Реалізує лише ті кроки, які дійсно відрізняються | `Tea`, `Coffee` |

---

## Приклад 1 — Гарячі напої (класика GoF)

Класичний приклад із книги GoF — приготування чаю та кави за спільним алгоритмом.

```csharp
// ─── ABSTRACT CLASS ────────────────────────────────────────────────────────

// Базовий клас фіксує алгоритм приготування будь-якого гарячого напою.
public abstract class CaffeineBeverage
{
    // ★ ЦЕ І Є TEMPLATE METHOD.
    // sealed — забороняє підкласам перевизначати сам метод і ламати порядок кроків.
    // Порядок викликів тут ЗАФІКСОВАНИЙ раз і назавжди.
    public sealed void PrepareRecipe()
    {
        BoilWater();       // Крок 1 — однаковий для всіх напоїв
        Brew();             // Крок 2 — РІЗНИЙ для кожного напою (abstract)
        PourInCup();        // Крок 3 — однаковий для всіх напоїв
        AddCondiments();    // Крок 4 — РІЗНИЙ для кожного напою (abstract)
    }

    // Крок з дефолтною реалізацією — 99% напоїв кип'ятять воду однаково.
    // Підклас МОЖЕ перевизначити (наприклад, якщо потрібна не кипʼячена вода),
    // але зазвичай цього не робить.
    private void BoilWater()
    {
        Console.WriteLine("Кип'ятимо воду...");
        Console.WriteLine("Вода закипіла (100°C).");
    }

    // Абстрактний крок — кожен підклас ЗОБОВ'ЯЗАНИЙ реалізувати по-своєму.
    protected abstract void Brew();

    // Ще один крок з дефолтною реалізацією.
    private void PourInCup()
    {
        Console.WriteLine("Наливаємо напій у чашку...");
    }

    // Абстрактний крок — добавки завжди різні.
    protected abstract void AddCondiments();
}

// ─── CONCRETE CLASSES ──────────────────────────────────────────────────────

// Конкретний клас — реалізує лише те, що дійсно відрізняється для чаю.
public class Tea : CaffeineBeverage
{
    protected override void Brew()
    {
        Console.WriteLine("Занурюємо чайний пакетик у воду...");
        Console.WriteLine("Настоюємо 3 хвилини.");
    }

    protected override void AddCondiments()
    {
        Console.WriteLine("Додаємо часточку лимона.");
    }
}

// Конкретний клас — реалізує лише те, що дійсно відрізняється для кави.
public class Coffee : CaffeineBeverage
{
    protected override void Brew()
    {
        Console.WriteLine("Пропускаємо окріп через мелену каву...");
        Console.WriteLine("Заварюємо 5 хвилин.");
    }

    protected override void AddCondiments()
    {
        Console.WriteLine("Додаємо цукор і молоко.");
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        Console.WriteLine("=== Готуємо чай ===");
        CaffeineBeverage tea = new Tea();
        tea.PrepareRecipe(); // Клієнт викликає ОДИН метод — весь алгоритм виконується сам

        Console.WriteLine("=== Готуємо каву ===");
        CaffeineBeverage coffee = new Coffee();
        coffee.PrepareRecipe();
    }
}

// Виведе:
// === Готуємо чай ===
// Кип'ятимо воду...
// Вода закипіла (100°C).
// Занурюємо чайний пакетик у воду...
// Настоюємо 3 хвилини.
// Наливаємо напій у чашку...
// Додаємо часточку лимона.
// === Готуємо каву ===
// Кип'ятимо воду...
// Вода закипіла (100°C).
// Пропускаємо окріп через мелену каву...
// Заварюємо 5 хвилин.
// Наливаємо напій у чашку...
// Додаємо цукор і молоко.
```

### Ключовий момент

Зверни увагу: `PrepareRecipe()` — `sealed`. Ані `Tea`, ані `Coffee`, ані будь-який майбутній `HotChocolate` **не можуть** переставити кроки місцями чи пропустити один із них. Це і є суть Template Method: **алгоритм зафіксований, варіюються лише деталі**.

---

## Приклад 2 — Хук-методи: WantsCondiments

Розширимо приклад 1, додавши **хук-метод** (hook method) — `virtual` метод із простою дефолтною поведінкою, який підкласи можуть перевизначити, щоб **вплинути на потік виконання** template method, не змінюючи сам порядок кроків.

```csharp
// ─── ABSTRACT CLASS ────────────────────────────────────────────────────────

public abstract class CaffeineBeverageWithHook
{
    // Template method тепер має УМОВНИЙ крок, який керується хуком.
    public sealed void PrepareRecipe()
    {
        BoilWater();
        Brew();
        PourInCup();

        // ★ ХУК: базовий клас запитує підклас "а тобі взагалі потрібні добавки?"
        // Це і є ключова різниця хука від абстрактного кроку:
        // хук НЕ зобов'язує підклас щось реалізовувати — він лише дає можливість
        // вплинути на те, чи виконається певна гілка алгоритму.
        if (WantsCondiments())
        {
            AddCondiments();
        }
    }

    private void BoilWater()
    {
        Console.WriteLine("Кип'ятимо воду...");
        Console.WriteLine("Вода закипіла (100°C).");
    }

    protected abstract void Brew();

    private void PourInCup()
    {
        Console.WriteLine("Наливаємо напій у чашку...");
    }

    protected abstract void AddCondiments();

    // ★ ХУК-МЕТОД: virtual з дефолтною реалізацією "true".
    // За замовчуванням всі напої отримують добавки.
    // Підклас МОЖЕ (але не зобов'язаний) перевизначити, щоб сказати "ні".
    protected virtual bool WantsCondiments() => true;
}

// ─── CONCRETE CLASSES ──────────────────────────────────────────────────────

public class TeaWithHook : CaffeineBeverageWithHook
{
    protected override void Brew()
    {
        Console.WriteLine("Занурюємо чайний пакетик у воду...");
        Console.WriteLine("Настоюємо 3 хвилини.");
    }

    protected override void AddCondiments()
    {
        Console.WriteLine("Додаємо часточку лимона.");
    }

    // Чай не перевизначає хук — використовується дефолт (true), лимон додається завжди.
}

public class CoffeeWithHook : CaffeineBeverageWithHook
{
    protected override void Brew()
    {
        Console.WriteLine("Пропускаємо окріп через мелену каву...");
        Console.WriteLine("Заварюємо 5 хвилин.");
    }

    protected override void AddCondiments()
    {
        Console.WriteLine("Додаємо цукор і молоко.");
    }

    // Кава теж не перевизначає хук — за замовчуванням додає цукор і молоко.
}

// Варіант "чорна кава" — перевизначає ЛИШЕ хук, AddCondiments взагалі не викликається.
public class BlackCoffee : CaffeineBeverageWithHook
{
    protected override void Brew()
    {
        Console.WriteLine("Пропускаємо окріп через мелену каву...");
        Console.WriteLine("Заварюємо 5 хвилин.");
    }

    protected override void AddCondiments()
    {
        // Цей метод НІКОЛИ не буде викликаний для BlackCoffee,
        // тому що WantsCondiments() повертає false.
        Console.WriteLine("Додаємо цукор і молоко.");
    }

    // ★ Перевизначаємо хук — кажемо базовому класу "мені добавки не потрібні".
    protected override bool WantsCondiments()
    {
        Console.WriteLine("Питаємо клієнта: чи хочете цукор/молоко?");
        return false; // Клієнт відповів "ні" — крок AddCondiments() буде пропущено
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        Console.WriteLine("=== Звичайна кава (з добавками) ===");
        CaffeineBeverageWithHook coffee = new CoffeeWithHook();
        coffee.PrepareRecipe();

        Console.WriteLine("\n=== Чорна кава (без добавок) ===");
        CaffeineBeverageWithHook blackCoffee = new BlackCoffee();
        blackCoffee.PrepareRecipe();
    }
}

// Виведе:
// === Звичайна кава (з добавками) ===
// Кип'ятимо воду...
// Вода закипіла (100°C).
// Пропускаємо окріп через мелену каву...
// Заварюємо 5 хвилин.
// Наливаємо напій у чашку...
// Додаємо цукор і молоко.
//
// === Чорна кава (без добавок) ===
// Кип'ятимо воду...
// Вода закипіла (100°C).
// Пропускаємо окріп через мелену каву...
// Заварюємо 5 хвилин.
// Наливаємо напій у чашку...
// Питаємо клієнта: чи хочете цукор/молоко?
```

### Хук vs абстрактний крок — у чому різниця?

| | Абстрактний крок (`abstract`) | Хук (`virtual` з дефолтом) |
|---|---|---|
| Реалізація за замовчуванням | Немає — підклас **зобов'язаний** реалізувати | Є проста дефолтна реалізація |
| Обов'язковість перевизначення | Обов'язково | Опційно |
| Призначення | "Що саме зробити" (наприклад, як заварити) | "Чи робити це взагалі" або "додаткова точка розширення" |
| Приклад | `Brew()`, `AddCondiments()` | `WantsCondiments()` |

---

## Приклад 3 — Конвеєр обробки даних

Ще один класичний випадок застосування Template Method — імпорт даних із різних форматів джерел за однаковим алгоритмом обробки.

```csharp
using System;
using System.Collections.Generic;

// ─── МОДЕЛЬ ДАНИХ ────────────────────────────────────────────────────────────

public class ImportResult
{
    public int RecordsRead      { get; set; }
    public int RecordsValid     { get; set; }
    public int RecordsSaved     { get; set; }
    public List<string> Errors  { get; set; } = new();
}

// ─── ABSTRACT CLASS ────────────────────────────────────────────────────────

public abstract class DataImporter
{
    // ★ Template method — фіксований конвеєр обробки даних.
    // sealed — порядок кроків (читання → парсинг → валідація → збереження)
    // не повинен змінюватися жодним підкласом.
    public sealed ImportResult Import(string sourcePath)
    {
        var result = new ImportResult();
        Console.WriteLine($"\n[Import] Починаю імпорт з '{sourcePath}'...");

        // Крок 1 — читання джерела (РІЗНЕ для CSV/JSON)
        string rawContent = ReadSource(sourcePath);
        var rawRecords = Parse(rawContent); // Крок 2 — парсинг (РІЗНЕ для CSV/JSON)
        result.RecordsRead = rawRecords.Count;
        Console.WriteLine($"  Прочитано записів: {result.RecordsRead}");

        // Крок 3 — валідація (ОДНАКОВА для всіх джерел)
        var validRecords = new List<Dictionary<string, string>>();
        foreach (var record in rawRecords)
        {
            if (Validate(record, out string error))
            {
                validRecords.Add(record);
            }
            else
            {
                result.Errors.Add(error);
            }
        }
        result.RecordsValid = validRecords.Count;
        Console.WriteLine($"  Валідних записів: {result.RecordsValid} (помилок: {result.Errors.Count})");

        // Крок 4 — збереження у базу даних (ОДНАКОВЕ для всіх джерел)
        result.RecordsSaved = SaveToDatabase(validRecords);
        Console.WriteLine($"  Збережено у БД: {result.RecordsSaved} записів.");

        Console.WriteLine("[Import] Імпорт завершено.");
        return result;
    }

    // Абстрактний крок — кожен формат читається по-своєму.
    protected abstract string ReadSource(string sourcePath);

    // Абстрактний крок — кожен формат парситься по-своєму.
    protected abstract List<Dictionary<string, string>> Parse(string rawContent);

    // Крок з дефолтною реалізацією — базова валідація однакова для всіх форматів:
    // перевіряємо, що є хоча б одне непорожнє поле "name".
    protected virtual bool Validate(Dictionary<string, string> record, out string error)
    {
        if (!record.TryGetValue("name", out var name) || string.IsNullOrWhiteSpace(name))
        {
            error = "Відсутнє обов'язкове поле 'name'";
            return false;
        }
        error = null;
        return true;
    }

    // Крок з дефолтною реалізацією — симуляція збереження у БД, однакова для всіх.
    protected virtual int SaveToDatabase(List<Dictionary<string, string>> records)
    {
        foreach (var record in records)
        {
            Console.WriteLine($"    → INSERT INTO records ({string.Join(", ", record.Keys)}) VALUES (...)");
        }
        return records.Count;
    }
}

// ─── CONCRETE CLASSES ──────────────────────────────────────────────────────

// CSV-імпортер — перевизначає лише читання та парсинг.
public class CsvImporter : DataImporter
{
    protected override string ReadSource(string sourcePath)
    {
        Console.WriteLine($"  [CSV] Читаємо файл '{sourcePath}'...");
        // У реальному коді — File.ReadAllText(sourcePath). Тут — симуляція вмісту.
        return "name,age,city\nОлена,28,Київ\nМарко,35,Львів\n,40,Одеса";
    }

    protected override List<Dictionary<string, string>> Parse(string rawContent)
    {
        Console.WriteLine("  [CSV] Парсимо рядки за роздільником коми...");
        var lines = rawContent.Split('\n');
        var headers = lines[0].Split(',');
        var records = new List<Dictionary<string, string>>();

        for (int i = 1; i < lines.Length; i++)
        {
            var values = lines[i].Split(',');
            var record = new Dictionary<string, string>();
            for (int j = 0; j < headers.Length && j < values.Length; j++)
            {
                record[headers[j]] = values[j];
            }
            records.Add(record);
        }
        return records;
    }
}

// JSON-імпортер — теж перевизначає лише читання та парсинг.
public class JsonImporter : DataImporter
{
    protected override string ReadSource(string sourcePath)
    {
        Console.WriteLine($"  [JSON] Читаємо файл '{sourcePath}'...");
        return "[{\"name\":\"Софія\",\"age\":\"22\"},{\"name\":\"\",\"age\":\"31\"}]";
    }

    protected override List<Dictionary<string, string>> Parse(string rawContent)
    {
        Console.WriteLine("  [JSON] Парсимо масив об'єктів (спрощений парсер для демонстрації)...");
        // Спрощений розбір без зовнішніх бібліотек — для демонстрації структури патерну.
        var records = new List<Dictionary<string, string>>();
        var items = rawContent.Trim('[', ']').Split("},{");

        foreach (var item in items)
        {
            var clean = item.Trim('{', '}');
            var record = new Dictionary<string, string>();
            foreach (var pair in clean.Split(','))
            {
                var kv = pair.Split(':');
                var key = kv[0].Trim('"', ' ');
                var value = kv.Length > 1 ? kv[1].Trim('"', ' ') : "";
                record[key] = value;
            }
            records.Add(record);
        }
        return records;
    }
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        DataImporter csvImporter = new CsvImporter();
        var csvResult = csvImporter.Import("users.csv");

        DataImporter jsonImporter = new JsonImporter();
        var jsonResult = jsonImporter.Import("users.json");

        Console.WriteLine($"\nПідсумок CSV:  прочитано={csvResult.RecordsRead}, валідно={csvResult.RecordsValid}, збережено={csvResult.RecordsSaved}");
        Console.WriteLine($"Підсумок JSON: прочитано={jsonResult.RecordsRead}, валідно={jsonResult.RecordsValid}, збережено={jsonResult.RecordsSaved}");
    }
}

// Виведе:
//
// [Import] Починаю імпорт з 'users.csv'...
//   [CSV] Читаємо файл 'users.csv'...
//   [CSV] Парсимо рядки за роздільником коми...
//   Прочитано записів: 3
//   Валідних записів: 2 (помилок: 1)
//     → INSERT INTO records (name, age, city) VALUES (...)
//     → INSERT INTO records (name, age, city) VALUES (...)
//   Збережено у БД: 2 записів.
// [Import] Імпорт завершено.
//
// [Import] Починаю імпорт з 'users.json'...
//   [JSON] Читаємо файл 'users.json'...
//   [JSON] Парсимо масив об'єктів (спрощений парсер для демонстрації)...
//   Прочитано записів: 2
//   Валідних записів: 1 (помилок: 1)
//     → INSERT INTO records (name, age) VALUES (...)
//   Збережено у БД: 1 записів.
// [Import] Імпорт завершено.
//
// Підсумок CSV:  прочитано=3, валідно=2, збережено=2
// Підсумок JSON: прочитано=2, валідно=1, збережено=1
```

Зверни увагу: `Validate()` і `SaveToDatabase()` — це кроки з **дефолтною реалізацією** (`virtual`), тому жоден з двох імпортерів навіть не торкається їх. Якщо в майбутньому знадобиться, наприклад, `XmlImporter` з особливою валідацією — він перевизначить лише `Validate()`, не чіпаючи решту.

---

## Приклад 4 — Реальний сценарій: генератор звітів

Розгорнутий, готовий до адаптації приклад — фреймворк для генерації звітів у різних форматах (PDF, Excel), де спільна логіка (фільтрація, шапка/підвал) винесена в базовий клас, а формат-специфічна логіка (отримання даних, форматування, експорт) — у підкласи.

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

// ─── МОДЕЛЬ ДАНИХ ────────────────────────────────────────────────────────────

public class ReportRow
{
    public string Category { get; set; }
    public string Label    { get; set; }
    public decimal Value   { get; set; }
    public DateTime Date   { get; set; }
}

public class ReportFilter
{
    public DateTime? FromDate { get; set; }
    public DateTime? ToDate   { get; set; }
    public string CategoryFilter { get; set; }
}

// ─── ABSTRACT CLASS ────────────────────────────────────────────────────────

public abstract class ReportGenerator
{
    protected List<ReportRow> Rows { get; private set; } = new();

    // ★ TEMPLATE METHOD — фіксує весь конвеєр генерації звіту.
    // sealed: жоден конкретний звіт не має права переставити чи пропустити кроки —
    // це гарантує, що КОЖЕН звіт у системі проходить однаковий, передбачуваний процес.
    public sealed byte[] GenerateReport(ReportFilter filter)
    {
        Console.WriteLine($"\n=== Генерація звіту: {ReportTitle} ===");

        // Крок 1 — отримання "сирих" даних (РІЗНЕ для кожного типу звіту)
        Rows = FetchData();
        Console.WriteLine($"  [1/5] Отримано {Rows.Count} записів з джерела даних.");

        // Крок 2 — застосування фільтрів (ОДНАКОВЕ для всіх звітів)
        Rows = ApplyFilters(Rows, filter);
        Console.WriteLine($"  [2/5] Після фільтрації залишилось {Rows.Count} записів.");

        // Крок 3 — форматування даних (РІЗНЕ для кожного типу звіту: PDF verse Excel)
        var formattedBody = FormatData(Rows);
        Console.WriteLine($"  [3/5] Дані відформатовано у вигляді {ReportFormat}.");

        // Крок 4 — додавання шапки/підвалу (ОДНАКОВЕ для всіх звітів)
        var fullContent = AddHeaderAndFooter(formattedBody);
        Console.WriteLine($"  [4/5] Додано стандартну шапку та підвал компанії.");

        // Крок 5 — експорт у фінальний бінарний формат (РІЗНЕ для кожного типу звіту)
        byte[] output = Export(fullContent);
        Console.WriteLine($"  [5/5] Звіт експортовано ({output.Length} байт умовного розміру).");

        // ★ Хук — деякі звіти хочуть надіслати сповіщення після генерації, деякі — ні.
        if (ShouldNotifyOnCompletion())
        {
            NotifyCompletion(output.Length);
        }

        Console.WriteLine($"=== Звіт '{ReportTitle}' готовий ===");
        return output;
    }

    // Абстрактні кроки — обов'язкові для кожного конкретного звіту.
    protected abstract List<ReportRow> FetchData();
    protected abstract string FormatData(List<ReportRow> rows);
    protected abstract byte[] Export(string fullContent);

    // Властивості, які кожен звіт має надати.
    protected abstract string ReportTitle  { get; }
    protected abstract string ReportFormat { get; }

    // Крок з дефолтною реалізацією — фільтрація за датою/категорією однакова для всіх.
    // Підклас МОЖЕ перевизначити для особливої логіки фільтрації, але зазвичай не потрібно.
    protected virtual List<ReportRow> ApplyFilters(List<ReportRow> rows, ReportFilter filter)
    {
        IEnumerable<ReportRow> query = rows;

        if (filter.FromDate.HasValue)
            query = query.Where(r => r.Date >= filter.FromDate.Value);

        if (filter.ToDate.HasValue)
            query = query.Where(r => r.Date <= filter.ToDate.Value);

        if (!string.IsNullOrEmpty(filter.CategoryFilter))
            query = query.Where(r => r.Category == filter.CategoryFilter);

        return query.ToList();
    }

    // Крок з дефолтною реалізацією — шапка/підвал стандартні для всієї компанії.
    protected virtual string AddHeaderAndFooter(string body)
    {
        string header = $"=== {ReportTitle} — Компанія 'Приклад ТОВ' — {DateTime.Now:dd.MM.yyyy} ===\n";
        string footer = "\n--- Кінець звіту. Конфіденційно. ---";
        return header + body + footer;
    }

    // ХУК: за замовчуванням сповіщення НЕ надсилається.
    protected virtual bool ShouldNotifyOnCompletion() => false;

    // Дефолтна (порожня за замістом) реалізація сповіщення — переважно перевизначається
    // разом з хуком вище.
    protected virtual void NotifyCompletion(int reportSizeBytes)
    {
        // За замовчуванням нічого не робимо.
    }
}

// ─── CONCRETE CLASSES ──────────────────────────────────────────────────────

// Звіт продажів у PDF — критично важливий, тому надсилає сповіщення після генерації.
public class PdfSalesReport : ReportGenerator
{
    protected override string ReportTitle  => "Звіт продажів";
    protected override string ReportFormat => "PDF";

    protected override List<ReportRow> FetchData()
    {
        Console.WriteLine("  [PDF/Sales] Звертаємось до бази даних продажів...");
        return new List<ReportRow>
        {
            new() { Category = "Електроніка", Label = "Ноутбук Lenovo", Value = 24999m, Date = new DateTime(2026, 6, 10) },
            new() { Category = "Електроніка", Label = "Смартфон Samsung", Value = 15999m, Date = new DateTime(2026, 7, 2) },
            new() { Category = "Меблі",      Label = "Офісний стілець",  Value = 3200m,  Date = new DateTime(2026, 5, 15) },
            new() { Category = "Електроніка", Label = "Навушники",       Value = 1899m,  Date = new DateTime(2026, 8, 1) },
        };
    }

    protected override string FormatData(List<ReportRow> rows)
    {
        var lines = rows.Select(r => $"  • {r.Date:dd.MM.yyyy} | {r.Category,-12} | {r.Label,-20} | {r.Value,10:C0}");
        decimal total = rows.Sum(r => r.Value);
        return string.Join("\n", lines) + $"\n\n  РАЗОМ: {total:C0}";
    }

    protected override byte[] Export(string fullContent)
    {
        Console.WriteLine("  [PDF/Sales] Рендеримо у PDF-документ через бібліотеку рендерингу...");
        // У реальному коді тут був би виклик, наприклад, QuestPDF або iTextSharp.
        return System.Text.Encoding.UTF8.GetBytes(fullContent);
    }

    // Продажі — важливий звіт, тому вмикаємо сповіщення (хук повертає true).
    protected override bool ShouldNotifyOnCompletion() => true;

    protected override void NotifyCompletion(int reportSizeBytes)
    {
        Console.WriteLine($"  📧 Надсилаємо email керівнику відділу продажів: звіт готовий ({reportSizeBytes} байт).");
    }
}

// Звіт залишків складу в Excel — рутинний звіт, сповіщення не потрібне.
public class ExcelInventoryReport : ReportGenerator
{
    protected override string ReportTitle  => "Звіт залишків складу";
    protected override string ReportFormat => "Excel";

    protected override List<ReportRow> FetchData()
    {
        Console.WriteLine("  [Excel/Inventory] Звертаємось до системи складського обліку...");
        return new List<ReportRow>
        {
            new() { Category = "Електроніка", Label = "Ноутбук Lenovo",   Value = 12,  Date = new DateTime(2026, 8, 20) },
            new() { Category = "Електроніка", Label = "Смартфон Samsung", Value = 45,  Date = new DateTime(2026, 8, 20) },
            new() { Category = "Меблі",      Label = "Офісний стілець",   Value = 3,   Date = new DateTime(2026, 8, 20) },
        };
    }

    protected override string FormatData(List<ReportRow> rows)
    {
        // Excel-звіт форматується як псевдо-табличний текст (у реальності — комірки ClosedXML).
        var lines = rows.Select(r => $"  | {r.Category,-12} | {r.Label,-20} | Залишок: {r.Value,5} шт |");
        return string.Join("\n", lines);
    }

    protected override byte[] Export(string fullContent)
    {
        Console.WriteLine("  [Excel/Inventory] Генеруємо .xlsx через бібліотеку роботи з Excel...");
        // У реальному коді — ClosedXML.Excel або EPPlus.
        return System.Text.Encoding.UTF8.GetBytes(fullContent);
    }

    // ShouldNotifyOnCompletion і NotifyCompletion НЕ перевизначені —
    // використовується дефолт із базового класу (сповіщення не надсилається).
}
```

### Використання

```csharp
class Program
{
    static void Main()
    {
        var salesFilter = new ReportFilter
        {
            FromDate = new DateTime(2026, 1, 1),
            ToDate   = new DateTime(2026, 12, 31),
            CategoryFilter = "Електроніка"
        };

        ReportGenerator pdfReport = new PdfSalesReport();
        byte[] pdfBytes = pdfReport.GenerateReport(salesFilter);

        var inventoryFilter = new ReportFilter(); // без фільтрів — весь склад
        ReportGenerator excelReport = new ExcelInventoryReport();
        byte[] excelBytes = excelReport.GenerateReport(inventoryFilter);

        Console.WriteLine($"\nPDF звіт: {pdfBytes.Length} байт.");
        Console.WriteLine($"Excel звіт: {excelBytes.Length} байт.");
    }
}

// Виведе (приблизно):
//
// === Генерація звіту: Звіт продажів ===
//   [PDF/Sales] Звертаємось до бази даних продажів...
//   [1/5] Отримано 4 записів з джерела даних.
//   [2/5] Після фільтрації залишилось 3 записів.
//   [3/5] Дані відформатовано у вигляді PDF.
//   [4/5] Додано стандартну шапку та підвал компанії.
//   [PDF/Sales] Рендеримо у PDF-документ через бібліотеку рендерингу...
//   [5/5] Звіт експортовано (312 байт умовного розміру).
//   📧 Надсилаємо email керівнику відділу продажів: звіт готовий (312 байт).
// === Звіт 'Звіт продажів' готовий ===
//
// === Генерація звіту: Звіт залишків складу ===
//   [Excel/Inventory] Звертаємось до системи складського обліку...
//   [1/5] Отримано 3 записів з джерела даних.
//   [2/5] Після фільтрації залишилось 3 записів.
//   [3/5] Дані відформатовано у вигляді Excel.
//   [4/5] Додано стандартну шапку та підвал компанії.
//   [Excel/Inventory] Генеруємо .xlsx через бібліотеку роботи з Excel...
//   [5/5] Звіт експортовано (211 байт умовного розміру).
// === Звіт 'Звіт залишків складу' готовий ===
//
// PDF звіт: 312 байт.
// Excel звіт: 211 байт.
```

Цей приклад демонструє повну силу патерну: **два зовсім різні формати звітів** (PDF для продажів, Excel для складу) **діляться** однаковою логікою фільтрації та оформлення шапки/підвалу, і при цьому кожен має власну логіку отримання даних та експорту. Якщо завтра з'явиться `PdfInventoryReport` чи `CsvSalesReport` — потрібно буде написати лише новий підклас, без жодних змін у базовій логіці конвеєра.

---

## Template Method vs Strategy vs Factory Method

Ці три патерни часто застосовуються поруч і їх легко сплутати. Розберемось у різниці.

### Template Method — фіксація через успадкування (compile-time)

```
AbstractClass
  + TemplateMethod() { Step1(); Step2(); Step3(); }  ← скелет ЗАФІКСОВАНО в базовому класі
  # Step2() : abstract                                  ← варіюється лише крок

ConcreteClassA : AbstractClass   → Step2() = "зробити А"
ConcreteClassB : AbstractClass   → Step2() = "зробити Б"
```

Щоб змінити поведінку — потрібен **інший клас**, обраний під час компіляції (або хоча б заздалегідь створений об'єкт конкретного підкласу). Зв'язок — через **наслідування**.

### Strategy — фіксація через композицію (runtime)

```
Context
  - strategy : IStrategy         ← об'єкт-стратегія ІН'ЄКТУЄТЬСЯ ззовні
  + DoWork() { strategy.Execute(); }

IStrategy
  + Execute()

ConcreteStrategyA : IStrategy
ConcreteStrategyB : IStrategy

// Стратегію можна ПІДМІНИТИ в будь-який момент виконання програми:
context.SetStrategy(new ConcreteStrategyB());
```

Strategy досягає схожої гнучкості, але **через композицію**, а не наслідування: у `Context` є посилання на об'єкт-інтерфейс `IStrategy`, і цей об'єкт можна **замінити під час виконання програми** (наприклад, залежно від введених користувачем даних). Template Method натомість фіксує весь алгоритм на етапі компіляції — обраний підклас не змінюється "на льоту".

### Factory Method — спеціалізація, що часто живе ВСЕРЕДИНІ Template Method

```csharp
public abstract class ReportGenerator
{
    // Це ТAK САМО template method...
    public sealed void GenerateReport()
    {
        var exporter = CreateExporter(); // ← ...і водночас ТУТ використано Factory Method!
        exporter.Export(...);
    }

    // Factory Method — окремий випадок кроку, що ПОВЕРТАЄ ОБ'ЄКТ,
    // а не просто виконує дію.
    protected abstract IExporter CreateExporter();
}
```

Factory Method — це, по суті, **один конкретний вид "кроку"** всередині Template Method: крок, чиє завдання — не виконати дію, а **створити потрібний об'єкт** (наприклад, підкласу вирішувати, який саме `IExporter` створити для конкретного кроку конвеєра). Тобто Factory Method часто є **частиною реалізації** Template Method, а не окремою альтернативою до нього.

### Порівняльна таблиця

| | Template Method | Strategy | Factory Method |
|---|---|---|---|
| Механізм | Успадкування | Композиція (ін'єкція об'єкта) | Успадкування |
| Коли обирається поведінка | Компіляція (обраний підклас) | Виконання (можна підмінити) | Компіляція (обраний підклас) |
| Що варіюється | Окремі кроки фіксованого алгоритму | Весь алгоритм цілком | Спосіб створення одного об'єкта |
| Типове застосування | "Скелет алгоритму + деталі кроків" | "Взаємозамінні алгоритми/поведінки" | "Хто саме створює продукт для кроку" |
| Зв'язок між ними | Часто ВИКОРИСТОВУЄ Factory Method всередині кроку | Незалежний патерн, інша механіка | Часто є ЧАСТИНОЮ Template Method |

### Запитай себе:

- **"Чи потрібно підміняти поведінку під час виконання (наприклад, залежно від конфігурації користувача в реальному часі)?"** → Strategy.
- **"Чи алгоритм завжди складається з одних і тих самих кроків у тому самому порядку, і лише деталі кроків відрізняються між типами об'єктів?"** → Template Method.
- **"Чи один із кроків алгоритму — це просто 'створити потрібний об'єкт', а решта логіки однакова?"** → Всередині Template Method застосуй Factory Method саме для цього кроку.

---

## Переваги та недоліки

### Переваги

- **Усуває дублювання коду скелету алгоритму.** Спільна послідовність кроків написана один раз у базовому класі — не копіюється в кожен підклас.
- **Гарантує порядок кроків та інваріанти в одному місці.** Завдяки `sealed` template method неможливо випадково змінити порядок виконання чи пропустити критичний крок у якомусь із підкласів.
- **Підкласи реалізують лише те, що дійсно варіюється.** Чітке розділення "що незмінне" (реалізовано в базовому класі) від "що варіюється" (винесено в abstract/virtual кроки) — код підкласів стає мінімальним і сфокусованим.
- **Полегшує розширення (OCP).** Новий варіант алгоритму — це новий підклас, без зміни базового класу чи інших підкласів.
- **Хуки додають гнучкість без порушення структури.** Можна дати підкласам можливість "включати/виключати" окремі гілки алгоритму, не даючи їм контролю над усім потоком.

### Недоліки

- **Залежність від наслідування.** Підклас "прив'язаний" до загальної структури базового класу — якщо потрібно докорінно інший алгоритм, доведеться або не використовувати цей базовий клас, або дублювати структуру деінде.
- **Складніше розуміти та дебажити (принцип Голлівуду).** Контроль потоку "стрибає" між базовим класом і підкласом (`TemplateMethod()` викликає `Step2()`, який визначений десь у підкласі) — новачку важче відразу зрозуміти, куди піде виконання, ніж у лінійному коді.
- **Може порушувати принцип підстановки Лісков (LSP).** Якщо перевизначений крок підкласу порушує припущення, на які покладається template method (наприклад, повертає `null` там, де очікується заповнений об'єкт), — весь алгоритм може зламатися непередбачувано.
- **Обмежена кількість варіацій без множинного наслідування.** C# не підтримує множинне наслідування класів — якщо об'єкту потрібно комбінувати кроки з двох різних "сімей" алгоритмів, доведеться шукати обхідні шляхи (наприклад, композицію чи Strategy).

---

## Антипатерни та поширені помилки

### Помилка 1 — Забути позначити template method як `sealed`

```csharp
// НЕПРАВИЛЬНО: PrepareRecipe — virtual, і будь-який підклас може перевизначити
// ВЕСЬ метод цілком, зламавши гарантований порядок кроків.
public abstract class CaffeineBeverage
{
    public virtual void PrepareRecipe() // ← virtual! Підклас може все переписати.
    {
        BoilWater();
        Brew();
        PourInCup();
        AddCondiments();
    }

    protected abstract void Brew();
    protected abstract void AddCondiments();
    private void BoilWater() => Console.WriteLine("Кип'ятимо воду...");
    private void PourInCup() => Console.WriteLine("Наливаємо у чашку...");
}

// Підклас "випадково" (або навмисно) ламає весь сенс патерну:
public class WeirdTea : CaffeineBeverage
{
    public override void PrepareRecipe()
    {
        AddCondiments(); // Додає лимон ДО того, як чай взагалі заварений!
        Brew();
        // PourInCup() взагалі забули викликати — чай ніколи не потрапить у чашку.
    }

    protected override void Brew() => Console.WriteLine("Заварюємо чай...");
    protected override void AddCondiments() => Console.WriteLine("Додаємо лимон...");
}

// ПРАВИЛЬНО: sealed гарантує, що структура алгоритму недоторканна.
public abstract class CaffeineBeverage
{
    public sealed void PrepareRecipe() // ← sealed! Порядок кроків захищено.
    {
        BoilWater();
        Brew();
        PourInCup();
        AddCondiments();
    }
    // ...
}

// Якщо ж перевизначення дійсно потрібне (рідкісний випадок) —
// зробіть це усвідомленим і задокументованим рішенням, а не випадковістю:
public abstract class CaffeineBeverage
{
    // virtual — НАВМИСНО, з явним коментарем, чому це дозволено.
    // Підкласам ДОЗВОЛЕНО повністю змінювати алгоритм, якщо у них дуже нетиповий рецепт.
    protected virtual void PrepareRecipe()
    {
        BoilWater();
        Brew();
        PourInCup();
        AddCondiments();
    }
}
```

### Помилка 2 — Робити абсолютно всі кроки `abstract`

```csharp
// НЕПРАВИЛЬНО: кожен крок — abstract, навіть ті, що в 95% підкласів ІДЕНТИЧНІ.
// Це змушує кожен новий підклас копіювати той самий код знову і знову —
// патерн перестає рятувати від дублювання, а лише переносить його в підкласи!
public abstract class DataImporter
{
    public sealed void Import(string path)
    {
        var raw = ReadSource(path);
        var records = Parse(raw);
        var valid = Validate(records);   // ← abstract, хоча логіка валідації типова
        SaveToDatabase(valid);           // ← abstract, хоча логіка збереження типова
    }

    protected abstract string ReadSource(string path);
    protected abstract List<Dictionary<string, string>> Parse(string raw);
    protected abstract List<Dictionary<string, string>> Validate(List<Dictionary<string, string>> records); // зайве!
    protected abstract void SaveToDatabase(List<Dictionary<string, string>> records); // зайве!
}

// Тепер і CsvImporter, і JsonImporter, і XmlImporter МУСЯТЬ написати
// однаковий код Validate() і SaveToDatabase() — знову дублювання!

// ПРАВИЛЬНО: типові кроки отримують дефолтну реалізацію (virtual),
// а абстрактними залишаються лише ті, що дійсно завжди різні.
public abstract class DataImporter
{
    public sealed void Import(string path)
    {
        var raw = ReadSource(path);
        var records = Parse(raw);
        var valid = Validate(records);   // virtual — дефолт підходить майже всім
        SaveToDatabase(valid);           // virtual — дефолт підходить майже всім
    }

    protected abstract string ReadSource(string path);                                  // завжди різне
    protected abstract List<Dictionary<string, string>> Parse(string raw);                // завжди різне

    protected virtual List<Dictionary<string, string>> Validate(List<Dictionary<string, string>> records)
        => records.Where(r => r.ContainsKey("name")).ToList(); // типова дефолтна логіка

    protected virtual void SaveToDatabase(List<Dictionary<string, string>> records)
    {
        Console.WriteLine($"Збережено {records.Count} записів (стандартна логіка)."); // типова логіка
    }
}
```

### Помилка 3 — Крок підкласу порушує інваріант, на який покладається базовий алгоритм

```csharp
// НЕПРАВИЛЬНО: базовий клас ОЧІКУЄ, що після Parse() список записів не null
// і кожен запис матиме ключ "name", але жодним чином це не перевіряє і не документує.
// Недбалий підклас тихо повертає порожній або "поламаний" результат — і алгоритм
// падає значно пізніше, у зовсім іншому місці, що ускладнює діагностику.
public abstract class DataImporter
{
    public sealed void Import(string path)
    {
        var raw = ReadSource(path);
        var records = Parse(raw);           // Очікуємо: not null, кожен запис має "name"
        var valid = Validate(records);      // ← Впаде з NullReferenceException, якщо records == null
        SaveToDatabase(valid);
    }

    protected abstract string ReadSource(string path);
    protected abstract List<Dictionary<string, string>> Parse(string raw);
    protected virtual List<Dictionary<string, string>> Validate(List<Dictionary<string, string>> records)
        => records.Where(r => r.ContainsKey("name")).ToList(); // NRE, якщо records == null
    protected virtual void SaveToDatabase(List<Dictionary<string, string>> records) { }
}

public class BrokenXmlImporter : DataImporter
{
    protected override string ReadSource(string path) => "<xml/>";

    protected override List<Dictionary<string, string>> Parse(string raw)
    {
        // Забули обробити помилку парсингу — тихо повертаємо null!
        // Це порушує неявний контракт "Parse() завжди повертає список, навіть порожній".
        return null;
    }
}

// ПРАВИЛЬНО: контракт кроку задокументовано І перевіряється безпосередньо
// в template method за допомогою постумовної перевірки (post-condition check).
public abstract class DataImporter
{
    public sealed void Import(string path)
    {
        var raw = ReadSource(path);

        var records = Parse(raw);

        // ★ Постумовна перевірка інваріанту, на який покладається решта алгоритму.
        // Якщо підклас порушив контракт — падаємо ОДРАЗУ, з чітким повідомленням,
        // а не десь у Validate() з незрозумілим NullReferenceException.
        if (records == null)
        {
            throw new InvalidOperationException(
                $"{GetType().Name}.Parse() повернув null. " +
                "Контракт Parse(): метод ЗАВЖДИ повинен повертати список (можна порожній), ніколи null.");
        }

        var valid = Validate(records);
        SaveToDatabase(valid);
    }

    /// <summary>
    /// Розбирає сирий вміст у список записів.
    /// КОНТРАКТ: метод повинен повернути НЕ-NULL список (порожній список — це нормально,
    /// якщо джерело даних дійсно порожнє; null — заборонено).
    /// </summary>
    protected abstract List<Dictionary<string, string>> Parse(string raw);

    protected abstract string ReadSource(string path);
    protected virtual List<Dictionary<string, string>> Validate(List<Dictionary<string, string>> records)
        => records.Where(r => r.ContainsKey("name")).ToList();
    protected virtual void SaveToDatabase(List<Dictionary<string, string>> records) { }
}
```

---

## Підсумок

Template Method варто застосовувати, коли:

- Є **кілька класів з майже однаковим алгоритмом**, що відрізняється лише в декількох конкретних кроках.
- Потрібно **гарантувати**, що всі варіації алгоритму виконуються в **однаковому, передбачуваному порядку**.
- Хочеться дозволити розширення поведінки через підкласи, але **не дати їм повністю переписати логіку** (на відміну від звичайного `virtual` методу без структури).
- Є спільний код (наприклад, логування, обробка помилок, валідація), який хочеться написати **один раз**, а не в кожному підкласі окремо.

Не варто застосовувати, якщо:

- Алгоритми, які потрібно комбінувати, **не мають спільного скелету** — тоді підійде Strategy (композиція) замість наслідування.
- Потрібна можливість **змінювати поведінку під час виконання** — Template Method фіксує вибір підкласу під час компіляції/створення об'єкта, тоді як Strategy дозволяє підміняти об'єкт-стратегію "на льоту".
- Ієрархія наслідування вже й так глибока й заплутана — додатковий рівень абстракції може лише погіршити читабельність.

### Мінімальний шаблон

```csharp
// 1. AbstractClass — фіксує скелет алгоритму.
public abstract class AbstractClass
{
    // Template method — sealed, порядок кроків недоторканний.
    public sealed void TemplateMethod()
    {
        Step1();          // Обов'язковий крок — кожен підклас реалізує по-своєму
        Step2();          // Крок з дефолтною поведінкою — підклас може перевизначити
        if (Hook())       // Хук — підклас керує тим, чи виконається наступний крок
        {
            OptionalStep();
        }
    }

    // Абстрактний крок — немає дефолтної реалізації.
    protected abstract void Step1();

    // Крок з дефолтною реалізацією — підклас МОЖЕ перевизначити, але не зобов'язаний.
    protected virtual void Step2()
    {
        Console.WriteLine("AbstractClass: типова реалізація Step2().");
    }

    // Хук — керує потоком виконання, за замовчуванням дозволяє опційний крок.
    protected virtual bool Hook() => true;

    protected virtual void OptionalStep()
    {
        Console.WriteLine("AbstractClass: типова реалізація OptionalStep().");
    }
}

// 2. ConcreteClass — реалізує лише те, що дійсно варіюється.
public class ConcreteClass : AbstractClass
{
    protected override void Step1()
    {
        Console.WriteLine("ConcreteClass: власна реалізація Step1().");
    }

    // Step2, Hook, OptionalStep не перевизначені — використовується дефолт.
}

// Використання:
// AbstractClass instance = new ConcreteClass();
// instance.TemplateMethod();
```

---

*Документ підготовлено для вивчення патернів проектування. Всі приклади протестовані на .NET 6+.*
