# Гайд з підготовки — Патерни проектування + Мережі + Бази даних

> **Мета:** єдина точка входу по всьому матеріалу репозиторію.  
> **Контекст:** підготовка до технічної співбесіди в компанії, що спеціалізується на **мережевому моніторингу та управлінні (Network Management)** — SolarWinds — плюс повноцінна серія з баз даних до сіньйорського рівня.  
> **Загальний обсяг:** 23 патерни проектування (GoF, C#) + 10 детальних розборів мережевих технологій + 16 документів з баз даних (реляційні + NoSQL) ≈ 44 000+ рядків матеріалу.

---

## Зміст

1. [Як користуватися цим гайдом](#як-користуватися-цим-гайдом)
2. [Структура репозиторію](#структура-репозиторію)
3. [Частина I — Патерни проектування (23)](#частина-i--патерни-проектування-23)
   - [Породжуючі (Creational) — 5](#породжуючі-creational--5)
   - [Структурні (Structural) — 7](#структурні-structural--7)
   - [Поведінкові (Behavioral) — 10](#поведінкові-behavioral--10)
4. [Частина II — Комп'ютерні мережі (10)](#частина-ii--компютерні-мережі-10)
5. [Частина III — Бази даних (16)](#частина-iii--бази-даних-16)
   - [Реляційні (Relational) — 9](#реляційні-relational--9)
   - [NoSQL — 7](#nosql--7)
6. [Рекомендований план підготовки](#рекомендований-план-підготовки)
7. [Наскрізні теми, що з'являються в кількох файлах](#наскрізні-теми-що-зявляються-в-кількох-файлах)
8. [Швидкий чекліст перед співбесідою](#швидкий-чекліст-перед-співбесідою)

---

## Як користуватися цим гайдом

Кожен файл у репозиторії — це **самодостатній, глибокий розбір однієї теми**: означення, аналогія з реального життя, розбір проблеми "без патерну/без розуміння", кілька прикладів коду на C#, реальний сценарій, поширені помилки, і — для мережевих файлів — окремий блок **"Питання та відповіді для співбесіди"** (12-20 готових Q&A на файл).

Цей гайд **не дублює** той контент. Його завдання:
- дати швидку навігацію ("що взагалі є в репозиторії і де це шукати"),
- дати **пріоритизацію** — що вчити першим, враховуючи специфіку компанії,
- зібрати наскрізні зв'язки між файлами, які легко пропустити, читаючи файли по одному.

---

## Структура репозиторію

```
патерни/
├── study-guide.md                      ← цей файл
│
├── Породжуючі/                         (Creational patterns)
│   ├── singleton-pattern-csharp.md
│   ├── factory-method-pattern-csharp.md
│   ├── abstract-factory-pattern-csharp.md
│   ├── builder-pattern-csharp.md
│   └── prototype-pattern-csharp.md
│
├── Структурні/                         (Structural patterns)
│   ├── adapter-pattern-csharp.md
│   ├── bridge-pattern-csharp.md
│   ├── composite-pattern-csharp.md
│   ├── decorator-pattern-csharp.md
│   ├── facade-pattern-csharp.md
│   ├── flyweight-pattern-csharp.md
│   └── proxy-pattern-csharp.md
│
├── Поведінкові/                        (Behavioral patterns)
│   ├── chain-of-responsibility-pattern-csharp.md
│   ├── command-pattern-csharp.md
│   ├── iterator-pattern-csharp.md
│   ├── mediator-pattern-csharp.md
│   ├── memento-pattern-csharp.md
│   ├── observer-pattern-csharp.md
│   ├── state-pattern-csharp.md
│   ├── strategy-pattern-csharp.md
│   ├── template-method-pattern-csharp.md
│   └── visitor-pattern-csharp.md
│
├── Мережі/                             (Networking fundamentals)
│   ├── osi-tcp-ip-model.md
│   ├── tcp-vs-udp.md
│   ├── ip-addressing-subnetting.md
│   ├── routing-fundamentals.md
│   ├── switching-and-vlans.md
│   ├── snmp-protocol.md                ← найважливіший файл для SolarWinds
│   ├── syslog-and-netflow-monitoring.md
│   ├── dns-and-dhcp.md
│   ├── network-security-basics.md
│   └── http-https-and-web-protocols.md
│
├── ecommerce-architecture-guide.md     (застосування патернів у реальному e-commerce)
│
└── БазиДаних/                          (Databases: Junior → Senior)
    ├── Реляційні/                      (Relational)
    │   ├── 1-relational-model-and-normalization.md
    │   ├── 2-sql-query-language-deep-dive.md
    │   ├── 3-indexes-and-query-optimization.md
    │   ├── 4-transactions-acid-and-isolation-levels.md
    │   ├── 5-database-storage-engines-internals.md
    │   ├── 6-replication-and-scaling-relational.md
    │   ├── 7-orm-entity-framework-and-dapper.md
    │   ├── 8-backup-recovery-and-high-availability.md
    │   └── 9-distributed-transactions-and-saga.md
    │
    └── NoSQL/
        ├── 1-nosql-overview-and-cap-theorem.md      ← вхідна точка в NoSQL
        ├── 2-document-databases-mongodb.md
        ├── 3-key-value-and-redis.md
        ├── 4-wide-column-stores-cassandra.md
        ├── 5-graph-databases.md
        ├── 6-nosql-data-modeling-and-consistency.md
        └── 7-polyglot-persistence-and-choosing-the-right-database.md  ← капстоун усієї серії БД
```

---

## Частина I — Патерни проектування (23)

### Породжуючі (Creational) — 5

Відповідають на питання **"як створити об'єкт"** так, щоб код не був жорстко прив'язаний до конкретних класів.

| Патерн | Файл | Яку проблему вирішує | Коли використовувати |
|---|---|---|---|
| **Singleton** | `singleton-pattern-csharp.md` | Гарантує рівно один екземпляр класу у всьому додатку + глобальну точку доступу | Спільний стан (конфігурація, логер, кеш), дорогий у створенні об'єкт |
| **Factory Method** | `factory-method-pattern-csharp.md` | Делегує створення об'єкта підкласам через один метод-фабрику замість `new` напряму | Родина схожих об'єктів, потрібно розв'язати створення й використання |
| **Abstract Factory** | `abstract-factory-pattern-csharp.md` | Створює **родини** пов'язаних об'єктів без прив'язки до конкретних класів | Потрібно підміняти цілу узгоджену родину продуктів (напр. крос-платформний UI) |
| **Builder** | `builder-pattern-csharp.md` | Покроково будує складний об'єкт, той самий процес → різні представлення | Багато опціональних параметрів, складна багатоетапна конструкція |
| **Prototype** | `prototype-pattern-csharp.md` | Створює нові об'єкти клонуванням наявного екземпляра | Створення "з нуля" дороге; потрібні копії з невеликими відмінностями |

### Структурні (Structural) — 7

Відповідають на питання **"як компонувати класи/об'єкти у більші структури"**, залишаючи їх гнучкими та ефективними.

| Патерн | Файл | Яку проблему вирішує | Коли використовувати |
|---|---|---|---|
| **Adapter** | `adapter-pattern-csharp.md` | Перетворює інтерфейс одного класу на інший, очікуваний клієнтом | Інтеграція несумісного стороннього/legacy-коду |
| **Bridge** | `bridge-pattern-csharp.md` | Розділяє абстракцію та реалізацію, щоб вони змінювались незалежно | Уникнути "вибуху класів" від кількох незалежних вимірів варіативності |
| **Composite** | `composite-pattern-csharp.md` | Єдине трактування окремого об'єкта і групи об'єктів через спільний інтерфейс (дерево) | Ієрархії "частина-ціле" (файлова система, UI-компоненти, оргструктура) |
| **Decorator** | `decorator-pattern-csharp.md` | Динамічно додає відповідальності об'єкту, альтернатива успадкуванню | Потрібне гнучке комбінування поведінок під час виконання |
| **Facade** | `facade-pattern-csharp.md` | Простий інтерфейс до складної підсистеми з багатьох класів | Спростити роботу клієнта зі складною підсистемою |
| **Flyweight** | `flyweight-pattern-csharp.md` | Ділить спільний (intrinsic) стан між великою кількістю об'єктів, економлячи пам'ять | Мільйони схожих об'єктів (символи тексту, дерева в лісі, кулі в грі) |
| **Proxy** | `proxy-pattern-csharp.md` | Об'єкт-замінник, що контролює доступ до реального об'єкта | Лінива ініціалізація, контроль доступу, кешування, remote-виклики |

### Поведінкові (Behavioral) — 10

Відповідають на питання **"як об'єкти взаємодіють і розподіляють відповідальність"**.

| Патерн | Файл | Яку проблему вирішує | Коли використовувати |
|---|---|---|---|
| **Chain of Responsibility** | `chain-of-responsibility-pattern-csharp.md` | Передає запит по ланцюжку обробників, поки хтось не обробить | Кілька потенційних обробників, треба розв'язати відправника й отримувача |
| **Command** | `command-pattern-csharp.md` | Перетворює запит/дію на об'єкт | Потрібні undo/redo, черги завдань, логування дій |
| **Iterator** | `iterator-pattern-csharp.md` | Послідовний доступ до елементів колекції без розкриття її внутрішньої структури | Уніфікований обхід різних типів колекцій/дерев |
| **Mediator** | `mediator-pattern-csharp.md` | Централізує комунікацію між об'єктами-колегами | Зменшити зв'язність many-to-many між компонентами UI/системи |
| **Memento** | `memento-pattern-csharp.md` | Знімок і відновлення внутрішнього стану об'єкта без порушення інкапсуляції | Undo/rollback (текстовий редактор, save/load гри) |
| **Observer** | `observer-pattern-csharp.md` | Один-до-багатьох залежність з автоматичним сповіщенням при зміні стану | Event-driven системи, pub/sub, UI-біндинги |
| **State** | `state-pattern-csharp.md` | Змінює поведінку об'єкта залежно від внутрішнього стану, ніби змінюється клас | Поведінка залежить від стану з великою кількістю умовних розгалужень |
| **Strategy** | `strategy-pattern-csharp.md` | Родина взаємозамінних алгоритмів | Потрібно підміняти алгоритм під час виконання |
| **Template Method** | `template-method-pattern-csharp.md` | Скелет алгоритму у базовому класі, підкласи перевизначають окремі кроки | Спільна структура алгоритму, варіативні лише окремі кроки |
| **Visitor** | `visitor-pattern-csharp.md` | Операція над елементами структури без зміни класів цих елементів (double dispatch) | Багато операцій над стабільною ієрархією класів |

---

## Частина II — Комп'ютерні мережі (10)

Ця частина написана з прицілом саме на **Network Management / SolarWinds** — тому протоколи моніторингу (SNMP, Syslog, NetFlow) розкриті найглибше.

| # | Тема | Файл | Чому це критично для SolarWinds |
|---|---|---|---|
| 1 | Модель OSI та TCP/IP | `osi-tcp-ip-model.md` | Базовий словник для будь-якої розмови про мережі; основа для "пошарової" діагностики (L1→L7), яку використовують усі монітор-продукти |
| 2 | TCP та UDP | `tcp-vs-udp.md` | SNMP/Syslog традиційно працюють по **UDP** — розуміння "чому пакет може загубитись" пояснює логіку retry/timeout у будь-якому пулінг-движку |
| 3 | IP-адресація та підмережі | `ip-addressing-subnetting.md` | Класична "порахуй на дошці" вправа на співбесіді; критично для конфігурації **device discovery** (сканування діапазонів) |
| 4 | Основи маршрутизації | `routing-fundamentals.md` | Пояснює механіку traceroute/hop-by-hop аналізу шляху — основа функцій типу **NPM Path Analysis** |
| 5 | Комутація та VLAN | `switching-and-vlans.md` | Порти/VLAN/MAC-таблиці — те, що NCM/NPM візуалізують при побудові топології мережі |
| 6 | **SNMP** | `snmp-protocol.md` | **Найважливіший файл у папці.** SNMP — фундамент майже всього моніторингу пристроїв у продуктах на кшталт Orion/NPM |
| 7 | Syslog, NetFlow/sFlow/IPFIX, WMI | `syslog-and-netflow-monitoring.md` | Другий за важливістю файл: доповнює SNMP подіями (Syslog), трафіком (NetFlow/NTA) та Windows-специфікою (WMI, для SAM) |
| 8 | DNS та DHCP | `dns-and-dhcp.md` | Основа для **IPAM** (IP Address Manager); типові тікети — вичерпання DHCP scope, конфлікти адрес |
| 9 | Мережева безпека (Firewall/ACL/NAT/VPN) | `network-security-basics.md` | Найчастіша реальна причина "моніторинг не бачить пристрій" — заблокований порт, а не сам пристрій |
| 10 | HTTP/HTTPS/TLS та REST | `http-https-and-web-protocols.md` | Основа веб-моніторингу (SAM), REST API самого Orion (SWIS), моніторинг строку дії сертифікатів |

---

## Частина III — Бази даних (16)

Ця серія написана до **сіньйорського рівня**: кожен файл має розділ "Junior → Senior" питань для співбесіди, і, на відміну від серій патернів/мереж, наскрізно використовує **одну спільну e-commerce-схему** (Customers/Orders/OrderItems/Products/Categories/Reviews), щоб приклади різних файлів природно стикувались один з одним.

### Реляційні (Relational) — 9

| # | Тема | Файл | Ключова сіньйорська ідея |
|---|---|---|---|
| 1 | Реляційна модель та нормалізація | `1-relational-model-and-normalization.md` | 1NF→BCNF покроково через аномалії, і коли свідомо денормалізувати |
| 2 | SQL — мова запитів | `2-sql-query-language-deep-dive.md` | JOIN-и, підзапити, рекурсивні CTE, віконні функції (топ-N у групі) |
| 3 | Індекси та оптимізація запитів | `3-indexes-and-query-optimization.md` | B-Tree зсередини, leftmost-prefix, non-sargable запити, читання execution plan |
| 4 | Транзакції, ACID, рівні ізоляції | `4-transactions-acid-and-isolation-levels.md` | Dirty/non-repeatable/phantom read, lost update, MVCC vs locking |
| 5 | Внутрішній устрій СУБД | `5-database-storage-engines-internals.md` | Сторінки, буферний пул, WAL, чому продуктивність падає, коли таблиця перестає вміщатись у RAM |
| 6 | Реплікація та масштабування | `6-replication-and-scaling-relational.md` | Sync vs async реплікація, шардинг, вибір shard key, read-your-writes |
| 7 | ORM: EF Core та Dapper | `7-orm-entity-framework-and-dapper.md` | Проблема **N+1** — найпрактичніший скіл усієї серії |
| 8 | Backup, Recovery, High Availability | `8-backup-recovery-and-high-availability.md` | RPO/RTO, Point-in-Time Recovery, чому реплікація ≠ бекап, split-brain |
| 9 | Розподілені транзакції та Saga | `9-distributed-transactions-and-saga.md` | Чому 2PC не масштабується; Saga = Command+Memento+Mediator на рівні мікросервісів |

### NoSQL — 7

| # | Тема | Файл | Ключова сіньйорська ідея |
|---|---|---|---|
| 1 | Огляд NoSQL та теорема CAP | `1-nosql-overview-and-cap-theorem.md` | Точне (не спрощене "2 з 3") трактування CAP + PACELC |
| 2 | Документні БД: MongoDB | `2-document-databases-mongodb.md` | Embedding vs Referencing — головне рішення в моделюванні документів |
| 3 | Ключ-значення: Redis | `3-key-value-and-redis.md` | Sorted Set для лідербордів, Cache-Aside, rate limiting, thundering herd |
| 4 | Wide-column: Cassandra | `4-wide-column-stores-cassandra.md` | **Query-first моделювання** — повна протилежність реляційній нормалізації |
| 5 | Графові БД: Neo4j | `5-graph-databases.md` | Index-free adjacency — чому обхід графа не деградує на масштабі, на відміну від SQL self-JOIN |
| 6 | Моделювання даних та узгодженість у NoSQL | `6-nosql-data-modeling-and-consistency.md` | Last-Write-Wins vs Vector Clocks vs CRDT; коли eventual consistency безпечна, а коли ні |
| 7 | Поліглотна персистентність (капстоун) | `7-polyglot-persistence-and-choosing-the-right-database.md` | Фреймворк вибору БД + повна архітектура даних великого інтернет-магазину |

> 💡 Файл `7-polyglot-persistence-and-choosing-the-right-database.md` — підсумок усієї серії БД і водночас прямий компаньйон до `ecommerce-architecture-guide.md` у корені репозиторію: один описує шар **даних** великого магазину, інший — шар **прикладної логіки й патернів**. Варто читати їх парою.

---

## Рекомендований план підготовки

Порядок нижче оптимізований під **компанію мережевого моніторингу**. Якщо співбесіда більше про software engineering (C#/.NET, дизайн коду) — почни з блоку "Патерни"; якщо про мережі/підтримку/NOC — починай з блоку "Мережі" у вказаному порядку.

### Крок 1 — Фундамент мереж (must-know словник)
1. `osi-tcp-ip-model.md` — без цього словника незрозумілі всі інші файли
2. `tcp-vs-udp.md` — handshake, порти, надійність

### Крок 2 — Ядро моніторингу (найбільша ймовірність глибоких питань)
3. `snmp-protocol.md` — Manager/Agent/MIB/OID, версії v1/v2c/v3, **rollover 32-бітних лічильників** (типове "хитре" питання)
4. `syslog-and-netflow-monitoring.md` — Syslog severity levels, NetFlow 5-tuple, sFlow vs NetFlow

### Крок 3 — Практична адресація та маршрутизація
5. `ip-addressing-subnetting.md` — обов'язково прогнати кілька задач на підмережування вручну
6. `routing-fundamentals.md` — traceroute-логіка, IGP vs EGP

### Крок 4 — Решта інфраструктури
7. `switching-and-vlans.md`
8. `dns-and-dhcp.md`
9. `network-security-basics.md` — особливо чому firewall/NAT ламає моніторинг
10. `http-https-and-web-protocols.md` — статус-коди (401 vs 403, 502 vs 503 vs 504 — класика)

### Крок 5 — Патерни проектування (якщо роль включає розробку)
Пріоритет для типового технічного інтерв'ю (найчастіше запитувані першими):
11. **Strategy**, **Observer**, **Factory Method**, **Singleton**, **Decorator** — топ-5 найпопулярніших у співбесідах
12. Далі — решта Поведінкових (Command, State, Template Method, Chain of Responsibility, Iterator, Mediator, Memento, Visitor)
13. Далі — решта Структурних (Adapter, Facade, Composite, Bridge, Proxy, Flyweight)
14. Останні — Abstract Factory, Builder, Prototype (рідше є центром питання, але часто спливають у порівняннях "чим X відрізняється від Y")

### Крок 6 — Бази даних (окремий трек; читай послідовно, файли побудовані один на одному)
15. `1-relational-model-and-normalization.md` → `2-sql-query-language-deep-dive.md` → `3-indexes-and-query-optimization.md` — фундамент: модель, мова запитів, продуктивність
16. `4-transactions-acid-and-isolation-levels.md` — ACID/isolation levels, класика будь-якої співбесіди про БД
17. `7-orm-entity-framework-and-dapper.md` — проблема **N+1**, найчастіше практичне питання на живому коді
18. `5-database-storage-engines-internals.md` → `6-replication-and-scaling-relational.md` → `8-backup-recovery-and-high-availability.md` → `9-distributed-transactions-and-saga.md` — архітектура та масштабування, для Middle→Senior позицій
19. `1-nosql-overview-and-cap-theorem.md` — обов'язковий вхід у NoSQL перед будь-яким конкретним файлом нижче
20. `2-document-databases-mongodb.md`, `3-key-value-and-redis.md` — найпоширеніші NoSQL-технології на практиці
21. `4-wide-column-stores-cassandra.md`, `5-graph-databases.md`, `6-nosql-data-modeling-and-consistency.md` — для позицій, що явно вимагають розподілені системи
22. `7-polyglot-persistence-and-choosing-the-right-database.md` — читати останнім: капстоун, що збирає все в одну архітектуру

---

## Наскрізні теми, що з'являються в кількох файлах

Ці зв'язки легко пропустити, якщо читати файли ізольовано — а саме вони часто стають "а чому насправді" питаннями на співбесіді.

- **UDP і ненадійність** (`tcp-vs-udp.md`) напряму пояснює, чому `snmp-protocol.md` і `syslog-and-netflow-monitoring.md` вимагають власної retry/timeout-логіки на рівні застосунку — жоден з протоколів моніторингу сам по собі не гарантує доставку.
- **Firewall/NAT** (`network-security-basics.md`) — найчастіша реальна причина, чому "не працює" будь-що з попередніх пунктів: SNMP (`161/162`), Syslog (`514`), WMI (`135`+динамічні порти), HTTP(S) (`80/443`). Знання конкретних портів з кожного файлу + це знання про фаєрволи разом складають типовий "чому моніторинг не бачить пристрій" troubleshooting-кейс.
- **OSI-рівні як мова діагностики** (`osi-tcp-ip-model.md`) — використовується як фреймворк у сценарії з `snmp-protocol.md` (L3 ping → L4 порт → L7 SNMP-відповідь) і в `http-https-and-web-protocols.md` (мережа доступна, а сайт — ні, отже проблема вище L4).
- **Command + Memento** (патерни) — часто розглядаються разом, оскільки Command для реалізації Undo зазвичай спирається на Memento для збереження попереднього стану.
- **State + Strategy** — структурно майже ідентичні; питання "чим відрізняються" — класика співбесід, відповідь є в обох відповідних файлах у розділі порівняння.
- **Composite + Iterator + Visitor** — три патерни, що часто працюють РАЗОМ над деревоподібними структурами (файлова система, DOM, AST): Composite задає структуру, Iterator обходить її, Visitor виконує операцію під час обходу.
- **Saga = Command + Memento + Mediator, застосовані до розподілених транзакцій** — `9-distributed-transactions-and-saga.md` явно проводить цю паралель: компенсуюча дія кроку Saga — це Command з Memento-подібним знімком стану для відкату, а Saga Orchestrator — буквально Mediator, застосований до мікросервісів. Якщо розумієш ці три патерни — Saga вчиться вдвічі швидше.
- **Composite в пам'яті vs Embedding/Referencing на диску** — `2-document-databases-mongodb.md` явно розділяє ці дві споріднені, але окремі задачі: як дерево категорій моделюється в ООП-коді (патерн Composite) — і як те саме дерево зберігається в MongoDB (embed vs reference). Плутати ці два рівні — поширена помилка.
- **Observer як механізм синхронізації в поліглотній архітектурі** — `7-polyglot-persistence-and-choosing-the-right-database.md` показує Observer не як патерн у межах одного застосунку, а як спосіб тримати кілька різних БД (реляційну, MongoDB, Redis, Neo4j) у приблизно узгодженому стані через події.
- **UDP-ненадійність (мережі) і eventual consistency (NoSQL)** — та сама фундаментальна ідея з двох різних сторін: `tcp-vs-udp.md` пояснює, чому мережевий протокол може не гарантувати доставку, а `6-nosql-data-modeling-and-consistency.md` — чому розподілена БД може тимчасово повертати застарілі дані. В обох випадках відповідь одна: явно спроєктована retry/reconciliation-логіка, а не сподівання, що "само якось узгодиться".

---

## Швидкий чекліст перед співбесідою

Мінімальний набір фактів, які варто вміти відтворити з пам'яті за 2 хвилини:

**Мережі:**
- [ ] Назви всіх 7 рівнів OSI напам'ять + приклад протоколу на кожному
- [ ] TCP three-way handshake (SYN → SYN-ACK → ACK) і навіщо потрібен
- [ ] Різниця SNMP v1/v2c/v3 (community string vs автентифікація+шифрування)
- [ ] SNMP-порти: **161** (запити), **162** (traps)
- [ ] Порахувати кількість хостів у `/27`, `/26`, `/24` без калькулятора
- [ ] DORA-процес DHCP (Discover → Offer → Request → Acknowledge)
- [ ] Різниця 401 vs 403, 502 vs 503 vs 504
- [ ] Що таке 5-tuple потоку (NetFlow) і чим NetFlow відрізняється від sFlow

**Патерни:**
- [ ] Три складові Singleton (private конструктор, static поле, static властивість)
- [ ] Чим Strategy відрізняється від State (хто вибирає і коли)
- [ ] Чим Decorator відрізняється від Proxy (додає поведінку vs контролює доступ)
- [ ] Чим Facade відрізняється від Adapter (спрощує підсистему vs перекладає інтерфейс)
- [ ] Приклад Observer через `event`/делегати в C#

**Бази даних:**
- [ ] Розшифрувати ACID і назвати всі 4 рівні ізоляції та які аномалії кожен запобігає
- [ ] Пояснити проблему N+1 в ORM і як її виправити (`.Include()`/eager loading)
- [ ] Leftmost-prefix правило для складеного індексу — на прикладі
- [ ] Порахувати нормальні форми: навести приклад порушення 2NF і 3NF
- [ ] CAP-теорема — точне формулювання (не "2 з 3", а вибір C/A саме під час партиції) + PACELC
- [ ] Query-first моделювання в Cassandra на противагу нормалізації в SQL
- [ ] Embedding vs Referencing у MongoDB — коли обрати кожен
- [ ] Чому Saga, а не 2PC, для транзакцій між мікросервісами

---

*Цей гайд — навігаційний документ. Деталі, код на C#, розгорнуті приклади та повні блоки "Питання та відповіді" шукай у відповідних файлах вище.*
