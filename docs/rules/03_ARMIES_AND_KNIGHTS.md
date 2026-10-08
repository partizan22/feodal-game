# Феодали — Армії та лицарі: детальна специфікація V1

[Основні правила](../01_GAME_RULES_V1.md). Усі правила нижче збережені без змін із початкового документа.

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

Experience зростає після боїв залежно від масштабу противника та повільно з часом. Battle Experience отримують тільки живі після battle Knight, які реально входили до combat calculation. Knight, що виконали pre-battle Retreat до calculation, XP не отримують; загиблі в цьому battle Knight також не отримують XP. Базовий XP participating survivor визначається функцією від pre-combat strength противника та, за потреби, власної side і не залежить від фактичних втрат конкретного Unit. Commander-in-Chief отримує додатковий Experience bonus за статус Commander.

Passive Experience нараховується всім існуючим живим Knight безперервно незалежно від `Castle / Camp / Movement / Regrouping` та combat lock. Ready unnamed Knight ще не є Knight entity і Experience не накопичує; dead Knight також не накопичує Experience.

Battle Experience нараховується після будь-якого фактично виконаного combat calculation: player-vs-player battle, Attack Neutral Defense і City Raid. Для Neutral Defense/City Defense opponent scale визначається їх pre-combat defender strength; eligibility surviving Knight і Commander bonus лишаються такими самими.

Конкретні функції battle/passive Experience, Commander bonus і перетворення Experience у коефіцієнт задаються конфігурацією.

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

Гравець обирає Soldier Type і кількість. Повна вартість усього order (`per-soldier cost × quantity`) списується upfront при постановці замовлення; refund немає.

Кожний Castle має одну FIFO Recruitment Queue. Новий order завжди додається в кінець, існуючі order не можна reorder-ити або змінювати priority; сусідні order одного Type не зобов'язані об'єднуватися.

Soldier рекрутуються по одному. Order `quantity = N` означає N послідовних повних recruitment cycles; кожний завершений cycle створює рівно одного Soldier і зменшує `remaining_quantity` на 1. Отже без pause повний час order дорівнює N × per-Soldier recruitment time.

Якщо Barracks не має хоча б одного вільного місця, progress поточного Soldier pause-иться негайно й відновлюється з того самого значення після появи місця. Якщо progress уже досяг completion, але місця немає, він залишається ready і Soldier створюється одразу після появи Capacity. Звільнене місце може бути знову зайняте іншою дією до completion, тоді Recruitment знову pause-иться.

Новий order не можна додати, якщо Barracks повна, Castle має `empty_food` або Player має `empty_coins`. Водночас не потрібно мати Capacity для всієї quantity: за наявності хоча б одного вільного місця можна додати order будь-якого додатного розміру, якщо Player має всю upfront cost. Штучного max queue length або max quantity у V1 немає.

Active Recruitment також pause-иться при `empty_food` або `empty_coins` і автоматично продовжується після зникнення blocker. Manual cancellation незавершеного Recruitment у V1 немає.

---


# 10. Unit та Army

Gameplay Unit складається рівно з одного Knight і нуля або більше Soldier. Knight без Soldier є повноцінним Unit.

Unit має home Castle, який збігається з home Castle його Knight. Внутрішній склад Unit можна змінювати лише у home Castle за звичайних умов Barracks.

Кілька Unit можуть бути об'єднані в Army. Army має одного Commander-in-Chief, вибраного серед Knight цієї Army. Army не містить вкладених Army; при merge попередня Army-структура не зберігається.

У V1 немає жорсткого ліміту Soldier у Unit або Unit в Army.

Merge і split дозволені тільки Army у звичайному Camp одного Player і одного CampInRegion, якщо вони не command-locked CombatSituation. Regrouping merge/split забороняє. Commander можна змінювати у Camp або під час звичайного Movement, якщо Army не command-locked; під час Regrouping зміна Commander заборонена.

Зміна складу Soldier виконується тільки через Castle reserve <-> Knight у home Castle і також не допускається для command-locked Army.

Кожна Army має три persistent loss threshold:

- Defense Loss Threshold;
- Target Combat Threshold;
- Incidental Combat Threshold.

Persistent thresholds можна змінювати під час Camp або Movement, доки Army не command-locked CombatSituation. Під час Regrouping persistent thresholds не змінюються: effective Defense Loss Threshold Army примусово дорівнює `0`, тому при атаці вона Retreat-ить, якщо є legal Retreat; якщо legal Retreat немає, загальне правило примусово робить effective threshold `100%`. Після завершення Regrouping знову діє збережений persistent threshold, який тоді можна змінити. Після Registration attacker уже locked; potential defender отримує lock на CombatSituation Start. Значення конкретного battle остаточно фіксуються на Battle Start.

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

Під час Occupation Castle Region власні Unit всередині Castle набувають стану `BlockedInCastle`. Вони не можуть рухатись, атакувати, переходити до Camp або брати участь у CombatSituation, але можуть змінювати склад Soldier, склад Army та Commander за звичайними правилами home Castle і Barracks. Якщо Occupation завершується, блокування знімається. За зміни occupier — не знімається. Звичайний Retreat інших Army не допускає входу всередину заблокованого Castle.

Якщо така Army отримує Movement order, усі її Unit починають Movement синхронно й одразу; Unit, які перебували в Castle, миттєво виходять із нього та звільняють відповідну Barracks Capacity. Окремого Dt, попередньої команди «вийти з Castle» або додаткової перевірки Capacity для виходу немає.

Надалі, коли різниця неважлива, термін Camp охоплює обидва варіанти перебування біля home Castle.

---


