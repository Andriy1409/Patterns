# Комутація та VLAN — Детальний розбір для співбесіди

> **Категорія:** Комп'ютерні мережі (Data Link Layer)  
> **Контекст:** Підготовка до співбесіди (Network Management / SolarWinds)  
> **Мова прикладів:** C# (.NET)

---

## Зміст

1. [Що таке комутатор (switch) і як він працює?](#1-що-таке-комутатор-switch-і-як-він-працює)
2. [Broadcast-домени та Collision-домени](#2-broadcast-домени-та-collision-домени)
3. [Що таке VLAN і навіщо він потрібен](#3-що-таке-vlan-і-навіщо-він-потрібен)
4. [VLAN tagging — 802.1Q](#4-vlan-tagging--8021q)
5. [Inter-VLAN Routing](#5-inter-vlan-routing)
6. [Spanning Tree Protocol (STP) — коротко](#6-spanning-tree-protocol-stp--коротко)
7. [Практичні приклади на C#](#7-практичні-приклади-на-c)
8. [Реальний сценарій: чому VLAN/switching важливі для мережевого моніторингу](#8-реальний-сценарій-чому-vlanswitching-важливі-для-мережевого-моніторингу)
9. [Поширені проблеми та діагностика](#9-поширені-проблеми-та-діагностика)
10. [Питання та відповіді для співбесіди](#10-питання-та-відповіді-для-співбесіди)
11. [Підсумок](#11-підсумок)

---

## 1. Що таке комутатор (switch) і як він працює?

Комутатор (switch) — це пристрій **другого рівня моделі OSI (Data Link Layer, L2)**, який пересилає кадри (frames) на основі **MAC-адрес**. На відміну від хаба (hub), який працює на фізичному рівні (L1) і просто повторює електричний сигнал на всі порти, комутатор аналізує заголовок Ethernet-кадру і приймає "розумне" рішення — куди саме надіслати цей кадр.

### 🧩 Реальна аналогія

Уявіть офіс з поштовим відділенням.

- **Хаб** — це як людина, яка отримує лист і **вигукує його вміст на весь будинок**, сподіваючись, що потрібний адресат почує. Усі кабінети чують усе, незалежно від того, кому лист адресований. Це неефективно і небезпечно (усі бачать чужу пошту).
- **Комутатор** — це досвідчений сортувальник пошти. Коли він отримує **перший** лист від співробітника з кабінету №5, він **запам'ятовує**: "автор цього листа сидить за дверима №5". Наступного разу, коли хтось надсилає лист **цьому** співробітнику, сортувальник вже точно знає, в які двері постукати, і не турбує решту офісу.

### Як комутатор приймає рішення: MAC Address Table (CAM Table)

Комутатор веде таблицю відповідності `MAC-адреса ↔ порт`, яка називається **CAM table** (Content Addressable Memory) або MAC address table.

**Процес навчання (MAC learning):**

1. Кадр приходить на порт `Gi0/1` із source MAC `AA:AA:AA:AA:AA:AA`.
2. Комутатор перевіряє: чи є ця MAC-адреса вже в таблиці?
   - Якщо ні — додає запис `AA:AA:AA:AA:AA:AA → Gi0/1`.
   - Якщо є, але на іншому порту — оновлює запис (пристрій міг переміститись, або це MAC-flapping — див. розділ 9).
3. Кожен запис має **timeout** (типово 300 секунд для Cisco за замовчуванням) — якщо з цієї MAC-адреси довго немає трафіку, запис видаляється (aging).

**Процес пересилання (forwarding decision):**

Отримавши кадр, комутатор дивиться на **destination MAC**:

| Ситуація | Дія комутатора |
|---|---|
| Destination MAC є в таблиці, порт відомий і **відрізняється** від порту-джерела | ✅ **Unicast forwarding** — надіслати кадр тільки на цей один порт |
| Destination MAC є в таблиці, і порт **той самий**, що й порт-джерело | ❌ Кадр не пересилається (адресат і відправник на одному сегменті) |
| Destination MAC **невідома** (немає в таблиці) | 🔀 **Flooding** — надіслати кадр на всі порти, крім порту-джерела (unknown unicast flood) |
| Destination MAC — broadcast (`FF:FF:FF:FF:FF:FF`) | 🔀 Надіслати на всі порти в тому ж broadcast-домені (крім порту-джерела) |
| Destination MAC — multicast | Залежить від налаштувань (flood за замовчуванням, або IGMP snooping для оптимізації) |

### Ключова відмінність від хаба

| | Hub (L1) | Switch (L2) |
|---|---|---|
| Рівень OSI | Physical | Data Link |
| Приймає рішення? | Ні, просто повторює сигнал | Так, на основі MAC-адрес |
| Collision domain | **Один спільний** для всіх портів | **Окремий на кожен порт** |
| Broadcast domain | Один спільний | Один спільний (за замовчуванням, без VLAN) |
| Дуплекс | Зазвичай half-duplex | Full-duplex (колізії неможливі) |
| Продуктивність при зростанні кількості пристроїв | Деградує швидко | Масштабується значно краще |

Оскільки кожен порт комутатора — це окремий **collision domain**, а з'єднання, як правило, full-duplex, колізії (collision) на сучасних комутованих мережах практично відсутні. Це принципова перевага над хабом, де всі пристрої "борються" за одне спільне середовище передачі.

[⬆ Повернутись до змісту](#зміст)

---

## 2. Broadcast-домени та Collision-домени

Важливо чітко розділяти два поняття:

- **Collision domain (домен колізій)** — сегмент мережі, де кадри можуть "зіткнутися" при одночасній передачі кількома пристроями (актуально для half-duplex/спільного середовища).
- **Broadcast domain (домен трансляції)** — набір пристроїв, які отримають broadcast-кадр (`FF:FF:FF:FF:FF:FF`), надісланий будь-ким усередині цього домену.

### Що робить комутатор

Комутатор **розбиває collision domain на окремі частини** — кожен порт стає власним collision domain'ом. Але за замовчуванням він **НЕ розбиває broadcast domain** — усі порти одного комутатора (і всіх з'єднаних з ним комутаторів без VLAN-сегментації) належать до **одного спільного broadcast domain**.

```
                    ОДИН BROADCAST DOMAIN
   ┌───────────────────────────────────────────────────┐
   │   Switch A            Switch B          Switch C  │
   │  [P1][P2][P3] ── uplink ── [P1][P2][P3] ── uplink ── [P1][P2][P3]  │
   │   ▲     ▲                                          │
   │   │     └── кожен порт: окремий collision domain   │
   │   └── усі порти + всі комутатори: один broadcast   │
   │       domain (без VLAN-поділу)                     │
   └───────────────────────────────────────────────────┘
```

### Чому необмежений broadcast domain — це проблема

Коли broadcast domain стає надто великим (сотні чи тисячі пристроїв):

1. **Broadcast storm** — лавиноподібне зростання broadcast-трафіку (ARP-запити, DHCP Discover тощо), яке "з'їдає" пропускну здатність і CPU кожного пристрою в домені.
2. **Деградація продуктивності** — кожен хост змушений обробляти *кожен* broadcast-кадр у мережі, навіть якщо він йому не потрібен, витрачаючи CPU-цикли.
3. **Проблеми безпеки** — будь-який пристрій у broadcast domain потенційно бачить широкомовний трафік всієї мережі (ARP, DHCP), що спрощує деякі атаки (ARP spoofing, DHCP starvation).
4. **Зона враження при петлі (loop)** — якщо в мережі є фізична петля без захисту (STP), broadcast storm може паралізувати весь домен миттєво (кадри при L2 не мають TTL — див. розділ 6).

Саме ця проблема — **необхідність обмежити розмір broadcast domain** — і є головною мотивацією для впровадження **VLAN**.

[⬆ Повернутись до змісту](#зміст)

---

## 3. Що таке VLAN і навіщо він потрібен

**VLAN (Virtual Local Area Network)** — це спосіб логічно розділити один фізичний комутатор (або групу з'єднаних комутаторів) на кілька **незалежних broadcast domain'ів**, не змінюючи фізичну структуру кабелів.

Пристрої в різних VLAN не бачать широкомовний трафік один одного і за замовчуванням **не можуть спілкуватися напряму на L2** — навіть якщо фізично підключені до одного й того ж комутатора.

### 🧩 Реальна аналогія

Уявіть один великий поверх офісу відкритого планування (open-space) — фізично це одне велике приміщення, одна електропроводка, одна система вентиляції. Але за допомогою **бейджів доступу** компанія ділить цей самий фізичний простір на віртуальні "кімнати": відділ бухгалтерії, відділ розробки, відділ підтримки. Люди фізично сидять у тому самому приміщенні, але бейдж бухгалтера не відкриє "віртуальні двері" у зону розробників, і оголошення для бухгалтерії не почує розробник, хоча стіни між ними немає — розділення суто логічне, засноване на конфігурації (кому видано який бейдж), а не на фізичній стіні.

VLAN працює так само: одна й та сама фізична проводка та комутатор, але порти "призначені" різним VLAN, і трафік між VLAN логічно ізольований.

### Переваги VLAN

| Перевага | Опис |
|---|---|
| 🔒 **Ізоляція безпеки** | Трафік однієї VLAN недоступний іншій без явного L3-маршрутизування. Наприклад, VLAN для гостьового Wi-Fi ізольована від VLAN внутрішньої мережі компанії |
| 📉 **Контроль broadcast domain** | Broadcast-кадри залишаються в межах однієї VLAN, а не заповнюють всю мережу |
| 🗂️ **Логічне групування незалежно від фізичного розташування** | Пристрої одного відділу можуть бути розкидані по різних поверхах/будівлях, підключені до різних комутаторів, але перебувати в одній VLAN |
| 🔧 **Спрощення переміщень (moves/adds/changes)** | Коли співробітник переїжджає в інший кабінет, достатньо призначити новий порт до потрібної VLAN — не треба перепрокладати кабелі |
| 📊 **Керування якістю трафіку (QoS)** | Можна виділити окрему VLAN, наприклад, для VoIP-трафіку та застосувати до неї особливі політики пріоритизації |

Типові приклади VLAN у корпоративній мережі: VLAN 10 — користувачі, VLAN 20 — сервери, VLAN 30 — VoIP-телефони, VLAN 99 — management (керування самими мережевими пристроями), VLAN 999 — гостьовий Wi-Fi.

[⬆ Повернутись до змісту](#зміст)

---

## 4. VLAN tagging — 802.1Q

Щоб комутатори могли передавати трафік **кількох VLAN через одне фізичне з'єднання** (наприклад, uplink між двома комутаторами), потрібен механізм **маркування (tagging)** кадрів — це і робить стандарт **IEEE 802.1Q**.

### Типи портів

- **Access port (порт доступу)** — належить рівно **одній** VLAN. Кадри, що виходять через access-порт, **не мають тегу** (untagged) — кінцевий пристрій (ПК, принтер, IP-телефон) навіть не підозрює про існування VLAN. Використовується для підключення кінцевих хостів.
- **Trunk port (транкований порт)** — може передавати трафік **кількох VLAN одночасно** через одне фізичне з'єднання. Кожен кадр маркується 802.1Q-тегом із номером VLAN, щоб приймаючий комутатор знав, до якої VLAN цей кадр належить. Використовується між комутаторами, а також між комутатором і маршрутизатором/L3-пристроєм.

```
   [ПК: VLAN 10]        [ПК: VLAN 20]
        │  untagged           │  untagged
        │  (access port)      │  (access port)
   ┌────▼──────────────────────▼────┐
   │          Switch A               │
   │                                  │
   │   trunk port (802.1Q tagged) ────┼──── передає VLAN 10 + VLAN 20
   └───────────────┬──────────────────┘      одним фізичним кабелем
                    │
                    │  тегований трафік: 
                    │  VLAN10-frame, VLAN20-frame, VLAN10-frame...
                    │
   ┌────────────────▼──────────────────┐
   │          Switch B                 │
   │   trunk port (802.1Q tagged)      │
   │                                    │
   │   access port          access port │
   └───┬─────────────────────┬──────────┘
       │ untagged             │ untagged
  [ПК: VLAN 10]         [ПК: VLAN 20]
```

### Структура 802.1Q тегу

802.1Q **вставляє додаткові 4 байти** всередину стандартного Ethernet-кадру, між полем Source MAC та EtherType/Length:

```
Звичайний Ethernet-кадр (без тегу):
┌─────────────┬────────────┬───────────┬─────────┬─────┐
│  Dest MAC   │ Source MAC │ EtherType │ Payload │ FCS │
│   6 bytes   │  6 bytes   │  2 bytes  │         │     │
└─────────────┴────────────┴───────────┴─────────┴─────┘

Ethernet-кадр з 802.1Q тегом (Tagged frame):
┌─────────────┬────────────┬──────────────────────┬───────────┬─────────┬─────┐
│  Dest MAC   │ Source MAC │   802.1Q Tag (4B)     │ EtherType │ Payload │ FCS │
│   6 bytes   │  6 bytes   │ ┌──────┬─────┬───────┐│  2 bytes  │         │     │
│             │            │ │ TPID │ PRI │  VID  ││           │         │     │
│             │            │ │2byte │3bit │1bit│12b│           │         │     │
│             │            │ │0x8100│ CoS │DEI │VLAN ID       │         │     │
│             │            │ └──────┴─────┴───────┘│           │         │     │
└─────────────┴────────────┴──────────────────────┴───────────┴─────────┴─────┘
```

Розшифровка полів тегу:

| Поле | Розмір | Призначення |
|---|---|---|
| **TPID** (Tag Protocol Identifier) | 16 біт | Фіксоване значення `0x8100`, що сигналізує: "далі йде 802.1Q тег", а не звичайний EtherType |
| **PCP** (Priority Code Point, інколи "PRI"/CoS) | 3 біти | Пріоритет кадру для QoS (0–7), використовується для, наприклад, пріоритизації VoIP-трафіку |
| **DEI** (Drop Eligible Indicator) | 1 біт | Позначає кадри, які можна відкинути першими при перевантаженні |
| **VID** (VLAN Identifier) | 12 біт | Номер VLAN. Оскільки поле 12-бітне, теоретично можливо **2^12 = 4096** значень, з яких `0` і `4095` зарезервовані → практично **до 4094 VLAN** (1–4094) |

Через додавання 4 байтів тегу, максимальний розмір Ethernet-кадру збільшується з 1518 до **1522 байтів** (для тегованого кадру).

### Native VLAN

На trunk-порту одна VLAN може бути позначена як **native VLAN** — кадри цієї VLAN передаються **без тегу (untagged)** через транк. Це зроблено для сумісності зі старішим обладнанням, яке не розуміє 802.1Q.

⚠️ **Важливий момент для співбесіди:** якщо native VLAN не збігається на обох кінцях trunk-з'єднання (наприклад, Switch A вважає native VLAN 1, а Switch B — native VLAN 99), виникає **VLAN hopping** або плутанина трафіку — untagged-кадри потраплять не в ту VLAN, яку очікували. Це поширена помилка конфігурації та поширене питання на співбесідах. З міркувань безпеки native VLAN зазвичай **не залишають VLAN 1 за замовчуванням**, а призначають окрему, невикористовувану VLAN.

[⬆ Повернутись до змісту](#зміст)

---

## 5. Inter-VLAN Routing

VLAN за визначенням ізолюють трафік на **другому рівні (L2)**. Це означає, що пристрій у VLAN 10 **не може** напряму обмінюватися кадрами з пристроєм у VLAN 20 — навіть якщо вони підключені до одного й того ж фізичного комутатора. Щоб дозволити спілкування між VLAN, потрібна **маршрутизація на третьому рівні (L3)**.

Кожна VLAN, як правило, відповідає **окремій IP-підмережі** (1:1 mapping): наприклад, VLAN 10 = `10.0.10.0/24`, VLAN 20 = `10.0.20.0/24`. Щоб хост із VLAN 10 надіслав пакет хосту у VLAN 20, пакет має пройти через маршрутизатор (або L3-пристрій), який змінить рішення на рівні IP-маршрутизації.

### Варіант 1: Router-on-a-Stick

Один фізичний **транкований** канал з'єднує комутатор з маршрутизатором. На маршрутизаторі створюються **логічні субінтерфейси** — по одному на кожну VLAN, кожен зі своєю IP-адресою (шлюзом за замовчуванням для цієї VLAN).

```
                    ┌──────────────┐
                    │   Router     │
                    │              │
                    │ Gi0/0.10 ────┼── VLAN 10 gateway: 10.0.10.1
                    │ Gi0/0.20 ────┼── VLAN 20 gateway: 10.0.20.1
                    └──────┬───────┘
                           │ один фізичний trunk-кабель
                           │ (802.1Q, VLAN 10 + VLAN 20)
                    ┌──────▼───────┐
                    │    Switch    │
                    └───┬──────┬───┘
                  access│      │access
                  VLAN10│      │VLAN20
                  [PC A]│      │[PC B]
```

**Плюси:** не потребує L3-функціональності на комутаторі, дешевше в обладнанні.
**Мінуси:** увесь inter-VLAN трафік йде через один фізичний канал → потенційне вузьке місце (bottleneck) при великих обсягах трафіку; додаткова затримка через прохід до окремого маршрутизатора.

### Варіант 2: Layer 3 Switch зі SVI (Switched Virtual Interface)

Сучасніший і найпоширеніший у датацентрах/офісах підхід — використання **комутатора третього рівня (L3 switch/multilayer switch)**, який поєднує функції комутації та маршрутизації в одному пристрої. Для кожної VLAN створюється **SVI** — віртуальний L3-інтерфейс, прив'язаний до VLAN, який виконує роль шлюзу за замовчуванням прямо на самому комутаторі.

```
                    ┌───────────────────────────┐
                    │      L3 Switch             │
                    │                             │
                    │  interface VLAN10           │
                    │    ip address 10.0.10.1/24  │  ← SVI VLAN 10
                    │  interface VLAN20            │
                    │    ip address 10.0.20.1/24  │  ← SVI VLAN 20
                    │                             │
                    │  [P1: access VLAN10]        │
                    │  [P2: access VLAN20]        │
                    └───┬──────────────────┬──────┘
                  access│                  │access
                  VLAN10│                  │VLAN20
                  [PC A]│                  │[PC B]
```

**Плюси:** маршрутизація відбувається "на швидкості лінії" (wire-speed) всередині одного пристрою, без додаткової затримки; немає єдиної точки вузького місця, як у router-on-a-stick.
**Мінуси:** L3-комутатори дорожчі за прості L2-комутатори.

На співбесіді важливо чітко проговорити: **VLAN розділяє трафік на L2, а поєднати VLAN назад може лише пристрій, що виконує маршрутизацію на L3** (класичний роутер, router-on-a-stick або L3-switch зі SVI).

[⬆ Повернутись до змісту](#зміст)

---

## 6. Spanning Tree Protocol (STP) — коротко

### Проблема: петлі на L2

У реальних мережах часто навмисно прокладають **резервні (redundant) фізичні лінки** між комутаторами — для відмовостійкості (якщо один кабель/лінк вийде з ладу, є запасний шлях). Але на відміну від IP-пакетів (L3), у яких є поле **TTL (Time To Live)**, що обмежує кількість "стрибків" і не дає пакету блукати мережею вічно, у звичайному Ethernet-кадрі (L2) **поля TTL немає**.

Якщо в топології є фізична петля (loop) без жодного захисту, і в мережу потрапляє широкомовний (broadcast) кадр, станеться таке:

```
        ┌─────────┐  Link 1   ┌─────────┐
        │Switch A │───────────│Switch B │
        │         │           │         │
        │         │───────────│         │
        └─────────┘  Link 2   └─────────┘
        
   Кадр-broadcast заходить у Switch A → пересилається
   через Link 1 і Link 2 одночасно на Switch B → Switch B
   пересилає його знову назад на Switch A через обидва лінки
   → кадр циркулює нескінченно, множиться → BROADCAST STORM
```

Кадр буде циркулювати мережею **нескінченно**, дублюючись з кожним проходом через кожен лінк, доки не "з'їсть" всю пропускну здатність і не покладе всі комутатори в петлі (CPU перевантажується обробкою мільйонів копій кадру за секунду). Це і є **broadcast storm**, одна з найнебезпечніших аварій у L2-мережах.

### Рішення: Spanning Tree Protocol (STP, IEEE 802.1D)

STP вирішує цю проблему, будуючи логічне дерево без петель (spanning tree) поверх фізичної топології з резервними зв'язками:

1. **Обирається Root Bridge** — один комутатор у мережі, який стає "коренем" логічного дерева (обирається за найнижчим Bridge ID: пріоритет + MAC-адреса).
2. Кожен інший комутатор визначає свій **найкоротший шлях до root bridge** і позначає відповідний порт як **Root Port**.
3. Для кожного сегмента мережі обирається **Designated Port** — порт, що пересилає трафік у цьому сегменті.
4. Всі **інші** порти, що створюють надлишкові (дублюючі) шляхи, переводяться у стан **Blocking** — вони фізично підключені, але **не пересилають** звичайний трафік, тим самим розриваючи петлю логічно, зберігаючи фізичну надлишковість як резерв.

```
        ┌─────────┐  Link 1 (Designated/Root Port — forwarding) ┌─────────┐
        │Switch A │═══════════════════════════════════════════│Switch B │
        │ (Root)  │                                             │         │
        │         │╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌│         │
        └─────────┘  Link 2 (Blocking — резерв, не пересилає)   └─────────┘

   Якщо Link 1 вийде з ладу — STP перерахує топологію і Link 2
   автоматично перейде у стан Forwarding, відновивши зв'язність.
```

Якщо активний лінк (Link 1) вийде з ладу, STP перераховує топологію і **автоматично активує** резервний лінк (Link 2), переводячи його з Blocking у Forwarding — так забезпечується і захист від петель, і відмовостійкість.

### Стани портів STP (класичний 802.1D)

| Стан | Опис |
|---|---|
| **Blocking** | Порт не пересилає трафік користувача, лише слухає STP BPDU |
| **Listening** | Перехідний стан, порт готується стати forwarding, обробляє BPDU, ще не пересилає дані |
| **Learning** | Починає вивчати MAC-адреси, але ще не пересилає користувацький трафік |
| **Forwarding** | Порт повністю активний — пересилає трафік і вивчає MAC-адреси |
| **Disabled** | Порт адміністративно вимкнений |

### RSTP — швидша сучасна версія

Класичний STP (802.1D) може перебудовувати топологію **30–50 секунд** — неприйнятно повільно для сучасних мереж. **RSTP (Rapid Spanning Tree Protocol, 802.1w)** — це вдосконалена версія, яка досягає збіжності (convergence) за **секунди, а не десятки секунд**, завдяки спрощеній моделі станів (тільки Discarding/Learning/Forwarding) і механізмам швидкого узгодження між сусідніми комутаторами. У сучасних мережах RSTP (або його розширення MSTP — Multiple Spanning Tree для декількох VLAN) є практичним стандартом де-факто.

[⬆ Повернутись до змісту](#зміст)

---

## 7. Практичні приклади на C#

⚠️ Важливо розуміти: реальне налаштування VLAN, trunk-портів чи STP на фізичному обладнанні виконується через CLI/SSH/NETCONF на самому комутаторі (Cisco IOS, Juniper Junos тощо), а **не** безпосередньо з C#-коду. Але C#-розробник у компанії на кшталт SolarWinds постійно працює з **логікою поверх** цих концепцій: симуляціями для навчання/тестування, моделями даних, отриманими через SNMP, звітами про стан мережі. Нижче — три приклади саме такого рівня.

### Приклад 1: Симуляція MAC Address Table комутатора

Проста програмна модель того, як комутатор навчається MAC-адрес і приймає рішення про пересилання/флудинг/дроп.

```csharp
using System;
using System.Collections.Generic;

namespace SwitchSimulation
{
    /// <summary>
    /// Спрощена модель кадру Ethernet для навчальної симуляції.
    /// </summary>
    public record EthernetFrame(string SourceMac, string DestinationMac, string Payload);

    /// <summary>
    /// Результат рішення про пересилання кадру.
    /// </summary>
    public enum ForwardingDecision
    {
        ForwardToPort,   // unicast forward — знайдено конкретний порт
        Flood,           // порт призначення невідомий, або broadcast
        Drop             // джерело і призначення на тому самому порту
    }

    /// <summary>
    /// Спрощена симуляція MAC address table (CAM table) комутатора.
    /// Демонструє механізм навчання (learning) і пересилання (forwarding).
    /// </summary>
    public class MacAddressTable
    {
        private const string BroadcastMac = "FF:FF:FF:FF:FF:FF";

        // MAC -> порт, куди підключений пристрій з цією MAC-адресою
        private readonly Dictionary<string, int> _table = new();

        /// <summary>
        /// Викликається кожного разу, коли кадр приходить на порт.
        /// Комутатор "запам'ятовує" звідки прийшов source MAC.
        /// </summary>
        public void Learn(string sourceMac, int incomingPort)
        {
            if (!_table.TryGetValue(sourceMac, out var existingPort))
            {
                _table[sourceMac] = incomingPort;
                Console.WriteLine($"[LEARN] Нова MAC-адреса {sourceMac} -> порт {incomingPort}");
            }
            else if (existingPort != incomingPort)
            {
                // MAC "переїхала" на інший порт — оновлюємо запис.
                // Часті такі оновлення можуть свідчити про MAC flapping (див. розділ 9).
                Console.WriteLine($"[MOVE]  MAC-адреса {sourceMac} перемістилась: порт {existingPort} -> {incomingPort}");
                _table[sourceMac] = incomingPort;
            }
        }

        /// <summary>
        /// Приймає рішення про пересилання кадру на основі destination MAC.
        /// </summary>
        public ForwardingDecision Decide(EthernetFrame frame, int incomingPort, out int? outPort)
        {
            outPort = null;

            if (frame.DestinationMac == BroadcastMac)
            {
                Console.WriteLine($"[BCAST] Кадр {frame.SourceMac} -> BROADCAST: флудимо на всі порти, крім {incomingPort}");
                return ForwardingDecision.Flood;
            }

            if (_table.TryGetValue(frame.DestinationMac, out var knownPort))
            {
                if (knownPort == incomingPort)
                {
                    Console.WriteLine($"[DROP]  Джерело і призначення на одному порту ({incomingPort}) — кадр не пересилається");
                    return ForwardingDecision.Drop;
                }

                outPort = knownPort;
                Console.WriteLine($"[FWD]   {frame.SourceMac} -> {frame.DestinationMac}: unicast на порт {knownPort}");
                return ForwardingDecision.ForwardToPort;
            }

            Console.WriteLine($"[FLOOD] Destination MAC {frame.DestinationMac} невідома — флудимо на всі порти, крім {incomingPort}");
            return ForwardingDecision.Flood;
        }

        public void PrintTable()
        {
            Console.WriteLine("\n--- MAC Address Table (CAM Table) ---");
            Console.WriteLine($"{"MAC Address",-20}{"Port",-6}");
            foreach (var (mac, port) in _table)
            {
                Console.WriteLine($"{mac,-20}{port,-6}");
            }
            Console.WriteLine("--------------------------------------\n");
        }
    }

    public static class Program
    {
        public static void Main()
        {
            var switchTable = new MacAddressTable();

            // Крок 1: PC-A (порт 1) надсилає ARP-запит (broadcast) — комутатор вчиться MAC PC-A
            var frame1 = new EthernetFrame("AA:AA:AA:AA:AA:01", "FF:FF:FF:FF:FF:FF", "ARP: Хто має 10.0.10.5?");
            switchTable.Learn(frame1.SourceMac, incomingPort: 1);
            switchTable.Decide(frame1, incomingPort: 1, out _);

            // Крок 2: PC-B (порт 2) відповідає напряму PC-A — MAC PC-A вже відома!
            var frame2 = new EthernetFrame("BB:BB:BB:BB:BB:02", "AA:AA:AA:AA:AA:01", "ARP Reply: Я тут!");
            switchTable.Learn(frame2.SourceMac, incomingPort: 2);
            switchTable.Decide(frame2, incomingPort: 2, out var outPort);
            Console.WriteLine(outPort.HasValue
                ? $"=> Кадр пересилається тільки на порт {outPort.Value}, інші порти не турбуємо\n"
                : "=> Флудинг\n");

            switchTable.PrintTable();
        }
    }
}
```

Очікуваний консольний вивід ілюструє послідовність: перший broadcast-кадр змушує зробити flood (адресат ще невідомий), але вже другий кадр (відповідь) пересилається unicast'ом напряму на потрібний порт, оскільки MAC-адреса відправника першого кадру вже "вивчена".

### Приклад 2: Модель VLAN-членства портів та перевірка L2/L3 зв'язності

Проста модель, що показує різницю: чи можуть два порти спілкуватись напряму на L2 (та сама VLAN), чи їм потрібна маршрутизація на L3 (різні VLAN).

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace VlanSimulation
{
    public enum PortMode { Access, Trunk }

    /// <summary>
    /// Модель фізичного порту комутатора з точки зору VLAN-конфігурації.
    /// </summary>
    public class SwitchPort
    {
        public string Name { get; }
        public PortMode Mode { get; }

        /// <summary>Для access-порту: єдина VLAN. Для trunk: не використовується напряму.</summary>
        public int? AccessVlanId { get; }

        /// <summary>Для trunk-порту: список дозволених VLAN (allowed vlan list).</summary>
        public HashSet<int> AllowedVlans { get; }

        public SwitchPort(string name, int accessVlanId)
        {
            Name = name;
            Mode = PortMode.Access;
            AccessVlanId = accessVlanId;
            AllowedVlans = new HashSet<int> { accessVlanId };
        }

        public SwitchPort(string name, IEnumerable<int> trunkAllowedVlans)
        {
            Name = name;
            Mode = PortMode.Trunk;
            AllowedVlans = new HashSet<int>(trunkAllowedVlans);
        }
    }

    public enum ConnectivityResult
    {
        DirectLayer2,       // та сама VLAN - спілкування напряму, без маршрутизації
        RequiresLayer3,     // різні VLAN - потрібен router/L3 switch (SVI)
        Blocked             // VLAN не дозволена на одному з портів (наприклад, не в allowed list трансу)
    }

    public static class VlanConnectivityChecker
    {
        /// <summary>
        /// Перевіряє, чи можуть два порти спілкуватись напряму на L2,
        /// потребують маршрутизації на L3, чи взагалі заблоковані.
        /// </summary>
        public static ConnectivityResult Check(SwitchPort portA, SwitchPort portB, int trafficVlanId)
        {
            bool aCarries = portA.AllowedVlans.Contains(trafficVlanId);
            bool bCarries = portB.AllowedVlans.Contains(trafficVlanId);

            if (!aCarries || !bCarries)
            {
                return ConnectivityResult.Blocked;
            }

            // Якщо обидва - access-порти в одній VLAN -> пряме спілкування на L2
            if (portA.Mode == PortMode.Access && portB.Mode == PortMode.Access)
            {
                return portA.AccessVlanId == portB.AccessVlanId
                    ? ConnectivityResult.DirectLayer2
                    : ConnectivityResult.RequiresLayer3;
            }

            // Якщо хоча б один - trunk, і обидва пропускають трафік цієї VLAN,
            // вважаємо, що кадр з тегом trafficVlanId пройде на L2 в межах цієї VLAN.
            return ConnectivityResult.DirectLayer2;
        }
    }

    public static class Program
    {
        public static void Main()
        {
            // Access-порти кінцевих пристроїв
            var pcA = new SwitchPort("Gi0/1 (PC-A)", accessVlanId: 10);
            var pcB = new SwitchPort("Gi0/2 (PC-B)", accessVlanId: 10);   // та ж VLAN, що і PC-A
            var serverC = new SwitchPort("Gi0/3 (Server-C)", accessVlanId: 20); // інша VLAN

            // Trunk-порт до іншого комутатора, дозволяє VLAN 10, 20, 30
            var trunkUplink = new SwitchPort("Gi0/24 (trunk to Switch-B)", new[] { 10, 20, 30 });

            var scenarios = new (string Description, SwitchPort A, SwitchPort B, int Vlan)[]
            {
                ("PC-A <-> PC-B (обидва VLAN 10)", pcA, pcB, 10),
                ("PC-A <-> Server-C (VLAN 10 vs VLAN 20)", pcA, serverC, 10),
                ("PC-A <-> Trunk uplink (VLAN 10 дозволена на trunk)", pcA, trunkUplink, 10),
                ("Server-C <-> Trunk uplink, але трафік VLAN 99 (не в allowed list)", serverC, trunkUplink, 99),
            };

            foreach (var s in scenarios)
            {
                var result = VlanConnectivityChecker.Check(s.A, s.B, s.Vlan);
                var icon = result switch
                {
                    ConnectivityResult.DirectLayer2 => "✅ L2 напряму",
                    ConnectivityResult.RequiresLayer3 => "🔀 Потрібна L3-маршрутизація (inter-VLAN routing)",
                    ConnectivityResult.Blocked => "❌ Заблоковано (VLAN не дозволена)",
                    _ => "?"
                };

                Console.WriteLine($"{s.Description,-65} => {icon}");
            }
        }
    }
}
```

Очікуваний вивід показує чотири класичні ситуації: пряме L2-спілкування в межах однієї VLAN, необхідність L3-маршрутизації між різними VLAN, успішне проходження трафіку через дозволений trunk, і блокування через відсутність VLAN у allowed list транка.

### Приклад 3: Модель звіту про стан портів комутатора (як це робить інструмент моніторингу)

Інструмент на кшталт **SolarWinds NPM** регулярно опитує комутатори через **SNMP** (наприклад, MIB `IF-MIB`, `BRIDGE-MIB`, `Q-BRIDGE-MIB`) і будує зручні звіти про стан кожного порту: до якої VLAN він належить, яка MAC-адреса підключена, чи активний лінк, яка швидкість/дуплекс. Нижче — спрощена модель того, як така "сира" інформація з SNMP перетворюється на структуровану модель і звіт (без реальних SNMP-викликів, лише ілюстрація логіки обробки даних).

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

namespace SwitchMonitoring
{
    public enum LinkStatus { Up, Down, AdminDown }

    /// <summary>
    /// Модель стану одного порту комутатора, зібрана з SNMP-даних
    /// (типово з поєднання IF-MIB, Q-BRIDGE-MIB, dot1qTpFdbTable тощо).
    /// Саме такі моделі лежать в основі "Node Details" сторінок NPM/NCM.
    /// </summary>
    public record SwitchPortStatus(
        string PortName,
        int VlanId,
        string? ConnectedMac,
        LinkStatus Status,
        string Speed,
        string Duplex
    );

    /// <summary>
    /// Симулює обробку "сирих" даних, отриманих від SNMP-опитування комутатора,
    /// перетворюючи їх у зручну для відображення модель.
    /// </summary>
    public static class SwitchPortReportBuilder
    {
        public static List<SwitchPortStatus> BuildFromRawSnmpData()
        {
            // У реальному інструменті ці дані прийшли б з SNMP GET/WALK
            // (наприклад, через сторонню бібліотеку типу Lextm.SharpSnmpLib),
            // а не були б захардкоджені, як тут - для навчальної ілюстрації.
            return new List<SwitchPortStatus>
            {
                new("Gi0/1", VlanId: 10, ConnectedMac: "AA:AA:AA:AA:AA:01", Status: LinkStatus.Up, Speed: "1 Gbps", Duplex: "Full"),
                new("Gi0/2", VlanId: 10, ConnectedMac: "BB:BB:BB:BB:BB:02", Status: LinkStatus.Up, Speed: "1 Gbps", Duplex: "Full"),
                new("Gi0/3", VlanId: 20, ConnectedMac: null,                Status: LinkStatus.Down, Speed: "-", Duplex: "-"),
                new("Gi0/4", VlanId: 99, ConnectedMac: "CC:CC:CC:CC:CC:04", Status: LinkStatus.Up, Speed: "100 Mbps", Duplex: "Half"),
                new("Gi0/24", VlanId: 1,  ConnectedMac: null,                Status: LinkStatus.AdminDown, Speed: "-", Duplex: "-"),
            };
        }

        /// <summary>
        /// Друкує форматовану таблицю стану портів - спрощений аналог
        /// того, що бачить адміністратор на сторінці стану комутатора в NPM.
        /// </summary>
        public static void PrintReport(IEnumerable<SwitchPortStatus> ports)
        {
            Console.WriteLine($"{"Port",-8}{"VLAN",-6}{"MAC Address",-20}{"Status",-12}{"Speed",-12}{"Duplex",-6}");
            Console.WriteLine(new string('-', 64));

            foreach (var p in ports)
            {
                var statusLabel = p.Status switch
                {
                    LinkStatus.Up => "🟢 Up",
                    LinkStatus.Down => "🔴 Down",
                    LinkStatus.AdminDown => "⚪ Adm-Down",
                    _ => p.Status.ToString()
                };

                // Half-дуплекс на сучасному порту - типовий "жовтий прапорець"
                // для моніторингу: часто свідчить про проблему з auto-negotiation.
                var duplexFlag = p.Duplex == "Half" ? "⚠️ Half" : p.Duplex;

                Console.WriteLine(
                    $"{p.PortName,-8}{p.VlanId,-6}{p.ConnectedMac ?? "-",-20}{statusLabel,-12}{p.Speed,-12}{duplexFlag,-6}");
            }
        }

        /// <summary>
        /// Приклад практичної перевірки: знайти порти, де auto-negotiation
        /// дав half-duplex - типова причина продуктивнісних скарг користувачів.
        /// </summary>
        public static IEnumerable<SwitchPortStatus> FindDuplexMismatchCandidates(IEnumerable<SwitchPortStatus> ports)
            => ports.Where(p => p.Status == LinkStatus.Up && p.Duplex == "Half");
    }

    public static class Program
    {
        public static void Main()
        {
            var ports = SwitchPortReportBuilder.BuildFromRawSnmpData();

            Console.WriteLine("=== Звіт про стан портів комутатора (SNMP-based, як у SolarWinds NPM) ===\n");
            SwitchPortReportBuilder.PrintReport(ports);

            var duplexIssues = SwitchPortReportBuilder.FindDuplexMismatchCandidates(ports).ToList();
            if (duplexIssues.Any())
            {
                Console.WriteLine("\n⚠️ Виявлено потенційні duplex mismatch на портах:");
                foreach (var p in duplexIssues)
                {
                    Console.WriteLine($"   - {p.PortName} (VLAN {p.VlanId}, MAC {p.ConnectedMac})");
                }
            }
        }
    }
}
```

Такий підхід — читання "сирих" даних пристрою і перетворення їх у зрозумілу для адміністратора модель/звіт — **саме те**, чим займається значна частина backend-коду в продуктах на кшталт NPM (Network Performance Monitor) чи NCM (Network Configuration Manager).

[⬆ Повернутись до змісту](#зміст)

---

## 8. Реальний сценарій: чому VLAN/switching важливі для мережевого моніторингу

Компанії на кшталт **SolarWinds** будують продукти (**Orion Platform, NPM, NCM**), основне завдання яких — дати мережевому адміністратору **видимість (visibility)** того, що відбувається в мережі, і допомогти швидко діагностувати проблеми. Розуміння комутації та VLAN — це не абстрактна теорія, а щоденна практична основа роботи таких інструментів.

### Що саме "бачить" інструмент моніторингу

- **Топологію мережі** — NPM опитує комутатори через SNMP (CDP/LLDP для сусідства між пристроями, `BRIDGE-MIB`/`Q-BRIDGE-MIB` для MAC address table і VLAN-призначень) і будує наочну карту: який комутатор до якого підключений, скільки VLAN сконфігуровано, які порти активні.
- **Відповідність "порт ↔ MAC-адреса ↔ VLAN"** — так званий **"Port-to-MAC mapping"** — дозволяє відповісти на запитання "хто фізично підключений до порту Gi0/14 комутатора у 3-й кімнаті на 2-му поверсі?" без фізичного походу до серверної.
- **Конфігурацію VLAN на кожному порту** (NCM зберігає бекапи конфігурацій комутаторів і може відслідковувати саме таку інформацію — access vlan, trunk allowed vlan list).

### Практичний кейс: "Користувач втратив зв'язок після переміщення в інший кабінет"

**Симптом:** Співробітника перевели в новий кабінет. IT підключив його ноутбук у розетку на стіні (яка веде на порт `Gi0/12` найближчого комутатора). Ноутбук отримав IP-адресу по DHCP, але не може достукатися до внутрішніх ресурсів компанії (файлового сервера, внутрішнього порталу), хоча інтернет ніби працює.

**Діагностика через призму VLAN:**

1. Спочатку перевіряють базову L1/L2-зв'язність: лінк активний? (Link status Up — так, судячи з отриманого DHCP-адреси).
2. Перевіряють **яку саме VLAN** призначено на порт `Gi0/12` — виявляється, це VLAN **50 (гостьовий Wi-Fi/guest network)**, а не VLAN **10 (корпоративна мережа)**, як мало б бути.
3. Причина: старий орендар цього кабінету (наприклад, переговорна кімната для гостей) використовував порт для гостьового доступу, і порт залишився налаштований на VLAN 50 — гостьова мережа зазвичай має вихід в інтернет, але **навмисно ізольована** від внутрішніх ресурсів (файлових серверів, порталів) через фаєрвол/ACL між VLAN.
4. **Рішення:** змінити конфігурацію порту `Gi0/12` — призначити access VLAN 10 замість VLAN 50.

**Роль інструменту моніторингу в цьому кейсі:** Замість того, щоб інженер вручну підключався по SSH до кожного підозрюваного комутатора і вводив `show vlan brief` та `show interface status`, інструмент типу NPM **одразу показує** на дашборді порт, його VLAN-призначення і статус лінка, різко скорочуючи час діагностики (MTTR — Mean Time To Resolution). Саме тому питання на кшталт "як VLAN впливає на зв'язність користувача" — типова тема співбесіди у компанії з профілю мережевого моніторингу.

### VLAN Configuration Drift — типовий сценарій підтримки

Однією з ключових цінностей продуктів як **NCM (Network Configuration Manager)** є відстеження **дрейфу конфігурації (configuration drift)**:

- Мережевий інженер (або сам продукт) має "еталонну" (baseline) конфігурацію для кожного комутатора — яка VLAN на якому порту, які trunk-порти дозволяють які VLAN тощо.
- Часто хтось (інженер під тиском на місці інциденту, стажер, підрядник) **вручну** заходить на комутатор через консоль/SSH і змінює VLAN на порту "тимчасово, щоб вирішити проблему прямо зараз" — і **забуває задокументувати** або відкотити зміну, або внести її назад через систему управління конфігураціями.
- Це створює **невідповідність** між тим, що *має* бути (за документацією/еталоном), і тим, що *реально* налаштовано на пристрої.
- NCM автоматично виявляє такі розбіжності (порівнюючи поточну конфігурацію з baseline або з попереднім знятим знімком), сповіщає адміністратора про **несанкціоновану зміну** і, за потреби, дозволяє автоматично "відкотити" конфігурацію.

Це реальна й дуже поширена проблема в експлуатації мереж: **невідповідність VLAN-конфігурації** — одна з найчастіших причин "мережа працювала вчора, а сьогодні щось зламалось, хоча ніхто нічого не міняв" (хоча насправді хтось таки міняв, просто не через "правильний" процес).

[⬆ Повернутись до змісту](#зміст)

---

## 9. Поширені проблеми та діагностика

| Проблема | Типовий симптом | Діагностика / рішення |
|---|---|---|
| **Пристрій у неправильному VLAN** | Пристрій отримує IP, але не бачить потрібних ресурсів (файлові сервери, портал), або навпаки — бачить те, чого не повинен | Перевірити конфігурацію порту: `show interface Gi0/12 switchport` → перевірити `access vlan X`; призначити правильну VLAN |
| **Broadcast storm** | Мережа "лягає" повністю або частково, надзвичайно висока утилізація CPU на комутаторах, індикатори лінків миготять шалено | Перевірити наявність петель у топології; перевірити стан STP-портів (`show spanning-tree`) — чи не вимкнений випадково якийсь blocking-порт (наприклад, хтось відключив STP на порту або підключив некерований hub, що створив петлю в обхід STP) |
| **Trunk не передає потрібний VLAN** | Пристрої в одній VLAN, але на різних комутаторах, не бачать один одного, хоча в межах свого комутатора все працює | Перевірити `allowed vlan` list на trunk-порту (`show interface trunk`) — можливо, потрібна VLAN просто не додана в дозволений список на одному з кінців |
| **MAC address flapping** | Та сама MAC-адреса в CAM table постійно "стрибає" між різними портами за короткий проміжок часу | Найчастіші причини: (1) фізична петля в мережі без належного STP-захисту, (2) дублювання MAC-адрес (два пристрої з однаковою MAC, часто через невдале клонування віртуальних машин або несправну NIC), (3) неправильно налаштований etherchannel/port-channel (лінки в агрегованому каналі не згруповані належним чином) |
| **Native VLAN mismatch на trunk** | Дивна поведінка untagged-трафіку, потенційна плутанина VLAN, попередження в логах STP про native VLAN mismatch | Переконатись, що native VLAN однакова на обох кінцях trunk-з'єднання |
| **Duplex mismatch** | Повільна, нестабільна робота конкретного порту, велика кількість помилок CRC/collision у лічильниках інтерфейсу | Перевірити налаштування auto-negotiation з обох боків з'єднання; примусово виставити однаковий дуплекс/швидкість на обох кінцях, якщо auto-negotiation не спрацьовує коректно |
| **STP topology change / port flapping** | Періодичні короткочасні "провали" зв'язності по всій мережі | Перевірити логи STP на предмет частих Topology Change Notifications (TCN) — часто свідчить про нестабільний фізичний лінк, що постійно то з'являється, то зникає |

### Корисні команди (Cisco IOS CLI) — для впізнаваності на співбесіді

| Команда | Призначення |
|---|---|
| `show mac address-table` | Показати поточну CAM-таблицю: MAC-адреса ↔ VLAN ↔ порт |
| `show vlan brief` | Показати список всіх VLAN та які порти до якої VLAN призначені |
| `show interface trunk` | Показати які порти в режимі trunk, і які VLAN на них дозволені (allowed vlan) |
| `show spanning-tree` | Показати стан STP: хто root bridge, стан кожного порту (forwarding/blocking) |
| `show interface status` | Швидкий огляд стану всіх портів: up/down, VLAN, дуплекс, швидкість |
| `show interfaces <port> switchport` | Детальна L2-конфігурація конкретного порту: mode (access/trunk), access vlan, native vlan |

### SNMP-моніторинг цих же даних

Інструменти моніторингу отримують еквівалентну інформацію не через CLI, а програмно через SNMP-опитування відповідних MIB:

- **`IF-MIB`** — базовий статус інтерфейсів (up/down, швидкість, лічильники трафіку/помилок).
- **`BRIDGE-MIB`** (`dot1dTpFdbTable`) — таблиця MAC-адрес (forwarding database).
- **`Q-BRIDGE-MIB`** (`dot1qVlanStaticTable`, `dot1qPvid`) — VLAN-конфігурація портів, включаючи VLAN ID кожного порту та список VLAN на trunk-портах.
- **CDP/LLDP MIB** — інформація про сусідні пристрої для побудови топологічної карти.

Саме опитування цих MIB регулярно (кожні кілька хвилин) і лежить в основі того, як NPM будує актуальну картину мережі та своєчасно сповіщає про зміни (наприклад, лінк впав, з'явився новий MAC на порту, змінилась VLAN-конфігурація).

[⬆ Повернутись до змісту](#зміст)

---

## 10. Питання та відповіді для співбесіди

**1. Чим комутатор відрізняється від хаба?**
Хаб працює на фізичному рівні (L1) і просто ретранслює електричний сигнал на всі порти без жодного аналізу — усі порти утворюють один спільний collision domain. Комутатор працює на канальному рівні (L2), аналізує MAC-адреси в заголовку кадру і пересилає його цілеспрямовано лише на потрібний порт, а кожен порт комутатора утворює окремий collision domain.

**2. Що таке VLAN і навіщо він потрібен?**
VLAN (Virtual LAN) — це спосіб логічно розділити один фізичний комутатор (чи групу комутаторів) на кілька незалежних broadcast domain'ів. Потрібен для ізоляції трафіку з міркувань безпеки, обмеження розміру broadcast domain, логічного групування пристроїв незалежно від їх фізичного розташування та спрощення адміністрування мережі.

**3. Поясни різницю між access і trunk портом.**
Access-порт належить рівно одній VLAN і передає/приймає кадри без тегу (untagged) — використовується для підключення кінцевих пристроїв. Trunk-порт може передавати трафік кількох VLAN одночасно через одне фізичне з'єднання, маркуючи кожен кадр 802.1Q-тегом з номером VLAN — використовується між комутаторами або між комутатором і маршрутизатором.

**4. Що таке 802.1Q tagging?**
Це стандарт IEEE, що визначає, як вставити в Ethernet-кадр додаткові 4 байти між source MAC і EtherType: 2-байтовий TPID (`0x8100`, ознака наявності тегу), 3-бітний пріоритет (PCP/CoS для QoS), 1-бітний DEI, і 12-бітний VLAN ID (від 0 до 4095, з яких практично використовується 1–4094). Це дозволяє одному фізичному каналу передавати трафік декількох VLAN одночасно.

**5. Що таке native VLAN?**
На trunk-порту одна VLAN може бути позначена як native — трафік цієї VLAN передається без 802.1Q тегу (untagged), для сумісності зі старим обладнанням. Якщо native VLAN не збігається на двох кінцях trunk-з'єднання, виникає плутанина untagged-трафіку (VLAN hopping/mismatch) — типова помилка конфігурації.

**6. Навіщо потрібен inter-VLAN routing і як він реалізується?**
VLAN ізолюють трафік на рівні L2 за замовчуванням, тому для спілкування між різними VLAN потрібна маршрутизація на L3. Реалізується або через "router-on-a-stick" (один trunk-канал до маршрутизатора з окремим логічним субінтерфейсом на кожну VLAN), або через L3-комутатор зі SVI (Switched Virtual Interface) — віртуальним L3-інтерфейсом на самому комутаторі для кожної VLAN.

**7. Що таке STP і яку проблему він вирішує?**
Spanning Tree Protocol вирішує проблему петель (loops) у мережах з резервними фізичними зв'язками між комутаторами. Оскільки Ethernet-кадри не мають TTL, кадр у петлі циркулював би нескінченно, викликаючи broadcast storm. STP будує логічне дерево без петель, обираючи root bridge, визначаючи root/designated порти (forwarding), а надлишкові порти переводить у стан blocking, зберігаючи їх як резерв на випадок відмови активного лінку.

**8. Що таке broadcast storm і чому це небезпечно?**
Broadcast storm — це лавиноподібне накопичення широкомовного (broadcast) трафіку в мережі, зазвичай через фізичну петлю без захисту STP. Оскільки L2-кадри не мають TTL, вони циркулюють і множаться нескінченно, споживаючи всю пропускну здатність і CPU-ресурси комутаторів та кінцевих пристроїв, фактично паралізуючи мережу.

**9. Скільки VLAN можна теоретично створити (враховуючи 12-бітний VLAN ID)?**
Поле VLAN ID в 802.1Q-тезі має 12 біт, що дає 2^12 = 4096 можливих значень (0–4095). VLAN ID 0 і 4095 зарезервовані технічними стандартами, тому практично доступно **4094 VLAN** (1–4094), хоча VLAN 1 зазвичай також не використовується для користувацького трафіку з міркувань безпеки (це VLAN за замовчуванням на більшості обладнання).

**10. Що таке MAC address flapping і про що він може свідчити?**
Це ситуація, коли та сама MAC-адреса в CAM-таблиці комутатора постійно змінює порт-прив'язку за короткий проміжок часу. Найчастіші причини: фізична петля в мережі без належного STP-захисту, два пристрої з дублюючою MAC-адресою (частий випадок при клонуванні віртуальних машин), або некоректно налаштований EtherChannel/port-channel.

**11. Чим відрізняється broadcast domain від collision domain?**
Collision domain — це сегмент мережі, де можливе одночасне зіткнення (колізія) кадрів кількох пристроїв (актуально для half-duplex); комутатор ділить collision domain окремо на кожен порт. Broadcast domain — це набір пристроїв, які отримають будь-який broadcast-кадр; комутатор за замовчуванням **не** ділить broadcast domain — для цього потрібні VLAN.

**12. Чому у комутованих full-duplex мережах колізії практично неможливі?**
Тому що кожен порт комутатора — окремий collision domain, а full-duplex з'єднання дозволяє одночасно передавати і приймати дані по окремих парах провідників/каналах, тож фізичного "зіткнення" сигналів двох пристроїв просто не відбувається, на відміну від half-duplex/спільного середовища передачі (як у хабі чи старому коаксіальному Ethernet).

**13. Що станеться, якщо два порти комутатора налаштовані на різні VLAN, а користувач фізично переносить кабель з одного порту на інший?**
Пристрій опиниться в іншій VLAN, а отже — в іншій IP-підмережі та broadcast domain. Якщо в новій VLAN немає DHCP-сервера, доступного через relay, або пристрій має статичну IP-адресу зі старої підмережі — зв'язність зникне, оскільки шлюз за замовчуванням буде недосяжний. Це саме той сценарій, який часто діагностується інструментами моніторингу (розділ 8).

**14. Як монітор мережі (наприклад, NPM) визначає, до якого порту комутатора підключений конкретний пристрій?**
Через SNMP-опитування CAM-таблиці комутатора (`BRIDGE-MIB`, `dot1dTpFdbTable`) — зіставляючи MAC-адресу пристрою з номером порту, а також через `Q-BRIDGE-MIB` для отримання інформації про VLAN цього порту. Це і є "Port-to-MAC mapping" — одна з ключових функцій топологічного модуля NPM.

[⬆ Повернутись до змісту](#зміст)

---

## 11. Підсумок

### Шпаргалка: Hub vs Switch vs Router

| | Hub | Switch | Router |
|---|---|---|---|
| Рівень OSI | L1 (Physical) | L2 (Data Link) | L3 (Network) |
| Рішення на основі | — (просто повторює сигнал) | MAC-адреси | IP-адреси |
| Collision domain | Один спільний | Окремий на порт | Окремий на порт |
| Broadcast domain | Один спільний | Один спільний (без VLAN) | Розділяє broadcast domain (кожен інтерфейс — окремий) |
| Об'єднує VLAN? | Не має поняття VLAN | Ізолює VLAN одна від одної | ✅ Об'єднує (маршрутизує) VLAN між собою |

### Шпаргалка: Access vs Trunk порт

| | Access Port | Trunk Port |
|---|---|---|
| Кількість VLAN | Одна | Кілька (за списком allowed vlan) |
| Тегування кадрів | Untagged | Tagged (802.1Q), крім native VLAN |
| Типове застосування | Підключення кінцевого пристрою (ПК, принтер, IP-телефон) | З'єднання між комутаторами, або комутатор ↔ маршрутизатор |

### Шпаргалка: VLAN ID

- Розмір поля: **12 біт** → 2^12 = 4096 значень
- Зарезервовані: **0** і **4095**
- Практично доступний діапазон: **1–4094**
- VLAN **1** — зазвичай default VLAN, не рекомендується для користувацького трафіку
- **802.1Q tag** додає 4 байти до кадру (TPID 2B + PCP/DEI/VID 2B)

### Шпаргалка: Стани портів STP

| Стан (802.1D) | Пересилає трафік? | Вивчає MAC? |
|---|---|---|
| Blocking | ❌ Ні | ❌ Ні |
| Listening | ❌ Ні | ❌ Ні |
| Learning | ❌ Ні | ✅ Так |
| Forwarding | ✅ Так | ✅ Так |
| Disabled | ❌ Ні (порт вимкнено) | ❌ Ні |

RSTP (802.1w) — та сама ідея, але зі спрощеною моделлю станів (Discarding/Learning/Forwarding) і збіжністю за секунди замість десятків секунд.

### Головна думка для співбесіди

Комутація (L2) вирішує проблему ефективної передачі кадрів у межах локальної мережі через навчання MAC-адрес. VLAN вирішує проблему **масштабованості та безпеки** цієї ж локальної мережі, логічно розділяючи один фізичний комутатор на кілька ізольованих широкомовних доменів. STP вирішує проблему **надійності** — дозволяє мати фізичну надлишковість зв'язків без ризику катастрофічних петель. А inter-VLAN routing (L3) повертає можливість **контрольованого** обміну трафіком між цими ізольованими сегментами. Для будь-якого інструменту мережевого моніторингу (NPM, NCM) розуміння всіх цих механізмів — це основа того, як він будує топологію, діагностує проблеми зв'язності та відстежує зміни конфігурації мережі.

---

*Документ підготовлено для підготовки до технічної співбесіди. Матеріал орієнтовано на компанії у сфері мережевого моніторингу та управління (Network Management).*
