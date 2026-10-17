# Backend models — combat

Player-vs-player CombatSituation and combat lifecycle.

Загальні conventions, system-computed semantics та спільні припущення див. у [MODELS.md](MODELS.md).

---

## 7. `CombatSituation`

`CombatSituation` у цій секції — одна модель для player-vs-player БС та миттєвих атак Neutral Defense/City з registration/start/active lifecycle; відмінності між registration reasons реалізуються умовною domain logic всередині цієї моделі. Окремі inherited submodels поки не вводяться. Attack на Neutral Defense та City Raid — миттєві CombatSituation (Registration, Start, resolve в одному game event); вони не входять у PvP FIFO й не чекають `Dt`.

Одна CombatSituation завжди має рівно одну attacking Army. Дві Army одного Player, які послідовно створюють умови атаки в одній Region, реєструють дві окремі CombatSituation; вони не об'єднуються в одну attacking side і не є reinforcement одна одній.

### Прямі характеристики

- `region`.
- `attacker` — єдина attacking Army; фіксується на Registration.
- `registration_reason` — Camp-bound entry у чужу Owned/Occupied або власну Occupied Region; Camp-bound entry у Neutral Region за наявності чужої `Camp`/Regrouping/`EnteringCamp` presence; допустимий Transit interaction у Owned Region; instant Neutral Defense attack / City Raid. Transit у Neutral Region БС не створює.
- `attacker_entry_from_region` — snapshot `Movement.entry_from_region` на Registration для кожної PvP БС, створеної при вході в Region, включно з Retreat і queued situation; використовується для напрямків Retreat.
- `status` — `Registered / Active / Resolved`.
- `start_time` — встановлюється тільки при CombatSituation Start; `null` для queued Registered.
- `defender_player` — визначається лише на Start за актуальним ownership/occupation або присутністю чужих Camp/Regrouping/`EnteringCamp` Army у Neutral Region.
- `defenders[]` — `null/empty` до Start; на Start фіксуються доступні Camp/Regrouping/entered-for-Camp Army defender Player. Army, яка сама є attacker іншої unresolved CombatSituation (Active або queued), виключається.
- `defenders_retreat_decisions` — рішення Retreat для окремих defender Army. Воно не виконує Retreat одразу, а локально для цієї CombatSituation підміняє effective `defense_loss_threshold` відповідної Army на `0`. Persistent `Army.defense_loss_threshold` не змінюється. Допустимість і destination Retreat визначаються на рівні всієї defender side.
- `attacker_retreat_region` — side-level Retreat destination attacker side або `null`, якщо ще не потрібна/не вибрана.
- `defender_retreat_region` — side-level Retreat destination defender side або `null`, якщо ще не потрібна/не вибрана. Якщо частина defender Army виконує pre-battle Retreat, Region вибирається один раз для defender side; якщо решта defender Army пізніше також мають Retreat у межах цієї CombatSituation, використовується те саме значення.
- `transit_decision` — `null / allow / fight` для non-aggressive Transit; після ручного вибору не змінюється.
- `attacker_destination_mode` — `Camp | Transit`; фіксується за початковою метою Movement на Registration і не змінюється під час FIFO.
- `target_opponent` — snapshot `Movement.target_opponent` на Registration; у Neutral Region може бути `null`, а актуальний defender визначається на Start за присутністю.
- `transit_mode` — `null` до CombatSituation Start; для Transit interaction у Owned non-Occupied Region на Start один раз визначається за зафіксованим `target_opponent`: Aggressive, якщо Region формально належить `target_opponent`, інакше NonAggressive. Transit через Occupied Region реєструє БС, але `transit_mode` для нього не визначається.
- `locked_combat_parameters` — остаточно фіксуються на Battle Start.

### Динамічні характеристики

- `battle_start_progress` — рахується тільки для Active CombatSituation; від Start до Battle Start завжди `Dt`.

### Обчислювальні характеристики

- `is_queued` — `status == Registered`; situation без попередників одразу переходить у `Active`, тому стабільний `Registered` стан означає очікування у FIFO-черзі.
- `is_active` — `status == Active`.
- `battle_start_progress_rate` / `required_battle_start_progress` — дають рівно `Dt` від Start до Battle Start.
- `is_battle_valid` — чи після pre-battle resolution лишилися умови реального battle.
- `attacker_loss_threshold` — target threshold, якщо фактичний `defender_player == target_opponent`, і incidental threshold інакше (включно з Neutral Camp-bound entry при `target_opponent == null`). У V1 пороги > 0; без legal Retreat effective threshold `1.0`.
- `defender_combat_threshold` — на Battle Start для кожної defender Army береться effective threshold: `0`, якщо для неї встановлено pre-battle Retreat decision, інакше її persistent `defense_loss_threshold`. Спочатку перевіряється side-level Retreat availability. Якщо defender side не має legal Retreat Region, її combat threshold примусово стає `1.0` і жодна Army не виконує pre-battle Retreat. Якщо legal Retreat є, Army з effective threshold `0` спочатку виконують Retreat без casualties; після їх виключення спільний `defender_combat_threshold` дорівнює мінімальному effective threshold серед Army, що лишилися. Якщо жодної defender Army не лишилося, battle не відбувається.
- `defender_loss_threshold` — чинне значення `defender_combat_threshold`; один спільний threshold для всієї defense side, що lock-иться на Battle Start.
- `attacker_strength` — `attacker.attack_strength` для будь-якого player-vs-player battle.
- `defender_strength` — сума `defense_strength` participating defender Army для PvP defense; для instant Neutral Defense/City Raid використовується абстрактна оборона.

### Registration

Класифікація V1:

- **Зайнята Region**: при допустимому вході Army у чужу Owned/Occupied Region або власну Occupied Region БС реєструється незалежно від наявності defenders. Вхід owner у власну non-Occupied Region — виняток; спеціальні правила Transit залишаються чинними. На Start фіксуються defender Army у Camp/Regrouping/entered-for-Camp; за їх відсутності БС скасовується.
- **Neutral Region між Player**: Camp-bound entry автоматично створює БС за наявності чужої Army у `Camp`/Regrouping або `EnteringCamp`. Camp різних Player одночасно заборонений. Transit через Neutral Region вільний і не створює БС. FIFO та lifecycle ті самі, що для Camp-bound entry у зайняту Region.
- **Neutral Defense / City Raid**: миттєва CombatSituation з усіма стадіями в одному game event, поза FIFO, без `Dt`; defender — абстрактна оборона.

На Registration фіксуються attacker, Movement snapshot та причина; `defender_player` і `defenders[]` визначаються на Start для всіх PvP БС. Battle parameters фіксуються на Battle Start.

Якщо в Region немає Active/earlier Registered CombatSituation, Start відбувається одразу. Інакше situation лишається Registered у єдиній FIFO-черзі Region. Attacker отримує waiting command-lock без зміни фізичного state: Movement attacker зберігає Movement context із paused progress. Нові player-vs-player situations стають у чергу виключно за `порядком FIFO-реєстрації`; пріоритетів і «незалежних» паралельних PvP situations у Neutral Region немає.

### Start

CombatSituation Start — це:

- момент Registration, якщо в Region немає Active або earlier Registered CombatSituation;
- момент завершення або дострокового завершення попередньої Active CombatSituation, коли ця situation стала першою у FIFO-черзі Region.

На Start за актуальним станом Region:

- для територіальних registration reasons визначається фактичний `defender_player` за current ownership/occupation interaction; якщо queued Transit situation була зареєстрована в non-Occupied Region, але **на момент її Start** Region є Occupied, situation завершується без battle, бо Transit через Occupied Region не створює CombatSituation;
- для Camp-bound entry у Neutral Region визначається `defender_player` за актуальною чужою Camp/Regrouping/`EnteringCamp` presence; за її відсутності БС завершується без battle;
- для Transit classification береться зафіксований `target_opponent` і current ownership/occupation Region;
- фіксуються `defenders[]`;
- situation, яка за актуальними Start-умовами має завершитися без battle одразу (зокрема Camp-bound situation без доступного defender, Transit без Camp/Camp-bound defender ), завершується без запуску pre-battle interval;
- для решти situations починається повний pre-battle interval `Dt`, і defender отримує відповідний decision dialog.

Для Camp-bound entry у Neutral Region interaction на Start існує лише за наявності доступної чужої Army у `Camp`/Regrouping або `EnteringCamp`; Transit presence не враховується. За відсутності defender БС завершується без battle. Для будь-якої Camp-bound БС без battle час від entry зараховується до `entry -> Camp` `Dt`, а наслідки Camp arrival застосовуються до Start наступної FIFO situation. Transit attacker продовжує route; queued БС не видаляється до свого Start.

### Target opponent and Aggressive Transit

При створенні Movement фіксується [цільовий суперник]:

- final Region F Neutral -> `target_opponent = null`;
- F належить Player B і не Occupied -> `target_opponent = B`, незалежно від наявності військ B у F;
- F належить Player B, але Occupied Player C -> `target_opponent = C`.

Подальші зміни ownership/occupation F самі по собі ЦС не змінюють. Player може вручну запитати refresh ЦС за актуальним станом F; нове значення набуває чинності тільки при вході Army в наступну Region маршруту і не змінює CombatSituation, уже зареєстровану в поточній Region.

Для Transit через Owned non-Occupied Region, який створив CombatSituation:

- якщо Region формально належить `target_opponent` -> `Aggressive`;
- якщо Region належить іншому Player -> `NonAggressive`.

Transit через Occupied Region реєструє CombatSituation, але `allow_transit`, `Aggressive/NonAggressive` classification і defender Transit decision для такого проходу не застосовуються.

`attacker_destination_mode` (`Camp` або `Transit`) як і раніше визначається початковим Movement task і не переобчислюється CombatSituation.

### Defenders

Склад `defenders[]` остаточно фіксується на Start CombatSituation.

До `defenders[]` на Start належать — усі **доступні** Army defender Player у `Camp`/`Regrouping` і всі доступні Army, що вже мають `is_entered_for_camp == true` не пізніше Start. Army, яка є attacker іншої unresolved CombatSituation, не є доступним defender. Останні не утворюють окремої категорії battle participation: оскільки entry -> Camp і Start -> Battle Start обидва завжди дорівнюють `Dt`, на Battle Start вони гарантовано вже будуть у Camp або Regrouping і беруть участь на тих самих умовах. Це включає `retreat-local` Army, яка ввійшла в Region до Start.

Transit Army не долучаються до defense у V1 і продовжують маршрут незалежно від результату battle. Можливість такого долучення залишається відкладеною ідеєю в правилах, без полів або методів у поточній моделі.

Не може брати участь:

- Army, яка до Start уже вийшла з Camp і рухається до межі Region;
- будь-яка Army defender Player, яка ввійшла в Region після Start.

Army, яка входить після Start, ніколи не reinforcement цієї CombatSituation. Її подальша поведінка визначається її route та станом Region після завершення поточного battle; якщо вона сама створює умови атаки, реєструється окрема CombatSituation.

Після Start defender Army не може отримати Movement command, merge/split, зміну Commander або іншу реорганізацію до завершення CombatSituation. Player може тільки залишити конкретну Army для бою або наказати їй Retreat. Retreat застосовується за результатом/завершенням CombatSituation, а не миттєво в момент рішення; pre-battle Retreat не завдає втрат і після нього Army проходить Regrouping.

### No-battle resolution for Transit

Ці правила стосуються тільки Transit через Owned non-Occupied Region. Transit через Occupied Region не реєструє CombatSituation; якщо queued Transit situation була зареєстрована раніше, але **на момент її Start** Region є Occupied, вона завершується без battle.

Якщо attacker має `destination_mode = Transit` і на Start немає жодної Army defender Player у Camp-presence або вже entered-for-Camp state, CombatSituation завершується без battle. Самі по собі defender Transit Army, навіть якщо вони ввійшли до Start, не створюють battle context: attacker і ці defender Transit Army продовжують рух, а defender Transit Army не отримують можливості зупинитися для interception.

Якщо Transit `NonAggressive` і defenders є, defender протягом `Dt` може одноразово обрати `Allow` або `Fight`:

- `Allow` — situation завершується без battle, attacker продовжує Transit, усі defender Army лишаються у своїх поточних states;
- `Fight` — situation лишається active до Battle Start;
- якщо ручного рішення немає до Battle Start, `Region.allow_transit` є fallback: `true` -> no battle, `false` -> battle. Цей fallback існує тільки для non-Occupied Region.

### Default defender behavior at Battle Start

Якщо battle context зберігається до Battle Start:

- У V1 усі Transit Army продовжують Transit і не беруть участі в defense;
- для кожної defender Army effective threshold дорівнює її persistent `defense_loss_threshold`, якщо Player не задав pre-battle Retreat decision; Army, що на Start була entered-for-Camp, на Battle Start уже є звичайною Camp/Regrouping Army;
- pre-battle Retreat decision підміняє effective threshold тільки цієї Army на `0`.

Після цього **спочатку** перевіряється side-level legal Retreat. Якщо Defender не має жодної legal Retreat Region, спільний threshold примусово стає `1.0`, і жодна Army з effective `0` не відходить. Якщо legal Retreat є, усі defender Army з effective `0` виконують Retreat без casualties; тільки після їх виходу для решти Army береться minimum effective threshold.

Army як контейнери до Battle Start не перегруповуються: заборонені merge/split, transfer Unit/Soldier і зміна Commander. Persistent thresholds CombatSituation не змінює.

### Neutral Camp-bound entry and attack on occupier

У Neutral Region Camp-bound entry за чужої Camp/Regrouping/`EnteringCamp` presence автоматично реєструє БС. На Start фіксуються актуальний defender Player та `defenders[]`, виключаючи attackers інших unresolved situations. Army, які входять після Start, не додаються. Чужий Transit вільно продовжується. FIFO й command-lock такі самі, як при вході в зайняту Region.

Ті самі defender eligibility та command-lock rules застосовуються до Army occupier, коли їх атакує formal owner Region або третій Player.

### Queue

У межах кожної Region існує одна спільна FIFO-черга всіх player-vs-player CombatSituation. Це правило однакове для Owned, Occupied і Neutral Region та не залежить від того, які пари Player беруть участь. Одночасно Active може бути максимум одна player-vs-player CombatSituation Region.

Наступна Registered situation не стартує до завершення або дострокового завершення попередньої Active situation. Після її завершення найстаріша Registered situation Start-ує одразу. Якщо вона не завершується без battle прямо на Start, то отримує повний `Dt` до свого Battle Start. Таким чином між двома фактичними послідовними battle у цій Region не може бути менше `Dt`.

Queued PvP situation не має зафіксованих `defender_player` чи `defenders[]`: вони визначаються лише на Start. Attacker waiting command-locked зі збереженням Movement context. Для Neutral Camp-bound entry не вводяться окремі правила FIFO.

Attack на Neutral Defense та City Raid є миттєвими CombatSituation, але не входять у цю queue.

Player не може атакувати Neutral Defense або City в цій Region, якщо він є attacker будь-якої незавершеної CombatSituation у цій Region або defender Active CombatSituation у цій Region.

### User methods

- `user_set_transit_decision(decision)` — `Allow/Fight` для Active NonAggressive Transit; після вибору рішення immutable.
- `user_set_pre_battle_retreat_decision(army, retreat)` — до Battle Start задає для конкретної defender Army локальний override effective Defense Loss Threshold: `retreat = true` -> `0`, `false` -> її persistent `defense_loss_threshold`. Рішення можна задати незалежно від поточної на цей момент наявності Retreat destination; остаточна можливість Retreat перевіряється тільки на Battle Start.

### Domain methods

- `start()` — переводить Registered situation в Active, визначає актуального `defender_player` за ownership/occupation або чужою Camp/Regrouping/`EnteringCamp` presence у Neutral Region; фіксує `defenders[]`, виключаючи attackers інших unresolved situations. За відсутності defender завершує без battle. Для допустимого Transit у Owned Region визначає `transit_mode`; для решти запускає повний `Dt` до Battle Start.
- `determine_transit_mode()` — викликається тільки для Transit CombatSituation в Owned non-Occupied Region; повертає Aggressive, якщо Region формально належить зафіксованому `target_opponent`, інакше NonAggressive.
- `collect_defenders()` — на Start фіксує остаточний склад `defenders[]` для всіх доступних Army defender Player у Camp/Regrouping і всіх доступних Army з `is_entered_for_camp == true`, виключаючи attackers інших unresolved situations; останні гарантовано стануть Camp/Regrouping до Battle Start і не є окремим типом participant. Army, які входять після Start, не додаються.
- `resolve_pre_battle()` — спочатку застосовує Transit Allow/Fight. Якщо result — `Allow`, defender Retreat decisions не виконуються й Army лишаються у своїх states. Для situation, що доходить до Battle Start, method формує effective thresholds: defender pre-battle Retreat decision дає локальний `0`, інші Army зберігають persistent threshold; attacker використовує свій target/incidental threshold.
- `lock_combat_parameters()` — на Battle Start до zero-threshold Retreat викликає `get_legal_retreat_regions(role)` для обох sides, фіксує legal Retreat sets, effective thresholds та combat strengths у `locked_combat_parameters`. Загальне правило: якщо legal Retreat немає, threshold side примусово стає `1.0`. **Винятки:** при Camp-bound атаці Castle Region без legal Retreat повністю розміщені всередині Castle Defender Army з effective threshold `0` виходять із combat calculation без casualties, змішані Army беруть участь із `1.0`; при Transit-атаці Castle Region відсутність legal Retreat не примушує threshold до `1.0`, а Army з threshold `0` не беруть участі в calculation і залишаються в Region. Якщо legal Retreat є, Army з effective threshold `0` виконують pre-battle Retreat без casualties. Після виключення цих Army спільний Defender threshold визначається за мінімумом effective thresholds фактичних participants. Якщо Attacker Retreat-ить або Defender participants не залишилося, combat calculation пропускається.
- `get_legal_retreat_regions(role)` — використовує `region.neighbors[]`, `attacker_entry_from_region`, `attacker.player`/`defender_player`, `Region.player`, `Region.is_occupied`, `Region.is_castle_region`, `Region.blocking_camp_presence_player_ids[]`; повертає один side-level набір допустимих сусідніх Region. Для attacker розглядаються три напрями до source side, для defender — три протилежні напрями; Foreign Castle Region не допускається. Own Occupied Region не допускається. Для **foreign Owned** Region Retreat дозволений тільки якщо в ній немає жодної Army у `Camp`/`Regrouping` і жодної Army з `is_entered_for_camp == true`, незалежно від Player цієї Army; pure Transit не блокує. Neutral Region із locally-present just-fought opponent не допускається. Neutral Defense саме по собі Retreat не блокує. Оскільки всі Army однієї side знаходяться в одній combat Region і мають ту саму combat role/opponent, допустимість Region не залежить від конкретної Army.
- `select_retreat_region(role)` — один раз вибирає side-specific Retreat Region (`attacker_retreat_region` або `defender_retreat_region`) з side-level набору. Для Defender пріоритет: own non-Occupied Region > Neutral Region > допустима foreign Region. Серед own Region кожна кандидатна Region оцінюється відносно **Castle, до якого приєднана сама ця Region**; перевага має Region, ближча до свого `region.castle`. Це властивість кандидатної Region і не залежить від home Castle, Commander або складу конкретної Army. Серед Neutral Region перевага ближчій до власної territory; для foreign Region використовується визначений географічний критерій. Для Attacker найвищий пріоритет має source Region, якщо вона допустима, інакше застосовуються ті самі групові пріоритети. Якщо після всіх priority rules лишається кілька рівнозначних Region, вибір випадковий. Для кількох defender Army один результат застосовується до всіх Army цієї side, що виконують Retreat.
- `resolve_combat()`.
- `apply_casualties(result)` — після застосування всіх casualties до всіх Unit викликає `ensure_commander_after_casualties()` для кожної surviving Army; зміна Commander не впливає на вже завершений combat calculation, але діє для подальшого post-combat state.
- `apply_result(result)` — виконує Retreat, Occupation, Camp/Transit continuation. Якщо Camp-bound результат приводить до Occupation Castle Region, до відступу зовнішніх Unit виконується `Army.split_for_castle_occupation(castle)` і `Army.block_in_castle(castle)` через `Region.set_occupied_by()`; це стосується також defender Army, які вийшли з розрахунку бою через effective threshold `0` без casualties. Зміна occupier без перерви Occupation не розблоковує Army Castle. Після визначення loser викликає `select_retreat_region(role)` для відповідної side, якщо її Retreat Region ще не була вибрана; якщо програє defender side з кількома Army, всі вони використовують одну `defender_retreat_region`. Якщо attacker мав Transit, будь-який result не змушує defender Army, що ввійшли після Start, змінювати свій route; вони завершують свій planned movement після поточного battle. Якщо attacker мав Camp і програв — так само. Якщо attacker мав Camp, виграв і встановив Occupation, пізніша Army formal owner реєструє окрему CombatSituation проти occupier тільки якщо входить/лишається з локальною метою Camp; Transit через вже Occupied Region продовжується без CombatSituation.
- `finish_without_battle(reason)` — застосовує reason-specific continuation, знімає combat locks і **викликає `finish()` рівно один раз** для terminal status/FIFO handoff; для Camp-bound attacker без battle на Start враховує elapsed time від фізичного входу до Camp та блокує interleaving user commands між змінами Occupation і Start наступної FIFO situation. Для `Allow`/відсутності battle context defender Army не змінюють state. Якщо battle не відбувся через zero-threshold Retreat Attacker-а або всіх Defender Army, виконує відповідні side-level Retreat. В інших no-battle cases для attacker застосовує належне Camp/Transit/Occupation continuation. Цей же terminal path використовується для Neutral Camp-bound situation, що втратила defender до Start.
- `finish()` — terminal transition зі статусом `Resolved`; після повного застосування `apply_result()` або `finish_without_battle()` (включно з Occupation, Castle blocking/unblocking і Camp consequences) **синхронно викликає `Region.start_next_combat_if_possible()`**, без окремого trigger чи user action. Якщо наступна situation за Start-умовами завершується без battle, FIFO обробляється далі; зміни між terminal state та наступним Start не допускають interleaving user commands.

### Triggers

- `battle_start` `[event trigger]` тільки для Active player-vs-player CombatSituation.

### Trigger methods

- `check_trigger_battle_start()` — прогнозує Start + `Dt`.
- `on_trigger_battle_start()` — лише для Active situation, яка досягла `start_time + Dt`; послідовно викликає `resolve_pre_battle()` → `lock_combat_parameters()` → за наявності participants `resolve_combat()` → `apply_casualties(result)` → `apply_result(result)` → `finish()`. Якщо interaction невалідна, Transit дозволено або combat calculation не потрібний через pre-battle Retreat, викликає `finish_without_battle(reason)` з відповідними Retreat/continuation наслідками. `finish()` забезпечує FIFO handoff; обробка атомарна.

### Combat calculation outside `CombatSituation`

Neutral Defense attack і City Raid не є `CombatSituation`. Це миттєві Army actions, які використовують спільний combat-calculation mechanism, але не створюють lifecycle/queue object і не мають Battle Start delay.

- Neutral Defense destruction: attacker — одна конкретна Camp Army; Defender strength = current Neutral Defense + full City Defense, якщо City є; defender threshold = `1.0`; attacker використовує `target_combat_threshold`. При fail Neutral Defense повертається/лишається на pre-combat current value, City Defense незмінна. При success Neutral Defense = `0`.
- City Raid: attacker — одна конкретна Camp Army; Defender = City Defense; fixed raid threshold; attacker використовує `target_combat_threshold`; success викликає `City.complete_raid()`. При defeat Army не retreat-ить у сусідню Region: вона лишається в тому самому Camp і одразу входить у Regrouping на `Dt`.

---
