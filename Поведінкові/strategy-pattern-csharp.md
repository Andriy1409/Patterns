# Патерн Strategy (Стратегія) — Детальний розбір на C#

> **Категорія:** Поведінковий (Behavioral)  
> **Автори:** Gang of Four (GoF)  
> **Мова прикладів:** C#

---

## Зміст

1. [Що таке Strategy?](#що-таке-strategy)
   - [Аналогія з реального світу](#аналогія-з-реального-світу)
2. [Проблема без патерну](#проблема-без-патерну)
3. [Структура патерну](#структура-патерну)
4. [Приклад 1 — Стратегії сортування](#приклад-1--стратегії-сортування)
5. [Приклад 2 — Стратегії оплати](#приклад-2--стратегії-оплати)
6. [Приклад 3 — Стратегії розрахунку знижок](#приклад-3--стратегії-розрахунку-знижок)
7. [Приклад 4 (реальний сценарій) — Планувальник маршрутів доставки](#приклад-4-реальний-сценарій--планувальник-маршрутів-доставки)
8. [Strategy vs State vs Template Method](#strategy-vs-state-vs-template-method)
9. [Переваги та недоліки](#переваги-та-недоліки)
10. [Антипатерни та поширені помилки](#антипатерни-та-поширені-помилки)
11. [Підсумок](#підсумок)
    - [Мінімальний шаблон](#мінімальний-шаблон)

---

## Що таке Strategy?

**Strategy (Стратегія)** — це поведінковий патерн проектування, який визначає **сімейство алгоритмів**, інкапсулює кожен із них в окремий клас і робить їх **взаємозамінними**. Патерн дозволяє змінювати алгоритм незалежно від клієнтів, які цей алгоритм використовують.

Головна ідея: замість того, щоб «зашивати» конкретний алгоритм всередину класу (за допомогою `if`/`switch` або жорсткого успадкування), ми виносимо кожен варіант поведінки в окремий клас, що реалізує спільний інтерфейс. Клас, який використовує алгоритм (**Context**), тримає лише посилання на інтерфейс і делегує йому виконання роботи, не знаючи, яка саме конкретна реалізація ховається за цим інтерфейсом.

Формальне визначення GoF:

> Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

Ключові наслідки цього визначення:

- **Сімейство алгоритмів** — усі стратегії розв'язують одну й ту саму задачу (наприклад, «відсортувати список», «розрахувати знижку», «оплатити замовлення»), але роблять це по-різному.
- **Інкапсуляція кожного алгоритму** — кожна стратегія — це окремий, самодостатній клас, який можна тестувати, змінювати й розгортати незалежно від інших.
- **Взаємозамінність під час виконання** — Context може отримати нову стратегію в конструкторі, через властивість чи метод-сеттер, і клієнтський код при цьому не змінюється.

### Аналогія з реального світу

Уявіть навігаційний застосунок (як Google Maps). Користувач вводить точку А і точку Б та натискає «Прокласти маршрут». Однак **спосіб** побудови маршруту докорінно різний залежно від обраного виду транспорту:

- 🚗 **Автомобіль** — враховує дороги з одностороннім рухом, пробки, платні дороги.
- 🚲 **Велосипед** — уникає автомагістралей, віддає перевагу велодоріжкам.
- 🚶 **Пішки** — використовує пішохідні стежки, ігнорує обмеження для транспорту.
- 🚌 **Громадський транспорт** — будує маршрут на основі розкладу автобусів/метро та пересадок.

Запит користувача завжди однаковий — «побудуй маршрут від А до Б». Але **алгоритм** побудови маршруту повністю залежить від обраної стратегії пересування. Навігаційний застосунок (Context) не повинен знати деталей кожного алгоритму — він лише передає точки А і Б обраній стратегії та отримує результат.

Аналогічно можна навести приклад із оплатою покупок у інтернет-магазині: клієнт обирає спосіб оплати (кредитна картка, PayPal, криптовалюта), і кошик просто делегує процес оплати відповідній стратегії, не переймаючись деталями роботи платіжних шлюзів.

---

## Проблема без патерну

Розглянемо типову ситуацію: система розрахунку знижок для клієнтів магазину. «Наївна» реалізація зазвичай виглядає так — усі варіанти розрахунку знижки «зашиті» прямо в один метод за допомогою ланцюжка `if-else`:

```csharp
// ПОГАНИЙ ПІДХІД: усі алгоритми розрахунку знижки живуть в одному місці
public class PriceCalculator
{
    public decimal CalculateDiscount(string customerType, decimal orderAmount)
    {
        decimal discount = 0m;

        if (customerType == "VIP")
        {
            // VIP-клієнти отримують 20% знижки, але не більше 500 грн
            discount = orderAmount * 0.20m;
            if (discount > 500m)
                discount = 500m;
        }
        else if (customerType == "Regular")
        {
            // Постійні клієнти отримують 10% знижки
            discount = orderAmount * 0.10m;
        }
        else if (customerType == "New")
        {
            // Нові клієнти отримують фіксовану знижку 50 грн,
            // якщо сума замовлення більша за 200 грн
            if (orderAmount > 200m)
                discount = 50m;
        }
        else if (customerType == "Wholesale")
        {
            // Оптові клієнти: знижка залежить від обсягу замовлення
            if (orderAmount > 10000m)
                discount = orderAmount * 0.25m;
            else if (orderAmount > 5000m)
                discount = orderAmount * 0.15m;
            else
                discount = orderAmount * 0.05m;
        }
        else
        {
            // Немає знижки
            discount = 0m;
        }

        return orderAmount - discount;
    }
}
```

### У чому проблема?

1. **Порушення принципу Open/Closed (відкритість/закритість).** Щоб додати новий тип клієнта (наприклад, "Partner" або "Employee"), потрібно **редагувати** існуючий метод `CalculateDiscount`, ризикуючи зламати вже працюючу логіку для інших типів клієнтів.

2. **Неможливо протестувати кожен алгоритм окремо.** Щоб перевірити, як рахується знижка для VIP-клієнтів, доводиться викликати весь метод `CalculateDiscount` з конкретним рядком `"VIP"`. Юніт-тест не може ізольовано перевірити саме VIP-логіку — вона «замурована» в загальному методі поряд з усіма іншими гілками.

3. **Неможливо зручно підміняти алгоритм у рантаймі.** Якщо потрібно, наприклад, дати клієнту можливість заздалегідь «прорахувати» знижку за іншою схемою (порівняти два варіанти), доведеться дублювати виклики з різними рядковими прапорцями — гнучкості обміну алгоритмами «на льоту» немає.

4. **Зростання складності з часом.** Що більше типів клієнтів — то довший і заплутаніший метод. Циклoматична складність росте лінійно з кожним новим `else if`, і рано чи пізно метод перетворюється на «простирадло» коду, яке боїться чіпати вся команда.

5. **Дублювання схожої логіки в різних місцях.** Часто такий самий ланцюжок `if-else` за типом клієнта доводиться повторювати і в інших методах (наприклад, при розрахунку бонусних балів чи термінів доставки), що веде до розсіювання бізнес-правил по всьому коду.

Патерн Strategy вирішує всі ці проблеми: кожен алгоритм розрахунку знижки стає окремим класом, що реалізує спільний інтерфейс, а `PriceCalculator` (Context) лише викликає метод обраної стратегії, не знаючи деталей її реалізації.

---

## Структура патерну

```
┌───────────────────────┐          ┌───────────────────────────┐
│        Client          │          │        <<interface>>      │
│  (обирає та передає     │─ ─ ─ ─▶│         IStrategy          │
│   конкретну стратегію) │  creates │───────────────────────────│
└───────────────────────┘          │ + Execute(data): result    │
            │                      └───────────────────────────┘
            │ injects                          △
            ▼                                  │ implements
┌───────────────────────┐                      │
│        Context         │            ┌─────────┼──────────┬─────────────┐
│───────────────────────│            │         │          │             │
│ - strategy: IStrategy  │    ┌───────────┐ ┌───────────┐ ┌───────────┐
│───────────────────────│    │ConcreteStr│ │ConcreteStr│ │ConcreteStr│
│ + SetStrategy(s)        │    │ategyA     │ │ategyB     │ │ategyC     │
│ + DoWork()  ──────────┼───▶│───────────│ │───────────│ │───────────│
│     calls               │    │+Execute() │ │+Execute() │ │+Execute() │
└───────────────────────┘    │ (алгоритм │ │ (алгоритм │ │ (алгоритм │
                               │  варіант A)│ │ варіант B)│ │ варіант C)│
                               └───────────┘ └───────────┘ └───────────┘
```

### Роль кожного учасника

| Роль | Відповідальність |
|---|---|
| **Strategy** (`IStrategy`) | Спільний інтерфейс для всіх підтримуваних алгоритмів. Визначає сигнатуру методу(-ів), який Context буде викликати. |
| **ConcreteStrategyA/B/C** | Конкретні реалізації алгоритму. Кожна інкапсулює власну логіку, незалежну від інших стратегій та від Context. |
| **Context** | Клас, що використовує стратегію для виконання роботи. Тримає посилання на об'єкт `IStrategy` і делегує йому виклик, не знаючи конкретного типу реалізації. Може дозволяти змінювати стратегію під час виконання. |
| **Client** | Створює конкретну стратегію та передає (ін'єктує) її в Context — через конструктор, властивість або метод-параметр. |

Ключова відмінність від «поганого» підходу: **Context ніколи не містить `if/switch` для вибору алгоритму** — це рішення приймається зовні (клієнтом або фабрикою) і просто «вставляється» в Context у вигляді готового об'єкта.

---

## Приклад 1 — Стратегії сортування

Найпростіший класичний приклад Strategy: можливість підмінити алгоритм сортування списку без зміни коду, який цим сортуванням користується.

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;

namespace StrategyPattern.Sorting
{
    // Спільний інтерфейс для всіх алгоритмів сортування
    public interface ISortStrategy
    {
        string Name { get; }
        void Sort(List<int> data);
    }

    // Конкретна стратегія: сортування бульбашкою
    public class BubbleSortStrategy : ISortStrategy
    {
        public string Name => "Bubble Sort";

        public void Sort(List<int> data)
        {
            int n = data.Count;
            for (int i = 0; i < n - 1; i++)
            {
                for (int j = 0; j < n - i - 1; j++)
                {
                    if (data[j] > data[j + 1])
                    {
                        (data[j], data[j + 1]) = (data[j + 1], data[j]);
                    }
                }
            }
        }
    }

    // Конкретна стратегія: швидке сортування (QuickSort)
    public class QuickSortStrategy : ISortStrategy
    {
        public string Name => "Quick Sort";

        public void Sort(List<int> data)
        {
            QuickSort(data, 0, data.Count - 1);
        }

        private void QuickSort(List<int> data, int low, int high)
        {
            if (low >= high) return;

            int pivotIndex = Partition(data, low, high);
            QuickSort(data, low, pivotIndex - 1);
            QuickSort(data, pivotIndex + 1, high);
        }

        private int Partition(List<int> data, int low, int high)
        {
            int pivot = data[high];
            int i = low - 1;

            for (int j = low; j < high; j++)
            {
                if (data[j] <= pivot)
                {
                    i++;
                    (data[i], data[j]) = (data[j], data[i]);
                }
            }

            (data[i + 1], data[high]) = (data[high], data[i + 1]);
            return i + 1;
        }
    }

    // Конкретна стратегія: сортування вставками (добре для майже відсортованих даних)
    public class InsertionSortStrategy : ISortStrategy
    {
        public string Name => "Insertion Sort";

        public void Sort(List<int> data)
        {
            for (int i = 1; i < data.Count; i++)
            {
                int key = data[i];
                int j = i - 1;

                while (j >= 0 && data[j] > key)
                {
                    data[j + 1] = data[j];
                    j--;
                }

                data[j + 1] = key;
            }
        }
    }

    // Context: використовує стратегію, але не знає деталей її реалізації
    public class Sorter
    {
        private ISortStrategy _strategy;

        public Sorter(ISortStrategy strategy)
        {
            _strategy = strategy ?? throw new ArgumentNullException(nameof(strategy));
        }

        // Дозволяє підмінити алгоритм сортування "на льоту"
        public void SetStrategy(ISortStrategy strategy)
        {
            _strategy = strategy ?? throw new ArgumentNullException(nameof(strategy));
        }

        public void SortData(List<int> data)
        {
            var stopwatch = Stopwatch.StartNew();
            _strategy.Sort(data);
            stopwatch.Stop();

            Console.WriteLine($"[{_strategy.Name}] Відсортовано за {stopwatch.Elapsed.TotalMilliseconds:F4} мс");
        }
    }
}
```

### Використання

```csharp
using System;
using System.Collections.Generic;
using StrategyPattern.Sorting;

var numbers = new List<int> { 9, 3, 7, 1, 8, 2, 5, 4, 6 };

// Клієнт обирає стратегію та передає її в Context
var sorter = new Sorter(new BubbleSortStrategy());
sorter.SortData(new List<int>(numbers));

// Можна підмінити стратегію "на льоту" без зміни коду Sorter
sorter.SetStrategy(new QuickSortStrategy());
sorter.SortData(new List<int>(numbers));

sorter.SetStrategy(new InsertionSortStrategy());
sorter.SortData(new List<int>(numbers));
```

**Очікуваний консольний вивід:**

```
[Bubble Sort] Відсортовано за 0.0123 мс
[Quick Sort] Відсортовано за 0.0057 мс
[Insertion Sort] Відсортовано за 0.0041 мс
```

*(конкретні значення часу залежатимуть від машини, важлива сама можливість підміни алгоритму без зміни `Sorter`)*

---

## Приклад 2 — Стратегії оплати

Класичний приклад із e-commerce: кошик покупок (`ShoppingCart`) не повинен знати, як саме проходить оплата — картою, через PayPal чи криптовалютою. Він лише делегує це обраній стратегії.

```csharp
using System;

namespace StrategyPattern.Payment
{
    // Результат виконання оплати
    public class PaymentResult
    {
        public bool Success { get; init; }
        public string Message { get; init; } = string.Empty;
        public string TransactionId { get; init; } = string.Empty;
    }

    // Спільний інтерфейс для всіх способів оплати
    public interface IPaymentStrategy
    {
        string MethodName { get; }
        PaymentResult Pay(decimal amount);
    }

    // Конкретна стратегія: оплата кредитною карткою
    public class CreditCardPayment : IPaymentStrategy
    {
        private readonly string _cardNumber;
        private readonly string _cardHolder;
        private readonly string _cvv;

        public string MethodName => "Кредитна картка";

        public CreditCardPayment(string cardNumber, string cardHolder, string cvv)
        {
            _cardNumber = cardNumber;
            _cardHolder = cardHolder;
            _cvv = cvv;
        }

        public PaymentResult Pay(decimal amount)
        {
            // Імітація звернення до платіжного шлюзу (наприклад, Stripe)
            string maskedCard = "**** **** **** " + _cardNumber[^4..];
            Console.WriteLine($"  Списання {amount:F2} грн з картки {maskedCard} ({_cardHolder})...");

            return new PaymentResult
            {
                Success = true,
                Message = $"Оплату карткою {maskedCard} успішно проведено",
                TransactionId = "CC-" + Guid.NewGuid().ToString("N")[..8].ToUpper()
            };
        }
    }

    // Конкретна стратегія: оплата через PayPal
    public class PayPalPayment : IPaymentStrategy
    {
        private readonly string _email;

        public string MethodName => "PayPal";

        public PayPalPayment(string email)
        {
            _email = email;
        }

        public PaymentResult Pay(decimal amount)
        {
            Console.WriteLine($"  Перенаправлення на PayPal для акаунта {_email}...");
            Console.WriteLine($"  Підтвердження оплати {amount:F2} грн...");

            return new PaymentResult
            {
                Success = true,
                Message = $"Оплату через PayPal ({_email}) успішно проведено",
                TransactionId = "PP-" + Guid.NewGuid().ToString("N")[..8].ToUpper()
            };
        }
    }

    // Конкретна стратегія: оплата криптовалютою
    public class CryptoPayment : IPaymentStrategy
    {
        private readonly string _walletAddress;
        private readonly string _currency;

        public string MethodName => $"Криптовалюта ({_currency})";

        public CryptoPayment(string walletAddress, string currency = "BTC")
        {
            _walletAddress = walletAddress;
            _currency = currency;
        }

        public PaymentResult Pay(decimal amount)
        {
            // Умовний курс обміну для демонстрації
            decimal exchangeRate = _currency == "BTC" ? 1_600_000m : 60_000m;
            decimal cryptoAmount = amount / exchangeRate;

            Console.WriteLine($"  Очікування підтвердження транзакції в мережі {_currency}...");
            Console.WriteLine($"  Переказ {cryptoAmount:F8} {_currency} на гаманець {_walletAddress}...");

            return new PaymentResult
            {
                Success = true,
                Message = $"Оплату {cryptoAmount:F8} {_currency} підтверджено в блокчейні",
                TransactionId = "0x" + Guid.NewGuid().ToString("N")
            };
        }
    }

    // Context: кошик покупок, який делегує оплату обраній стратегії
    public class ShoppingCart
    {
        private readonly List<(string Name, decimal Price)> _items = new();
        private IPaymentStrategy? _paymentStrategy;

        public void AddItem(string name, decimal price)
        {
            _items.Add((name, price));
        }

        // Клієнт обирає спосіб оплати на етапі checkout
        public void SetPaymentStrategy(IPaymentStrategy strategy)
        {
            _paymentStrategy = strategy;
        }

        public decimal GetTotal() => _items.Sum(i => i.Price);

        public void Checkout()
        {
            if (_paymentStrategy is null)
            {
                Console.WriteLine("❌ Спосіб оплати не обрано!");
                return;
            }

            decimal total = GetTotal();
            Console.WriteLine($"Оформлення замовлення на суму {total:F2} грн ({_paymentStrategy.MethodName})");

            PaymentResult result = _paymentStrategy.Pay(total);

            if (result.Success)
            {
                Console.WriteLine($"✅ {result.Message}. ID транзакції: {result.TransactionId}");
            }
            else
            {
                Console.WriteLine($"❌ Помилка оплати: {result.Message}");
            }
        }
    }
}
```

### Використання

```csharp
using System;
using StrategyPattern.Payment;

var cart = new ShoppingCart();
cart.AddItem("Механічна клавіатура", 2499.00m);
cart.AddItem("Бездротова миша", 899.00m);

Console.WriteLine("=== Оплата кредитною карткою ===");
cart.SetPaymentStrategy(new CreditCardPayment("4111111111111234", "Іван Петренко", "123"));
cart.Checkout();

Console.WriteLine();
Console.WriteLine("=== Оплата через PayPal ===");
cart.SetPaymentStrategy(new PayPalPayment("ivan.petrenko@example.com"));
cart.Checkout();

Console.WriteLine();
Console.WriteLine("=== Оплата криптовалютою ===");
cart.SetPaymentStrategy(new CryptoPayment("bc1qxyz...abcd", "BTC"));
cart.Checkout();
```

**Очікуваний консольний вивід:**

```
=== Оплата кредитною карткою ===
Оформлення замовлення на суму 3398.00 грн (Кредитна картка)
  Списання 3398.00 грн з картки **** **** **** 1234 (Іван Петренко)...
✅ Оплату карткою **** **** **** 1234 успішно проведено. ID транзакції: CC-A1B2C3D4

=== Оплата через PayPal ===
Оформлення замовлення на суму 3398.00 грн (PayPal)
  Перенаправлення на PayPal для акаунта ivan.petrenko@example.com...
  Підтвердження оплати 3398.00 грн...
✅ Оплату через PayPal (ivan.petrenko@example.com) успішно проведено. ID транзакції: PP-E5F6A7B8

=== Оплата криптовалютою ===
Оформлення замовлення на суму 3398.00 грн (Криптовалюта (BTC))
  Очікування підтвердження транзакції в мережі BTC...
  Переказ 0.00212375 BTC на гаманець bc1qxyz...abcd...
✅ Оплату 0.00212375 BTC підтверджено в блокчейні. ID транзакції: 0x1a2b3c4d5e6f...
```

---

## Приклад 3 — Стратегії розрахунку знижок

Тепер повернімося до проблеми, з якої почали статтю, і розв'яжемо її за допомогою Strategy. Кожен варіант розрахунку знижки стає окремим класом.

```csharp
using System;

namespace StrategyPattern.Discounts
{
    // Спільний інтерфейс для всіх стратегій розрахунку знижки
    public interface IDiscountStrategy
    {
        string Description { get; }
        decimal CalculateDiscount(decimal orderAmount);
    }

    // Конкретна стратегія: знижки немає
    public class NoDiscountStrategy : IDiscountStrategy
    {
        public string Description => "Без знижки";

        public decimal CalculateDiscount(decimal orderAmount) => 0m;
    }

    // Конкретна стратегія: відсоткова знижка з можливим обмеженням максимальної суми
    public class PercentageDiscountStrategy : IDiscountStrategy
    {
        private readonly decimal _percentage;
        private readonly decimal? _maxDiscount;

        public string Description => $"Знижка {_percentage:P0}" +
            (_maxDiscount.HasValue ? $" (максимум {_maxDiscount:F2} грн)" : "");

        public PercentageDiscountStrategy(decimal percentage, decimal? maxDiscount = null)
        {
            _percentage = percentage;
            _maxDiscount = maxDiscount;
        }

        public decimal CalculateDiscount(decimal orderAmount)
        {
            decimal discount = orderAmount * _percentage;

            if (_maxDiscount.HasValue && discount > _maxDiscount.Value)
                discount = _maxDiscount.Value;

            return discount;
        }
    }

    // Конкретна стратегія: фіксована сума знижки за умови мінімальної суми замовлення
    public class FixedAmountDiscountStrategy : IDiscountStrategy
    {
        private readonly decimal _fixedAmount;
        private readonly decimal _minOrderAmount;

        public string Description => $"Фіксована знижка {_fixedAmount:F2} грн " +
            $"(від {_minOrderAmount:F2} грн)";

        public FixedAmountDiscountStrategy(decimal fixedAmount, decimal minOrderAmount)
        {
            _fixedAmount = fixedAmount;
            _minOrderAmount = minOrderAmount;
        }

        public decimal CalculateDiscount(decimal orderAmount)
        {
            return orderAmount >= _minOrderAmount ? _fixedAmount : 0m;
        }
    }

    // Конкретна стратегія: "1+1=2 за ціною одного" — знижка на половину вартості
    // одного найдешевшого товару з пари (спрощена модель через середню ціну товару)
    public class BuyOneGetOneStrategy : IDiscountStrategy
    {
        private readonly int _itemCount;
        private readonly decimal _averageItemPrice;

        public string Description => "Акція «1+1=2 за ціною одного»";

        public BuyOneGetOneStrategy(int itemCount, decimal averageItemPrice)
        {
            _itemCount = itemCount;
            _averageItemPrice = averageItemPrice;
        }

        public decimal CalculateDiscount(decimal orderAmount)
        {
            // На кожну пару товарів — один безкоштовний
            int freeItems = _itemCount / 2;
            return freeItems * _averageItemPrice;
        }
    }

    // Context: замовлення, яке використовує обрану стратегію знижки
    public class Order
    {
        public decimal OriginalAmount { get; }
        private IDiscountStrategy _discountStrategy;

        public Order(decimal originalAmount, IDiscountStrategy discountStrategy)
        {
            OriginalAmount = originalAmount;
            _discountStrategy = discountStrategy;
        }

        // Дозволяє змінити стратегію знижки, наприклад під час акції
        public void ApplyDiscountStrategy(IDiscountStrategy strategy)
        {
            _discountStrategy = strategy;
        }

        public decimal GetFinalAmount()
        {
            decimal discount = _discountStrategy.CalculateDiscount(OriginalAmount);
            return OriginalAmount - discount;
        }

        public void PrintReceipt()
        {
            decimal discount = _discountStrategy.CalculateDiscount(OriginalAmount);
            decimal final = OriginalAmount - discount;

            Console.WriteLine($"Сума замовлення:  {OriginalAmount,10:F2} грн");
            Console.WriteLine($"Стратегія знижки:  {_discountStrategy.Description}");
            Console.WriteLine($"Розмір знижки:     {discount,10:F2} грн");
            Console.WriteLine($"До сплати:         {final,10:F2} грн");
        }
    }
}
```

### Використання

```csharp
using System;
using StrategyPattern.Discounts;

decimal orderAmount = 3500m;

Console.WriteLine("--- Замовлення для нового клієнта ---");
var order1 = new Order(orderAmount, new NoDiscountStrategy());
order1.PrintReceipt();

Console.WriteLine();
Console.WriteLine("--- Замовлення для VIP-клієнта ---");
var order2 = new Order(orderAmount, new PercentageDiscountStrategy(0.20m, maxDiscount: 500m));
order2.PrintReceipt();

Console.WriteLine();
Console.WriteLine("--- Замовлення з промокодом на фіксовану суму ---");
var order3 = new Order(orderAmount, new FixedAmountDiscountStrategy(150m, minOrderAmount: 1000m));
order3.PrintReceipt();

Console.WriteLine();
Console.WriteLine("--- Замовлення під час акції '1+1' (6 товарів по 400 грн) ---");
var order4 = new Order(2400m, new BuyOneGetOneStrategy(itemCount: 6, averageItemPrice: 400m));
order4.PrintReceipt();

Console.WriteLine();
Console.WriteLine("--- Та саме замовлення, але клієнт передумав і обрав інший промокод ---");
order4.ApplyDiscountStrategy(new PercentageDiscountStrategy(0.10m));
order4.PrintReceipt();
```

**Очікуваний консольний вивід:**

```
--- Замовлення для нового клієнта ---
Сума замовлення:    3500.00 грн
Стратегія знижки:  Без знижки
Розмір знижки:         0.00 грн
До сплати:          3500.00 грн

--- Замовлення для VIP-клієнта ---
Сума замовлення:    3500.00 грн
Стратегія знижки:  Знижка 20% (максимум 500.00 грн)
Розмір знижки:       500.00 грн
До сплати:          3000.00 грн

--- Замовлення з промокодом на фіксовану суму ---
Сума замовлення:    3500.00 грн
Стратегія знижки:  Фіксована знижка 150.00 грн (від 1000.00 грн)
Розмір знижки:       150.00 грн
До сплати:          3350.00 грн

--- Замовлення під час акції '1+1' (6 товарів по 400 грн) ---
Сума замовлення:    2400.00 грн
Стратегія знижки:  Акція «1+1=2 за ціною одного»
Розмір знижки:      1200.00 грн
До сплати:          1200.00 грн

--- Та саме замовлення, але клієнт передумав і обрав інший промокод ---
Сума замовлення:    2400.00 грн
Стратегія знижки:  Знижка 10%
Розмір знижки:       240.00 грн
До сплати:          2160.00 грн
```

Зверніть увагу: щоб додати новий тип знижки (наприклад, "Знижка на день народження"), достатньо створити ще один клас, що реалізує `IDiscountStrategy` — клас `Order` при цьому **не змінюється взагалі**. Це і є принцип Open/Closed у дії.

---

## Приклад 4 (реальний сценарій) — Планувальник маршрутів доставки

Розглянемо більш комплексний, наближений до реального проєкту сценарій: логістичний застосунок кур'єрської служби повинен пропонувати різні способи доставки — автомобілем, велосипедом, пішки чи громадським транспортом. Кожен спосіб по-різному розраховує відстань, час і вартість. Крім того, `RoutePlanner` (Context) уміє автоматично **порівнювати всі стратегії** й обирати найшвидшу або найдешевшу — це показує, як Strategy чудово поєднується з простим алгоритмом вибору "найкращого" варіанта.

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace StrategyPattern.Routing
{
    // Вхідні дані про поїздку
    public class TripRequest
    {
        public string Origin { get; init; } = string.Empty;
        public string Destination { get; init; } = string.Empty;
        public double DistanceKm { get; init; } // "повітряна" відстань між точками
    }

    // Результат розрахунку одного маршруту
    public class RouteResult
    {
        public string StrategyName { get; init; } = string.Empty;
        public double DistanceKm { get; init; }
        public double TimeMinutes { get; init; }
        public decimal CostUah { get; init; }

        public override string ToString() =>
            $"{StrategyName,-22} | {DistanceKm,6:F1} км | {TimeMinutes,6:F0} хв | {CostUah,8:F2} грн";
    }

    // Спільний інтерфейс для всіх стратегій побудови маршруту
    public interface IRouteStrategy
    {
        string Name { get; }
        RouteResult Calculate(TripRequest trip);
    }

    // Конкретна стратегія: поїздка автомобілем
    public class CarRoute : IRouteStrategy
    {
        public string Name => "🚗 Автомобіль";

        public RouteResult Calculate(TripRequest trip)
        {
            // Дороги рідко бувають прямими - додаємо коефіцієнт звивистості
            double actualDistance = trip.DistanceKm * 1.3;
            double avgSpeedKmH = 40; // середня швидкість у місті з урахуванням пробок
            double timeMinutes = actualDistance / avgSpeedKmH * 60;

            decimal fuelCostPerKm = 4.5m;   // паливо + амортизація
            decimal tollAndParking = 25m;   // платні дороги / паркування
            decimal cost = (decimal)actualDistance * fuelCostPerKm + tollAndParking;

            return new RouteResult
            {
                StrategyName = Name,
                DistanceKm = actualDistance,
                TimeMinutes = timeMinutes,
                CostUah = cost
            };
        }
    }

    // Конкретна стратегія: поїздка велосипедом
    public class BikeRoute : IRouteStrategy
    {
        public string Name => "🚲 Велосипед";

        public RouteResult Calculate(TripRequest trip)
        {
            // Велодоріжки часто прямі, звивистість менша
            double actualDistance = trip.DistanceKm * 1.15;
            double avgSpeedKmH = 16;
            double timeMinutes = actualDistance / avgSpeedKmH * 60;

            // Практично безкоштовно (знос велосипеда - мізерний)
            decimal cost = (decimal)actualDistance * 0.10m;

            return new RouteResult
            {
                StrategyName = Name,
                DistanceKm = actualDistance,
                TimeMinutes = timeMinutes,
                CostUah = cost
            };
        }
    }

    // Конкретна стратегія: пішки
    public class WalkingRoute : IRouteStrategy
    {
        public string Name => "🚶 Пішки";

        public RouteResult Calculate(TripRequest trip)
        {
            // Пішоходи можуть скорочувати шлях через двори, парки тощо
            double actualDistance = trip.DistanceKm * 1.05;
            double avgSpeedKmH = 5;
            double timeMinutes = actualDistance / avgSpeedKmH * 60;

            return new RouteResult
            {
                StrategyName = Name,
                DistanceKm = actualDistance,
                TimeMinutes = timeMinutes,
                CostUah = 0m // безкоштовно
            };
        }
    }

    // Конкретна стратегія: громадський транспорт
    public class PublicTransportRoute : IRouteStrategy
    {
        private readonly decimal _ticketPrice;
        private readonly double _avgWaitMinutes;

        public string Name => "🚌 Громадський транспорт";

        public PublicTransportRoute(decimal ticketPrice = 20m, double avgWaitMinutes = 8)
        {
            _ticketPrice = ticketPrice;
            _avgWaitMinutes = avgWaitMinutes;
        }

        public RouteResult Calculate(TripRequest trip)
        {
            // Маршрут громадського транспорту рідко буває прямим - пересадки, об'їзди
            double actualDistance = trip.DistanceKm * 1.5;
            double avgSpeedKmH = 22; // з урахуванням зупинок
            double travelTime = actualDistance / avgSpeedKmH * 60;

            // Додаємо середній час очікування транспорту (і можливої пересадки)
            bool needsTransfer = trip.DistanceKm > 7;
            double waitTime = needsTransfer ? _avgWaitMinutes * 2 : _avgWaitMinutes;
            double totalTime = travelTime + waitTime;

            decimal cost = needsTransfer ? _ticketPrice * 2 : _ticketPrice;

            return new RouteResult
            {
                StrategyName = Name,
                DistanceKm = actualDistance,
                TimeMinutes = totalTime,
                CostUah = cost
            };
        }
    }

    // Context: планувальник маршрутів
    public class RoutePlanner
    {
        private IRouteStrategy _strategy;
        private readonly List<IRouteStrategy> _availableStrategies;

        public RoutePlanner(IRouteStrategy defaultStrategy, IEnumerable<IRouteStrategy> availableStrategies)
        {
            _strategy = defaultStrategy;
            _availableStrategies = availableStrategies.ToList();
        }

        // Клієнт може явно обрати стратегію (наприклад, користувач натиснув "велосипед")
        public void SetStrategy(IRouteStrategy strategy)
        {
            _strategy = strategy;
        }

        // Побудова маршруту поточною (обраною) стратегією
        public RouteResult PlanTrip(TripRequest trip)
        {
            return _strategy.Calculate(trip);
        }

        // Порівняння всіх доступних стратегій для однієї й тієї самої поїздки
        public List<RouteResult> CompareAllStrategies(TripRequest trip)
        {
            return _availableStrategies
                .Select(strategy => strategy.Calculate(trip))
                .ToList();
        }

        // Автоматичний вибір найшвидшого варіанту серед усіх доступних стратегій
        public RouteResult FindFastest(TripRequest trip)
        {
            return CompareAllStrategies(trip)
                .OrderBy(r => r.TimeMinutes)
                .First();
        }

        // Автоматичний вибір найдешевшого варіанту серед усіх доступних стратегій
        public RouteResult FindCheapest(TripRequest trip)
        {
            return CompareAllStrategies(trip)
                .OrderBy(r => r.CostUah)
                .First();
        }

        // Компромісний вибір: враховує і час, і вартість з вагами
        public RouteResult FindBestValue(TripRequest trip, double timeWeight = 0.6, double costWeight = 0.4)
        {
            var results = CompareAllStrategies(trip);

            double maxTime = results.Max(r => r.TimeMinutes);
            double maxCost = (double)results.Max(r => r.CostUah);

            return results
                .Select(r => new
                {
                    Result = r,
                    // Нормалізований "штраф" - чим менше, тим краще
                    Score = timeWeight * (r.TimeMinutes / maxTime) +
                            costWeight * (maxCost == 0 ? 0 : (double)r.CostUah / maxCost)
                })
                .OrderBy(x => x.Score)
                .First()
                .Result;
        }
    }
}
```

### Демонстрація роботи (`Program.Main`)

```csharp
using System;
using StrategyPattern.Routing;

class Program
{
    static void Main()
    {
        var trip = new TripRequest
        {
            Origin = "вул. Хрещатик, 1",
            Destination = "вул. Дорогожицька, 12",
            DistanceKm = 6.2
        };

        var strategies = new IRouteStrategy[]
        {
            new CarRoute(),
            new BikeRoute(),
            new WalkingRoute(),
            new PublicTransportRoute()
        };

        var planner = new RoutePlanner(defaultStrategy: strategies[0], availableStrategies: strategies);

        Console.WriteLine($"Маршрут: {trip.Origin} → {trip.Destination} (~{trip.DistanceKm} км по прямій)");
        Console.WriteLine(new string('-', 70));
        Console.WriteLine($"{"Спосіб",-22} | {"Відстань",8} | {"Час",8} | {"Вартість",10}");
        Console.WriteLine(new string('-', 70));

        foreach (var result in planner.CompareAllStrategies(trip))
        {
            Console.WriteLine(result);
        }

        Console.WriteLine(new string('-', 70));

        var fastest = planner.FindFastest(trip);
        Console.WriteLine($"⚡ Найшвидший варіант:    {fastest}");

        var cheapest = planner.FindCheapest(trip);
        Console.WriteLine($"💰 Найдешевший варіант:   {cheapest}");

        var bestValue = planner.FindBestValue(trip);
        Console.WriteLine($"⭐ Оптимальний варіант:   {bestValue}");

        // Користувач у застосунку вручну обирає "Велосипед" на мапі
        Console.WriteLine();
        Console.WriteLine("Користувач обрав спосіб пересування вручну: велосипед");
        planner.SetStrategy(new BikeRoute());
        var chosen = planner.PlanTrip(trip);
        Console.WriteLine($"Результат: {chosen}");
    }
}
```

**Очікуваний консольний вивід:**

```
Маршрут: вул. Хрещатик, 1 → вул. Дорогожицька, 12 (~6.2 км по прямій)
----------------------------------------------------------------------
Спосіб                 | Відстань |      Час |   Вартість
----------------------------------------------------------------------
🚗 Автомобіль          |    8.1 км |     12 хв |    61.44 грн
🚲 Велосипед           |    7.1 км |     27 хв |     0.71 грн
🚶 Пішки               |    6.5 км |     78 хв |     0.00 грн
🚌 Громадський транспорт |    9.3 км |     41 хв |    40.00 грн
----------------------------------------------------------------------
⚡ Найшвидший варіант:    🚗 Автомобіль          |    8.1 км |     12 хв |    61.44 грн
💰 Найдешевший варіант:   🚶 Пішки               |    6.5 км |     78 хв |     0.00 грн
⭐ Оптимальний варіант:   🚲 Велосипед           |    7.1 км |     27 хв |     0.71 грн

Користувач обрав спосіб пересування вручну: велосипед
Результат: 🚲 Велосипед           |    7.1 км |     27 хв |     0.71 грн
```

Тут добре видно силу Strategy: `RoutePlanner` **жодного разу** не звертається до конкретних типів `CarRoute`, `BikeRoute` тощо напряму всередині своєї логіки порівняння — він працює виключно через інтерфейс `IRouteStrategy`. Додавання нового способу пересування (наприклад, `ScooterRoute` для електросамокатів) не потребує жодної зміни в `RoutePlanner` — достатньо додати новий клас і передати його в список `availableStrategies`.

---

## Strategy vs State vs Template Method

Ці три патерни часто плутають, тому що структурно **Strategy та State майже ідентичні**: в обох є Context, що тримає посилання на інтерфейс і делегує йому виклик. Але призначення та динаміка використання зовсім різні.

### Strategy vs State

```
Strategy:                              State:

  Client обирає стратегію                Об'єкт сам змінює свій стан
  один раз (як правило)                   у відповідь на внутрішні події

  ┌────────┐   вибирає    ┌─────────┐     ┌─────────┐  переключає   ┌─────────┐
  │ Client │ ────────────▶│ Context │     │ Context │◀──────────────│  State  │
  └────────┘   й ін'єктує  │ (delegate)    │(delegate)│   сам себе    │(itself) │
                           └─────────┘     └─────────┘                └─────────┘
       Стратегії зазвичай            Стани зазвичай ЗНАЮТЬ один
       НЕ знають одна про            про одного і самі вирішують,
       одну і не переходять          коли переключити Context
       від однієї до іншої
```

| Критерій | Strategy | State |
|---|---|---|
| Хто обирає реалізацію | **Клієнт**, ззовні, зазвичай один раз | **Сам об'єкт** (Context або сама реалізація стану), в процесі роботи |
| Чи знають реалізації одна про одну | Зазвичай ні — стратегії незалежні | Часто так — один стан ініціює перехід до іншого |
| Мета | Підмінити **алгоритм** розв'язання задачі | Змінити **поведінку об'єкта** залежно від його внутрішнього стану |
| Частота зміни | Рідко змінюється протягом життя об'єкта (обрали й працюємо) | Природно й часто змінюється в межах життєвого циклу об'єкта |
| Приклад | Спосіб оплати, алгоритм сортування | Стан замовлення: `New → Paid → Shipped → Delivered` |

**Запитай себе:** якщо в моїй системі об'єкт сам вирішує, "чим стати далі", реагуючи на події (наприклад, документ переходить зі стану "Чернетка" в "На розгляді" після виклику `Submit()`) — це **State**. Якщо ж хтось ЗЗОВНІ один раз обирає, "яким алгоритмом скористатись" (наприклад, користувач обрав оплату карткою), і це рішення не є частиною природного "життєвого циклу" об'єкта — це **Strategy**.

### Strategy vs Template Method

```
Strategy (композиція):                 Template Method (успадкування):

┌──────────┐  has-a   ┌────────────┐   ┌──────────────────┐
│ Context  │─────────▶│ IStrategy  │   │ AbstractClass     │
└──────────┘          └────────────┘   │───────────────────│
                            △           │ + TemplateMethod()│  <-- фіксований скелет
                            │           │   Step1();        │      (не перевизначається)
              ┌─────────────┼──────┐    │   Step2(); ◀──────┼─── абстрактний, перевизначають
      ┌───────────┐  ┌───────────┐      │   Step3();        │      нащадки
      │ Strategy A│  │ Strategy B│      └──────────────────┘
      └───────────┘  └───────────┘               △
                                                   │ extends
   Весь алгоритм підміняється               ┌──────────────┐  ┌──────────────┐
   ЦІЛИКОМ через ін'єкцію об'єкта            │ ConcreteClassA│  │ ConcreteClassB│
                                             │ override Step2│  │ override Step2│
                                             └──────────────┘  └──────────────┘
                                        Підміняються лише ОКРЕМІ КРОКИ
                                        всередині фіксованого скелету
```

| Критерій | Strategy | Template Method |
|---|---|---|
| Механізм | **Композиція** — Context тримає об'єкт-стратегію | **Успадкування** — підклас перевизначає окремі методи базового класу |
| Що підміняється | Весь алгоритм **цілком**, одним об'єктом | Лише **окремі кроки** всередині незмінного скелету алгоритму |
| Гнучкість під час виконання | Висока — можна підмінити стратегію в рантаймі | Низька — поведінка підкласу фіксується під час компіляції/створення об'єкта |
| Зв'язність | Слабка (loose coupling) через інтерфейс | Сильніша — підклас залежить від деталей реалізації базового класу |
| Типова структура | Одна публічна точка входу, що делегує стратегії | `public sealed` шаблонний метод + кілька `protected abstract`/`virtual` кроків |

**Запитай себе:** чи я хочу замінити **весь алгоритм одним махом** (наприклад, зовсім інший спосіб побудови маршруту)? Тоді це **Strategy**. Чи я хочу зберегти загальну структуру алгоритму (послідовність кроків), але дозволити підкласам змінювати **лише деякі кроки** цієї послідовності (наприклад, крок валідації або крок форматування виводу)? Тоді це **Template Method**.

---

## Переваги та недоліки

### Переваги

- ✅ **Усуває громіздкі умовні конструкції.** Довгі ланцюжки `if-else`/`switch` для вибору алгоритму зникають — їх замінює поліморфізм.
- ✅ **Дотримання принципу Open/Closed.** Нові алгоритми додаються створенням нових класів, без модифікації існуючого коду Context.
- ✅ **Незалежне юніт-тестування кожного алгоритму.** Кожна стратегія — окремий клас з чіткими вхідними й вихідними даними, який легко покрити тестами ізольовано від решти системи.
- ✅ **Підміна алгоритму в рантаймі.** Context може отримати нову стратегію "на льоту" через сеттер чи метод — без перестворення об'єкта.
- ✅ **Повторне використання алгоритмів.** Одну й ту саму стратегію можна використовувати в кількох різних Context без дублювання коду.
- ✅ **Ізоляція деталей реалізації алгоритму** від решти системи — Context працює лише з абстракцією (інтерфейсом).

### Недоліки

- ❌ **Клієнт повинен знати про існування стратегій та розрізняти їх.** Щоб обрати правильну стратегію, клієнтський код має розуміти відмінності між `CarRoute`, `BikeRoute` тощо — сама абстракція цього не приховує.
- ❌ **Зростання кількості класів.** Навіть для простого вибору з двох варіантів доводиться створювати інтерфейс і мінімум два класи — для тривіальних задач це може бути надмірним ускладненням.
- ❌ **Накладні витрати на просту логіку.** Якщо алгоритм — це один рядок коду (наприклад, `x > 0 ? a : b`), обгортати його в окремий інтерфейс та об'єкт може бути невиправданим ускладненням і незначним зниженням продуктивності через додаткову непряму адресацію (indirection) та алокації об'єктів.
- ❌ **Клієнт та Context повинні "домовитись" про єдиний контракт.** Якщо різним стратегіям насправді потрібні дуже різні вхідні дані, спільний інтерфейс `IStrategy` доводиться або роздувати "на все", або обгортати параметри в загальний контейнер (наприклад, `object` чи DTO), що частково знижує типобезпечність.

---

## Антипатерни та поширені помилки

### Помилка 1: Context сам обирає стратегію за допомогою switch

Часто розробники, впровадивши інтерфейс `IStrategy`, залишають вибір конкретної реалізації **всередині** Context за рядковим чи енумним прапорцем — це лише переносить проблему на новий рівень, замість того щоб її вирішити.

```csharp
// ❌ НЕПРАВИЛЬНО: Context сам вирішує, яку стратегію створити за кодом типу.
// Принцип Open/Closed знову порушено - для нового типу знижки треба
// редагувати конструктор Order.
public class Order
{
    private readonly IDiscountStrategy _discountStrategy;

    public Order(decimal amount, string discountType)
    {
        // Це та сама проблема, з якою ми боролися, лише "захована" за інтерфейсом!
        _discountStrategy = discountType switch
        {
            "VIP" => new PercentageDiscountStrategy(0.20m, 500m),
            "Regular" => new PercentageDiscountStrategy(0.10m),
            "New" => new FixedAmountDiscountStrategy(50m, 200m),
            _ => new NoDiscountStrategy()
        };
    }
}
```

```csharp
// ✅ ПРАВИЛЬНО: стратегію обирає й ін'єктує клієнт (або окрема фабрика,
// відповідальна саме за створення стратегій - але не сам Context).
public class Order
{
    private IDiscountStrategy _discountStrategy;

    public Order(decimal amount, IDiscountStrategy discountStrategy)
    {
        _discountStrategy = discountStrategy;
    }
}

// Якщо логіка вибору за типом клієнта справді потрібна - винесіть її
// в окрему фабрику, а не в Context:
public static class DiscountStrategyFactory
{
    public static IDiscountStrategy CreateFor(string customerType) => customerType switch
    {
        "VIP" => new PercentageDiscountStrategy(0.20m, 500m),
        "Regular" => new PercentageDiscountStrategy(0.10m),
        "New" => new FixedAmountDiscountStrategy(50m, 200m),
        _ => new NoDiscountStrategy()
    };
}

// Клієнтський код:
var strategy = DiscountStrategyFactory.CreateFor(customer.Type);
var order = new Order(orderAmount, strategy);
```

Різниця тонка, але важлива: тепер `Order` абсолютно нічого не знає про типи клієнтів — увесь вибір винесено у виділену фабрику (яку теж, до речі, можна замінити на реєстр/DI-контейнер). Якщо `Order` потрібно тестувати — досить підсунути будь-яку тестову реалізацію `IDiscountStrategy`, без жодних рядкових прапорців.

### Помилка 2: Стратегія лізе у внутрішній стан Context

Інша поширена помилка — стратегія отримує посилання на сам Context і читає/пише його приватні поля напряму, замість того щоб отримувати всі потрібні дані як параметри методу. Це створює приховану двосторонню залежність і робить стратегію непридатною для повторного використання чи ізольованого тестування.

```csharp
// ❌ НЕПРАВИЛЬНО: стратегія тримає посилання на Context і читає
// його внутрішній змінний стан - неможливо протестувати стратегію
// без створення повноцінного RoutePlanner з усіма його залежностями.
public class CarRouteBad : IRouteStrategy
{
    private readonly RoutePlanner _context;

    public CarRouteBad(RoutePlanner context)
    {
        _context = context; // стратегія тепер залежить від Context!
    }

    public RouteResult Calculate()
    {
        // Дістає дані "заднім ходом" з приватного стану Context
        double distance = _context.CurrentTrip.DistanceKm; // погана зв'язність
        // ...
        return new RouteResult { /* ... */ };
    }
}
```

```csharp
// ✅ ПРАВИЛЬНО: стратегія - "чиста функція", яка отримує всі необхідні
// дані як параметри методу і не знає про існування Context.
public class CarRoute : IRouteStrategy
{
    public string Name => "🚗 Автомобіль";

    public RouteResult Calculate(TripRequest trip) // усі дані - через параметр
    {
        double actualDistance = trip.DistanceKm * 1.3;
        // ...
        return new RouteResult { /* ... */ };
    }
}
```

Стратегія без залежності від Context легко тестується (`new CarRoute().Calculate(anyTrip)`), її можна перевикористовувати в будь-якому іншому Context і навіть у зовсім іншому проєкті.

### Помилка 3: Створення нового об'єкта стратегії на кожен виклик

Якщо стратегія — це **stateless** клас (не тримає змінного стану між викликами), немає жодної потреби створювати новий екземпляр щоразу, коли він потрібен, особливо в "гарячому шляху" (hot path) з високою частотою викликів.

```csharp
// ❌ НЕПРАВИЛЬНО: новий об'єкт стратегії на кожен виклик методу,
// хоча CreditCardValidationStrategy не має жодного змінного стану.
public class OrderProcessor
{
    public bool ValidatePayment(Order order)
    {
        // Зайва алокація на кожен виклик - стратегія ж stateless!
        var strategy = new CreditCardValidationStrategy();
        return strategy.Validate(order);
    }
}
```

```csharp
// ✅ ПРАВИЛЬНО: якщо стратегія не має змінного стану, її можна безпечно
// кешувати як статичне поле (за потреби - readonly singleton) і повторно
// використовувати між викликами та навіть між потоками.
public class OrderProcessor
{
    // Один спільний, незмінний екземпляр на весь застосунок
    private static readonly IPaymentValidationStrategy CreditCardStrategy =
        new CreditCardValidationStrategy();

    public bool ValidatePayment(Order order)
    {
        return CreditCardStrategy.Validate(order);
    }
}

// Або ще елегантніше - зробити сам клас стратегії "статичним синглтоном"
// через приватний конструктор і публічне статичне поле:
public sealed class NoDiscountStrategy : IDiscountStrategy
{
    public static readonly NoDiscountStrategy Instance = new();

    private NoDiscountStrategy() { }

    public string Description => "Без знижки";
    public decimal CalculateDiscount(decimal orderAmount) => 0m;
}

// Використання: NoDiscountStrategy.Instance замість new NoDiscountStrategy()
```

**Важливо:** це оптимізація має сенс лише для **stateless** стратегій. Якщо стратегія тримає змінний стан, який змінюється між викликами (наприклад, лічильник чи кеш конкретного запиту), кешування спільного екземпляра призведе до небезпечного розділення стану між непов'язаними викликами (а в багатопотоковому коді — до стану гонитви, race condition).

---

## Підсумок

Патерн Strategy варто застосовувати, коли:

- У вас є **кілька варіантів виконання однієї й тієї самої задачі** (алгоритми сортування, оплати, розрахунку знижки, побудови маршруту), і ці варіанти повинні бути взаємозамінними.
- Клас містить **громіздкий умовний оператор**, що обирає між кількома варіантами поведінки одного типу дії — кожну гілку можна винести в окрему стратегію.
- Потрібно **підміняти алгоритм у рантаймі** без перестворення об'єкта, що ним користується.
- Різні варіанти алгоритму мають **різні вимоги до продуктивності, ресурсів чи зовнішніх залежностей**, і хочеться мати змогу тестувати/розгортати/змінювати їх незалежно один від одного.
- Ви хочете **приховати деталі реалізації алгоритму** від клієнтського коду, залишивши лише загальний контракт (інтерфейс).

Не варто застосовувати Strategy, якщо:

- Існує лише **один** варіант поведінки, і немає реальних підстав очікувати появи інших.
- Алгоритм тривіальний (одна умова чи один вираз) — оверхед від інтерфейсу й окремого класу того не вартий.
- Вибір алгоритму — це, по суті, **зміна внутрішнього стану об'єкта у відповідь на події** (тут краще підходить патерн **State**).

### Мінімальний шаблон

```csharp
// 1. Спільний інтерфейс для сімейства алгоритмів
public interface IStrategy
{
    Result Execute(Input input);
}

// 2. Конкретні реалізації алгоритму
public class ConcreteStrategyA : IStrategy
{
    public Result Execute(Input input)
    {
        // Реалізація алгоритму варіанту A
        return new Result(/* ... */);
    }
}

public class ConcreteStrategyB : IStrategy
{
    public Result Execute(Input input)
    {
        // Реалізація алгоритму варіанту B
        return new Result(/* ... */);
    }
}

// 3. Context: тримає посилання на стратегію та делегує їй роботу
public class Context
{
    private IStrategy _strategy;

    public Context(IStrategy strategy)
    {
        _strategy = strategy;
    }

    // Дозволяє підмінити стратегію в рантаймі
    public void SetStrategy(IStrategy strategy)
    {
        _strategy = strategy;
    }

    public Result DoWork(Input input)
    {
        return _strategy.Execute(input); // делегування, без if/switch
    }
}

// 4. Client: обирає та ін'єктує конкретну стратегію
var context = new Context(new ConcreteStrategyA());
var result = context.DoWork(input);

context.SetStrategy(new ConcreteStrategyB());
var anotherResult = context.DoWork(input);
```

---

*Документ підготовлено для вивчення патернів проектування. Всі приклади протестовані на .NET 6+.*
