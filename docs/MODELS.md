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

Рівні всіх Building одного Castle зберігаються як одна характеристика `levels`.

Wood / Stone / Iron зберігаються як одна характеристика `resources` — один value object / helper class із трьома значеннями.

У секціях methods:

- **Використовує** — характеристики цієї Model та безпосередньо пов'язаних Model, які потрібні method.
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
- `coin_balance` — effective rate зміни Coins; агрегує поточні Coin income/upkeep з `regions[]`, `castles[]` та інших уже піднятих у них економічних характеристик.
- `gold_balance` — effective rate зміни Gold; агрегує Gold production з `regions[]`.
- `silver_balance` — effective rate зміни Silver; агрегує Silver production з `regions[]`.
- `empty_coins` — `coins == 0 && coin_balance <= 0`.

### User methods

- немає зафіксованих на цей момент.

### Domain methods

- `can_pay_global_cost(cost)` — перевіряє одноразову вартість у Coins/Gold/Silver. **Використовує:** `coins`, `gold`, `silver`. **Викликає:** нічого.
- `pay_global_cost(cost)` — списує одноразову глобальну частину вартості після validation. **Використовує:** `coins`, `gold`, `silver`. **Викликає:** `can_pay_global_cost()`.
- `add_coins(amount)` — зараховує разовий Coin reward, зокрема City Raid reward. **Використовує:** `coins`. **Викликає:** нічого.
- `create_castle(region, name, founder_knight)` — створює новий Castle після завершення Founding. **Використовує:** `castles[]`, параметри нового Castle. **Викликає:** `Region.become_castle_region()`, `Knight.change_home_castle()` і initialization нового `Castle`.

### Triggers

- `empty_coins` `[state trigger]` — реагує на зміну стану `empty_coins`; boundary потрібний для Recruitment, City Wealth growth та інших залежних rates.

### Trigger methods

- `check_trigger_empty_coins()` — визначає поточний boolean state `empty_coins` і, якщо `coin_balance < 0`, прогнозує момент досягнення `coins == 0`. **Використовує:** `coins`, `coin_balance`, `empty_coins`. **Викликає:** нічого.
- `on_trigger_empty_coins()` — окремої прямої зміни Player state не робить; GameEvent фіксує temporal boundary, після якої залежні computed rates перераховуються через звичайний propagation. **Використовує:** `empty_coins`. **Викликає:** нічого.

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

- `*regions[]` — Region, приєднані до цього Castle.
- `*knights[]` — Knight, для яких цей Castle є home Castle.
- `*building_upgrades[]` — BuildingUpgrade цього Castle, включно з history instances; active визначається їх `status`.
- `*recruitment` — active Recruitment Queue Castle або `null`.
- `*knight_replacements[]` — KnightReplacement цього Castle.
- `resource_balance` — effective rate для `wood`, `stone`, `iron`; агрегує `Region.resource_flow_to_castle` з `regions[]` і враховує storage boundaries.
- `food_balance` — effective rate зміни Food; агрегує `Region.food_flow_to_castle`, Population та військове споживання, яке належить цьому Castle.
- `coin_balance` — Coin income/upkeep Castle: Building income, Bank multiplier, Palace upkeep та військові витрати, які відносяться до цього Castle.
- `warehouse_capacity` — Capacity Warehouse відповідно до `levels`.
- `granary_capacity` — Capacity Granary відповідно до `levels`.
- `storage_full` — структура boolean для `wood`, `stone`, `iron`, `food`.
- `empty_food` — `food == 0 && food_balance <= 0`.
- `barracks_capacity` — Barracks Capacity за `levels`.
- `barracks_used` — Soldier reserve + Soldier у Unit цього Castle, які перебувають у Castle/Barracks.
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
- `can_house_unit(knight)` — перевіряє вхід Unit у Barracks. **Використовує:** `barracks_free_capacity`, `Knight.soldier_count`. **Викликає:** нічого.
- `move_soldiers_between_reserve_and_knight(knight, composition_delta)` — переносить Soldier тільки `Castle reserve <-> Knight`. **Використовує:** `soldier_reserve`, `barracks_capacity`, `Knight.castle`, `Knight.location_state`, `Knight.soldiers`. **Викликає:** `Knight.set_soldiers()`.
- `add_recruited_soldier(type)` — додає завершеного recruit у reserve. **Використовує:** `soldier_reserve`, `barracks_free_capacity`. **Викликає:** нічого.
- `can_start_building_upgrade(building_type)` — перевіряє prerequisite та відсутність іншого active upgrade цієї Building. **Використовує:** `levels`, `building_upgrades[]`, local resources, `player`. **Викликає:** `can_pay_local_cost()`, `Player.can_pay_global_cost()`.
- `apply_building_upgrade(building_type, target_level)` — встановлює новий level. Для Palace increase створює Knight для нового slot. **Використовує:** `levels`, `palace_capacity`, `knights[]`. **Викликає:** `create_knight_for_palace_slot()` за потреби.
- `create_knight_for_palace_slot()` — створює нового Knight у вільному Palace slot. **Використовує:** `palace_capacity`, `knights[]`. **Викликає:** initialization нового `Knight`.
- `create_knight_replacement()` — створює KnightReplacement після смерті home Knight. **Використовує:** `knight_replacements[]`, `palace_capacity`. **Викликає:** initialization нового `KnightReplacement`.
- `complete_knight_replacement(replacement)` — створює replacement Knight і завершує відповідний Palace slot replacement. **Використовує:** `active_knight_replacement_id`, `knights[]`, `palace_capacity`. **Викликає:** `create_knight_for_palace_slot()`.
- `can_annex_region(region, camp)` — перевіряє Governor Capacity, допустимий зв'язок Region з цим Castle і наявність у Camp хоча б одного Unit з home Castle = цей Castle. **Використовує:** `governor_capacity`, `external_region_count`, `regions[]`, `region`, `CampInRegion.home_castle_ids[]`. **Викликає:** нічого.
- `recalculate_region_connections()` — централізовано перераховує `Region.is_connection_valid` після occupation/loss/restore/annexation і переводить остаточно disconnected Region у Neutral за правилами гри. **Використовує:** `region`, `regions[]`, топологію через `Region.neighbors[]`, їх ownership/occupation. **Викликає:** `Region.set_connection_valid()`, `Region.become_neutral()` для Region, які юридично втрачаються.

### Triggers

- `empty_food` `[state trigger]` — реагує на зміну `empty_food`.
- `storage_capacity` `[state trigger]` — окремо відстежує full/not-full для Food та кожного компонента `resources`.

### Trigger methods

- `check_trigger_empty_food()` — визначає current state та прогнозує момент `food == 0`, якщо `food_balance < 0`. **Використовує:** `food`, `food_balance`, `empty_food`. **Викликає:** нічого.
- `on_trigger_empty_food()` — прямого state не змінює; boundary потрібний для перерахунку Recruitment та інших залежних rates. **Використовує:** `empty_food`. **Викликає:** нічого.
- `check_trigger_storage_capacity()` — визначає `storage_full` і для кожного ресурсу з позитивним effective rate прогнозує момент досягнення Capacity. **Використовує:** `resources`, `food`, `resource_balance`, `food_balance`, `warehouse_capacity`, `granary_capacity`, `storage_full`. **Викликає:** нічого.
- `on_trigger_storage_capacity()` — прямого state не змінює; boundary змушує наступний commit зафіксувати нові effective balances при full/not-full transition. **Використовує:** `storage_full`. **Викликає:** нічого.

---

## 3. `Region`

### Прямі характеристики

- `player` — формальний owner; `null` для Neutral Region.
- `castle` — Castle, до якого Region приєднана; `null`, якщо не належить Castle.
- `occupier_player_id` — ID поточного occupier для Occupied Region; `null`, якщо Region не Occupied.
- `resource_sites` — структура ResourceSite за типами ресурсів і рівнями.
- `allow_transit` — правило Transit для Owned Region.
- `is_connection_valid` — чи має Owned Region чинний зв'язок зі своїм Castle.
- `neutral_defense` — актуальний стан Neutral Defense.
- `neutral_defense_recovery_started_at` — Game Time початку recovery, якщо recovery активний.

### Динамічні характеристики

- немає зафіксованих на цей момент.

### Обчислювальні характеристики

- `*neighbors[]` — шість сусідніх Region, визначені топологією карти.
- `*armies[]` — Army, що фізично знаходяться в Region за `Army.current_region`.
- `*camps[]` — active CampInRegion цієї Region.
- `*city` — City цієї Region або `null`.
- `*resource_site_upgrades[]` — ResourceSiteUpgrade цієї Region.
- `*castle_foundings[]` — CastleFounding у цій Region.
- `camp_player_ids[]` — ID Player, для яких у Region є active CampInRegion; scalar IDs, не Player references.
- `valid_connected_neighbor_player_ids[]` — ID Player, для яких серед `neighbors[]` є Owned Region з `is_connection_valid == true`.
- `is_occupied` — `occupier_player_id != null`.
- `has_any_troops` — чи є в `armies[]` будь-які фізично присутні війська, включно з Movement/Regrouping.
- `has_active_founding` — чи є в `castle_foundings[]` active CastleFounding.
- `local_resource_production` — production Wood/Stone/Iron за `resource_sites`.
- `local_food_production` — Food production за `resource_sites`.
- `gold_production` — Gold production за `resource_sites`.
- `silver_production` — Silver production за `resource_sites`.
- `resource_flow_to_castle` — потік Wood/Stone/Iron до attached Castle з урахуванням `is_connection_valid`, occupation та DistanceEfficiency.
- `food_flow_to_castle` — Food flow/deficit для attached Castle за правилами local production/consumption та DistanceEfficiency.
- `coin_balance_for_owner` — recurring Coin effect Region для формального owner: upkeep, City income та інші region-level effects.
- `city_wealth_growth_enabled` — true лише для Owned, non-Occupied Region і коли owner Player не має `empty_coins`; Region може читати `player.empty_coins`, бо `player` є її прямим relationship.
- `neutral_defense_recovery_at` — Game Time повного recovery, якщо `neutral_defense_recovery_started_at != null`.

### User methods

- `user_change_allow_transit(value)` — змінює Transit rule власної Region. **Використовує:** `player`, `allow_transit`. **Викликає:** нічого.

### Domain methods

- `find_active_camp(player)` — знаходить active CampInRegion цього Player у `camps[]`. **Використовує:** `camps[]`. **Викликає:** нічого.
- `get_or_create_camp(player)` — повертає існуючий active CampInRegion або створює новий. **Використовує:** `camps[]`. **Викликає:** `find_active_camp()` і initialization нового `CampInRegion` за потреби.
- `resolve_arrival(army, movement)` — визначає актуальний результат Arrival Resolution: безбойовий Camp або створення CombatSituation за current ownership/occupation/presence. **Використовує:** `player`, `occupier_player_id`, `camp_player_ids[]`, Castle Region status, `allow_transit`. **Викликає:** `get_or_create_camp()`, `Army.enter_camp()` або factory/domain initialization `CombatSituation`.
- `set_occupied_by(player)` — встановлює Occupation після переходу переможця в Camp чужої Owned Region. **Використовує:** `player`, `castle`, `occupier_player_id`. **Викликає:** `Castle.recalculate_region_connections()` у attached Castle.
- `restore_owner_control()` — очищує Occupation, коли occupier повністю залишає Region. **Використовує:** `occupier_player_id`, `castle`. **Викликає:** `Castle.recalculate_region_connections()`.
- `annex_to(player, castle)` — змінює formal owner/attached Castle, очищує occupation і завершує Neutral/Occupied state transition. **Використовує:** `player`, `castle`, `occupier_player_id`, `neutral_defense`. **Викликає:** `Castle.recalculate_region_connections()` для старого й нового Castle за потреби.
- `become_neutral()` — остаточно прибирає ownership при втраті connectivity. **Використовує:** `player`, `castle`, `occupier_player_id`. **Викликає:** `City` напряму не змінює; його Wealth зберігається й реагує через computed state.
- `become_castle_region(new_castle, player)` — робить Region Castle Region після Founding. **Використовує:** `player`, `castle`, occupation state. **Викликає:** `Castle.recalculate_region_connections()` старого Castle, якщо Region була від'єднана від нього.
- `set_connection_valid(value)` — змінює збережений structural connectivity flag. **Використовує:** `is_connection_valid`. **Викликає:** нічого.
- `apply_resource_site_upgrade(resource_type, target_level, quantity)` — переносить `quantity` ResourceSite у наступний level. **Використовує:** `resource_sites`, `resource_site_upgrades[]`. **Викликає:** нічого.
- `can_start_resource_site_upgrade(resource_type, quantity)` — перевіряє layered-upgrade rule та concurrent upgrades. **Використовує:** `resource_sites`, `resource_site_upgrades[]`, `castle`, `player`. **Викликає:** `Castle.can_pay_local_cost()`, `Player.can_pay_global_cost()` через direct related Castle/Player за правилами вартості.
- `on_army_presence_changed()` — керує lifecycle recovery Neutral Defense: при появі будь-яких військ recovery не рахується; після виходу останніх військ із знищеної Neutral Defense запускає recovery. **Використовує:** `player`, `neutral_defense`, `has_any_troops`, `neutral_defense_recovery_started_at`. **Викликає:** нічого.
- `destroy_neutral_defense()` — фіксує повне знищення Neutral Defense. **Використовує:** `neutral_defense`, `has_any_troops`, `neutral_defense_recovery_started_at`. **Викликає:** `on_army_presence_changed()`.

### Triggers

- `neutral_defense_recovery_complete` `[event trigger]` — повне recovery Neutral Defense після конфігураційного часу без військ у Region.

### Trigger methods

- `check_trigger_neutral_defense_recovery_complete()` — якщо recovery active і `has_any_troops == false`, повертає час до `neutral_defense_recovery_at`. **Використовує:** `neutral_defense`, `has_any_troops`, `neutral_defense_recovery_started_at`, `neutral_defense_recovery_at`. **Викликає:** нічого.
- `on_trigger_neutral_defense_recovery_complete()` — повторно перевіряє відсутність військ і відновлює Neutral Defense повністю; очищує recovery start. **Використовує:** `has_any_troops`, `neutral_defense`, `neutral_defense_recovery_started_at`. **Викликає:** нічого.

---

## 4. `City`

### Прямі характеристики

- `region`

### Динамічні характеристики

- `wealth`

### Обчислювальні характеристики

- `wealth_balance` — effective rate зміни Wealth; використовує `region.city_wealth_growth_enabled` та конфігурацію.
- `coin_income` — recurring Coin income City для owner Region; `0`, якщо Region Neutral або Occupied.
- `raid_reward` — поточний разовий Coin reward, який може бути отриманий Raid за current Wealth/configuration.

### User methods

- `user_raid_city(camp)` — виконує Raid локальними Camp troops. **Використовує:** `region`, `wealth`, `raid_reward`, `CampInRegion.player`, `CampInRegion.region`, Neutral Defense/raid conditions через `region`. **Викликає:** `apply_raid()`, `Player.add_coins()`; якщо Raid вимагає combat із Neutral Defense — створення/налаштування `CombatSituation` перед фактичним reward.

### Domain methods

- `apply_raid()` — застосовує наслідок Raid до Wealth/тимчасових city values відповідно до конфігурації. **Використовує:** `wealth`, `raid_reward`. **Викликає:** нічого.

### Triggers

- немає зафіксованих на цей момент.

---

## 5. `Knight`

Knight разом зі своїми Soldier представляє gameplay Unit; окремої backend-моделі Unit немає.

### Прямі характеристики

- `name`
- `castle` — home Castle.
- `army` — поточна Army.
- `soldiers` — кількість Soldier за Type.
- `base_strength` — особиста базова бойова сила Knight.
- `location_state` — зокрема Castle/Barracks або Camp-side state, якщо це потрібно окремо від Army state.
- `status` — active/dead/replaced lifecycle state.

### Динамічні характеристики

- `experience`

### Обчислювальні характеристики

- `soldier_count` — сума `soldiers`.
- `experience_coefficient` — coefficient Knight за Experience.
- `unit_attack_strength` — Attack strength Soldier + `base_strength`, помножені на `experience_coefficient`.
- `unit_defense_strength` — Defense strength Soldier + `base_strength`, помножені на `experience_coefficient`.
- `food_consumption` — Food consumption Unit у Camp/Castle режимі.
- `coin_upkeep` — recurring Coin upkeep Unit за current placement/movement state.
- `current_region` — Region Army цього Knight; піднімає `army.current_region`, щоб інші Model не проходили `Knight -> Army -> Region`.
- `is_regrouping` — чи `army.state` є Regrouping.
- `experience_balance` — passive effective rate Experience.

### User methods

- `user_enter_castle()` — переводить Unit із Camp у home Castle/Barracks. **Використовує:** `castle`, `army`, `soldier_count`, `location_state`. **Викликає:** `Castle.can_house_unit()`; після validation змінює `location_state`.
- `user_leave_castle()` — переводить Unit із Castle/Barracks у Camp у Castle Region. **Використовує:** `castle`, `army`, `location_state`. **Викликає:** `Region.get_or_create_camp()` через `castle.region`, `Army.enter_camp()` за потреби.
- `user_change_unit_composition(composition_delta)` — змінює Soldier composition тільки у home Castle. **Використовує:** `castle`, `location_state`, `soldiers`. **Викликає:** `Castle.move_soldiers_between_reserve_and_knight()`.

### Domain methods

- `set_soldiers(new_composition)` — контрольовано змінює `soldiers`. **Використовує:** `soldiers`. **Викликає:** нічого.
- `set_army(army)` — змінює membership Unit в Army при merge/split/dissolve. **Використовує:** `army`. **Викликає:** нічого.
- `add_battle_experience(amount)` — додає разовий battle Experience. **Використовує:** `experience`. **Викликає:** нічого.
- `apply_casualties(soldier_losses, knight_dies)` — застосовує визначені CombatSituation casualties; Knight death дозволений лише після втрати всіх його Soldier. **Використовує:** `soldiers`, `status`. **Викликає:** `die()` за потреби.
- `die()` — переводить Knight у dead state та запускає replacement у home Castle. **Використовує:** `status`, `castle`, `army`. **Викликає:** `Castle.create_knight_replacement()`.
- `change_home_castle(new_castle)` — змінює home Castle, зокрема для founder після Founding. **Використовує:** `castle`. **Викликає:** нічого.

### Triggers

- немає зафіксованих на цей момент.

---

## 6. `Army`

### Прямі характеристики

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
- `player_id` — ID Player Army, отриманий з owner home Castle її Knight; scalar ID для порівнянь.
- `home_castle_ids[]` — унікальні ID home Castle усіх `knights[]`; scalar IDs.
- `soldier_count` — загальна кількість Soldier.
- `attack_strength` — сума `Knight.unit_attack_strength` з Commander coefficient.
- `defense_strength` — сума `Knight.unit_defense_strength` з Commander coefficient.
- `food_consumption` — сумарне Camp Food consumption.
- `coin_upkeep` — сумарний upkeep за current Army state.
- `regrouping_progress_rate` — rate Regrouping; у V1 конфігураційна константа, ненульова тільки в Regrouping.

### User methods

- `user_merge_armies(armies, commander, thresholds)` — об'єднує Army/Unit одного Player в одному Camp/місці. **Використовує:** `camp`, `state`, `knights[]`, thresholds, input Army `camp/state/player_id`. **Викликає:** `CampInRegion.validate_reorganization()`, `Knight.set_army()` для всіх Unit; старі Army логічно завершуються, створюється нова Army.
- `user_split_army(groups)` — розділяє Army на нові Army, які успадковують current thresholds. **Використовує:** `camp`, `state`, `knights[]`, thresholds. **Викликає:** `CampInRegion.validate_reorganization()`, `Knight.set_army()`; створює нові Army.
- `user_change_commander(knight)` — змінює Commander-in-Chief. **Використовує:** `knights[]`, `commander`, `state`. **Викликає:** нічого.
- `user_change_combat_thresholds(values)` — змінює три thresholds; дозволено також у Movement/Regrouping, але active CombatSituation використовує вже locked values. **Використовує:** threshold characteristics. **Викликає:** нічого.

### Domain methods

- `enter_region(region)` — фіксує фізичний вхід Army у Region. **Використовує:** `current_region`. **Викликає:** `Region.on_army_presence_changed()` для old/new Region.
- `enter_camp(camp)` — переводить Army у Camp і встановлює прямий `camp`. **Використовує:** `current_region`, `camp`, `state`, `player_id`. **Викликає:** `CampInRegion.accept_army()`/validation і `Region.on_army_presence_changed()` за потреби.
- `leave_camp()` — очищує `camp` перед Movement/Retreat/іншим виходом. **Використовує:** `camp`, `state`. **Викликає:** `CampInRegion.on_army_left()` після зміни relationship.
- `start_movement(movement)` — переводить Army у Movement. **Використовує:** `state`, `camp`, `current_region`, `movement`. **Викликає:** `leave_camp()`.
- `finish_movement()` — завершує Movement-side state перед Camp/interaction. **Використовує:** `state`, `movement`. **Викликає:** нічого.
- `stop_transit_for_defense(combat)` — спеціальний виняток: припиняє Transit і долучає Army до defense до battle start. **Використовує:** `state`, `movement`, `current_region`. **Викликає:** `Movement.stop_for_defense()`, `CombatSituation.add_defender()`.
- `start_regrouping()` — переводить Army у Regrouping і скидає `regrouping_progress`. **Використовує:** `state`, `regrouping_progress`, `camp`. **Викликає:** нічого.
- `finish_regrouping()` — повертає Army зі стану Regrouping у звичайний Camp. **Використовує:** `state`, `regrouping_progress`, `camp`. **Викликає:** нічого.
- `retreat_to(region)` — переводить loser у retreat Region та запускає локальний рух до Camp/Regrouping. **Використовує:** `current_region`, `camp`, `state`. **Викликає:** `leave_camp()`, `enter_region()`, `start_regrouping()` у момент досягнення Camp відповідно до retreat flow.
- `dissolve_after_commander_death()` — розпускає Army на окремі Unit після смерті Commander. **Використовує:** `commander`, `knights[]`, thresholds. **Викликає:** `Knight.set_army()` та створення окремих Army/Unit containers.

### Triggers

- `regrouping_complete` `[event trigger]`.

### Trigger methods

- `check_trigger_regrouping_complete()` — у Regrouping прогнозує момент `regrouping_progress` completion. **Використовує:** `state`, `regrouping_progress`, `regrouping_progress_rate`, конфігураційний required progress. **Викликає:** нічого.
- `on_trigger_regrouping_complete()` — повторно перевіряє state і завершує Regrouping. **Використовує:** `state`, `regrouping_progress`. **Викликає:** `finish_regrouping()`.

---

# Persistent interaction/process models

## 7. `CombatSituation`

### Прямі характеристики

- `region`
- `attacker` — Attacker Army.
- `defenders[]` — Defender Army/Unit containers.
- `combat_type` — player-vs-player, Neutral Defense та потрібний context.
- `defenders_retreat_decisions` — pre-battle retreat decisions.
- `locked_combat_parameters` — thresholds, Target/Incidental classification та інші parameters, які фіксуються до resolve.
- `status`

### Динамічні характеристики

- `battle_start_progress`

### Обчислювальні характеристики

- `battle_start_progress_rate` — у V1 конфігураційна константа для відповідного combat start delay.
- `required_battle_start_progress` — progress boundary battle start.
- `is_battle_valid` — чи сторони та умови combat все ще актуальні.
- `defender_loss_threshold` — мінімальний ненульовий Defense Loss Threshold після pre-battle zero-threshold retreats і forced-max cases.
- `attacker_loss_threshold` — Target або Incidental threshold із `locked_combat_parameters`.
- `attacker_strength` — current locked/start strength Attacker для потрібного combat mode.
- `defender_strength` — сума strength Defender Army, які реально беруть участь.

### User methods

- `user_stop_transit_for_defense(army)` — додає допустиму own Transit Army до defense до battle start. **Використовує:** `region`, `defenders[]`, `status`, `battle_start_progress`. **Викликає:** `Army.stop_transit_for_defense()` і `add_defender()`.
- `user_set_pre_battle_retreat_decision(army, destination)` — фіксує/змінює pre-battle retreat до battle start. **Використовує:** `defenders[]`, `defenders_retreat_decisions`, `status`, `region`. **Викликає:** `get_legal_retreat_regions()` для validation.
- `user_attack_neutral_defense(attacker, region)` — ініціалізує combat проти Neutral Defense. **Використовує:** `attacker`, `region.neutral_defense`, `region`, Camp presence Attacker. **Викликає:** initialization `CombatSituation`, `lock_combat_parameters()`.

### Domain methods

- `add_defender(army)` — додає Army до Defender side до battle start. **Використовує:** `defenders[]`, `status`, `region`. **Викликає:** нічого.
- `lock_combat_parameters()` — фіксує thresholds/classification/параметри, які не повинні змінитися після start. **Використовує:** `attacker`, `defenders[]`, їх thresholds, Route destination context. **Викликає:** `get_legal_retreat_regions()` для forced-max checks.
- `get_legal_retreat_regions(army, role)` — визначає legal Retreat destinations за геометрією й current Region states. **Використовує:** `region.neighbors[]`, combat context, Army entry/source direction та сусідні Region ownership/presence. **Викликає:** нічого.
- `resolve_combat()` — виконує combat calculation, Luck reroll on exact tie, loss fractions, casualties та winner/loser consequences. **Використовує:** `combat_type`, `locked_combat_parameters`, `attacker_strength`, `defender_strength`, thresholds, `region`. **Викликає:** `apply_casualties()`, `apply_result()`.
- `apply_casualties(result)` — розподіляє casualties між Unit/Soldier, виконує Soldier-before-Knight rule і battle Experience. **Використовує:** `attacker`, `defenders[]`, combat result, deterministic RNG context. **Викликає:** `Knight.apply_casualties()`, `Knight.add_battle_experience()`, `Army.dissolve_after_commander_death()` за потреби.
- `apply_result(result)` — виконує Retreat, Camp/Occupation/Transit continuation та Neutral Defense result. **Використовує:** `region`, `attacker`, `defenders[]`, `combat_type`. **Викликає:** `Army.retreat_to()`, `Army.start_regrouping()`, `Region.destroy_neutral_defense()`, `Region.set_occupied_by()`, `Region.get_or_create_camp()`, `Army.enter_camp()` або Movement continuation залежно від context.

### Triggers

- `battle_start` `[event trigger]`.

### Trigger methods

- `check_trigger_battle_start()` — прогнозує completion battle start delay; при scheduled check повторно враховує `is_battle_valid`. **Використовує:** `status`, `battle_start_progress`, `battle_start_progress_rate`, `required_battle_start_progress`, `is_battle_valid`. **Викликає:** нічого.
- `on_trigger_battle_start()` — якщо combat усе ще valid, фіксує остаточні parameters і resolve-ить battle; якщо invalid — завершує CombatSituation без combat. **Використовує:** `status`, `is_battle_valid`. **Викликає:** `lock_combat_parameters()`, `resolve_combat()`.

---

## 8. `CampInRegion`

Один active instance представляє один безперервний епізод присутності Army конкретного Player у Camp конкретної Region. Модель існує для власної, Neutral та чужої Region. Коли остання Army залишає Camp, instance логічно деактивується; фізичний запис зберігається. Наступна поява військ створює новий instance.

### Прямі характеристики

- `player`
- `region`
- `status`

### Динамічні характеристики

- `control_progress`

### Обчислювальні характеристики

- `*armies[]` — Army, для яких цей active CampInRegion є `Army.camp`.
- `home_castle_ids[]` — унікальні scalar ID home Castle, присутні серед `Army.home_castle_ids[]`.
- `has_valid_adjacent_owned_region` — чи `region.valid_connected_neighbor_player_ids[]` містить ID `player`.
- `can_progress` — чи Annexation control progress може накопичуватися зараз: Region Neutral/Occupied для цього Player, Neutral Defense знищений якщо потрібний, є eligible Camp presence, немає foreign eligible Camp presence, є valid adjacent owned Region і процес не заблокований іншими правилами.
- `control_progress_rate` — effective rate control progress; `0`, якщо `can_progress == false`.
- `required_control_progress` — потрібний progress для Neutral/Occupied Region за конфігурацією.
- `is_ready_for_annexation` — `control_progress >= required_control_progress`.

### User methods

- `user_annex_region(castle)` — виконує ручну Annexation. **Використовує:** `player`, `region`, `is_ready_for_annexation`, `home_castle_ids[]`, `region.camp_player_ids[]`, `region.has_active_founding`, `castle`. **Викликає:** `can_annex_to()`, `Castle.can_annex_region()`, `Region.annex_to()`.

### Domain methods

- `accept_army(army)` — validation, що Army належить `player` і знаходиться в `region`; actual relationship встановлює сама Army. **Використовує:** `player`, `region`, `armies[]`, `Army.player_id`, `Army.current_region`. **Викликає:** нічого.
- `on_army_left(army)` — після очищення `Army.camp` перевіряє, чи Camp спорожнів; якщо так — деактивує instance, а control progress більше не зберігається для нового епізоду. **Використовує:** `armies[]`, `status`, `control_progress`, `region`. **Викликає:** `Region.restore_owner_control()` якщо це був останній Camp occupier і правила Occupation вимагають restore.
- `validate_reorganization(armies)` — перевіряє, що всі Army належать цьому Camp і не перебувають у забороненому стані. **Використовує:** `armies[]`, input Army `state/camp/player_id`. **Викликає:** нічого.
- `can_annex_to(castle)` — перевіряє Camp-level умови Annexation перед Castle-specific validation. **Використовує:** `is_ready_for_annexation`, `region.camp_player_ids[]`, `region.has_active_founding`, `home_castle_ids[]`. **Викликає:** `Castle.can_annex_region()`.

### Triggers

- `can_progress` `[state trigger]`.
- `ready_for_annexation` `[event trigger]`.

### Trigger methods

- `check_trigger_can_progress()` — визначає state `can_progress`; якщо зараз progress іде, також дає engine можливість перепланувати completion boundary через `ready_for_annexation`. **Використовує:** `can_progress`, `control_progress_rate`, `region`, `armies[]`. **Викликає:** нічого.
- `on_trigger_can_progress()` — прямого state не змінює; окрема boundary GameEvent фіксує новий `control_progress_rate`. Якщо Camp уже empty, lifecycle закривається через `on_army_left()`, а не цим trigger-ом. **Використовує:** `can_progress`. **Викликає:** нічого.
- `check_trigger_ready_for_annexation()` — якщо `control_progress_rate > 0`, прогнозує момент досягнення `required_control_progress`; якщо уже досягнуто — повертає `0`. **Використовує:** `control_progress`, `control_progress_rate`, `required_control_progress`, `is_ready_for_annexation`. **Викликає:** нічого.
- `on_trigger_ready_for_annexation()` — не виконує Annexation автоматично; фіксує GameEvent досягнення eligibility, після якої `is_ready_for_annexation == true`. **Використовує:** `is_ready_for_annexation`, `status`. **Викликає:** нічого.

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

- `founder_valid` — founder Knight живий, знаходиться в потрібній Region, не має Soldier; Regrouping не скасовує процес, але не є progress-eligible Camp presence.
- `can_progress` — `founder_valid`, founder не Regrouping, немає foreign eligible Camp Player у `region.camp_player_ids[]` та виконані інші pause conditions.
- `progress_rate` — effective Founding rate; `0`, якщо `can_progress == false`.
- `required_progress` — required Founding progress.
- `is_complete` — `progress >= required_progress`.

### User methods

- `user_start_castle_founding(player, region, founder_knight, castle_name)` — створює/ініціалізує Founding і списує start cost. **Використовує:** `player`, `region`, `founder_knight`, `founder_knight.castle`, `region.camp_player_ids[]`, `region.has_active_founding`. **Викликає:** `validate_start()`, `Castle.pay_local_cost()`, `Player.pay_global_cost()`.

### Domain methods

- `validate_start()` — перевіряє Neutral/own Region rules, founder без Soldier, фізичну Camp presence, відсутність blocking foreign Camp та інших active Founding. **Використовує:** `player`, `region`, `founder_knight`, `founder_valid`, `region.camp_player_ids[]`, `region.has_active_founding`. **Викликає:** нічого.
- `cancel()` — terminal state без refund, якщо founder leave/die/отримує Soldier або інша cancel-condition. **Використовує:** `status`, `founder_valid`. **Викликає:** нічого.
- `complete()` — створює Castle і переводить Region/founder у новий стан. **Використовує:** `player`, `region`, `founder_knight`, `castle_name`, `is_complete`. **Викликає:** `Player.create_castle()`; далі створений Castle отримує стартові Warehouse/Granary/Palace levels згідно з Founding rules.

### Triggers

- `can_progress` `[state trigger]`.
- `complete` `[event trigger]`.

### Trigger methods

- `check_trigger_can_progress()` — визначає pause/resume state; якщо `founder_valid == false` через cancel-condition, повертає `0` для окремої GameEvent, яка закриє процес. **Використовує:** `founder_valid`, `can_progress`, `status`. **Викликає:** нічого.
- `on_trigger_can_progress()` — якщо founder став invalid через cancel-condition — `cancel()`; інакше прямого state не змінює, а boundary фіксує новий `progress_rate`. **Використовує:** `founder_valid`, `can_progress`, `status`. **Викликає:** `cancel()` за потреби.
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

- `user_start_movement(army, route[])` — створює та запускає Movement. **Використовує:** `army`, `route[]`, `army.current_region`, `army.camp`, `army.state`. **Викликає:** `validate_route()`, `Army.start_movement()`, `start_phase()`.
- `user_change_planned_route(route[])` — змінює тільки ще не зафіксовану future частину Route. **Використовує:** `route[]`, `current_region`, `next_region`, `local_exit_region`, `phase`, `status`. **Викликає:** `validate_route()`.

### Domain methods

- `validate_route(route[])` — перевіряє adjacency і заборони Castle Region/Transit для future route. **Використовує:** `route[]`, `current_region`, `Region.neighbors[]`, ownership/transit characteristics Region. **Викликає:** нічого.
- `start_phase(current_region, next_region, phase)` — фіксує локальну мету/exit, скидає `progress` і `direction_revealed`. **Використовує:** `current_region`, `next_region`, `phase`, `local_exit_region`, `progress`, `direction_revealed`, `route[]`. **Викликає:** нічого.
- `stop_for_defense()` — завершує поточний Transit як спеціальний defense exception. **Використовує:** `phase`, `status`, `army`. **Викликає:** `Army.finish_movement()`.
- `advance_to_next_region()` — переводить Army через border і запускає наступну local phase. **Використовує:** `current_region`, `next_region`, `route[]`, `phase`, `status`. **Викликає:** `Army.enter_region()`, `start_phase()`, а при вході/перед combat — відповідну Region/Combat domain logic.
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
- `on_trigger_camp_reached()` — завершує local phase і виконує Arrival Resolution. **Використовує:** `status`, `phase`, `progress`. **Викликає:** `resolve_camp_arrival()`; для retreat flow також `Army.start_regrouping()` після досягнення Camp.

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

- `user_start_resource_site_upgrade(region, resource_type, quantity)` — validation layered rule, prepayment і запуск process. **Використовує:** `region`, `resource_type`, `target_level`, `quantity`, `region.resource_sites`, `region.resource_site_upgrades[]`, `region.castle/player`. **Викликає:** `Region.can_start_resource_site_upgrade()`, `Castle.pay_local_cost()`, `Player.pay_global_cost()`.

### Domain methods

- `complete()` — застосовує level changes і завершує process. **Використовує:** `region`, `resource_type`, `target_level`, `quantity`, `is_complete`, `status`. **Викликає:** `Region.apply_resource_site_upgrade()`.

### Triggers

- `complete` `[event trigger]`.

### Trigger methods

- `check_trigger_complete()` — прогнозує completion. **Використовує:** `progress`, `progress_rate`, `required_progress`, `is_complete`, `status`. **Викликає:** нічого.
- `on_trigger_complete()` — повторно перевіряє process і застосовує ResourceSite changes. **Використовує:** `is_complete`, `status`. **Викликає:** `complete()`.

---

## 14. `KnightReplacement`

Один instance = один process створення replacement Knight для звільненого Palace slot. Process створює Castle як наслідок Knight death. Після завершення фізично не видаляється.

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
