# Феодали — Армії та лицарі: детальна специфікація V1

[Основні правила](../01_GAME_RULES_V1.md). Документ містить актуальні уточнення правил V1.

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

Passive Experience нараховується всім існуючим живим Knight безперервно незалежно від `Camp / Transit / LeavingCamp / EnteringCamp / BlockedInCastle`, Regrouping та combat lock. Ready unnamed Knight ще не є Knight entity і Experience не накопичує; dead Knight також не накопичує Experience.

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

Територіальний стан задається для всієї Army і має рівно п’ять значень: `Camp`, `Transit`, `LeavingCamp` (вийшла з Camp і рухається до межі Region), `EnteringCamp` (увійшла в Region і рухається до Camp), `BlockedInCastle` (заблокована в Castle через Occupation). `Regrouping` і command-lock CombatSituation — не територіальні стани, а додаткові обмеження Army у `Camp`. `Regrouping` застосовується до всієї Army; у Castle Region можливий лише після Retreat із сусідньої Region.

Gameplay Unit складається рівно з одного Knight і нуля або більше Soldier. Knight без Soldier є повноцінним Unit.

Unit має home Castle, який збігається з home Castle його Knight. Внутрішній склад Unit можна змінювати лише у home Castle за звичайних умов Barracks.

Кілька Unit можуть бути об'єднані в Army. Army має одного Commander-in-Chief, вибраного серед Knight цієї Army. Army не містить вкладених Army; при merge попередня Army-структура не зберігається.

У V1 немає жорсткого ліміту Soldier у Unit або Unit в Army.

Зовнішні Merge і split дозволені тільки Army у звичайному `Camp` одного Player і одного CampInRegion, якщо вони не command-locked CombatSituation. **Виняток — внутрішні Merge і split Army у `BlockedInCastle` одного Castle**, які дозволені за §24 і не створюють взаємодій із військами поза замком. Regrouping merge/split забороняє, але не перешкоджає перемиканню індивідуального режиму «у замку» / «поза замком». Таке перемикання дозволене і під час активної CombatSituation, попри command-lock, якщо виконані звичайні умови Barracks; бойовий режим фіксується на Battle Start. Commander можна змінювати у Camp або під час звичайного Movement, якщо Army не command-locked; під час Regrouping зміна Commander заборонена.

Зміна складу Soldier виконується тільки через Castle reserve <-> Knight у home Castle і також не допускається для command-locked Army. Перебування Knight у чужому для нього home Castle, навіть якщо цей Castle належить тому самому Player і Knight розміщений «у замку», не дозволяє змінювати Soldier його Unit.

Кожна Army має три persistent loss threshold:

- Defense Loss Threshold;
- Target Combat Threshold;
- Incidental Combat Threshold.

Persistent thresholds можна змінювати під час Camp або Movement, доки Army не command-locked CombatSituation. Під час Regrouping persistent thresholds не змінюються: effective Defense Loss Threshold Army примусово дорівнює `0`, тому при атаці вона Retreat-ить, якщо є legal Retreat; якщо legal Retreat немає, зазвичай effective threshold стає `100%`, крім Camp-bound атаки Castle Region (повністю розміщена «у замку» Army з effective threshold `0` достроково виходить із бою) та Transit-атаки Castle Region (звичайні thresholds діють без сусіднього Retreat destination). Після завершення Regrouping знову діє збережений persistent threshold, який тоді можна змінити. Після Registration attacker уже locked; potential defender отримує lock на CombatSituation Start. Значення конкретного battle остаточно фіксуються на Battle Start.

При merge thresholds нової Army задаються явно. Після split нові Army отримують поточні thresholds вихідної Army, доки Player не змінить їх.

---

# 11. Режими розміщення у Castle Region

Кожен Knight разом зі своїми Soldier у Castle Region має один із двох режимів розміщення: **«у замку»** або **«поза замком»**. Це не територіальні стани Army: до блокування Army перебуває в територіальному стані `Camp` незалежно від режимів її Knight. Режим визначається індивідуально для кожного Knight навіть усередині однієї Army; змішані режими не заважають об'єднанню Unit в одну Army.

Knight може перейти «у замку» в **будь-якому Castle власного Player**, незалежно від його home Castle; у Castle іншого Player — не може. Для входу всіх Soldier його Unit потрібна вільна Capacity Barracks поточного Castle. Knight без Soldier може ввійти навіть за повної Barracks. Перемикання «у замку» / «поза замком» миттєве, не є Movement і **дозволене під час Regrouping та активної CombatSituation, навіть за command-lock**. Для Battle використовується індивідуальний режим кожного Knight на Battle Start; вимоги Barracks Capacity залишаються чинними.

При Occupation Castle Region Knight у режимі «у замку» залишаються в Castle; Army, що складається лише з таких Knight, цілком переходить до територіального стану `BlockedInCastle`, **без розділення, збереженням складу та Commander**. Якщо Army містить також Knight «поза замком», лише Unit Knight «у замку» відділяються, кожен в окрему заблоковану Army; Unit «поза замком» **залишаються разом у початковій Army** та Retreat-ять. Якщо Commander відступаючої Army лишився в Castle, Commander призначається автоматично серед Knight з найбільшим Experience; при рівності — випадково з відтворюваним random seed. Те саме розділення при Occupation застосовується і до змішаної Army, яка достроково вийшла з battle calculation через threshold `0` / pre-battle Retreat; обидві частини не мають combat casualties. `BlockedInCastle` Army не бере участі в зовнішніх взаємодіях, не може Movement/Attack або змінити режим на «поза замком», але допускає внутрішні зміни Soldier (лише у home Castle), Army та Commander за спеціальними правилами блокування. Заміна occupier блокування не знімає; завершення Occupation (у тому числі відвоювання owner) переводить заблоковані Army з `BlockedInCastle` у `Camp` **без Regrouping**, зі збереженням складу, Commander і режимів «у замку».

При Movement order для звичайної Army у `Camp` усі Unit починають Movement синхронно; Knight «у замку» миттєво виходять із Castle, звільняючи Barracks Capacity, без додаткового Dt чи окремої команди.

---


