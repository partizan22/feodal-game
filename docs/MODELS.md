# Backend models

Робочий опис backend game-logic моделей. Для кожної Model тут фіксуються:

- прямі характеристики;
- динамічні характеристики;
- обчислювальні характеристики;
- user methods, що реалізують основну логіку відповідних user actions;
- внутрішні domain methods, які використовуються іншими game methods;
- triggers;
- `check_trigger_*()` та `on_trigger_*()` для кожного trigger-а.

Root model для user GameEvent поки не визначена до проєктування взаємодії з frontend/API. Тому `user_*` тут означає Model, яка реалізує основну логіку дії, а не обов'язково майбутню root model. Після визначення root model префікси методів за потреби будуть змінені.

Обчислювальна характеристика, позначена `*` перед назвою, є **system-computed**: її значення не зберігається прямо і не обчислюється game logic самої Model, а надається infrastructure. Типові приклади — reverse relationships та топологічні зв'язки карти. `[]` у назві означає колекцію; для system-computed relationship це список посилань на Model, для звичайної computed characteristic це може бути список scalar values.

Для computed characteristics діє правило: якщо `A.computed` потрібне значення з `A -> B -> C`, відповідна інформація зазвичай піднімається в computed characteristic Model `B`, а `A` читає вже її. Прямий system-computed зв'язок `A -> C` додається лише коли він сам є природною характеристикою `A`.

Це обмеження не поширюється на domain/user methods: method може обходити потрібні relationships і читати характеристики пов'язаних Model без створення проміжних computed characteristics лише заради такого обходу.

Рівні всіх Building одного Castle зберігаються як одна характеристика `levels`.

Wood / Stone / Iron зберігаються як одна характеристика `resources` — один value object / helper class із трьома значеннями.

Food consumption одного Soldier не залежить від Soldier Type. Один Knight також рахується як одна food-consumption unit. Компенсація нестачі Food у Coins використовує один глобальний конфігураційний курс `coins_per_food`.

`Dt` — одна базова game-time константа для фіксованих просторових і бойових інтервалів: вхід у Region -> Camp, Camp -> вхід у сусідню Region, початок CombatSituation -> Battle Start, підготовка player-vs-player атаки з Camp у Neutral Region, звичайний post-defeat рух до Camp-Regrouping, Regrouping та мінімальний інтервал між послідовними player-vs-player боями в одній Region. Базовий Transit однієї Region також дорівнює `Dt`, але в майбутньому тільки звичайний Transit без CombatSituation може отримати speed coefficient. Attack на Neutral Defense та City Raid відбуваються без `Dt`-черги; поразка від Neutral Defense або City Defense не має окремого post-defeat `Dt` до Regrouping.

У секціях methods:

- **Використовує** — характеристики цієї Model та relationships/характеристики пов'язаних Model, які потрібні method.
- **Викликає** — інші domain methods, якщо вони потрібні. Характеристики іншої Model напряму не змінюються.

---

# Основні game-world models

## 1. `Player`

### Прямі характеристики

- `name`

### Динамічні характеристики

- `coins`
- `gold`
- `silver`

### Обчислювальні характеристики

- `*regions[]` — Region, формальним owner яких є цей Player.
- `*castles[]` — Castle цього Player.
- `*armies[]` — Army цього Player за прямим `Army.player`.
- `*camps[]` — active CampInRegion цього Player за прямим `CampInRegion.player`.
- `coin_balance` — effective rate зміни Coins. Складає region/castle income та upkeep, звичайний `Army.coin_upkeep`, `Army.food_coin_compensation` для Army у Movement, `CampInRegion.food_coin_compensation` поза Castle Region і `Castle.food_coin_compensation` при нестачі Food у Castle.
- `gold_balance` — effective rate зміни Gold; агрегує `Region.gold_income_for_owner` з `regions[]`, тому Occupied/disconnected Region не дають Gold формальному owner.
- `silver_balance` — effective rate зміни Silver; агрегує `Region.silver_income_for_owner` з `regions[]`, тому Occupied/disconnected Region не дають Silver формальному owner.
- `empty_coins` — `coins == 0 && coin_balance <= 0`.

### User methods

- немає зафіксованих на цей момент.

### Domain methods

- `can_pay_global_cost(cost)` — перевіряє одноразову вартість у Coins/Gold/Silver.
- `pay_global_cost(cost)` — списує одноразову глобальну частину вартості після validation.
- `add_coins(amount)` — зараховує разовий Coin reward, зокрема City Raid reward.
- `create_castle(region, name, founder_knight)` — створює новий Castle після завершення Founding; викликає initialization нового `Castle`, `Region.become_castle_region()`, `Knight.change_home_castle()`.

### Triggers

- `empty_coins` `[state trigger]`.

### Trigger methods

- `check_trigger_empty_coins()` — визначає current boolean state `empty_coins` і, якщо `coin_balance < 0`, прогнозує момент досягнення `coins == 0`.
- `on_trigger_empty_coins()` — прямої зміни Player state не робить; boundary потрібний для перерахунку залежних computed rates.

---

## 2. `Castle`

### Прямі характеристики

- `player` — власник Castle.
- `region` — Castle Region.
- `name`
- `levels` — рівні всіх Building Castle як одна структурна характеристика.
- `soldier_reserve` — не призначені Knight Soldier за типами.

### Динамічні характеристики

- `resources` — один клас із `wood`, `stone`, `iron`.
- `food`

### Обчислювальні характеристики

- `*regions[]` — Region, приєднані до цього Castle, включно з Castle Region.
- `*knights[]` — Knight, для яких цей Castle є home Castle.
- `*building_upgrades[]` — BuildingUpgrade цього Castle, включно з history instances; active визначається їх `status`.
- `*recruitment` — active Recruitment Queue Castle або `null`.
- `*knight_replacements[]` — KnightReplacement цього Castle.
- `resource_balance` — effective rate для `wood`, `stone`, `iron`; агрегує позитивні `Region.resource_surplus_to_castle` з `regions[]` і враховує storage boundaries.
- `food_income` — сума позитивних `Region.food_surplus_to_castle` з `regions[]`. Негативний Food balance звичайних Region до Castle не передається.
- `non_military_food_consumption` — Castle-level Food consumption від Population/Building effects.
- `castle_region_army_food_consumption` — `region.camp_food_consumption`; усі Army, що стоять у Castle Region, включно з Regrouping, споживають Food із Castle.
- `food_balance` — `food_income - non_military_food_consumption - castle_region_army_food_consumption`.
- `food_coin_compensation` — якщо `food == 0 && food_balance < 0`, дорівнює `(-food_balance) * coins_per_food`, інакше `0`.
- `coin_balance` — Coin income/upkeep самого Castle.
- `warehouse_capacity`, `granary_capacity`, `storage_full`, `empty_food`.
- `barracks_capacity`, `barracks_used`, `barracks_free_capacity`.
- `governor_capacity`, `external_region_count`.
- `palace_capacity`, `active_knight_replacement_id`.

### Domain methods

- `can_pay_local_cost(cost)`, `pay_local_cost(cost)`.
- `can_house_unit(knight)` — перевіряє, що Knight може мати `location_state = Castle`.
- `move_soldiers_between_reserve_and_knight(knight, composition_delta)` — переносить Soldier тільки `Castle reserve <-> Knight`.
- `add_recruited_soldier(type)`.
- `can_start_building_upgrade(building_type)`, `apply_building_upgrade(building_type, target_level)`.
- `create_knight_for_palace_slot()`, `create_knight_replacement()`, `complete_knight_replacement(replacement)`.
- `can_annex_region(region, camp)` — перевіряє Governor Capacity, допустимий зв'язок Region з цим Castle і наявність у `camp` хоча б одного Knight, чия Army має Camp-presence (`Camp` або `Regrouping`) і чий home Castle дорівнює цьому Castle. Method напряму обходить `CampInRegion -> armies[] -> knights[]`.
- `recalculate_region_connections()` — централізовано перераховує `Region.is_connection_valid` після occupation/loss/restore/annexation і переводить остаточно disconnected Region у Neutral за правилами гри.

### Triggers

- `empty_food` `[state trigger]`.
- `storage_capacity` `[state trigger]`.

### Trigger methods

- `check_trigger_empty_food()`, `on_trigger_empty_food()`.
- `check_trigger_storage_capacity()`, `on_trigger_storage_capacity()`.

---

## 3. `Region`

### Прямі характеристики

- `player` — формальний owner; `null` для Neutral Region.
- `castle` — Castle, до якого Region приєднана; `null`, якщо не належить Castle.
- `occupier_player_id` — ID поточного occupier; `null`, якщо Region не Occupied.
- `resource_sites`.
- `allow_transit` — fallback rule для неагресивного Transit через Owned Region, якщо defender не прийняв ручного рішення до Battle Start.
- `is_connection_valid`.

### Динамічні характеристики

- `neutral_defense` — поточна сила Neutral Defense.

### Обчислювальні характеристики

- `*neighbors[]` — шість сусідніх Region.
- `*armies[]` — Army, що фізично знаходяться в Region за `Army.current_region`.
- `*camps[]` — active CampInRegion цієї Region.
- `*city` — City цієї Region або `null`.
- `*resource_site_upgrades[]`, `*castle_foundings[]`.
- `*combat_situations[]` — CombatSituation цієї Region, включно з queued/active/history instances.
- `camp_player_ids[]` — ID Player, для яких у Region є active CampInRegion.
- `eligible_camp_player_ids[]` — ID Player, чиї CampInRegion мають хоча б одну Army у Camp-presence; `Regrouping` рахується так само, як `Camp` для Annexation/Founding presence.
- `valid_connected_neighbor_player_ids[]`.
- `is_occupied`, `is_castle_region`, `has_any_troops`, `has_active_founding`.
- `food_production`, `camp_food_consumption`, `food_balance`.
- `resource_production`, `gold_production`, `silver_production`.
- `resource_surplus_to_castle`, `food_surplus_to_castle`, `gold_income_for_owner`, `silver_income_for_owner`, `coin_balance_for_owner`.
- `city_wealth_growth_enabled`.
- `neutral_defense_full_strength`, `neutral_defense_recovery_rate`.
- `active_combat_situation` — єдина Active player-vs-player CombatSituation у Region або `null`.
- `next_queued_combat` — найраніше зареєстрована `Registered` CombatSituation у єдиній FIFO-черзі Region. Усі player-vs-player CombatSituation цієї Region, включно з Neutral Region і будь-якими парами Player, використовують одну чергу. Attack на Neutral Defense та City Raid у цю чергу не входять.

### User methods

- `user_change_allow_transit(value)` — змінює fallback Transit rule власної Region.

### Domain methods

- `find_active_camp(player)`, `get_or_create_camp(player)`.
- `register_combat(attacker, registration_reason)` — реєструє player-vs-player CombatSituation. На Registration фіксується attacking Army, Region, причина та порядок реєстрації; Defender Player і potential defenders не фіксуються. Одна CombatSituation завжди має рівно одну attacking Army. Якщо в Region уже є Active або earlier Registered CombatSituation, нова стає в єдину FIFO-чергу Region. Attacker отримує combat lock очікування, але його фізичний стан не змінюється: Camp Army лишається Camp-presence, Movement Army лишається у своєму Movement context з paused progress.
- `start_next_combat_if_possible()` — якщо Active CombatSituation немає, запускає найстарішу Registered CombatSituation Region. Саме цей момент є CombatSituation Start і початком її `Dt` до Battle Start.
- `resolve_arrival(army, movement)` — визначає Camp/Transit context і, коли є одна з registration conditions, реєструє player-vs-player CombatSituation. Army, що входить після CombatSituation Start, не може стати defender цієї CombatSituation. У Neutral Region сам вхід у Camp не запускає combat із Neutral Defense або City Defense.
- `set_occupied_by(player)` — встановлює Occupation. Якщо в Region вже рухаються Army формального owner, які ввійшли до Occupation, але запізно для попередньої defense, для кожної такої Army окремо реєструється CombatSituation проти актуального occupier. Кожна Army є attacker своєї окремої CombatSituation.
- `restore_owner_control(expected_occupier_player)` — Occupation припиняється після виходу останньої Army occupier із Camp.
- `annex_to(player, castle)`, `become_neutral()`, `become_castle_region(new_castle, player)`, `set_connection_valid(value)`.
- `apply_resource_site_upgrade(...)`, `can_start_resource_site_upgrade(...)`.
- `on_army_presence_changed()`.
- `destroy_neutral_defense()`.

### Triggers

- `neutral_defense_recovery_complete` `[event trigger]`.

### Trigger methods

- `check_trigger_neutral_defense_recovery_complete()`.
- `on_trigger_neutral_defense_recovery_complete()`.

---

## 4. `City`

### Прямі характеристики

- `region`

### Динамічні характеристики

- `wealth` — повний довгостроковий економічний потенціал City.
- `active_wealth_ratio` — активна частка Wealth `0..1`; після успішного Raid стає `0` і поступово відновлюється.

### Обчислювальні характеристики

- `wealth_balance` — позитивний тільки коли `active_wealth_ratio == 1` і `region.city_wealth_growth_enabled == true`.
- `active_wealth_ratio_balance` — recovery rate при `active_wealth_ratio < 1`.
- `effective_wealth = wealth * active_wealth_ratio`.
- `city_defense = CityDefense(wealth)` — залежить тільки від повного `wealth`.
- `coin_income` — recurring Coin income для owner Region на основі `effective_wealth`; `0` для Neutral/Occupied.
- `raid_reward` — разовий Coin reward Raid.

### Domain methods

- `complete_raid(player)` — зараховує reward за pre-raid `effective_wealth` і скидає `active_wealth_ratio = 0`; `wealth` і `city_defense` не змінюються.

### Triggers

- `active_wealth_recovered` `[event trigger]`.

### Trigger methods

- `check_trigger_active_wealth_recovered()`.
- `on_trigger_active_wealth_recovered()`.

---

## 5. `Knight`

Knight разом зі своїми Soldier представляє gameplay Unit; окремої backend-моделі Unit немає.

`location_state` описує тільки локальне розміщення Unit у Castle/Barracks або Camp для upkeep/Barracks Capacity. Для movement, combat, Annexation, Founding та інших військових взаємодій місце Unit визначається його Army та `Army.camp`.

### Прямі характеристики

- `name`
- `castle` — home Castle.
- `army` — поточна Army.
- `soldiers` — кількість Soldier за Type.
- `base_strength`.
- `location_state` — `Castle` або `Camp`.
- `status`.

### Динамічні характеристики

- `experience`

### Обчислювальні характеристики

- `soldier_count`.
- `food_consumption = soldier_count + 1` для active Knight.
- `experience_coefficient`.
- `unit_attack_strength`, `unit_defense_strength`.
- `coin_upkeep`.
- `current_region` — `army.current_region`.
- `current_camp` — `army.camp`.
- `is_regrouping` — чи `army.state == Regrouping`.
- `is_camp_presence` — `current_camp != null && army.state in {Camp, Regrouping}`.
- `experience_balance`.

### User methods

- `user_enter_castle()`, `user_leave_castle()`.
- `user_change_unit_composition(composition_delta)` — дозволено тільки у home Castle і коли Army Knight не `is_command_locked` жодною CombatSituation.

### Domain methods

- `set_location_state(state)` — `Castle` дозволений тільки при Camp-presence у home Castle Region.
- `leave_castle_for_movement()`.
- `set_soldiers(new_composition)`, `set_army(army)`.
- `add_battle_experience(amount)`, `apply_casualties(...)`, `die()`, `change_home_castle(new_castle)`.

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
- `target_opponent` — [цільовий суперник] поточного campaign/movement intent; Player або `null`. При створенні нового маршруту визначається за станом його final Region: Neutral -> `null`; Owned non-Occupied -> formal owner; Owned Occupied -> occupier. Не змінюється автоматично через подальшу зміну стану final Region.
- `pending_target_opponent` — конкретне нове значення [цільового суперника], обчислене й зафіксоване в момент натискання Player «Оновити ЦС»; може бути Player або `null`. Подальші зміни стану final Region це pending value не змінюють.
- `has_pending_target_opponent_update` — окремий boolean, що відрізняє «pending update відсутній» від валідного `pending_target_opponent = null`. При вході Army в наступну Region, якщо `true`, виконується `target_opponent = pending_target_opponent`, після чого flag скидається.
- `state` — `Camp / Movement / Regrouping` та потрібні V1 підстани. `Regrouping` вважається Camp-presence для всіх правил, крім заборони самій Army починати Movement або Attack. Очікування queued CombatSituation не є окремим фізичним state: Camp attacker залишається `Camp`, Movement attacker залишається в Movement context з paused progress.

### Динамічні характеристики

- `regrouping_progress`.

### Обчислювальні характеристики

- `*knights[]`.
- `*movement` — active Movement або `null`; якщо Army є queued attacker (`is_combat_waiting == true`), Movement може зберігати незавершений route/context, але progress призупинений.
- `*attacking_combat_situation` — незавершена CombatSituation, де ця Army є attacker, або `null`; invariant: максимум одна.
- `*active_defender_combat_situation` — Active CombatSituation, де Army входить у зафіксований список potential defenders, або `null`.
- `is_combat_waiting` — `attacking_combat_situation.status == Registered`. Це command-lock, а не фізичний location/state.
- `soldier_count`, `food_consumption`.
- `attack_strength`, `defense_strength`.
- `coin_upkeep`.
- `food_coin_compensation` — повна Food-компенсація в Coins для Movement; Camp/Regrouping споживають local Food. Attacker, що чекає queued CombatSituation з Camp, продовжує local Food consumption; attacker, що чекає з Movement, зберігає Movement food semantics.
- `regrouping_progress_rate` — так підібраний, щоб Regrouping тривав рівно `Dt`.
- `is_command_locked` — true для attacker будь-якої незавершеної CombatSituation та для potential defender Active CombatSituation; блокує Movement, merge/split, Commander/unit-composition reorganization та інші commands, які могли б змінити participant state. Combat-specific decisions залишаються доступними.

### User methods

- `user_merge_armies(armies, commander, thresholds)` — дозволено Army одного Player в одному CampInRegion, включно з Regrouping, якщо жодна з них не `is_command_locked`. Regrouping саме по собі merge/split не забороняє.
- `user_split_army(groups)` — ті самі restrictions.
- `user_change_commander(knight)` — дозволено, якщо Army не `is_command_locked` жодною CombatSituation; Regrouping саме по собі не забороняє зміну Commander.
- `user_change_combat_thresholds(values)` — змінює persistent thresholds, якщо вони не lock-нуті поточною battle phase; CombatSituation-specific defender decision має пріоритет для конкретного бою.
- `user_attack_player(target_camp)` — player-vs-player attack із Camp у Neutral Region. Дозволено тільки `state == Camp` (не Regrouping), target Camp іншого Player у тій самій Neutral Region, Army не attacker іншої незавершеної CombatSituation і не potential defender active CombatSituation. Реєструє окрему CombatSituation з цією Army як єдиним attacker і одразу фіксує `defender_player = target_camp.player`.
- `user_raid_city(city)` — City Raid тільки для `state == Camp`. Заборонено, якщо Player цієї Army у цій Region є attacker будь-якої незавершеної CombatSituation або potential defender Active CombatSituation. Raid не входить у player-vs-player CombatSituation queue і відбувається миттєво.
- `user_attack_neutral_defense()` — миттєва атака Neutral Defense цієї Region конкретною Army у `state == Camp`; Regrouping не може її ініціювати. Заборонено, якщо Player цієї Army у цій Region є attacker будь-якої незавершеної CombatSituation або potential defender Active CombatSituation. Якщо в Region є City, до abstract Defender strength додається full City Defense. Ця дія не створює CombatSituation і не входить у FIFO-чергу Region.

### Domain methods

- `enter_region(region)` — фіксує фізичний вхід Army у Region.
- `enter_camp(camp)` — переводить Army у Camp і встановлює `camp`.
- `leave_camp()` — очищує `camp`; якщо це остання Army occupier у Camp, Occupation припиняється в цей момент.
- `start_movement(movement)` — дозволено тільки якщо Army не Regrouping і не command-locked CombatSituation.
- `finish_movement()`.
- `set_combat_waiting(combat)` — застосовує command-lock queued attacker без зміни фізичного state. Для Camp Army `camp` і Camp-presence зберігаються; для Movement Army Movement переходить у paused-for-combat state без втрати route/context.
- `clear_combat_waiting()` — при CombatSituation Start/termination знімає waiting-lock; подальший Movement/Camp context визначається поточною CombatSituation.
- `start_regrouping()` — переводить Army у Regrouping і скидає progress. Camp зберігається; Army продовжує бути звичайною Camp-presence для Food, Annexation, Founding, Occupation, defense та Camp lifecycle.
- `finish_regrouping()` — повертає Army у Camp після `Dt`.
- `start_retreat_to(region)` — звичайний player-vs-player Retreat: Army одразу вважається такою, що увійшла в retreat Region, після чого `retreat-local` триває `Dt` до Camp-Regrouping.
- `start_loss_regrouping_in_current_camp()` — спеціальний результат поразки від Neutral Defense або City Defense: без Retreat у сусідню Region і без окремого post-defeat `Dt`; Army лишається/переходить у той самий Camp і одразу починає Regrouping.
- `remove_dead_knights()`, `dissolve_after_commander_death()`.

### Triggers

- `regrouping_complete` `[event trigger]`.

### Trigger methods

- `check_trigger_regrouping_complete()` — прогнозує `Dt` completion.
- `on_trigger_regrouping_complete()` — завершує Regrouping.

---

# Persistent interaction/process models

## 7. `CombatSituation`

`CombatSituation` у цій секції — одна модель для всіх player-vs-player interaction з registration/start/active lifecycle; відмінності між registration reasons реалізуються умовною domain logic всередині цієї моделі. Окремі inherited submodels поки не вводяться. Attack на Neutral Defense та City Raid використовують combat calculation, але не входять у player-vs-player queue і не чекають `Dt`.

Одна CombatSituation завжди має рівно одну attacking Army. Дві Army одного Player, які послідовно створюють умови атаки в одній Region, реєструють дві окремі CombatSituation; вони не об'єднуються в одну attacking side і не є reinforcement одна одній.

### Прямі характеристики

- `region`.
- `attacker` — єдина attacking Army; фіксується на Registration.
- `registration_reason` — entry into foreign Owned Region / entry into own occupied Region / own Region became occupied around already-present late Army / explicit Neutral Camp attack.
- `registration_order` — FIFO order у єдиній player-vs-player черзі Region.
- `status` — `Registered / Active / Resolved`.
- `start_time` — встановлюється тільки при CombatSituation Start; `null` для queued Registered.
- `defender_player` — для explicit Neutral Camp attack фіксується на Registration як Player обраного target Camp; для територіальних registration reasons лишається `null` до Start і визначається за актуальним ownership/occupation interaction.
- `potential_defenders[]` — `null/empty` до Start; у момент Start фіксуються Camp/Regrouping/entered-for-Camp Army defender Player за їх актуальним станом.
- `transit_defender_candidates[]` — Army defender Player, які ввійшли в Region для Transit до Start і на Start ще не вийшли. Вони не беруть участі автоматично й не блокуються як potential defenders, доки явно не оберуть залишитися для defense.
- `transit_defender_join_decisions` — явні рішення pre-Start Transit candidates залишитися для defense; після такого рішення Army приєднується до participating defenders і отримує combat command-lock.
- `defenders_retreat_decisions` — рішення Retreat для окремих potential defender Army; рішення не перегруповує Army і не змінює її Commander/composition.
- `defender_combat_threshold` — CombatSituation-specific Defense Loss Threshold для Army, що залишаються для бою, якщо Player задав його в pre-battle dialog.
- `transit_decision` — `null / allow / fight` для non-aggressive Transit; після ручного вибору не змінюється.
- `attacker_destination_mode` — `Camp | Transit`; походить із початкового Movement task для входу в цю Region і не змінюється queueing.
- `target_opponent` — snapshot `Army.target_opponent`, чинного при вході attacker у Region / Registration цієї CombatSituation. Подальше ручне refresh ЦС не змінює вже зареєстровану CombatSituation.
- `transit_mode` — `null` до CombatSituation Start; на Start для Transit один раз визначається за зафіксованим `target_opponent` та актуальним на Start станом current Region: Aggressive тільки якщо Region формально належить `target_opponent` і не Occupied; в усіх інших випадках NonAggressive.
- `locked_combat_parameters` — остаточно фіксуються на Battle Start.

### Динамічні характеристики

- `battle_start_progress` — рахується тільки для Active CombatSituation; від Start до Battle Start завжди `Dt`.

### Обчислювальні характеристики

- `is_queued` — `status == Registered` і ця CombatSituation ще не стала Active; у Region перед нею є Active або earlier Registered CombatSituation.
- `is_active` — `status == Active`.
- `battle_start_progress_rate` / `required_battle_start_progress` — дають рівно `Dt` від Start до Battle Start.
- `is_battle_valid` — чи після pre-battle resolution лишилися умови реального battle.
- `attacker_loss_threshold` — якщо фактичний `defender_player == target_opponent`, використовується `attacker.target_combat_threshold`; для бою з будь-яким іншим Player використовується `attacker.incidental_combat_threshold`. Це правило діє в будь-якій Region, включно з final Region F, якщо фактичний opponent там відрізняється від зафіксованого ЦС. Значення lock-иться на Battle Start.
- `defender_loss_threshold` — один CombatSituation-specific Defense Loss Threshold для всіх participating defender Army. Він не є per-Army threshold.
- `attacker_strength` — для Neutral Camp player-vs-player combat використовується Attack strength.
- `defender_strength` — для Neutral Camp player-vs-player combat також використовується Attack strength; для invasion/occupation defense — відповідний звичайний defense mode.

### Registration

CombatSituation реєструється, коли:

- Army Player A входить в Owned Region Player B;
- Army Player A входить у свою Region, яка на цей момент Occupied іншим Player;
- Region Player A стає Occupied, коли Army A вже рухається в цій Region, увійшла туди до Occupation, але вже не може приєднатися до попередньої defense;
- Player A і B мають Camp-presence в одній Neutral Region і A через конкретну Army ініціює attack на B.

На Registration фіксується `attacker`, `target_opponent` snapshot, `attacker_destination_mode` та registration metadata. Для explicit Neutral Camp attack також одразу фіксується конкретний `defender_player`, якого Player атакував. Для територіальних registration reasons `defender_player` на Registration не фіксується. `potential_defenders[]` і `transit_defender_candidates[]` формуються тільки на Start; battle thresholds lock-яться пізніше.

Якщо в Region немає Active/earlier Registered CombatSituation, Start відбувається одразу. Інакше situation лишається Registered у єдиній FIFO-черзі Region. Attacker отримує waiting command-lock без зміни фізичного state: Camp attacker лишається Camp-presence; Movement attacker зберігає Movement context із paused progress. Нові player-vs-player situations стають у чергу виключно за `registration_order`; пріоритетів і «незалежних» паралельних PvP situations у Neutral Region немає.

### Start

CombatSituation Start — це:

- момент Registration, якщо в Region немає Active або earlier Registered CombatSituation;
- момент завершення або дострокового завершення попередньої Active CombatSituation, коли ця situation стала першою у FIFO-черзі Region.

На Start за актуальним станом Region:

- для територіальних registration reasons визначається фактичний `defender_player` за current ownership/occupation interaction;
- для explicit Neutral Camp attack використовується зафіксований на Registration `defender_player` і перевіряється, чи interaction з ним досі актуальний; situation не може перенаправитися на іншого Player;
- для Transit classification береться зафіксований `target_opponent` і current ownership/occupation Region;
- фіксуються `potential_defenders[]` і `transit_defender_candidates[]`;
- починається повний pre-battle interval `Dt`;
- defender отримує відповідний decision dialog.

Якщо explicit Neutral Camp attack на Start більше не актуальний щодо зафіксованого `defender_player`, CombatSituation видаляється без battle, attacker lock знімається, після чого Region може Start-нути наступну Registered situation. Для територіальних registration reasons, якщо на Start Region стала Neutral або немає Player, проти якого за актуальними правилами має відбутися interaction, CombatSituation завершується без battle; attacker продовжує попередній Movement route. Жодна queued situation не завершується або не видаляється раніше власного Start лише через те, що майбутні умови, ймовірно, зникли.

### Target opponent and Aggressive Transit

При створенні маршруту для Army фіксується [цільовий суперник]:

- final Region F Neutral -> `target_opponent = null`;
- F належить Player B і не Occupied -> `target_opponent = B`, незалежно від наявності військ B у F;
- F належить Player B, але Occupied Player C -> `target_opponent = C`.

Подальші зміни ownership/occupation F самі по собі ЦС не змінюють. Player може вручну запитати refresh ЦС за актуальним станом F; нове значення набуває чинності тільки при вході Army в наступну Region маршруту і не змінює CombatSituation, уже зареєстровану в поточній Region.

Для Transit через конкретну Region:

- якщо Region формально належить `target_opponent` і не Occupied -> `Aggressive`;
- у всіх інших випадках, включно з Region іншого Player або Occupied Region -> `NonAggressive`.

`attacker_destination_mode` (`Camp` або `Transit`) як і раніше визначається початковим Movement task і не переобчислюється CombatSituation.

### Potential defenders

Potential defender фіксується тільки на Start CombatSituation.

Potential defenders на Start:

- Army defender Player у `Camp`;
- Army defender Player у `Regrouping`;
- Army defender Player, яка до Start уже ввійшла в Region з локальною метою Camp. Оскільки entry -> Camp і Start -> Battle Start обидва завжди дорівнюють `Dt`, така Army гарантовано досягне Camp не пізніше Battle Start.

Окремо Army defender Player, яка ввійшла в Region для Transit **до Start** і на Start ще не вийшла з Region, потрапляє в `transit_defender_candidates[]`, а не в `potential_defenders[]`. Поки вона не обрала defense, вона не command-locked цією CombatSituation і продовжує Transit. До фактичного виходу з Region Player може явно наказати їй залишитися; тоді вона додається до participating defenders і від цього моменту отримує combat command-lock. Якщо такого рішення немає до виходу, Army продовжує Transit і більше не може приєднатися.

Не може брати участь:

- Army, яка до Start уже вийшла з Camp і рухається до межі Region;
- будь-яка Army defender Player, яка ввійшла в Region після Start.

Army, яка входить після Start, ніколи не reinforcement цієї CombatSituation. Її подальша поведінка визначається її route та станом Region після завершення поточного battle; якщо вона сама створює умови атаки, реєструється окрема CombatSituation.

Після Start potential defender Army не може отримати Movement command, merge/split, зміну Commander або іншу реорганізацію до завершення CombatSituation. Player може тільки залишити конкретну Army для бою або наказати їй Retreat. Retreat застосовується за результатом/завершенням CombatSituation, а не миттєво в момент рішення; pre-battle Retreat не завдає втрат і після нього Army проходить Regrouping.

### No-battle resolution for Transit

Якщо attacker має `destination_mode = Transit` і на Start немає жодної Army defender Player у Camp-presence або вже entered-for-Camp state, CombatSituation завершується без battle. Самі по собі defender Transit Army, навіть якщо вони ввійшли до Start, не створюють battle context: attacker і ці defender Transit Army продовжують рух, а defender Transit Army не отримують можливості зупинитися для interception.

Якщо Transit `NonAggressive` і potential defenders є, defender протягом `Dt` може одноразово обрати `Allow` або `Fight`:

- `Allow` — situation завершується без battle, attacker продовжує Transit, усі defender Army лишаються у своїх поточних states;
- `Fight` — situation лишається active до Battle Start;
- якщо ручного рішення немає до Battle Start, `Region.allow_transit` є fallback: `true` -> no battle, `false` -> battle.

### Default defender behavior at Battle Start

Якщо battle має відбутися і defender не задав окремих рішень:

- Transit Army, що ввійшли до Start і отримали можливість приєднатися до вже існуючого battle context, за відсутності явного рішення залишитися не беруть участі й продовжують Transit;
- Camp/Regrouping/entered-for-Camp potential defender Army залишаються для battle;
- для всіх Army, що беруть участь у defense, застосовується один спільний `defender_combat_threshold` цієї CombatSituation.

Player може до Battle Start вибрати Retreat для всіх potential defender Army або для окремих Army. Army як контейнери не перегруповуються: заборонені merge/split, transfer Unit/Soldier і зміна Commander. Defense Loss Threshold задається один раз для CombatSituation і спільний для всіх defender Army, що залишаються в battle.

### Neutral Camp attack and attack on occupier

Explicit attack A -> B між Camp у Neutral Region на Registration фіксує `defender_player = B`. На Start ця situation або лишається атакою саме проти B, або, якщо interaction з B вже не актуальний, видаляється без battle; вона ніколи не перенаправляється на C чи іншого Player. Army B, що entered-for-Camp до Start, є potential defender. Army B, що entered-for-Transit до Start і ще не вийшла з Region, є transit defender candidate та може явно залишитися для defense до моменту свого виходу; leaving-Camp Army участі не бере. Army B, що входить після Start, не може приєднатися.

Ті самі defender eligibility та command-lock rules застосовуються до Army occupier, коли їх атакує formal owner Region або третій Player.

### Queue

У межах кожної Region існує одна спільна FIFO-черга всіх player-vs-player CombatSituation. Це правило однакове для Owned, Occupied і Neutral Region та не залежить від того, які пари Player беруть участь. Одночасно Active може бути максимум одна player-vs-player CombatSituation Region.

Наступна Registered situation не стартує до завершення або дострокового завершення попередньої Active situation. Після її завершення найстаріша Registered situation Start-ує одразу і отримує повний `Dt` до свого Battle Start. Таким чином між послідовними battle у цій Region не може бути менше `Dt`.

Queued situation не має defender Player або potential defenders. Її attacker має waiting command-lock, але зберігає фізичний Camp/Movement context; навіть звичайний Transit може бути затриманий queueing.

Attack на Neutral Defense та City Raid не створюють CombatSituation і не входять у цю queue.

Player не може атакувати Neutral Defense або City в цій Region, якщо він є attacker будь-якої незавершеної CombatSituation у цій Region або potential defender Active CombatSituation у цій Region.

### User methods

- `user_set_transit_decision(decision)` — `Allow/Fight` для Active NonAggressive Transit; після вибору рішення immutable.
- `user_set_pre_battle_retreat_decision(army, retreat)` — задає Retreat для конкретної potential defender Army до Battle Start.
- `user_join_defense_from_transit(army)` — для Army з `transit_defender_candidates[]` до її фактичного виходу з Region фіксує рішення залишитися для defense, додає її до participating defenders і застосовує combat command-lock.
- `user_set_defender_combat_threshold(value)` — один threshold для всіх defender Army, що залишаються для battle.

### Domain methods

- `start()` — переводить Registered situation в Active, визначає актуального defender Player, transit mode, potential defenders і запускає `Dt` до Battle Start. Якщо interaction більше не існує, завершує situation без battle та відновлює attacker Movement.
- `determine_transit_mode()` — для Transit повертає Aggressive, якщо current Region формально належить зафіксованому `target_opponent` і не Occupied; інакше NonAggressive.
- `collect_potential_defenders()` — на Start окремо фіксує `potential_defenders[]` для Camp/Regrouping/entered-for-Camp та `transit_defender_candidates[]` для pre-Start Transit; Army, які входять після Start, не додаються.
- `resolve_pre_battle()` — застосовує Transit allow/fight та Retreat/default decisions.
- `lock_combat_parameters()` — на Battle Start фіксує actual participating defenders, thresholds, strengths, retreat destinations та інші battle parameters.
- `get_legal_retreat_regions(army, role)`, `select_retreat_region(army, role)`.
- `resolve_combat()`, `apply_casualties(result)`.
- `apply_result(result)` — виконує Retreat, Occupation, Camp/Transit continuation. Якщо attacker мав Transit, будь-який result не змушує defender Army, що ввійшли після Start, змінювати свій route; вони завершують свій planned movement після поточного battle. Якщо attacker мав Camp і програв — так само. Якщо attacker мав Camp, виграв і встановив Occupation, Army formal owner, що входять пізніше, при актуальному arrival реєструють власні окремі CombatSituation проти occupier.
- `finish_without_battle()` — завершує situation і відновлює Movement/стани без battle.
- `delete_if_obsolete_explicit_attack()` — тільки для explicit Neutral Camp attack на Start: якщо interaction із зафіксованим `defender_player` більше не актуальний, видаляє situation без battle, знімає attacker lock і передає чергу наступній Registered CombatSituation.
- `finish()` — terminal transition; після цього Region може Start-нути наступну Registered CombatSituation у FIFO-черзі.

### Triggers

- `battle_start` `[event trigger]` тільки для Active player-vs-player CombatSituation.

### Trigger methods

- `check_trigger_battle_start()` — прогнозує Start + `Dt`.
- `on_trigger_battle_start()` — спочатку виконує pre-battle resolution; якщо situation закінчується без battle, викликає `finish_without_battle()`. Інакше lock-ає parameters і resolve-ить battle.

### Combat calculation outside `CombatSituation`

Neutral Defense attack і City Raid не є `CombatSituation`. Це миттєві Army actions, які використовують спільний combat-calculation mechanism, але не створюють lifecycle/queue object і не мають Battle Start delay.

- Neutral Defense destruction: attacker — одна конкретна Camp Army; Defender strength = current Neutral Defense + full City Defense, якщо City є; defender threshold = `1.0`; attacker використовує `target_combat_threshold`. При fail Neutral Defense повертається/лишається на pre-combat current value, City Defense незмінна. При success Neutral Defense = `0`.
- City Raid: attacker — одна конкретна Camp Army; Defender = City Defense; fixed raid threshold; attacker використовує `target_combat_threshold`; success викликає `City.complete_raid()`. При defeat Army не retreat-ить у сусідню Region: вона лишається в тому самому Camp і одразу входить у Regrouping на `Dt`.

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
- `food_consumption` — сума `Army.food_consumption` усіх `armies[]`, включно з Regrouping.
- `has_eligible_presence` — чи є хоча б одна Army з `state in {Camp, Regrouping}`.
- `food_coin_compensation` — `0` у Castle Region; поза Castle Region пропорційна частка локального Food deficit.
- `has_valid_adjacent_owned_region`.
- `can_progress` — Annexation progress можливий за ownership/Neutral Defense/adjacency rules і тільки за відсутності blocking foreign Camp-presence: foreign Army у `Camp`/`Regrouping` або foreign Army, що вже ввійшла в Region з локальною метою Camp, блокує progress. Foreign Transit Army не блокує Annexation, навіть якщо через її Transit у Region існує CombatSituation. Саме така foreign Camp/Camp-bound presence, а не наявність CombatSituation, призупиняє progress. Regrouping Army свого Player підтримує progress так само, як Camp Army.
- `control_progress_rate`, `required_control_progress`, `is_ready_for_annexation`.

### User methods

- `user_annex_region(castle)`.

### Domain methods

- `accept_army(army)`.
- `on_army_left(army)` — якщо Camp спорожнів, деактивує instance; якщо Player є occupier, вихід останньої Army з Camp припиняє Occupation.
- `get_defending_armies()` — повертає Camp/Regrouping Army; остаточний potential-defender set для конкретної CombatSituation фіксує сама CombatSituation на Start і також враховує entered-for-Camp Army.
- `validate_reorganization(armies)` — дозволяє Camp і Regrouping Army, якщо вони не command-locked CombatSituation. Regrouping саме по собі merge/split не блокує.
- `can_annex_to(castle)`.

### Triggers

- `can_progress` `[state trigger]`.
- `ready_for_annexation` `[event trigger]`.

### Trigger methods

- `check_trigger_can_progress()`, `on_trigger_can_progress()`.
- `check_trigger_ready_for_annexation()`, `on_trigger_ready_for_annexation()`.

---

## 9. `CastleFounding`

### Прямі характеристики

- `player`.
- `region`.
- `founder_knight`.
- `castle_name`.
- `status`.

### Динамічні характеристики

- `progress`.

### Обчислювальні характеристики

- `founder_valid` — founder Knight живий, фізично має потрібний current Camp, належить Player і не має Soldier. Leave Region/death/отримання Soldier скасовує process.
- `founder_progress_eligible` — `founder_valid && founder_knight.is_camp_presence`; Regrouping не pause-ить Founding.
- `can_progress` — `founder_progress_eligible`; для Neutral Region `neutral_defense == 0`; у Region немає blocking foreign Camp-presence: foreign Army у `Camp`/`Regrouping` або foreign Army, що вже ввійшла з локальною метою Camp. Foreign Transit Army не блокує Founding, навіть якщо через її Transit у Region існує CombatSituation. Саме foreign Camp/Camp-bound presence, а не наявність CombatSituation, призупиняє Founding.
- `progress_rate`, `required_progress`, `is_complete`.

### User methods

- `user_start_castle_founding(...)` — founder може бути у Camp або Regrouping, якщо всі інші умови виконані.

### Domain methods

- `validate_start()`.
- `cancel()` — terminal state без refund при founder invalid.
- `complete()` — створює Castle через `Player.create_castle()`.

### Triggers

- `can_progress` `[state trigger]`.
- `complete` `[event trigger]`.

### Trigger methods

- `check_trigger_can_progress()`, `on_trigger_can_progress()`.
- `check_trigger_complete()`, `on_trigger_complete()`.

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
- [цільовий суперник] зберігається на Army як `Army.target_opponent`; Movement визначає/оновлює його за final Region маршруту.

### Динамічні характеристики

- `progress`.

### Обчислювальні характеристики

- `progress_rate` — `0`, якщо Movement paused через queued/Active CombatSituation interaction. Для Camp-exit -> next Region entry та entry -> Camp фаза завжди має тривалість `Dt`. Для звичайного Transit без CombatSituation базова тривалість `Dt`, потенційно з майбутнім movement-speed coefficient.
- `direction_reveal_progress`.
- `phase_completion_progress`.

### User methods

- `user_start_movement(army, route[])` — заборонено для Regrouping або command-locked Army. При створенні маршруту визначає `Army.target_opponent` за актуальним станом final Region F: Neutral -> `null`; Owned non-Occupied -> formal owner; Owned Occupied -> occupier.
- `user_change_planned_route(route[])` — змінює тільки future route, коли Army не locked CombatSituation; сама зміна Route не переобчислює `Army.target_opponent`.
- `user_refresh_target_opponent()` — у момент явної дії Player одразу обчислює [цільового суперника] за актуальним станом поточної final Region F і фіксує отримане значення в `pending_target_opponent`, виставляючи `has_pending_target_opponent_update = true`. До входу в наступну Region це значення не переобчислюється, навіть якщо F зміниться. Воно не впливає на current Region або вже зареєстровану CombatSituation.

### Domain methods

- `validate_route(route[])` — перевіряє adjacency і фактичну доступність future route; перед реальним border entry наступна Region перевіряється знову, тому Castle Region, створена після planning, блокує майбутній entry. Grandfathering стосується тільки Army, яка вже фізично знаходилася всередині Region на момент створення Castle Region.
- `start_phase(...)`.
- `start_retreat_local(...)` — post-defeat local movement до Camp триває `Dt`.
- `determine_target_opponent(final_region)` — повертає `null` для Neutral final Region, formal owner для non-Occupied Owned final Region, occupier для Occupied final Region.
- `pause_for_combat_waiting()` / `resume_after_combat_waiting()` — зберігають route/context без просування progress; не змінюють фізичний state Camp Army, якщо queued combat було ініційовано з Camp.
- `advance_to_next_region()` — безпосередньо при border entry, якщо `has_pending_target_opponent_update == true`, присвоює вже зафіксоване `target_opponent = pending_target_opponent` без нового обчислення за станом final Region, після чого скидає pending flag і викликає `Region.resolve_arrival()`. Тому новий [цільовий суперник] використовується вже для interaction у щойно введеній Region. Arrival після Start чужої Active CombatSituation не додає Army до її defenders.
- `resolve_camp_arrival()`.
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

## 11. `Recruitment`

Один Castle має одну послідовну Recruitment Queue.

### Прямі характеристики

- `castle`.
- `queue` — orders `{soldier_type, remaining_quantity}`.
- `current_order_index`.
- `status`.

### Динамічні характеристики

- `progress`.

### Обчислювальні характеристики

- `current_order`.
- `can_progress` — є current order, `castle.empty_food == false`, `castle.player.empty_coins == false`, `castle.barracks_free_capacity > 0`.
- `progress_rate`, `required_progress`, `current_recruit_finished`.

### User methods

- `user_add_recruitment_order(soldier_type, quantity)`.

### Domain methods

- `validate_new_order(type, quantity)`.
- `finish_current_recruit()`.

### Triggers

- `can_progress` `[state trigger]`.
- `current_recruit_finished` `[event trigger]`.

### Trigger methods

- `check_trigger_can_progress()`, `on_trigger_can_progress()`.
- `check_trigger_current_recruit_finished()`, `on_trigger_current_recruit_finished()`.

---

## 12. `BuildingUpgrade`

Один instance = один process upgrade однієї Building.

### Прямі характеристики

- `castle`, `building_type`, `target_level`, `status`.

### Динамічні характеристики

- `progress`.

### Обчислювальні характеристики

- `progress_rate`, `required_progress`, `is_complete`.

### User methods

- `user_start_building_upgrade(castle, building_type)`.

### Domain methods

- `complete()` — застосовує target level.

### Triggers / Trigger methods

- `complete` `[event trigger]`.
- `check_trigger_complete()`, `on_trigger_complete()`.

---

## 13. `ResourceSiteUpgrade`

Один instance = один process upgrade вибраної кількості ResourceSite одного resource type/level.

### Прямі характеристики

- `region`, `resource_type`, `target_level`, `quantity`, `status`.

### Динамічні характеристики

- `progress`.

### Обчислювальні характеристики

- `progress_rate`, `required_progress`, `is_complete`.

### User methods

- `user_start_resource_site_upgrade(region, resource_type, quantity)`.

### Domain methods

- `complete()` — застосовує level changes.

### Triggers / Trigger methods

- `complete` `[event trigger]`.
- `check_trigger_complete()`, `on_trigger_complete()`.

---

## 14. `KnightReplacement`

Один instance = один process створення replacement Knight для звільненого Palace slot.

### Прямі характеристики

- `castle`, `status`.

### Динамічні характеристики

- `progress`.

### Обчислювальні характеристики

- `is_queue_head`.
- `progress_rate` — ненульовий тільки для queue head.
- `required_progress`, `is_complete`.

### Domain methods

- `complete()` — створює replacement Knight.

### Triggers / Trigger methods

- `complete` `[event trigger]`.
- `check_trigger_complete()`, `on_trigger_complete()`.
