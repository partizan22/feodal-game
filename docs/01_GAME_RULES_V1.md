# Феодали — логіка і механіки гри
## Версія 1

Цей документ описує актуальні правила першої версії гри. Він не описує технології реалізації, структуру коду, формат конфігураційних файлів, інтерфейс користувача, генерацію карти або процес навчання. Числові значення, таблиці балансу, тривалості, ціни, коефіцієнти та конкретний вигляд налаштовуваних функцій задаються окремо в конфігурації.

Якщо формула визначає саму логіку механіки, а не баланс, у документі наводиться її структура із символічними функціями та коефіцієнтами.

---

# 1. Загальна структура світу

Ігровий світ — постійна шестикутна карта, поділена на області (Region). Кожна Region має шість сусідніх Region.

Гравець може мати один або кілька замків (Castle). Castle є окремим економічним і військовим центром. До нього прив'язуються його Region, локальні запаси масових ресурсів, будівлі, населення, лицарі та сформовані в ньому загони.

Частина ресурсів зберігається локально в Castle, а частина має єдиний глобальний баланс гравця.

Уся гра працює в ігровому часі (Game Time). Побудова, апгрейди, найм, пересування, перегрупування, приєднання, відновлення та інші процеси використовують Game Time. Для тестування Game Time може прискорюватися.

Події не відбуваються одночасно. Навіть якщо кілька подій мають однаковий номінальний момент Game Time, вони обробляються послідовно у визначеному порядку. Кожна наступна подія бачить уже змінений попередньою подією стан світу.

---

# 2. Стани області та територіальний контроль

Region може бути в одному з трьох базових станів:

- нейтральна (Neutral);
- належить гравцю (Owned);
- окупована (Occupied).

## 2.1 Neutral

Neutral Region не має власника.

У ній можуть одночасно перебувати в Camp війська різних гравців. Сам факт такої присутності не запускає бій.

Транзит через Neutral Region завжди дозволений і не може бути заблокований іншими гравцями.

Neutral Region може містити ресурси, Food production, Neutral Defense та City.

## 2.2 Owned

Owned Region формально належить одному гравцю та прив'язана до конкретного його Castle.

Її економічні потоки надходять до цього Castle, крім глобальних ресурсів.

Власник сплачує регулярне утримання Region. Розмір утримання залежить від відстані до Castle через налаштовувану функцію.


## 2.3 Occupied

Occupied Region формально продовжує належати попередньому власнику, але в її Camp є війська одного ворожого Player — поточного occupier.

Під час Occupation:

- формальний owner не змінюється і продовжує сплачувати утримання Region;
- Region не дає формальному owner нормального економічного потоку;
- occupier не отримує її економічний потік до Annexation;
- Occupation припиняється в момент, коли **остання Army occupier залишає Camp**, а не лише після її виходу за межі Region; контроль одразу повертається формальному owner.

У Region не може одночасно бути кілька occupier. Якщо третій Player входить у цю Region з локальною метою Camp, він взаємодіє з поточним occupier; після перемоги й переходу в Camp він стає новим occupier. Якщо перемагає формальний owner, Occupation припиняється й відновлюється звичайний Owned state.

Transit через Occupied Region не створює CombatSituation незалежно від того, хто проходить Region, включно з формальним owner. Для такого Transit не застосовуються Allow Transit та Aggressive/NonAggressive classification.

Якщо формальний owner входить у свою Occupied Region з локальною метою **Camp**, він атакує поточного occupier.

---

# 3. Територіальна зв'язність

Звичайна Region може бути Annexed до Castle лише тоді, коли вона межує принаймні з однією Region, яка вже належить цьому самому Castle. Таким чином усі звичайні володіння Castle повинні утворювати безперервний ланцюг до Castle Region.

Якщо проміжна Owned Region стає Occupied, усі Owned Region за нею, що втратили безперервний шлях до Castle, не втрачаються юридично, але їх Wood/Stone/Iron/Food потоки та Gold/Silver income припиняються, доки зв'язок не відновиться.

Якщо проміжна Region остаточно втрачається власником, усі його Region, які після цього не мають безперервного ланцюга власних Region до свого Castle, також автоматично втрачаються і стають Neutral.

При такій автоматичній втраті:

- рівні ResourceSite зберігаються;
- Wealth City, якщо City є, зберігається, але перестає зростати, поки Region Neutral;
- Neutral Defense надалі відновлюється за звичайними правилами Neutral Region;
- війська колишнього власника, які вже перебувають у Camp у цій Region, залишаються на місці й надалі вважаються військами в Camp у Neutral Region.

Castle Region є винятком із звичайних правил війни у V1: її не можна атакувати і через неї не можна прокладати новий ворожий Transit. Якщо чужа Army вже почала Transit через Region до моменту завершення Founding Castle, вона має право завершити цей уже розпочатий Transit.

---

# 4. Ресурси

## 4.1 Локальні масові ресурси

До локальних ресурсів належать:

- дерево (Wood);
- камінь (Stone);
- залізо (Iron);
- їжа (Food).

Wood, Stone, Iron і Food зберігаються окремо для кожного Castle.

Wood, Stone та Iron зберігаються на складі (Warehouse). Warehouse має одну місткість, яка застосовується незалежно до кожного з трьох ресурсів: заповнення Wood не зменшує доступну місткість Stone або Iron.

Food зберігається в амбарі (Granary).

Якщо відповідне сховище заповнене, надлишковий ресурс, який мав бути зарахований до Castle, втрачається.

## 4.2 Глобальні ресурси

Глобальний баланс гравця мають:

- монети (Coins);
- золото (Gold);
- срібло (Silver).

Coins є основною валютою та використовуються в розвитку й регулярних витратах.

Gold є рідкісним стратегічним ресурсом розвитку і може використовуватися вже відносно рано, але в малих кількостях.

Silver у V1 видобувається та накопичується, але ще не витрачається.

Coins, Gold та Silver не мають складської місткості.

---


# 5. ResourceSite і видобуток

Region може містити кілька природних ResourceSite одного ресурсу.

ResourceSite не обов'язково існують як окремі іменовані об'єкти. Для кожного ресурсу в Region достатньо зберігати кількість ResourceSite кожного рівня.

Природна кількість ResourceSite є властивістю Region. Їх рівень розвитку може змінювати власник за правилами Upgrade.

Для одного типу ресурсу ResourceSite розвиваються пошарово. Гравець може покращувати будь-яку кількість ResourceSite, що знаходяться на поточному мінімальному рівні. Не можна починати наступний рівень для частини ResourceSite, доки всі ResourceSite цього ресурсу в Region не досягли попереднього рівня.

Базове виробництво визначається без distance penalty:

LocalProduction = Σ ResourceSiteOutput(level_i)

Для Wood, Stone та Iron у Owned non-Occupied Region з чинною територіальною зв'язністю позитивне виробництво надходить до Castle з коефіцієнтом відстані:

CastleFlow = LocalProduction × DistanceEfficiency(distance)

Для Gold і Silver distance coefficient не застосовується. Їх production зараховується до глобального балансу Player тільки якщо Region є Owned, не Occupied і має чинну територіальну зв'язність.

Occupied або disconnected Region не передає ці ресурси формальному owner.

Розвиток ResourceSite має власну вартість і час, що задаються конфігурацією.

---


# 6. Food та населення

Будівлі Castle створюють Population відповідно до конфігурації. Population у V1 не є окремою складною системою і впливає насамперед на постійне споживання Food у Castle.

Один Soldier незалежно від Type споживає однакову кількість Food. Один Knight також рахується як одна food-consumption unit.

Food shortage не замінює звичайний Coin upkeep військ: компенсація нестачі Food у Coins додається поверх нього. Для всієї гри використовується один конфігураційний курс coins_per_food.

## 6.1 Army у Camp звичайної Region

Army у Camp або Regrouping споживають локальне Food production Region.

Якщо Food достатньо, локальне production покриває весь Camp consumption. Якщо Food недостатньо, доступна кількість ділиться між Player пропорційно сумарному Food consumption їхніх Camp-військ у цій Region. Жоден Player або Army не має пріоритету.

Непокритий дефіцит кожного Player компенсується Coins за coins_per_food.

Для звичайної Owned non-Occupied Region з чинною територіальною зв'язністю тільки **позитивний** залишок Food після Camp consumption передається до Castle з DistanceEfficiency(distance). Castle не постачає Food назад у звичайні Region. Негативний локальний balance ніколи не перетворюється на demand до Castle.

У Neutral, foreign, Occupied або disconnected Region діє та сама локальна Camp-consumption логіка, але позитивний залишок не надходить до Castle формального owner.

## 6.2 Castle Region

Castle Region є спеціальним випадком.

Food consumption Army у Camp/Regrouping Castle Region не віднімається від локального production самої Region. Уся позитивна Food production Castle Region надходить у Castle з coefficient 1.

Food consumption усіх Army у Camp/Regrouping Castle Region віднімається вже на рівні Food balance Castle разом із non-military consumption.

Якщо запас Food Castle дорівнює 0 і його Food balance від'ємний, непокрита частина компенсується Coins за тим самим coins_per_food.

## 6.3 Army у Movement

Army у Movement не використовує Food жодної Region або Castle.

Увесь її Food-equivalent consumption повністю компенсується Coins за coins_per_food протягом усіх movement phases, включно з Transit, final-local, retreat-local та paused-for-combat Movement.

## 6.4 Дефіцит

Фактичний запас Food і Coins не може ставати від'ємним. Від'ємний розрахунковий balance сам по собі не забороняє Building Upgrade або іншу одноразову дію, якщо для її upfront cost достатньо фактичних ресурсів.

Castle має empty_food, коли Food == 0 && food_balance <= 0. У цьому стані Recruitment не може починатися або продовжуватися. Додаткова Coin compensation виникає тільки при food_balance < 0.

Player має empty_coins, коли Coins == 0 && coin_balance <= 0. У цьому стані:

- active Recruitment pause-иться;
- нові Recruitment orders не створюються;
- Wealth City не зростає.

Після зникнення блокуючої умови призупинені процеси автоматично можуть продовжитися.

V1 не вводить окремих штрафів голодування, автоматичного розпуску військ або загибелі населення через одночасну відсутність Food і Coins.

---

# 7. Будівлі Castle

Для більшості Building рівень відсутньої будівлі вважається нульовим; побудова — це перехід до першого побудованого рівня.

Кілька різних Building можуть одночасно будуватися або покращуватися. Одна й та сама Building не може виконувати кілька послідовних Upgrade паралельно.

У V1 немає ліміту слотів Building.

Вартість і час Construction/Upgrade є конфігураційними.

## 7.1 Warehouse

Зберігає Wood, Stone та Iron. Його Capacity застосовується окремо до кожного з них.

## 7.2 Granary

Зберігає Food.

## 7.3 Palace

Палац (Palace) визначає кількість доступних місць для Knight у конкретному Castle.

Кількість Knight slots структурно дорівнює Palace level. Кожне збільшення Palace level відкриває одне нове місце для Knight. Новий Knight створюється автоматично для вільного місця.

Якщо Knight гине, Palace level не зменшується. Вільне місце автоматично заповнюється новим Knight після налаштовуваного часу. В одному Palace заміна загиблих Knight відбувається послідовно через одну чергу.

Palace має регулярний Coin upkeep за функцією його level.

## 7.4 Governor's House

Будинок губернатора (Governor's House) визначає максимальну кількість зовнішніх Region, які можуть бути Annexed до цього Castle. Структурно ця Capacity дорівнює Governor's House level.

Castle Region у цей ліміт не входить.

Capacity перевіряється саме в момент виконання Annexation, а не при початку накопичення часу контролю.

## 7.5 Forge

Кузня (Forge) у V1 є передумовою для побудови Barracks.

Інших ефектів Forge у V1 немає.

## 7.6 Barracks

Казарма (Barracks) потрібна для Recruitment і для дешевого розміщення Soldier у Castle.

Barracks має Capacity. У ній знаходяться як неназначені Soldier, так і Soldier сформованих Unit, що переведені зі стану Camp у Castle.

Unit не може перейти з Camp у Castle, якщо Barracks не має достатньо вільної Capacity для всіх його Soldier. Частковий вхід не допускається.

Якщо Barracks заповнена, Recruitment pause до появи вільного місця.

## 7.7 Coin-producing Buildings

У Castle можуть існувати Building, що створюють Coin income. Конкретний набір таких Building і функції доходу задаються конфігурацією.

Market може існувати як економічна Building і приносити Coins навіть якщо повноцінна торгівля між гравцями ще не реалізована.

## 7.8 Bank

Банк (Bank) множить сумарний eligible Coin income інших Building.

Структурна формула:

`FinalIncome = Σ EligibleBuildingIncome_i × BankMultiplier(level)`

Конкретна функція `BankMultiplier` належить конфігурації.

---

# 8. Knight

Лицар (Knight) є індивідуальним юнітом і командиром.

Knight має:

- ім'я;
- home Castle;
- базову особисту бойову силу;
- Experience;
- коефіцієнт, що залежить від Experience;
- стан життя/заміни.

Knight може існувати без Soldier.

Experience зростає після боїв залежно від масштабу противника та повільно з часом. Конкретні функції зростання й перетворення Experience у коефіцієнт задаються конфігурацією.

Жорсткої верхньої межі Experience немає; ефект Experience має зростати зі спадною віддачею.

---

# 9. Soldier і Recruitment

У V1 є такі типи солдатів (Soldier Type):

- легкий піхотинець (Light Infantry) — дешевий універсальний;
- списник (Spearman) — орієнтований на Defense;
- мечник (Swordsman) — орієнтований на Attack;
- важкий піхотинець (Heavy Infantry) — сильний універсальний;
- алебардник (Halberdier) — елітний Defense;
- вершник (Rider) — елітний Attack.

Rider у V1 не потребує Stable.

Soldier одного Type не є індивідуальними сутностями; система оперує кількістю Soldier кожного Type.

Для Type конфігурація задає Attack strength, Defense strength, Recruitment cost/time та upkeep. Food consumption у V1 однакове для всіх Type; елітні Type мають вищий Coin upkeep.

## 9.1 Recruitment

Recruitment виконується в Barracks конкретного Castle.

Гравець обирає Soldier Type і кількість. Вартість списується при постановці замовлення.

Кожний Castle має одну Recruitment Queue. У ній може бути кілька замовлень, які виконуються послідовно. Soldier з'являються поступово відповідно до Recruitment time.

Якщо для наступного Soldier немає вільної Barracks Capacity, Queue ставиться на pause. Уже сплачена вартість не повертається.

Recruitment також ставиться на pause, якщо Castle перебуває у стані нестачі Food або Player перебуває у стані нестачі Coins за правилами розділу 6.4. У цих станах нове замовлення Recruitment створити не можна. Після зникнення блокуючої умови Queue автоматично продовжується, якщо Barracks має Capacity.

Скасування незавершеного Recruitment у V1 немає.

---


# 10. Unit та Army

Gameplay Unit складається рівно з одного Knight і нуля або більше Soldier. Knight без Soldier є повноцінним Unit.

Unit має home Castle, який збігається з home Castle його Knight. Внутрішній склад Unit можна змінювати лише у home Castle за звичайних умов Barracks.

Кілька Unit можуть бути об'єднані в Army. Army має одного Commander-in-Chief, вибраного серед Knight цієї Army. Army не містить вкладених Army; при merge попередня Army-структура не зберігається.

У V1 немає жорсткого ліміту Soldier у Unit або Unit в Army.

Merge і split дозволені тільки Army у звичайному Camp одного Player і одного CampInRegion, якщо вони не command-locked CombatSituation. Regrouping merge/split забороняє. Зміна Commander сама по собі під час Regrouping дозволена, якщо немає command lock.

Зміна складу Soldier виконується тільки через Castle reserve <-> Knight у home Castle і також не допускається для command-locked Army.

Кожна Army має три persistent loss threshold:

- Defense Loss Threshold;
- Target Combat Threshold;
- Incidental Combat Threshold.

Threshold можна змінювати під час Camp, Regrouping або Movement, доки Army не command-locked CombatSituation. Після Registration attacker уже locked; potential defender отримує lock на CombatSituation Start. Значення конкретного battle остаточно фіксуються на Battle Start.

При merge thresholds нової Army задаються явно. Після split нові Army отримують поточні thresholds вихідної Army, доки Player не змінить їх.

---

# 11. Castle та Camp у Castle Region

Військовий підрозділ може бути:

- у Castle;
- у таборі (Camp) в Castle Region.

Для можливих дій ці стани еквівалентні. Вони відрізняються лише upkeep.

Перехід між Castle і Camp у Castle Region миттєвий і не є Movement.

Перехід у Castle можливий лише якщо Barracks має Capacity для всіх Soldier відповідного Unit.

Стан Castle/Camp задається окремо для кожного Unit навіть усередині однієї Army. Unit в Castle і Unit у Camp тієї самої Castle Region вважаються такими, що знаходяться в одному місці, і можуть бути об'єднані в Army.

Якщо така Army отримує Movement order, вона починає Movement одразу; окремої попередньої команди «вийти з Castle» не потрібно.

Надалі, коли різниця неважлива, термін Camp охоплює обидва варіанти перебування біля home Castle.

---


# 12. Upkeep військ

Soldier і Knight мають регулярний Coin upkeep, незалежний від Food. Конкретні ставки та можливі відмінності між Castle/Camp задаються конфігурацією.

Food consumption є окремою системою. Компенсація Food shortage у Coins завжди додається до звичайного Coin upkeep, а не замінює його.

Для Army у Movement весь її Food-equivalent consumption додатково переводиться в Coins за coins_per_food.

---

# 13. Route, Movement, Dt і фіксація наміру

Player задає Army фізичний Route та кінцеву Region. Локальна мета Camp або Transit визначається самим movement order і після входу в поточну Region не переобчислюється через зміну ownership, Occupation або наявності військ.

Проміжна Region маршруту проходиться як Transit. Якщо поточна Region є кінцевою для цього order, Army рухається до Camp. CombatSituation може pause-ити цей рух, але не змінює початкову локальну мету.

У V1 використовується одна базова константа часу Dt:

- Camp -> entry у сусідню Region: Dt;
- entry у Region -> Camp: Dt;
- базовий звичайний Transit однієї Region: Dt;
- CombatSituation Start -> Battle Start: Dt;
- звичайний Retreat entry -> Camp: Dt;
- Regrouping: Dt.

У майбутньому speed modifier може змінювати тільки звичайний Transit без CombatSituation; інші перелічені інтервали лишаються рівно Dt.

Якщо Player змінює Route під час Transit, зміна стосується тільки майбутньої частини маршруту. Уже зафіксований exit із поточної Region не змінюється. Army не може зупинитися посеред Neutral Transit і перетворити його на Camp; щоб зупинитися там, вона повинна вийти й зайти знову з відповідною метою. Виняток — Transit Army defender, яка явно приєднується до вже Active CombatSituation за правилами розділу 17.

Після половини поточного Transit іншим Player може бути відкритий тільки наступний exit direction, а не весь Route.

## 13.1 Target opponent Movement

При створенні Movement фіксується target_opponent за станом final Region:

- Neutral -> null;
- Owned non-Occupied Player B -> B;
- Owned Occupied Player C -> поточний occupier C.

target_opponent є характеристикою **Movement**, а не Army. Подальша зміна ownership/Occupation final Region автоматично його не змінює.

Player може явно виконати refresh target opponent. Нове значення обчислюється в момент refresh, зберігається як pending і набуває чинності тільки при вході Army в наступну Region. Уже зареєстрована CombatSituation від цього не змінюється.

---

# 14. Transit і Aggressive Transit

Звичайний Transit через Neutral Region не створює територіальної CombatSituation.

При вході чужої Army в Owned non-Occupied Region CombatSituation реєструється і для Camp, і для Transit.

Для Transit такої CombatSituation на Start використовується snapshot target_opponent, зафіксований при Registration:

- якщо formal owner поточної Region == target_opponent -> Aggressive Transit;
- інакше -> NonAggressive Transit.

Occupied Region є винятком: Transit через неї CombatSituation не створює взагалі, Allow Transit не застосовується, Aggressive/NonAggressive classification не визначається.

---

# 15. Allow Transit для Owned non-Occupied Region

Allow Transit є fallback rule тільки для NonAggressive Transit через Owned non-Occupied Region.

На CombatSituation Start, якщо у defender немає жодної Army у Camp/Regrouping або вже entered-for-Camp, battle context не виникає: attacking Transit Army продовжує Route. Defender Transit Army самі по собі не створюють можливості interception.

Якщо Camp/Camp-bound defense context є, defender протягом Dt може один раз вручну обрати:

- Allow — CombatSituation завершується без battle, attacker продовжує Transit;
- Fight — situation доходить до Battle Start.

Ручний вибір immutable. Якщо його немає до Battle Start, використовується поточне Allow Transit: true -> Allow, false -> Fight.

Для Aggressive Transit Allow Transit не застосовується. Якщо defense context є, battle відбувається.

---

# 16. CombatSituation: Registration, Start і FIFO

Одна CombatSituation завжди має рівно одну attacking Army. Дві Army одного Player створюють окремі situations.

CombatSituation реєструється, коли:

- чужа Army входить у Owned non-Occupied Region — Camp або Transit;
- Army входить у Occupied Region з локальною метою Camp і не є current occupier;
- Region стає Occupied, коли Camp-bound Army formal owner уже ввійшла в неї, але запізно для попередньої defense;
- Army у Camp Neutral Region явно атакує конкретного іншого Player, який має active Camp у цій самій Region.

У кожній Region існує одна спільна FIFO-черга **всіх** player-vs-player CombatSituation незалежно від пар Player. Одночасно Active може бути максимум одна.

Якщо черги немає, Start відбувається одразу після Registration. Інакше situation лишається Registered і Start-ує тільки після завершення попередніх. Queued attacker command-locked: Movement attacker pause-иться зі збереженням route/context, Camp attacker фізично лишається в Camp.

Для territorial situation defender Player визначається тільки на Start за актуальним interaction. Для explicit attack у Neutral Region конкретний defender Player фіксується вже на Registration і situation ніколи не retarget-иться на іншого Player.

Якщо на власному Start interaction уже не існує, situation завершується без battle. Queued situation не видаляється наперед лише тому, що її майбутні умови змінилися.

Від Start до Battle Start завжди проходить повний Dt, якщо situation не завершилась достроково без battle.

Попередження про наближення Army та UI-деталі не змінюють цих lifecycle rules.

---

# 17. Defender participation і pre-battle decisions

На CombatSituation Start фіксуються potential defenders defender Player:

- Army у Camp;
- Army у Regrouping;
- Army, яка ввійшла в Region з локальною метою Camp не пізніше Start.

Army, що вже вийшла з Camp, та Army, яка входить після Start, не можуть брати участі в цій CombatSituation.

Окремо defender Army, яка ввійшла для Transit **до Start** і на Start ще не вийшла, стає transit defender candidate. Вона продовжує Transit і не отримує lock автоматично. До Battle Start і до фактичного виходу з Region Player може явно залишити її для defense. Тоді її старий Movement/Route припиняється, Army переходить у звичайний Camp, стає potential defender і отримує combat command-lock.

Potential defender до Battle Start може отримати індивідуальне рішення pre-battle Retreat. Воно **не виконує Retreat одразу** і не змінює persistent Army threshold: тільки локально для цієї CombatSituation підміняє effective Defense Loss Threshold цієї Army на `0`.

За відсутності такого рішення Army використовує свій persistent Defense Loss Threshold. Transit candidates без явного join продовжують Transit.

На Battle Start спочатку перевіряється наявність legal Retreat Region для обох sides. Якщо side не має legal Retreat, її effective threshold примусово стає `100%`; лише після цієї перевірки Army/side, в яких effective threshold лишився `0`, виконують Retreat. Destination визначається side-level за розділом 23.

Potential defenders після Start command-locked для Movement, merge/split, зміни Commander і composition до завершення situation. Combat-specific Retreat/join decisions залишаються доступними там, де це передбачено.

Army defender Player, що входить після Start, не reinforcement. Якщо після завершення поточного battle Region уже Occupied, Camp-bound Army formal owner може створити нову CombatSituation проти occupier; Transit продовжується без CombatSituation.

---

# 18. Combat roles і loss thresholds

Attacker у player-vs-player CombatSituation завжди одна Army. Defender може складатися з кількох Army одного Player.

Attacker має Target Combat Threshold і Incidental Combat Threshold.

Для CombatSituation, зареєстрованої через Movement:

- якщо фактичний defender Player == snapshot target_opponent цієї situation, використовується Target Combat Threshold;
- інакше використовується Incidental Combat Threshold.

Для explicit attack між Camp у Neutral Region завжди використовується Target Combat Threshold, бо це свідома атака на явно обраного Player і Movement target_opponent тут не застосовується.

Кожна defender Army приносить власний persistent Defense Loss Threshold. Pre-battle Retreat decision локально підміняє його на `0` тільки для поточної CombatSituation.

На Battle Start порядок фіксований:

1. для Attacker side і Defender side визначається наявність legal Retreat;
2. якщо side не має legal Retreat, її effective combat threshold примусово стає `100%`; для Defender жодна локальна `0`-підміна тоді не виконує Retreat;
3. тільки після цього всі Army/side з effective threshold `0` і legal Retreat виходять до combat calculation без casualties;
4. якщо Defender Army залишились, спільний Defender threshold стає мінімальним effective threshold серед них;
5. після цього lock-яться strengths і виконується combat calculation.

Отже, звичайний persistent threshold `0` також означає відхід до combat, якщо Retreat можливий. Якщо `0` має Attacker і legal Retreat є, Attacker Retreat-ить і battle не розраховується.

---

# 19. Player-vs-player combat у Neutral Region

Explicit attack у Neutral Region можливий Army у Camp проти конкретного іншого Player, який має active Camp у цій самій Neutral Region. Після Registration актуальність уже зафіксованої interaction на Start перевіряється ширше: defender Player повинен мати Camp/Regrouping presence або Army, що вже entered-for-Camp.

На Registration defender Player фіксується. На Start situation або лишається атакою саме проти нього, або завершується без battle; retarget на третього Player не відбувається.

До defender side застосовуються ті самі правила potential defenders, Transit join, pre-battle Retreat та command-lock, що й у territorial CombatSituation.

Battle Start настає через Dt і situation використовує ту саму загальну FIFO-чергу Region.

Для розрахунку сили і Attacker, і Defender використовують Attack characteristics Soldier. Attacker використовує Target Combat Threshold.

Для Retreat напрямкового обмеження за вектором входу немає: обидві сторони розглядають усі шість сусідніх Region за іншими правилами Retreat.

Camp/Transit війська третіх Player у цей battle не включаються.

---

# 20. Розрахунок сили Unit і Army

Сила Soldier залежить від типу combat:

- у звичайному combat Attacker використовує Attack strength Soldier, Defender — Defense strength Soldier;
- у combat між двома Camp у Neutral Region обидві сторони використовують Attack strength.

Особиста сила Knight однакова в Attack і Defense.

Для кожного Unit спочатку визначається базова сила його Soldier для потрібного режиму і додається особиста базова сила Knight. Потім застосовується Experience coefficient цього Knight:

`UnitStrength = (SoldierStrength + KnightBaseStrength) × KnightCoefficient(experience)`

Для Army з кількох Unit:

`ArmyStrength = Σ UnitStrength_i × CommanderCoefficient(commanderExperience)`

Коефіцієнт Commander-in-Chief застосовується до всієї суми.

Якщо Army складається з одного Unit, той самий Knight одночасно є безпосереднім командиром і Commander-in-Chief; обидва рівні командного впливу застосовуються.

Конкретні функції Experience coefficient є конфігураційними.

---

# 21. Загальна логіка Combat

Combat розраховується аналітично на основі співвідношення початкових бойових сил сторін.

До співвідношення сил застосовується обмежений випадковий Luck Factor. Його розподіл та межі задаються конфігурацією, але випадковість повинна бути симетричною щодо сторін і не перекривати велику стратегічну перевагу.

Після Luck Factor визначаються функції накопичення частки втрат обох сторін. Структурно одна сторона нормалізується до базового темпу, а темп втрат другої визначається монотонною функцією скоригованого співвідношення сил. Конкретна функція належить конфігурації.

Combat завершується, коли одна зі сторін першою досягає свого loss threshold. Вона програє/відступає, інша сторона вважається переможцем.

Якщо обидві сторони досягають відповідного threshold в один і той самий момент, такий результат не приймається: Luck Factor генерується повторно і combat перераховується.

Фактичні casualties визначаються лише після завершення combat.

---


# 22. Об'єднана Defender side

Якщо Defender складається з кількох Army, вони зберігають власну організаційну структуру, але для combat утворюють одну Defender side.

Combat Strength кожної Army спочатку розраховується окремо з її власним Commander-in-Chief coefficient. Після цього сили всіх participating defender Army підсумовуються.

До combat calculation не входять Army, які після перевірки Retreat availability мають effective threshold `0` і виконали Retreat, а також Army, які інакше не є participant за правилами CombatSituation.

Якщо legal Retreat у Defender side є, спільний Defender threshold дорівнює мінімальному effective Defense Loss Threshold Army, що лишилися після zero-threshold Retreat. Якщо legal Retreat немає, спільний threshold примусово дорівнює `100%` і pre-battle Retreat не виконується.

Якщо Defender програє, усі Army, що брали участь, вважаються такими, що програли, і використовують **одну спільну Retreat Region**, вибрану для всієї Defender side. Casualties розподіляються тільки між фактичними participants; Army, що виконали pre-battle Retreat, casualties не отримують.

---


# 23. Retreat destination

Legal Retreat Region визначаються окремо для Attacker side і Defender side, але **не окремо для кожної Army**. Усі Army однієї combat side знаходяться в одній combat Region, мають ту саму combat role та opponent, тому мають один side-level набір legal Retreat Region.

Якщо side не має жодної legal Retreat Region, її loss threshold для цього battle примусово стає максимальним.

## 23.1 Геометричне обмеження

Для Defender розглядаються три сусідні Region на боці, протилежному напрямку входу Attacker.

Для Attacker — три Region у бік, звідки він прийшов.

Для explicit player-vs-player attack між Camp в одній Neutral Region напрямкового обмеження немає: розглядаються всі шість сусідніх Region.

## 23.2 Presence і заборонені Region

Для Retreat blocking local presence враховує Army у Camp, Regrouping та Army, що вже entered-for-Camp. Не враховуються Army, які ще не ввійшли в Region, чистий Transit або Army, що вже leaving-Camp.

Не можна Retreat:

- у чужу Castle Region;
- у свою Region, яку зараз Occupied інший Player;
- у чужу Owned Region, якщо туди вже ввійшла для Camp хоча б одна Army будь-якого Player або там уже є Army у Camp/Regrouping; pure Transit при цьому не блокує Retreat;
- у Neutral Region, де є local presence opponent, якому ця side щойно програла.

Neutral Defense саме по собі Retreat не блокує.

Порожня foreign Owned non-Occupied Region може бути legal Retreat destination. Retreat entry є спеціальною взаємодією і **не реєструє нову CombatSituation**. Після Dt до Camp така Army встановлює Occupation цієї Region.

## 23.3 Пріоритет вибору

Для Defender пріоритет:

1. власні non-Occupied Region;
2. Neutral Region;
3. legal foreign Region.

Серед own Region кожна кандидатна Region оцінюється відносно Castle, до якого приєднана **сама ця Region**: перевага має Region, ближча до свого Castle. Це не home Castle Army і не Castle Commander.

Серед Neutral Region перевага надається ближчій до власної territory; для foreign Region використовується визначений географічний критерій.

Для Attacker найвищий пріоритет має source Region, з якої він увійшов у combat Region, якщо вона legal. Інакше використовуються ті самі групові пріоритети.

Якщо після всіх priority rules кілька Region рівнозначні, одна вибирається випадково.

Вибір виконується один раз для відповідної side. Усі Army цієї side, що Retreat-ять у межах CombatSituation, використовують той самий destination.

# 24. Наслідки Retreat, перемоги та Regrouping

При звичайному player-vs-player Retreat Army одразу вважається такою, що залишила combat Region і ввійшла в обрану сусідню Region. Нової CombatSituation на цьому entry не створюється.

Далі Army протягом Dt рухається локально до Camp. Після Camp arrival вона входить у Regrouping ще на Dt.

Pre-battle Retreat використовує ті самі movement/Regrouping rules, але не завдає combat casualties.

Якщо кілька defender Army Retreat-ять, усі використовують одну Defender retreat Region. Якщо destination — empty foreign Owned non-Occupied Region, після Camp arrival створюється одна Occupation цього Player.

Regrouping є звичайною Camp-presence для Food, Occupation, Annexation, Founding, defense та Camp lifecycle. Army у Regrouping не може ініціювати Movement або Attack і не може Merge/Split. Зміна Commander, threshold та composition дозволяються за звичайними location/combat-lock rules; composition усе одно потребує home Castle.

Якщо виграє Defender, participating defender Army лишаються у Camp.

Якщо виграє Camp-bound Attacker, після battle він завершує перехід у Camp; у foreign Owned Region це створює Occupation. Якщо Attacker мав Transit і виграв, він продовжує свій зафіксований Route без повторного проходження поточної Region.

Поразка від Neutral Defense або City Defense є спеціальним винятком: Army не Retreat-ить у сусідню Region і не має окремого retreat-local Dt; вона лишається в тому самому Camp і одразу починає Regrouping на Dt.

---

# 25. Casualties

Persistent HP у Soldier і Knight немає.

Для розподілу casualties під час одного combat використовується внутрішня умовна величина Casualty Health.

Кожен Soldier має однакову базову Casualty Health. Knight має значно більшу Casualty Health. Конкретні значення задаються конфігурацією і не показуються гравцю.

Загальний budget втрат сторони:

`LossBudget = TotalCasualtyHealth × FinalLossFraction`

`TotalCasualtyHealth` включає Soldier і Knight.

## 25.1 Розподіл між Unit

LossBudget розподіляється між Unit приблизно пропорційно їх Casualty Health, але не строго.

До ваг кожного Unit застосовується невелике випадкове відхилення, після чого ваги нормалізуються так, щоб сумарний LossBudget не змінився.

Це запобігає штучним результатам, коли однакові Unit завжди втрачають точно однакову кількість Soldier.

## 25.2 Розподіл усередині Unit

Усередині Unit budget між Soldier Type також розподіляється приблизно пропорційно з невеликим випадковим відхиленням і наступною нормалізацією.

Правило округлення до цілих Soldier повинно зберігати очікуваний загальний обсяг casualties і використовувати ігрове джерело випадковості.

## 25.3 Knight death

Поки в Unit після розподілу casualties залишається хоча б один Soldier, Knight цього Unit не може загинути.

Лише після загибелі всіх його Soldier залишковий LossBudget може перейти на Knight.

Ймовірність Knight death визначається відношенням отриманої ним умовної шкоди до Knight Casualty Health і окремим mortality coefficient.

Commander-in-Chief використовує менший mortality coefficient, ніж звичайний Knight, але не є невразливим.

Умовна шкода Knight не переноситься між combat.

Якщо Commander-in-Chief гине, це визначається після завершення розподілу всіх casualties. Army **не розпадається**: серед живих Knight цієї Army автоматично новим Commander-in-Chief стає Knight із найбільшим Experience. Це не змінює результат уже розрахованого combat; Army зберігає thresholds і свій подальший Retreat/Camp/Movement/Regrouping context. Якщо живих Knight в Army не лишилося, сама Army припиняє існування. При рівному Experience використовується детермінований tie-breaker.

Всі випадкові рішення combat повинні бути відтворюваними при однаковому повному стані та однаковому random seed.

---


# 26. Neutral Defense

Neutral Defense — абстрактна поточна сила Neutral Region, а не Army.

Вхід Army у Camp Neutral Region сам по собі не запускає battle. Neutral Defense атакується окремою миттєвою дією конкретної Army у Camp; Regrouping Army не може її ініціювати. Ця дія не створює CombatSituation і не входить у player-vs-player FIFO.

Якщо в Region є City, defender strength при атаці Neutral Defense дорівнює:

current Neutral Defense + full City Defense

Для обох складових defender threshold дорівнює 100%. Attacker використовує Target Combat Threshold.

При перемозі Attacker поточна Neutral Defense стає 0; City Defense не змінюється.

При поразці часткові втрати Neutral Defense не зберігаються: після combat вона лишається на тому самому pre-combat current value. City Defense також не змінюється. Attacker лишається у своєму Camp і одразу починає Regrouping на Dt.

Neutral Defense recovery є поступовим. Recovery rate ненульовий тільки коли Region Neutral, current strength нижча за full strength і в Region немає **жодної фізично присутньої Army** будь-якого Player.

Будь-яка Army у Camp, Regrouping або Movement всередині Region pause-ить recovery. Після виходу останньої Army recovery продовжується від уже досягнутого current value, а не починається заново.

Player не може ініціювати Attack Neutral Defense у Region, якщо він є attacker будь-якої unresolved CombatSituation у цій Region або potential defender Active CombatSituation.

---

# 27. Annexation Neutral Region

Annexation progress для Player може йти, якщо одночасно:

- Neutral Defense == 0;
- Player має в Region active Camp-presence; Regrouping рахується так само, як Camp;
- є хоча б одна valid-connected adjacent Owned Region цього Player;
- немає active Founding;
- немає blocking foreign Camp-presence: foreign Camp, Regrouping або entered-for-Camp Army.

Foreign pure Transit не pause-ить Annexation progress, навіть якщо через цей Transit існує CombatSituation.

Поява blocking foreign Camp/Camp-bound presence pause-ить progress, але не скидає його. Якщо власний Camp episode повністю завершується до Annexation, накопичений control progress цього episode втрачається.

Досягнення required control time не виконує Annexation автоматично. Region лише стає ready.

При manual Annexation Player обирає конкретний Castle. Потрібні:

- valid territorial connection до цього Castle;
- вільна Governor's House Capacity;
- у Camp має бути хоча б один Knight, чия Army має Camp-presence і чий home Castle — саме обраний Castle;
- відсутність blocking foreign Camp/Camp-bound presence;
- відсутність active Founding.

Annexation не має окремої одноразової resource cost у V1.

Неважливо, хто саме знищив Neutral Defense; право на control progress визначається поточною presence та connectivity.

---

# 28. Annexation Occupied Region

Current occupier може Annex Occupied Region за тією самою control-progress логікою, але Neutral Defense для цього не потрібна.

Progress потребує власної Camp/Regrouping presence occupier, valid-connected adjacent Owned Region та відсутності blocking foreign Camp/Camp-bound presence. Pure Transit не блокує.

Після накопичення required time Annexation виконується окремою user action з вибором Castle та перевірками connection, Governor Capacity і наявності в Camp Knight з home Castle, що дорівнює обраному Castle.

До завершення Annexation формальний owner лишається owner і продовжує Region upkeep.

Після Annexation ownership переходить occupier, Region приєднується до обраного Castle, а territorial connectivity колишнього owner перераховується. Region, які через остаточну втрату bridge більше не мають шляху до свого Castle, стають Neutral за правилами розділу 3.

---

# 29. City

City є об'єктом Region і має дві економічні характеристики:

- wealth — повний довгостроковий економічний потенціал;
- active_wealth_ratio — активна частка від 0 до 1.

effective_wealth = wealth × active_wealth_ratio.

Коли active_wealth_ratio == 1, Region є Owned non-Occupied і Player не має empty_coins, wealth поступово зростає.

Після успішного Raid wealth не зменшується, а active_wealth_ratio скидається до 0. Поки ratio < 1, wealth не росте.

Recovery active_wealth_ratio до 1 відбувається поступово **незалежно від ownership, Occupation та стану Coins Player**.

Recurring City Coin income і Raid reward базуються на effective_wealth. Neutral або Occupied City не дає recurring income формальному owner.

City має окрему City Defense = f(wealth). Вона залежить від повного wealth, не зменшується після Raid і не залежить від active ratio.

---

# 30. City Raid

City Raid — миттєва локальна Attack-дія конкретної Army у Camp тієї самої Region. Army у Movement або Regrouping не може її ініціювати.

Defender Raid — **тільки City Defense**. Neutral Defense не бере участі.

City Defense використовує fixed raid loss threshold із configuration; Attacker використовує Target Combat Threshold.

У Neutral Region можна просто ввійти в Camp і окремо Raid City без попередньої атаки Neutral Defense.

У чужій Owned Region Army спочатку повинна отримати Camp-presence за звичайними player-vs-player/Occupation rules; після цього Raid є окремою дією.

При успіху reward визначається pre-raid effective_wealth, Coins одразу зараховуються Player, після чого active_wealth_ratio = 0. wealth і City Defense не змінюються. Raid при низькому ratio дозволений і знову скидає ratio до 0.

При поразці Army не Retreat-ить у сусідню Region: вона лишається в тому самому Camp і одразу починає Regrouping на Dt.

Player не може ініціювати City Raid у Region, якщо він є attacker будь-якої unresolved CombatSituation у цій Region або potential defender Active CombatSituation.

---

# 31. Заснування нового Castle

Новий Castle можна заснувати:

- у Neutral Region;
- у власній Annexed non-Castle Region.

Для Neutral Region не потрібні contiguity, Governor Capacity або попередній Annexation progress, але Neutral Defense має бути 0.

Founder — Knight без Soldier, фізично присутній у Camp цієї Region. Regrouping рахується Camp-presence: він не забороняє старт Founding і не pause-ить progress сам по собі.

На Start:

- active Founding у Region не повинно бути;
- foreign blocking Camp-presence не повинна існувати;
- Player сплачує local founding cost із home Castle founder Knight і global cost із Player;
- задається name нового Castle.

Foreign Camp, Regrouping або entered-for-Camp presence після Start pause-ить Founding progress. Pure Transit не pause-ить його, навіть якщо через Transit існує CombatSituation.

Founder повинен залишатися живим, у потрібному Camp і без Soldier увесь process. Якщо він залишає Region/Camp, гине або отримує Soldier, Founding скасовується без refund.

У Neutral Region active Founding і Annexation взаємовиключні.

Якщо під час завершення Founding у Region уже знаходиться foreign Transit Army, Castle все одно створюється, а ця Army має право завершити вже розпочатий Transit. Нові hostile entries у Castle Region після цього заборонені V1.

Після completion Region стає Castle Region нового Castle. Якщо вона раніше належала іншому Castle цього Player, зв'язок із старим Castle припиняється. Створюються Warehouse 1, Granary 1 і Palace 1. Founder змінює home Castle на новий і займає початковий Palace slot; додатковий Knight через цей стартовий slot не генерується. У старому home Castle founder-а звільнений Palace slot запускає звичайний KnightReplacement mechanism так само, як slot після загибелі Knight.

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

У цей документ свідомо не включені системи, яких немає у першій версії: alliances, формальна політична ієрархія, захоплення Castle, terrain movement modifiers, Stable, розширена роль Forge, спеціальні recruitment-speed Building, player-to-player trade та resource transfer, prestige systems, повна intelligence/scouting system, складна supply logistics, persistent HP та інші механіки, перелічені у файлі майбутніх систем.
