# Феодали — Бойова система: детальна специфікація V1

[Основні правила](../01_GAME_RULES_V1.md). Документ містить актуальні уточнення правил V1.

# 16. CombatSituation: Registration, Start і FIFO

Одна CombatSituation завжди має рівно одну attacking Army. Дві Army одного Player створюють окремі situations.

CombatSituation реєструється, коли:

- чужа Army входить у Owned non-Occupied Region з метою Transit — за звичайними правилами Transit;
- Army входить у чужу Owned/Occupied Region з метою Camp (включно з Retreat): CombatSituation реєструється, **лише якщо в Region уже є Army іншого Player у `Camp`/Regrouping або Army іншого Player, яка раніше ввійшла для Camp (`EnteringCamp`)**. За відсутності таких Army нова CombatSituation не реєструється, Army переходить до Camp за звичайний `Dt`;
- Army formal owner входить у власну Region з метою Camp, якщо там уже є чужа Army у `Camp`/Regrouping або чужа Army, яка раніше ввійшла для Camp (`EnteringCamp`), навіть якщо Region ще не Occupied;
- Якщо у Region уже є unresolved CombatSituation між різними Player, кожна наступна Camp-bound Army, яка при вході бачить чужу `Camp` або раніше введену `EnteringCamp` Army, реєструє власну CombatSituation в FIFO, навіть якщо належить Player першого attacker/defender. Defender не фіксується до Start; якщо на Start між цією Army та актуальним противником немає interaction, situation завершується без battle;
- це включає Camp-bound Army формального owner, яка атакує occupier власної Castle Region; Army всередині заблокованого Castle не беруть участі;
- Region стає Occupied, коли Camp-bound Army formal owner уже ввійшла в неї, але запізно для попередньої defense;
- Army у Camp Neutral Region (але не в Regrouping) явно атакує конкретного іншого Player, який має в цій самій Region принаймні одну Army у територіальному стані `Camp`, **включно з Regrouping**. Лише entered-for-Camp без Army у `Camp` недостатньо для ініціації explicit attack.

У кожній Region існує одна спільна FIFO-черга **всіх** player-vs-player CombatSituation незалежно від пар Player. Одночасно Active може бути максимум одна.

Якщо черги немає, Start відбувається одразу після Registration. Інакше situation лишається Registered і Start-ує тільки після завершення попередніх. Queued attacker command-locked: Movement attacker pause-иться зі збереженням route/context, Camp attacker фізично лишається в Camp.

Для territorial situation defender Player визначається тільки на Start за актуальним interaction. Для explicit attack у Neutral Region конкретний defender Player фіксується вже на Registration і situation ніколи не retarget-иться на іншого Player.

Якщо на власному Start interaction уже не існує (наприклад, Region стала Neutral або для explicit attack у Neutral Region зникла presence зафіксованого defender), situation завершується без battle. **Якщо territorial Camp-bound CombatSituation була зареєстрована через наявність чужої Army у `Camp` або `EnteringCamp`, але на Start interaction зникла, вона завершується без battle.** Queued situation не видаляється наперед лише тому, що її майбутні умови змінилися.

Від Start до Battle Start завжди проходить повний Dt, якщо situation не завершилась достроково без battle. На Battle Start Camp-bound Attacker, який переміг, одразу переходить у `Camp`; зміна Occupation/Owned застосовується до Start наступної situation у FIFO. Якщо попередня Camp-bound situation завершилася без battle через зникнення interaction, її attacker продовжує перехід до Camp за звичайними правилами локального руху.

Попередження про наближення Army та UI-деталі не змінюють цих lifecycle rules.

---

# 17. Defender participation і pre-battle decisions

Unit і Army зі станом `BlockedInCastle` не беруть участі в жодній CombatSituation, включно з боєм за звільнення Castle або заміну occupier. Їх не можна атакувати окремо у V1.

На CombatSituation Start фіксуються potential defenders defender Player: усі Army у Camp/Regrouping та всі Army, що вже entered-for-Camp не пізніше Start. Для battle participation entered-for-Camp не є окремою категорією: оскільки entry -> Camp і Start -> Battle Start обидва тривають рівно `Dt`, така Army гарантовано буде у Camp або Regrouping на Battle Start і бере участь на тих самих умовах. Це так само стосується Army у `retreat-local`, яка ввійшла в Region до Start.

Army, що вже вийшла з Camp, та Army, яка входить після Start, не можуть брати участі в цій CombatSituation.

Окремо defender Army, яка ввійшла для Transit **до Start** і на Start ще не вийшла, стає transit defender candidate. Вона продовжує Transit і не отримує lock автоматично. До Battle Start і до фактичного виходу з Region Player може явно залишити її для defense. Тоді її старий Movement/Route припиняється, Army переходить у звичайний Camp, стає potential defender і отримує combat command-lock.

Potential defender до Battle Start може отримати індивідуальне рішення pre-battle Retreat. Воно **не виконує Retreat одразу** і не змінює persistent Army threshold: тільки локально для цієї CombatSituation підміняє effective Defense Loss Threshold цієї Army на `0`. До Battle Start це рішення можна змінювати; чинним є останнє значення. На Battle Start воно lock-иться разом з іншими combat parameters.

Manual pre-battle decision може примусово поставити `0` звичайній Army або до Battle Start скасувати власний попередній manual `0`, але не може перебити forced Regrouping rule. Effective Defense Loss Threshold остаточно визначається на Battle Start за актуальним станом Army: якщо Army на цей момент усе ще у Regrouping, effective threshold = `0` незалежно від manual decision; якщо Regrouping уже завершився, застосовується актуальне manual decision, а за його відсутності — persistent Defense Loss Threshold. Transit candidates без явного join продовжують Transit.

На Battle Start спочатку перевіряється наявність legal Retreat Region для обох sides. Якщо legal Retreat немає, застосовується примусовий threshold `100%`, **крім двох спеціальних випадків**: для Camp-bound атаки Castle Region діє достроковий вихід повністю розміщених у замку Army з effective threshold `0` за §24; для Transit-атаки Castle Region діють звичайні thresholds і нульові pre-battle decisions без вимоги відступити до сусідньої Region за §24. Для інших випадків лише Army/side з effective threshold `0` і допустимим Retreat виходять до battle calculation. Destination визначається side-level за §23.

Potential defenders після Start command-locked **для нових Movement-команд і зміни Route**, merge/split, зміни Commander і composition до завершення situation. **Цей command-lock не зупиняє вже розпочатий `EnteringCamp` / `retreat-local` перехід до Camp**: Army, що ввійшла не пізніше Start, завершує його до Battle Start за правилами вище. **Перемикання режиму окремого Knight «у замку» / «поза замком» під час активної CombatSituation дозволене**, якщо виконуються звичайні вимоги Castle та Barracks Capacity; для battle береться режим на Battle Start. Combat-specific Retreat/join decisions залишаються доступними там, де це передбачено.

Army defender Player, що входить після Start, не reinforcement. Якщо після завершення поточного battle Region уже Occupied, Camp-bound Army formal owner може створити нову CombatSituation проти occupier; Transit продовжується без CombatSituation.

---

# 18. Combat roles і loss thresholds

Attacker у player-vs-player CombatSituation завжди одна Army. Defender може складатися з кількох Army одного Player.

Attacker має Target Combat Threshold і Incidental Combat Threshold. Для Attacker у V1 допустимі значення обох threshold строго більші за `0`; pre-battle Retreat через zero threshold для Attacker не передбачений.

Для CombatSituation, зареєстрованої через Movement:

- якщо фактичний defender Player == snapshot target_opponent цієї situation, використовується Target Combat Threshold;
- інакше використовується Incidental Combat Threshold.

Для explicit attack між Camp у Neutral Region завжди використовується Target Combat Threshold, бо це свідома атака на явно обраного Player і Movement target_opponent тут не застосовується.

Кожна defender Army приносить власний persistent Defense Loss Threshold. Pre-battle Retreat decision локально підміняє його на `0` тільки для поточної CombatSituation.

На Battle Start порядок фіксований:

1. для Attacker side і Defender side визначається наявність legal Retreat;
2. якщо side не має legal Retreat, її effective combat threshold примусово стає `100%`, **крім спеціальних правил Castle Region**: при Camp-bound атаці до примусового порогу із бою без втрат виходять лише Army, всі Knight яких уже мають режим «у замку» і effective threshold `0`; при Transit-атаці не застосовується примусовий поріг лише через відсутність напрямку Retreat, а всі нульові thresholds виключають Army з battle calculation без зміни її розміщення;
3. в інших випадках Army/side з effective threshold `0` та legal Retreat виконують звичайний Retreat без casualties до combat calculation;
4. якщо Defender Army залишились, спільний Defender threshold є мінімальним effective threshold серед них за наявності legal Retreat або при Transit-атаці Castle Region; при Camp-bound атаці Castle Region без legal Retreat він дорівнює `100%` для всіх Army, що залишились;
5. після цього lock-яться strengths і виконується combat calculation. Якщо після дострокового виходу Army захисників не лишилось, Attacker перемагає без числового combat calculation.

Отже, persistent Defense Loss Threshold `0` у Defender означає відхід до combat, якщо Retreat можливий. Attacker threshold `0` у V1 заборонений.

Якщо potential Defender Army від початку відсутні **або** всі potential Defender Army достроково вийшли з battle calculation через threshold `0` / pre-battle Retreat (у тому числі за спеціальним правилом Castle Region), CombatSituation все одно доходить до Battle Start. Формально Attacker перемагає; Defender не отримують combat casualties, а їхня подальша поведінка визначається типом атаки та §24. Числовий combat calculation не проводиться і Battle Experience не нараховується. Attacker з локальною метою `Camp` **одразу на Battle Start переходить у `Camp`**, без додаткового руху чи затримки після перемоги; у foreign Owned Region в цей самий момент створюється Occupation за звичайними правилами. Attacker з локальною метою `Transit` продовжує початковий Transit context.

---

# 19. Player-vs-player combat у Neutral Region

Explicit attack у Neutral Region може ініціювати Army у `Camp`, **яка сама не перебуває в Regrouping**, проти конкретного іншого Player, якщо цей Player має в тій самій Neutral Region принаймні одну Army у `Camp`, **включно з Army у Regrouping**. Regrouping не захищає від атаки, але забороняє ініціювати її. Лише entered-for-Camp без Army у `Camp` недостатньо для ініціації атаки. Після Registration актуальність уже зафіксованої interaction на Start перевіряється ширше: defender Player повинен мати Camp/Regrouping presence або Army, що вже entered-for-Camp.

На Registration defender Player фіксується. На Start situation або лишається атакою саме проти нього, або завершується без battle; retarget на третього Player не відбувається.

Усі Army зафіксованого defender Player, які на Start є у Camp/Regrouping або вже entered-for-Camp, обов'язково входять до potential defenders; для entered-for-Camp це не опціональне приєднання. До defender side застосовуються ті самі правила Transit join, pre-battle Retreat та command-lock, що й у territorial CombatSituation.

Battle Start настає через Dt і situation використовує ту саму загальну FIFO-чергу Region.

Для розрахунку сили і Attacker, і Defender використовують Attack characteristics Soldier. Attacker використовує Target Combat Threshold.

Для Retreat напрямкового обмеження за вектором входу немає: обидві сторони розглядають усі шість сусідніх Region за іншими правилами Retreat.

Camp/Transit війська третіх Player у цей battle не включаються. Інші Army атакуючого Player також не приєднуються автоматично: Attacker є тільки Army, яка ініціювала explicit Attack; для іншої Army потрібна окрема CombatSituation у спільній FIFO-черзі.

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

Якщо обидві сторони досягають відповідного threshold в один і той самий момент, такий результат не приймається: Luck Factor генерується повторно і combat перераховується. При кожній наступній генерації допустимий діапазон відхилення Luck Factor від `1` симетрично розширюється; величина розширення задається конфігурацією. Перерахунок повторюється до результату без нічиєї.

Фактичні casualties визначаються лише після завершення combat.

---


# 22. Об'єднана Defender side

Якщо Defender складається з кількох Army, вони зберігають власну організаційну структуру, але для combat утворюють одну Defender side.

Combat Strength кожної Army спочатку розраховується окремо з її власним Commander-in-Chief coefficient. Після цього сили всіх participating defender Army підсумовуються.

До combat calculation не входять Army, які після перевірки Retreat availability мають effective threshold `0` і виконали Retreat, а також Army, які інакше не є participant за правилами CombatSituation.

Якщо legal Retreat у Defender side є, спільний Defender threshold дорівнює мінімальному effective Defense Loss Threshold Army, що лишилися після zero-threshold Retreat. Для Transit-атаки Castle Region діють ті самі звичайні thresholds і виключення нульових Army незалежно від наявності сусіднього Retreat destination. Якщо legal Retreat немає при Camp-bound атаці Castle Region, Army, повністю розміщені «у замку», з effective threshold `0` виходять із бойового розрахунку без casualties; для решти Defender Army спільний threshold дорівнює `100%`. Якщо legal Retreat немає за інших обставин, загальне правило примусово встановлює `100%` і не дозволяє pre-battle Retreat.

Якщо Defender програє, усі Army, що брали участь, вважаються такими, що програли, і використовують **одну спільну Retreat Region**, вибрану для всієї Defender side. Casualties розподіляються тільки між фактичними participants; Army, що виконали pre-battle Retreat, casualties не отримують. Для casualty allocation не існує окремої квоти на кожну defender Army: усі Unit усіх participating defender Army утворюють один спільний side-level набір, а Army structure впливає на combat strength через Commander, але не на окремий budget втрат.

---


# 23. Retreat destination

Legal Retreat Region визначаються окремо для Attacker side і Defender side, але **не окремо для кожної Army**. Усі Army однієї combat side знаходяться в одній combat Region, мають ту саму combat role та opponent, тому мають один side-level набір legal Retreat Region.

Якщо side не має жодної legal Retreat Region, її loss threshold для цього battle примусово стає максимальним (`100%`), з винятками Castle Region, наведеними у §24. При Camp-bound атаці Castle Region достроково виходять лише повністю розміщені «у замку» Defender Army з effective threshold `0`; для решти поріг `100%`. При Transit-атаці Castle Region відсутність сусіднього Retreat destination не примушує поріг до `100%`. Якщо сторона програє battle, маючи застосований примусовий поріг `100%` через неможливість Retreat, результат — fight-to-destruction: усі Soldier і Knight **Army, що брали участь у бою**, гарантовано гинуть без звичайної Knight mortality randomness. Army, які достроково вийшли з бою без casualties, цим знищенням не зачіпаються.

## 23.1 Геометричне обмеження

Для Defender розглядаються три сусідні Region на боці, протилежному напрямку входу Attacker.

Для Attacker — три Region у бік, звідки він прийшов.

Для explicit player-vs-player attack між Camp в одній Neutral Region напрямкового обмеження немає: розглядаються всі шість сусідніх Region.

## 23.2 Presence і заборонені Region

Для Retreat blocking local presence враховує Army у Camp, Regrouping та Army, що вже entered-for-Camp. Не враховуються Army, які ще не ввійшли в Region, чистий Transit або Army, що вже leaving-Camp.

Не можна Retreat:

- у чужу Castle Region;
- у свою Region, яку зараз Occupied інший Player;
- у Castle, який заблокований через Occupation: звичайний Retreat не дає входу всередину Castle;
- у чужу Owned Region, якщо туди вже ввійшла для Camp хоча б одна Army будь-якого Player або там уже є Army у Camp/Regrouping; pure Transit при цьому не блокує Retreat;
- у Neutral Region, де є local presence opponent, якому ця side щойно програла.

Neutral Defense саме по собі Retreat не блокує.

Порожня foreign Owned non-Occupied Region може бути legal Retreat destination. Retreat entry перевіряє присутність чужих Army у `Camp`/Regrouping або раніше введених `EnteringCamp`: за їх відсутності CombatSituation не реєструється і Retreat Army досягає Camp через `Dt`, створюючи Occupation; за наявності реєструється CombatSituation за загальними правилами FIFO.

## 23.3 Пріоритет вибору

Для Defender пріоритет:

1. власні non-Occupied Region;
2. Neutral Region;
3. legal foreign Region.

Серед own Region кожна кандидатна Region оцінюється відносно Castle, до якого приєднана **сама ця Region**: перевага має Region, ближча до свого Castle. Це не home Castle Army і не Castle Commander.

Серед Neutral Region перевага має candidate з меншою мінімальною стандартною hex-grid distance до будь-якої Owned non-Occupied Region цього Player. Disconnected Owned Region враховується, доки вона формально не втрачена; Occupied Region як опорна точка не враховується.

Серед legal foreign Owned Region перевага має candidate з меншою стандартною hex-grid distance до найближчої Neutral Region: Army намагається якомога швидше залишити чужу territory. Якщо на карті немає жодної Neutral Region, fallback-критерієм є менша мінімальна hex-grid distance до будь-якої власної non-Occupied Region цього Player.

Для Attacker найвищий пріоритет має source Region, з якої він увійшов у combat Region, якщо вона legal. Інакше використовуються ті самі групові пріоритети.

Якщо після всіх priority rules кілька Region рівнозначні, одна вибирається seeded random tie-breaker.

Вибір виконується один раз для відповідної side. Усі Army цієї side, що Retreat-ять у межах CombatSituation, використовують той самий destination.

# 24. Наслідки Retreat, перемоги та Regrouping

При звичайному player-vs-player Retreat Army одразу вважається такою, що залишила combat Region і ввійшла в обрану сусідню Region. **Якщо це чужа Owned Region, на entry застосовується перевірка реєстрації territorial Camp-bound CombatSituation за §16;** для інших destination діють їхні звичайні правила Registration. Retreat у Neutral Region із Camp третього Player дозволений, якщо інші правила Retreat не забороняють destination; така Camp-presence третього Player сама по собі не блокує Retreat.

Далі Army рухається локально до Camp за правилами §13 і §16. Якщо на entry виникла CombatSituation, перехід до Camp підпорядкований її FIFO та результату: лише Army, яка фактично досягла Camp, починає Regrouping на Dt. Якщо після бою Army знову Retreat-ить, застосовується новий Retreat entry, а не Regrouping у попередній Region.

Pre-battle Retreat вважається поразкою відповідного Defender без combat casualties. Він використовує ті самі movement/Regrouping rules, крім особливих правил оборони Castle Region нижче. **Якщо Camp-bound Attacker окупує Castle Region і Defender Army має змішані режими Knight, її pre-battle Retreat спершу виключає всю Army з combat calculation, але не переносить усі її Unit із Region. На наслідках Occupation Unit «у замку» відділяються кожен в окрему заблоковану Army, а Unit «поза замком» залишаються разом у початковій Army та відступають; бойових втрат не має жодна частина.**

Якщо кілька defender Army Retreat-ять, усі використовують одну Defender retreat Region. Якщо destination — empty foreign Owned non-Occupied Region, Army того самого Player можуть увійти послідовно без CombatSituation, доки немає чужої `Camp`/`EnteringCamp` Army. Occupation виникає при першому фактичному переході в Camp. Якщо між входами з'являється чужа Army, кожен наступний Camp-bound entry перевіряється за §16 і за потреби реєструє окрему situation у FIFO.

Regrouping є додатковим обмеженням всієї Army у територіальному стані `Camp`, а не окремим станом. У Castle Region воно виникає лише при Retreat із сусідньої Region; зміна режиму Knight «у замку» / «поза замком» під час Regrouping дозволена. Regrouping є звичайною Camp-presence для Food, Occupation, Annexation, Founding, defense та Camp lifecycle. Army у Regrouping не може ініціювати Movement або Attack, не може Merge/Split, змінювати Commander або persistent combat thresholds. Її effective Defense Loss Threshold під час Regrouping дорівнює `0`. Якщо legal Retreat немає, загальне правило примусово встановлює `100%`, **крім Camp-bound атаки Castle Region** (повністю розміщена «у замку» Army з threshold `0` виходить із бою без втрат) та **Transit-атаки Castle Region** (нульовий threshold діє без потреби відступати до сусідньої Region). Після завершення Regrouping знову використовується збережений persistent threshold. Composition під час Regrouping дозволена лише за звичайних home Castle/location/combat-lock rules.

Якщо виграє Defender, participating defender Army лишаються у Camp.

### Поразка захисника не столичної Castle Region

**Camp-bound атака, legal Retreat відсутній.** На Battle Start, **до combat calculation**, кожна Defender Army перевіряється як ціле. Якщо **всі її Knight вже мають режим «у замку»** і effective Defense Loss Threshold цієї Army дорівнює `0` (у тому числі через pre-battle Retreat або Regrouping), Army достроково виходить із бою без casualties, зберігаючи склад і Commander. Змішана Army, де хоча б один Knight перебуває «поза замком», не відповідає цій умові навіть за threshold `0`. Усі інші Defender Army беруть участь із примусовим threshold `100%`; їхній pre-battle Retreat скасовується. Якщо вони програють battle, відбувається повне знищення всіх їхніх Soldier і Knight — fight-to-destruction. Якщо Attacker перемагає й створює Occupation, Army, які достроково вийшли з бою, залишаються у замку та переходять до `BlockedInCastle`; автоматичного split немає. Якщо Defender перемагає, режими розміщення всіх Knight не змінюються.

**Camp-bound атака, legal Retreat наявний.** Діють звичайні нульові thresholds і pre-battle Retreat; наслідки для Knight «у замку» / «поза замком» при Occupation описано нижче.

Якщо Camp-bound Attacker перемагає і займає Camp Castle Region, створюючи Occupation, а для Defender є legal Retreat, результат визначається **для кожного Unit за режимом розміщення Knight на Battle Start**, **у тому числі для Army, що достроково вийшли із combat calculation через threshold `0` / pre-battle Retreat**:

- Unit, Knight якого перебуває в режимі «у замку», залишається всередині Castle разом із Soldier, що вижили; Army з таких Unit переходить у територіальний стан `BlockedInCastle`. Немає Retreat та Regrouping; це стосується й Unit, який обрав pre-battle Retreat (він не має combat casualties).
- Unit, Knight якого перебуває в режимі «поза замком», здійснює звичайний Retreat у сусідню Region, потім Regrouping.
- Якщо Army містила Knight «у замку» та «поза замком», **тільки Unit Knight «у замку» відділяються, кожний в окрему Army** у `BlockedInCastle`; вони не зливаються. **Усі Unit «поза замком» залишаються разом у початковій Army**, яка відступає без додаткового поділу. Якщо Commander-in-Chief залишився у Castle, Commander відступаючої Army автоматично стає Knight з найбільшим Experience серед її Unit; за рівності Experience кандидат обирається випадково з ігровим random seed (результат відтворюваний).
- Якщо всі Knight Army були в режимі «у замку», Army не розділяється, переходить до `BlockedInCastle` й залишається в замку зі збереженням початкового складу й Commander, без Regrouping.
- Усі власні війська всередині Castle блокуються при виникненні Occupation, навіть якщо вони не брали участі у відповідному battle.

**Transit-атака Castle Region** не використовує спеціальний алгоритм для Camp-bound атаки без legal Retreat. Незалежно від наявності сусіднього Retreat destination діють звичайні effective thresholds, включно з `0` і pre-battle Retreat: такі Army не беруть участі в combat calculation, але не покидають Region. Якщо Attacker перемагає в **транзитній атаці на не столичну Castle Region**, він продовжує Transit; Region не стає Occupied, Castle не блокується. Усі Army захисника залишаються в цій самій Region **без Retreat і без Regrouping**, у територіальному стані `Camp`. Це однаково стосується Army, що брали участь у battle, і Army, які не брали участі через pre-battle Retreat або effective Defense Loss Threshold = `0`: вони просто не беруть участі в бойовому розрахунку й залишаються в Region. Knight, які до бою перебували в режимі «у замку», залишаються «у замку»; Knight «поза замком» залишаються «поза замком». Жодна з цих Army не переходить у Regrouping. Втрати від battle застосовуються тільки до фактичних учасників бою. Це спеціальний виняток із загальних правил Retreat і Regrouping.

### Війська під блокадою Castle

`BlockedInCastle` є п’ятим територіальним станом Army, а не Regrouping. Такі Army та їх Unit не можуть Movement, Attack, перейти в режим «поза замком» або брати участь у будь-якій CombatSituation. Soldier залишаються в Barracks, займаючи звичайну Capacity. Дозволено змінювати склад Unit **лише Knight, для яких заблокований Castle є home Castle**; Knight з іншим home Castle не можуть змінювати склад Unit у цьому Castle. Merge/split **між Army у `BlockedInCastle` того самого Castle** та зміна Commander дозволені як внутрішні дії; це спеціальний виняток із загального правила Merge/split тільки у звичайному `Camp` (§10). До них не можуть приєднуватися Army у `Camp` або ззовні Castle. Зміна Soldier у Unit можлива лише в його home Castle. Нові Knight, створені в заблокованому Castle, також одразу блокуються.

Якщо owner Camp-bound Army атакує occupier і перемагає, Occupation знімається, але заблоковані Castle Army не допомагають їй у battle. Якщо третій Player перемагає occupier і стає новим occupier, облога **не** знімається; Castle Army не вступають у додатковий бій. Якщо всі occupier Army залишили Camp, блокування й облога одразу завершуються; те саме відбувається, коли owner перемагає occupier і відновлює Owned state. **Усі Army у `BlockedInCastle` переходять у `Camp` без Regrouping**, зберігають структуру, Commander та індивідуальні режими Knight «у замку». Після розблокування вони можуть брати участь у наступній обороні.

Якщо виграє Camp-bound Attacker, **у момент завершення battle він одразу отримує стан `Camp` у combat Region**, без додаткового `Dt` або локального руху до Camp. Якщо Region є чужою Owned або Occupied, наслідки перемоги для Occupation / відновлення Owned застосовуються **в цей самий момент**. Лише після застосування результатів battle, переходу переможця в `Camp` та оновлення територіального контролю ця CombatSituation завершується й може початися наступна situation у FIFO-черзі; її defender і interaction визначаються за вже оновленим станом Region. Якщо Attacker мав Transit і виграв, він продовжує свій зафіксований Route без повторного проходження поточної Region.

Поразка від Neutral Defense або City Defense є спеціальним винятком: Army не Retreat-ить у сусідню Region і не має окремого retreat-local Dt; вона лишається в тому самому Camp та одразу починає Regrouping на Dt. Обидва види такого бою відбуваються поза Castle Region: Neutral Defense атакують у Neutral Region, а Castle не можна заснувати в Region із City. У Castle Region Regrouping можливий лише після Retreat із сусідньої Region.

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

До ваг кожного Unit застосовується невелике випадкове відхилення, після чого ваги нормалізуються так, щоб сумарний LossBudget не змінився. Якщо виділений конкретному Unit budget перевищує його максимальну доступну Casualty Health, надлишок не губиться, а перерозподіляється між іншими Unit цієї side; перерозподіл повторюється, доки budget можна застосувати або вся side повністю вичерпала Casualty Health.

Це запобігає штучним результатам, коли однакові Unit завжди втрачають точно однакову кількість Soldier.

## 25.2 Розподіл усередині Unit

Усередині Unit budget між Soldier Type також розподіляється приблизно пропорційно з невеликим випадковим відхиленням і наступною нормалізацією. Якщо Soldier Type вичерпано, його надлишковий budget перерозподіляється між іншими Soldier Type того самого Unit. Після вичерпання всіх Soldier залишок може застосовуватися до Knight; після вичерпання доступної Casualty Health Unit надлишок переходить до інших Unit цієї side.

Правило округлення до цілих Soldier повинно зберігати очікуваний загальний обсяг casualties і використовувати ігрове джерело випадковості.

## 25.3 Knight death

Поки в Unit після розподілу casualties залишається хоча б один Soldier, Knight цього Unit не може загинути.

Лише після загибелі всіх його Soldier залишковий LossBudget може перейти на Knight.

Ймовірність Knight death визначається відношенням отриманої ним умовної шкоди до Knight Casualty Health і окремим mortality coefficient.

Commander-in-Chief використовує менший mortality coefficient, ніж звичайний Knight, але не є невразливим.

Умовна шкода Knight не переноситься між combat.

Якщо Commander-in-Chief гине, це визначається після завершення розподілу всіх casualties. Army **не розпадається**: серед живих Knight цієї Army автоматично новим Commander-in-Chief стає Knight із найбільшим Experience. Це не змінює результат уже розрахованого combat; Army зберігає thresholds і свій подальший Retreat/Camp/Movement/Regrouping context. Якщо живих Knight в Army не лишилося, сама Army припиняє існування. Оскільки кожний Unit має рівно одного Knight, який не може загинути за наявності живих Soldier у своєму Unit, Army без живих Knight не може мати живих Soldier. При рівному Experience новий Commander обирається **випадково серед Knight із найбільшим Experience** з ігровим random seed, щоб результат був відтворюваним; це саме правило діє для відступаючої частини змішаної Army після окупації Castle.

Всі випадкові рішення combat повинні бути відтворюваними при однаковому повному стані та однаковому random seed.

---


# 30. City Raid

City Raid — миттєва локальна Attack-дія конкретної Army у Camp тієї самої Region. Army у Movement або Regrouping не може її ініціювати. Player не може Raid-ити City у Region, formal owner якої — цей самий Player.

Defender Raid — **тільки City Defense**. Neutral Defense не бере участі.

City Defense використовує fixed raid loss threshold із configuration; Attacker використовує Target Combat Threshold.

У Neutral Region можна просто ввійти в Camp і окремо Raid City без попередньої атаки Neutral Defense.

У чужій Owned Region Army спочатку повинна отримати Camp-presence за звичайними player-vs-player/Occupation rules; після цього Raid є окремою дією.

При успіху reward визначається pre-raid effective_wealth, Coins одразу зараховуються Player, після чого active_wealth_ratio = 0. wealth і City Defense не змінюються. Raid при низькому ratio дозволений і знову скидає ratio до 0.

При поразці Army не Retreat-ить у сусідню Region: вона лишається в тому самому Camp. Після поразки одразу починається Regrouping на Dt. City Raid можливий лише в Region з City, а заснування Castle в такій Region заборонено, тому City Raid у Castle Region неможливий.

Player не може ініціювати City Raid у Region, якщо він є attacker будь-якої unresolved CombatSituation у цій Region або potential defender Active CombatSituation.

---

