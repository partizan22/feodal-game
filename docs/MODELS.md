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

- `can_pay_global_cost(cost)` — перевіряє одноразову вартість у Coins/Gold/Silver. **Використовує:** `coins`, `gold`, `silver`. **Викликає:** нічого.
- `pay_global_cost(cost)` — списує одноразову глобальну частину вартості після validation. **Використовує:** `coins`, `gold`, `silver`. **Викликає:** `can_pay_global_cost()`.
- `add_coins(amount)` — зараховує разовий Coin reward, зокрема City Raid reward. **Використовує:** `coins`. **Викликає:** нічого.
- `create_castle(region, name, founder_knight)` — створює новий Castle після завершення Founding. **Використовує:** `castles[]`, параметри нового Castle. **Викликає:** initialization нового `Castle`, `Region.become_castle_region()`, `Knight.change_home_castle()`.

### Triggers

- `empty_coins` `[state trigger]` — реагує на зміну стану `empty_coins`; boundary потрібний для Recruitment, City Wealth growth та інших залежних rates.

### Trigger methods

- `check_trigger_empty_coins()` — визначає current boolean state `empty_coins` і, якщо `coin_balance < 0`, прогнозує момент досягнення `coins == 0`. **Використовує:** `coins`, `coin_balance`, `empty_coins`. **Викликає:** нічого.
- `on_trigger_empty_coins()` — прямої зміни Player state не робить; boundary потрібний для перерахунку залежних computed rates. **Використовує:** `empty_coins`. **Викликає:** нічого.

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
- `non_military_food_consumption` — Castle-level Food consumption від Population/Building effects за конфігурацією.
- `castle_region_army_food_consumption` — `region.camp_food_consumption`; усі Army, що стоять у Castle Region, споживають Food із Castle незалежно від `Knight.location_state`.
- `food_balance` — `food_income - non_military_food_consumption - castle_region_army_food_consumption`.
- `food_coin_compensation` — якщо `food == 0 && food_balance < 0`, дорівнює `(-food_balance) * coins_per_food`, інакше `0`.
- `coin_balance` — Coin income/upkeep самого Castle: Building income, Bank multiplier, Palace upkeep та інші Castle-level recurring effects; food compensation враховується Player окремо через `food_coin_compensation`.
- `warehouse_capacity` — Capacity Warehouse відповідно до `levels`.
- `granary_capacity` — Capacity Granary відповідно до `levels`.
- `storage_full` — структура boolean для `wood`, `stone`, `iron`, `food`.
- `empty_food` — `food == 0 && food_balance <= 0`.
- `barracks_capacity` — Barracks Capacity за `levels`.
- `barracks_used` — Soldier reserve + Soldier Knight цього Castle з `location_state == Castle`.
- `barracks_free_capacity` — `barracks_capacity - barracks_used`.
- `governor_capacity` — максимальна кількість зовнішніх Region за Governor's House level.
- `external_region_count` — кількість `regions[]`, крім Castle Region.
- `palace_capacity` — кількість Knight slots за Palace level.
- `active_knight_replacement_id` — ID першого незавершеного KnightReplacement у послідовній черзі або `null`.

### User methods

- немає зафіксованих на цей момент.

### Domain methods

- `can_pay_local_cost(cost)` — перевіряє одноразову Castle-local частину ціни. **Використовує:** `resources`, `food`. **Викликає:** нічого.
- `pay_local_cost(cost)` — списує Castle-local частину ціни. **Використовує:** `resources`, `food`. **Викликає:** `can_pay_local_cost()`.
- `can_house_unit(knight)` — перевіряє, що Knight може мати `location_state = Castle`: Army перебуває у CampInRegion Castle Region цього home Castle і Barracks має Capacity для всіх Soldier Unit. **Використовує:** `region`, `barracks_free_capacity`, `Knight.soldier_count`, `Knight.current_camp`, `Knight.castle`. **Викликає:** нічого.
- `move_soldiers_between_reserve_and_knight(knight, composition_delta)` — переносить Soldier тільки `Castle reserve <-> Knight`. **Використовує:** `soldier_reserve`, `barracks_capacity`, `Knight.castle`, `Knight.location_state`, `Knight.soldiers`. **Викликає:** `Knight.set_soldiers()`.
- `add_recruited_soldier(type)` — додає завершеного recruit у reserve. **Використовує:** `soldier_reserve`, `barracks_free_capacity`. **Викликає:** нічого.
- `can_start_building_upgrade(building_type)` — перевіряє prerequisite та відсутність іншого active upgrade цієї Building. **Використовує:** `levels`, `building_upgrades[]`, `resources`, `food`, `player`. **Викликає:** `can_pay_local_cost()`, `Player.can_pay_global_cost()`.
- `apply_building_upgrade(building_type, target_level)` — встановлює новий level. Для Palace increase створює Knight для нового slot. **Використовує:** `levels`, `palace_capacity`, `knights[]`. **Викликає:** `create_knight_for_palace_slot()` за потреби.
- `create_knight_for_palace_slot()` — створює нового Knight у вільному Palace slot. **Використовує:** `player`, `palace_capacity`, `knights[]`. **Викликає:** initialization нового `Knight` і його початкової одиночної Army у Camp Castle Region.
- `create_knight_replacement()` — створює KnightReplacement після смерті home Knight. **Використовує:** `knight_replacements[]`, `palace_capacity`. **Викликає:** initialization нового `KnightReplacement`.
- `complete_knight_replacement(replacement)` — створює replacement Knight і завершує відповідний Palace slot replacement. **Використовує:** `active_knight_replacement_id`, `knights[]`, `palace_capacity`. **Викликає:** `create_knight_for_palace_slot()`.
- `can_annex_region(region, camp)` — перевіряє Governor Capacity, допустимий зв'язок Region з цим Castle і наявність у `camp` хоча б одного Knight, чия Army має `state == Camp` і чий home Castle дорівнює цьому Castle. Method напряму обходить `CampInRegion -> armies[] -> knights[]`; окрема computed характеристика з ID Castle для цього не потрібна. **Використовує:** `governor_capacity`, `external_region_count`, `regions[]`, `region`, `camp.armies[]`, `Army.state`, `Army.knights[]`, `Knight.castle`. **Викликає:** нічого.
- `recalculate_region_connections()` — централізовано перераховує `Region.is_connection_valid` після occupation/loss/restore/annexation і переводить остаточно disconnected Region у Neutral за правилами гри. **Використовує:** `region`, `regions[]`, топологію через `Region.neighbors[]`, їх ownership/occupation. **Викликає:** `Region.set_connection_valid()`, `Region.become_neutral()` для Region, які юридично втрачаються.

### Triggers

- `empty_food` `[state trigger]` — реагує на зміну `empty_food`.
- `storage_capacity` `[state trigger]` — окремо відстежує full/not-full для Food та кожного компонента `resources`.

### Trigger methods

- `check_trigger_empty_food()` — визначає current state та прогнозує момент `food == 0`, якщо `food_balance < 0`. **Використовує:** `food`, `food_balance`, `empty_food`. **Викликає:** нічого.
- `on_trigger_empty_food()` — прямого state не змінює; boundary потрібний для переходу від витрачання запасу Food до `food_coin_compensation` та для Recruitment. **Використовує:** `empty_food`, `food_coin_compensation`. **Викликає:** нічого.
- `check_trigger_storage_capacity()` — визначає `storage_full` і для кожного ресурсу з позитивним effective rate прогнозує момент досягнення Capacity. **Використовує:** `resources`, `food`, `resource_balance`, `food_balance`, `warehouse_capacity`, `granary_capacity`, `storage_full`. **Викликає:** нічого.
- `on_trigger_storage_capacity()` — прямого state не змінює; boundary фіксує нові effective balances при full/not-full transition. **Використовує:** `storage_full`. **Викликає:** нічого.

---

## 3. `Region`

### Прямі характеристики

- `player` — формальний owner; `null` для Neutral Region.
- `castle` — Castle, до якого Region приєднана; `null`, якщо не належить Castle.
- `occupier_player_id` — ID поточного occupier для Occupied Region; `null`, якщо Region не Occupied.
- `resource_sites` — структура ResourceSite за типами ресурсів і рівнями.
- `allow_transit` — правило Transit для Owned Region.
- `is_connection_valid` — чи має Owned Region чинний зв'язок зі своїм Castle.

### Динамічні характеристики

- `neutral_defense` — поточна сила Neutral Defense. Після успішного combat на знищення стає `0`; поступово відновлюється тільки в Neutral Region, коли в ній немає військ жодного Player.

### Обчислювальні характеристики

- `*neighbors[]` — шість сусідніх Region, визначені топологією карти.
- `*armies[]` — Army, що фізично знаходяться в Region за `Army.current_region`.
- `*camps[]` — active CampInRegion цієї Region.
- `*city` — City цієї Region або `null`.
- `*resource_site_upgrades[]` — ResourceSiteUpgrade цієї Region.
- `*castle_foundings[]` — CastleFounding у цій Region.
- `*combat_situations[]` — active/history CombatSituation цієї Region; pending визначається їх `status`.
- `camp_player_ids[]` — ID Player, для яких у Region є active CampInRegion; scalar IDs, не Player references.
- `eligible_camp_player_ids[]` — ID Player, чиї CampInRegion мають хоча б одну Army у звичайному Camp state; Regrouping не входить. Використовується для Annexation/Founding presence.
- `valid_connected_neighbor_player_ids[]` — ID Player, для яких серед `neighbors[]` є Owned Region з `is_connection_valid == true`.
- `is_occupied` — `occupier_player_id != null`.
- `is_castle_region` — Region є `castle.region` свого Castle.
- `has_any_troops` — чи є в `armies[]` будь-які фізично присутні війська, включно з Movement/Regrouping.
- `has_active_founding` — чи є в `castle_foundings[]` active CastleFounding.
- `food_production` — Food production за `resource_sites`.
- `camp_food_consumption` — сума `CampInRegion.food_consumption` усіх `camps[]`; включає Regrouping Army, бо вони фізично залишаються в CampInRegion.
- `food_balance` — для звичайної Region `food_production - camp_food_consumption`; для Castle Region дорівнює `food_production`, бо військове споживання Castle Region віднімає сам Castle.
- `resource_production` — production Wood/Stone/Iron за `resource_sites`.
- `gold_production` — Gold production за `resource_sites`.
- `silver_production` — Silver production за `resource_sites`.
- `resource_surplus_to_castle` — позитивний потік Wood/Stone/Iron до attached Castle з урахуванням `is_connection_valid`, occupation та DistanceEfficiency.
- `food_surplus_to_castle` — `0`, якщо Region Occupied або її зв'язок із Castle нечинний; інакше тільки позитивний `food_balance`, переданий attached Castle з DistanceEfficiency; для Castle Region коефіцієнт `1`. Негативний balance звичайної Region ніколи не створює Food demand із Castle.
- `gold_income_for_owner` — `gold_production`, тільки якщо Region має owner, не Occupied і `is_connection_valid == true`; інакше `0`. DistanceEfficiency не застосовується.
- `silver_income_for_owner` — `silver_production`, тільки якщо Region має owner, не Occupied і `is_connection_valid == true`; інакше `0`. DistanceEfficiency не застосовується.
- `coin_balance_for_owner` — recurring Coin effect Region для формального owner: upkeep, City income та інші region-level effects.
- `city_wealth_growth_enabled` — true лише для Owned, non-Occupied Region і коли owner Player не має `empty_coins`.
- `neutral_defense_full_strength` — нормальна повністю відновлена сила Neutral Defense Region за конфігурацією/властивостями Region.
- `neutral_defense_recovery_rate` — `0`, якщо Region не Neutral (`player != null`), якщо `has_any_troops == true` або `neutral_defense >= neutral_defense_full_strength`; інакше конфігураційний позитивний rate поступового recovery.

### User methods

- `user_change_allow_transit(value)` — змінює Transit rule власної Region. **Використовує:** `player`, `allow_transit`. **Викликає:** нічого.

### Domain methods

- `find_active_camp(player)` — знаходить active CampInRegion цього Player у `camps[]`. **Використовує:** `camps[]`. **Викликає:** нічого.
- `get_or_create_camp(player)` — повертає існуючий active CampInRegion або створює новий. **Використовує:** `camps[]`. **Викликає:** `find_active_camp()` і initialization нового `CampInRegion` за потреби.
- `find_pending_combat_for_player(player)` — знаходить pending CombatSituation, у якій війська Player мають автоматично долучитися до defense/interaction при arrival. **Використовує:** `combat_situations[]`, їх `status` і сторони. **Викликає:** нічого.
- `resolve_arrival(army, movement)` — визначає актуальний Arrival Resolution: Camp, combat із owner/occupier, автоматичне приєднання reinforcement до pending defense або continuation за route context. У Neutral Region сам вхід у Camp не запускає combat із Neutral Defense або City Defense. **Використовує:** `player`, `occupier_player_id`, `camps[]`, `combat_situations[]`, Castle Region status, `allow_transit`, `Army.player`, Movement local context. **Викликає:** `find_pending_combat_for_player()`, `get_or_create_camp()`, `Army.enter_camp()`, `CombatSituation.add_defender()` або initialization нового `CombatSituation`.
- `set_occupied_by(player)` — встановлює Occupation після переходу переможця в Camp чужої Owned Region. **Використовує:** `player`, `castle`, `occupier_player_id`. **Викликає:** `Castle.recalculate_region_connections()` у attached Castle.
- `restore_owner_control(expected_occupier_player)` — очищує Occupation тільки якщо `occupier_player_id` досі відповідає очікуваному occupier; Occupation припиняється одразу після виходу останньої Army occupier із Camp, не після фізичного перетину межі Region. **Використовує:** `occupier_player_id`, `castle`. **Викликає:** `Castle.recalculate_region_connections()`.
- `annex_to(player, castle)` — змінює formal owner/attached Castle, очищує occupation і завершує Neutral/Occupied state transition. **Використовує:** `player`, `castle`, `occupier_player_id`, `neutral_defense`. **Викликає:** `Castle.recalculate_region_connections()` для старого й нового Castle за потреби.
- `become_neutral()` — остаточно прибирає ownership при втраті connectivity. **Використовує:** `player`, `castle`, `occupier_player_id`, `neutral_defense`. **Викликає:** нічого в City; `wealth` та `active_wealth_ratio` City зберігаються й реагують через свої rates.
- `become_castle_region(new_castle, player)` — робить Region Castle Region після Founding. **Використовує:** `player`, `castle`, occupation state. **Викликає:** `Castle.recalculate_region_connections()` старого Castle, якщо Region була від'єднана від нього.
- `set_connection_valid(value)` — змінює збережений structural connectivity flag. **Використовує:** `is_connection_valid`. **Викликає:** нічого.
- `apply_resource_site_upgrade(resource_type, target_level, quantity)` — переносить `quantity` ResourceSite у наступний level. **Використовує:** `resource_sites`, `resource_site_upgrades[]`. **Викликає:** нічого.
- `can_start_resource_site_upgrade(resource_type, quantity)` — перевіряє layered-upgrade rule та concurrent upgrades. **Використовує:** `resource_sites`, `resource_site_upgrades[]`, `castle`, `player`. **Викликає:** `Castle.can_pay_local_cost()`, `Player.can_pay_global_cost()`.
- `on_army_presence_changed()` — не змінює Neutral Defense напряму; зміна ownership/`has_any_troops` автоматично визначає `neutral_defense_recovery_rate`. **Використовує:** `player`, `has_any_troops`, `neutral_defense`, `neutral_defense_full_strength`, `neutral_defense_recovery_rate`. **Викликає:** нічого.
- `destroy_neutral_defense()` — після перемоги у combat на знищення встановлює `neutral_defense = 0`. City Defense, якщо City є, не змінюється. **Використовує:** `neutral_defense`, `city`. **Викликає:** нічого.

### Triggers

- `neutral_defense_recovery_complete` `[event trigger]` — досягнення `neutral_defense_full_strength` під час поступового recovery; trigger потрібний тільки як boundary для точного припинення dynamic growth на максимумі.

### Trigger methods

- `check_trigger_neutral_defense_recovery_complete()` — якщо `neutral_defense_recovery_rate > 0`, прогнозує момент досягнення `neutral_defense_full_strength`; якщо значення вже досягнуто — повертає `0`. **Використовує:** `neutral_defense`, `neutral_defense_full_strength`, `neutral_defense_recovery_rate`. **Викликає:** нічого.
- `on_trigger_neutral_defense_recovery_complete()` — нормалізує `neutral_defense` до `neutral_defense_full_strength`; після цього recovery rate стає `0`. **Використовує:** `neutral_defense`, `neutral_defense_full_strength`. **Викликає:** нічого.

---

## 4. `City`

### Прямі характеристики

- `region`

### Динамічні характеристики

- `wealth` — повний довгостроковий економічний потенціал City.
- `active_wealth_ratio` — активна частка Wealth у діапазоні `0..1`; після будь-якого успішного Raid стає `0` і поступово відновлюється до `1`.

### Обчислювальні характеристики

- `wealth_balance` — effective rate зміни `wealth`; позитивний тільки коли `active_wealth_ratio == 1` і `region.city_wealth_growth_enabled == true`, інакше `0`.
- `active_wealth_ratio_balance` — конфігураційний recovery rate, якщо `active_wealth_ratio < 1`; recovery не залежить від ownership Region або `Player.empty_coins`. При `active_wealth_ratio >= 1` дорівнює `0`.
- `effective_wealth` — `wealth * active_wealth_ratio`.
- `city_defense` — `CityDefense(wealth)`; залежить тільки від повного `wealth`, не від `active_wealth_ratio`, ownership або попередніх Raid.
- `coin_income` — recurring Coin income City для owner Region на основі `effective_wealth`; `0`, якщо Region Neutral або Occupied.
- `raid_reward` — поточний разовий Coin reward Raid, розрахований із `effective_wealth` та конфігурації.

### User methods

- немає; City Raid ініціює конкретна `Army` через `Army.user_raid_city()`.

### Domain methods

- `complete_raid(player)` — фіксує reward за pre-raid `effective_wealth`, негайно зараховує його Player і скидає `active_wealth_ratio = 0`; `wealth` та `city_defense` не змінюються. **Використовує:** `wealth`, `active_wealth_ratio`, `effective_wealth`, `raid_reward`, `region`. **Викликає:** `Player.add_coins()`.

### Triggers

- `active_wealth_recovered` `[event trigger]` — досягнення `active_wealth_ratio == 1` після Raid.

### Trigger methods

- `check_trigger_active_wealth_recovered()` — якщо `active_wealth_ratio_balance > 0`, прогнозує момент досягнення `1`; якщо `active_wealth_ratio >= 1`, повертає `0`. **Використовує:** `active_wealth_ratio`, `active_wealth_ratio_balance`. **Викликає:** нічого.
- `on_trigger_active_wealth_recovered()` — нормалізує `active_wealth_ratio = 1`; після цього `wealth_balance` знову може стати позитивним за звичайними ownership/Coins conditions. **Використовує:** `active_wealth_ratio`. **Викликає:** нічого.

---

## 5. `Knight`

Knight разом зі своїми Soldier представляє gameplay Unit; окремої backend-моделі Unit немає.

`location_state` описує тільки локальне розміщення Unit у Castle/Barracks або Camp для upkeep/Barracks Capacity. Для movement, combat, Annexation, Founding та інших військових взаємодій місце Unit визначається його Army та `Army.camp`. `location_state == Castle` допустимий лише коли Army перебуває в CampInRegion home Castle Region цього Knight.

### Прямі характеристики

- `name`
- `castle` — home Castle.
- `army` — поточна Army.
- `soldiers` — кількість Soldier за Type.
- `base_strength` — особиста базова бойова сила Knight.
- `location_state` — `Castle` або `Camp`; не змінює map-level state Army.
- `status` — active/dead/replaced lifecycle state.

### Динамічні характеристики

- `experience`

### Обчислювальні характеристики

- `soldier_count` — сума `soldiers`.
- `food_consumption` — `soldier_count + 1` для active Knight; Soldier Type не впливає на Food consumption.
- `experience_coefficient` — coefficient Knight за Experience.
- `unit_attack_strength` — Attack strength Soldier + `base_strength`, помножені на `experience_coefficient`.
- `unit_defense_strength` — Defense strength Soldier + `base_strength`, помножені на `experience_coefficient`.
- `coin_upkeep` — постійний recurring Coin upkeep Unit; залежить від Soldier Type, Knight та `location_state`/Army mode, але не включає food compensation.
- `current_region` — піднімає `army.current_region`.
- `current_camp` — піднімає `army.camp`; доступний CastleFounding та іншим без переходу `Knight -> Army -> CampInRegion`.
- `is_regrouping` — чи `army.state` є Regrouping.
- `is_regular_camp_presence` — `current_camp != null && army.state == Camp`.
- `experience_balance` — passive effective rate Experience.

### User methods

- `user_enter_castle()` — змінює тільки `location_state` на `Castle`; Army і її CampInRegion не змінюються. **Використовує:** `castle`, `army`, `current_camp`, `soldier_count`, `location_state`. **Викликає:** `Castle.can_house_unit()`, `set_location_state()`.
- `user_leave_castle()` — змінює тільки `location_state` на `Camp`; Army залишається в тому самому CampInRegion. **Використовує:** `castle`, `army`, `current_camp`, `location_state`. **Викликає:** `set_location_state()`.
- `user_change_unit_composition(composition_delta)` — змінює Soldier composition тільки у home Castle. **Використовує:** `castle`, `current_camp`, `location_state`, `soldiers`. **Викликає:** `Castle.move_soldiers_between_reserve_and_knight()`.

### Domain methods

- `set_location_state(state)` — контрольовано змінює `location_state`; `Castle` дозволений тільки при `army.state == Camp` і `current_camp.region == castle.region`. **Використовує:** `location_state`, `army`, `current_camp`, `castle`. **Викликає:** нічого.
- `leave_castle_for_movement()` — перед стартом Movement переводить `location_state = Camp`, якщо Knight був у Barracks; це звільняє Barracks Capacity без окремої user action. **Використовує:** `location_state`, `army`. **Викликає:** `set_location_state()`.
- `set_soldiers(new_composition)` — контрольовано змінює `soldiers`. **Використовує:** `soldiers`. **Викликає:** нічого.
- `set_army(army)` — змінює membership Unit в Army при merge/split/dissolve. **Використовує:** `army`. **Викликає:** нічого.
- `add_battle_experience(amount)` — додає разовий battle Experience. **Використовує:** `experience`. **Викликає:** нічого.
- `apply_casualties(soldier_losses, knight_dies)` — застосовує casualties; Knight death дозволений лише після втрати всіх його Soldier. **Використовує:** `soldiers`, `status`. **Викликає:** `die()` за потреби.
- `die()` — переводить Knight у dead state та запускає replacement у home Castle; membership у Army остаточно чиститься після розподілу всіх casualties CombatSituation. **Використовує:** `status`, `castle`, `army`. **Викликає:** `Castle.create_knight_replacement()`.
- `change_home_castle(new_castle)` — змінює home Castle, зокрема для founder після Founding. **Використовує:** `castle`. **Викликає:** нічого.

### Triggers

- немає зафіксованих на цей момент.

---

## 6. `Army`

### Прямі характеристики

- `player` — власник Army; прямий relationship.
- `current_region` — поточна Region Army.
- `camp` — поточний CampInRegion або `null`.
- `commander` — Commander-in-Chief.
- `defense_loss_threshold`
- `target_combat_threshold`
- `incidental_combat_threshold`
- `state` — Camp / Movement / Regrouping та потрібні V1 підстани.

### Динамічні характеристики

- `regrouping_progress`

### Обчислювальні характеристики

- `*knights[]` — Knight, які входять до Army.
- `*movement` — поточний active Movement цієї Army або `null`.
- `soldier_count` — загальна кількість Soldier.
- `food_consumption` — сума `Knight.food_consumption`; використовується CampInRegion або як food-equivalent consumption у Movement.
- `attack_strength` — сума `Knight.unit_attack_strength` з Commander coefficient.
- `defense_strength` — сума `Knight.unit_defense_strength` з Commander coefficient.
- `coin_upkeep` — сумарний звичайний recurring Coin upkeep Knight/Soldier; не включає food compensation.
- `food_coin_compensation` — якщо `state == Movement`, дорівнює `food_consumption * coins_per_food`; інакше `0`. Таким чином усі Movement phases, включно з Transit/final-local/retreat-local, не споживають локальну Food.
- `regrouping_progress_rate` — rate Regrouping; у V1 конфігураційна константа, ненульова тільки в Regrouping.

### User methods

- `user_merge_armies(armies, commander, thresholds)` — об'єднує Army/Unit одного Player в одному CampInRegion. **Використовує:** `player`, `camp`, `state`, `knights[]`, thresholds, input Army `player/camp/state`. **Викликає:** `CampInRegion.validate_reorganization()`, `Knight.set_army()` для всіх Unit; старі Army логічно завершуються, створюється нова Army з тим самим `player` і `camp`.
- `user_split_army(groups)` — розділяє Army на нові Army, які успадковують current thresholds, `player`, `current_region` і `camp`. **Використовує:** `player`, `camp`, `state`, `knights[]`, thresholds. **Викликає:** `CampInRegion.validate_reorganization()`, `Knight.set_army()`; створює нові Army.
- `user_change_commander(knight)` — змінює Commander-in-Chief. **Використовує:** `knights[]`, `commander`, `state`. **Викликає:** нічого.
- `user_change_combat_thresholds(values)` — змінює три thresholds; дозволено також у Movement/Regrouping, але вже started CombatSituation використовує locked values. **Використовує:** threshold characteristics. **Викликає:** нічого.
- `user_attack_player(target_camp)` — ініціює player-vs-player combat у Neutral Region саме цією Army як Attacker. Дозволено тільки коли Army має `state == Camp`, її `camp` active, target Camp належить іншому Player і знаходиться в тій самій Neutral Region. Defender side складається з усіх defensive-eligible Army target Camp; війська третіх Player не зачіпаються. **Використовує:** `player`, `state`, `camp`, `current_region`, `target_camp`, `target_camp.armies[]`. **Викликає:** `CampInRegion.get_defending_armies()`, initialization `CombatSituation` з `combat_type = neutral_camp_player_combat` і `attacker = this`.
- `user_raid_city(city)` — ініціює City Raid саме цією Army. Дозволено тільки для `state == Camp`, коли `camp.region == city.region`, Army не заблокована іншою active CombatSituation та інші raid conditions виконані. Neutral Defense не бере участі. City Defense є abstract Defender, defender threshold береться з raid config, а Attacker використовує звичайний `target_combat_threshold` цієї Army. **Використовує:** `player`, `state`, `camp`, `current_region`, `target_combat_threshold`, `city.region`, `city.city_defense`, `city.raid_reward`, active combat context Region. **Викликає:** initialization `CombatSituation` з `combat_type = city_raid`, `attacker = this`, `city = city`.

### Domain methods

- `enter_region(region)` — фіксує фізичний вхід Army у Region. **Використовує:** `current_region`. **Викликає:** `Region.on_army_presence_changed()` для old/new Region.
- `enter_camp(camp)` — переводить Army у Camp і встановлює прямий `camp`. **Використовує:** `player`, `current_region`, `camp`, `state`. **Викликає:** `CampInRegion.accept_army()` і `Region.on_army_presence_changed()` за потреби.
- `leave_camp()` — очищує `camp` перед Movement/Retreat/іншим виходом. Якщо це остання Army occupier у Camp, Occupation припиняється саме в цей момент через CampInRegion lifecycle. **Використовує:** `camp`, `state`. **Викликає:** `CampInRegion.on_army_left()` після зміни relationship.
- `start_movement(movement)` — переводить Army у Movement. Усі Knight із `location_state == Castle` автоматично виходять із Barracks у Camp-state перед початком руху. **Використовує:** `state`, `camp`, `current_region`, `movement`, `knights[]`. **Викликає:** `Knight.leave_castle_for_movement()` для потрібних Knight, `leave_camp()`.
- `finish_movement()` — завершує Movement-side state перед Camp/interaction. **Використовує:** `state`, `movement`. **Викликає:** нічого.
- `stop_transit_for_defense(combat)` — спеціальний виняток: припиняє Transit і долучає Army до defense до battle start. **Використовує:** `state`, `movement`, `current_region`, `player`. **Викликає:** `Movement.stop_for_defense()`, `CombatSituation.add_defender()`.
- `start_regrouping()` — переводить Army у Regrouping і скидає `regrouping_progress`; `camp` зберігається, але Army не є eligible presence для Annexation/Founding. **Використовує:** `state`, `regrouping_progress`, `camp`. **Викликає:** нічого.
- `finish_regrouping()` — повертає Army зі стану Regrouping у звичайний Camp. **Використовує:** `state`, `regrouping_progress`, `camp`. **Викликає:** нічого.
- `start_retreat_to(region)` — у момент Retreat Army вважається такою, що вже увійшла в retreat Region, після чого запускається `retreat-local` Movement до Camp. **Використовує:** `current_region`, `camp`, `state`, `player`. **Викликає:** `leave_camp()`, `enter_region()`, initialization `Movement`, `Movement.start_retreat_local()`.
- `remove_dead_knights()` — після завершення casualty distribution від'єднує dead Knight від Army; якщо dead Knight був Commander, залишені живі Unit обробляються через dissolve. **Використовує:** `knights[]`, `commander`. **Викликає:** `Knight.set_army(null)`, `dissolve_after_commander_death()` за потреби.
- `dissolve_after_commander_death()` — розпускає Army на окремі Unit після смерті Commander; нові одиночні Army успадковують map/camp context та thresholds. **Використовує:** `player`, `commander`, `knights[]`, `camp`, `current_region`, thresholds. **Викликає:** `Knight.set_army()` та initialization окремих Army containers.

### Triggers

- `regrouping_complete` `[event trigger]`.

### Trigger methods

- `check_trigger_regrouping_complete()` — у Regrouping прогнозує момент completion. **Використовує:** `state`, `regrouping_progress`, `regrouping_progress_rate`, конфігураційний required progress. **Викликає:** нічого.
- `on_trigger_regrouping_complete()` — повторно перевіряє state і завершує Regrouping. **Використовує:** `state`, `regrouping_progress`. **Викликає:** `finish_regrouping()`.

---

# Persistent interaction/process models

## 7. `CombatSituation`

### Прямі характеристики

- `region`
- `attacker` — Attacker Army.
- `defenders[]` — Defender Army/Unit containers для player-vs-player combat; для abstract defense combat може бути порожнім.
- `city` — City, якщо combat використовує City Defense (`city_raid` або `neutral_defense_destruction` у Region з City), інакше `null`.
- `combat_type` — зокрема normal player combat, neutral-camp player combat, `neutral_defense_destruction`, `city_raid`.
- `defenders_retreat_decisions` — pre-battle retreat decisions; user задає саме факт Retreat, destination обирає domain logic.
- `locked_combat_parameters` — thresholds, Target/Incidental classification, abstract defense strength, retreat destinations та інші parameters, які фіксуються безпосередньо в момент Battle Start.
- `status`

### Динамічні характеристики

- `battle_start_progress`

### Обчислювальні характеристики

- `battle_start_progress_rate` — у V1 конфігураційна константа для відповідного combat start delay.
- `required_battle_start_progress` — progress boundary Battle Start.
- `is_battle_valid` — чи сторони та умови combat все ще актуальні.
- `defender_loss_threshold` — для player combat мінімальний ненульовий Defense Loss Threshold після pre-battle zero-threshold retreats і forced-max cases; для `neutral_defense_destruction` примусово `1.0`; для `city_raid` — raid-specific fixed threshold із конфігурації.
- `attacker_loss_threshold` — для `neutral_defense_destruction` і `city_raid` завжди `attacker.target_combat_threshold`; для player-vs-player combat — Target або Incidental threshold відповідно до звичайної classification. Значення lock-иться на Battle Start.
- `attacker_strength` — strength Attacker у locked Battle Start state для потрібного combat mode.
- `defender_strength` — для player combat сума strength Defender Army; для `neutral_defense_destruction` дорівнює `region.neutral_defense + city.city_defense` якщо City є, інакше тільки `region.neutral_defense`; для `city_raid` дорівнює `city.city_defense`.

### User methods

- `user_stop_transit_for_defense(army)` — додає допустиму own Transit Army до defense до Battle Start. **Використовує:** `region`, `defenders[]`, `status`, `battle_start_progress`, `Army.player`. **Викликає:** `Army.stop_transit_for_defense()` і `add_defender()`.
- `user_set_pre_battle_retreat_decision(army, retreat)` — фіксує або скасовує рішення Retreat до Battle Start; destination користувач не задає. **Використовує:** `defenders[]`, `defenders_retreat_decisions`, `status`, `region`. **Викликає:** нічого.
- `user_attack_neutral_defense(attacker, region)` — ініціалізує combat для знищення поточної Neutral Defense. Якщо Region має City, його `city_defense` автоматично додається до Defender strength, але не стає окремою persistent ціллю. Attacker використовує свій звичайний `target_combat_threshold`. **Використовує:** `attacker`, `attacker.target_combat_threshold`, `region.neutral_defense`, `region.city`, Camp presence Attacker. **Викликає:** initialization `CombatSituation` з `combat_type = neutral_defense_destruction`; parameters ще не lock-аються.

### Domain methods

- `add_defender(army)` — додає Army до Defender side до Battle Start, зокрема reinforcement, що автоматично прибув у final Region. **Використовує:** `defenders[]`, `status`, `region`, `Army.player`. **Викликає:** нічого.
- `lock_combat_parameters()` — викликається тільки в момент Battle Start; фіксує thresholds, Target/Incidental classification, participating defenders, strength inputs, abstract City/Neutral Defense values і retreat destinations. Для `neutral_defense_destruction` defender threshold = `1.0`; для `city_raid` — конфігураційний City Defense threshold; Attacker обох abstract combats використовує `target_combat_threshold`. **Використовує:** `attacker`, `defenders[]`, current thresholds/states, `region`, `city`, Movement destination context. **Викликає:** `select_retreat_region()` для сторін, які можуть Retreat.
- `get_legal_retreat_regions(army, role)` — визначає legal Retreat destinations за геометрією й current Region states. Method напряму перевіряє військову присутність у сусідніх Region через їх `armies[]`/Camp/Movement context: Transit Army, що лише проходить Region і не створює примусової взаємодії, сама по собі не блокує Retreat; локальна Camp/Regrouping/інша presence, яка спричинила б примусовий combat для retreating Army, блокує відповідну Region. Окрема computed характеристика для цього не потрібна. **Використовує:** `region.neighbors[]`, combat context, Army source/entry direction, ownership/occupation сусідніх Region, їх `armies[]`, `Army.state`, `Army.camp`, active Movement context. **Викликає:** нічого.
- `select_retreat_region(army, role)` — застосовує правила пріоритету Retreat; при рівнозначних candidates використовує deterministic RNG. Якщо legal Region немає, threshold цієї Army для combat стає максимальним. **Використовує:** результат `get_legal_retreat_regions()`, home-territory/distance context, deterministic RNG. **Викликає:** `get_legal_retreat_regions()`.
- `resolve_combat()` — виконує combat calculation, Luck reroll on exact tie, loss fractions, casualties Attacker і winner/loser consequences. Abstract Neutral Defense/City Defense не отримують persistent partial casualties. **Використовує:** `combat_type`, `locked_combat_parameters`, `attacker_strength`, `defender_strength`, thresholds, `region`, `city`. **Викликає:** `apply_casualties()`, `apply_result()`.
- `apply_casualties(result)` — розподіляє casualties між реальними Unit/Soldier сторін, виконує Soldier-before-Knight rule і battle Experience. Для abstract Defender (`neutral_defense_destruction`, `city_raid`) persistent Defender casualties не записуються: при програші Attacker Neutral Defense лишається на pre-combat current value, City Defense завжди незмінна. **Використовує:** `attacker`, `defenders[]`, combat result, `combat_type`, deterministic RNG context. **Викликає:** `Knight.apply_casualties()`, `Knight.add_battle_experience()`, після повного distribution `Army.remove_dead_knights()`.
- `apply_result(result)` — виконує Retreat, Camp/Occupation/Transit continuation та abstract-defense results. Для `neutral_defense_destruction` успіх Attacker викликає `Region.destroy_neutral_defense()`; City Defense не змінюється. Для `city_raid` успіх Attacker викликає `City.complete_raid()`; Neutral Defense не читається і не змінюється. **Використовує:** `region`, `attacker`, `defenders[]`, `combat_type`, `city`. **Викликає:** `Army.start_retreat_to()`, `Region.destroy_neutral_defense()`, `Region.set_occupied_by()`, `Region.get_or_create_camp()`, `Army.enter_camp()`, `Movement.continue_after_transit_combat()` або `City.complete_raid()` залежно від context.

### Triggers

- `battle_start` `[event trigger]`.

### Trigger methods

- `check_trigger_battle_start()` — прогнозує completion Battle Start delay; при scheduled check повторно враховує `is_battle_valid`. **Використовує:** `status`, `battle_start_progress`, `battle_start_progress_rate`, `required_battle_start_progress`, `is_battle_valid`. **Викликає:** нічого.
- `on_trigger_battle_start()` — якщо combat усе ще valid, саме тут вперше lock-ає остаточні parameters і resolve-ить battle; якщо invalid — завершує CombatSituation без combat. **Використовує:** `status`, `is_battle_valid`. **Викликає:** `lock_combat_parameters()`, `resolve_combat()`.

---

## 8. `CampInRegion`

Один active instance представляє один безперервний епізод присутності Army конкретного Player у Camp конкретної Region. Модель існує для власної, Neutral та чужої Region. Коли остання Army залишає Camp, instance логічно деактивується; фізичний запис зберігається. Наступна поява військ створює новий instance.

### Прямі характеристики

- `player` — Player цього CampInRegion; прямий relationship.
- `region`
- `status`

### Динамічні характеристики

- `control_progress`

### Обчислювальні характеристики

- `*armies[]` — Army, для яких цей active CampInRegion є `Army.camp`.
- `food_consumption` — сума `Army.food_consumption` усіх `armies[]`, включно з Regrouping Army.
- `has_eligible_presence` — чи є хоча б одна Army з `state == Camp`; Regrouping не рахується eligible presence для Annexation/Founding.
- `food_coin_compensation` — `0` у Castle Region; `0`, якщо `region.food_balance >= 0`; інакше частка дефіциту Region пропорційно `food_consumption`: `(-region.food_balance) * food_consumption / region.camp_food_consumption * coins_per_food`. Таким чином у Neutral Region дефіцит розподіляється між Player, а не між окремими Army.
- `has_valid_adjacent_owned_region` — чи `region.valid_connected_neighbor_player_ids[]` містить ID `player`.
- `can_progress` — чи Annexation control progress може накопичуватися зараз: Region Neutral/Occupied для цього Player; для Neutral Region `region.neutral_defense == 0`; `has_eligible_presence`; немає foreign eligible Camp presence; є valid adjacent owned Region; процес не заблокований іншими правилами. City Defense окремо не блокує progress після успішного `neutral_defense_destruction`.
- `control_progress_rate` — effective rate control progress; `0`, якщо `can_progress == false`.
- `required_control_progress` — потрібний progress для Neutral/Occupied Region за конфігурацією.
- `is_ready_for_annexation` — `control_progress >= required_control_progress`.

### User methods

- `user_annex_region(castle)` — виконує ручну Annexation. **Використовує:** `player`, `region`, `is_ready_for_annexation`, `region.eligible_camp_player_ids[]`, `region.has_active_founding`, `castle`. **Викликає:** `can_annex_to()`, `Castle.can_annex_region()`, `Region.annex_to()`.

### Domain methods

- `accept_army(army)` — validation, що Army має той самий `player` і знаходиться в `region`; actual relationship встановлює сама Army. **Використовує:** `player`, `region`, `armies[]`, `Army.player`, `Army.current_region`. **Викликає:** нічого.
- `on_army_left(army)` — після очищення `Army.camp` перевіряє, чи Camp спорожнів; якщо так — деактивує instance, а control progress нового епізоду почнеться з нуля. Якщо цей Player є occupier, вихід останньої його Army з Camp одразу припиняє Occupation. **Використовує:** `player`, `armies[]`, `status`, `control_progress`, `region`. **Викликає:** `Region.restore_owner_control(player)` якщо цей Player досі є occupier.
- `get_defending_armies()` — повертає Army цього Camp, які фізично можуть брати участь у defense; Regrouping Army включаються, Movement Army не входять до `armies[]`. **Використовує:** `armies[]`, їх `state`. **Викликає:** нічого.
- `validate_reorganization(armies)` — перевіряє, що всі Army належать цьому Camp і не перебувають у забороненому стані; Regrouping не дозволяє merge/split. **Використовує:** `armies[]`, input Army `state/camp/player`. **Викликає:** нічого.
- `can_annex_to(castle)` — перевіряє Camp-level умови Annexation перед Castle-specific validation; вимогу щодо home Castle конкретного Knight перевіряє сам `Castle.can_annex_region()` через relationships Camp/Army/Knight. **Використовує:** `is_ready_for_annexation`, `region.eligible_camp_player_ids[]`, `region.has_active_founding`, `castle`. **Викликає:** `Castle.can_annex_region()`.

### Triggers

- `can_progress` `[state trigger]`.
- `ready_for_annexation` `[event trigger]`.

### Trigger methods

- `check_trigger_can_progress()` — визначає state `can_progress`. **Використовує:** `can_progress`, `control_progress_rate`, `region`, `armies[]`. **Викликає:** нічого.
- `on_trigger_can_progress()` — прямого state не змінює; boundary фіксує новий `control_progress_rate`. Якщо Camp empty, lifecycle закривається через `on_army_left()`. **Використовує:** `can_progress`. **Викликає:** нічого.
- `check_trigger_ready_for_annexation()` — якщо `control_progress_rate > 0`, прогнозує момент досягнення `required_control_progress`; якщо уже досягнуто — повертає `0`. **Використовує:** `control_progress`, `control_progress_rate`, `required_control_progress`, `is_ready_for_annexation`. **Викликає:** нічого.
- `on_trigger_ready_for_annexation()` — не виконує Annexation автоматично; фіксує eligibility. **Використовує:** `is_ready_for_annexation`, `status`. **Викликає:** нічого.

---

## 9. `CastleFounding`

### Прямі характеристики

- `player`
- `region`
- `founder_knight`
- `castle_name`
- `status`

### Динамічні характеристики

- `progress`

### Обчислювальні характеристики

- `founder_valid` — founder Knight живий, `founder_knight.current_camp` належить потрібній Region і тому самому Player, Knight не має Soldier. Leave Region/death/отримання Soldier робить процес invalid і скасовує його.
- `founder_progress_eligible` — `founder_valid && founder_knight.is_regular_camp_presence`; Regrouping не скасовує Founding, але pause-ить progress.
- `can_progress` — `founder_progress_eligible`; для Neutral Region `region.neutral_defense == 0`; немає foreign Player у `region.eligible_camp_player_ids[]` та інших pause conditions. City Defense окремо не блокує Founding після знищення Neutral Defense.
- `progress_rate` — effective Founding rate; `0`, якщо `can_progress == false`.
- `required_progress` — required Founding progress.
- `is_complete` — `progress >= required_progress`.

### User methods

- `user_start_castle_founding(player, region, founder_knight, castle_name)` — створює/ініціалізує Founding і списує start cost. Для Neutral Region вимагає `region.neutral_defense == 0`. **Використовує:** `player`, `region`, `founder_knight`, `founder_knight.castle`, `founder_knight.current_camp`, `founder_knight.is_regular_camp_presence`, `region.eligible_camp_player_ids[]`, `region.has_active_founding`, `region.neutral_defense`. **Викликає:** `validate_start()`, `Castle.pay_local_cost()`, `Player.pay_global_cost()`.

### Domain methods

- `validate_start()` — перевіряє Neutral/own Region rules, founder без Soldier у звичайному Camp, для Neutral Region `neutral_defense == 0`, відсутність blocking foreign eligible Camp та інших active Founding. **Використовує:** `player`, `region`, `founder_knight`, `founder_valid`, `founder_progress_eligible`, `region.eligible_camp_player_ids[]`, `region.has_active_founding`, `region.neutral_defense`. **Викликає:** нічого.
- `cancel()` — terminal state без refund, якщо founder leave/die/отримує Soldier або інша cancel-condition. **Використовує:** `status`, `founder_valid`. **Викликає:** нічого.
- `complete()` — створює Castle і переводить Region/founder у новий стан. **Використовує:** `player`, `region`, `founder_knight`, `castle_name`, `is_complete`. **Викликає:** `Player.create_castle()`; стартові Warehouse/Granary/Palace levels встановлюються в new Castle creation logic без створення додаткового Knight за початковий Palace slot.

### Triggers

- `can_progress` `[state trigger]`.
- `complete` `[event trigger]`.

### Trigger methods

- `check_trigger_can_progress()` — визначає pause/resume state; якщо `founder_valid == false` через cancel-condition, повертає `0` для окремої GameEvent, яка закриє процес. **Використовує:** `founder_valid`, `founder_progress_eligible`, `can_progress`, `status`. **Викликає:** нічого.
- `on_trigger_can_progress()` — якщо founder став invalid — `cancel()`; інакше прямого state не змінює, boundary фіксує новий `progress_rate`. **Використовує:** `founder_valid`, `can_progress`, `status`. **Викликає:** `cancel()` за потреби.
- `check_trigger_complete()` — прогнозує момент `progress == required_progress`, якщо `progress_rate > 0`. **Використовує:** `progress`, `progress_rate`, `required_progress`, `is_complete`, `status`. **Викликає:** нічого.
- `on_trigger_complete()` — повторно перевіряє validity/completion і завершує Founding. **Використовує:** `is_complete`, `founder_valid`, `status`. **Викликає:** `complete()`.

---

## 10. `Movement`

### Прямі характеристики

- `army`
- `route[]` — запланована послідовність Region references.
- `current_region`
- `next_region`
- `phase` — transit / final-local / retreat-local та потрібні підфази.
- `local_exit_region` — зафіксований вихід із current Region для transit phase.
- `status`
- `direction_revealed` — frontend-visible факт reveal поточної transit phase.

### Динамічні характеристики

- `progress`

### Обчислювальні характеристики

- `progress_rate` — effective movement rate.
- `direction_reveal_progress` — boundary reveal поточної transit phase.
- `phase_completion_progress` — boundary завершення current phase.

### User methods

- `user_start_movement(army, route[])` — створює та запускає Movement. **Використовує:** `army`, `route[]`, `army.current_region`, `army.camp`, `army.state`, `army.player`. **Викликає:** `validate_route()`, `Army.start_movement()`, `start_phase()`.
- `user_change_planned_route(route[])` — змінює тільки ще не зафіксовану future частину Route. **Використовує:** `route[]`, `current_region`, `next_region`, `local_exit_region`, `phase`, `status`. **Викликає:** `validate_route()`.

### Domain methods

- `validate_route(route[])` — перевіряє adjacency і заборони Castle Region/Transit для future route. **Використовує:** `route[]`, `current_region`, `Region.neighbors[]`, ownership/transit characteristics Region, `Army.player`. **Викликає:** нічого.
- `start_phase(current_region, next_region, phase)` — фіксує локальну мету/exit, скидає `progress` і `direction_revealed`. **Використовує:** `current_region`, `next_region`, `phase`, `local_exit_region`, `progress`, `direction_revealed`, `route[]`. **Викликає:** нічого.
- `start_retreat_local(army, retreat_region)` — ініціалізує `retreat-local` phase після того, як Army уже вважається такою, що увійшла в retreat Region. **Використовує:** `army`, `current_region`, `phase`, `progress`, `status`. **Викликає:** `start_phase()`.
- `stop_for_defense()` — завершує поточний Transit як спеціальний defense exception. **Використовує:** `phase`, `status`, `army`. **Викликає:** `Army.finish_movement()`.
- `advance_to_next_region()` — переводить Army через border і запускає наступну local phase; якщо arrival має долучити reinforcement до pending combat, це визначається Region. **Використовує:** `current_region`, `next_region`, `route[]`, `phase`, `status`. **Викликає:** `Army.enter_region()`, `Region.resolve_arrival()` там, де interaction виникає при entry, `start_phase()`.
- `resolve_camp_arrival()` — виконує Arrival Resolution у final Region. **Використовує:** `army`, `current_region`, `phase`, `status`. **Викликає:** `Army.finish_movement()`, `Region.resolve_arrival()`.
- `continue_after_transit_combat()` — після перемоги в combat під час Transit запускає вже зафіксовану наступну phase без повторного проходження current Region. **Використовує:** `route[]`, `current_region`, `next_region`, `phase`. **Викликає:** `start_phase()`/`advance_to_next_region()` відповідно до locked local state.

### Triggers

- `direction_revealed` `[state trigger]`.
- `next_region_reached` `[event trigger]`.
- `camp_reached` `[event trigger]`.

### Trigger methods

- `check_trigger_direction_revealed()` — визначає state `progress >= direction_reveal_progress` для transit phase і прогнозує boundary. **Використовує:** `phase`, `progress`, `progress_rate`, `direction_reveal_progress`. **Викликає:** нічого.
- `on_trigger_direction_revealed()` — синхронізує direct `direction_revealed` з актуальним trigger state; transition у true використовується frontend/notification layer. **Використовує:** `phase`, `progress`, `direction_reveal_progress`, `direction_revealed`. **Викликає:** нічого.
- `check_trigger_next_region_reached()` — для transit phase прогнозує `phase_completion_progress`. **Використовує:** `phase`, `progress`, `progress_rate`, `phase_completion_progress`. **Викликає:** нічого.
- `on_trigger_next_region_reached()` — повторно перевіряє active phase і переводить Army у next Region. **Використовує:** `status`, `phase`, `progress`, `phase_completion_progress`. **Викликає:** `advance_to_next_region()`.
- `check_trigger_camp_reached()` — для final-local/retreat-local phase прогнозує completion. **Використовує:** `phase`, `progress`, `progress_rate`, `phase_completion_progress`. **Викликає:** нічого.
- `on_trigger_camp_reached()` — для final-local виконує `resolve_camp_arrival()`; для retreat-local створює/знаходить CampInRegion retreat Region, переводить Army в Camp і запускає Regrouping. **Використовує:** `status`, `phase`, `progress`, `army`, `current_region`. **Викликає:** `resolve_camp_arrival()` або `Region.get_or_create_camp()`, `Army.finish_movement()`, `Army.enter_camp()`, `Army.start_regrouping()`.

---

## 11. `Recruitment`

Один Castle має одну послідовну Recruitment Queue.

### Прямі характеристики

- `castle`
- `queue` — orders `{soldier_type, remaining_quantity}` у порядку виконання.
- `current_order_index`
- `status`

### Динамічні характеристики

- `progress`

### Обчислювальні характеристики

- `current_order` — current queue item або `null`.
- `can_progress` — є current order, `castle.empty_food == false`, `castle.player.empty_coins == false` і `castle.barracks_free_capacity > 0`.
- `progress_rate` — effective recruitment rate поточного Soldier; `0`, якщо `can_progress == false`.
- `required_progress` — recruitment time/progress current Soldier Type.
- `current_recruit_finished` — `progress >= required_progress`.

### User methods

- `user_add_recruitment_order(soldier_type, quantity)` — додає prepaid order у Queue. **Використовує:** `castle`, `queue`, `current_order`, `castle.empty_food`, `castle.barracks_free_capacity`, `castle.player.empty_coins`. **Викликає:** `validate_new_order()`, `Castle.pay_local_cost()`, `Player.pay_global_cost()`.

### Domain methods

- `validate_new_order(type, quantity)` — перевіряє Barracks/recruitment availability, current shortages і cost. **Використовує:** `castle`, `queue`, `castle.empty_food`, `castle.barracks_capacity`, `castle.player.empty_coins`. **Викликає:** `Castle.can_pay_local_cost()`, `Player.can_pay_global_cost()`.
- `finish_current_recruit()` — додає Soldier у reserve, зменшує quantity order, переходить до наступного order й скидає progress. **Використовує:** `current_order`, `current_order_index`, `queue`, `progress`, `castle`. **Викликає:** `Castle.add_recruited_soldier()`.

### Triggers

- `can_progress` `[state trigger]`.
- `current_recruit_finished` `[event trigger]`.

### Trigger methods

- `check_trigger_can_progress()` — визначає pause/resume state Recruitment. **Використовує:** `can_progress`, `status`. **Викликає:** нічого.
- `on_trigger_can_progress()` — direct state не змінює; boundary фіксує новий `progress_rate`. **Використовує:** `can_progress`. **Викликає:** нічого.
- `check_trigger_current_recruit_finished()` — при `progress_rate > 0` прогнозує completion current Soldier. **Використовує:** `progress`, `progress_rate`, `required_progress`, `current_recruit_finished`, `current_order`. **Викликає:** нічого.
- `on_trigger_current_recruit_finished()` — повторно перевіряє completion і завершує одного Soldier. **Використовує:** `current_recruit_finished`, `current_order`, `status`. **Викликає:** `finish_current_recruit()`.

---

## 12. `BuildingUpgrade`

Один instance = один конкретний process upgrade однієї Building. Після завершення не видаляється фізично, а переходить у terminal status.

### Прямі характеристики

- `castle`
- `building_type`
- `target_level`
- `status`

### Динамічні характеристики

- `progress`

### Обчислювальні характеристики

- `progress_rate` — rate upgrade progress; у поточних правилах конфігураційний.
- `required_progress` — required time/progress до `target_level`.
- `is_complete` — `progress >= required_progress`.

### User methods

- `user_start_building_upgrade(castle, building_type)` — validation, prepayment та запуск process. **Використовує:** `castle`, `building_type`, `target_level`, `castle.levels`, `castle.building_upgrades[]`. **Викликає:** `Castle.can_start_building_upgrade()`, `Castle.pay_local_cost()`, `Player.pay_global_cost()`.

### Domain methods

- `complete()` — застосовує target level і завершує process. **Використовує:** `castle`, `building_type`, `target_level`, `status`, `is_complete`. **Викликає:** `Castle.apply_building_upgrade()`.

### Triggers

- `complete` `[event trigger]`.

### Trigger methods

- `check_trigger_complete()` — прогнозує completion. **Використовує:** `progress`, `progress_rate`, `required_progress`, `is_complete`, `status`. **Викликає:** нічого.
- `on_trigger_complete()` — повторно перевіряє active/completed state і застосовує upgrade. **Використовує:** `is_complete`, `status`. **Викликає:** `complete()`.

---

## 13. `ResourceSiteUpgrade`

Один instance = один process upgrade вибраної кількості ResourceSite одного resource type/level. Після завершення фізично не видаляється.

### Прямі характеристики

- `region`
- `resource_type`
- `target_level`
- `quantity`
- `status`

### Динамічні характеристики

- `progress`

### Обчислювальні характеристики

- `progress_rate` — конфігураційний rate process.
- `required_progress` — required progress для `target_level` і `quantity`.
- `is_complete` — `progress >= required_progress`.

### User methods

- `user_start_resource_site_upgrade(region, resource_type, quantity)` — validation layered rule, prepayment і запуск process. **Використовує:** `region`, `resource_type`, `target_level`, `quantity`, `region.resource_sites`, `region.resource_site_upgrades[]`, `region.castle`, `region.player`. **Викликає:** `Region.can_start_resource_site_upgrade()`, `Castle.pay_local_cost()`, `Player.pay_global_cost()`.

### Domain methods

- `complete()` — застосовує level changes і завершує process. **Використовує:** `region`, `resource_type`, `target_level`, `quantity`, `is_complete`, `status`. **Викликає:** `Region.apply_resource_site_upgrade()`.

### Triggers

- `complete` `[event trigger]`.

### Trigger methods

- `check_trigger_complete()` — прогнозує completion. **Використовує:** `progress`, `progress_rate`, `required_progress`, `is_complete`, `status`. **Викликає:** нічого.
- `on_trigger_complete()` — повторно перевіряє process і застосовує ResourceSite changes. **Використовує:** `is_complete`, `status`. **Викликає:** `complete()`.

---

## 14. `KnightReplacement`

Один instance = один process створення replacement Knight для звільненого Palace slot. Process створюється Castle як наслідок Knight death. Після завершення фізично не видаляється.

### Прямі характеристики

- `castle`
- `status`

### Динамічні характеристики

- `progress`

### Обчислювальні характеристики

- `is_queue_head` — ID цього instance дорівнює `castle.active_knight_replacement_id`.
- `progress_rate` — replacement rate, якщо `is_queue_head == true`, і `0` для waiting processes.
- `required_progress` — required replacement time/progress.
- `is_complete` — `progress >= required_progress`.

### User methods

- немає: creation є наслідком Knight death через `Castle.create_knight_replacement()`.

### Domain methods

- `complete()` — створює replacement Knight і переводить process у terminal state. **Використовує:** `castle`, `is_queue_head`, `is_complete`, `status`. **Викликає:** `Castle.complete_knight_replacement()`.

### Triggers

- `complete` `[event trigger]`.

### Trigger methods

- `check_trigger_complete()` — тільки для queue head прогнозує completion. **Використовує:** `is_queue_head`, `progress`, `progress_rate`, `required_progress`, `is_complete`, `status`. **Викликає:** нічого.
- `on_trigger_complete()` — повторно перевіряє, що process усе ще queue head і complete, після чого створює Knight. **Використовує:** `is_queue_head`, `is_complete`, `status`. **Викликає:** `complete()`.