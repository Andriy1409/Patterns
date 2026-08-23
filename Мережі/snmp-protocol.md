# SNMP (Simple Network Management Protocol) — Детальний розбір для співбесіди

> **Категорія:** Комп'ютерні мережі (Application Layer / Network Management)  
> **Контекст:** Підготовка до співбесіди (Network Management / SolarWinds — ключова тема!)  
> **Мова прикладів:** C# (.NET)

---

## Зміст

1. [Що таке SNMP і навіщо він потрібен?](#1-що-таке-snmp-і-навіщо-він-потрібен)
2. [Архітектура SNMP: Manager, Agent, MIB](#2-архітектура-snmp-manager-agent-mib)
3. [Операції SNMP](#3-операції-snmp)
4. [Версії SNMP: v1, v2c, v3](#4-версії-snmp-v1-v2c-v3)
5. [Транспорт: чому SNMP використовує UDP](#5-транспорт-чому-snmp-використовує-udp)
6. [Практичні приклади на C#](#6-практичні-приклади-на-c)
7. [Реальний сценарій: SNMP як основа мережевого моніторингу (SolarWinds)](#7-реальний-сценарій-snmp-як-основа-мережевого-моніторингу-solarwinds)
8. [Поширені проблеми та діагностика](#8-поширені-проблеми-та-діагностика)
9. [Питання та відповіді для співбесіди](#9-питання-та-відповіді-для-співбесіди)
10. [Підсумок](#10-підсумок)

---

## 1. Що таке SNMP і навіщо він потрібен?

**SNMP (Simple Network Management Protocol)** — це стандартний протокол прикладного рівня (Application Layer), призначений для **моніторингу та управління мережевими пристроями** віддалено: маршрутизаторами, комутаторами, серверами, принтерами, джерелами безперебійного живлення (UPS), точками доступу Wi-Fi, файрволами тощо.

Це фактично **de facto стандарт** для збору телеметрії з мережевого обладнання ще з кінця 1980-х років (перша версія — RFC 1157, 1988 рік), і саме на ньому побудована практично вся індустрія систем моніторингу мереж — включно з продуктами на кшталт **SolarWinds Orion, NPM (Network Performance Monitor), NCM (Network Configuration Manager)**, а також PRTG, Zabbix, Nagios, Cacti, LibreNMS та десятками інших.

> 📊 Якщо ви йдете на співбесіду в компанію, яка робить мережевий моніторинг (як SolarWinds) — SNMP це **найважливіша тема**, бо саме навколо роботи з SNMP (polling engine, MIB-парсинг, обробка traps, walk-и) будується левова частка бекенду таких продуктів.

### Навіщо він потрібен?

Уявіть мережу з тисячами пристроїв: маршрутизатори, комутатори, сервери. Адміністратору потрібно знати:
- Яке навантаження на CPU кожного пристрою?
- Скільки трафіку проходить через кожен інтерфейс?
- Чи не впав якийсь лінк?
- Скільки вільної пам'яті лишилось?
- Який рівень заряду батареї в UPS?

Робити це вручну (заходити на кожен пристрій по SSH/консолі) — нереально в масштабі. SNMP дає **єдиний уніфікований протокол**, яким можна опитувати (і в деяких випадках — конфігурувати) практично будь-який мережевий пристрій незалежно від виробника, якщо він підтримує стандарт SNMP.

### Аналогія з реального життя

Уявіть багатоквартирний будинок:

- **Керуючий будинку (Manager / SolarWinds Orion Polling Engine)** — центральна особа, яка хоче знати стан усіх квартир.
- **Розумний лічильник у кожній квартирі (Agent)** — невеликий модуль на пристрої, який знає поточні показники (споживання води, електрики) і готовий їх повідомити на запит.
- **Опитування (Polling)** — керуючий будинку періодично (наприклад, раз на годину) обходить квартири й запитує: *"Який у вас зараз показник лічильника?"* Це активна, ініційована менеджером дія — аналог SNMP **GET**.
- **Аварійне сповіщення (Trap)** — якщо в якійсь квартирі прорвало трубу, мешканець сам, не чекаючи запитання, дзвонить керуючому: *"У нас потоп!"* Це проактивне, ініційоване агентом повідомлення — аналог SNMP **TRAP**. Важливо: керуючий будинку може навіть не встигнути відповісти чи підтвердити дзвінок — повідомлення просто "вилітає" в ефір (як TRAP), або ж мешканець може вимагати підтвердження дзвінка (як INFORM).

Ця комбінація — **періодичний опитний моніторинг (polling) + проактивні сповіщення (traps)** — і є суттю того, як SNMP-моніторинг працює на практиці, і саме так побудовані продукти типу SolarWinds NPM.

### Ключові факти про SNMP

| Характеристика | Значення |
|---|---|
| Рівень OSI | Прикладний (Application Layer, L7) |
| Транспортний протокол | UDP |
| Стандартні порти | 161 (запити до агента), 162 (traps/informs до менеджера) |
| Перша версія | 1988, RFC 1157 (SNMPv1) |
| Поточна рекомендована версія | SNMPv3 (з безпекою) |
| Типове застосування | Моніторинг CPU/пам'яті/трафіку/статусу інтерфейсів, дискавері пристроїв, базова конфігурація |
| Хто використовує | SolarWinds, PRTG, Zabbix, Nagios, Cacti, LibreNMS, HP OpenView, і практично всі NMS (Network Management Systems) |

---

## 2. Архітектура SNMP: Manager, Agent, MIB

SNMP побудований на трьох ключових поняттях: **Manager**, **Agent**, **MIB**.

```
┌─────────────────────────────┐                    ┌──────────────────────────────┐
│         MANAGER              │                    │           AGENT                │
│  (напр. SolarWinds Orion     │   GET / GET-NEXT /  │   (сервіс/демон на пристрої:   │
│   Polling Engine, NMS)        │   GET-BULK / SET     │   маршрутизатор, комутатор,    │
│                               │ ────────────────────>│   сервер, UPS, принтер...)     │
│  - Ініціює запити            │                      │                                │
│  - Збирає й агрегує дані      │ <────────────────────│   - Слухає на UDP/161          │
│  - Візуалізує (графіки,       │      RESPONSE         │   - Читає/пише значення з MIB  │
│    дашборди, алерти)          │                      │   - Надсилає TRAP/INFORM        │
│  - Слухає traps на UDP/162    │ <════════════════════│     при подіях                 │
│                               │   TRAP / INFORM       │                                │
└─────────────────────────────┘                    └──────────────────────────────┘
```

### Manager (Менеджер)

Це центральна система моніторингу — програмне забезпечення, яке:
- Опитує (polls) агентів на пристроях через певні інтервали;
- Отримує та обробляє traps/informs;
- Зберігає історичні дані (для графіків, трендів);
- Генерує алерти, якщо значення виходять за пороги (thresholds).

У продукті SolarWinds це — **Polling Engine** оркестрового рівня Orion Platform.

### Agent (Агент)

Це невеликий програмний компонент, що працює **на самому пристрої** (вбудований у прошивку маршрутизатора/комутатора, або окремий процес — daemon — на Linux/Windows-сервері, наприклад `snmpd` на Linux або Windows SNMP Service). Агент:
- Тримає локальну базу даних значень (лічильники, статуси, конфігураційні параметри), організовану за схемою **MIB**;
- Відповідає на GET/GET-NEXT/GET-BULK/SET запити менеджера;
- Може самостійно надсилати TRAP/INFORM повідомлення при настанні подій (наприклад, лінк впав — `linkDown` trap).

### MIB (Management Information Base)

**MIB** — це ієрархічна, стандартизована "база даних"/схема, яка описує, **які саме дані доступні** на пристрої та як вони структуровані. Це не файл із живими даними, а скоріше **словник-специфікація**: "за таким-то ідентифікатором лежить значення такого-то типу з таким-то значенням". Практично — MIB-файл (текстовий, у форматі ASN.1) описує ці ідентифікатори людською мовою (`sysDescr`, `ifInOctets` тощо) і мапить їх на числові адреси.

### OID (Object Identifier)

**OID** — це унікальна, ієрархічна, крапко-числова адреса конкретної одиниці даних у дереві MIB. Наприклад:

```
1.3.6.1.2.1.1.3.0   →  sysUpTime.0   (час роботи системи з моменту останнього рестарту)
1.3.6.1.2.1.1.1.0   →  sysDescr.0    (текстовий опис пристрою)
1.3.6.1.2.1.2.2.1.10.1  → ifInOctets.1  (кількість вхідних байтів на інтерфейсі №1)
```

Дерево OID виглядає так (спрощено, найважливіша гілка — `mib-2`):

```
                                    root
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                 │
                  ccitt(0)        iso(1)            joint-iso-ccitt(2)
                                     │
                                   org(3)
                                     │
                                  dod(6)
                                     │
                               internet(1)
                                     │
              ┌──────────────────────┼───────────────────────┐
              │                      │                        │
        directory(1)             mgmt(2)                 private(4)
                                     │                        │
                                 mib-2(1)                enterprises(1)
                                     │                        │
        ┌──────────┬────────────────┼──────────┐      ┌───────┴────────┐
        │          │                │           │      │                │
    system(1)  interfaces(2)      ip(4)      tcp(6)  cisco(9)   sonicwall(8741)
        │          │                                       ...   ...    ...
   ┌────┼────┐    ifTable(2)                                       │
   │    │    │      │                                        (кастомні OID
sysDescr sysUpTime  │                                        конкретного вендора)
 (1.0)   (3.0)    ifEntry(1)
                     │
        ┌────────────┼────────────┐
        │            │             │
   ifIndex(1)   ifDescr(2)   ifInOctets(10)  ifOutOctets(16)  ifOperStatus(8) ...
```

Тобто повний OID для `sysDescr.0` формується проходом по дереву:
`iso(1).org(3).dod(6).internet(1).mgmt(2).mib-2(1).system(1).sysDescr(1).0`
= **`1.3.6.1.2.1.1.1.0`**

### Стандартні гілки MIB, важливі для моніторингу

| OID (префікс) | Назва | Що містить |
|---|---|---|
| `1.3.6.1.2.1.1` | `system` | Загальна інформація: опис, uptime, ім'я, контакт, локація |
| `1.3.6.1.2.1.2.2.1` | `ifTable` / `ifEntry` | Таблиця мережевих інтерфейсів (індекс, назва, статус, лічильники трафіку) |
| `1.3.6.1.2.1.4` | `ip` | IP-статистика |
| `1.3.6.1.2.1.25` | `host` (Host Resources MIB) | CPU, диски, процеси (для серверів) |
| `1.3.6.1.4.1` | `enterprises` | Кореневий вузол для **вендор-специфічних** MIB |

### Vendor-специфічні (proprietary) MIB

Стандартний MIB-II (`1.3.6.1.2.1.*`) покриває загальні речі (uptime, інтерфейси, IP), але не описує специфічні для конкретного виробника дані — наприклад, температуру процесора Cisco, рівень тонера принтера HP, стан батареї APC UPS. Для цього кожен вендор отримує **Private Enterprise Number (PEN)** від IANA і будує своє піддерево під `1.3.6.1.4.1.<vendor-id>`:

```
1.3.6.1.4.1.9      → Cisco Systems
1.3.6.1.4.1.318    → American Power Conversion (APC) — UPS-и
1.3.6.1.4.1.2021   → Net-SNMP (UCD-SNMP) extended MIB
1.3.6.1.4.1.11     → Hewlett-Packard
```

Щоб система моніторингу (наприклад, SolarWinds) могла показувати ці дані людською мовою ("Battery Charge: 87%" замість "OID 1.3.6.1.4.1.318.1.1.1.2.2.1.0 = 87"), їй потрібно **завантажити (compile) MIB-файл цього вендора** — про це детальніше в розділі 7.

### SMI (Structure of Management Information)

**SMI** — це набір правил (визначений у RFC 1155/2578), який описує, **як саме структуруються та типізуються** дані в MIB: які базові типи даних дозволені (`INTEGER`, `OCTET STRING`, `Counter32`, `Counter64`, `Gauge32`, `TimeTicks`, `IpAddress` тощо), як формуються OID, як описуються таблиці (`SEQUENCE OF`). Простими словами — SMI це "граматика" мови, якою написані MIB-файли (сам MIB використовує підмножину ASN.1 — Abstract Syntax Notation One).

---

## 3. Операції SNMP

SNMP визначає невелику кількість базових операцій (PDU — Protocol Data Unit типів), якими Manager і Agent обмінюються повідомленнями.

| Операція | Хто ініціює | Напрямок | Призначення |
|---|---|---|---|
| **GET** | Manager | Manager → Agent | Отримати значення **одного конкретного** OID |
| **GET-NEXT** | Manager | Manager → Agent | Отримати наступний OID у дереві (за лексикографічним порядком) — основа для "walk" |
| **GET-BULK** (з v2) | Manager | Manager → Agent | Отримати **багато** значень за один запит (ефективніше за серію GET-NEXT) |
| **SET** | Manager | Manager → Agent | Змінити значення на агенті (запис) |
| **TRAP** | Agent | Agent → Manager | Проактивне сповіщення про подію, **без підтвердження** |
| **INFORM** (з v2) | Agent | Agent → Manager | Те ж саме, що TRAP, але **з підтвердженням (ack)** |
| **RESPONSE** | Agent | Agent → Manager | Відповідь на GET/GET-NEXT/GET-BULK/SET |

### GET

Найпростіша операція: менеджер каже "дай мені значення OID X", агент повертає значення. Приклад: запит `sysUpTime.0` → відповідь `Timeticks: 123456789 (14 days, 6:56:07.89)`.

### GET-NEXT

Повертає значення **наступного** (за порядком дерева) OID, а не того, що запитали. Це дозволяє "обходити" дерево, не знаючи наперед точних адрес — саме на цьому механізмі побудований **SNMP walk**.

**SNMP walk** — це не окрема операція протоколу, а **техніка**: менеджер багаторазово викликає GET-NEXT, кожного разу передаючи OID, отриманий у попередній відповіді, поки не вийде за межі потрібного піддерева (або агент не поверне помилку `endOfMibView`). Так можна перерахувати всю таблицю інтерфейсів, не знаючи наперед, скільки їх на пристрої.

```
Запит 1:  GET-NEXT(1.3.6.1.2.1.2.2.1.2)         → відповідь: 1.3.6.1.2.1.2.2.1.2.1 = "GigabitEthernet0/1"
Запит 2:  GET-NEXT(1.3.6.1.2.1.2.2.1.2.1)       → відповідь: 1.3.6.1.2.1.2.2.1.2.2 = "GigabitEthernet0/2"
Запит 3:  GET-NEXT(1.3.6.1.2.1.2.2.1.2.2)       → відповідь: 1.3.6.1.2.1.2.2.1.2.3 = "GigabitEthernet0/3"
...
Запит N:  GET-NEXT(...)                          → відповідь: OID поза межами ifDescr → зупинка walk
```

### GET-BULK (SNMPv2c+)

Проблема послідовних GET-NEXT: якщо в таблиці інтерфейсів 200 рядків, потрібно 200+ окремих round-trip запитів — повільно і дорого по мережі. **GET-BULK** дозволяє запросити одразу N наступних записів **в одному пакеті-запиті**, суттєво зменшуючи кількість round-trip-ів (аналогічно до pipelining). Параметри `non-repeaters` та `max-repetitions` контролюють, скільки саме об'єктів повертати.

### SET

Дозволяє **записати** нове значення на агента (наприклад, змінити `sysContact`, чи навіть перезавантажити порт комутатора, залежно від MIB). Це потужна, але й найнебезпечніша операція — саме тому доступ до SET традиційно жорстко обмежують (окрема community string "write", ACL, або у v3 — окрема user-based політика доступу). У більшості моніторингових систем (включно з SolarWinds) SET використовується рідко і обережно — основне навантаження це GET/GET-BULK для читання.

### TRAP

Агент **сам, без запиту**, надсилає повідомлення менеджеру при настанні події (лінк впав/піднявся, перевищено поріг температури, перезавантаження пристрою). TRAP надсилається на UDP-порт **162** менеджера. Головна особливість: **TRAP не потребує і не отримує підтвердження** — це "fire-and-forget" повідомлення. Якщо пакет загубиться в мережі (а UDP цього не гарантує), менеджер про подію просто не дізнається.

### INFORM (SNMPv2c+)

Те саме, що TRAP, але **з підтвердженням**: отримавши INFORM, менеджер відправляє назад acknowledgment. Якщо агент не отримав ack за певний час — він **повторює** відправку. Це робить доставку сповіщень значно надійнішою за звичайні TRAP, ціною невеликого додаткового навантаження (потрібно зберігати стан "чи підтверджено" на боці агента, поки не прийде ack або не вичерпаються спроби).

```
TRAP:                                  INFORM:
Agent ──── TRAP ───────> Manager       Agent ──── INFORM ──────> Manager
     (і забув)                              <──── ACK ──────────
                                        (якщо ack не прийшов за timeout — Agent повторює INFORM)
```

---

## 4. Версії SNMP: v1, v2c, v3

### SNMPv1 (1988, RFC 1157)

Перша, найпростіша версія.

- Підтримує: GET, GET-NEXT, SET, TRAP (в оригінальному, "класичному" форматі, відмінному від v2/v3).
- **Немає GET-BULK** — тільки послідовні GET-NEXT для обходу таблиць.
- "Безпека" — лише **community string**: простий текстовий "пароль", що передається у **відкритому вигляді (plaintext)** в кожному пакеті. Типово `"public"` для читання, `"private"` для запису (стандартні дефолти, які **обов'язково потрібно міняти** в продакшені).
- Немає шифрування, немає надійної автентифікації користувача — фактично, будь-хто, хто знає (чи підслухав) community string і має мережевий доступ до UDP/161 пристрою, може читати (а якщо це "private" — і писати) дані.

### SNMPv2c (1996, RFC 1901-1908)

Найпоширеніша на практиці версія (літера **"c"** означає **community-based** — модель безпеки залишилась та сама, що й у v1).

- Додає **GET-BULK** — значно ефективніший обхід великих таблиць.
- Додає **INFORM** — надійні (з підтвердженням) сповіщення.
- Покращує формати помилок і типи даних (наприклад, `Counter64` для 64-бітних лічильників, важливо для швидких інтерфейсів — про це в розділі 7).
- Модель безпеки — та ж сама community string, що і в v1: **все ще передається у відкритому вигляді**, і все ще немає ні автентифікації користувача, ні шифрування.
- Попри відсутність безпеки, саме v2c **історично найбільш розповсюджена версія в реальних мережах** — через простоту налаштування (просто задати community string на всіх пристроях) і широку сумісність.

### SNMPv3 (1998-2004, RFC 3410-3418, згодом оновлено)

Версія, що нарешті додає **справжню безпеку**.

- **USM (User-based Security Model)** — замість спільного "паролю"-community на всіх, кожен користувач має власні облікові дані:
  - **Автентифікація (Authentication)** — підтвердження, що пакет справді від того користувача, за якого себе видає, і не був змінений у дорозі. Алгоритми: **MD5, SHA (SHA-1, SHA-224/256/384/512 у сучасних реалізаціях)**.
  - **Приватність/шифрування (Privacy, "priv")** — шифрування вмісту пакету, щоб його не можна було прочитати при перехопленні. Алгоритми: **DES (застарілий), AES (128/192/256-біт, рекомендовано)**.
  - Три рівні безпеки (**Security Levels**): `noAuthNoPriv` (як v1/v2c, без захисту), `authNoPriv` (є автентифікація, без шифрування), `authPriv` (і автентифікація, і шифрування — рекомендований рівень).
- **VACM (View-based Access Control Model)** — гнучке керування доступом: які саме частини дерева MIB (View) конкретний користувач/група може читати чи писати, а не просто "весь MIB чи нічого".
- Складніший у налаштуванні (потрібно створювати users, задавати auth/priv паролі й алгоритми на кожному пристрої і в менеджері), тому на практиці частина мереж досі використовує v2c "просто тому що так історично склалось" — хоча для будь-якого нового розгортання рекомендують **v3**.

### Чому безпека v1/v2c — це реальна проблема

Community string у v1/v2c передається **у відкритому вигляді в кожному UDP-пакеті**. Будь-хто в тому ж сегменті мережі (чи на шляху трафіку), маючи можливість знімати пакети (sniffing, наприклад через Wireshark), миттєво побачить community string у форматі звичайного тексту. Якщо це "private" (write-доступ) — зловмисник отримує можливість **змінювати конфігурацію пристроїв** мережі: змінювати маршрутизацію, вимикати інтерфейси, змінювати паролі SNMP тощо. Навіть read-only доступ ("public") дає зловмиснику детальну карту всієї інфраструктури: топологію, версії ПЗ, серійні номери, конфігурацію — цінну розвідувальну інформацію для подальшої атаки.

Саме тому в сучасних best practices (і в вимогах compliance на кшталт PCI-DSS) рекомендується:
- Вимикати SNMPv1/v2c там, де можливо, і переходити на **SNMPv3 з `authPriv`**;
- Якщо v2c все ж використовується — обмежувати доступ через ACL (дозволяти SNMP-запити лише з IP менеджера моніторингу) і не використовувати дефолтні community strings.

### Порівняльна таблиця версій

| | **SNMPv1** | **SNMPv2c** | **SNMPv3** |
|---|---|---|---|
| Рік / RFC | 1988, RFC 1157 | 1996, RFC 1901-1908 | 1998-2004, RFC 3410-3418 |
| Модель безпеки | Community string (plaintext) | Community string (plaintext) | USM: user + auth (MD5/SHA) + priv (DES/AES) |
| Контроль доступу | Немає гнучкого | Немає гнучкого | VACM (View-based, гранулярний) |
| GET / GET-NEXT / SET | ✅ | ✅ | ✅ |
| GET-BULK | ❌ | ✅ | ✅ |
| TRAP | ✅ (старий формат) | ✅ | ✅ |
| INFORM (з підтвердженням) | ❌ | ✅ | ✅ |
| Counter64 (64-бітні лічильники) | ❌ | ✅ | ✅ |
| Транспорт | UDP (161/162) | UDP (161/162) | UDP (161/162) |
| Типове застосування зараз | Легасі-обладнання | Найпоширеніша на практиці | Рекомендована для нових розгортань |

---

## 5. Транспорт: чому SNMP використовує UDP

SNMP працює поверх **UDP**, а не TCP:
- **UDP/161** — порт, на якому агент слухає запити менеджера (GET/GET-NEXT/GET-BULK/SET) і надсилає відповіді;
- **UDP/162** — порт, на якому менеджер слухає **traps та informs**, надіслані агентами.

### Чому саме UDP, а не TCP?

1. **Легковаговість і низькі накладні витрати.** UDP не має handshake (3-way handshake TCP), не тримає з'єднання, не має накладних витрат на підтвердження кожного сегмента. Для короткого запит-відповідь обміну (типовий SNMP GET — це один невеликий пакет туди, один назад) TCP-з'єднання додавало б непропорційно багато накладних витрат, особливо коли менеджер опитує **тисячі пристроїв** з інтервалом у кілька секунд-хвилин.
2. **Масштабованість опитування (polling).** Система моніторингу типу SolarWinds NPM може одночасно опитувати десятки тисяч OID на сотнях/тисячах пристроїв. Підтримувати стільки одночасних TCP-з'єднань (з їх буферами, чергами retransmission, state machine) значно дорожче за розсилку UDP-датаграм.
3. **Надійність доставки — відповідальність прикладного рівня.** Так, UDP не гарантує доставку. Але SNMP-протокол компенсує це на своєму рівні: менеджер, не отримавши відповіді на GET за таймаут, сам вирішує — повторити запит (retry) чи ні. Це узгоджується з фундаментальним принципом Інтернет-архітектури (end-to-end principle) — надійність реалізується там, де вона дійсно потрібна, а не нав'язується всім транспортом за замовчуванням.
4. **TRAP — "fire-and-forget" за задумом.** Traps в оригінальній моделі SNMPv1 навмисно не потребують підтвердження — це узгоджується з ідеєю "легкого, дешевого" сповіщення. Якщо потрібна гарантована доставка сповіщення — використовують **INFORM** (з'явився в v2c), який додає ack-механізм **на прикладному рівні SNMP**, а не змінює транспорт на TCP.

### Практичний наслідок для розробників систем моніторингу

Оскільки UDP може **мовчки** загубити пакет (запит GET, або відповідь на нього), критично важливо, щоб программна логіка опитування (polling engine) не робила поспішних висновків:

> ❌ **Неправильно:** не отримали відповідь на один GET-запит → одразу позначили пристрій як "Down".
>
> ✅ **Правильно:** реалізувати логіку **retry з таймаутом** (наприклад, 3 спроби з інтервалом 1-2 секунди) — і лише якщо **всі** спроби провалились, позначати пристрій недоступним. Один загублений UDP-пакет — це нормальне, очікуване явище в мережі, а не обов'язково ознака реальної проблеми з пристроєм.

Це саме той шаблон "retry N разів з таймаутом", який детально розглядається в документі про TCP/UDP, і в SNMP-моніторингу він застосовується буквально — реалізовано в прикладі 3 нижче (`SnmpPoller`).

---

## 6. Практичні приклади на C#

> **Важливо:** у реальних .NET-проєктах SNMP **майже ніколи не реалізують "з нуля"** через сирі сокети — кодування/декодування повідомлень відбувається у форматі **ASN.1 BER (Basic Encoding Rules)**, що доволі складно і легко зробити помилку. Замість цього використовують готові бібліотеки, найпопулярніша в екосистемі .NET — **[Lextm.SharpSnmpLib](https://github.com/lextm/sharpsnmplib) (SharpSNMP)**, доступна через NuGet.
>
> Нижче — комбінація: (1) спрощена ілюстрація "як це працює під капотом" на голих UDP-сокетах (виключно навчальний приклад, без справжнього ASN.1), (2) реалістичний приклад з реальною бібліотекою SharpSNMP, (3) шаблон опитування з retry, (4) прийомник traps.

### Приклад 1 — спрощена симуляція SNMP GET на голих UDP-сокетах (навчальний приклад)

Мета — показати механіку запит/відповідь без реального ASN.1-кодування. Тут ми імітуємо: "агент" — окремий Task, що слухає UDP і відповідає на один хардкоджений OID; "менеджер" — надсилає текстовий рядок-запит і чекає відповідь.

```csharp
using System;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Threading.Tasks;

// ====== СПРОЩЕНИЙ "АГЕНТ" (імітує пристрій, що відповідає на SNMP-подібний GET) ======
// УВАГА: це НЕ реальний SNMP-протокол (немає ASN.1/BER), а спрощена ілюстрація
// механіки "запит по UDP -> відповідь по UDP" для навчальних цілей.
class MockSnmpAgent
{
    private readonly int _port;
    private readonly string _oid = "1.3.6.1.2.1.1.1.0"; // sysDescr.0
    private readonly string _value = "Cisco IOS Router, Model ISR4321, Version 16.9.4";

    public MockSnmpAgent(int port) => _port = port;

    public async Task RunAsync(System.Threading.CancellationToken token)
    {
        using var udpServer = new UdpClient(_port);
        Console.WriteLine($"[Agent] Слухаю запити на UDP/{_port}...");

        while (!token.IsCancellationRequested)
        {
            var result = await udpServer.ReceiveAsync(token);
            string request = Encoding.UTF8.GetString(result.Buffer);
            Console.WriteLine($"[Agent] Отримано запит: \"{request}\" від {result.RemoteEndPoint}");

            // Дуже спрощений "парсинг": формат запиту "GET <oid>"
            string response;
            if (request.StartsWith("GET ") && request[4..] == _oid)
            {
                response = $"RESPONSE {_oid} = \"{_value}\"";
            }
            else
            {
                response = "RESPONSE noSuchObject";
            }

            byte[] responseBytes = Encoding.UTF8.GetBytes(response);
            await udpServer.SendAsync(responseBytes, result.RemoteEndPoint, token);
            Console.WriteLine($"[Agent] Надіслано відповідь: \"{response}\"");
        }
    }
}

// ====== СПРОЩЕНИЙ "МЕНЕДЖЕР" (імітує SNMP-клієнт, що робить GET) ======
class MockSnmpManager
{
    public async Task<string> GetAsync(string host, int port, string oid, int timeoutMs = 2000)
    {
        using var client = new UdpClient();
        client.Client.ReceiveTimeout = timeoutMs;

        byte[] request = Encoding.UTF8.GetBytes($"GET {oid}");
        var endpoint = new IPEndPoint(IPAddress.Parse(host), port);
        await client.SendAsync(request, endpoint);
        Console.WriteLine($"[Manager] Надіслано GET-запит на {host}:{port} для OID {oid}");

        using var cts = new System.Threading.CancellationTokenSource(timeoutMs);
        try
        {
            var result = await client.ReceiveAsync(cts.Token);
            return Encoding.UTF8.GetString(result.Buffer);
        }
        catch (OperationCanceledException)
        {
            return "TIMEOUT — відповідь не отримана вчасно";
        }
    }
}

// ====== Точка входу ======
class Program
{
    static async Task Main()
    {
        const int port = 16100; // умовний "локальний UDP/161" для демо (реальний 161 вимагає прав адміністратора)
        using var cts = new System.Threading.CancellationTokenSource();

        var agent = new MockSnmpAgent(port);
        var agentTask = agent.RunAsync(cts.Token);

        await Task.Delay(300); // даємо агенту час стартувати

        var manager = new MockSnmpManager();
        string reply = await manager.GetAsync("127.0.0.1", port, "1.3.6.1.2.1.1.1.0");
        Console.WriteLine($"[Manager] Результат: {reply}");

        cts.Cancel();
    }
}
```

**Очікуваний вивід у консолі:**

```
[Agent] Слухаю запити на UDP/16100...
[Manager] Надіслано GET-запит на 127.0.0.1:16100 для OID 1.3.6.1.2.1.1.1.0
[Agent] Отримано запит: "GET 1.3.6.1.2.1.1.1.0" від 127.0.0.1:54321
[Agent] Надіслано відповідь: "RESPONSE 1.3.6.1.2.1.1.1.0 = "Cisco IOS Router, Model ISR4321, Version 16.9.4""
[Manager] Результат: RESPONSE 1.3.6.1.2.1.1.1.0 = "Cisco IOS Router, Model ISR4321, Version 16.9.4"
```

### Приклад 2 — реальний GET та WALK через бібліотеку Lextm.SharpSnmpLib

```bash
# Встановлення пакету
dotnet add package Lextm.SharpSnmpLib
```

```csharp
using System;
using System.Linq;
using System.Net;
using Lextm.SharpSnmpLib;
using Lextm.SharpSnmpLib.Messaging;

class RealSnmpExample
{
    // Реальний SNMP GET: запит sysDescr.0 у пристрою через SNMPv2c
    static void GetSysDescr(string ip, string community)
    {
        var endpoint = new IPEndPoint(IPAddress.Parse(ip), 161);

        // Formulate GET request for OID sysDescr.0 = 1.3.6.1.2.1.1.1.0
        var variables = new System.Collections.Generic.List<Variable>
        {
            new Variable(new ObjectIdentifier("1.3.6.1.2.1.1.1.0"))
        };

        // Виконуємо синхронний GET-запит по SNMPv2c
        IList<Variable> result = Messenger.Get(
            VersionCode.V2,
            endpoint,
            new OctetString(community),   // напр., "public"
            variables,
            timeout: 3000);               // таймаут у мілісекундах

        foreach (Variable v in result)
        {
            Console.WriteLine($"{v.Id} = {v.Data}");
        }
    }

    // Реальний SNMP WALK: обхід усієї таблиці інтерфейсів (ifTable)
    static void WalkInterfaceTable(string ip, string community)
    {
        var endpoint = new IPEndPoint(IPAddress.Parse(ip), 161);
        var results = new System.Collections.Generic.List<Variable>();

        // Кореневий OID таблиці інтерфейсів: 1.3.6.1.2.1.2.2.1 (ifEntry)
        var rootOid = new ObjectIdentifier("1.3.6.1.2.1.2.2.1");

        // Messenger.Walk сам виконує серію GET-NEXT (або GET-BULK для v2/v3)
        // під капотом, поки не вийде за межі піддерева rootOid
        Messenger.Walk(
            VersionCode.V2,
            endpoint,
            new OctetString(community),
            rootOid,
            results,
            timeout: 5000,
            WalkMode.WithinSubtree);

        foreach (Variable v in results)
        {
            Console.WriteLine($"{v.Id} = {v.Data}");
        }
    }

    static void Main()
    {
        const string deviceIp = "192.168.1.1";
        const string community = "public";

        Console.WriteLine("=== SNMP GET: sysDescr.0 ===");
        GetSysDescr(deviceIp, community);

        Console.WriteLine("\n=== SNMP WALK: ifTable (interfaces) ===");
        WalkInterfaceTable(deviceIp, community);
    }
}
```

**Приклад очікуваного виводу (дані пристрою — умовні, для ілюстрації):**

```
=== SNMP GET: sysDescr.0 ===
1.3.6.1.2.1.1.1.0 = Cisco IOS Software, C2960X Software, Version 15.2(7)E3

=== SNMP WALK: ifTable (interfaces) ===
1.3.6.1.2.1.2.2.1.1.1 = 1                          (ifIndex)
1.3.6.1.2.1.2.2.1.2.1 = GigabitEthernet0/1          (ifDescr)
1.3.6.1.2.1.2.2.1.3.1 = 6                           (ifType = ethernetCsmacd)
1.3.6.1.2.1.2.2.1.5.1 = 1000000000                  (ifSpeed = 1 Gbps)
1.3.6.1.2.1.2.2.1.8.1 = 1                           (ifOperStatus = up)
1.3.6.1.2.1.2.2.1.10.1 = 458213765231                (ifInOctets)
1.3.6.1.2.1.2.2.1.16.1 = 927481022119                (ifOutOctets)
1.3.6.1.2.1.2.2.1.1.2 = 2
1.3.6.1.2.1.2.2.1.2.2 = GigabitEthernet0/2
1.3.6.1.2.1.2.2.1.8.2 = 2                           (ifOperStatus = down)
...
```

> Реальна система моніторингу (наприклад, SolarWinds NPM) далі "збирає" ці плоскі рядки в структуровані об'єкти (по одному на кожен `ifIndex`), зіставляючи `ifDescr`, `ifOperStatus`, `ifInOctets` тощо за спільним індексом рядка таблиці, і зберігає результат у базі для побудови графіків трафіку та алертів "інтерфейс впав".

### Приклад 3 — SnmpPoller з логікою retry + timeout

Реалізація патерну "N спроб з таймаутом перед тим, як вважати пристрій недоступним", специфічно для SNMP-опитування.

```csharp
using System;
using System.Threading.Tasks;

class SnmpPoller
{
    private readonly int _maxRetries;
    private readonly TimeSpan _timeout;

    public SnmpPoller(int maxRetries = 3, int timeoutMs = 1500)
    {
        _maxRetries = maxRetries;
        _timeout = TimeSpan.FromMilliseconds(timeoutMs);
    }

    // Симулюємо ненадійний SNMP GET: перша спроба губиться (як типово буває з UDP),
    // друга — вдала. Реальна реалізація тут викликала б Messenger.Get(...) з SharpSnmpLib.
    private static int _attemptCounter = 0;

    private Task<string?> SimulatedSnmpGetAsync(string oid)
    {
        _attemptCounter++;
        if (_attemptCounter == 1)
        {
            // Симуляція втраченого UDP-пакета на першій спробі
            return Task.FromResult<string?>(null);
        }
        return Task.FromResult<string?>("Timeticks: 1234567 (3 days, 10:17:35.67)");
    }

    public async Task<string> PollWithRetryAsync(string deviceIp, string oid)
    {
        for (int attempt = 1; attempt <= _maxRetries; attempt++)
        {
            Console.WriteLine($"[Poller] Спроба {attempt}/{_maxRetries}: GET {oid} на {deviceIp} (timeout={_timeout.TotalMilliseconds}мс)");

            using var cts = new System.Threading.CancellationTokenSource(_timeout);
            var getTask = SimulatedSnmpGetAsync(oid);
            var completed = await Task.WhenAny(getTask, Task.Delay(_timeout, cts.Token));

            if (completed == getTask && getTask.Result is not null)
            {
                Console.WriteLine($"[Poller] ✅ Успішна відповідь на спробі {attempt}: {getTask.Result}");
                return getTask.Result;
            }

            Console.WriteLine($"[Poller] ❌ Спроба {attempt} невдала (timeout або втрачений пакет)");
        }

        throw new TimeoutException(
            $"Пристрій {deviceIp} не відповів на OID {oid} після {_maxRetries} спроб — позначаємо як недоступний (Down)");
    }

    static async Task Main()
    {
        var poller = new SnmpPoller(maxRetries: 3, timeoutMs: 1500);
        try
        {
            string result = await poller.PollWithRetryAsync("192.168.1.1", "1.3.6.1.2.1.1.3.0");
            Console.WriteLine($"[Main] Фінальний результат опитування: {result}");
        }
        catch (TimeoutException ex)
        {
            Console.WriteLine($"[Main] {ex.Message}");
        }
    }
}
```

**Очікуваний вивід:**

```
[Poller] Спроба 1/3: GET 1.3.6.1.2.1.1.3.0 на 192.168.1.1 (timeout=1500мс)
[Poller] ❌ Спроба 1 невдала (timeout або втрачений пакет)
[Poller] Спроба 2/3: GET 1.3.6.1.2.1.1.3.0 на 192.168.1.1 (timeout=1500мс)
[Poller] ✅ Успішна відповідь на спробі 2: Timeticks: 1234567 (3 days, 10:17:35.67)
[Main] Фінальний результат опитування: Timeticks: 1234567 (3 days, 10:17:35.67)
```

Саме такий шаблон — **не позначати пристрій "Down" після однієї невдалої спроби** — критично важливий у продуктах типу SolarWinds NPM, де хибний "false positive" алерт про недоступність тисяч пристроїв через тимчасову втрату одного UDP-пакета зробив би систему моніторингу непридатною для довіри.

### Приклад 4 — простий прийомник SNMP TRAP (UDP/162)

```csharp
using System;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Threading.Tasks;

// Спрощений (не-ASN.1) прийомник, що ілюструє: менеджер пасивно слухає traps,
// поки паралельно (в іншому потоці/задачі) виконує активний polling.
class SimpleTrapListener
{
    private const int TrapPort = 162;

    public async Task ListenAsync(System.Threading.CancellationToken token)
    {
        using var udpServer = new UdpClient(TrapPort);
        Console.WriteLine($"[TrapListener] Слухаю вхідні traps на UDP/{TrapPort}...");

        while (!token.IsCancellationRequested)
        {
            try
            {
                var result = await udpServer.ReceiveAsync(token);
                string message = Encoding.UTF8.GetString(result.Buffer);

                Console.WriteLine($"[TrapListener] 🔔 Отримано TRAP від {result.RemoteEndPoint.Address}: \"{message}\"");
                LogTrap(result.RemoteEndPoint.Address.ToString(), message);
            }
            catch (OperationCanceledException)
            {
                break;
            }
        }
    }

    private void LogTrap(string sourceIp, string message)
    {
        // У реальній системі тут був би запис в БД подій, генерація алерту,
        // можливо — ескалація (email/SMS/Slack), кореляція з іншими подіями тощо.
        Console.WriteLine($"[TrapListener] Записано подію в журнал: пристрій={sourceIp}, повідомлення=\"{message}\", час={DateTime.UtcNow:O}");
    }
}

// Симулятор агента, що надсилає trap "interface down"
class TrapSender
{
    public static async Task SendLinkDownTrapAsync(string managerIp)
    {
        using var client = new UdpClient();
        string trapMessage = "TRAP linkDown: ifIndex=3, ifDescr=GigabitEthernet0/3, ifOperStatus=down";
        byte[] data = Encoding.UTF8.GetBytes(trapMessage);
        await client.SendAsync(data, new IPEndPoint(IPAddress.Parse(managerIp), 162));
        Console.WriteLine($"[Agent] Надіслано TRAP на менеджер {managerIp}:162 — \"{trapMessage}\"");
    }
}

class Program
{
    static async Task Main()
    {
        using var cts = new System.Threading.CancellationTokenSource();
        var listener = new SimpleTrapListener();
        var listenTask = listener.ListenAsync(cts.Token);

        await Task.Delay(300); // час на старт listener-а

        // Симулюємо, що якийсь пристрій виявив падіння лінка й надіслав TRAP
        await TrapSender.SendLinkDownTrapAsync("127.0.0.1");

        await Task.Delay(500);
        cts.Cancel();
    }
}
```

**Очікуваний вивід:**

```
[TrapListener] Слухаю вхідні traps на UDP/162...
[Agent] Надіслано TRAP на менеджер 127.0.0.1:162 — "TRAP linkDown: ifIndex=3, ifDescr=GigabitEthernet0/3, ifOperStatus=down"
[TrapListener] 🔔 Отримано TRAP від 127.0.0.1: "TRAP linkDown: ifIndex=3, ifDescr=GigabitEthernet0/3, ifOperStatus=down"
[TrapListener] Записано подію в журнал: пристрій=127.0.0.1, повідомлення="TRAP linkDown: ifIndex=3, ifDescr=GigabitEthernet0/3, ifOperStatus=down", час=2026-08-23T...
```

> Примітка: у реальному .NET-коді слухач traps теж зазвичай реалізується через `Lextm.SharpSnmpLib` (клас `TrapListener` з бібліотеки), який вже вміє декодувати справжні ASN.1-закодовані SNMPv1/v2c/v3 trap-повідомлення. Наведений приклад — спрощена ілюстрація архітектурної ролі такого компонента (пасивний UDP-listener на 162 порту, що працює паралельно з активним пулінгом).

---

## 7. Реальний сценарій: SNMP як основа мережевого моніторингу (SolarWinds)

Розглянемо, як продукт на кшталт **SolarWinds NPM (Network Performance Monitor)** насправді використовує SNMP "під капотом". Це саме той рівень деталізації, який очікують на технічній співбесіді в компанії такого профілю.

### 7.1. Device Discovery (виявлення пристроїв)

Коли адміністратор додає новий підрозділ мережі в моніторинг, продукт спочатку проводить **discovery** (наприклад, сканування діапазону IP-адрес):

1. Для кожної відповідної IP-адреси надсилається SNMP GET на OID `sysObjectID` (`1.3.6.1.2.1.1.2.0`) — це унікальний ідентифікатор, який каже, **що це за пристрій** (наприклад, конкретна модель маршрутизатора Cisco чи комутатора HP). За цим значенням продукт визначає **vendor і тип пристрою**, і відповідно — який набір MIB/OID варто далі опитувати (наприклад, чи є сенс питати про temperature sensors, чи це проста некерована точка доступу без таких даних).
2. Паралельно опитується `sysDescr` (текстовий опис — версія прошивки, модель) та виконується "легкий" walk по `system` та базовій частині `ifTable`, щоб зрозуміти, скільки в пристрою інтерфейсів і які вони.
3. На основі цих даних система будує **інвентар (asset inventory)**: тип пристрою, вендор, кількість портів, версія ОС — усе це стає основою для подальшого специфічного моніторингу (наприклад, "це UPS — треба опитувати battery MIB", "це Cisco-маршрутизатор — можна показати температуру CPU через Cisco-специфічний OID").

### 7.2. Polling Engine (основний цикл опитування)

Це серце продукту. Через задані інтервали (типово, наприклад, кожні 60 секунд для статусу інтерфейсів, кожні 9 хвилин для "важчих" даних, налаштовується) polling engine надсилає GET/GET-BULK запити на набір ключових OID для кожного пристрою:

| Метрика | OID (приклад) | Призначення |
|---|---|---|
| CPU load | вендор-специфічний (напр. Cisco `1.3.6.1.4.1.9.9.109...`) або `hrProcessorLoad` (`1.3.6.1.2.1.25.3.3.1.2`) | Навантаження на процесор |
| Пам'ять | `hrStorageUsed` / вендор-специфічний | Використання оперативної пам'яті |
| Вхідний трафік інтерфейсу | `ifInOctets` (`1.3.6.1.2.1.2.2.1.10.X`) | Байти, отримані на інтерфейсі X |
| Вихідний трафік інтерфейсу | `ifOutOctets` (`1.3.6.1.2.1.2.2.1.16.X`) | Байти, надіслані з інтерфейсу X |
| Статус інтерфейсу (адмін.) | `ifAdminStatus` (`1.3.6.1.2.1.2.2.1.7.X`) | Чи інтерфейс адміністративно увімкнений |
| Статус інтерфейсу (робочий) | `ifOperStatus` (`1.3.6.1.2.1.2.2.1.8.X`) | Чи інтерфейс фактично працює (up/down) |

Отримані "сирі" лічильники (наприклад, `ifInOctets`) самі по собі — це просто накопичувальні числа з моменту рестарту пристрою. Щоб отримати корисну метрику "швидкість трафіку зараз" (Mbps), продукт бере **різницю (delta)** між двома послідовними опитуваннями і ділить на час між ними.

### 7.3. Проблема переповнення лічильників (Counter Rollover)

Це **справжня, добре відома в індустрії проблема**, яку варто знати детально.

Класичні лічильники MIB-II (`ifInOctets`, `ifOutOctets` та інші, визначені як тип `Counter32`) — це **32-бітні беззнакові цілі числа**. Максимальне значення 32-бітного лічильника:

```
2^32 - 1 = 4 294 967 295  (≈ 4.29 мільярда)
```

На дуже завантаженому інтерфейсі (наприклад, 1 Gbps лінк, що працює близько до повного навантаження) лічильник байтів може **"перегорнутися" (wrap around) до нуля менш ніж за хвилину**:

```
1 Gbps ≈ 125 000 000 байт/сек
4 294 967 295 байт / 125 000 000 байт/сек ≈ 34 секунди до повного переповнення
```

Тобто на гігабітному інтерфейсі під повним навантаженням лічильник обнуляється **кожні ~34 секунди**! Якщо система моніторингу опитує рідше (наприклад, раз на хвилину) і **не враховує** можливість переповнення, вона побачить, що нове значення менше за попереднє, і або (а) неправильно порахує "від'ємний" трафік, або (б) якщо наївно обробить це як помилку — втратить дані за весь інтервал.

**Рішення 1 — правильна обробка rollover у самому коді:** якщо нове значення лічильника менше за попереднє, вважати, що стався один (чи кілька) циклів переповнення, і додати `2^32` (для Counter32) до різниці:

```csharp
using System;

static class CounterDeltaCalculator
{
    private const long Counter32Max = 4_294_967_296L; // 2^32

    /// <summary>
    /// Обчислює коректну дельту між двома вимірюваннями 32-бітного лічильника,
    /// враховуючи можливе переповнення (rollover) за час між опитуваннями.
    /// </summary>
    public static long CalculateDelta(uint previousValue, uint currentValue)
    {
        if (currentValue >= previousValue)
        {
            // Звичайний випадок: лічильник просто зріс
            return currentValue - previousValue;
        }

        // currentValue < previousValue → трапилось переповнення (rollover).
        // Рахуємо: скільки лишилось "до кінця" 32-бітного діапазону від previousValue,
        // плюс скільки вже "накопичилось" від нуля до currentValue.
        long delta = (Counter32Max - previousValue) + currentValue;
        return delta;
    }

    static void Main()
    {
        // Приклад: за 60 секунд лічильник переповнився один раз
        uint previous = 4_294_960_000;   // майже на межі 2^32
        uint current  = 50_000;          // "перегорнувся" і почав рахувати заново

        long delta = CalculateDelta(previous, current);
        double bitsPerSecond = (delta * 8.0) / 60.0; // припустимо, інтервал опитування 60 сек

        Console.WriteLine($"Попереднє значення лічильника: {previous}");
        Console.WriteLine($"Поточне значення лічильника:   {current}");
        Console.WriteLine($"Коректна дельта (з урахуванням rollover): {delta} байт");
        Console.WriteLine($"Розрахована швидкість: {bitsPerSecond / 1_000_000:F2} Mbps");

        // Наївний (НЕПРАВИЛЬНИЙ) розрахунок без урахування rollover для порівняння:
        long naiveDelta = (long)current - previous; // буде від'ємним!
        Console.WriteLine($"❌ Наївна (хибна) дельта без урахування rollover: {naiveDelta} (від'ємне число — явна ознака бага!)");
    }
}
```

**Вивід:**

```
Попереднє значення лічильника: 4294960000
Поточне значення лічильника:   50000
Коректна дельта (з урахуванням rollover): 57296 байт
Розрахована швидкість: 0.01 Mbps
❌ Наївна (хибна) дельта без урахування rollover: -4294910000 (від'ємне число — явна ознака бага!)
```

**Рішення 2 — 64-бітні "High Capacity" (HC) лічильники.** Починаючи з SNMPv2 та розширеного MIB (`IF-MIB`, RFC 2233/2863), для швидкісних інтерфейсів визначені **64-бітні** варіанти лічильників — `ifHCInOctets`, `ifHCOutOctets` (тип `Counter64`). З 64-бітним діапазоном (`2^64 ≈ 1.8×10^19`) переповнення на практиці стає майже неможливим навіть на найшвидших сучасних лінках (десятки років безперервного навантаження на 100+ Gbps). Тому **добра практика систем моніторингу** (і те, що робить SolarWinds та подібні продукти) — **завжди пріоритизувати HC-лічильники, якщо пристрій їх підтримує**, і лише як fallback використовувати звичайні 32-бітні з обов'язковою rollover-логікою для старішого обладнання, що HC не підтримує.

> 📊 На співбесіді ця тема (rollover 32-бітних лічильників) — один з найпопулярніших "практичних" запитань саме для компаній network monitoring, бо це реальний, неочевидний баг, з яким стикається кожен розробник polling-логіки.

### 7.4. Traps як доповнення до polling: швидкість реакції

Polling має фундаментальне обмеження: дані **свіжі рівно настільки, наскільки частий інтервал опитування**. Якщо інтервал — одна хвилина, а лінк впав одразу після опитування, менеджер дізнається про це аж через майже хвилину (при наступному polling циклі).

**TRAP-и заповнюють цю прогалину**: коли на пристрої стається подія (наприклад, `linkDown`/`linkUp` — стандартні generic traps з MIB-II, чи `coldStart`/`warmStart` при перезавантаженні), агент **негайно** (в момент події, а не при наступному опитуванні) відправляє TRAP менеджеру. Це дає системі моніторингу набагато швидшу (майже реального часу) реакцію на критичні події, ніж чекати наступного polling-циклу.

Тому добре спроєктована система (як SolarWinds NPM) використовує **обидва механізми одночасно**:
- **Polling** — регулярний, передбачуваний "пульс" стану всіх метрик (навіть якщо нічого не змінилось — підтверджує, що пристрій живий і відповідає);
- **Traps/Informs** — миттєва реакція на конкретні події, які не варто чекати до наступного polling-циклу.

Це дає найкраще з обох світів: traps — швидкість реакції, polling — надійність і повноту даних (адже traps можуть загубитись в дорозі — UDP, а polling рано чи пізно все одно виявить проблему при наступному циклі, навіть якщо trap не дійшов).

### 7.5. Кастомні/вендорські MIB у SolarWinds

Стандартний MIB-II покриває базові речі (uptime, інтерфейси), але не покриває специфічні для вендора дані: температуру CPU Cisco-маршрутизатора, стан батареї APC UPS, рівень тонера принтера HP, стан RAID-масиву на сервері Dell. Ці дані лежать під `1.3.6.1.4.1.<vendor-id>.*` і **не мають людських назв "з коробки"** — доки продукт моніторингу не знає, як їх інтерпретувати.

Тому SolarWinds (і аналогічні продукти) надають механізм **завантаження (компіляції) MIB-файлів**: адміністратор імпортує `.mib`-файл, наданий виробником обладнання (наприклад, `APC-UPS-MIB.mib`), і продукт "вчиться" зіставляти числові OID з людськими назвами й типами даних із цього файлу — після чого можна будувати "Universal Device Poller" (UnDP у термінології SolarWinds) — кастомний монітор конкретного вендорського OID, що додається до стандартного набору метрик пристрою.

Це критично важливо розуміти: **функціональність продукту прямо залежить від якості й повноти набору MIB, які він вміє інтерпретувати** — чим більше вендорських MIB "з коробки" підтримує продукт, тим ширше коло пристроїв воно може моніторити "з коробки" без ручної конфігурації користувачем.

---

## 8. Поширені проблеми та діагностика

| Проблема | Типові причини | Як діагностувати |
|---|---|---|
| **Timeout на GET-запиті** (агент не відповідає) | Невірний community string / user credentials; ACL на пристрої блокує IP менеджера; firewall блокує UDP/161; SNMP-сервіс вимкнений на пристрої; неправильна SNMP-версія (напр. запит v2c до агента, налаштованого лише на v3) | `snmpget -v2c -c public <ip> 1.3.6.1.2.1.1.1.0` з тієї ж машини, де стоїть менеджер; перевірити firewall-правила (`telnet <ip> 161` для базової перевірки досяжності порту — хоч UDP і не про "з'єднання"); перевірити community/credentials на самому пристрої |
| **Traps не приходять на менеджер** | Firewall блокує вхідний UDP/162 на менеджері; на пристрої неправильно (або взагалі не) налаштована trap destination адреса менеджера; NAT між пристроєм і менеджером ламає зворотну адресу | Перевірити конфігурацію "trap receiver"/"SNMP host" на пристрої (чи вказана правильна IP і порт менеджера); зняти трафік через Wireshark на менеджері, фільтр `udp.port==162`, щоб побачити, чи взагалі приходять пакети |
| **Лічильники трафіку раптово "падають" до нуля / стають від'ємними** | Класичний **32-бітний rollover лічильника** (`ifInOctets`/`ifOutOctets`) на високонавантаженому інтерфейсі; продукт моніторингу не використовує HC (`ifHCInOctets`) варіант і/або неправильно обробляє rollover | Перевірити, чи пристрій/MIB підтримує `ifHCInOctets`/`ifHCOutOctets` (64-біт); перевірити логіку розрахунку delta в коді — чи враховує wraparound (див. розділ 7.3) |
| **Часткові/неповні дані з WALK** (обхід обривається раніше, ніж очікувалось) | MIB для конкретного OID не "завантажений"/не розпізнається (для vendor-специфічних гілок); OID недоступний на цій конкретній моделі/версії прошивки пристрою; VACM (у v3) обмежує доступ лише до частини дерева | Спробувати `snmpwalk` вручну на конкретному піддереві й порівняти з очікуваним; перевірити версію прошивки пристрою на предмет підтримки конкретного MIB; перевірити налаштування Views/VACM для v3-користувача |
| **SET не спрацьовує** | Community string лише read-only ("public" замість "private"/read-write); для v3 — недостатні права у VACM (write view не налаштовано для user) | Перевірити, чи взагалі дозволений запис на цьому OID (деякі OID read-only за визначенням MIB); перевірити права доступу community/user |
| **SNMPv3-запит відхиляється / "authentication failure"** | Невірний пароль автентифікації; невідповідний алгоритм (SHA замість MD5 або навпаки); неправильний час (SNMPv3 чутливий до розсинхронізації часу — "clock skew" між менеджером і агентом) | Перевірити, що auth/priv паролі й алгоритми ідентичні на менеджері й агенті; перевірити синхронізацію часу (NTP) на обох сторонах |

### Стандартні інструменти діагностики

- **`snmpget`, `snmpwalk`, `snmpbulkwalk`, `snmptrap`** — набір CLI-утиліт з пакету **Net-SNMP** — де-факто стандарт для ручного тестування SNMP з командного рядка (доступний на Linux, macOS, є порти для Windows). Приклад:
  ```bash
  snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.2.2.1
  ```
- **MIB Browser** (наприклад, iReasoning MIB Browser, або вбудований у SolarWinds/PRTG) — GUI-інструмент, що дозволяє візуально гортати дерево MIB, робити GET/WALK, завантажувати кастомні MIB-файли й одразу бачити людські назви OID замість голих чисел.
- **Wireshark** — для перегляду сирих SNMP-пакетів на дроті (корисно для діагностики проблем з firewall, community strings, форматом trap-повідомлень).

---

## 9. Питання та відповіді для співбесіди

**1. Що таке SNMP і для чого він використовується?**
SNMP (Simple Network Management Protocol) — стандартний протокол прикладного рівня для моніторингу й управління мережевими пристроями (маршрутизаторами, комутаторами, серверами, UPS тощо) віддалено. Він лежить в основі практично всіх систем мережевого моніторингу, включно з SolarWinds Orion/NPM.

**2. Поясни архітектуру Manager-Agent-MIB.**
**Manager** — центральна система моніторингу, що ініціює запити й збирає/аналізує дані (наприклад, SolarWinds Polling Engine). **Agent** — програмний компонент на самому пристрої, що зберігає дані та відповідає на запити. **MIB (Management Information Base)** — ієрархічна стандартизована схема/словник, що описує, які саме дані доступні на агенті і як вони структуровані (типи, назви, адреси-OID).

**3. Що таке OID, наведи приклад.**
OID (Object Identifier) — унікальна ієрархічна крапко-числова адреса конкретного елементу даних у дереві MIB. Наприклад, `1.3.6.1.2.1.1.3.0` — це `sysUpTime.0` (час роботи системи), а `1.3.6.1.2.1.1.1.0` — `sysDescr.0` (опис пристрою).

**4. Чим відрізняються SNMPv1, v2c, v3?**
v1 — найпростіша версія, лише GET/GET-NEXT/SET/TRAP, безпека — лише plaintext community string. v2c додає GET-BULK (ефективний масовий запит) та INFORM (trap з підтвердженням), але зберігає ту саму незахищену модель community string. v3 додає справжню безпеку: автентифікацію (MD5/SHA), шифрування (DES/AES) через модель USM, і гранулярний контроль доступу через VACM.

**5. Чому SNMPv3 вважається безпечнішим?**
Тому що community strings у v1/v2c передаються у відкритому вигляді (plaintext) і можуть бути легко перехоплені сніффінгом трафіку, після чого зловмисник отримує read (а можливо й write) доступ до пристрою. SNMPv3 додає автентифікацію користувача (перевірку, що пакет справді від довіреного джерела й не змінений) і опційне шифрування вмісту пакета, роблячи перехоплення й підробку значно складнішими.

**6. Чим TRAP відрізняється від INFORM?**
Обидва — проактивні сповіщення від агента менеджеру про подію. TRAP не потребує і не отримує підтвердження — "fire-and-forget", і якщо пакет загубиться (UDP не гарантує доставку), менеджер про подію просто не дізнається. INFORM (з'явився в SNMPv2c) — те ж саме, але з ack: менеджер підтверджує отримання, і якщо ack не прийшов, агент повторює відправку — це надійніше, але дорожче в реалізації (потрібно тримати стан очікування підтвердження).

**7. Чому SNMP працює по UDP, а не TCP?**
Через легковаговість: SNMP-обмін типово короткий (один запит-відповідь), і TCP-handshake/утримання з'єднання додавали б непропорційні накладні витрати, особливо коли менеджер одночасно опитує тисячі пристроїв. Надійність доставки реалізується на прикладному рівні самого SNMP (retry-логіка менеджера для запитів, INFORM з ack для сповіщень), а не нав'язується транспортом.

**8. Що таке SNMP WALK і як він реалізується технічно?**
SNMP WALK — техніка обходу цілого піддерева OID (наприклад, усієї таблиці інтерфейсів), а не окрема команда протоколу. Реалізується через послідовні виклики **GET-NEXT**: кожен запит повертає значення наступного за порядком OID, і клієнт продовжує запитувати "наступний", поки не вийде за межі потрібного піддерева. У SNMPv2c/v3 walk часто прискорюють через **GET-BULK**, що повертає одразу багато записів за один round-trip.

**9. Що таке community string і чому це небезпечно в v1/v2c?**
Community string — простий текстовий "пароль" (типово `"public"` для читання, `"private"` для запису), що передається в кожному SNMP-пакеті v1/v2c у **відкритому вигляді**. Небезпека в тому, що будь-хто, хто може перехопити трафік у мережі (sniffing), одразу бачить цей рядок і, за наявності мережевого доступу до пристрою, отримує такий самий рівень доступу (read чи навіть write), не потребуючи жодного зламу.

**10. Поясни проблему переповнення (rollover) 32-бітних лічильників інтерфейсу.**
Стандартні лічильники трафіку (`ifInOctets`, `ifOutOctets`) типу `Counter32` — 32-бітні, максимум ≈4.29 млрд. На високонавантаженому інтерфейсі (наприклад, 1 Gbps під повним навантаженням) лічильник може переповнюватись (обнулятись) кожні ~34 секунди. Система моніторингу повинна детектувати ситуацію "нове значення менше за попереднє" й коректно рахувати дельту, додаючи `2^32` до різниці, або (краще) використовувати 64-бітні HC-лічильники (`ifHCInOctets`), де переповнення практично неможливе.

**11. Які порти використовує SNMP і для чого кожен?**
UDP/161 — порт агента, куди менеджер надсилає GET/GET-NEXT/GET-BULK/SET запити і звідки отримує відповіді. UDP/162 — порт менеджера, на якому він слухає вхідні TRAP та INFORM повідомлення від агентів.

**12. Що таке GET-BULK і чим він кращий за послідовні GET-NEXT?**
GET-BULK (з'явився в SNMPv2c) дозволяє за один запит отримати одразу багато послідовних значень з таблиці MIB, замість того, щоб робити окремий round-trip GET-NEXT для кожного рядка. Це суттєво зменшує кількість мережевих round-trip-ів і затримку при обході великих таблиць (наприклад, таблиці інтерфейсів на пристрої з сотнями портів).

**13. Що таке MIB і чим він відрізняється від OID?**
MIB — це вся ієрархічна схема/база даних-специфікація, що описує доступні дані пристрою (їх назви, типи, структуру). OID — конкретна "адреса" одного елементу всередині цієї схеми. Тобто MIB — це "карта", а OID — координати конкретної точки на цій карті.

**14. Навіщо потрібні вендор-специфічні (proprietary) MIB, і де вони розташовані в дереві OID?**
Стандартний MIB-II покриває лише загальні, універсальні дані (uptime, інтерфейси, IP-статистика). Специфічні для конкретного виробника дані (температура CPU Cisco, заряд батареї APC UPS тощо) описуються у власних MIB виробника, розташованих під гілкою `1.3.6.1.4.1.<enterprise-id>` (`enterprises`), де `enterprise-id` — унікальний номер, виданий IANA конкретній компанії.

**15. Як менеджер моніторингу дізнається, з яким типом пристрою він має справу, при discovery?**
Через SNMP GET на OID `sysObjectID` (`1.3.6.1.2.1.1.2.0`), який повертає унікальний ідентифікатор моделі/вендора пристрою. На основі цього значення система вирішує, який набір OID/MIB далі опитувати (наприклад, чи має сенс перевіряти сенсори температури, чи це проста ненавантажена точка доступу).

**16. Чому важливо не позначати пристрій "Down" після однієї невдалої SNMP-спроби?**
Тому що SNMP працює по UDP, який не гарантує доставку — окремий втрачений пакет (запит чи відповідь) є нормальним, очікуваним явищем у мережі, а не обов'язково ознакою реальної проблеми з пристроєм. Правильна практика — реалізувати retry з таймаутом (кілька спроб) і позначати пристрій недоступним лише якщо провалились усі спроби підряд.

**17. Чим відрізняються ifOperStatus і ifAdminStatus?**
`ifAdminStatus` — адміністративно заданий стан інтерфейсу (чи адміністратор увімкнув/вимкнув порт, наприклад командою `shutdown` на Cisco). `ifOperStatus` — фактичний робочий стан інтерфейсу "по факту" (наприклад, порт адміністративно увімкнений, але фізично `down`, бо кабель не підключений). Система моніторингу відслідковує обидва, щоб відрізняти навмисно вимкнені порти від реальних збоїв.

**18. Що таке VACM у SNMPv3?**
View-based Access Control Model — модель контролю доступу у SNMPv3, що дозволяє гранулярно визначати, які саме частини дерева MIB (views) конкретний користувач чи група користувачів можуть читати чи записувати, замість спрощеної моделі "весь MIB доступний або ні", характерної для v1/v2c community strings.

**19. Опишіть різницю між "authNoPriv" і "authPriv" у SNMPv3.**
`authNoPriv` — застосовується лише автентифікація (перевірка, що пакет справді від очікуваного джерела й не був змінений), але вміст пакета передається у відкритому вигляді (не шифрується). `authPriv` — застосовується і автентифікація, і шифрування (privacy) вмісту пакета, найвищий рівень безпеки з трьох, доступних у SNMPv3.

**20. Чому polling і traps використовуються разом, а не окремо?**
Polling дає передбачуваний, регулярний "пульс" стану всіх метрик, але його дані свіжі лише настільки, наскільки частий інтервал опитування — між опитуваннями може статись і залишитись непоміченою критична подія. Traps дають майже миттєву реакцію на конкретні події (наприклад, падіння лінка), але можуть загубитись у дорозі, бо надсилаються по UDP без гарантії доставки (якщо це не INFORM). Разом вони компенсують слабкі сторони одне одного: traps — швидкість, polling — надійність і повнота.

---

## 10. Підсумок

### Версії SNMP

| | v1 | v2c | v3 |
|---|---|---|---|
| Безпека | Community string (plaintext) | Community string (plaintext) | User-based (auth: MD5/SHA, priv: DES/AES) |
| GET-BULK | ❌ | ✅ | ✅ |
| INFORM | ❌ | ✅ | ✅ |
| Counter64 | ❌ | ✅ | ✅ |
| Контроль доступу | Немає | Немає | VACM (гранулярний) |
| Рекомендація | Легасі | Найпоширеніша практично | Рекомендована для нових розгортань |

### Ключові OID для моніторингу

| OID | Назва | Що показує |
|---|---|---|
| `1.3.6.1.2.1.1.1.0` | `sysDescr.0` | Опис пристрою (модель, версія ПЗ) |
| `1.3.6.1.2.1.1.2.0` | `sysObjectID.0` | Ідентифікатор моделі/вендора (для discovery) |
| `1.3.6.1.2.1.1.3.0` | `sysUpTime.0` | Час роботи з моменту останнього рестарту |
| `1.3.6.1.2.1.1.5.0` | `sysName.0` | Ім'я пристрою (hostname) |
| `1.3.6.1.2.1.2.2.1.10.X` | `ifInOctets.X` | Вхідний трафік інтерфейсу X (32-біт, схильний до rollover) |
| `1.3.6.1.2.1.2.2.1.16.X` | `ifOutOctets.X` | Вихідний трафік інтерфейсу X (32-біт) |
| `1.3.6.1.2.1.31.1.1.1.6.X` | `ifHCInOctets.X` | Вхідний трафік інтерфейсу X (64-біт, HC-варіант) |
| `1.3.6.1.2.1.31.1.1.1.10.X` | `ifHCOutOctets.X` | Вихідний трафік інтерфейсу X (64-біт, HC-варіант) |
| `1.3.6.1.2.1.2.2.1.7.X` | `ifAdminStatus.X` | Адміністративний стан інтерфейсу X |
| `1.3.6.1.2.1.2.2.1.8.X` | `ifOperStatus.X` | Фактичний робочий стан інтерфейсу X (up/down) |

### Порти

| Порт | Транспорт | Призначення |
|---|---|---|
| **161** | UDP | Запити менеджера до агента (GET/GET-NEXT/GET-BULK/SET) і відповіді агента |
| **162** | UDP | Trap/Inform повідомлення від агента до менеджера |

### Операції — коротко

| Операція | Ким ініційована | Підтвердження |
|---|---|---|
| GET / GET-NEXT / GET-BULK / SET | Manager | Так (RESPONSE) |
| TRAP | Agent | ❌ Ні |
| INFORM | Agent | ✅ Так (ack) |

### Три речі, які варто пам'ятати найкраще для співбесіди в SolarWinds-подібну компанію

1. **UDP + retry-логіка на боці менеджера** — один втрачений пакет ≠ пристрій недоступний.
2. **Rollover 32-бітних лічильників** — реальна, часта практична проблема; рішення — HC (64-біт) лічильники або коректна delta-логіка з урахуванням переповнення.
3. **v1/v2c community strings передаються у відкритому вигляді** — фундаментальна причина, чому SNMPv3 (authPriv) рекомендований усюди, де це можливо.

---

*Документ підготовлено для підготовки до технічної співбесіди. Матеріал орієнтовано на компанії у сфері мережевого моніторингу та управління (Network Management) — SNMP є ключовою темою для SolarWinds.*
