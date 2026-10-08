# Феодали — Загальні положення

## Версія 1

Навігація по документах:

- [Загальні положення](00_OVERVIEW.md)
- [Світ і області](01_WORLD_AND_REGIONS.md)
- [Замки та економіка](02_CASTLES_AND_ECONOMY.md)
- [Армії та лицарі](03_ARMIES_AND_KNIGHTS.md)
- [Переміщення](04_MOVEMENT.md)
- [Бойова система](05_COMBAT.md)
- [Приєднання областей](06_TERRITORY_CONTROL.md)
- [Заснування замків](07_CASTLE_FOUNDING.md)

# 1. Загальна структура світу

Ігровий світ — постійна шестикутна карта, поділена на області (Region). Кожна Region має шість сусідніх Region.

Гравець може мати один або кілька замків (Castle). Початковий Castle є Capital Castle; перенести столицю у V1 не можна. На Capital Region заборонені атаки та чужий Transit. Castle є окремим економічним і військовим центром. До нього прив'язуються його Region, локальні запаси масових ресурсів, будівлі, населення, лицарі та сформовані в ньому загони.

Частина ресурсів зберігається локально в Castle, а частина має єдиний глобальний баланс гравця.

Уся гра працює в ігровому часі (Game Time). Побудова, апгрейди, найм, пересування, перегрупування, приєднання, відновлення та інші процеси використовують Game Time. Для тестування Game Time може прискорюватися.

Події не відбуваються одночасно. Навіть якщо кілька подій мають однаковий номінальний момент Game Time, вони обробляються послідовно у визначеному порядку. Кожна наступна подія бачить уже змінений попередньою подією стан світу.

---

# 32. Видимість і попередження у V1

У V1 стратегічна інформація в основному відкрита. Гравці бачать географію, приналежність і стан Region, Castle, ResourceSite та доступну військову інформацію про чужі Army/Unit. Повноцінна система Fog of War / intelligence відкладається.

Відкритість інформації не скасовує механіку напрямку Movement: майбутній Route чужої Army не показується повністю. Напрямок її виходу з поточної Region стає відомим лише після того, як Army пройшла половину локального часу цієї Region.

Власник Region отримує оперативне попередження про наближення чужої Army, коли її напрямок уже дозволяє визначити вхід у цю Region. Деталі того, як саме ця інформація подається в UI, описуються окремо.

---

# 33. Віртуальні гравці у прототипі

Для перевірки системи можуть використовуватися програмні віртуальні гравці (AI Players) з кількома простими стратегіями.

Вони повинні діяти за тими самими доменними правилами, що й людина. Для спрощення їх Recruitment може бути обмежений одним базовим Soldier Type.

Це тестова поведінка, а не окрема механіка світу.

---


# 34. Що є конфігурацією, а що логікою

До конфігурації належать, зокрема:

- prices, upkeep, capacities та Building effects;
- Construction, Recruitment, Annexation, Founding і recovery rates/times;
- одна базова game-time константа Dt;
- ResourceSite output та DistanceEfficiency;
- coins_per_food;
- Soldier Attack/Defense та Experience functions;
- combat luck/loss functions;
- доступні loss threshold values;
- Casualty Health, mortality coefficients і casualty randomness;
- Neutral Defense full strength/recovery rate;
- City wealth growth, active-ratio recovery, City Defense, Raid threshold/reward.

Окремого Combat Start Delay для Neutral Camp attack немає: player-vs-player CombatSituation Start -> Battle Start завжди дорівнює Dt.

До незмінної логіки V1 належать, зокрема:

- Castle-local і Player-global ресурси;
- layered ResourceSite upgrades;
- Unit/Army structure та два командні рівні;
- event serialization;
- Camp/Movement/Regrouping presence rules;
- фіксація Camp/Transit intent і поточного exit після входу;
- Movement-level target_opponent та його snapshot у CombatSituation;
- Aggressive/NonAggressive Transit classification тільки для Owned non-Occupied Region;
- одна FIFO-черга всіх player-vs-player CombatSituation Region;
- одна attacking Army на CombatSituation;
- side-level Defender threshold як minimum participating Army thresholds;
- side-level Retreat destination;
- Occupation окремо від ownership;
- Annexation як manual action після control progress;
- foreign Camp/Regrouping/entered-for-Camp presence блокує Annexation/Founding, pure Transit — ні;
- City Raid як окрема Camp action;
- gradual Neutral Defense recovery тільки без troops;
- Founding через Knight без Soldier;
- Food shortage compensation у Coins;
- balances Food/Coins не опускаються нижче 0.

---

# 35. Межі V1

У цей документ свідомо не включені системи, яких немає у першій версії: alliances, формальна політична ієрархія, зміна власника Castle (але Occupation Castle Region можлива), terrain movement modifiers, Stable, розширена роль Forge, спеціальні recruitment-speed Building, player-to-player trade та resource transfer, prestige systems, повна intelligence/scouting system, складна supply logistics, persistent HP та інші механіки, перелічені у файлі майбутніх систем.
