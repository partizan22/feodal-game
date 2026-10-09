# Backend models — military and movement

Military entities, Camp presence and Movement.

Загальні conventions, system-computed semantics та спільні припущення див. у [MODELS.md](MODELS.md).

---

## 5. `Knight`

Knight разом зі своїми Soldier представляє gameplay Unit; окремої backend-моделі Unit немає.

`location_state` описує тільки локальне розміщення Unit у Castle/Barracks або Camp для soldier-upkeep/Barracks Capacity. Воно не визначає map/combat presence: для movement, combat, Annexation, Founding та інших військових взаємодій місце Unit визначається його Army та `Army.camp`.

### Прямі характеристики

- `name`
- `castle` — home Castle.
- `army` — поточна Army.
- `soldiers` — кількість Soldier за Type.
- `base_strength`.
- `location_state` — `Castle` або `Camp`.
- `stationed_castle` — Castle, у Barracks якого фактично розміщений Knight/Unit при `location_state = Castle`, інакше `null`. Може відрізнятися від home `castle`; саме це поле визначає Barracks Capacity та Food consumption Castle.
- `status`.

### Динамічні характеристики

- `experience`

### Обчислювальні характеристики

- `soldier_count`.
- `food_consumption = soldier_count + 1` для active Knight.
- `experience_coefficient`.
- `unit_attack_strength`, `unit_defense_strength`.
- `coin_upkeep`.
- `player` — `castle.player`; проміжна характеристика для computed characteristics інших Model без ланцюга relationships.
- `current_region` — `army.current_region`.
- `current_camp` — `army.camp`.
- `is_regrouping` — чи `army.state == Regrouping`.
- `is_camp_presence` — `current_camp != null && army.state in {Camp, Regrouping}`.
- `experience_balance`.

### User methods

- `user_enter_castle(target_castle)`, `user_leave_castle()` — Knight може ввійти в будь-який Castle свого Player, якщо його Army має Camp-presence у Castle Region та `target_castle.can_house_unit(this)`; при вході встановлюються `location_state = Castle` і `stationed_castle = target_castle`, при виході — `location_state = Camp` і `stationed_castle = null`. Дозволено під час Regrouping і CombatSituation; самі ці дії не змінюють територіальний стан Army.
- `user_change_unit_composition(composition_delta)` — дозволено тільки при `location_state = Castle` у home Castle і коли Army Knight не `is_command_locked` жодною CombatSituation.

### Domain methods

- `set_location_state(state, target_castle = null)` — для `Castle` вимагає Camp-presence Army у Region `target_castle`, спільного Player та достатньої Capacity; оновлює `stationed_castle`. Для `Camp` очищує `stationed_castle`. Home Castle не змінюється.
- `leave_castle_for_movement()` — при виході Army у Movement очищує фактичне розміщення `stationed_castle` та переводить Knight у `location_state = Camp`.
- `set_soldiers(new_composition)`, `set_army(army)`.
- `add_battle_experience(amount)`.
- `apply_casualties(...)` — втрати Unit спочатку застосовуються до `soldiers`; Knight не може загинути, доки в нього лишається хоча б один Soldier.
- `die()` — переводить Knight у dead state, звільняє `stationed_castle` (Barracks Capacity) та викликає `castle.enqueue_knight_replacement()` для звільненого Palace slot. Commander тут не перепризначається негайно: це робиться після завершення розподілу всіх casualties combat, щоб не вибрати Knight, який також загине в цьому самому combat.
- `change_home_castle(new_castle)` — змінює home Castle, не підміняючи фактичне `stationed_castle` без окремого переміщення. Якщо active Knight залишає старий home Castle (зокрема founder після завершення Castle Founding), його старий Palace slot вважається звільненим і запускає той самий `KnightReplacement` queue mechanism, що й після death. У новому Castle цей Knight займає власний slot і не породжує додаткового Knight.

---

## 6. `Army`

### Прямі характеристики

- `player`.
- `current_region`.
- `camp` — current CampInRegion або `null`.
- `commander`.
- `defense_loss_threshold`.
- `target_combat_threshold`.
- `incidental_combat_threshold`.
- `state` — `Camp / Movement / Regrouping` та потрібні V1 підстани. `Regrouping` вважається Camp-presence для всіх правил, крім заборони самій Army починати Movement або Attack. Очікування queued CombatSituation не є окремим фізичним state: Camp attacker залишається `Camp`, Movement attacker залишається в Movement context з paused progress.

### Динамічні характеристики

- `regrouping_progress`.

### Обчислювальні характеристики

- `*knights[]`.
- `player_id` — ID `player`; проміжна scalar characteristic для computed characteristics інших Model без другого relationship hop.
- `*movement` — active Movement або `null`; якщо Army є queued attacker (`is_combat_waiting == true`), Movement може зберігати незавершений route/context, але progress призупинений.
- `*attacking_combat_situation` — незавершена CombatSituation, де ця Army є attacker, або `null`; invariant: максимум одна.
- `*active_defender_combat_situation` — Active CombatSituation, де Army входить у зафіксований список potential defenders, або `null`.
- `is_combat_waiting` — `attacking_combat_situation.status == Registered`. Це command-lock, а не фізичний location/state.
- `soldier_count`, `food_consumption`.
- `attack_strength`, `defense_strength`.
- `coin_upkeep`.
- `food_coin_compensation` — для Army у Movement дорівнює `food_consumption * coins_per_food`, включно з `transit`, `final-local`, `retreat-local` та paused-for-combat Movement; для Camp/Regrouping дорівнює `0`, бо там використовується local Food. Attacker, що чекає queued CombatSituation з Camp, продовжує local Food consumption.
- `is_entered_for_camp` — `true`, коли Army уже фізично ввійшла в current Region і ще рухається локально до Camp (`Movement.phase` є `final-local` або `retreat-local` з локальною метою `Camp`); leaving-Camp movement і чистий Transit дають `false`. Це проміжна характеристика для інших computed characteristics без ланцюга relationships.
- `regrouping_progress_rate` — так підібраний, щоб Regrouping тривав рівно `Dt`.
- `is_command_locked` — true для attacker будь-якої незавершеної CombatSituation та для potential defender Active CombatSituation; блокує Movement, merge/split, Commander/unit-composition reorganization та інші commands, які могли б змінити participant state. Combat-specific decisions залишаються доступними.

### User methods

- `user_merge_armies(armies, commander, thresholds)` — дозволено тільки Army одного Player у `state == Camp` в одному CampInRegion, якщо жодна з них не `is_command_locked`. Army у Regrouping merge не може.
- `user_split_army(groups)` — дозволено тільки для `state == Camp` і за відсутності command lock; Regrouping split забороняє.
- `user_change_commander(knight)` — дозволено у `Camp` або `Movement`, якщо Army не `is_command_locked`; у `Regrouping` заборонено.
- `user_change_combat_thresholds(values)` — змінює persistent thresholds тільки у `Camp` або `Movement` і коли Army не `is_command_locked`. У `Regrouping` persistent values не змінюються; effective `defense_loss_threshold = 0`, а за відсутності legal Retreat battle logic примусово використовує `1.0`. Attacker після Registration та potential defender після CombatSituation Start locked; Transit defender candidate може змінювати persistent `defense_loss_threshold`, доки не приєднався до defense і не отримав lock.
- `user_attack_player(target_camp)` — player-vs-player attack із Camp у Neutral Region. Дозволено тільки `state == Camp` (не Regrouping), якщо target Player має в цій самій Neutral Region хоча б одну Army з `state == Camp`; самі лише Regrouping або `is_entered_for_camp == true` ініціацію не дозволяють. Attacking Army не може бути attacker іншої незавершеної CombatSituation або potential defender active CombatSituation. Викликає `Region.register_combat(..., defender_player = target_camp.player)`, тому окрема CombatSituation одразу фіксує цю Army як єдиного attacker і конкретного defender Player.
- `user_raid_city(city)` — City Raid тільки для `state == Camp`. Заборонено, якщо Player цієї Army у цій Region є attacker будь-якої незавершеної CombatSituation або potential defender Active CombatSituation. Raid не входить у player-vs-player CombatSituation queue і відбувається миттєво.
- `user_attack_neutral_defense()` — миттєва атака Neutral Defense цієї Region конкретною Army у `state == Camp`; Regrouping не може її ініціювати. Заборонено, якщо Player цієї Army у цій Region є attacker будь-якої незавершеної CombatSituation або potential defender Active CombatSituation. Якщо в Region є City, до abstract Defender strength додається full City Defense. Ця дія не створює CombatSituation і не входить у FIFO-чергу Region.

### Domain methods

- `enter_region(region)` — фіксує фізичний вхід Army у Region.
- `enter_camp(camp)` — переводить Army у Camp і встановлює `camp`. Якщо на цей момент існує active Movement, він завершується/закривається; старий route більше не продовжується. Для retreat-local після цього окремо запускається Regrouping.
- `leave_camp()` — очищує `camp`; якщо це остання Army occupier у Camp, Occupation припиняється в цей момент.
- `start_movement(movement)` — дозволено тільки якщо Army не Regrouping і не command-locked CombatSituation; перед початком Movement усі Knight цієї Army з `location_state = Castle` автоматично переходять у `Camp` через `Knight.leave_castle_for_movement()`.
- `finish_movement()`.
- `set_combat_waiting(combat)` — застосовує command-lock queued attacker без зміни фізичного state. Для Camp Army `camp` і Camp-presence зберігаються; для Movement Army Movement переходить у paused-for-combat state без втрати route/context.
- `clear_combat_waiting()` — при CombatSituation Start/termination знімає waiting-lock; подальший Movement/Camp context визначається поточною CombatSituation.
- `start_regrouping()` — переводить Army у Regrouping і скидає progress. Camp зберігається; Army продовжує бути звичайною Camp-presence для Food, Annexation, Founding, Occupation, defense та Camp lifecycle. Persistent thresholds зберігаються без змін; для defense effective threshold під час Regrouping = `0` (або `1.0`, якщо legal Retreat відсутній).
- `finish_regrouping()` — повертає Army у Camp після `Dt`; знову діють збережені persistent thresholds.
- `start_retreat_to(region)` — звичайний player-vs-player Retreat: Army одразу фізично входить у допустиму retreat Region і починає `retreat-local` із метою Camp. Викликає `Region.resolve_arrival()` за тими самими Camp-bound registration conditions §16: у чужій Owned/Occupied Region CombatSituation створюється лише за наявності відповідної чужої Camp/Regrouping/раніше entered-for-Camp presence. За відсутності situation після `Dt` Army переходить у Camp, застосовує control/Occupation consequences і починає Regrouping; за наявності situation діють FIFO та CombatSituation Start -> Battle Start. Спільний Retreat кількох Army однієї side не створює дубльованих Occupation.
- `start_loss_regrouping_in_current_camp()` — спеціальний результат поразки від Neutral Defense або City Defense: без Retreat у сусідню Region і без окремого post-defeat `Dt`; Army лишається/переходить у той самий Camp і одразу починає Regrouping.
- `remove_dead_knights()`.
- `ensure_commander_after_casualties()` — після завершення розподілу casualties, якщо попередній Commander загинув, але в Army лишився хоча б один живий Knight, автоматично призначає Commander-ом живого Knight із найбільшим `experience`. При однаковому `experience` використовується детермінований stable tie-breaker. Army при цьому не розпадається, зберігає свої thresholds, Movement/Camp/Regrouping context і post-combat consequences. Якщо живих Knight не лишилося, Army припиняє існування як бойова сутність.

### Triggers

- `regrouping_complete` `[event trigger]`.

### Trigger methods

- `check_trigger_regrouping_complete()` — прогнозує `Dt` completion.
- `on_trigger_regrouping_complete()` — завершує Regrouping.

---

# Persistent interaction/process models

---

## 8. `CampInRegion`

Один active instance представляє один безперервний епізод присутності Army конкретного Player у Camp конкретної Region. Коли остання Army залишає Camp, instance логічно деактивується; наступна поява створює новий instance.

### Прямі характеристики

- `player`.
- `region`.
- `status`.

### Динамічні характеристики

- `control_progress`.

### Обчислювальні характеристики

- `*armies[]` — Army, для яких цей CampInRegion є `Army.camp`.
- `player_id` — ID `player`; проміжна scalar characteristic для computed characteristics інших Model без другого relationship hop.
- `food_consumption` — сума `Army.food_consumption` усіх `armies[]`, включно з Regrouping; для Castle Region локальна частина відокремлюється у `local_food_consumption`.
- `local_food_consumption` — для звичайної Region дорівнює `food_consumption`; для Castle Region враховує тільки Knight/Unit із `location_state = Camp` (поза Castle), а не Soldier/Unit, що споживають Food із запасів Castle.
- `has_eligible_presence` — чи є хоча б одна Army з `state in {Camp, Regrouping}`.
- `food_coin_compensation` — для звичайної Region та для **Occupied Castle Region** обчислюється з локального дефіциту: якщо `region.food_production < region.camp_food_consumption`, частка цього Camp дорівнює `local_food_consumption / region.camp_food_consumption` від непокритого дефіциту, помноженого на `coins_per_food`; інакше `0`. Для неокупованої Castle Region дорівнює `0`: власні Camp Army споживають Food Castle, а не локальний Food.
- `has_valid_adjacent_owned_region` — `player_id in region.valid_connected_neighbor_player_ids[]`; не обходить `Region -> neighbors[] -> Player`.
- `can_progress` — Annexation progress можливий за ownership/Neutral Defense/adjacency rules, тільки якщо `region.has_active_founding == false` і `region.blocking_camp_presence_player_ids[]` не містить іншого Player. Foreign Transit Army не блокує Annexation, навіть якщо через її Transit у Region існує CombatSituation. Саме foreign Camp/Camp-bound presence, а не наявність CombatSituation, призупиняє progress. Regrouping Army свого Player підтримує progress так само, як Camp Army.
- `control_progress_rate`, `required_control_progress`, `is_ready_for_annexation`.

### User methods

- `user_annex_region(castle)`.

### Domain methods

- `accept_army(army)`.
- `on_army_left(army)` — якщо Camp спорожнів, деактивує instance; якщо Player є occupier, вихід останньої Army з Camp припиняє Occupation.
- `get_defending_armies()` — повертає Camp/Regrouping Army; остаточний potential-defender set CombatSituation фіксується на Start і також включає Army, які вже entered-for-Camp та тому гарантовано стануть Camp/Regrouping до Battle Start.
- `validate_reorganization(armies)` — для merge/split вимагає `state == Camp` для всіх Army та відсутність command lock; Regrouping merge/split блокує. Зміна Commander і composition перевіряються окремими правилами.
- `can_annex_to(castle)`.

### Triggers

- `can_progress` `[state trigger]`.
- `ready_for_annexation` `[event trigger]`.

### Trigger methods

- `check_trigger_can_progress()`, `on_trigger_can_progress()`.
- `check_trigger_ready_for_annexation()`, `on_trigger_ready_for_annexation()`.

---

## 10. `Movement`

### Прямі характеристики

- `army`.
- `route[]` — запланована послідовність Region references.
- `current_region`.
- `next_region`.
- `destination_mode` — `Camp | Transit`; визначається початковим task для поточної local interaction і не змінюється через CombatSituation queue.
- `phase` — transit / final-local / retreat-local та потрібні підфази.
- `local_exit_region` — зафіксований вихід із current Region для transit phase.
- `status` — active / paused-for-combat / completed.
- `direction_revealed`.
- `target_opponent` — [цільовий суперник] цього Movement; Player або `null`. При створенні Movement визначається за станом final Region: Neutral -> `null`; Owned non-Occupied -> formal owner; Owned Occupied -> occupier. Не змінюється автоматично через подальшу зміну стану final Region.
- `pending_target_opponent` — конкретне нове значення [цільового суперника], обчислене й зафіксоване в момент explicit refresh; може бути Player або `null`. Подальші зміни final Region pending value не змінюють.
- `has_pending_target_opponent_update` — boolean, що відрізняє відсутність pending update від валідного `pending_target_opponent = null`.

### Динамічні характеристики

- `progress`.

### Обчислювальні характеристики

- `progress_rate` — `0`, якщо Movement paused через queued/Active CombatSituation interaction. Для Camp-exit -> next Region entry та entry -> Camp фаза завжди має тривалість `Dt`. Для звичайного Transit без CombatSituation базова тривалість `Dt`, потенційно з майбутнім movement-speed coefficient.
- `direction_reveal_progress` — для Transit досягається після половини local Transit progress; після цього іншим Player, яким доступна ця інформація, може бути показаний лише наступний exit direction Army.
- `phase_completion_progress`.

### User methods

- `user_start_movement(army, route[])` — заборонено для Regrouping або command-locked Army. При створенні Movement визначає власний `target_opponent` за актуальним станом final Region F: Neutral -> `null`; Owned non-Occupied -> formal owner; Owned Occupied -> occupier.
- `user_change_planned_route(route[])` — змінює тільки future route, коли Army не locked CombatSituation; уже зафіксований `local_exit_region` поточної Transit phase не змінюється, і сама зміна Route не переобчислює `Movement.target_opponent`. Army, яка вже виконує Transit усередині Neutral Region, не може перетворити поточний Transit на зупинку в цій Region: для Camp там вона повинна спочатку вийти з Region і зайти знову з відповідною local метою.
- `user_refresh_target_opponent()` — у момент явної дії Player одразу обчислює [цільового суперника] за актуальним станом поточної final Region F і фіксує отримане значення в `pending_target_opponent`, виставляючи `has_pending_target_opponent_update = true`. До входу в наступну Region це значення не переобчислюється, навіть якщо F зміниться. Воно не впливає на current Region або вже зареєстровану CombatSituation.

### Domain methods

- `validate_route(route[])` — перевіряє adjacency і фактичну доступність future route; перед реальним border entry наступна Region перевіряється знову, тому Castle Region, створена після planning, блокує майбутній entry. Grandfathering стосується тільки Army, яка вже фізично знаходилася всередині Region на момент створення Castle Region.
- `start_phase(...)` — для нової Transit phase фіксує `local_exit_region`, скидає `direction_revealed = false` і запускає local progress; подальша зміна future route не змінює вже зафіксований exit цієї phase.
- `start_retreat_local(...)` — player-vs-player Retreat local movement до Camp триває `Dt`, включно з pre-battle Retreat; Retreat entry не запускає нову CombatSituation. Після Camp arrival у чужу Owned non-Occupied Region викликається Occupation; після цього Army переходить у Regrouping.
- `determine_target_opponent(final_region)` — повертає `null` для Neutral final Region, formal owner для non-Occupied Owned final Region, occupier для Occupied final Region.
- `pause_for_combat_waiting()` / `resume_after_combat_waiting()` — зберігають route/context без просування progress; не змінюють фізичний state Camp Army, якщо queued combat було ініційовано з Camp.
- `advance_to_next_region()` — безпосередньо при border entry, якщо `has_pending_target_opponent_update == true`, присвоює вже зафіксоване `target_opponent = pending_target_opponent` без нового обчислення за станом final Region, після чого скидає pending flag і викликає `Region.resolve_arrival()`. Тому новий [цільовий суперник] використовується вже для interaction у щойно введеній Region. Arrival після Start чужої Active CombatSituation не додає Army до її defenders.
- `resolve_camp_arrival()` — завершує normal Camp arrival; для `retreat-local` виконує retreat-specific Camp arrival без нового combat registration, за потреби встановлює Occupation чужої Owned Region і запускає Regrouping.
- `continue_after_transit_combat()` — продовжує попередній route після завершення interaction, якщо battle/no-battle result це дозволяє.

### Triggers

- `direction_revealed` `[state trigger]`.
- `next_region_reached` `[event trigger]`.
- `camp_reached` `[event trigger]`.

### Trigger methods

- `check_trigger_direction_revealed()`, `on_trigger_direction_revealed()`.
- `check_trigger_next_region_reached()`, `on_trigger_next_region_reached()`.
- `check_trigger_camp_reached()`, `on_trigger_camp_reached()`.

---
