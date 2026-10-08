# Феодали — Замки та економіка: детальна специфікація V1

[Основні правила](../01_GAME_RULES_V1.md). Усі правила нижче збережені без змін із початкового документа.

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

Coins, Gold та Silver не мають складської місткості. Проте production Gold і Silver проходить економічну перевірку через Castle, до якого Region приєднана, до зарахування на глобальний баланс Player. Глобальне зберігання не обходить відсутність територіального зв'язку, Occupation або блокування Castle.

---


# 5. ResourceSite і видобуток

Region може містити кілька природних ResourceSite одного ресурсу.

ResourceSite не обов'язково існують як окремі іменовані об'єкти. Для кожного ресурсу в Region достатньо зберігати кількість ResourceSite кожного рівня.

Природна кількість ResourceSite є властивістю Region. Їх рівень розвитку може змінювати власник за правилами Upgrade.

Для одного типу ресурсу ResourceSite розвиваються пошарово. Гравець може покращувати будь-яку кількість ResourceSite, що знаходяться на поточному мінімальному рівні. Не можна починати наступний рівень для частини ResourceSite, доки всі ResourceSite цього ресурсу в Region не досягли попереднього рівня.

Базове виробництво визначається без distance penalty:

LocalProduction = Σ ResourceSiteOutput(level_i)

Для Wood, Stone та Iron у Owned non-Occupied Region з чинною територіальною зв'язністю та **незаблокованим Castle** позитивне виробництво надходить до Castle з коефіцієнтом відстані:

CastleFlow = LocalProduction × DistanceEfficiency(distance)

Для Gold і Silver distance coefficient не застосовується. Їх production зараховується до глобального балансу Player через відповідний Castle тільки якщо Region є Owned, не Occupied, має чинну територіальну зв'язність і **Castle не заблокований**. Джерела з Occupied або disconnected Region не дають доходу.

Occupied або disconnected Region не передає ці ресурси формальному owner.

Розвиток ResourceSite має власну вартість і час, що задаються конфігурацією.

Player не керує окремими іменованими Site: для одного resource type і поточного level він задає кількість Site, які треба покращити. Один ResourceSiteUpgrade резервує вибрану кількість Site одразу; зарезервовані Site недоступні іншим паралельним order. Кілька order одного level можуть працювати одночасно, якщо лишилися незарезервовані eligible Site.

Всі Site одного order покращуються паралельно й завершуються одночасно. Тривалість переходу level -> level+1 не залежить від `quantity`; cost дорівнює per-site cost × quantity і повністю списується upfront при старті. Wood / Stone / Iron / Food беруться з Castle, до якого Region приєднана на момент Start; Coins / Gold / Silver — з current owner Player. Mixed cost перевіряється і списується атомарно. Подальша зміна owner, Castle association або neutralization не переносить і не повертає вже сплачену cost. Під час upgrade Site продовжує виробляти як старий level до completion.

Layered rule перевіряє фактично досягнуті level: наступний layer не можна почати, доки всі Site цього resource type реально не завершили попередній level. Active upgrade/reservation не рахується як уже завершений level.

Новий ResourceSiteUpgrade можна почати тільки у власній, non-Occupied і territorially connected Region. Чужа Army у pure Transit цьому не заважає. Після старту upgrade ніколи не pause-иться й не скасовується через Occupation, disconnection, перехід Region у Neutral, зміну owner або Castle association; він завершується в цій Region і підвищує рівні Site для того, хто володітиме Region на той момент. Manual cancellation у V1 немає, refund немає.

---


# 6. Food та населення

Будівлі Castle створюють Population відповідно до конфігурації. Population у V1 не є окремою складною системою і впливає насамперед на постійне споживання Food у Castle.

Один Soldier незалежно від Type споживає однакову кількість Food. Один Knight також рахується як одна food-consumption unit.

Food shortage не замінює звичайний Coin upkeep військ: компенсація нестачі Food у Coins додається поверх нього. Для всієї гри використовується один конфігураційний курс coins_per_food.

## 6.1 Army у Camp звичайної Region

Army у територіальному стані `Camp`, зокрема під обмеженням Regrouping, споживають локальне Food production Region.

Якщо Food достатньо, локальне production покриває весь Camp consumption. Якщо Food недостатньо, доступна кількість ділиться між Player пропорційно сумарному Food consumption їхніх Camp-військ у цій Region. Жоден Player або Army не має пріоритету.

Непокритий дефіцит кожного Player компенсується Coins за coins_per_food.

Для звичайної Owned non-Occupied Region з чинною територіальною зв'язністю та **незаблокованим Castle** тільки **позитивний** залишок Food після Camp consumption передається до Castle з DistanceEfficiency(distance). Castle не постачає Food назад у звичайні Region. Негативний локальний balance ніколи не перетворюється на demand до Castle.

У Neutral, foreign, Occupied або disconnected Region діє та сама локальна Camp-consumption логіка, але позитивний залишок не надходить до Castle формального owner.

## 6.2 Castle Region

Castle Region є спеціальним випадком.

Food consumption Army у Camp/Regrouping Castle Region не віднімається від локального production самої Region. Уся позитивна Food production Castle Region надходить у Castle з coefficient 1 **лише якщо Castle не заблокований**. Під час Occupation навіть Food production самої Castle Region не надходить у Castle.

Food consumption власних Army у Camp/Regrouping Castle Region віднімається на рівні Food balance Castle разом із non-military consumption, доки Region не окупована. Якщо Castle заблокований, його Building/Population та Unit всередині Castle продовжують споживати Food із Castle stocks; Army occupier у Camp не споживають Food зі складів Castle, а забезпечують власне споживання за локальними Camp rules. **Army формального owner у Camp окупованої Castle Region поза Castle** (можлива під час черги CombatSituation, коли кілька Army owner увійшли атакувати occupier) також споживає Food **за звичайними локальними правилами для кількох Army у Region без Castle (§6.1)**. Вона не використовує Food stocks Castle і бере участь у пропорційному розподілі місцевого Food production між Player разом з Army occupier.

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

Вартість і час Construction/Upgrade є конфігураційними. Повна вартість списується upfront при старті; manual cancellation і refund у V1 немає. Після старту BuildingUpgrade не pause-иться через зміну інших умов.

Prerequisites задаються конфігурацією як набір мінімальних level інших Building; усі умови мають виконуватися одночасно. У V1 prerequisites перевіряються тільки при construction `0 -> 1`; подальші level тієї самої Building їх повторно не перевіряють. Building не демонтуються і їх level не зменшуються.

`empty_food` або `empty_coins` самі по собі не є окремим blocker BuildingUpgrade: старт можливий, якщо фактичних ресурсів достатньо для повної upfront cost.

## 7.1 Warehouse

**Економіка заблокованого Castle.** За Occupation Castle Region надходження Wood, Stone, Iron, Food, Gold і Silver із **усіх** прив'язаних Region (включно з Castle Region) припиняються. Gold/Silver не зараховуються навіть на глобальні Player balances. Монети від усіх City, прив'язаних до заблокованого Castle, не надходять. Однак Coin-producing Building самого Castle продовжують працювати (включно з Bank effect). Coin upkeep та інші витрати не змінюються. Building/Upgrade дозволені за наявності upfront resources; Recruitment дозволений за звичайних вимог Food/Coins і Barracks Capacity; створення Knight через Palace дозволене, але вони залишаються BlockedInCastle. Нові Region не можна Annex до заблокованого Castle, навіть якщо територіальний зв'язок наявний. Обмеження завершуються одночасно зі зняттям Occupation.

## 7.1 Warehouse

Зберігає Wood, Stone та Iron. Його Capacity застосовується окремо до кожного з них.

## 7.2 Granary

Зберігає Food.

## 7.3 Palace

Палац (Palace) визначає кількість доступних місць для Knight у конкретному Castle.

Кількість Knight slots структурно дорівнює Palace level. Кожне збільшення Palace level одразу відкриває одне нове місце. Для нового slot немає окремого replacement delay, але Knight фактично не існує, доки Player не надасть йому ім'я.

Якщо Knight гине, Palace level не зменшується. Для звільненого slot одразу стартує `KnightReplacement` timing. Якщо гине кілька Knight, replacement timers обробляються послідовно однією чергою. Коли timer конкретного replacement завершився, він додає одного готового Knight у спільну FIFO-чергу Knight, що чекають ім'я, і більше не блокує запуск timer наступного загиблого Knight.

Palace upgrade і завершений KnightReplacement додають однаковий елемент у цю спільну чергу очікування імені. До отримання імені такі готові Knight нічим не відрізняються. Кожний ready unnamed entry уже резервує один доступний Palace slot у сенсі Capacity, хоча Knight entity ще не існує; відкладання імені не створює додаткової вільної Capacity. Player дає їм імена строго по черзі; лише в цей момент відповідний Knight створюється.

Новий Knight після отримання імені створюється у своєму home Castle з 0 Soldier і територіальний стан Army `Camp` і режим розміщення Knight «у замку». Одночасно створюється окрема Army з одного його Unit, де цей Knight є Commander-in-Chief. У V1 кожний active Knight завжди належить рівно одній Army; окремого стану active Knight без Army немає.

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

Barracks має Capacity, що вимірюється в Soldier: один Soldier будь-якого Type займає одну одиницю Capacity. У ній знаходяться як reserve Soldier, так і Soldier сформованих Unit, чиї Knight перебувають у режимі «у замку» в цьому Castle. Сам Knight Capacity не займає.

Knight разом зі своїм Unit може перейти з режиму «поза замком» у режим «у замку» в будь-якому власному Castle Player (не обов'язково home Castle), якщо Barracks цього Castle має достатньо вільної Capacity для всіх його Soldier. У чужому Castle такий перехід заборонений. Частковий вхід Unit не допускається. Knight з 0 Soldier може перейти «у замку» навіть при повній Barracks. Перемикання режиму дозволене й під час Regrouping.

Soldier Unit у режимі «поза замком» не займають Barracks Capacity. Якщо Barracks заповнена, Recruitment pause до появи вільного місця.

## 7.7 Coin-producing Buildings

У Castle можуть існувати Building, що створюють Coin income. Конкретний набір таких Building і функції доходу задаються конфігурацією.

Market може існувати як економічна Building і приносити Coins навіть якщо повноцінна торгівля між гравцями ще не реалізована.

## 7.8 Bank

Банк (Bank) множить сумарний eligible Coin income інших Building.

Структурна формула:

`FinalIncome = Σ EligibleBuildingIncome_i × BankMultiplier(level)`

Конкретна функція `BankMultiplier` належить конфігурації.

---

# 12. Upkeep військ

Soldier і Knight мають регулярний Coin upkeep, незалежний від Food. Reserve Soldier у Barracks також мають звичайний Coin upkeep відповідно до Soldier Type і починають його сплачувати одразу після завершення Recruitment. Конкретні ставки та можливі відмінності між Castle/Camp задаються конфігурацією.

Food consumption є окремою системою. Reserve Soldier споживають Food саме Castle, де вони зберігаються. Компенсація Food shortage у Coins завжди додається до звичайного Coin upkeep, а не замінює його.

Для Army у Movement весь її Food-equivalent consumption додатково переводиться в Coins за coins_per_food.

Castle-level Food production/flows та Food consumption Population, reserve Soldier і Army у Castle Region підсумовуються в одному Food balance Castle. Якщо Food бракує, загальний непокритий дефіцит компенсується Coins один раз, без пріоритету окремих груп споживачів і без додаткових penalties.

Coin upkeep нараховується безперервно в Game Time за поточним станом/location Soldier або Knight: до переходу діє попередня ставка, після переходу — нова. Минулі нарахування не перераховуються.

---

