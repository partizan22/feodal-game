# Феодали — логіка і механіки гри
## Версія 1

Цей документ описує актуальні правила першої версії гри. Він не описує технології реалізації, структуру коду, формат конфігураційних файлів, інтерфейс користувача, генерацію карти або процес навчання. Числові значення, таблиці балансу, тривалості, ціни, коефіцієнти та конкретний вигляд налаштовуваних функцій задаються окремо в конфігурації.

Якщо формула визначає саму логіку механіки, а не баланс, у документі наводиться її структура із символічними функціями та коефіцієнтами.

---

Цей документ — самодостатній опис основних правил V1. У `docs/rules/` збережено докладну специфікацію: умови, алгоритми, порядок подій, винятки та edge cases. Тематичні документи уточнюють основні правила, не змінюючи їх. Ці основні правила враховують погоджені зміни щодо Capital, Castle Occupation та заблокованого Castle.

## Детальні документи

- [Загальні положення](rules/00_OVERVIEW.md) — §1, §32, §33, §34, §35
- [Світ і області](rules/01_WORLD_AND_REGIONS.md) — §2, §3, §26, §29
- [Замки й економіка](rules/02_CASTLES_AND_ECONOMY.md) — §4, §5, §6, §7, §12
- [Війська](rules/03_ARMIES_AND_KNIGHTS.md) — §8, §9, §10, §11
- [Переміщення](rules/04_MOVEMENT.md) — §13, §14, §15
- [Бойова система](rules/05_COMBAT.md) — §16, §17, §18, §19, §20, §21, §22, §23, §24, §25, §30
- [Приєднання територій](rules/06_TERRITORY_CONTROL.md) — §27, §28
- [Заснування замків](rules/07_CASTLE_FOUNDING.md) — §31

---

# 1. Загальна структура світу

Світ — постійна шестикутна карта областей (Region). Гравець володіє одним або кількома замками (Castle), до яких прив'язані території, господарство й війська. Його початковий Castle є Capital, статус якої у V1 не переноситься. Процеси відбуваються в Game Time, а події, навіть з однаковим часом, обробляються послідовно.

[Детальна специфікація §1](rules/00_OVERVIEW.md).

---

# 2. Стани області та територіальний контроль

Region може бути Neutral, Owned або Occupied. Нейтральна область допускає одночасне перебування Camp різних гравців без автоматичного бою. Власна область дає економічний потік до свого Castle й потребує утримання. Окупована область залишається у власності попереднього owner, але контролюється іншим гравцем через Camp; економічні потоки припиняються. Окупація зникає після виходу останньої Army окупанта з Camp. Formal owner не може пройти через свою Occupied Region Transit, тільки атакувати occupier із метою Camp. Occupation не столичної Castle Region дозволена, але Annexation її заборонена; вона блокує Castle.

[Детальна специфікація §2](rules/01_WORLD_AND_REGIONS.md).

---

# 3. Територіальна зв'язність

Зовнішні Region кожного Castle повинні утворювати безперервний ланцюг власних Region до Castle. Окупація проміжної Region може відрізати економічні потоки, але не змінює власності інших областей. Остаточна втрата територіального мосту робить відрізані Region нейтральними. Власник може відмовитися від звичайної Region, але не від Castle Region. Capital Region не можна атакувати чи пройти чужим Transit; інші Castle Region підпорядковані звичайним правилам Owned/Occupied з винятками блокування Castle і особливого Retreat.

[Детальна специфікація §3](rules/01_WORLD_AND_REGIONS.md).

---

# 4. Ресурси

Wood, Stone, Iron та Food зберігаються локально в Castle; Wood/Stone/Iron — у Warehouse, Food — у Granary. Coins, Gold та Silver — глобальні баланси Player, але Gold/Silver production проходить через Castle: Occupied/disconnected Region та заблокований Castle не дають відповідних надходжень. Надлишок понад місткість сховища не накопичується.

[Детальна специфікація §4](rules/02_CASTLES_AND_ECONOMY.md).

---

# 5. ResourceSite і видобуток

ResourceSite у Region виробляють ресурси залежно від рівня розвитку. Поліпшення відбуваються пошарово й оплачуються за правилами Upgrade. Надходження до Castle залежать від територіального контролю, зв'язності, блокування Castle та DistanceEfficiency.

[Детальна специфікація §5](rules/02_CASTLES_AND_ECONOMY.md).

---

# 6. Food та населення

Population, Soldier та Knight споживають Food. Camp у звичайній Region використовує місцеве виробництво; у неокупованій Castle Region Food власних Army враховується в балансі Castle. Під час Occupation Army occupier та Army власника у Camp поза Castle споживають місцеве Food за звичайними правилами, не зі складів Castle; Army у Movement компенсують Food у Coins аж до фактичного входу в Camp. У Movement замість Food сплачуються Coins; дефіцит Food також компенсується додатковими Coins. Фактичні запаси Food і Coins не можуть ставати від'ємними, а дефіцит може призупиняти Recruitment.

[Детальна специфікація §6](rules/02_CASTLES_AND_ECONOMY.md).

---

# 7. Будівлі Castle

Castle має Building з рівнями, тривалістю та вартістю покращень. Якщо Castle Region окупована, Castle заблокований: повністю припиняються надходження ресурсів із Region і Coins від прив'язаних City, але Coin income власних Building і регулярні витрати зберігаються. Будівництво, Recruitment і створення Knight залишаються дозволеними за звичайних вимог. Palace визначає місця Knight і механізм їх заміни; Governor's House — кількість зовнішніх Region; Barracks — місткість Soldier і Recruitment. Warehouse/Granary зберігають ресурси, Forge відкриває Barracks, економічні Building виробляють Coins, Bank збільшує відповідний дохід.

[Детальна специфікація §7](rules/02_CASTLES_AND_ECONOMY.md).

---

# 8. Knight

Knight — індивідуальний командир з ім'ям, home Castle і Experience. Він може існувати без Soldier. Experience зростає з часом і за участь у боях; бойовий досвід отримують лише лицарі, які вижили й брали участь у розрахунку.

[Детальна специфікація §8](rules/03_ARMIES_AND_KNIGHTS.md).

---

# 9. Soldier і Recruitment

V1 використовує шість Soldier Type: Light Infantry, Spearman, Swordsman, Heavy Infantry, Halberdier та Rider. Soldier одного типу враховуються кількістю, а не як індивідуальні об'єкти. Recruitment здійснюється у Castle, створює reserve Soldier та потребує часу, ресурсів і місця в Barracks.

[Детальна специфікація §9](rules/03_ARMIES_AND_KNIGHTS.md).

---

# 10. Unit та Army

Unit містить рівно одного Knight та нуль або більше Soldier; склад можна змінювати тільки в home Castle. Army об'єднує один або більше Unit і має Commander-in-Chief з їхніх Knight. Об'єднання й поділ Army підпорядковані правилам стану й розташування. Knight не може загинути, доки в його Unit залишаються Soldier.

[Детальна специфікація §10](rules/03_ARMIES_AND_KNIGHTS.md).

---

Очікувана модель Army: рівно п’ять територіальних станів — `Camp`, `Transit`, `LeavingCamp`, `EnteringCamp`, `BlockedInCastle`. `Regrouping` і combat command-lock — додаткові обмеження, а не територіальні стани. `Regrouping` застосовується до всієї Army тільки у `Camp` (у Castle Region — лише після відступу із сусідньої Region), тоді як combat command-lock може діяти й під час Movement і не забороняє перемикати режим розміщення лицарів.

Кожен Knight у Castle Region, у тому числі без Soldier, має режим розміщення **«у замку»** або **«поза замком»**; це не територіальний стан Army. Перехід між режимами миттєвий, у тому числі під час Regrouping та активної CombatSituation (попри command-lock), і доступний у будь-якому Castle власного Player незалежно від home Castle Knight, за достатньої Barracks Capacity; у чужому Castle неможливий. Movement усієї Army починається одночасно для всіх Unit. При Occupation Knight у режимі «у замку» переходять до `BlockedInCastle`; якщо вся Army залишається у замку, вона не розділяється. Якщо Army змішана, при Occupation кожен Unit «у замку» відділяється в окрему заблоковану Army, а всі Unit «поза замком» **залишаються разом у початковій Army** й відступають. Це стосується також Army, що вийшла з battle calculation через pre-battle Retreat / threshold `0`: її Unit не отримують бойових втрат. Якщо Commander залишився в замку, у відступаючій Army його замінює Knight з найбільшим Experience (при рівності — випадковий вибір з ігровим seed). Після завершення Occupation заблоковані Army повертаються у `Camp` без Regrouping, зі збереженням складу, Commander і режиму розміщення. Заблоковані Army не взаємодіють із зовнішніми військами; внутрішні merge/split між Army того самого заблокованого Castle та зміна Commander дозволені як виняток із вимоги звичайного `Camp`. Змінювати Soldier Unit можна лише в home Castle цього Knight.

[Детальна специфікація §11](rules/03_ARMIES_AND_KNIGHTS.md).

---

# 12. Upkeep військ

Knight, Soldier і резервні Soldier мають постійний Coin upkeep окремо від Food consumption. Upkeep залежить від актуального місця та стану військ; минулі періоди не перераховуються. Нестача Food створює додаткові витрати Coins.

[Детальна специфікація §12](rules/02_CASTLES_AND_ECONOMY.md).

---

# 13. Route, Movement, Dt і фіксація наміру

Гравець задає Army Route та кінцеву Region. Для поточної Region Army входить із наміром Camp або Transit, який фіксується після входу. Пересування використовує базову тривалість Dt; зміна обставин не переписує вже зафіксований намір чи маршрутні дані.

[Детальна специфікація §13](rules/04_MOVEMENT.md).

---

# 14. Transit і Aggressive Transit

Transit означає прохід без зупинки у Camp. Formal owner не може Transit через власну Occupied Region; чужий Transit через Capital Region заборонений. У Neutral Region він не створює територіального CombatSituation. Для Owned non-Occupied Region Transit може бути Aggressive або NonAggressive залежно від зафіксованого target opponent; Occupied Region має окремі правила.

[Детальна специфікація §14](rules/04_MOVEMENT.md).

---

# 15. Allow Transit для Owned non-Occupied Region

При NonAggressive Transit через Owned non-Occupied Region owner може дозволити прохід або битися, якщо є оборонний контекст. За відсутності явного вибору застосовується поточне Allow Transit. Для Aggressive Transit це правило не діє.

[Детальна специфікація §15](rules/04_MOVEMENT.md).

---

# 16. CombatSituation: Registration, Start і FIFO

Кожна CombatSituation має одну Attacker Army та Defender side й проходить стадії Registration, Start, Battle Start. Усі player-vs-player CombatSituation однієї Region впорядковані у FIFO-чергу. Кілька атакуючих Army формують окремі situations. Camp-bound entry у чужу Owned/Occupied Region (включно з Retreat) реєструє CombatSituation лише за наявності чужих Army у Camp або раніше введених для Camp Army в `EnteringCamp`; інакше Camp досягається за звичайний `Dt` без CombatSituation. Вхід formal owner у власну ще non-Occupied Region з чужою `EnteringCamp` Army також реєструє CombatSituation. Кожен наступний Camp-bound entry за наявності чужої Army створює власну situation у FIFO; defender визначається на Start. Якщо попередній бій усунув interaction, queued situation завершується без battle. Camp-bound territorial CombatSituation у чужій Owned/Occupied Region **не завершується достроково лише через відсутність defender Army**: через повний `Dt` на Battle Start Attacker автоматично перемагає порожню defender side й одразу переходить у `Camp` без додаткового `Dt`; зміни Occupation / Owned застосовуються до Start наступної CombatSituation у FIFO.

[Детальна специфікація §16](rules/05_COMBAT.md).

---

# 17. Defender participation і pre-battle decisions

На CombatSituation Start визначаються потенційні захисники, включно з Camp, Regrouping та Army, що вже прямують до Camp. Defender може ухвалювати передбойові рішення; дозволений pre-battle Retreat виводить війська з подальшого бойового розрахунку.

[Детальна специфікація §17](rules/05_COMBAT.md).

---

# 18. Combat roles і loss thresholds

Attacker має Target та Incidental Combat Threshold, обидва більше за нуль. Defender side використовує спільний поріг на основі participating Army. Поріг визначає готовність сторони залишити бій після відповідних втрат.

[Детальна специфікація §18](rules/05_COMBAT.md).

---

# 19. Player-vs-player combat у Neutral Region

У Neutral Region Army у Camp **без Regrouping** може окремою дією атакувати Camp конкретного іншого Player, **включно з Army-захисником у Regrouping**. Army в Regrouping сама не може ініціювати Attack. Така атака створює CombatSituation; інші власні Army автоматично до Attacker не приєднуються.

[Детальна специфікація §19](rules/05_COMBAT.md).

---

# 20. Розрахунок сили Unit і Army

Сила Unit складається із сили Soldier для потрібного Attack/Defense режиму, особистої сили Knight і коефіцієнта Experience. Army отримує ефект свого Commander-in-Chief. Спільна Defender side підсумовує сили окремих Army.

[Детальна специфікація §20](rules/05_COMBAT.md).

---

# 21. Загальна логіка Combat

Combat порівнює початкові сили сторін із випадковим Luck Factor. Функції втрат і loss thresholds визначають результат бою та частку втрат. Конкретні балансні функції й параметри належать конфігурації.

[Детальна специфікація §21](rules/05_COMBAT.md).

---

# 22. Об'єднана Defender side

Кілька Army Defender беруть участь як одна сторона, але зберігають власні структури, командирів і коефіцієнти. Поріг відступу й допустимий Retreat destination визначаються на рівні сторони.

[Детальна специфікація §22](rules/05_COMBAT.md).

---

# 23. Retreat destination

Retreat здійснюється в допустиму сусідню Region з урахуванням геометрії, присутності військ і пріоритетів. Для однієї combat side обирається один destination. Якщо законного напрямку немає, загальне правило примушує сторону битися з порогом втрат `100%`; поразка означає повне знищення учасників бою. Винятки для Castle Region: при Camp-bound атаці повністю розміщені «у замку» Army захисника з effective threshold `0` достроково виходять із бою без втрат, а решта б'ються з порогом `100%`; при Transit-атаці діють звичайні пороги без потреби у сусідньому напрямку Retreat.

[Детальна специфікація §23](rules/05_COMBAT.md).

---

# 24. Наслідки Retreat, перемоги та Regrouping

Після звичайного Retreat Army входить у сусідню Region і прямує до Camp. Якщо це чужа Owned Region, вона спочатку проходить CombatSituation (за потреби очікує у FIFO); лише фактичне досягнення Camp запускає Regrouping. Якщо під час нової CombatSituation армія знову програє та Retreat-ить, починається наступний Retreat, а не Regrouping у цій Region. Для Camp-bound атаки Castle Region без legal Retreat на Battle Start лише Army, повністю розміщені «у замку», з effective threshold `0` (включно з pre-battle Retreat) достроково виходять із бою без втрат; усі інші Army б'ються з порогом `100%` і при поразці повністю знищуються. При Occupation достроково виведені Army залишаються в замку в `BlockedInCastle`, не розділяючись; при перемозі захисника розміщення не змінюється. Виняток для не столичної Castle Region: після поразки від Camp-bound Attacker Unit у режимі «у замку» залишаються всередині та блокуються, Unit «поза замком» відступають. Після перемоги Transit Attacker у не столичній Castle Region усі Defender Army залишаються в цій Region без Retreat і Regrouping, зберігаючи попередній режим розміщення: «у замку» залишається «у замку», «поза замком» — «поза замком». Це стосується як учасників бою, так і тих, хто не брав участі через pre-battle Retreat або нульовий поріг втрат. Pre-battle Retreat теж вважається програшем. При звільненні Castle заблоковані Army не беруть участі в бою, а заміна occupier не завершує облогу. Переможець, залежно від наміру Camp або Transit, займає Camp, створює Occupation чи продовжує Route. Regrouping забороняє частину команд, але вважається Camp-presence для інших механік.

[Детальна специфікація §24](rules/05_COMBAT.md).

---

# 25. Casualties

Втрати визначаються через тимчасову Casualty Health, без постійного HP. Загальний budget втрат розподіляється між Unit і типами Soldier з перерозподілом надлишку. Knight не гине, поки в його Unit залишилися Soldier.

[Детальна специфікація §25](rules/05_COMBAT.md).

---

# 26. Neutral Defense

Neutral Defense — абстрактна оборона Neutral Region, а не Army. Її можна атакувати окремою миттєвою дією з Camp, поза CombatSituation. Якщо є City, при цій атаці враховується її Defense. Neutral Defense відновлюється за передбачених умов.

[Детальна специфікація §26](rules/01_WORLD_AND_REGIONS.md).

---

# 27. Annexation Neutral Region

Annexation Neutral Region потребує Neutral Defense = 0, Camp-presence, територіального зв'язку та відсутності блокуючих чужих військ. Після накопичення control progress гравець виконує окрему дію Annexation з перевіркою Governor's House Capacity і Knight потрібного незаблокованого Castle.

[Детальна специфікація §27](rules/06_TERRITORY_CONTROL.md).

---

# 28. Annexation Occupied Region

Current occupier може Annex звичайну Occupied Region за близькою процедурою, але не Castle Region; до заблокованого Castle нові Region також не можна Annex. Після успіху Region переходить до обраного Castle occupier, а зв'язність попереднього власника перераховується.

[Детальна специфікація §28](rules/06_TERRITORY_CONTROL.md).

---

# 29. City

City має довгостроковий wealth і частку active_wealth_ratio. Їх добуток задає поточне effective wealth. Зростання багатства, Coin income, відновлення активної частки й залежність від стану Region визначаються детальними правилами.

[Детальна специфікація §29](rules/01_WORLD_AND_REGIONS.md).

---

# 30. City Raid

City Raid — окрема миттєва дія Army у Camp проти City Defense; Neutral Defense не бере участі. Власну City атакувати не можна. Успіх дає Coins і скидає active_wealth_ratio; поразка залишає Army у тій самій Region; після поразки починається Regrouping; City Raid у Castle Region неможливий, оскільки заснування Castle в Region із City заборонене.

[Детальна специфікація §30](rules/05_COMBAT.md).

---

# 31. Заснування нового Castle

У V1 Castle можна заснувати тільки в Neutral Region із Neutral Defense = 0 **та без City**. До завершення Founding діють Neutral rules, після — Owned rules; уже розпочатий Transit завершується без CombatSituation. Новий Castle не є Capital. Founder — Knight без Soldier у Camp; Founding має оплачуваний Start, час виконання, умови pause і cancel. Після завершення створюється Castle, зберігаються існуючі об'єкти Region, а founder переходить у його Palace slot. Уже розпочатий чужий Transit може завершитися.

[Детальна специфікація §31](rules/07_CASTLE_FOUNDING.md).

---

# 32. Видимість і попередження у V1

Основна стратегічна інформація відкрита, але повний майбутній Route чужої Army не показується. Напрямок руху розкривається за окремим часовим правилом, а owner отримує попередження про наближення.

[Детальна специфікація §32](rules/00_OVERVIEW.md).

---

# 33. Віртуальні гравці у прототипі

Для прототипу можливі AI Players із простими стратегіями; вони використовують ті самі правила, що й гравці.

[Детальна специфікація §33](rules/00_OVERVIEW.md).

---

# 34. Що є конфігурацією, а що логікою

Ціни, тривалості, параметри, функції й коефіцієнти задає конфігурація. Структурні інваріанти Army/Unit, порядок подій, правила володіння, руху, бою, Annexation і Founding — частина фіксованої логіки.

[Детальна специфікація §34](rules/00_OVERVIEW.md).

---

# 35. Межі V1

V1 не включає повноцінні alliances, політичну ієрархію, зміну власника Castle (хоча Occupation Castle Region можлива), торгівлю між Player, Fog of War, складну логістику, persistent HP та інші системи, відкладені на пізніші версії.

[Детальна специфікація §35](rules/00_OVERVIEW.md).

---
