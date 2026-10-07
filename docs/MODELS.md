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

Обчислювальна характеристика, позначена `*` перед назвою, є **system-computed**: її актуальне значення не обчислюється прямо в рамках game logic самої Model, а надається infrastructure. Це не окремий persistence type: як і інші computed characteristics, system-computed значення зберігається при `commit()`. Типові приклади — reverse relationships та топологічні зв'язки карти. `[]` у назві означає колекцію; для system-computed relationship це список посилань на Model, для звичайної computed characteristic це може бути список scalar values.

Для computed characteristics діє правило: якщо `A.computed` потрібне значення з `A -> B -> C`, відповідна інформація зазвичай піднімається в computed characteristic Model `B`, а `A` читає вже її. Прямий system-computed зв'язок `A -> C` додається лише коли він сам є природною характеристикою `A`.

Це обмеження не поширюється на domain/user methods: method може обходити потрібні relationships і читати характеристики пов'язаних Model без створення проміжних computed characteristics лише заради такого обходу.

Рівні всіх Building одного Castle зберігаються як одна характеристика `levels`.

Wood / Stone / Iron зберігаються як одна характеристика `resources` — один value object / helper class із трьома значеннями.

Food consumption одного Soldier не залежить від Soldier Type. Один Knight також рахується як одна food-consumption unit. Компенсація нестачі Food у Coins використовує один глобальний конфігураційний курс `coins_per_food`.

`Dt` — одна базова game-time константа для фіксованих просторових і бойових інтервалів: вхід у Region -> Camp, Camp -> вхід у сусідню Region, початок CombatSituation -> Battle Start (включно з player-vs-player атакою з Camp у Neutral Region; це не окремий додатковий interval), звичайний player-vs-player Retreat рух до Camp-Regrouping (включно з pre-battle Retreat), Regrouping та мінімальний інтервал між послідовними player-vs-player боями в одній Region. Базовий Transit однієї Region також дорівнює `Dt`, але в майбутньому тільки звичайний Transit без CombatSituation може отримати speed coefficient. Attack на Neutral Defense та City Raid відбуваються без `Dt`-черги; поразка від Neutral Defense або City Defense не має окремого post-defeat `Dt` до Regrouping.

У секціях methods:

- **Використовує** — характеристики цієї Model та relationships/характеристики пов'язаних Model, які потрібні method.
- **Викликає** — інші domain methods, якщо вони потрібні. Характеристики іншої Model напряму не змінюються.

## Player-specific projections

Між backend game-world models і frontend вводиться окремий шар **player-specific projections**. Frontend не отримує domain models напряму: projection збирає з них інформацію, доступну конкретному Player, застосовує правила видимості/доступу та є джерелом даних і оновлень для frontend subscriptions.

Конкретний набір projection models, їх characteristics, lifecycle, dependencies та формат frontend subscriptions/updates буде визначено окремо під час проєктування frontend/API.

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
- `create_castle(region, name, founder_knight)` — створює новий Castle після завершення Founding: ініціалізує `Warehouse = 1`, `Granary = 1`, `Palace = 1`, викликає `Region.become_castle_region()` і `Knight.change_home_castle()`. Founder займає початковий Palace slot нового Castle; додатковий Knight при створенні Castle не генерується.

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
- `player_empty_coins` — `player.empty_coins`; проміжна характеристика для залежних computed characteristics інших Model без ланцюга relationships.
- `warehouse_capacity`, `granary_capacity`, `storage_full`.
- `empty_food` — `food == 0 && food_balance <= 0`.
- `barracks_capacity`.
- `barracks_used` — кількість/occupancy Unit, чиї Knight мають `location_state = Castle` у цьому home Castle.
- `barracks_free_capacity = barracks_capacity - barracks_used`.
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
- `recalculate_region_connections()` — централізовано перераховує `Region.is_connection_valid` після occupation/loss/restore/annexation. Temporary disconnect через Occupation лише робить downstream Region invalid-connected; у Neutral вони переходять тільки після остаточної втрати ownership bridge Region.

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
- `allow_transit` — fallback rule для неагресивного Transit тільки через Owned **non-Occupied** Region, якщо defender не прийняв ручного рішення до Battle Start. Для Occupied Region ця rule не застосовується: Transit через Occupied Region не створює CombatSituation.
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
- `camp_player_ids[]` — `CampInRegion.player_id` для active CampInRegion цієї Region.
- `eligible_camp_player_ids[]` — `CampInRegion.player_id` для CampInRegion, що мають `has_eligible_presence == true`; `Regrouping` рахується так само, як `Camp`.
- `blocking_camp_presence_player_ids[]` — `Army.player_id` для Player, які мають у Region хоча б одну Army у `Camp`/`Regrouping` або Army з `is_entered_for_camp == true`. Саме ця характеристика використовується для перевірки foreign Camp/Camp-bound presence в Annexation і Castle Founding; чистий Transit сюди не входить.
- `player_id` — ID `player`; проміжна scalar characteristic для computed characteristics інших Model без другого relationship hop.
- `valid_connected_neighbor_player_ids[]` — `player_id` сусідніх Owned Region з валідним connection; використовує `neighbors[].player_id`, а не `neighbors[] -> player`.
- `is_occupied`, `is_castle_region`, `has_active_founding`.
- `has_any_troops` — чи є в Region хоча б одна фізично присутня Army незалежно від її Camp/Transit/Regrouping context.
- `food_production`.
- `camp_food_consumption` — сумарне Food consumption усіх Army у Camp/Regrouping цієї Region.
- `food_balance` — для звичайної Region `food_production - camp_food_consumption`; для Castle Region дорівнює `food_production`, бо споживання Army Castle Region віднімається вже на рівні Castle.
- `resource_production`, `gold_production`, `silver_production`.
- `resource_surplus_to_castle` — передає позитивну Resource production до Castle тільки для Owned non-Occupied Region із `is_connection_valid == true`, із distance efficiency; інакше `0`.
- `food_surplus_to_castle` — для звичайної Owned non-Occupied Region з `is_connection_valid == true` передає до Castle тільки позитивний `food_balance` з distance efficiency; для Castle Region передає весь позитивний `food_production` з coefficient `1`. Негативний balance звичайної Region до Castle не передається; Occupied/invalid-connected Region нічого не передає.
- `gold_income_for_owner`, `silver_income_for_owner` — для Owned non-Occupied Region з `is_connection_valid == true` дорівнюють відповідній production без distance penalty; інакше `0`.
- `coin_balance_for_owner` — recurring Coin balance Region для formal owner. В Occupied Region позитивний income не нараховується, але формальний owner продовжує сплачувати Region upkeep до втрати ownership; після переходу Region у Neutral цей upkeep припиняється.
- `city_wealth_growth_enabled` — `true` тільки для Owned non-Occupied Region, якщо її formal owner не `empty_coins`; для Neutral/Occupied Region `false`.
- `neutral_defense_full_strength`.
- `neutral_defense_recovery_rate` — ненульовий тільки для Neutral Region, коли `neutral_defense < neutral_defense_full_strength` і в Region немає жодних troops; будь-яка фізична присутність Army ставить recovery rate в `0`, а після виходу останньої Army recovery продовжується від поточного значення.
- `active_combat_situation` — єдина Active player-vs-player CombatSituation у Region або `null`.
- `next_queued_combat` — найраніше зареєстрована `Registered` CombatSituation у єдиній FIFO-черзі Region. Усі player-vs-player CombatSituation цієї Region, включно з Neutral Region і будь-якими парами Player, використовують одну чергу. Attack на Neutral Defense та City Raid у цю чергу не входять.

### User methods

- `user_change_allow_transit(value)` — змінює fallback Transit rule власної Owned non-Occupied Region; для Occupied Region `allow_transit` не використовується.

### Domain methods

- `find_active_camp(player)`, `get_or_create_camp(player)`.
- `register_combat(attacker, registration_reason, defender_player = null)` — реєструє player-vs-player CombatSituation. На Registration фіксуються attacking Army, Region, причина та порядок реєстрації. Для explicit Neutral Camp attack переданий `defender_player` фіксується одразу; для територіальних registration reasons він має бути `null` і визначається тільки на Start. `potential_defenders[]` і `transit_defender_candidates[]` на Registration не фіксуються. Одна CombatSituation завжди має рівно одну attacking Army. Якщо в Region уже є Active або earlier Registered CombatSituation, нова стає в єдину FIFO-чергу Region. Attacker отримує combat lock очікування, але його фізичний стан не змінюється: Camp Army лишається Camp-presence, Movement Army лишається у своєму Movement context з paused progress.
- `start_next_combat_if_possible()` — якщо Active CombatSituation немає, запускає найстарішу Registered CombatSituation Region. Саме цей момент є CombatSituation Start; якщо situation не завершується одразу за Start-умовами, від цього моменту починається її повний `Dt` до Battle Start.
- `resolve_arrival(army, movement)` — визначає Camp/Transit context і, коли є одна з registration conditions, реєструє player-vs-player CombatSituation. **Player-vs-player Retreat entry, включно з pre-battle Retreat, є окремим винятком:** воно не реєструє нову CombatSituation в retreat Region; допустимість Region уже перевірена правилами Retreat, а після `retreat-local` Army доходить до Camp і застосовує звичайні control consequences, включно з Occupation чужої Owned Region. Для звичайного Movement Transit через Occupied Region CombatSituation не реєструє і продовжується без `allow_transit`; Camp-entry в Occupied Region реєструє CombatSituation проти актуального occupier, якщо arriving Player не є самим occupier. Army, що входить після CombatSituation Start, не може стати defender цієї CombatSituation. У Neutral Region сам вхід у Camp не запускає combat із Neutral Defense або City Defense.
- `set_occupied_by(player)` — встановлює Occupation. Якщо в Region вже рухаються Army формального owner, які ввійшли до Occupation, але запізно для попередньої defense, окрема CombatSituation проти актуального occupier реєструється тільки для Army з локальною метою Camp. Army, що проходять Region Transit-ом, після Occupation не створюють CombatSituation і продовжують Transit. Після зміни Occupation запускає перерахунок connectivity відповідного Castle через `Castle.recalculate_region_connections()`.
- `restore_owner_control(expected_occupier_player)` — Occupation припиняється після виходу останньої Army occupier із Camp; після restore запускається перерахунок connectivity Castle.
- `annex_to(player, castle)` — встановлює formal owner/Castle і запускає перерахунок connectivity Castle.
- `become_neutral()` — очищує formal owner/Castle/Occupation, зберігаючи ResourceSite та City/її `wealth`; для колишнього Castle запускає перерахунок connectivity.
- `become_castle_region(new_castle, player)`, `set_connection_valid(value)`.
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
- `active_wealth_ratio_balance` — recovery rate при `active_wealth_ratio < 1`; recovery не залежить від ownership, Occupation або стану Coins Player.
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

`location_state` описує тільки локальне розміщення Unit у Castle/Barracks або Camp для soldier-upkeep/Barracks Capacity. Воно не визначає map/combat presence: для movement, combat, Annexation, Founding та інших військових взаємодій місце Unit визначається його Army та `Army.camp`.

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
- `player` — `castle.player`; проміжна характеристика для computed characteristics інших Model без ланцюга relationships.
- `current_region` — `army.current_region`.
- `current_camp` — `army.camp`.
- `is_regrouping` — чи `army.state == Regrouping`.
- `is_camp_presence` — `current_camp != null && army.state in {Camp, Regrouping}`.
- `experience_balance`.

### User methods

- `user_enter_castle()`, `user_leave_castle()` — змінюють тільки `location_state`; `Castle` доступний лише при Camp-presence Army у Region home Castle.
- `user_change_unit_composition(composition_delta)` — дозволено тільки при `location_state = Castle` у home Castle і коли Army Knight не `is_command_locked` жодною CombatSituation.

### Domain methods

- `set_location_state(state)` — `Castle` дозволений тільки при Camp-presence у home Castle Region.
- `leave_castle_for_movement()`.
- `set_soldiers(new_composition)`, `set_army(army)`.
- `add_battle_experience(amount)`.
- `apply_casualties(...)` — втрати Unit спочатку застосовуються до `soldiers`; Knight не може загинути, доки в нього лишається хоча б один Soldier.
- `die()` — переводить Knight у dead state і викликає `castle.create_knight_replacement()` для звільненого Palace slot. Commander тут не перепризначається негайно: це робиться після завершення розподілу всіх casualties combat, щоб не вибрати Knight, який також загине в цьому самому combat.
- `change_home_castle(new_castle)` — змінює home Castle. Якщо active Knight залишає старий home Castle (зокрема founder після завершення Castle Founding), його старий Palace slot вважається звільненим і запускає той самий `KnightReplacement` queue mechanism, що й після death. У новому Castle цей Knight займає власний slot і не породжує додаткового Knight.

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
- `user_change_commander(knight)` — дозволено, якщо Army не `is_command_locked` жодною CombatSituation; Regrouping саме по собі не забороняє зміну Commander.
- `user_change_combat_thresholds(values)` — змінює persistent thresholds тільки коли Army не `is_command_locked`. Тому attacker після Registration та potential defender після CombatSituation Start уже не можуть змінити свої thresholds; Transit defender candidate може змінювати власний `defense_loss_threshold`, доки не приєднався до defense і не отримав lock. Усі thresholds, потрібні для майбутньої CombatSituation, мають бути визначені під час формування/відправлення Army або іншою попередньою UI-дією; точний UI flow буде визначено окремо.
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
- `start_regrouping()` — переводить Army у Regrouping і скидає progress. Camp зберігається; Army продовжує бути звичайною Camp-presence для Food, Annexation, Founding, Occupation, defense та Camp lifecycle.
- `finish_regrouping()` — повертає Army у Camp після `Dt`.
- `start_retreat_to(region)` — звичайний player-vs-player Retreat: Army одразу вважається такою, що увійшла в retreat Region **без реєстрації нової CombatSituation**, після чого `retreat-local` триває `Dt` до Camp-Regrouping. При досягненні Camp у чужій Owned non-Occupied Region викликається `Region.set_occupied_by(player)`; кілька Army однієї сторони, що Retreat-ять разом, приходять у ту саму Region/Camp і встановлюють одну Occupation.
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

## 7. `CombatSituation`

`CombatSituation` у цій секції — одна модель для всіх player-vs-player interaction з registration/start/active lifecycle; відмінності між registration reasons реалізуються умовною domain logic всередині цієї моделі. Окремі inherited submodels поки не вводяться. Attack на Neutral Defense та City Raid використовують combat calculation, але не входять у player-vs-player queue і не чекають `Dt`.

Одна CombatSituation завжди має рівно одну attacking Army. Дві Army одного Player, які послідовно створюють умови атаки в одній Region, реєструють дві окремі CombatSituation; вони не об'єднуються в одну attacking side і не є reinforcement одна одній.

### Прямі характеристики

- `region`.
- `attacker` — єдина attacking Army; фіксується на Registration.
- `registration_reason` — entry into foreign Owned non-Occupied Region / Camp entry into Occupied Region against current occupier / own Region became occupied around already-present late Camp-bound Army / explicit Neutral Camp attack.
- `registration_order` — FIFO order у єдиній player-vs-player черзі Region.
- `status` — `Registered / Active / Resolved`.
- `start_time` — встановлюється тільки при CombatSituation Start; `null` для queued Registered.
- `defender_player` — для explicit Neutral Camp attack фіксується на Registration як Player обраного target Camp; для територіальних registration reasons лишається `null` до Start і визначається за актуальним ownership/occupation interaction.
- `potential_defenders[]` — `null/empty` до Start; у момент Start фіксуються Camp/Regrouping/entered-for-Camp Army defender Player за їх актуальним станом.
- `transit_defender_candidates[]` — Army defender Player, які ввійшли в Region для Transit до Start і на Start ще не вийшли. Вони не беруть участі автоматично й не блокуються як potential defenders, доки явно не оберуть залишитися для defense; join можливий лише до Battle Start і до фактичного виходу candidate з Region, після чого candidate більше не може приєднатися.
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
- `attacker_loss_threshold` — initial effective value: для explicit attack з Camp у Neutral Region `attacker.target_combat_threshold`; для CombatSituation через Movement — target threshold, якщо фактичний `defender_player == target_opponent`, і incidental threshold інакше. На Battle Start спочатку перевіряється side-level Retreat availability; якщо attacker side не має legal Retreat Region, effective attacker threshold примусово стає `1.0`. Якщо legal Retreat є і effective threshold == `0`, attacker виконує Retreat до combat calculation.
- `defender_combat_threshold` — на Battle Start для кожної potential defender Army береться effective threshold: `0`, якщо для неї встановлено pre-battle Retreat decision, інакше її persistent `defense_loss_threshold`. Спочатку перевіряється side-level Retreat availability. Якщо defender side не має legal Retreat Region, її combat threshold примусово стає `1.0` і жодна Army не виконує pre-battle Retreat. Якщо legal Retreat є, Army з effective threshold `0` спочатку виконують Retreat без casualties; після їх виключення спільний `defender_combat_threshold` дорівнює мінімальному effective threshold серед Army, що лишилися. Якщо жодної defender Army не лишилося, battle не відбувається.
- `defender_loss_threshold` — чинне значення `defender_combat_threshold`; один спільний threshold для всієї defense side, що lock-иться на Battle Start.
- `attacker_strength` — `attacker.attack_strength` для будь-якого player-vs-player battle.
- `defender_strength` — сума strength усіх participating defender Army: для explicit Neutral Camp battle використовується їх `attack_strength`, для territorial invasion/occupation defense — їх `defense_strength`.

### Registration

CombatSituation реєструється, коли:

- Army Player A входить у Owned **non-Occupied** Region Player B (`A != B`); це стосується і Camp, і Transit entry;
- Army Player A входить у Occupied Region з локальною метою Camp, якщо `A` не є current occupier; CombatSituation спрямована проти актуального occupier. Transit через Occupied Region CombatSituation не створює;
- Region Player A стає Occupied, коли Camp-bound Army A вже рухається всередині цієї Region, увійшла туди до Occupation, але вже не може приєднатися до попередньої defense; Transit Army CombatSituation не створює;
- Army Player A у `state == Camp` у Neutral Region ініціює attack на Player B, який має в цій Region хоча б одну Army у `state == Camp`; самі лише Regrouping або entered-for-Camp Army B для Registration недостатні.

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
- situation, яка за актуальними Start-умовами має завершитися без battle одразу (зокрема Transit без Camp/Camp-bound defender або Transit через Region, що на Start є Occupied), завершується без запуску pre-battle interval;
- для решти situations починається повний pre-battle interval `Dt`, і defender отримує відповідний decision dialog.

Для explicit Neutral Camp attack interaction на Start вважається актуальним, якщо зафіксований `defender_player` усе ще має в Region хоча б одну Army у `Camp`/`Regrouping` або Army, що вже ввійшла з локальною метою Camp. Сам по собі Transit цього Player interaction актуальним не робить. Якщо ця умова не виконується, CombatSituation завершується без battle зі статусом `Resolved`, attacker lock знімається, після чого Region може Start-нути наступну Registered situation. Для територіальних registration reasons, якщо на Start Region стала Neutral або немає Player, проти якого за актуальними правилами має відбутися interaction, CombatSituation завершується без battle; attacker продовжує попередній Movement route. Жодна queued situation не завершується або не видаляється раніше власного Start лише через те, що майбутні умови, ймовірно, зникли.

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

Potential defenders на Start — усі Army defender Player у `Camp`/`Regrouping` і всі Army, що вже мають `is_entered_for_camp == true` не пізніше Start. Останні не утворюють окремої категорії battle participation: оскільки entry -> Camp і Start -> Battle Start обидва завжди дорівнюють `Dt`, на Battle Start вони гарантовано вже будуть у Camp або Regrouping і беруть участь на тих самих умовах. Це включає `retreat-local` Army, яка ввійшла в Region до Start.

Окремо Army defender Player, яка ввійшла в Region для Transit **до Start** і на Start ще не вийшла з Region, потрапляє в `transit_defender_candidates[]`, а не в `potential_defenders[]`. Поки вона не обрала defense, вона не command-locked цією CombatSituation і продовжує Transit. До Battle Start і до фактичного виходу з Region Player може явно наказати їй залишитися; тоді Army видаляється з `transit_defender_candidates[]`, додається до `potential_defenders[]` і від цього моменту отримує combat command-lock. Якщо такого рішення немає до першої з цих меж, Army більше не може приєднатися.

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

Explicit attack A -> B у Neutral Region можна зареєструвати тільки якщо B на цей момент має хоча б одну Army у `state == Camp`; Regrouping або entered-for-Camp без звичайної Camp Army для ініціації недостатньо. На Registration фіксується `defender_player = B`. На Start ця situation або лишається атакою саме проти B, або, якщо interaction з B вже не актуальний, завершується без battle зі статусом `Resolved`; вона ніколи не перенаправляється на C чи іншого Player. Усі Army B, які на Start є у Camp/Regrouping або вже entered-for-Camp, обов'язково входять у `potential_defenders[]`; для entered-for-Camp це не опціональне приєднання. Army B, що entered-for-Transit до Start і ще не вийшла з Region, є transit defender candidate та може явно залишитися для defense тільки до Battle Start і до моменту свого виходу; leaving-Camp Army участі не бере. Army B, що входить після Start, не може приєднатися.

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

- `start()` — переводить Registered situation в Active. Для territorial registration reasons визначає актуального `defender_player`; для explicit Neutral Camp attack не переобчислює його, а перевіряє актуальність interaction із зафіксованим Player. Queued Transit situation, для якої Region **на момент Start** є Occupied, завершується без battle. Для іншої чинної Transit situation визначає `transit_mode`, після чого формує defender sets; Transit без Camp/Camp-bound defender завершується одразу без `Dt`, а решта situations запускають повний `Dt` до Battle Start. Якщо interaction більше не існує, завершує situation через no-battle path зі статусом `Resolved`.
- `determine_transit_mode()` — викликається тільки для Transit CombatSituation в Owned non-Occupied Region; повертає Aggressive, якщо Region формально належить зафіксованому `target_opponent`, інакше NonAggressive.
- `collect_potential_defenders()` — на Start фіксує `potential_defenders[]` для всіх Army defender Player у Camp/Regrouping і всіх Army з `is_entered_for_camp == true`; останні гарантовано стануть Camp/Regrouping до Battle Start і не є окремим типом participant. Окремо фіксує `transit_defender_candidates[]` для Transit Army, що ввійшли строго до Start та ще не вийшли; Army, які входять після Start, не додаються.
- `resolve_pre_battle()` — спочатку застосовує Transit Allow/Fight. Якщо result — `Allow`, defender Retreat decisions не виконуються й Army лишаються у своїх states. Для situation, що доходить до Battle Start, method формує effective thresholds: defender pre-battle Retreat decision дає локальний `0`, інші Army зберігають persistent threshold; attacker використовує свій target/incidental threshold.
- `lock_combat_parameters()` — на Battle Start **до будь-якого zero-threshold Retreat** обчислює та lock-ить side-level legal Retreat sets для Attacker і Defender. Якщо side не має legal Retreat Region, її effective combat threshold примусово стає `1.0`; для Defender це також скасовує виконання локальних `0` override. Після цього Army/side з effective threshold `0` і legal Retreat виконують Retreat без casualties. Якщо Defender після цього має participants, його спільний threshold = minimum їх effective thresholds; strengths lock-яться вже для фактичних participants. Якщо Attacker Retreat-ить або всі Defender Army Retreat-ять, battle calculation не виконується.
- `get_legal_retreat_regions(role)` — повертає один side-level набір допустимих сусідніх Region. Для attacker розглядаються три напрями до source side, для defender — три протилежні напрями; для explicit player-vs-player attack між Camp в одній Neutral Region напрямкового обмеження немає і розглядаються всі шість. Foreign Castle Region не допускається. Own Occupied Region не допускається. Для **foreign Owned** Region Retreat дозволений тільки якщо в ній немає жодної Army у `Camp`/`Regrouping` і жодної Army з `is_entered_for_camp == true`, незалежно від Player цієї Army; pure Transit не блокує. Neutral Region із locally-present just-fought opponent не допускається. Neutral Defense саме по собі Retreat не блокує. Оскільки всі Army однієї side знаходяться в одній combat Region і мають ту саму combat role/opponent, допустимість Region не залежить від конкретної Army.
- `select_retreat_region(role)` — один раз вибирає side-specific Retreat Region (`attacker_retreat_region` або `defender_retreat_region`) з side-level набору. Для Defender пріоритет: own non-Occupied Region > Neutral Region > допустима foreign Region. Серед own Region кожна кандидатна Region оцінюється відносно **Castle, до якого приєднана сама ця Region**; перевага має Region, ближча до свого `region.castle`. Це властивість кандидатної Region і не залежить від home Castle, Commander або складу конкретної Army. Серед Neutral Region перевага ближчій до власної territory; для foreign Region використовується визначений географічний критерій. Для Attacker найвищий пріоритет має source Region, якщо вона допустима, інакше застосовуються ті самі групові пріоритети. Якщо після всіх priority rules лишається кілька рівнозначних Region, вибір випадковий. Для кількох defender Army один результат застосовується до всіх Army цієї side, що виконують Retreat.
- `resolve_combat()`.
- `apply_casualties(result)` — після застосування всіх casualties до всіх Unit викликає `ensure_commander_after_casualties()` для кожної surviving Army; зміна Commander не впливає на вже завершений combat calculation, але діє для подальшого post-combat state.
- `apply_result(result)` — виконує Retreat, Occupation, Camp/Transit continuation. Після визначення loser викликає `select_retreat_region(role)` для відповідної side, якщо її Retreat Region ще не була вибрана; якщо програє defender side з кількома Army, всі вони використовують одну `defender_retreat_region`. Army, що приєдналася до defense з Transit, до battle вже є звичайною Camp Army і далі обробляється так само, як інші defender Army. Якщо attacker мав Transit, будь-який result не змушує defender Army, що ввійшли після Start, змінювати свій route; вони завершують свій planned movement після поточного battle. Якщо attacker мав Camp і програв — так само. Якщо attacker мав Camp, виграв і встановив Occupation, пізніша Army formal owner реєструє окрему CombatSituation проти occupier тільки якщо входить/лишається з локальною метою Camp; Transit через вже Occupied Region продовжується без CombatSituation.
- `finish_without_battle(reason)` — завершує situation зі статусом `Resolved`, знімає combat locks і застосовує reason-specific continuation. Для `Allow`/відсутності battle context defender Army не змінюють state. Якщо battle не відбувся через zero-threshold Retreat Attacker-а або всіх Defender Army, виконує відповідні side-level Retreat. В інших no-battle cases для attacker застосовує належне Camp/Transit/Occupation continuation. Цей же terminal path використовується для obsolete explicit Neutral Camp attack.
- `finish()` — terminal transition; після цього Region може Start-нути наступну Registered CombatSituation у FIFO-черзі.

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
- `food_consumption` — сума `Army.food_consumption` усіх `armies[]`, включно з Regrouping.
- `has_eligible_presence` — чи є хоча б одна Army з `state in {Camp, Regrouping}`.
- `food_coin_compensation` — `0` у Castle Region; поза Castle Region, якщо `region.food_balance < 0`, дорівнює пропорційній частці локального deficit за часткою цього Camp у `region.camp_food_consumption`, помноженій на `coins_per_food`; при відсутності deficit дорівнює `0`.
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

- `founder_valid` — founder Knight живий, фізично має потрібний current Camp, `founder_knight.player == player` і не має Soldier. Leave Region/death/отримання Soldier скасовує process.
- `founder_progress_eligible` — `founder_valid && founder_knight.is_camp_presence`; Regrouping не pause-ить Founding.
- `can_progress` — `founder_progress_eligible`; для Neutral Region `neutral_defense == 0`; `region.blocking_camp_presence_player_ids[]` не містить іншого Player. Foreign Transit Army не блокує Founding, навіть якщо через її Transit у Region існує CombatSituation. Саме foreign Camp/Camp-bound presence, а не наявність CombatSituation, призупиняє Founding.
- `progress_rate`, `required_progress`, `is_complete`.

### User methods

- `user_start_castle_founding(...)` — founder може бути у Camp або Regrouping. Після `validate_start()` списує local founding cost із home Castle founder Knight та global cost із Player; скасування process cost не повертає.

### Domain methods

- `validate_start()` — Region має бути Neutral або власною annexed non-Castle Region, active Founding у ній відсутній, founder валідний і фізично присутній, а blocking foreign Camp/Camp-bound presence відсутня на момент старту. Для Neutral Region не потрібні contiguity, Governor Capacity або попередній control-progress; інші стартові умови/вартість перевіряються тут.
- `cancel()` — terminal state без refund при founder invalid.
- `complete()` — створює Castle через `Player.create_castle()`; founder стає Knight нового Castle і займає його початковий Palace slot.

### Triggers

- `founder_invalid` `[state trigger]` — founder перестав задовольняти `founder_valid`.
- `can_progress` `[state trigger]`.
- `complete` `[event trigger]`.

### Trigger methods

- `check_trigger_founder_invalid()` — відстежує перехід `founder_valid` у `false` для active Founding.
- `on_trigger_founder_invalid()` — викликає `cancel()`; це cancellation, а не pause.
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
- `can_progress` — є current order, `castle.empty_food == false`, `castle.player_empty_coins == false`, `castle.barracks_free_capacity > 0`.
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

- `is_queue_head` — визначається через `castle.active_knight_replacement_id`.
- `progress_rate` — ненульовий тільки для queue head.
- `required_progress`, `is_complete`.

### Domain methods

- `complete()` — створює replacement Knight.

### Triggers / Trigger methods

- `complete` `[event trigger]`.
- `check_trigger_complete()`, `on_trigger_complete()`.
