# SQL — Мова запитів від основ до Senior-рівня

> **Категорія:** Бази даних / Реляційні (Relational)  
> **Рівень:** Junior → Senior  
> **Мова прикладів:** SQL + C# (.NET, Dapper/EF Core)

---

## Зміст

1. [Наскрізна схема даних для прикладів](#наскрізна-схема-даних-для-прикладів)
2. [Вступ: декларативна природа SQL](#вступ-декларативна-природа-sql)
3. [Основи: SELECT, WHERE, ORDER BY, GROUP BY, HAVING](#основи-select-where-order-by-group-by-having)
4. [JOIN-и — детальний розбір усіх типів](#join-и--детальний-розбір-усіх-типів)
   - [INNER JOIN](#inner-join)
   - [LEFT (OUTER) JOIN](#left-outer-join)
   - [RIGHT (OUTER) JOIN](#right-outer-join)
   - [FULL OUTER JOIN](#full-outer-join)
   - [CROSS JOIN](#cross-join)
   - [Self-JOIN](#self-join)
5. [Підзапити (Subqueries)](#підзапити-subqueries)
6. [CTE (Common Table Expressions) та рекурсивні CTE](#cte-common-table-expressions-та-рекурсивні-cte)
7. [Віконні функції (Window Functions)](#віконні-функції-window-functions)
8. [Множинні операції: UNION, UNION ALL, INTERSECT, EXCEPT](#множинні-операції-union-union-all-intersect-except)
9. [Практичні приклади на C#](#практичні-приклади-на-c)
   - [Приклад 1: ADO.NET з параметризованим запитом](#приклад-1-adonet-з-параметризованим-запитом)
   - [Приклад 2: Dapper](#приклад-2-dapper)
   - [Приклад 3: EF Core (LINQ та SQL)](#приклад-3-ef-core-linq-та-sql)
10. [Реальний сценарій: аналітичний запит для адмінки інтернет-магазину](#реальний-сценарій-аналітичний-запит-для-адмінки-інтернет-магазину)
11. [Поширені помилки](#поширені-помилки)
12. [Питання для співбесіди за рівнями](#питання-для-співбесіди-за-рівнями)
13. [Підсумок](#підсумок)

---

## Наскрізна схема даних для прикладів

Щоб не вигадувати нову таблицю під кожен приклад (і щоб цей документ читався як одна суцільна історія, а не набір розрізнених фрагментів), **усі** запити нижче — від першого `SELECT` до найскладнішого звіту з CTE та віконними функціями — працюють проти однієї й тієї самої схеми інтернет-магазину.

```
┌────────────────┐        ┌────────────────┐        ┌──────────────────┐
│   Customers    │        │     Orders     │        │   OrderItems     │
├────────────────┤        ├────────────────┤        ├──────────────────┤
│ CustomerId  PK │◄──┐    │ OrderId     PK │◄──┐    │ OrderItemId  PK  │
│ FullName       │   └────│ CustomerId  FK │   └────│ OrderId      FK  │
│ Email          │        │ OrderDate      │        │ ProductId    FK ─┼──┐
│ Country        │        │ Status         │        │ Quantity         │  │
│ RegisteredAt   │        └────────────────┘        │ UnitPrice        │  │
└────────────────┘                                  └──────────────────┘  │
        ▲                                                                 │
        │                                                                 │
        │            ┌────────────────┐        ┌────────────────┐         │
        │            │   Categories   │        │    Products    │◄────────┘
        │            ├────────────────┤        ├────────────────┤
        │            │ CategoryId  PK │◄──┐    │ ProductId   PK │
        │            │ Name           │   └────│ CategoryId  FK │
        │            │ ParentCatId FK ┼───┘    │ Name           │
        │            └────────────────┘        │ Price          │
        │              (само-посилання:        │ CreatedAt      │
        │               ParentCategoryId →     └────────────────┘
        │               CategoryId — дерево            ▲
        │               категорій)                     │
        │                                       ┌────────────────┐
        └───────────────────────────────────────│    Reviews     │
                                                ├────────────────┤
                                                │ ReviewId    PK │
                                                │ ProductId   FK │
                                                │ CustomerId  FK │
                                                │ Rating (1-5)   │
                                                │ Comment        │
                                                │ CreatedAt      │
                                                └────────────────┘
```

**Зв'язки:**

| Таблиця A | Зв'язок | Таблиця B | Суть |
|---|---|---|---|
| `Customers` | 1 — * | `Orders` | один клієнт може мати багато замовлень |
| `Orders` | 1 — * | `OrderItems` | одне замовлення складається з кількох позицій |
| `Products` | 1 — * | `OrderItems` | один товар зустрічається в багатьох позиціях замовлень |
| `Categories` | 1 — * | `Products` | одна категорія містить багато товарів |
| `Categories` | 1 — * | `Categories` | **само-посилання** через `ParentCategoryId` — категорії утворюють дерево (наприклад, "Електроніка" → "Ноутбуки" → "Ігрові ноутбуки") |
| `Products` | 1 — * | `Reviews` | один товар може мати багато відгуків |
| `Customers` | 1 — * | `Reviews` | один клієнт може залишити багато відгуків |

**Про синтаксис у документі:** приклади написані у стилі, сумісному одночасно з **SQL Server** та **PostgreSQL** (наскільки це можливо) — це два найпоширеніші діалекти в .NET-екосистемі. Там, де діалекти суттєво розходяться, це явно позначено приміткою, наприклад:

- `SELECT TOP 10 ...` (SQL Server) проти `SELECT ... LIMIT 10` (PostgreSQL, MySQL);
- `GETDATE()` (SQL Server) проти `NOW()` / `CURRENT_TIMESTAMP` (PostgreSQL);
- `IDENTITY(1,1)` (SQL Server) проти `SERIAL` / `GENERATED ALWAYS AS IDENTITY` (PostgreSQL).

---

## Вступ: декларативна природа SQL

Перше й найважливіше, що варто зрозуміти про SQL — і що часто вислизає навіть від досвідчених розробників, які прийшли з C#/Java/Python — це те, що SQL є **декларативною** мовою, а не **імперативною**.

- **Імперативний код** (звичайний C#, наприклад) описує **ЯК** досягти результату: "візьми цей список, пройдись по ньому циклом, для кожного елемента перевір умову, якщо вона виконується — додай елемент до нового списку, потім відсортуй цей список...".
- **Декларативний SQL-запит** описує лише **ЩО** ти хочеш отримати: "дай мені імена клієнтів з України, відсортовані за датою реєстрації". Як саме база даних дістане ці рядки — через повне сканування таблиці, через індекс, у якому порядку об'єднає таблиці, чи буде використано хеш-з'єднання чи merge-з'єднання — це вже **не твоя турбота**. Це завдання компонента бази даних, який називається **оптимізатор запитів (query optimizer)**.

### Аналогія: замовлення страви в ресторані 🍽️

Коли ти замовляєш у ресторані "стейк середньої прожарки з картоплею фрі та соусом béarnaise", ти **не** кажеш кухарю: "візьми шматок м'яса вагою 250 грамів, розігрій сковорідку до 220°C, поклади м'ясо, почекай 3 хвилини, переверни, почекай ще 3 хвилини...". Ти просто описуєш **бажаний результат**, а кухар (= оптимізатор запитів) сам вирішує, яку сковорідку взяти, коли посолити і в якій послідовності готувати гарнір, щоб усе було готове одночасно.

SQL працює так само:

```sql
-- Це декларація бажаного результату, а не інструкція "як" його отримати
SELECT c.FullName, c.Country
FROM Customers c
WHERE c.Country = 'Ukraine'
ORDER BY c.RegisteredAt DESC;
```

Ти не вказуєш, чи буде рушій бази даних сканувати всю таблицю `Customers` послідовно, чи скористається індексом на колонці `Country`. Один і той самий запит на маленькій таблиці з 100 рядків та на таблиці з 500 мільйонів рядків **синтаксично виглядає однаково** — але оптимізатор під капотом обере кардинально різні фізичні плани виконання.

**Чому це важливо для Senior-рівня:** саме тому Senior-розробник не просто "вміє писати SQL", а розуміє, що відбувається **під капотом** — як індекси впливають на план виконання, чому один і той самий логічний результат можна отримати запитами з різною продуктивністю (наприклад, через `EXISTS` замість `IN`, або через віконну функцію замість корельованого підзапиту — про це докладно нижче). Декларативність не означає "продуктивність — не моя проблема"; вона означає, що ти впливаєш на продуктивність **опосередковано**: через структуру запиту, індекси та статистику, а не через ручний контроль циклів.

---

## Основи: SELECT, WHERE, ORDER BY, GROUP BY, HAVING

### SELECT — що показати

```sql
-- Погано в production-коді (детальніше — у розділі "Поширені помилки")
SELECT * FROM Products;

-- Добре: явний список колонок
SELECT ProductId, Name, Price, CategoryId
FROM Products;
```

### WHERE — фільтрація рядків

`WHERE` відкидає рядки **до** будь-якого групування — це працює на рівні окремого рядка вихідних таблиць.

```sql
SELECT ProductId, Name, Price
FROM Products
WHERE Price > 1000
  AND CategoryId = 3;

SELECT CustomerId, FullName, Country
FROM Customers
WHERE Country IN ('Ukraine', 'Poland', 'Germany');

SELECT Name, Price
FROM Products
WHERE Name LIKE 'iPhone%';       -- починається на "iPhone"

SELECT OrderId, Status
FROM Orders
WHERE Status IS NULL;             -- порівняння з NULL: лише IS NULL / IS NOT NULL, ніколи "= NULL"

SELECT ProductId, Name, Price
FROM Products
WHERE Price BETWEEN 500 AND 1500;
```

### ORDER BY — сортування результату

```sql
SELECT Name, Price
FROM Products
ORDER BY Price DESC, Name ASC;    -- спочатку за ціною спадаюче, потім за назвою за алфавітом
```

### GROUP BY та агрегатні функції

`GROUP BY` згортає багато рядків в один рядок на групу, а агрегатні функції (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`) обчислюють одне значення для кожної такої групи.

```sql
-- Скільки замовлень і на яку суму зробив кожен клієнт
SELECT
    o.CustomerId,
    COUNT(*)            AS OrdersCount,
    SUM(oi.Quantity * oi.UnitPrice) AS TotalSpent,
    AVG(oi.Quantity * oi.UnitPrice) AS AvgOrderItemValue,
    MIN(o.OrderDate)     AS FirstOrderDate,
    MAX(o.OrderDate)     AS LastOrderDate
FROM Orders o
JOIN OrderItems oi ON oi.OrderId = o.OrderId
GROUP BY o.CustomerId;
```

Кожен стовпець у `SELECT`, який **не** є агрегатною функцією, обов'язково має бути присутнім у `GROUP BY` (це правило суворо перевіряється в PostgreSQL; SQL Server теж вимагає цього, хоча повідомлення про помилку може відрізнятись).

### HAVING проти WHERE — класична пастка на співбесіді для Junior

Це одне з найпопулярніших питань на технічних співбесідах, і воно трапляється не тому, що це "хитрість", а тому що різниця дійсно фундаментальна для розуміння **порядку виконання** SQL-запиту.

- **`WHERE`** фільтрує **окремі рядки** вихідних таблиць **до** того, як відбулося групування. Він не має доступу до результатів агрегатних функцій, тому що на момент фільтрації в `WHERE` агрегати ще просто не обчислені.
- **`HAVING`** фільтрує вже **згруповані рядки** (групи), **після** того, як `GROUP BY` та агрегатні функції відпрацювали. Саме тому в `HAVING` можна використовувати `COUNT(*)`, `SUM(...)` тощо.

```sql
-- ❌ НЕПРАВИЛЬНО: помилка виконання!
-- "Invalid use of aggregate function" / "column must appear in GROUP BY or be used in an aggregate function"
SELECT CategoryId, COUNT(*) AS ProductsCount
FROM Products
WHERE COUNT(*) > 5          -- COUNT(*) ще не існує на етапі WHERE!
GROUP BY CategoryId;

-- ✅ ПРАВИЛЬНО: фільтрація груп через HAVING
SELECT CategoryId, COUNT(*) AS ProductsCount
FROM Products
GROUP BY CategoryId
HAVING COUNT(*) > 5;
```

Категорії, у яких понад 5 товарів середньою ціною вище 300:

```sql
SELECT
    CategoryId,
    COUNT(*)   AS ProductsCount,
    AVG(Price) AS AvgPrice
FROM Products
GROUP BY CategoryId
HAVING COUNT(*) > 5 AND AVG(Price) > 300;
```

`WHERE` і `HAVING` цілком можуть працювати разом в одному запиті — і це **найкраща практика**: спочатку відфільтрувати рядки якомога раніше (`WHERE`) — це зменшує обсяг даних, які взагалі потрапляють у групування, — а потім відфільтрувати вже готові групи (`HAVING`):

```sql
-- Категорії з понад 3 товарами, що коштують дорожче 100 (WHERE відкидає дешеві товари ЩЕ ДО групування)
SELECT CategoryId, COUNT(*) AS ExpensiveProductsCount
FROM Products
WHERE Price > 100
GROUP BY CategoryId
HAVING COUNT(*) > 3;
```

### 🔎 Логічний порядок виконання SQL-запиту

Розуміння цього порядку — фундамент для всього іншого в цьому документі (підзапитів, віконних функцій, аліасів):

| # | Етап | Приклад |
|---|---|---|
| 1 | `FROM` / `JOIN` | звідки беремо дані та як з'єднуємо таблиці |
| 2 | `WHERE` | фільтрація окремих рядків |
| 3 | `GROUP BY` | групування рядків |
| 4 | `HAVING` | фільтрація груп |
| 5 | `SELECT` (включно з віконними функціями) | обчислення колонок результату |
| 6 | `DISTINCT` | видалення дублікатів |
| 7 | `ORDER BY` | сортування результату |
| 8 | `OFFSET` / `FETCH` / `LIMIT` / `TOP` | обмеження кількості рядків |

Саме тому в `WHERE` не можна звертатись до аліасу, оголошеного в `SELECT` (`SELECT Price AS p FROM Products WHERE p > 100` — впаде в більшості діалектів), а в `ORDER BY` — можна: `ORDER BY` виконується **після** `SELECT`, тому аліас уже існує.

---

## JOIN-и — детальний розбір усіх типів

`JOIN` — це спосіб об'єднати рядки з двох (або більше) таблиць на основі умови зв'язку (найчастіше — рівність зовнішнього та первинного ключів). Нижче — усі типи `JOIN`, кожен із власною ASCII-ілюстрацією, прикладом на нашій схемі та поясненням, коли його застосовувати.

### INNER JOIN

```
   ┌─────────────┐     ┌─────────────┐
   │      A      │▓▓▓▓▓│      B      │
   │             │▓▓▓▓▓│             │
   └─────────────┘     └─────────────┘

   Результат = лише ▓▓▓ (рядки, для яких є відповідність в ОБОХ таблицях)
```

Найпоширеніший тип: повертає лише ті рядки, для яких є відповідність в обох таблицях.

```sql
-- Кожне замовлення разом з ім'ям клієнта, що його зробив
SELECT o.OrderId, o.OrderDate, c.FullName
FROM Orders o
INNER JOIN Customers c ON c.CustomerId = o.CustomerId;
```

**Коли використовувати:** коли тебе цікавлять лише рядки, що мають зв'язок з обох боків — наприклад, "усі товари, які хоча б раз замовляли" (товари без жодного замовлення тут просто не з'являться).

### LEFT (OUTER) JOIN

```
   ┌─────────────┐     ┌─────────────┐
   │▓▓▓▓▓▓▓▓▓▓▓▓▓│▓▓▓▓▓│             │
   │▓▓▓▓▓▓ A ▓▓▓▓│▓▓▓▓▓│      B      │
   └─────────────┘     └─────────────┘

   Результат = усі рядки A (▓) + ті рядки B, які мають відповідність (▓)
   Рядки B без відповідності в A — просто не увійдуть, а рядки A без
   відповідності в B отримають NULL замість колонок B.
```

Повертає **всі** рядки лівої таблиці, а для правої — відповідні дані, якщо вони є, або `NULL`, якщо немає.

```sql
-- Усі клієнти разом з їхніми замовленнями (якщо є)
SELECT c.CustomerId, c.FullName, o.OrderId, o.OrderDate
FROM Customers c
LEFT JOIN Orders o ON o.CustomerId = c.CustomerId;
```

### 🎯 Класичний патерн: "знайти рядки в A, для яких немає відповідності в B"

Це один із найпопулярніших практичних запитів на будь-якій співбесіді: **знайти клієнтів, які жодного разу нічого не замовили**.

```sql
SELECT c.CustomerId, c.FullName
FROM Customers c
LEFT JOIN Orders o ON o.CustomerId = c.CustomerId
WHERE o.OrderId IS NULL;   -- якщо для клієнта немає жодного замовлення,
                           -- усі колонки з Orders (включно з OrderId) будуть NULL
```

**Логіка:** `LEFT JOIN` спочатку повертає всіх клієнтів, доклеюючи до кожного всі його замовлення (якщо клієнт має 3 замовлення — буде 3 рядки; якщо жодного — 1 рядок з `NULL` у колонках `Orders`). Умова `WHERE o.OrderId IS NULL` відсіює саме той випадок, коли співпадіння не знайшлося взагалі — тобто саме клієнтів без жодного замовлення.

Цей самий патерн застосовується для будь-якої пари таблиць: "товари, які ніколи не купували", "категорії без жодного товару", "товари без жодного відгуку" і так далі:

```sql
-- Товари, які ще ніхто не купував
SELECT p.ProductId, p.Name
FROM Products p
LEFT JOIN OrderItems oi ON oi.ProductId = p.ProductId
WHERE oi.OrderItemId IS NULL;
```

### RIGHT (OUTER) JOIN

```
   ┌─────────────┐     ┌─────────────┐
   │             │▓▓▓▓▓│▓▓▓▓▓▓▓▓▓▓▓▓▓│
   │      A      │▓▓▓▓▓│▓▓▓▓▓▓ B ▓▓▓▓│
   └─────────────┘     └─────────────┘

   Дзеркальне відображення LEFT JOIN: усі рядки B + збіги з A.
```

`RIGHT JOIN` — це дзеркальне відображення `LEFT JOIN`: повертає всі рядки правої таблиці, а ліву доповнює `NULL`, де відповідності немає.

```sql
-- Той самий результат, що й попередній LEFT JOIN-приклад, але таблиці поміняні місцями
SELECT p.ProductId, p.Name
FROM OrderItems oi
RIGHT JOIN Products p ON p.ProductId = oi.ProductId
WHERE oi.OrderItemId IS NULL;
```

**Практична порада:** `RIGHT JOIN` синтаксично рівнозначний `LEFT JOIN` із переставленими таблицями. Багато команд та style-guide-ів (включно з реальними production-стандартами) **забороняють `RIGHT JOIN`** заради читабельності — легше сприймати запит, коли "головна" (та, з якої беруться всі рядки) таблиця завжди зазначена першою через `LEFT JOIN`, і не доводиться "перечитувати" запит справа наліво.

### FULL OUTER JOIN

```
   ┌─────────────┐     ┌─────────────┐
   │▓▓▓▓▓▓▓▓▓▓▓▓▓│▓▓▓▓▓│▓▓▓▓▓▓▓▓▓▓▓▓▓│
   │▓▓▓▓▓▓ A ▓▓▓▓│▓▓▓▓▓│▓▓▓▓▓▓ B ▓▓▓▓│
   └─────────────┘     └─────────────┘

   Результат = ВСЕ: рядки лише з A (з NULL у B), спільні рядки,
   і рядки лише з B (з NULL у A).
```

Повертає всі рядки з обох таблиць: там, де є відповідність — об'єднує їх; там, де немає — заповнює `NULL` з того боку, де відповідності бракує.

```sql
SELECT c.CustomerId, c.FullName, o.OrderId
FROM Customers c
FULL OUTER JOIN Orders o ON o.CustomerId = c.CustomerId;
```

**Коли використовувати:** для звірки (reconciliation) двох наборів даних, коли важливо побачити розбіжності з **обох** сторін одночасно — наприклад, "клієнти без замовлень" ТА "замовлення-сироти без валідного клієнта" в одному результаті.

**Діалектна примітка:** `FULL OUTER JOIN` підтримують SQL Server і PostgreSQL, але **MySQL (до версії 8.0.31) не підтримує** цей синтаксис напряму. Емуляція:

```sql
-- Емуляція FULL OUTER JOIN для діалектів без нативної підтримки
SELECT c.CustomerId, c.FullName, o.OrderId
FROM Customers c
LEFT JOIN Orders o ON o.CustomerId = c.CustomerId
UNION
SELECT c.CustomerId, c.FullName, o.OrderId
FROM Customers c
RIGHT JOIN Orders o ON o.CustomerId = c.CustomerId;
```

### CROSS JOIN

```
   A × B  (декартів добуток — КОЖЕН рядок A з'єднується з КОЖНИМ рядком B)

   A: {a1, a2}       B: {b1, b2, b3}

   Результат (2 × 3 = 6 рядків):
   (a1,b1) (a1,b2) (a1,b3)
   (a2,b1) (a2,b2) (a2,b3)
```

`CROSS JOIN` не має умови з'єднання — він повертає **декартів добуток**: кожен рядок першої таблиці комбінується з кожним рядком другої. N рядків × M рядків = N×M рядків результату.

```sql
-- Генеруємо "щільну" звітну сітку: усі категорії × усі місяці року,
-- навіть якщо в якомусь місяці по категорії не було жодного продажу (буде 0)
SELECT cat.Name AS CategoryName, m.MonthNumber
FROM Categories cat
CROSS JOIN (VALUES (1),(2),(3),(4),(5),(6),(7),(8),(9),(10),(11),(12)) AS m(MonthNumber);
```

**Коли використовувати:** генерація комбінаторних наборів для звітів (щоб у звіті не було "дірок" за місяці/категорії без даних), тестові дані, матриці "усі товари × усі знижки" тощо. **Обережно** на великих таблицях — `CROSS JOIN` двох таблиць по 100 000 рядків кожна дає 10 мільярдів рядків результату.

### Self-JOIN

Self-JOIN — це не окремий тип `JOIN` (технічно це звичайний `INNER`/`LEFT JOIN`), а прийом, коли таблиця з'єднується **сама з собою** через два різні аліаси. Найочевидніший кандидат у нашій схемі — `Categories`, яка має самопосилання через `ParentCategoryId`.

```sql
-- Кожна категорія разом з назвою її батьківської категорії
SELECT
    child.CategoryId,
    child.Name        AS CategoryName,
    parent.Name       AS ParentCategoryName
FROM Categories child
LEFT JOIN Categories parent ON child.ParentCategoryId = parent.CategoryId;
-- LEFT JOIN тут обов'язковий: у кореневих категорій ParentCategoryId = NULL,
-- і INNER JOIN просто відкинув би такі рядки
```

Ще один приклад — товари в тій самій категорії, що й заданий товар (найпростіша форма "схожі товари"):

```sql
SELECT p2.ProductId, p2.Name
FROM Products p1
JOIN Products p2 ON p2.CategoryId = p1.CategoryId AND p2.ProductId <> p1.ProductId
WHERE p1.ProductId = 42;
```

**Коли використовувати:** ієрархічні структури (категорії, організаційні структури "співробітник — керівник"), порівняння рядків однієї таблиці між собою (знайти дублікати, знайти пари, знайти "сусідні" за часом записи).

---

## Підзапити (Subqueries)

Підзапит (subquery) — це `SELECT`, вкладений усередину іншого запиту. Підзапити можуть з'являтися в `WHERE`, `SELECT`, `FROM` і навіть у `HAVING`.

### Скалярний підзапит (scalar subquery)

Повертає **рівно одне** значення (один рядок, одна колонка) — і тому може використовуватись будь-де, де очікується одне значення:

```sql
SELECT
    ProductId,
    Name,
    Price,
    (SELECT AVG(Price) FROM Products) AS AvgPriceAcrossAllProducts,
    Price - (SELECT AVG(Price) FROM Products) AS DiffFromAverage
FROM Products;
```

### Підзапити в WHERE: IN, EXISTS, NOT EXISTS

**`IN`** — перевіряє, чи належить значення до набору, поверненого підзапитом:

```sql
-- Клієнти, які зробили хоча б одне замовлення за останні 30 днів
SELECT CustomerId, FullName
FROM Customers
WHERE CustomerId IN (
    SELECT CustomerId
    FROM Orders
    WHERE OrderDate >= DATEADD(DAY, -30, GETDATE())   -- PostgreSQL: OrderDate >= NOW() - INTERVAL '30 days'
);
```

**`EXISTS`** — перевіряє лише **факт наявності** хоча б одного рядка, що задовольняє умову; сам вміст підзапиту не має значення (тому традиційно всередині пишуть `SELECT 1`):

```sql
SELECT c.CustomerId, c.FullName
FROM Customers c
WHERE EXISTS (
    SELECT 1
    FROM Orders o
    WHERE o.CustomerId = c.CustomerId
      AND o.OrderDate >= DATEADD(DAY, -30, GETDATE())
);
```

**`NOT EXISTS`** — зворотне до `EXISTS`, класична (і найбезпечніша) альтернатива патерну "знайти рядки без відповідності" з розділу про `LEFT JOIN`:

```sql
-- Клієнти без жодного замовлення (альтернатива LEFT JOIN ... WHERE ... IS NULL)
SELECT c.CustomerId, c.FullName
FROM Customers c
WHERE NOT EXISTS (
    SELECT 1 FROM Orders o WHERE o.CustomerId = c.CustomerId
);
```

### Корельовані проти некорельованих підзапитів

Це одне з ключових розрізнень, яке відрізняє Middle від Junior на співбесіді.

- **Некорельований (non-correlated) підзапит** — самодостатній, не посилається на жодну колонку зовнішнього запиту. Логічно (і, як правило, фізично) виконується **один раз**, а вже готовий результат використовується зовнішнім запитом. Приклад: `(SELECT AVG(Price) FROM Products)` вище — це число не залежить від того, який рядок зовнішнього запиту зараз обробляється.

- **Корельований (correlated) підзапит** — усередині нього є посилання на колонку зовнішнього запиту (у прикладі з `EXISTS` вище — `o.CustomerId = c.CustomerId`, де `c` — таблиця зовнішнього запиту). **Логічно** такий підзапит потрібно переобчислювати для **кожного** рядка зовнішньої таблиці, тому що результат залежить від поточного зовнішнього рядка.

**Про продуктивність:** із логічної точки зору корельований підзапит — це "вкладений цикл": для кожного рядка `Customers` рушій ніби заново виконує пошук по `Orders`. У реальності сучасні оптимізатори (SQL Server, PostgreSQL) часто **переписують** корельовані `EXISTS`/`IN` підзапити на ефективніші фізичні операції (semi-join, anti-join) з використанням індексів — тобто буквального "N окремих запитів" зазвичай не відбувається. Але це залежить від конкретного оптимізатора, версії СУБД, наявності індексів і складності умови — тому **розуміти** логічну модель "виконується на кожен зовнішній рядок" критично важливо, щоб передбачати, коли запит **може** стати повільним (особливо якщо в корельованому підзапиті немає індексу на колонці зв'язку).

### Підзапит у FROM (derived table / похідна таблиця)

```sql
-- Спочатку рахуємо загальну суму витрат кожного клієнта в похідній таблиці,
-- потім у зовнішньому запиті фільтруємо та приєднуємо додаткові дані
SELECT c.FullName, spending.TotalSpent
FROM Customers c
JOIN (
    SELECT o.CustomerId, SUM(oi.Quantity * oi.UnitPrice) AS TotalSpent
    FROM Orders o
    JOIN OrderItems oi ON oi.OrderId = o.OrderId
    GROUP BY o.CustomerId
) AS spending ON spending.CustomerId = c.CustomerId
WHERE spending.TotalSpent > 5000;
```

### Підзапит у SELECT

```sql
-- Для кожного клієнта — кількість його замовлень як окрема колонка
SELECT
    c.CustomerId,
    c.FullName,
    (SELECT COUNT(*) FROM Orders o WHERE o.CustomerId = c.CustomerId) AS OrdersCount
FROM Customers c;
```

### ⚠️ EXISTS проти IN: продуктивність та небезпека з NULL

На перший погляд `IN` і `EXISTS` виглядають взаємозамінними, але між ними є важлива різниця, яка на практиці стає **реальним джерелом багів**.

**Продуктивність:** `EXISTS` зупиняється, щойно знайшовши перший підходящий рядок (short-circuit) — йому не важливо, скільки всього збігів існує, важливий лише факт "є хоча б один". `IN` концептуально повинен матеріалізувати весь список значень підзапиту перед порівнянням. На великих підзапитах з дублікатами `EXISTS` зазвичай ефективніший (хоча сучасні оптимізатори часто зводять їх до одного й того самого плану виконання — знову ж, "розуміти логіку" важливіше за "заучити правило").

**Небезпека з NULL — критичний senior-gotcha:**

Уяви, що в `Orders` дозволені "гостьові" замовлення без прив'язки до клієнта, тобто `Orders.CustomerId` може бути `NULL`. Хочемо знайти клієнтів, які **не** мають жодного скасованого замовлення:

```sql
-- ❌ НЕБЕЗПЕЧНО: якщо підзапит поверне ХОЧА Б ОДИН NULL серед CustomerId,
-- цей запит поверне НУЛЬ РЯДКІВ ЗАГАЛОМ — навіть якщо насправді
-- десятки клієнтів підходять під умову!
SELECT CustomerId, FullName
FROM Customers
WHERE CustomerId NOT IN (
    SELECT CustomerId FROM Orders WHERE Status = 'Cancelled'
    -- якщо серед цих CustomerId є хоча б один NULL (гостьове замовлення) — біда
);
```

**Чому так відбувається:** SQL використовує трьохзначну логіку (`TRUE` / `FALSE` / `UNKNOWN`). Вираз `NOT IN (a, b, NULL)` розкривається як `x <> a AND x <> b AND x <> NULL`. А порівняння будь-чого з `NULL` завжди дає `UNKNOWN`, а не `TRUE` чи `FALSE`. Ланцюжок `AND`, у якому є хоча б один `UNKNOWN`, ніколи не може дати `TRUE` — і рядок відкидається. Це стається **для кожного рядка** зовнішнього запиту, тому результат — порожня множина, і жодної помилки при цьому не виникає (що робить баг особливо підступним — він мовчки "з'їдає" всі рядки).

```sql
-- ✅ БЕЗПЕЧНО, варіант 1: явно виключити NULL з підзапиту
SELECT CustomerId, FullName
FROM Customers
WHERE CustomerId NOT IN (
    SELECT CustomerId FROM Orders
    WHERE Status = 'Cancelled' AND CustomerId IS NOT NULL
);

-- ✅ БЕЗПЕЧНО, варіант 2 (кращий за замовчуванням): NOT EXISTS взагалі
-- не страждає від цієї проблеми, бо порівнює по одному значенню, а не через список
SELECT c.CustomerId, c.FullName
FROM Customers c
WHERE NOT EXISTS (
    SELECT 1 FROM Orders o
    WHERE o.CustomerId = c.CustomerId AND o.Status = 'Cancelled'
);
```

**Практичний висновок:** якщо є хоч найменша ймовірність `NULL` у колонці підзапиту для `NOT IN` — використовуй `NOT EXISTS`. Це не стилістична перевага, а **захист від реального, важко виявного бага**.

---

## CTE (Common Table Expressions) та рекурсивні CTE

### WITH — іменований тимчасовий результат усередині запиту

CTE (`WITH ... AS (...)`) дозволяє дати ім'я підзапиту й використовувати його далі так, ніби це звичайна таблиця — це насамперед інструмент **читабельності та повторного використання в межах одного запиту**, а не окрема сутність продуктивності (у більшості СУБД нерекурсивний CTE концептуально еквівалентний похідній таблиці з розділу вище).

```sql
WITH CustomerSpending AS (
    SELECT o.CustomerId, SUM(oi.Quantity * oi.UnitPrice) AS TotalSpent
    FROM Orders o
    JOIN OrderItems oi ON oi.OrderId = o.OrderId
    GROUP BY o.CustomerId
)
SELECT c.FullName, cs.TotalSpent
FROM Customers c
JOIN CustomerSpending cs ON cs.CustomerId = c.CustomerId
WHERE cs.TotalSpent > 5000
ORDER BY cs.TotalSpent DESC;
```

Можна ланцюжком оголошувати декілька CTE, де кожен наступний може використовувати попередній — це особливо цінно для складних звітів (побачимо це в дії у розділі "Реальний сценарій"):

```sql
WITH
CustomerSpending AS (
    SELECT o.CustomerId, SUM(oi.Quantity * oi.UnitPrice) AS TotalSpent
    FROM Orders o
    JOIN OrderItems oi ON oi.OrderId = o.OrderId
    GROUP BY o.CustomerId
),
TopSpenders AS (
    SELECT * FROM CustomerSpending WHERE TotalSpent > 5000
)
SELECT c.FullName, ts.TotalSpent
FROM Customers c
JOIN TopSpenders ts ON ts.CustomerId = c.CustomerId;
```

### Рекурсивні CTE — обхід ієрархій

Це, мабуть, найпотужніший інструмент у "звичайному" SQL для роботи з деревоподібними структурами — і пряма практична паралель до патерну проєктування **Composite** (дерево "ціле-частина"), тільки тут дерево живе в реляційній таблиці через самопосилання, а не в об'єктах.

Згадай нашу таблицю `Categories` з `ParentCategoryId`. Завдання: **знайти всі підкатегорії заданої категорії на будь-якій глибині вкладеності** (не лише прямих дітей, а й дітей дітей, і так далі).

```sql
WITH CategoryTree AS (
    -- 1) АНКЕРНИЙ ЧЛЕН (anchor member) — стартова точка рекурсії.
    --    Виконується РІВНО ОДИН РАЗ і формує "нульовий" рівень.
    SELECT
        CategoryId,
        Name,
        ParentCategoryId,
        0 AS Depth
    FROM Categories
    WHERE CategoryId = 5   -- наприклад, "Електроніка"

    UNION ALL

    -- 2) РЕКУРСИВНИЙ ЧЛЕН (recursive member) — посилається САМ НА СЕБЕ (на CategoryTree).
    --    Виконується знову і знову, доки повертає хоча б один новий рядок.
    SELECT
        c.CategoryId,
        c.Name,
        c.ParentCategoryId,
        ct.Depth + 1
    FROM Categories c
    INNER JOIN CategoryTree ct ON c.ParentCategoryId = ct.CategoryId
)
SELECT CategoryId, Name, Depth
FROM CategoryTree
ORDER BY Depth, Name;
```

### Як саме розгортається рекурсія — крок за кроком

Уяви такі дані в `Categories`:

| CategoryId | Name | ParentCategoryId |
|---|---|---|
| 5 | Електроніка | NULL |
| 12 | Ноутбуки | 5 |
| 13 | Смартфони | 5 |
| 20 | Ігрові ноутбуки | 12 |
| 21 | Ультрабуки | 12 |

Виконання `CategoryTree` для `CategoryId = 5` розгортається так:

1. **Ітерація 0 (анкерний член):** повертає рядок `{5, "Електроніка", NULL, Depth=0}`. Це "робочий набір" для наступного кроку.
2. **Ітерація 1 (перший прогін рекурсивного члена):** беремо робочий набір з ітерації 0 (тільки `{5}`) і шукаємо в `Categories` рядки, де `ParentCategoryId` дорівнює одному з `CategoryId` цього набору → знаходимо `{12, "Ноутбуки", 5, Depth=1}` та `{13, "Смартфони", 5, Depth=1}`. Ці два рядки і додаються до загального результату, і стають новим "робочим набором".
3. **Ітерація 2:** беремо робочий набір `{12, 13}` і шукаємо дітей уже **для них** → знаходимо `{20, "Ігрові ноутбуки", 12, Depth=2}` та `{21, "Ультрабуки", 12, Depth=2}` (у `13` дітей немає). Додаємо до результату.
4. **Ітерація 3:** робочий набір тепер `{20, 21}`. Шукаємо дітей для них — **нічого не знайдено**. Рекурсивний член повертає 0 рядків.
5. **Зупинка:** щойно рекурсивний член повертає порожній результат, рекурсія **автоматично зупиняється**. Фінальний результат — це об'єднання (`UNION ALL`) усіх рядків, накопичених на кожній ітерації: `{5}, {12,13}, {20,21}`.

Результат:

| CategoryId | Name | Depth |
|---|---|---|
| 5 | Електроніка | 0 |
| 12 | Ноутбуки | 1 |
| 13 | Смартфони | 1 |
| 20 | Ігрові ноутбуки | 2 |
| 21 | Ультрабуки | 2 |

### Важливі технічні деталі рекурсивних CTE

- **Обов'язково `UNION ALL`, а не `UNION`.** Стандарт SQL вимагає саме `UNION ALL` для рекурсивного члена (перевірка на дублікати на кожній ітерації була б надто дорогою, а для дерев без циклів дублікатів і так не буде за коректної умови з'єднання).
- **Захист від нескінченної рекурсії:** якщо в даних випадково утворився цикл (наприклад, через помилку `A → B → A` у `ParentCategoryId`), рекурсія піде нескінченно. SQL Server має вбудований запобіжник `OPTION (MAXRECURSION 100)` (за замовчуванням ліміт — 100 ітерацій, `0` означає "без обмежень" — обережно!); у PostgreSQL захист треба реалізовувати вручну, накопичуючи шлях і перевіряючи, чи вже відвідано вузол.
- **Діалектна відмінність:** PostgreSQL і MySQL 8+ вимагають писати `WITH RECURSIVE CategoryTree AS (...)` — ключове слово `RECURSIVE` обов'язкове. SQL Server розпізнає рекурсію автоматично, і достатньо просто `WITH CategoryTree AS (...)`.
- **Зворотна задача** (знайти всіх **предків** заданої категорії, тобто піднятися вгору по дереву замість спуску вниз) вирішується так само, просто `JOIN` в рекурсивному члені йде в інший бік: `INNER JOIN CategoryTree ct ON c.CategoryId = ct.ParentCategoryId`.

---

## Віконні функції (Window Functions)

Це один із найважливіших інструментів, який відрізняє Middle від Senior у практичному SQL. Віконні функції виконують обчислення **над набором рядків, пов'язаних із поточним рядком**, але, на відміну від `GROUP BY`, **не згортають** рядки в один — кожен вихідний рядок залишається на місці, просто отримує додаткову обчислену колонку.

### Синтаксис: OVER(), PARTITION BY, ORDER BY

```sql
<функція>() OVER (
    [PARTITION BY <колонка(и), що ділять дані на групи>]
    [ORDER BY <колонка(и) сортування всередині кожної групи>]
    [ROWS/RANGE BETWEEN <межа> AND <межа>]
)
```

**Ключова відмінність від `GROUP BY`:**

```sql
-- GROUP BY: 1 рядок на кожну категорію (деталі окремих товарів втрачені)
SELECT CategoryId, AVG(Price) AS AvgPrice
FROM Products
GROUP BY CategoryId;

-- Віконна функція: рядків стільки ж, скільки товарів,
-- але поруч із кожним товаром — середня ціна ЙОГО категорії
SELECT
    ProductId,
    Name,
    CategoryId,
    Price,
    AVG(Price) OVER (PARTITION BY CategoryId) AS AvgPriceInCategory
FROM Products;
```

Другий варіант дозволяє в одному рядку порівняти ціну конкретного товару із середньою ціною по його категорії — і саме тому віконні функції незамінні у звітності: вони поєднують деталізацію (кожен рядок окремо) з агрегацією (контекст групи).

### ROW_NUMBER(), RANK(), DENSE_RANK()

Усі три присвоюють порядковий номер рядкам усередині вікна, але по-різному обробляють **однакові значення (tie)**:

```sql
SELECT
    ProductId,
    Name,
    Price,
    ROW_NUMBER() OVER (ORDER BY Price DESC)   AS RowNum,
    RANK()       OVER (ORDER BY Price DESC)   AS Rnk,
    DENSE_RANK() OVER (ORDER BY Price DESC)   AS DenseRnk
FROM Products;
```

Уяви 4 товари з цінами `1000, 900, 900, 800`:

| Name | Price | ROW_NUMBER | RANK | DENSE_RANK |
|---|---|---|---|---|
| Товар A | 1000 | 1 | 1 | 1 |
| Товар B | 900 | 2 | 2 | 2 |
| Товар C | 900 | 3 | 2 | 2 |
| Товар D | 800 | 4 | 4 | 3 |

- **`ROW_NUMBER()`** завжди дає унікальні послідовні номери — навіть для однакових значень (порядок серед "рівних" по суті довільний, якщо `ORDER BY` не деталізує далі).
- **`RANK()`** дає однакове місце рядкам з однаковим значенням, але **пропускає** наступні номери (після двох рядків з рангом `2` наступний ранг — `4`, а не `3`) — так само, як у спортивному рейтингу: два перших місця → наступне місце "третє" не присуджується нікому.
- **`DENSE_RANK()`** теж дає однакове місце рівним рядкам, але **не пропускає** номери (`1, 2, 2, 3` — без "дірок").

### LAG() та LEAD() — доступ до сусіднього рядка

`LAG()` повертає значення з **попереднього** рядка вікна, `LEAD()` — зі **наступного**. Класичний приклад: обчислення зміни продажів день-до-дня.

```sql
WITH DailySales AS (
    SELECT
        CAST(o.OrderDate AS DATE) AS SaleDate,
        SUM(oi.Quantity * oi.UnitPrice) AS Revenue
    FROM Orders o
    JOIN OrderItems oi ON oi.OrderId = o.OrderId
    GROUP BY CAST(o.OrderDate AS DATE)
)
SELECT
    SaleDate,
    Revenue,
    LAG(Revenue)  OVER (ORDER BY SaleDate) AS PrevDayRevenue,
    Revenue - LAG(Revenue) OVER (ORDER BY SaleDate) AS DayOverDayChange,
    LEAD(Revenue) OVER (ORDER BY SaleDate) AS NextDayRevenue
FROM DailySales
ORDER BY SaleDate;
```

Для першого рядка `LAG(Revenue)` поверне `NULL` (попереднього дня немає) — це нормально, і варто врахувати це в подальших обчисленнях (наприклад, через `COALESCE`).

### Накопичувальний підсумок (running total)

```sql
WITH DailySales AS (
    SELECT
        CAST(o.OrderDate AS DATE) AS SaleDate,
        SUM(oi.Quantity * oi.UnitPrice) AS Revenue
    FROM Orders o
    JOIN OrderItems oi ON oi.OrderId = o.OrderId
    GROUP BY CAST(o.OrderDate AS DATE)
)
SELECT
    SaleDate,
    Revenue,
    SUM(Revenue) OVER (
        ORDER BY SaleDate
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS RunningTotal
FROM DailySales
ORDER BY SaleDate;
```

`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` буквально означає "від першого рядка вікна до поточного включно" — саме тому кожен наступний рядок накопичує суму всіх попередніх.

### NTILE() — розбиття на групи (бакети)

`NTILE(N)` ділить впорядкований набір рядків на `N` максимально рівних груп — зручно для перцентилів, квартилів, сегментації клієнтів:

```sql
-- Ділимо клієнтів на 4 квартилі за сумою витрат: 1 = найбільші витрати, 4 = найменші
WITH CustomerSpending AS (
    SELECT o.CustomerId, SUM(oi.Quantity * oi.UnitPrice) AS TotalSpent
    FROM Orders o
    JOIN OrderItems oi ON oi.OrderId = o.OrderId
    GROUP BY o.CustomerId
)
SELECT
    CustomerId,
    TotalSpent,
    NTILE(4) OVER (ORDER BY TotalSpent DESC) AS SpendingQuartile
FROM CustomerSpending;
```

### 🎯 Реалістичний приклад: топ-3 найпопулярніших товари в КОЖНІЙ категорії

Це, мабуть, найпоширеніше практичне завдання на технічних співбесідах Middle/Senior рівня — і воно неможливе (точніше, вкрай незручне) без `PARTITION BY`:

```sql
WITH ProductSales AS (
    SELECT
        p.ProductId,
        p.Name,
        p.CategoryId,
        SUM(oi.Quantity) AS SalesCount
    FROM Products p
    JOIN OrderItems oi ON oi.ProductId = p.ProductId
    GROUP BY p.ProductId, p.Name, p.CategoryId
),
RankedSales AS (
    SELECT
        ProductId,
        Name,
        CategoryId,
        SalesCount,
        ROW_NUMBER() OVER (PARTITION BY CategoryId ORDER BY SalesCount DESC) AS RankInCategory
    FROM ProductSales
)
SELECT ProductId, Name, CategoryId, SalesCount, RankInCategory
FROM RankedSales
WHERE RankInCategory <= 3
ORDER BY CategoryId, RankInCategory;
```

**Чому саме `ROW_NUMBER()`, а не `RANK()` тут:** якщо два товари продались однаково добре, `RANK()` дав би обом місце "1", і при фільтрі `<= 3` категорія могла б повернути 4 товари замість 3. `ROW_NUMBER()` гарантує **рівно** 3 товари на категорію незалежно від нічиїх (яка саме з двох "рівних" позицій потрапить у топ-3 — залежить від додаткового критерію сортування, якщо він потрібен для детермінованості, варто додати tie-breaker, наприклад `ORDER BY SalesCount DESC, ProductId ASC`).

**Важливо:** результат `ROW_NUMBER()`/`RANK()`/тощо — це колонка в `SELECT`, а фільтрувати по ній у `WHERE` того ж рівня запиту **не можна** (за правилами порядку виконання: `WHERE` виконується до обчислення віконних функцій у `SELECT`, див. таблицю в розділі "Основи"). Тому фільтрація завжди виконується в **зовнішньому** запиті (тут — через CTE `RankedSales`).

---

## Множинні операції: UNION, UNION ALL, INTERSECT, EXCEPT

Ці оператори комбінують результати **двох окремих запитів** (з однаковою кількістю колонок і сумісними типами) за принципами теорії множин.

### UNION та UNION ALL

```sql
-- UNION: об'єднує результати і ВИДАЛЯЄ дублікати
SELECT CustomerId FROM Orders WHERE Status = 'Cancelled'
UNION
SELECT CustomerId FROM Reviews WHERE Rating = 1;

-- UNION ALL: об'єднує результати і НЕ видаляє дублікати (швидше!)
SELECT CustomerId FROM Orders WHERE Status = 'Cancelled'
UNION ALL
SELECT CustomerId FROM Reviews WHERE Rating = 1;
```

**Різниця в продуктивності — критична для Senior:** щоб прибрати дублікати, `UNION` змушений виконати додаткову операцію (найчастіше — сортування або хешування) над **усім** об'єднаним набором даних, щоб знайти й відкинути повтори. `UNION ALL` просто конкатенує результати обох запитів "як є", без жодної додаткової обробки.

> **Правило за замовчуванням:** завжди використовуй `UNION ALL`, якщо ти явно **не** потребуєш видалення дублікатів. Якщо ти точно знаєш, що набори результатів не перетинаються (наприклад, об'єднуєш замовлення за 2024 і 2025 роки за різними фільтрами дат) — `UNION ALL` дає ідентичний результат `UNION`, але без зайвої витрати ресурсів на сортування/хешування, яке в цьому випадку взагалі нічого не прибере.

### INTERSECT

Повертає лише ті рядки, що присутні **в обох** результатах одночасно:

```sql
-- Клієнти, які і зробили замовлення, І залишили відгук
SELECT CustomerId FROM Orders
INTERSECT
SELECT CustomerId FROM Reviews;
```

### EXCEPT (SQL Server / PostgreSQL) / MINUS (Oracle)

Повертає рядки з першого запиту, яких **немає** в другому:

```sql
-- Клієнти, які зробили замовлення, але НІКОЛИ не залишали відгук
SELECT CustomerId FROM Orders
EXCEPT
SELECT CustomerId FROM Reviews;
```

**Діалектна примітка:** Oracle використовує ключове слово `MINUS` замість `EXCEPT` для тієї самої операції.

---

## Практичні приклади на C#

### Приклад 1: ADO.NET з параметризованим запитом

```csharp
using Microsoft.Data.SqlClient;

const string connectionString = "Server=localhost;Database=Shop;Trusted_Connection=True;";

// Дано: userCountryInput надходить від користувача (наприклад, з поля пошуку в адмінці)
string userCountryInput = "Ukraine";

// ❌ НЕПРАВИЛЬНО: конкатенація рядків — пряма діра для SQL-ін'єкції.
// Якщо userCountryInput = "Ukraine'; DROP TABLE Customers; --", запит стане руйнівним.
string dangerousSql = "SELECT CustomerId, FullName FROM Customers WHERE Country = '" + userCountryInput + "'";

// ✅ ПРАВИЛЬНО: параметризований запит — значення передається окремо від тексту запиту,
// провайдер бази даних ніколи не інтерпретує вміст параметра як частину SQL-синтаксису.
const string safeSql = @"
    SELECT c.CustomerId, c.FullName, COUNT(o.OrderId) AS OrdersCount
    FROM Customers c
    LEFT JOIN Orders o ON o.CustomerId = c.CustomerId
    WHERE c.Country = @Country
    GROUP BY c.CustomerId, c.FullName
    ORDER BY OrdersCount DESC";

using var connection = new SqlConnection(connectionString);
await connection.OpenAsync();

using var command = new SqlCommand(safeSql, connection);
command.Parameters.Add(new SqlParameter("@Country", userCountryInput));

using var reader = await command.ExecuteReaderAsync();
while (await reader.ReadAsync())
{
    var fullName = reader.GetString(reader.GetOrdinal("FullName"));
    var ordersCount = reader.GetInt32(reader.GetOrdinal("OrdersCount"));
    Console.WriteLine($"{fullName}: {ordersCount} замовлень");
}
```

**Консольний вивід (приклад):**

```
Олена Коваленко: 7 замовлень
Тарас Шевченко: 4 замовлень
Ірина Мельник: 0 замовлень
```

### Приклад 2: Dapper

Dapper — легка ORM-бібліотека (micro-ORM), яка усуває майже все ручне обслуговування `SqlCommand`/`SqlDataReader`, залишаючи повний контроль над самим SQL-запитом і зберігаючи параметризацію "з коробки".

```csharp
using Dapper;
using Microsoft.Data.SqlClient;

const string connectionString = "Server=localhost;Database=Shop;Trusted_Connection=True;";

public record CustomerOrdersSummary(int CustomerId, string FullName, int OrdersCount);

string userCountryInput = "Ukraine";

const string sql = @"
    SELECT c.CustomerId, c.FullName, COUNT(o.OrderId) AS OrdersCount
    FROM Customers c
    LEFT JOIN Orders o ON o.CustomerId = c.CustomerId
    WHERE c.Country = @Country
    GROUP BY c.CustomerId, c.FullName
    ORDER BY OrdersCount DESC";

using var connection = new SqlConnection(connectionString);

// Dapper бере на себе: відкриття/закриття читача, мапінг колонок на властивості record,
// і, як бачимо, параметр @Country передається так само безпечно через анонімний об'єкт.
IEnumerable<CustomerOrdersSummary> results = await connection.QueryAsync<CustomerOrdersSummary>(
    sql,
    new { Country = userCountryInput });

foreach (var row in results)
{
    Console.WriteLine($"{row.FullName}: {row.OrdersCount} замовлень");
}
```

**Консольний вивід (ідентичний попередньому прикладу):**

```
Олена Коваленко: 7 замовлень
Тарас Шевченко: 4 замовлень
Ірина Мельник: 0 замовлень
```

Той самий результат, той самий SQL, той самий рівень безпеки (параметризація) — але без ручного циклу `while (reader.Read())`, ручного `GetOrdinal`/`GetString`/`GetInt32` і ручного маніпулювання об'єктами `SqlCommand`/`SqlParameter`. Це і є головна цінність Dapper: **менше боілерплейту, той самий контроль над SQL**.

### Приклад 3: EF Core (LINQ та SQL)

EF Core йде на крок далі: замість написання рядків SQL ти пишеш LINQ-вирази на C#, а EF Core **транслює** їх у SQL під час виконання.

```csharp
public class ShopDbContext : DbContext
{
    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Order> Orders => Set<Order>();
}

string userCountryInput = "Ukraine";

var results = await dbContext.Customers
    .Where(c => c.Country == userCountryInput)
    .GroupBy(c => new { c.CustomerId, c.FullName })
    .Select(g => new
    {
        g.Key.CustomerId,
        g.Key.FullName,
        OrdersCount = g.SelectMany(c => c.Orders).Count()
    })
    .OrderByDescending(x => x.OrdersCount)
    .ToListAsync();

foreach (var row in results)
{
    Console.WriteLine($"{row.FullName}: {row.OrdersCount} замовлень");
}
```

**Концептуальний SQL, який згенерує EF Core** (спрощено — реальний SQL від провайдера SQL Server матиме додаткові технічні деталі на кшталт `[c].[CustomerId]` в квадратних дужках):

```sql
SELECT c.CustomerId, c.FullName, COUNT(o.OrderId) AS OrdersCount
FROM Customers c
LEFT JOIN Orders o ON o.CustomerId = c.CustomerId
WHERE c.Country = @__userCountryInput_0
GROUP BY c.CustomerId, c.FullName
ORDER BY OrdersCount DESC;
```

Параметр `@__userCountryInput_0` — це той самий механізм параметризації, що і в прикладах вище: EF Core **ніколи** не вставляє значення `userCountryInput` прямо в текст SQL, тому LINQ-запити до бази даних настільки ж захищені від SQL-ін'єкцій, як і ручні параметризовані запити.

**Ключова Senior-навичка:** не покладатися сліпо на те, що "LINQ якось перетвориться на SQL і буде працювати швидко". Сучасний EF Core (з версії 5+) надає метод `ToQueryString()` прямо на `IQueryable`, який повертає **реальний** SQL-текст, що буде виконано — без звернення до бази даних:

```csharp
var query = dbContext.Customers
    .Where(c => c.Country == userCountryInput)
    .GroupBy(c => new { c.CustomerId, c.FullName })
    .Select(g => new { g.Key.CustomerId, g.Key.FullName, OrdersCount = g.Count() });

string generatedSql = query.ToQueryString();
Console.WriteLine(generatedSql);   // виводить точний SQL, який EF Core відправить у базу
```

Це саме та звичка, яка відрізняє розробника, що "просто пише LINQ і сподівається на краще", від Senior-інженера, який **перевіряє** згенерований SQL перед тим, як код потрапить у продакшн — особливо для запитів зі складними `GroupBy`, вкладеними `Include()` чи великою кількістю `JOIN`-ів, де LINQ іноді транслюється в неочікувано неефективний SQL.

---

## Реальний сценарій: аналітичний запит для адмінки інтернет-магазину

**Бізнес-задача:** менеджер інтернет-магазину відкриває адмінку і хоче побачити звіт: *"Для кожної категорії покажи топ-3 товари за виручкою за останні 30 днів, разом із відсотком, який кожен товар складає від загальної виручки своєї категорії за цей період"*.

Це саме той тип запиту, який Senior-розробнику реально доводиться писати "наживо" — на співбесіді або на роботі для нового звіту в адмін-панелі. Побудуємо його поступово, крок за кроком, а не одразу фінальною версією.

### Крок 1: виручка по товарах за останні 30 днів

Спочатку відфільтруємо позиції замовлень за період і порахуємо виручку на рівні товару:

```sql
WITH RecentSales AS (
    SELECT
        p.ProductId,
        p.Name        AS ProductName,
        p.CategoryId,
        SUM(oi.Quantity * oi.UnitPrice) AS Revenue
    FROM OrderItems oi
    JOIN Orders o   ON o.OrderId = oi.OrderId
    JOIN Products p ON p.ProductId = oi.ProductId
    WHERE o.OrderDate >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.ProductId, p.Name, p.CategoryId
)
SELECT * FROM RecentSales;
```

На цьому кроці ми вже маємо "плаский" список: товар → категорія → виручка за 30 днів.

### Крок 2: виручка по категорії в цілому (для відсотка)

Тепер потрібне ще одне число на рядок — сумарна виручка **всієї категорії** цього товару. Це ідеальне місце для **віконної функції** з `PARTITION BY`, а не окремого `GROUP BY`-запиту: нам потрібно зберегти рядок на кожен товар, просто додавши до нього агрегат по категорії:

```sql
WITH RecentSales AS (
    SELECT
        p.ProductId,
        p.Name        AS ProductName,
        p.CategoryId,
        SUM(oi.Quantity * oi.UnitPrice) AS Revenue
    FROM OrderItems oi
    JOIN Orders o   ON o.OrderId = oi.OrderId
    JOIN Products p ON p.ProductId = oi.ProductId
    WHERE o.OrderDate >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.ProductId, p.Name, p.CategoryId
),
SalesWithCategoryTotal AS (
    SELECT
        *,
        SUM(Revenue) OVER (PARTITION BY CategoryId) AS CategoryTotalRevenue
    FROM RecentSales
)
SELECT * FROM SalesWithCategoryTotal;
```

Зверни увагу: `SUM(Revenue) OVER (PARTITION BY CategoryId)` тут — **без** `ORDER BY` всередині вікна, бо нам не потрібен накопичувальний підсумок — потрібна **загальна** сума по всій партиції (категорії), і кожен рядок цієї категорії отримає однакове значення `CategoryTotalRevenue`.

### Крок 3: ранжування товарів усередині категорії

Тепер додаємо `ROW_NUMBER()`, щоб пронумерувати товари всередині кожної категорії за спаданням виручки:

```sql
WITH RecentSales AS (
    SELECT
        p.ProductId,
        p.Name        AS ProductName,
        p.CategoryId,
        SUM(oi.Quantity * oi.UnitPrice) AS Revenue
    FROM OrderItems oi
    JOIN Orders o   ON o.OrderId = oi.OrderId
    JOIN Products p ON p.ProductId = oi.ProductId
    WHERE o.OrderDate >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.ProductId, p.Name, p.CategoryId
),
SalesWithCategoryTotal AS (
    SELECT
        *,
        SUM(Revenue) OVER (PARTITION BY CategoryId) AS CategoryTotalRevenue,
        ROW_NUMBER() OVER (PARTITION BY CategoryId ORDER BY Revenue DESC) AS RankInCategory
    FROM RecentSales
)
SELECT * FROM SalesWithCategoryTotal;
```

### Крок 4: фінальний запит — топ-3 з відсотком і назвою категорії

Залишається відфільтрувати `RankInCategory <= 3`, обчислити відсоток і приєднати назву категорії для читабельності звіту:

```sql
WITH RecentSales AS (
    SELECT
        p.ProductId,
        p.Name        AS ProductName,
        p.CategoryId,
        SUM(oi.Quantity * oi.UnitPrice) AS Revenue
    FROM OrderItems oi
    JOIN Orders o   ON o.OrderId = oi.OrderId
    JOIN Products p ON p.ProductId = oi.ProductId
    WHERE o.OrderDate >= DATEADD(DAY, -30, GETDATE())
    GROUP BY p.ProductId, p.Name, p.CategoryId
),
SalesWithCategoryTotal AS (
    SELECT
        *,
        SUM(Revenue) OVER (PARTITION BY CategoryId) AS CategoryTotalRevenue,
        ROW_NUMBER() OVER (PARTITION BY CategoryId ORDER BY Revenue DESC) AS RankInCategory
    FROM RecentSales
)
SELECT
    cat.Name                                    AS CategoryName,
    s.ProductName,
    s.Revenue,
    s.CategoryTotalRevenue,
    CAST(ROUND(100.0 * s.Revenue / s.CategoryTotalRevenue, 2) AS DECIMAL(5,2)) AS PercentOfCategory,
    s.RankInCategory
FROM SalesWithCategoryTotal s
JOIN Categories cat ON cat.CategoryId = s.CategoryId
WHERE s.RankInCategory <= 3
ORDER BY cat.Name, s.RankInCategory;
```

**Приклад результату:**

| CategoryName | ProductName | Revenue | CategoryTotalRevenue | PercentOfCategory | RankInCategory |
|---|---|---|---|---|---|
| Електроніка | Ноутбук X15 | 45000 | 120000 | 37.50 | 1 |
| Електроніка | Смартфон Z9 | 30000 | 120000 | 25.00 | 2 |
| Електроніка | Навушники Pro | 18000 | 120000 | 15.00 | 3 |
| Одяг | Куртка зимова | 12000 | 40000 | 30.00 | 1 |
| Одяг | Кросівки Runner | 9000 | 40000 | 22.50 | 2 |

**Що варто пояснити на співбесіді, презентуючи такий запит:**

1. Чому `CategoryTotalRevenue` обчислено через віконну функцію, а не через окремий `GROUP BY CategoryId` запит з наступним `JOIN` — тому що віконна функція дозволяє отримати "деталь + агрегат по групі" **в одному проході** над уже підготовленими даними CTE `RecentSales`, без повторного сканування `OrderItems`/`Orders`.
2. Чому `ROW_NUMBER()`, а не `RANK()` — щоб гарантовано отримати рівно 3 товари на категорію (обґрунтування — у розділі про віконні функції вище).
3. Чому фільтр `RankInCategory <= 3` знаходиться в **зовнішньому** запиті, а не в тому ж CTE, де обчислюється `ROW_NUMBER()` — тому що `WHERE`/`HAVING` того самого рівня запиту виконуються **до** обчислення віконних функцій у `SELECT` (порядок виконання, розділ "Основи").
4. Як індексувати цей запит у реальній системі: складений індекс на `Orders(OrderDate)` (для швидкого відсікання останніх 30 днів) та на `OrderItems(ProductId, OrderId)` суттєво прискорять `JOIN` і фільтрацію ще до того, як CTE взагалі почне агрегацію.

---

## Поширені помилки

### 1. `SELECT *` у production-коді

```sql
-- ❌ Погано: крихко до змін схеми, зайвий трафік, неявна залежність від порядку колонок
SELECT * FROM Products WHERE CategoryId = 3;
```

```sql
-- ✅ Добре: явний список колонок
SELECT ProductId, Name, Price FROM Products WHERE CategoryId = 3;
```

**Чому це проблема:** якщо хтось додасть нову колонку `Description NVARCHAR(MAX)` до `Products`, запит `SELECT *` мовчки почне тягнути додатковий (потенційно важкий) обсяг даних по мережі в кожному місці коду, де він використовується — навіть там, де ця колонка нікому не потрібна. Якщо ж хтось **перейменує** чи **видалить** колонку, код на C#, що звертається до результату за іменем (`reader["OldColumnName"]`) чи навіть за позицією, зламається непередбачувано. Явний список колонок робить контракт запиту видимим і стабільним.

### 2. `NOT IN` з підзапитом, що може повернути NULL

Докладно розібрано в розділі про підзапити — повторимо як конкретний "баг у продакшні": розробник пише звіт "клієнти без скасованих замовлень", `Orders.CustomerId` дозволяє `NULL` (гостьові замовлення), і звіт **мовчки повертає нуль рядків**, хоча насправді таких клієнтів сотні.

```sql
-- ❌ Погано: якщо підзапит поверне хоча б один NULL — результат буде порожнім
SELECT * FROM Customers
WHERE CustomerId NOT IN (SELECT CustomerId FROM Orders WHERE Status = 'Cancelled');
```

```sql
-- ✅ Добре: NOT EXISTS не має цієї проблеми
SELECT c.* FROM Customers c
WHERE NOT EXISTS (
    SELECT 1 FROM Orders o WHERE o.CustomerId = c.CustomerId AND o.Status = 'Cancelled'
);
```

### 3. Корельований підзапит там, де JOIN або віконна функція значно ефективніші

```sql
-- ❌ Погано: корельований підзапит виконується логічно "для кожного товару окремо"
SELECT
    p.ProductId,
    p.Name,
    (SELECT AVG(r.Rating) FROM Reviews r WHERE r.ProductId = p.ProductId) AS AvgRating
FROM Products p;
```

```sql
-- ✅ Добре: один LEFT JOIN + GROUP BY обробляє всі товари за один прохід по Reviews
SELECT
    p.ProductId,
    p.Name,
    AVG(r.Rating) AS AvgRating
FROM Products p
LEFT JOIN Reviews r ON r.ProductId = p.ProductId
GROUP BY p.ProductId, p.Name;
```

На маленькій таблиці різниці можна й не помітити. На таблиці `Products` в 500 000 рядків і `Reviews` в 10 мільйонів різниця між "один комбінований прохід із `JOIN`+`GROUP BY`" і "мільйони окремих підзапитів" (навіть якщо оптимізатор частково їх перепише) може бути різницею між звітом, що будується за секунди, і звітом, що будується хвилинами.

### 4. Використання `UNION` замість `UNION ALL` за замовчуванням

```sql
-- ❌ Погано (якщо дублікати не потрібно прибирати): зайве сортування/хешування
--    над можливо мільйонами рядків лише щоб перевірити на дублікати, яких і так немає
SELECT CustomerId FROM Orders WHERE OrderDate >= '2024-01-01' AND OrderDate < '2024-07-01'
UNION
SELECT CustomerId FROM Orders WHERE OrderDate >= '2024-07-01' AND OrderDate < '2025-01-01';
```

```sql
-- ✅ Добре: діапазони дат явно не перетинаються — UNION ALL дає той самий результат швидше
SELECT CustomerId FROM Orders WHERE OrderDate >= '2024-01-01' AND OrderDate < '2024-07-01'
UNION ALL
SELECT CustomerId FROM Orders WHERE OrderDate >= '2024-07-01' AND OrderDate < '2025-01-01';
```

### 5. Функція над індексованою колонкою у WHERE (порушення "sargability")

```sql
-- ❌ Погано: функція YEAR() над колонкою в WHERE унеможливлює використання
--    звичайного індексу на OrderDate — рушій змушений обчислити YEAR() для КОЖНОГО рядка
SELECT * FROM Orders WHERE YEAR(OrderDate) = 2025;
```

```sql
-- ✅ Добре: діапазон дат "sargable" (Search ARGument ABLE) — індекс на OrderDate
--    можна використати напряму, без обчислень над кожним рядком таблиці
SELECT * FROM Orders
WHERE OrderDate >= '2025-01-01' AND OrderDate < '2026-01-01';
```

**Загальне правило sargability:** якщо колонка з `WHERE` обгорнута у функцію, перетворення типу чи арифметичну операцію (`YEAR(col)`, `col + 1`, `UPPER(col)`, `CAST(col AS ...)`), оптимізатор здебільшого **не може** застосувати звичайний індекс на цій колонці для швидкого пошуку — доводиться сканувати й обчислювати вираз для кожного рядка. Переписування умови так, щоб "гола" колонка стояла з одного боку порівняння (як у прикладі з діапазоном дат вище), — стандартний прийом оптимізації запитів на Senior-рівні.

---

## Питання для співбесіди за рівнями

### Junior

**1. Чим `WHERE` відрізняється від `HAVING`?**

`WHERE` фільтрує окремі рядки вихідних таблиць **до** групування й не має доступу до результатів агрегатних функцій. `HAVING` фільтрує вже згруповані рядки **після** того, як `GROUP BY` та агрегатні функції відпрацювали, тому саме в `HAVING` можна писати умови на кшталт `COUNT(*) > 5`.

**2. Чим `INNER JOIN` відрізняється від `LEFT JOIN`?**

`INNER JOIN` повертає лише ті рядки, для яких є відповідність в обох таблицях. `LEFT JOIN` повертає **всі** рядки лівої таблиці незалежно від того, чи є відповідність у правій — якщо відповідності немає, колонки правої таблиці заповнюються `NULL`.

**3. Що робить `GROUP BY`?**

Групує рядки з однаковими значеннями вказаних колонок в один вихідний рядок на групу, дозволяючи застосувати до кожної групи агрегатні функції (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`).

**4. Що таке первинний ключ (`PRIMARY KEY`) у контексті `JOIN`?**

Первинний ключ — унікальний, ненульовий ідентифікатор рядка в таблиці (наприклад, `CustomerId` в `Customers`). У `JOIN` він зазвичай виступає "одним боком" зв'язку "один-до-багатьох": на нього посилається зовнішній ключ (`FOREIGN KEY`, наприклад `Orders.CustomerId`) з іншої таблиці, і саме за рівністю цих значень (`ON c.CustomerId = o.CustomerId`) рушій зіставляє рядки під час `JOIN`.

**5. Чим `DISTINCT` відрізняється від `GROUP BY`?**

Обидва можуть прибирати дублікати, але `DISTINCT` — це операція над готовим набором колонок результату (прибрати повністю однакові рядки), тоді як `GROUP BY` призначений для групування даних із метою застосування агрегатних функцій. Якщо агрегати не потрібні — `DISTINCT` зазвичай зрозуміліший запис наміру.

**6. Що таке `NULL` і чому `WHERE Column = NULL` не працює?**

`NULL` означає "відсутність значення", а не "порожнє значення" чи "нуль". Оператор `=` побудований для порівняння значень, і порівняння будь-чого з `NULL` дає невизначений результат (`UNKNOWN`), а не `TRUE`. Тому для перевірки на `NULL` потрібні спеціальні оператори `IS NULL` / `IS NOT NULL`.

### Middle

**1. Чим `EXISTS` відрізняється від `IN` за продуктивністю?**

`EXISTS` зупиняється, щойно знайдений перший підходящий рядок (short-circuit), і не потребує матеріалізації повного списку значень. `IN` концептуально порівнює значення з усім набором, поверненим підзапитом. На великих або дублікатних підзапитах `EXISTS` зазвичай ефективніший, хоча сучасні оптимізатори часто зводять обидва варіанти до однакового фізичного плану виконання (semi-join). Критичніша відмінність — поведінка з `NULL` у `NOT IN`/`NOT EXISTS` (див. нижче).

**2. Що таке корельований підзапит?**

Підзапит, який посилається на колонку із зовнішнього запиту (наприклад, `WHERE o.CustomerId = c.CustomerId` всередині підзапиту, де `c` — таблиця зовнішнього запиту). Логічно такий підзапит обчислюється заново для кожного рядка зовнішнього запиту, на відміну від некорельованого підзапиту, який обчислюється один раз незалежно від зовнішніх рядків.

**3. Навіщо потрібен CTE, якщо можна написати підзапит?**

CTE (`WITH ... AS`) насамперед покращує читабельність і дозволяє повторно використати той самий проміжний результат кілька разів у межах одного запиту без дублювання тексту підзапиту. Також CTE — єдиний спосіб написати **рекурсивний** запит у стандартному SQL (обхід ієрархій), чого звичайний вкладений підзапит зробити не може.

**4. Чим `UNION` відрізняється від `UNION ALL`?**

`UNION` об'єднує результати двох запитів і видаляє дублікати (що вимагає додаткового сортування/хешування над усім набором). `UNION ALL` просто конкатенує результати без перевірки на дублікати — і тому швидший. За замовчуванням варто використовувати `UNION ALL`, якщо немає явної потреби прибирати повтори.

**5. Що таке зовнішній ключ (`FOREIGN KEY`) і навіщо він потрібен?**

Зовнішній ключ — обмеження цілісності, яке гарантує, що значення в одній колонці (наприклад, `Orders.CustomerId`) обов'язково відповідає існуючому значенню первинного ключа в іншій таблиці (`Customers.CustomerId`), або є `NULL`, якщо колонка допускає це. Це запобігає "осиротілим" записам (замовленню з клієнтом, якого не існує) на рівні самої бази даних, а не лише на рівні коду застосунку.

**6. Що таке транзакція і властивості ACID?**

Транзакція — набір операцій над базою даних, що виконується як єдине неподільне ціле. ACID: **A**tomicity (усе або нічого), **C**onsistency (база переходить з одного коректного стану в інший), **I**solation (паралельні транзакції не заважають одна одній непередбачувано), **D**urability (після підтвердження — зміни зберігаються навіть у разі збою).

### Senior

**1. Напиши запит, що знаходить топ-N записів у кожній групі. Поясни підхід через віконні функції.**

Класичний підхід: обгорнути дані в CTE, всередині якого обчислюється `ROW_NUMBER() OVER (PARTITION BY <група> ORDER BY <критерій> DESC)`, а потім у зовнішньому запиті відфільтрувати `WHERE RankInGroup <= N`. Причина, чому фільтр не можна поставити одразу в тому ж рівні запиту, де обчислюється `ROW_NUMBER()`, — `WHERE` виконується до обчислення віконних функцій у `SELECT` за логічним порядком виконання запиту. `ROW_NUMBER()` варто обирати замість `RANK()`, якщо потрібна **гарантовано точна** кількість N записів на групу навіть за наявності нічиїх у значенні критерію сортування; для інших сценаріїв, де "нічия" повинна означати "обидва входять у топ", доречніший `RANK()`.

**2. Поясни, як рекурсивний CTE технічно виконується під капотом.**

Рекурсивний CTE логічно виконується ітеративно: спочатку одноразово обчислюється анкерний член, який формує "робочий набір" (стартові рядки). Далі рекурсивний член виконується повторно: на кожній ітерації він приєднується (`JOIN`) лише до рядків, доданих на **попередній** ітерації (а не до всього накопиченого результату — це важливий нюанс, який гарантує, що робота на кожному кроці пропорційна кількості нових рядків, а не всьому дереву), і повертає наступний "шар" даних. Це триває, доки черговий прогін рекурсивного члена не поверне 0 рядків — тоді рекурсія зупиняється. Фінальний результат — це `UNION ALL` усіх шарів. Обов'язковість `UNION ALL` (замість `UNION`) саме через це: перевірка на дублікати на кожній ітерації була б надто витратною і в стандарті вважається відповідальністю автора запиту (наприклад, через явне відстеження відвіданих вузлів для захисту від циклів).

**3. Коли віконна функція краща за `GROUP BY`, а коли навпаки?**

`GROUP BY` доречний, коли потрібен **лише** агрегований результат — по одному рядку на групу, деталі окремих рядків не важливі (наприклад, "сума продажів по категоріях" для діаграми). Віконна функція доречна, коли потрібно **зберегти** деталізацію на рівні окремого рядка, водночас маючи доступ до агрегату по групі в тому ж рядку — наприклад, "показати кожен товар поруч із середньою ціною його категорії", "пронумерувати товари всередині категорії", "порахувати накопичувальний підсумок", "порівняти рядок із сусіднім за часом (`LAG`/`LEAD`)". Якщо потрібні **і** деталізовані рядки, **і** агрегати одночасно в одному наборі результатів — це майже завжди сигнал використовувати віконну функцію замість `GROUP BY` з подальшим `JOIN` назад до деталей.

**4. Опиши небезпеку `NOT IN` з підзапитом, що може повернути `NULL`.**

SQL використовує трьохзначну логіку (`TRUE`/`FALSE`/`UNKNOWN`). Вираз `x NOT IN (a, b, NULL)` розкривається як `x <> a AND x <> b AND x <> NULL`. Порівняння з `NULL` завжди дає `UNKNOWN`, а ланцюжок `AND` із хоча б одним `UNKNOWN` ніколи не може дати `TRUE`. Тому якщо підзапит для `NOT IN` поверне серед своїх значень хоча б один `NULL`, **увесь** зовнішній запит поверне нуль рядків для кожного зовнішнього рядка — без жодної помилки виконання, що робить цей баг особливо небезпечним (мовчазна втрата даних у звіті). Захист: явно відфільтрувати `NULL` у підзапиті (`... WHERE колонка IS NOT NULL`) або, що надійніше за замовчуванням, використовувати `NOT EXISTS`, яка не страждає від цієї проблеми, бо порівнює значення попарно, а не через побудову списку.

**5. Як би ти оптимізував повільний звітний запит із кількома `JOIN`-ами?**

Послідовність дій: (1) отримати реальний план виконання (`EXPLAIN ANALYZE` в PostgreSQL, Actual Execution Plan у SQL Server) замість здогадок; (2) перевірити, чи є індекси на колонках, за якими відбувається `JOIN` та фільтрація в `WHERE` — особливо на зовнішніх ключах, які за замовчуванням **не** завжди індексуються автоматично; (3) перевірити sargability умов `WHERE` — чи не обгорнуті колонки у функції/перетворення типів, що унеможливлює використання індексу; (4) переконатися, що фільтрація (`WHERE`) відбувається якомога раніше і скорочує обсяг даних до важких `JOIN`-ів, а не після них; (5) розглянути заміну корельованих підзапитів на `JOIN` або віконні функції там, де це логічно можливо; (6) для звітів, що виконуються часто над великим обсягом історичних даних, розглянути матеріалізовані/індексовані представлення (materialized views / indexed views) або попередньо агреговані таблиці, що оновлюються за розкладом, замість перерахунку "наживо" щоразу; (7) перевірити застарілість статистики оптимізатора (`UPDATE STATISTICS` / `ANALYZE`) — застаріла статистика може змусити оптимізатор обрати неоптимальний план навіть за наявності правильних індексів.

---

## Підсумок

### Шпаргалка: типи JOIN

| JOIN | Що повертає | Типовий випадок використання |
|---|---|---|
| `INNER JOIN` | Лише рядки зі збігом в обох таблицях | "Замовлення разом з ім'ям клієнта" |
| `LEFT JOIN` | Усі рядки лівої таблиці + збіги справа (або `NULL`) | "Клієнти без жодного замовлення" (`WHERE right.id IS NULL`) |
| `RIGHT JOIN` | Усі рядки правої таблиці + збіги зліва (або `NULL`) | Рідко; зазвичай переписується як `LEFT JOIN` з іншим порядком таблиць |
| `FULL OUTER JOIN` | Усі рядки з обох таблиць, незбіжні — з `NULL` з іншого боку | Звірка (reconciliation) двох наборів даних |
| `CROSS JOIN` | Декартів добуток (N × M рядків) | Генерація щільної звітної сітки (категорії × місяці) |
| Self-JOIN | Таблиця з'єднана сама з собою через два аліаси | Ієрархії (`ParentCategoryId`), пошук схожих/дублікатів рядків |

### Шпаргалка: віконні функції

| Функція | Призначення |
|---|---|
| `ROW_NUMBER()` | Унікальний послідовний номер рядка у вікні (навіть для "нічиїх") |
| `RANK()` | Місце в рейтингу; "нічиї" отримують однакове місце, наступні номери пропускаються |
| `DENSE_RANK()` | Те саме, що `RANK()`, але без пропусків у нумерації |
| `LAG(col)` | Значення з попереднього рядка вікна |
| `LEAD(col)` | Значення з наступного рядка вікна |
| `SUM()/AVG()/COUNT() OVER (...)` | Агрегат по вікну без згортання рядків (на відміну від `GROUP BY`) |
| `NTILE(N)` | Розбиття впорядкованих рядків на N рівних груп (квартилі, перцентилі) |

### Коли що використовувати: короткий гід прийняття рішень

| Потрібно... | Використовуй |
|---|---|
| Знайти рядки в A без відповідності в B | `LEFT JOIN ... WHERE b.id IS NULL` або `NOT EXISTS` (безпечніше за `NOT IN`) |
| Перевірити лише факт існування збігу | `EXISTS` / `NOT EXISTS` |
| Отримати одне число з підзапиту в колонку | Скалярний підзапит або `JOIN` до похідної таблиці |
| Обхід дерева/ієрархії довільної глибини | Рекурсивний CTE (`WITH RECURSIVE` / `WITH ... UNION ALL`) |
| Топ-N записів у кожній групі | `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` + фільтр у зовнішньому запиті |
| Деталізовані рядки + агрегат по групі одночасно | Віконна функція (`OVER (PARTITION BY ...)`) |
| Лише згорнутий агрегат по групі, без деталей | `GROUP BY` |
| Об'єднати два набори без дублікатів | `UNION` (тільки якщо дублікати справді можливі й небажані) |
| Об'єднати два завідомо неперетинні набори | `UNION ALL` (за замовчуванням — швидше) |
| Умова на результат агрегатної функції | `HAVING`, а не `WHERE` |

**Головні принципи, які варто винести з цього документа:**

1. SQL — декларативна мова: описуй **що** отримати, довірся оптимізатору щодо **як**, але розумій його достатньо, щоб писати запити, які він може виконати ефективно.
2. `WHERE` фільтрує рядки до групування, `HAVING` — групи після нього; весь інший синтаксис випливає з логічного порядку виконання запиту (`FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY`).
3. `NOT IN` з підзапитом, що потенційно повертає `NULL`, — це прихована бомба сповільненої дії; `NOT EXISTS` — безпечніший інструмент за замовчуванням.
4. Віконні функції — це "деталізація плюс агрегація в одному рядку"; `GROUP BY` — це "лише агрегація, без деталізації".
5. Розуміти згенерований SQL (через `ToQueryString()` в EF Core чи аналогічні інструменти) — обов'язкова навичка, коли пишеш на LINQ/ORM, а не привілей "для тих, хто любить SQL".

---

*Документ підготовлено як частина навчальної серії з баз даних — від основ до сіньйорського рівня.*
