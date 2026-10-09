# Backend models — combat

Player-vs-player CombatSituation and combat lifecycle.

Загальні conventions, system-computed semantics та спільні припущення див. у [MODELS.md](MODELS.md).

---

## 7. `CombatSituation`

`CombatSituation` у цій секції — одна модель для всіх player-vs-player interaction з registration/start/active lifecycle; відмінності між registration reasons реалізуються умовною domain logic всередині цієї моделі. Окремі inherited submodels поки не вводяться. Attack на Neutral Defense та City Raid використовують combat calculation, але не входять у player-vs-player queue і не чекають `Dt`.

Одна CombatSituation завжди має рівно одну attacking Army. Дві Army одного Player, які послідовно створюють умови атаки в одній Region, реєструють дві окремі CombatSituation; вони не об'єднуються в одну attacking side і не є reinforcement одна одній.

### Прямі характеристики

- `region`.
- `attacker` — єдина attacking Army; фіксується на Registration.
- `registration_reason` — Transit entry into foreign Owned non-Occupied Region / Camp-bound entry (включно з Retreat) за наявності чужої Camp/Regrouping/раніше entered-for-Camp Army / own Region became occupied around already-present late Camp-bound Army / explicit Neutral Camp attack.
- `registration_order` — FIFO order у єдиній player-vs-player черзі Region.
- `status` — `Registered / Active / Resolved`.
- `start_time` — встановлюється тільки при CombatSituation Start; `null` для queued Registered.
- `defender_player` — для explicit Neutral Camp attack фіксується на Registration як Player обраного target Camp; для територіальних registration reasons лишається `null` до Start і визначається за актуальним ownership/occupation interaction.
- `potential_defenders[]` — `null/empty` до Start; на Start фіксуються доступні Camp/Regrouping/entered-for-Camp Army defender Player. Army, яка сама є attacker іншої unresolved CombatSituation (Active або queued), виключається.
- `transit_defender_candidates[]` — Army defender Player, які ввійшли в Region для Transit до Start, на Start ще не вийшли та не є attackers інших unresolved CombatSituations. Вони не беруть участі автоматично й не блокуються як potential defenders, доки явно не оберуть залишитися для defense; join можливий лише до Battle Start і до фактичного виходу candidate з Region, після чого candidate більше не може приєднатися.
- `transit_defender_join_decisions` — явні рішення Transit candidates, визначених на Start, залишитися для defense; після такого рішення Army припиняє свій Transit/старий route, переходить у `Region.get_or_create_camp(defender_player)`, видаляється з `transit_defender_candidates[]`, додається до `potential_defenders[]` і від цього моменту отримує combat command-lock.
- `defenders_retreat_decisions` — рішення Retreat для окремих potential defender Army. Воно не виконує Retreat одразу, а локально для цієї CombatSituation підміняє effective `defense_loss_threshold` відповідної Army на `0`. Persistent `Army.defense_loss_threshold` не змінюється. Допустимість і destination Retreat визначаються на рівні всієї defender side.
- `attacker_retreat_region` — side-level Retreat destination attacker side або `null`, якщо ще не потрібна/не вибрана.
- `defender_retreat_region` — side-level Retreat destination defender side або `null`, якщо ще не потрібна/не вибрана. Якщо частина defender Army виконує pre-battle Retreat, Region вибирається один раз для defender side; якщо решта defender Army пізніше також мають Retreat у межах цієї CombatSituation, використовується те саме значення.
- `transit_decision` — `null / allow / fight` для non-aggressive Transit; після ручного вибору не змінюється.
- `attacker_destination_mode` — `Camp | Transit`; для CombatSituation, створеної через Movement, походить із початкового Movement task для входу в цю Region і не змінюється queueing; для explicit Neutral Camp attack фіксується як `Camp`.
- `target_opponent` — тільки для CombatSituation, зареєстрованої через Movement: snapshot `Movement.target_opponent`, чинного при вході attacker у Region. Для explicit attack з Camp у Neutral Region не використовується і може бути `null`. Подальше ручне refresh ЦС не змінює вже зареєстровану CombatSituation.
- `transit_mode` — `null` до CombatSituation Start; для Transit interaction у Owned non-Occupied Region на Start один раз визначається за зафіксованим `target_opponent`: Aggressive, якщо Region формально належить `target_opponent`, інакше NonAggressive. Transit через Occupied Region CombatSituation не створює і `transit_mode` для нього не визначається.
- `locked_combat_parameters` — остаточно фіксуються на Battle Start.

### Динамічні характеристики

- `battle_start_progress` — рахується тільки для Active CombatSituation; від Start до Battle Start завжди `Dt`.

### Обчислювальні характеристики

- `is_queued` — `status == Registered`; situation без попередників одразу переходить у `Active`, тому стабільний `Registered` стан означає очікування у FIFO-черзі.
- `is_active` — `status == Active`.
- `battle_start_progress_rate` / `required_battle_start_progress` — дають рівно `Dt` від Start до Battle Start.
- `is_battle_valid` — чи після pre-battle resolution лишилися умови реального battle.
- `attacker_loss_threshold` — initial effective value: для explicit attack з Camp у Neutral Region `attacker.target_combat_threshold`; для CombatSituation через Movement — target threshold, якщо фактичний `defender_player == target_opponent`, і incidental threshold інакше. У V1 обидва attacker thresholds строго більші за `0`; при відсутності legal Retreat effective attacker threshold примусово стає `1.0`.
- `defender_combat_threshold` — на Battle Start для кожної potential defender Army береться effective threshold: `0`, якщо для неї встановлено pre-battle Retreat decision, інакше її persistent `defense_loss_threshold`. Спочатку перевіряється side-level Retreat availability. Якщо defender side не має legal Retreat Region, її combat threshold примусово стає `1.0` і жодна Army не виконує pre-battle Retreat. Якщо legal Retreat є, Army з effective threshold `0` спочатку виконують Retreat без casualties; після їх виключення спільний `defender_combat_threshold` дорівнює мінімальному effective threshold серед Army, що лишилися. Якщо жодної defender Army не лишилося, battle не відбувається.
- `defender_loss_threshold` — чинне значення `defender_combat_threshold`; один спільний threshold для всієї defense side, що lock-иться на Battle Start.
- `attacker_strength` — `attacker.attack_strength` для будь-якого player-vs-player battle.
- `defender_strength` — сума strength усіх participating defender Army: для explicit Neutral Camp battle використовується їх `attack_strength`, для territorial invasion/occupation defense — їх `defense_strength`.

### Registration

CombatSituation реєструється, коли:

- Army Player A входить у чужу Owned **non-Occupied** Region з метою **Transit** — за звичайними Transit registration rules;
- Army Player A входить у чужу Owned/Occupied Region з локальною метою **Camp**, включно з Retreat, лише якщо там уже є чужа Army у `Camp`/Regrouping або раніше entered-for-Camp (`EnteringCamp`); сама лише чужа Transit-presence CombatSituation не створює;
- formal owner входить у власну ще non-Occupied Region з метою Camp, якщо там є чужа Army, що вже entered-for-Camp, навіть якщо Occupation ще не виникла;
- Region Player A стає Occupied, коли Camp-bound Army A вже рухається всередині цієї Region, увійшла туди до Occupation, але вже не може приєднатися до попередньої defense; Transit Army CombatSituation не створює;
- Army Player A у `state == Camp` (не Regrouping) у Neutral Region ініціює attack на Player B, який має в цій Region хоча б одну Army у `Camp` або Regrouping; самі лише entered-for-Camp Army B для Registration недостатні.

На Registration фіксується `attacker`, `attacker_destination_mode` та registration metadata. Для CombatSituation через Movement також фіксується `target_opponent` snapshot. Для explicit Neutral Camp attack Movement `target_opponent` не використовується; натомість одразу фіксується конкретний `defender_player`, якого Player свідомо атакував. Для територіальних registration reasons `defender_player` на Registration не фіксується. `potential_defenders[]` і `transit_defender_candidates[]` формуються тільки на Start; battle thresholds lock-яться пізніше.

Якщо в Region немає Active/earlier Registered CombatSituation, Start відбувається одразу. Інакше situation лишається Registered у єдиній FIFO-черзі Region. Attacker отримує waiting command-lock без зміни фізичного state: Camp attacker лишається Camp-presence; Movement attacker зберігає Movement context із paused progress. Нові player-vs-player situations стають у чергу виключно за `registration_order`; пріоритетів і «незалежних» паралельних PvP situations у Neutral Region немає.

### Start

CombatSituation Start — це:

- момент Registration, якщо в Region немає Active або earlier Registered CombatSituation;
- момент завершення або дострокового завершення попередньої Active CombatSituation, коли ця situation стала першою у FIFO-черзі Region.

На Start за актуальним станом Region:

- для територіальних registration reasons визначається фактичний `defender_player` за current ownership/occupation interaction; якщо queued Transit situation була зареєстрована в non-Occupied Region, але **на момент її Start** Region є Occupied, situation завершується без battle, бо Transit через Occupied Region не створює CombatSituation;
- для explicit Neutral Camp attack використовується зафіксований на Registration `defender_player` і перевіряється, чи interaction з ним досі актуальний; situation не може перенаправитися на іншого Player;
- для Transit classification береться зафіксований `target_opponent` і current ownership/occupation Region;
- фіксуються `potential_defenders[]` і `transit_defender_candidates[]`;
- situation, яка за актуальними Start-умовами має завершитися без battle одразу (зокрема Camp-bound situation без доступного defender, Transit без Camp/Camp-bound defender або Transit через Region, що на Start є Occupied), завершується без запуску pre-battle interval;
- для решти situations починається повний pre-battle interval `Dt`, і defender отримує відповідний decision dialog.

Для explicit Neutral Camp attack interaction на Start вважається актуальним, якщо зафіксований `defender_player` усе ще має в Region хоча б одну Army у `Camp`/`Regrouping` або Army, що вже ввійшла з локальною метою Camp. Сам по собі Transit цього Player interaction актуальним не робить. Якщо ця умова не виконується, CombatSituation завершується без battle зі статусом `Resolved`, attacker lock знімається, після чого Region може Start-нути наступну Registered situation. Для територіальних registration reasons, якщо на Start Region стала Neutral або немає доступного defender Player за актуальною Camp/Regrouping/EnteringCamp presence, CombatSituation завершується без battle. Для Camp-bound attacker час очікування від фізичного входу в Region зараховується в `entry -> Camp` `Dt`: якщо `Dt` минув, він одразу переходить у Camp, інакше завершує лише залишок; контроль/Occupation оновлюються перед Start наступної situation. Transit attacker продовжує свій route. Жодна queued situation не завершується або не видаляється раніше власного Start лише через те, що майбутні умови, ймовірно, зникли.

### Target opponent and Aggressive Transit

При створенні Movement фіксується [цільовий суперник]:

- final Region F Neutral -> `target_opponent = null`;
- F належить Player B і не Occupied -> `target_opponent = B`, незалежно від наявності військ B у F;
- F належить Player B, але Occupied Player C -> `target_opponent = C`.

Подальші зміни ownership/occupation F самі по собі ЦС не змінюють. Player може вручну запитати refresh ЦС за актуальним станом F; нове значення набуває чинності тільки при вході Army в наступну Region маршруту і не змінює CombatSituation, уже зареєстровану в поточній Region.

Для Transit через Owned non-Occupied Region, який створив CombatSituation:

- якщо Region формально належить `target_opponent` -> `Aggressive`;
- якщо Region належить іншому Player -> `NonAggressive`.

Transit через Occupied Region CombatSituation взагалі не створює: `allow_transit`, `Aggressive/NonAggressive` classification і defender Transit decision для такого проходу не застосовуються.

`attacker_destination_mode` (`Camp` або `Transit`) як і раніше визначається початковим Movement task і не переобчислюється CombatSituation.

### Potential defenders

Potential defender фіксується тільки на Start CombatSituation.

Potential defenders на Start — усі **доступні** Army defender Player у `Camp`/`Regrouping` і всі доступні Army, що вже мають `is_entered_for_camp == true` не пізніше Start. Army, яка є attacker іншої unresolved CombatSituation, не є доступним defender. Останні не утворюють окремої категорії battle participation: оскільки entry -> Camp і Start -> Battle Start обидва завжди дорівнюють `Dt`, на Battle Start вони гарантовано вже будуть у Camp або Regrouping і беруть участь на тих самих умовах. Це включає `retreat-local` Army, яка ввійшла в Region до Start.

Окремо Army defender Player, яка ввійшла в Region для Transit **до Start**, на Start ще не вийшла з Region і **не є attacker іншої unresolved CombatSituation**, потрапляє в `transit_defender_candidates[]`, а не в `potential_defenders[]`. Поки вона не обрала defense, вона не command-locked цією CombatSituation і продовжує Transit. До Battle Start і до фактичного виходу з Region Player може явно наказати їй залишитися; тоді Army видаляється з `transit_defender_candidates[]`, додається до `potential_defenders[]` і від цього моменту отримує combat command-lock. Якщо такого рішення немає до першої з цих меж, Army більше не може приєднатися.

Не може брати участь:

- Army, яка до Start уже вийшла з Camp і рухається до межі Region;
- будь-яка Army defender Player, яка ввійшла в Region після Start.

Army, яка входить після Start, ніколи не reinforcement цієї CombatSituation. Її подальша поведінка визначається її route та станом Region після завершення поточного battle; якщо вона сама створює умови атаки, реєструється окрема CombatSituation.

Після Start potential defender Army не може отримати Movement command, merge/split, зміну Commander або іншу реорганізацію до завершення CombatSituation. Player може тільки залишити конкретну Army для бою або наказати їй Retreat. Retreat застосовується за результатом/завершенням CombatSituation, а не миттєво в момент рішення; pre-battle Retreat не завдає втрат і після нього Army проходить Regrouping.

### No-battle resolution for Transit

Ці правила стосуються тільки Transit через Owned non-Occupied Region. Transit через Occupied Region не реєструє CombatSituation; якщо queued Transit situation була зареєстрована раніше, але **на момент її Start** Region є Occupied, вона завершується без battle.

Якщо attacker має `destination_mode = Transit` і на Start немає жодної Army defender Player у Camp-presence або вже entered-for-Camp state, CombatSituation завершується без battle. Самі по собі defender Transit Army, навіть якщо вони ввійшли до Start, не створюють battle context: attacker і ці defender Transit Army продовжують рух, а defender Transit Army не отримують можливості зупинитися для interception.

Якщо Transit `NonAggressive` і potential defenders є, defender протягом `Dt` може одноразово обрати `Allow` або `Fight`:

- `Allow` — situation завершується без battle, attacker продовжує Transit, усі defender Army лишаються у своїх поточних states;
- `Fight` — situation лишається active до Battle Start;
- якщо ручного рішення немає до Battle Start, `Region.allow_transit` є fallback: `true` -> no battle, `false` -> battle. Цей fallback існує тільки для non-Occupied Region.

### Default defender behavior at Battle Start

Якщо battle context зберігається до Battle Start:

- Transit candidates без явного join продовжують Transit і не беруть участі;
- для кожної potential defender Army effective threshold дорівнює її persistent `defense_loss_threshold`, якщо Player не задав pre-battle Retreat decision; Army, що на Start була entered-for-Camp, на Battle Start уже є звичайною Camp/Regrouping Army;
- pre-battle Retreat decision підміняє effective threshold тільки цієї Army на `0`.

Після цього **спочатку** перевіряється side-level legal Retreat. Якщо Defender не має жодної legal Retreat Region, спільний threshold примусово стає `1.0`, і жодна Army з effective `0` не відходить. Якщо legal Retreat є, усі defender Army з effective `0` виконують Retreat без casualties; тільки після їх виходу для решти Army береться minimum effective threshold.

Army як контейнери до Battle Start не перегруповуються: заборонені merge/split, transfer Unit/Soldier і зміна Commander. Persistent thresholds CombatSituation не змінює.

### Neutral Camp attack and attack on occupier

Explicit attack A -> B у Neutral Region можна зареєструвати тільки якщо B на цей момент має хоча б одну Army у `Camp` або Regrouping; самі лише entered-for-Camp без Camp-presence для ініціації недостатні. На Registration фіксується `defender_player = B`. На Start ця situation або лишається атакою саме проти B, або, якщо interaction з B вже не актуальний, завершується без battle зі статусом `Resolved`; вона ніколи не перенаправляється на C чи іншого Player. Усі **доступні** Army B, які на Start є у Camp/Regrouping або вже entered-for-Camp, обов'язково входять у `potential_defenders[]`; attackers інших unresolved CombatSituations виключаються; для entered-for-Camp це не опціональне приєднання. Army B, що entered-for-Transit до Start і ще не вийшла з Region, є transit defender candidate та може явно залишитися для defense тільки до Battle Start і до моменту свого виходу; leaving-Camp Army участі не бере. Army B, що входить після Start, не може приєднатися.

Ті самі defender eligibility та command-lock rules застосовуються до Army occupier, коли їх атакує formal owner Region або третій Player.

### Queue

У межах кожної Region існує одна спільна FIFO-черга всіх player-vs-player CombatSituation. Це правило однакове для Owned, Occupied і Neutral Region та не залежить від того, які пари Player беруть участь. Одночасно Active може бути максимум одна player-vs-player CombatSituation Region.

Наступна Registered situation не стартує до завершення або дострокового завершення попередньої Active situation. Після її завершення найстаріша Registered situation Start-ує одразу. Якщо вона не завершується без battle прямо на Start, то отримує повний `Dt` до свого Battle Start. Таким чином між двома фактичними послідовними battle у цій Region не може бути менше `Dt`.

Queued territorial situation ще не має `defender_player`; queued explicit Neutral Camp attack уже має зафіксованого `defender_player` з Registration. Жодна queued situation ще не має `potential_defenders[]` або `transit_defender_candidates[]`: вони формуються тільки на Start. Attacker має waiting command-lock, але зберігає фізичний Camp/Movement context; Transit через non-Occupied Owned Region може бути затриманий queueing. Transit через Occupied Region CombatSituation не створює і в queue не потрапляє.

Attack на Neutral Defense та City Raid не створюють CombatSituation і не входять у цю queue.

Player не може атакувати Neutral Defense або City в цій Region, якщо він є attacker будь-якої незавершеної CombatSituation у цій Region або potential defender Active CombatSituation у цій Region.

### User methods

- `user_set_transit_decision(decision)` — `Allow/Fight` для Active NonAggressive Transit; після вибору рішення immutable.
- `user_set_pre_battle_retreat_decision(army, retreat)` — до Battle Start задає для конкретної potential defender Army локальний override effective Defense Loss Threshold: `retreat = true` -> `0`, `false` -> її persistent `defense_loss_threshold`. Рішення можна задати незалежно від поточної на цей момент наявності Retreat destination; остаточна можливість Retreat перевіряється тільки на Battle Start.
- `user_join_defense_from_transit(army)` — до Battle Start і до фактичного виходу Army з Region для Army з `transit_defender_candidates[]` фіксує рішення залишитися для defense: завершує її active Movement/старий route, переводить Army у `Region.get_or_create_camp(defender_player)`, видаляє її з `transit_defender_candidates[]`, додає до `potential_defenders[]` і застосовує стандартний defender combat command-lock. Від переходу в Camp діють звичайні Camp Food/presence rules; post-battle behavior такий самий, як для defender Army, що вже була в Camp.

### Domain methods

- `start()` — переводить Registered situation в Active. Для territorial registration reasons визначає актуального `defender_player` з пріоритетом доступної чужої Camp/Regrouping presence над `EnteringCamp`; якщо доступного defender немає, завершує situation без battle; для explicit Neutral Camp attack не переобчислює його, а перевіряє актуальність interaction із зафіксованим Player. Queued Transit situation, для якої Region **на момент Start** є Occupied, завершується без battle. Для іншої чинної Transit situation визначає `transit_mode`, після чого формує defender sets; Transit без Camp/Camp-bound defender завершується одразу без `Dt`, а решта situations запускають повний `Dt` до Battle Start. Якщо interaction більше не існує, завершує situation через no-battle path зі статусом `Resolved`.
- `determine_transit_mode()` — викликається тільки для Transit CombatSituation в Owned non-Occupied Region; повертає Aggressive, якщо Region формально належить зафіксованому `target_opponent`, інакше NonAggressive.
- `collect_potential_defenders()` — на Start фіксує `potential_defenders[]` для всіх доступних Army defender Player у Camp/Regrouping і всіх доступних Army з `is_entered_for_camp == true`, виключаючи attackers інших unresolved situations; останні гарантовано стануть Camp/Regrouping до Battle Start і не є окремим типом participant. Окремо фіксує `transit_defender_candidates[]` для Transit Army, що ввійшли строго до Start та ще не вийшли; Army, які входять після Start, не додаються.
- `resolve_pre_battle()` — спочатку застосовує Transit Allow/Fight. Якщо result — `Allow`, defender Retreat decisions не виконуються й Army лишаються у своїх states. Для situation, що доходить до Battle Start, method формує effective thresholds: defender pre-battle Retreat decision дає локальний `0`, інші Army зберігають persistent threshold; attacker використовує свій target/incidental threshold.
- `lock_combat_parameters()` — на Battle Start **до будь-якого zero-threshold Retreat** обчислює та lock-ить side-level legal Retreat sets для Attacker і Defender. Якщо side не має legal Retreat Region, її effective combat threshold примусово стає `1.0`; для Defender це також скасовує виконання локальних `0` override. Після цього Army/side з effective threshold `0` і legal Retreat виконують Retreat без casualties. Якщо Defender після цього має participants, його спільний threshold = minimum їх effective thresholds; strengths lock-яться вже для фактичних participants. Якщо Attacker Retreat-ить або всі Defender Army Retreat-ять, battle calculation не виконується.
- `get_legal_retreat_regions(role)` — повертає один side-level набір допустимих сусідніх Region. Для attacker розглядаються три напрями до source side, для defender — три протилежні напрями; для explicit player-vs-player attack між Camp в одній Neutral Region напрямкового обмеження немає і розглядаються всі шість. Foreign Castle Region не допускається. Own Occupied Region не допускається. Для **foreign Owned** Region Retreat дозволений тільки якщо в ній немає жодної Army у `Camp`/`Regrouping` і жодної Army з `is_entered_for_camp == true`, незалежно від Player цієї Army; pure Transit не блокує. Neutral Region із locally-present just-fought opponent не допускається. Neutral Defense саме по собі Retreat не блокує. Оскільки всі Army однієї side знаходяться в одній combat Region і мають ту саму combat role/opponent, допустимість Region не залежить від конкретної Army.
- `select_retreat_region(role)` — один раз вибирає side-specific Retreat Region (`attacker_retreat_region` або `defender_retreat_region`) з side-level набору. Для Defender пріоритет: own non-Occupied Region > Neutral Region > допустима foreign Region. Серед own Region кожна кандидатна Region оцінюється відносно **Castle, до якого приєднана сама ця Region**; перевага має Region, ближча до свого `region.castle`. Це властивість кандидатної Region і не залежить від home Castle, Commander або складу конкретної Army. Серед Neutral Region перевага ближчій до власної territory; для foreign Region використовується визначений географічний критерій. Для Attacker найвищий пріоритет має source Region, якщо вона допустима, інакше застосовуються ті самі групові пріоритети. Якщо після всіх priority rules лишається кілька рівнозначних Region, вибір випадковий. Для кількох defender Army один результат застосовується до всіх Army цієї side, що виконують Retreat.
- `resolve_combat()`.
- `apply_casualties(result)` — після застосування всіх casualties до всіх Unit викликає `ensure_commander_after_casualties()` для кожної surviving Army; зміна Commander не впливає на вже завершений combat calculation, але діє для подальшого post-combat state.
- `apply_result(result)` — виконує Retreat, Occupation, Camp/Transit continuation. Якщо Camp-bound результат приводить до Occupation Castle Region, до відступу зовнішніх Unit виконується `Army.split_for_castle_occupation(castle)` і `Army.block_in_castle(castle)` через `Region.set_occupied_by()`; це стосується також defender Army, які вийшли з розрахунку бою через effective threshold `0` без casualties. Зміна occupier без перерви Occupation не розблоковує Army Castle. Після визначення loser викликає `select_retreat_region(role)` для відповідної side, якщо її Retreat Region ще не була вибрана; якщо програє defender side з кількома Army, всі вони використовують одну `defender_retreat_region`. Army, що приєдналася до defense з Transit, до battle вже є звичайною Camp Army і далі обробляється так само, як інші defender Army. Якщо attacker мав Transit, будь-який result не змушує defender Army, що ввійшли після Start, змінювати свій route; вони завершують свій planned movement після поточного battle. Якщо attacker мав Camp і програв — так само. Якщо attacker мав Camp, виграв і встановив Occupation, пізніша Army formal owner реєструє окрему CombatSituation проти occupier тільки якщо входить/лишається з локальною метою Camp; Transit через вже Occupied Region продовжується без CombatSituation.
- `finish_without_battle(reason)` — завершує situation зі статусом `Resolved`, знімає combat locks і застосовує reason-specific continuation; для Camp-bound attacker без battle на Start враховує elapsed time від фізичного входу до Camp та блокує interleaving user commands між змінами Occupation і Start наступної FIFO situation. Для `Allow`/відсутності battle context defender Army не змінюють state. Якщо battle не відбувся через zero-threshold Retreat Attacker-а або всіх Defender Army, виконує відповідні side-level Retreat. В інших no-battle cases для attacker застосовує належне Camp/Transit/Occupation continuation. Цей же terminal path використовується для obsolete explicit Neutral Camp attack.
- `finish()` — terminal transition зі статусом `Resolved`; після повного застосування `apply_result()` або `finish_without_battle()` (включно з Occupation, Castle blocking/unblocking і Camp consequences) **синхронно викликає `Region.start_next_combat_if_possible()`**, без окремого trigger чи user action. Якщо наступна situation за Start-умовами завершується без battle, FIFO обробляється далі; зміни між terminal state та наступним Start не допускають interleaving user commands.

### Triggers

- `battle_start` `[event trigger]` тільки для Active player-vs-player CombatSituation.

### Trigger methods

- `check_trigger_battle_start()` — прогнозує Start + `Dt`.
- `on_trigger_battle_start()` — спочатку виконує pre-battle resolution; якщо situation закінчується без battle, викликає `finish_without_battle(reason)` з відповідною причиною. Інакше lock-ає parameters і resolve-ить battle.

### Combat calculation outside `CombatSituation`

Neutral Defense attack і City Raid не є `CombatSituation`. Це миттєві Army actions, які використовують спільний combat-calculation mechanism, але не створюють lifecycle/queue object і не мають Battle Start delay.

- Neutral Defense destruction: attacker — одна конкретна Camp Army; Defender strength = current Neutral Defense + full City Defense, якщо City є; defender threshold = `1.0`; attacker використовує `target_combat_threshold`. При fail Neutral Defense повертається/лишається на pre-combat current value, City Defense незмінна. При success Neutral Defense = `0`.
- City Raid: attacker — одна конкретна Camp Army; Defender = City Defense; fixed raid threshold; attacker використовує `target_combat_threshold`; success викликає `City.complete_raid()`. При defeat Army не retreat-ить у сусідню Region: вона лишається в тому самому Camp і одразу входить у Regrouping на `Dt`.

---
