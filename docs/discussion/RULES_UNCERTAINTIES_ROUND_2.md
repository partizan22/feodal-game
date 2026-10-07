# Невизначеності правил — повторний аудит

Цей файл тимчасовий і існує тільки для review. Він не є джерелом чинних правил.

Усі пункти нижче — місця, де `01_GAME_RULES_V1.md` на поточний момент допускає більше одного змістовного трактування. Для кожного наведено рекомендоване рішення.

---

## 1. Новий Knight після отримання імені: початковий стан

### Невизначеність

Визначено, що ready unnamed Knight фактично створюється лише після того, як Player дає йому ім'я, і що він займає Palace slot. Але не визначено:

- де саме він з'являється;
- чи має він одразу власну Army;
- чи може Unit/Knight існувати без Army.

Це потрібно для однозначного переходу від Palace queue до військової структури.

### Пропоноване рішення

Новий Knight після отримання імені:

- створюється у своєму home Castle;
- має `location = Castle`;
- має 0 Soldier;
- одразу створює окрему Army з одного Unit, де сам є Commander-in-Chief;
- Army має Camp-presence у Castle Region, хоча сам Unit локально розміщений у Castle за правилами секції 11.

Тобто у V1 кожний active Knight завжди належить рівно одній Army; окремого стану "Knight без Army" немає.

---

## 2. Pre-battle Retreat decision = false для Army у Regrouping

### Невизначеність

Під час Regrouping effective Defense Loss Threshold примусово дорівнює 0.

Водночас potential defender може задавати й змінювати pre-battle Retreat decision до Battle Start. Не визначено, чи explicit рішення "не Retreat" може перебити forced `0` Regrouping.

### Пропоноване рішення

Не може.

Regrouping є вищим за manual pre-battle decision правилом. Якщо Army на Battle Start усе ще в Regrouping, її effective Defense Loss Threshold = 0 незалежно від раніше заданого `retreat = false`.

Manual decision може:
- примусово поставити `0` звичайній Army;
- скасувати власний попередній manual `0` до Battle Start.

Але воно не може дозволити Regrouping Army залишитися в battle всупереч forced Regrouping threshold.

Як завжди, якщо legal Retreat немає, side-level правило замінює effective threshold на 100%.

---

## 3. Castle Founding завершується під час уже Active CombatSituation з foreign Transit

### Невизначеність

Founding у власній annexed Region не pause-иться через pure foreign Transit.

Foreign Transit через таку Owned Region може вже мати Active CombatSituation і чекати Battle Start. За цей `Dt` Founding може завершитися, і Region стане Castle Region.

Правила одночасно кажуть:
- Castle Region не можна атакувати;
- foreign Transit, який уже почався до завершення Founding, має право завершити Transit.

Не визначено, що робити з уже Registered/Active CombatSituation цього grandfathered Transit.

### Пропоноване рішення

CombatSituation, зареєстрована **до** completion Founding, не скасовується лише через появу Castle.

Вона завершується за своїм звичайним lifecycle:
- `Allow` -> Transit продовжується;
- `Fight`/Aggressive -> battle може відбутися навіть уже в новій Castle Region;
- після цього surviving Transit Army має право завершити лише вже розпочатий Transit через Castle Region.

Castle immunity блокує тільки **нові** hostile entries і нові CombatSituation, що вимагали б нового hostile entry після completion Founding.

---

## 4. Founding нового Castle в Region, що була bridge старого Castle

### Невизначеність

При Founding у власній annexed Region вона перестає належати старому Castle і стає Castle Region нового Castle.

Не сказано прямо, що відбувається з іншими Region старого Castle, для яких ця Region була єдиним territorial bridge.

### Пропоноване рішення

Completion Founding вважається остаточним вилученням цієї Region із territorial graph старого Castle.

Одразу після створення нового Castle перераховується connectivity старого Castle. Усі його ordinary Region, які більше не мають безперервного шляху до старої Castle Region, автоматично стають Neutral за звичайним правилом final disconnection.

Їх City/ResourceSite/active ResourceSiteUpgrade поводяться так само, як при будь-якій іншій автоматичній втраті Region.

---

## 5. Foreign Retreat priority, якщо на карті немає Neutral Region

### Невизначеність

Для legal foreign Owned candidate пріоритет тепер визначається мінімальною hex-distance до найближчої Neutral Region.

Правила не визначають fallback, якщо Neutral Region на карті взагалі немає.

### Пропоноване рішення

Якщо Neutral Region немає, для foreign candidates використовувати мінімальну hex-distance до будь-якої власної non-Occupied Region Player.

Тобто fallback зберігає той самий задум: Army намагається найкоротшим шляхом вийти з чужої territory.

Якщо і після цього candidates рівнозначні — seeded random tie-breaker.

---

## 6. Battle Experience у боях проти Neutral Defense та City Defense

### Невизначеність

Визначено eligibility Knight для Battle Experience, але не сказано, чи "battle" включає:
- Attack Neutral Defense;
- City Raid.

Обидві дії використовують спільний combat calculation, але не є player-vs-player CombatSituation.

### Пропоноване рішення

Так. Battle Experience нараховується після **будь-якого фактично виконаного combat calculation**:

- player-vs-player battle;
- Neutral Defense attack;
- City Raid.

Для Neutral Defense / City Defense opponent scale визначається їх pre-combat defender strength. Правила eligibility ті самі: XP отримують тільки surviving Knight атакуючої Army, які реально брали участь у calculation; Commander отримує Commander bonus.

---

## 7. Raid власного City

### Невизначеність

City Raid описаний як локальна дія Army у Camp. Є правила для Neutral та foreign Owned Region, але немає прямої заборони Player Raid-ити City у власній Owned Region.

### Пропоноване рішення

Заборонити Raid City, якщо formal owner Region == Player атакуючої Army.

City Raid призначений тільки для City, яке не належить attacker. Neutral City можна Raid-ити; foreign Owned/Occupied City — за правилами доступу до Camp.

---

## 8. Наступна Region стала foreign Castle Region після фіксації local exit

### Невизначеність

Поточний exit Transit фіксується і не може бути змінений Player.

Перед фактичним border entry доступність next Region перевіряється повторно. Тому Region може стати foreign Castle Region уже після фіксації exit, і entry буде заборонений.

Без окремого правила Army опиняється з immutable exit, який виконати неможливо.

### Пропоноване рішення

Якщо вже зафіксований next entry стає забороненим **до перетину border**, поточна local movement phase аварійно скасовується.

Army:
- лишається в current Region;
- переходить до Camp цієї current Region;
- шлях до Camp займає `Dt` від моменту виявлення blocked entry;
- після Camp arrival входить у звичайний Camp без Regrouping;
- старий Route завершується.

Якщо current Region foreign Owned non-Occupied, такий вимушений Camp arrival проходить через звичайні territorial interaction rules і може створити CombatSituation/Occupation.

Це не дає Player безкоштовно змінити зафіксований exit: fallback спрацьовує тільки коли сам entry став юридично неможливим.

---

## 9. Founding completion і стан City/ResourceSite самої Region

### Невизначеність

При створенні нового Castle явно описані нові Warehouse, Granary, Palace та transfer founder Knight, але не сказано окремо, чи існуючі City та ResourceSite Region переживають перетворення ordinary Region -> Castle Region.

З інших правил це логічно випливає, але для destructive/non-destructive semantics Founding краще мати пряме правило.

### Пропоноване рішення

Founding не reset-ить природний/міський стан Region:

- City зберігається разом із `wealth` та `active_wealth_ratio`;
- ResourceSite та їх levels зберігаються;
- active ResourceSiteUpgrade продовжуються без reset;
- поточний Neutral Defense після створення Castle більше не має gameplay-функції, бо Region перестає бути Neutral.

Founding змінює territorial/economic role Region, а не створює новий hex або нову Region entity.
