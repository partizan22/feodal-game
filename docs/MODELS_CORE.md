# Backend models — core

Core world/economy models.

Загальні conventions, system-computed semantics та спільні припущення див. у [MODELS.md](MODELS.md).

---

## 1. `Player`

### Прямі характеристики

- `name`

### Динамічні характеристики

- `coins`
- `gold`
- `silver`

### Обчислювальні зв'язки

- `*regions[]` — Region, формальним owner яких є цей Player.
- `*castles[]` — Castle цього Player.
- `*armies[]` — Army цього Player за прямим `Army.player`.
- `*camps[]` — active CampInRegion цього Player за прямим `CampInRegion.player`.

### Інші обчислювальні характеристики

- `coin_balance` — чистий баланс надходжень і витрат Coins до застосування обмежень запасу. Складає `Region.coin_balance_for_owner` із `regions[]`, `Castle.coin_balance` і `Castle.food_coin_compensation` із `castles[]`, звичайний `Army.coin_upkeep` і `Army.food_coin_compensation` для Army у Movement із `armies[]`, а також `CampInRegion.food_coin_compensation` із `camps[]` для Region окрім власних Region з замком. У власній Castle Region локальна Camp-компенсація дорівнює `0`, бо Food усіх власних Camp Unit враховується у Castle. Кожен дефіцит Food компенсується рівно один раз.
- `coins_rate` — `coin_balance` з обмеженням на мінімальний запас `coins = 0`: за нульового запасу від’ємна швидкість обрізається до `0`.
- `gold_balance` — чистий баланс зміни Gold без обмежень запасу; агрегує `Region.gold_income_for_owner` з `regions[]`, тому Occupied/disconnected Region не дають Gold формальному owner.
- `gold_rate` — `gold_balance` з обмеженням на мінімальний запас `gold = 0`.
- `silver_balance` — чистий баланс зміни Silver без обмежень запасу; агрегує `Region.silver_income_for_owner` з `regions[]`, тому Occupied/disconnected Region не дають Silver формальному owner.
- `silver_rate` — `silver_balance` з обмеженням на мінімальний запас `silver = 0`.

- `empty_coins` — `coins == 0 && coin_balance <= 0` (баланс зберігає дефіцит навіть коли `coins_rate = 0`).


### User methods

- немає зафіксованих на цей момент.

### Domain methods

- `can_pay_global_cost(cost)` — перевіряє одноразову вартість у Coins/Gold/Silver.
- `pay_global_cost(cost)` — списує одноразову глобальну частину вартості після validation.
- `add_coins(amount)` — зараховує разовий Coin reward, зокрема City Raid reward.
- `create_castle(region, name, founder_knight)` — після завершення Founding створює Castle з `Warehouse = 1`, `Granary = 1`, `Palace = 1`; викликає `Region.become_castle_region()` і `Knight.change_home_castle()` атомарно. Використовує `Player.castles[]`, `Region.player/castle` і `Knight.castle`. Founder переходить до нового home Castle; попередній Castle одразу ставить одну заміну в `KnightReplacement`. Стартовий рівень Palace нового Castle не створює додаткового Knight: першим Knight є founder.

### Triggers

- `empty_coins` `[passive state trigger]` — boundary зміни effective rates, не окрема ігрова дія.

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
- `is_capital` — `true` лише для початкового Castle Player; для новозаснованих Castle `false`. Capital Region має спеціальні правила входу/атаки.
- `ready_knights_awaiting_name` — кількість готових безіменних Knight; додається Palace upgrade та KnightReplacement, вилучається після надання імені.

### Динамічні характеристики

- `resources` — один клас із `wood`, `stone`, `iron`.
- `food`

### Обчислювальні зв'язки

- `*regions[]` — Region, приєднані до цього Castle, включно з Castle Region.
- `*knights[]` — Knight, для яких цей Castle є home Castle.
- `*stationed_knights[]` — живі Knight з `Knight.stationed_castle == this Castle` незалежно від home Castle; reverse relationship для фактичного розміщення Unit у Barracks.
- `*building_upgrades[]` — BuildingUpgrade цього Castle, включно з history instances; active визначається їх `status`.
- `*recruitment` — постійна Recruitment-модель цього Castle з можливо порожньою чергою.
- `*knight_replacement` — постійна KnightReplacement-модель цього Castle, що послідовно виконує заміни.

### Інші обчислювальні характеристики

- `resource_balance` — чистий баланс надходжень `wood`, `stone`, `iron`; агрегує `Region.resource_surplus_to_castle` з `regions[]` окремо для кожного ресурсу, без обмеження місткості. Одноразові витрати застосовуються методами, а не як від'ємний production rate.
- `resources_rate` — для кожного з `wood`, `stone`, `iron` застосовує `resource_balance` окремо: при досягненні `warehouse_capacity` позитивний баланс обрізається до `0`; при нульовому запасі від’ємна швидкість не допускається.
- `food_balance` — чистий баланс `sum(Region.food_surplus_to_castle з regions[]) - non_military_food_consumption - castle_stationed_food_consumption - castle_region_army_food_consumption`, без застосування межі `granary_capacity` чи нульового запасу. Баланс використовується для розрахунку дефіциту й компенсації, а фактичне накопичення визначає `food_rate`.
- `food_rate` — застосовує `food_balance` з обмеженнями `granary_capacity` та мінімального запасу `food = 0`; надлишок за повного сховища й дефіцит за нульового запасу дають швидкість `0`.

- `non_military_food_consumption` — Castle-level Food consumption від Population/Building effects.
- `castle_stationed_food_consumption` — Food consumption `soldier_reserve` та всіх живих `stationed_knights[]` (Knight разом зі своїм Unit); включає війська всередині Castle навіть під час Occupation.
- `castle_region_army_food_consumption` — `region.castle_owner_outside_camp_food_consumption` для неокупованої Castle Region, інакше `0`; Knight/Unit всередині Castle уже враховані в `castle_stationed_food_consumption`. Після Occupation зовнішні Camp Army формального owner та occupier використовують локальне Food, а не Food Castle.
- `food_coin_compensation` — якщо `food == 0 && food_balance < 0`, дорівнює `(-food_balance) * coins_per_food`, інакше `0`.
- `coin_balance` — Coin income/upkeep самого Castle, включно з регулярним Coin upkeep `soldier_reserve`. Coin-producing Buildings Castle (включно з Bank effect) продовжують працювати при Occupation. City income сюди не входить: він враховується у `Region.coin_balance_for_owner`. Coin upkeep Unit у складі Army враховується в `Player.coin_balance` через `Army.coin_upkeep` і вдруге тут не нараховується.
- `warehouse_capacity`, `granary_capacity` — незалежні межі: одна Warehouse Capacity застосовується **окремо** до Wood, Stone та Iron; Granary Capacity — до Food.
- `storage_full` — структуровані прапорці `wood / stone / iron / food` за фактичним досягненням відповідних capacity; заповнення одного ресурсу не блокує інші.
- `empty_food` — `food == 0 && food_balance <= 0` (баланс зберігає дефіцит навіть коли `food_rate = 0`).
- `barracks_capacity`.
- `barracks_used` — `soldier_reserve` + сумарна кількість Soldier усіх `stationed_knights[]` незалежно від їх home Castle; 1 Soldier будь-якого Type = 1 Capacity, Knight сам Capacity не займає.
- `barracks_free_capacity = barracks_capacity - barracks_used`.
- `governor_capacity`, `external_region_count`.



- `is_blocked` — `region.is_occupied`; визначає блокування потоків із усіх прив'язаних Region, але не припиняє власні Coin-producing Buildings Castle.

### User methods

- `user_name_next_ready_knight(name)` — user action власника Castle: перевіряє, що `ready_knights_awaiting_name > 0`, та коректність імені, викликає `name_next_ready_knight(name)`.

### Domain methods

- `can_pay_local_cost(cost)`, `pay_local_cost(cost)` — перевіряють/списують Wood, Stone, Iron і Food з фактичних запасів Castle; від'ємний effective balance не забороняє upfront payment, якщо запасу достатньо.
- `can_house_unit(knight)` — перевіряє, що Castle **не заблокований**, Knight живий, належить власнику цього Castle, наразі розміщений **поза Castle** (`location_state = Camp`), його Army має Camp-presence у Region цього Castle, а Barracks має Capacity для **всіх Soldier** Unit; home Castle може бути іншим. При Occupation зовнішня Army не може перейти до внутрішнього складу заблокованого Castle. Вхід атомарний, partial entry немає; Knight з 0 Soldier не потребує Capacity.
- `move_soldiers_between_reserve_and_knight(knight, composition_delta)` — перевіряє `knight.castle == this`, `knight.stationed_castle == this`, відсутність combat command-lock, достатність `soldier_reserve` для додавання та допустимий результат за типами. Атомарно змінює `soldier_reserve` і викликає `Knight.set_soldiers(new_composition)`; сумарна Barracks occupancy не змінюється від перенесення.
- `add_recruited_soldier(type)`.
- `can_start_building_upgrade(building_type)` — перевіряє відсутність active upgrade цієї Building, повну upfront cost (локальну через `can_pay_local_cost()`, глобальну через `player.can_pay_global_cost()`) і, тільки для `0 -> 1`, усі config-driven minimum-level prerequisites. Самі по собі `empty_food`/`player.empty_coins` не забороняють BuildingUpgrade, якщо фактичної upfront cost достатньо.
- `apply_building_upgrade(building_type, target_level)` — застосовує level; кожне підвищення рівня Palace збільшує `ready_knights_awaiting_name` на 1 без затримки заміни.
- `enqueue_knight_replacement()` — після загибелі Knight або зміни його home Castle викликає `knight_replacement.enqueue()`; кожний випадок додає рівно одну заміну. Завершення таймерів виконує KnightReplacement.
- `name_next_ready_knight(name)` — за `ready_knights_awaiting_name > 0` зменшує лічильник на 1 і створює живого Knight з указаним Player name у home Castle, `location_state = Castle`, `stationed_castle = this Castle`, без Soldier; **одночасно створює окрему Army** цього Player з цим Knight як Commander у Castle Region. У неокупованому Castle Army отримує `state = Camp`, `camp = Region.get_or_create_camp(player)`; в окупованому Castle — `state = BlockedInCastle`, `camp = null`. Готових безіменних Knight стає на одного менше; Palace slots не відстежуються.
- `can_annex_region(region, camp)` — перевіряє `is_blocked == false`, що Region не є Castle Region і відповідає правилам Annexation Neutral/Occupied, `camp.can_annex_to(this)` (включно з накопиченим control progress, відсутністю blocking presence/Founding), Governor Capacity, valid territorial connection саме до **цього** Castle та наявність у `camp` хоча б одного живого Knight, чия Army має Camp-presence (`Camp` або `Regrouping`) і чий home Castle дорівнює цьому Castle. Method може напряму обходити `CampInRegion -> armies[] -> knights[]`.
- `recalculate_region_connections()` — використовує `regions[]`, `Region.neighbors[]`, `Region.player`, `Region.is_occupied` і `Region.is_connection_valid`; викликає `Region.set_connection_valid()` та, лише після **остаточної** втрати ownership bridge, `Region.become_neutral()` для downstream Region. Temporary disconnect через Occupation не змінює formal ownership. Каскадний перерахунок виконується до стабілізації графа в одній транзакції без проміжних user commands.

### Triggers

- `empty_food` `[passive state trigger]`.
- `storage_capacity` `[passive state trigger]` — boundary заповнення ресурсних сховищ.

### Trigger methods

- `check_trigger_empty_food()`, `on_trigger_empty_food()`.
- `check_trigger_storage_capacity()`, `on_trigger_storage_capacity()`.

---

## 3. `Region`

### Прямі характеристики

- `player` — формальний owner; `null` для Neutral Region.
- `castle` — Castle, до якого Region приєднана; `null`, якщо не належить Castle.
- `occupier_player_id` — ID поточного occupier; `null`, якщо Region не Occupied. Зберігається атомарно з Occupation transition.
- `resource_sites`.
- `allow_transit` — fallback rule для неагресивного Transit тільки через Owned **non-Occupied** Region, якщо defender не прийняв ручного рішення до Battle Start. Для Occupied Region ця rule не застосовується: Transit через Occupied Region не створює CombatSituation.
- `is_connection_valid`.
- `combat_situation_queue_tail` — прямий зв'язок на останню актуальну PvP CombatSituation єдиної FIFO-черги Region або `null`.

### Динамічні характеристики

- `neutral_defense` — поточна сила Neutral Defense.

### Обчислювальні зв'язки

- `*neighbors[]` — шість сусідніх Region.
- `*armies[]` — Army, що фізично знаходяться в Region за `Army.current_region`.
- `*camps[]` — active CampInRegion цієї Region.
- `*city` — City цієї Region або `null`.
- `*resource_site_upgrades[]`, `*castle_foundings[]`.
- `*combat_situations[]` — актуальні CombatSituation Region (`Registered`/`Active`); історичні й завершені миттєві БС не входять.

### Інші обчислювальні характеристики

- `neutral_defense_rate` — `neutral_defense_recovery_rate`, обмежена до `0` при досягненні `neutral_defense_full_strength` і при відсутності умов для відновлення; не допускає виходу за межі `0..neutral_defense_full_strength`.

- `camp_player_ids[]` — `CampInRegion.player_id` для active CampInRegion цієї Region.
- `eligible_camp_player_ids[]` — `CampInRegion.player_id` для CampInRegion, що мають `has_eligible_presence == true`; `Regrouping` рахується так само, як `Camp`.
- `blocking_camp_presence_player_ids[]` — `Army.player_id` для Player, які мають у Region хоча б одну Army у `Camp`/`Regrouping` або Army з `is_entered_for_camp == true`. Саме ця характеристика використовується для перевірки foreign Camp/Camp-bound presence в Annexation і Castle Founding; чистий Transit сюди не входить.
- `player_id` — ID `player`; проміжна scalar characteristic для computed characteristics інших Model без другого relationship hop.
- `occupier_player` — nullable relationship до Player за `occupier_player_id`; `null` поза Occupation. Потрібний `Movement.determine_target_opponent()`, `CombatSituation.start()` і `Region.restore_owner_control()`, де потрібен Player object, а не лише ID.
- `valid_connected_neighbor_player_ids[]` — `player_id` сусідніх Owned Region з валідним connection; використовує `neighbors[].player_id`, а не `neighbors[] -> player`.
- `is_occupied`, `is_castle_region`, `has_active_founding`.
- `distance_to_castle` — пряма стандартна hex-grid distance між цією Region і її Castle Region (`0` для Castle Region); не є path length і використовується DistanceEfficiency/territorial distance rules.
- `has_any_troops` — чи є в Region хоча б одна фізично присутня Army незалежно від її Camp/Transit/Regrouping context, включно з Army у Movement та `BlockedInCastle`; використовується для паузи відновлення Neutral Defense.
- `owner_empty_coins` — `player.empty_coins` для Owned/Occupied Region; для Neutral Region `false`. Проміжна computed characteristic для City та інших залежностей без обходу `City -> Region -> Player`.
- `neutral_defense_full_strength`.
- `neutral_defense_recovery_rate` — ненульовий тільки для Neutral Region, коли `neutral_defense < neutral_defense_full_strength` і в Region немає жодних troops; будь-яка фізична присутність Army ставить recovery rate в `0`, а після виходу останньої Army recovery продовжується від поточного значення.
- `active_combat_situation` — єдина Active player-vs-player CombatSituation у Region або `null`.
- `next_queued_combat` — застарілий опис: окремої характеристики немає. Порядок визначають `CombatSituation.previous_combat_situation` і computed `next_combat_situation`; хвіст — `Region.combat_situation_queue_tail`. Миттєві БС у FIFO не входять.


- `food_production`.
- `camp_food_consumption` — сумарне Food consumption Army у Camp/Regrouping цієї Region; для Castle Region агрегує `Army.outside_castle_food_consumption`, бо розміщені всередині Castle Unit враховуються у `Castle.castle_stationed_food_consumption`.
- `castle_owner_outside_camp_food_consumption` — для Castle Region сумарне `Army.outside_castle_food_consumption` власних (`Army.player == Region.player`) Army у Camp/Regrouping; використовується `Castle.castle_region_army_food_consumption`, щоб Castle не обходив `Region -> Army -> Knight`.
- `food_balance` — для звичайної Region `food_production - camp_food_consumption`; для **неокупованої** Castle Region дорівнює `food_production`, бо споживання власних Camp Army поза Castle віднімається на рівні Castle. Для **Occupied Castle Region** дорівнює `food_production - camp_food_consumption`: усі Army у Camp поза Castle (і occupier, і formal owner) використовують локальне Food за звичайними правилами, а Unit всередині заблокованого Castle продовжують споживати його запаси.
- `resource_production`, `gold_production`, `silver_production`.
- `resource_surplus_to_castle` — передає позитивну Resource production до Castle тільки для Owned non-Occupied Region із `is_connection_valid == true` та `castle.is_blocked == false`, із distance efficiency; інакше `0`.
- `food_surplus_to_castle` — для звичайної Owned non-Occupied Region з `is_connection_valid == true` і `castle.is_blocked == false` передає до Castle тільки позитивний `food_balance` з distance efficiency; для неокупованої Castle Region із незаблокованим Castle передає весь позитивний `food_production` з coefficient `1`. Негативний balance звичайної Region до Castle не передається; Occupied/invalid-connected Region та Region із заблокованим Castle нічого не передають.
- `gold_income_for_owner`, `silver_income_for_owner` — для Owned non-Occupied Region з `is_connection_valid == true` і `castle.is_blocked == false` дорівнюють відповідній production без distance penalty; інакше `0`.
- `coin_balance_for_owner` — recurring Coin balance Region для formal owner. City income (`city.coin_income`, якщо City є) враховується тільки для Owned non-Occupied Region із `is_connection_valid == true` та `castle.is_blocked == false`; Region upkeep продовжує нараховуватися при Occupation, disconnection і блокуванні Castle до втрати ownership. Після переходу Region у Neutral upkeep припиняється. Відсутність City дає нульовий City income.
- `city_wealth_growth_enabled` — `true` тільки для Owned non-Occupied Region, якщо `owner_empty_coins == false`; territorial connection не потрібен. Для Neutral/Occupied Region `false`.

### User methods

- `user_change_allow_transit(value)` — змінює fallback Transit rule власної Owned non-Occupied Region; для Occupied Region `allow_transit` не використовується.
- `user_abandon_region()` — миттєво й без cost переводить власну ordinary Region у Neutral. Castle Region і Region з `active_combat_situation != null` заборонені. Не потребує Army/Knight presence; дозволена для disconnected або Occupied Region. Після neutralization запускає connectivity recalculation; downstream Region, що остаточно втратили connection до Castle, також стають Neutral. City Wealth, ResourceSite levels і active ResourceSiteUpgrade зберігаються; Army залишаються фізично на місці; уже active Annexation іншого Player не скасовується.

### Domain methods

- `find_active_camp(player)`, `get_or_create_camp(player)`.
- `register_combat(attacker, registration_reason)` — реєструє PvP CombatSituation при Movement/Retreat entry, фіксує attacking Army, Region, registration reason, FIFO order і snapshot `CombatSituation.attacker_entry_from_region = Army.movement.entry_from_region`. `defender_player` і `defenders[]` визначаються на Start. Єдина FIFO-черга для Owned, Occupied і Neutral Region; очікування зберігає Movement context і командне блокування attacker.
- `start_next_combat_if_possible()` — якщо Active CombatSituation немає, запускає найстарішу Registered CombatSituation Region. Викликається **синхронно після terminal transition** попередньої situation, уже після застосування наслідків Camp/Transit/Occupation/Retreat; окремого тригера чи очікування між завершенням попередньої ситуації та Start наступної немає. Якщо наступна situation одразу завершується без battle, обробляє наступну у FIFO в тому самому послідовному циклі. Для situation, що лишається Active, від Start починається повний `Dt` до Battle Start.
- `resolve_arrival(army, movement)` — визначає Camp/Transit context. Camp-bound entry у чужу Owned/Occupied чи власну Occupied Region реєструє БС за загальними правилами; Camp-bound entry у Neutral Region створює БС, якщо там уже є чужа Army у `Camp`/Regrouping або `EnteringCamp`. Camp різних Player одночасно заборонений. Transit у Neutral Region вільний і БС не створює; для Owned/Occupied Region зберігаються їхні Transit rules. Без БС Army завершує `entry -> Camp`/`retreat-local` за `Dt`; армії після Start не долучаються до вже активної БС. Neutral Defense/City автоматично не атакуються.
- `set_occupied_by(player)` — перевіряє `player` через фактичну Camp-presence і атомарно встановлює `occupier_player_id = player.id` (після чого computed `occupier_player` посилається на цього Player); встановлює Occupation тільки при фактичному зайнятті Camp, зокрема після Camp arrival без battle або перемоги Camp-bound attacker. При **першому** переході Castle Region з non-Occupied в Occupied опрацьовує внутрішні Army формального owner: викликає `Army.split_for_castle_occupation(castle)` для змішаних Army, `Army.block_in_castle(castle)` для внутрішніх Army, зовнішні частини відступають за combat rules. Якщо occupier змінюється без завершення Occupation, заблоковані Army залишаються заблокованими. Якщо в Region вже рухаються Army formal owner, які запізнилися до попередньої defense, для Camp-bound Army реєструється окрема CombatSituation проти актуального occupier; Transit продовжується за правилами Occupied Region. Після зміни Occupation запускає `Castle.recalculate_region_connections()`.
- `restore_owner_control(expected_occupier_player)` — Occupation припиняється після виходу останньої Army поточного occupier із Camp або звільнення Region формальним owner; `expected_occupier_player` захищає від застарілого виклику при заміні occupier. Атомарно очищує `occupier_player_id`, а для Castle Region **до завершення цього переходу й до запуску наступної CombatSituation** викликає `Army.unblock_from_castle(castle)` для всіх внутрішніх `BlockedInCastle` Army: вони переходять у `Camp` без Regrouping, зберігаючи склад і Commander. Далі перераховує connectivity Castle. Якщо інший occupier одразу замінив попереднього без проміжного Owned state, метод не розблоковує Army.
- `annex_to(player, castle)` — після успішної Annexation встановлює formal owner, новий Castle і очищує попередню Occupation. Перераховує connectivity **і нового Castle, і попереднього Castle/formal owner**, якщо Region була Occupied та перейшла від іншого власника; Region колишнього owner, які остаточно втратили зв'язок, стають Neutral за загальним правилом.
- `become_neutral()` — очищує formal owner/Castle/Occupation, встановлює `is_connection_valid = false`, зберігає ResourceSite levels, City `wealth` та `active_wealth_ratio`, а також active ResourceSiteUpgrade (вони продовжуються). Для попереднього Castle запускає перерахунок connectivity та остаточну втрату downstream Region без власного зв'язку. Army фізично залишаються в Region; активні Camp episodes не видаляються лише через зміну ownership.
- `become_castle_region(new_castle, player)` — після завершення Founding робить Region Owned Castle Region, встановлює `castle = new_castle`, `player`, `occupier_player_id = null`, `is_connection_valid = true` для Castle Region; зберігає ResourceSite levels і active ResourceSiteUpgrade. Уже фізично розпочатий чужий Transit може завершитися без нової CombatSituation за правилами Founding.
- `set_connection_valid(value)` — змінює збережену характеристику `is_connection_valid` за результатом перерахунку територіального графа; тимчасовий розрив через Occupation не скидає ownership.
- `can_start_resource_site_upgrade(resource_type, quantity)` — використовує `resource_sites`, `resource_site_upgrades[]`, `player`, `castle`, `is_occupied`, `is_connection_valid` і поточні level buckets. Перевіряє ownership, доступність незарезервованих Site потрібного рівня та layered rule; upfront cost перевіряється через `Castle.can_pay_local_cost()` і `Player.can_pay_global_cost()`; самі reservations не вважаються completed levels.
- `apply_resource_site_upgrade(resource_type, from_level, target_level, quantity)` — після завершення `ResourceSiteUpgrade.complete()` атомарно переносить рівно зарезервовану кількість Site між level buckets у `resource_sites`, незалежно від поточного ownership; повторне застосування terminal process заборонено.
- `on_army_presence_changed()` — викликається після `Army.enter_region()`, `Army.enter_camp()`, `Army.leave_camp()`, завершення Retreat, split/merge та знищення Army. Інфраструктура актуалізує system-computed `armies[]`/`camps[]` і залежні `camp_player_ids[]`, `blocking_camp_presence_player_ids[]`, `has_any_troops`, `CampInRegion.can_progress`, `CastleFounding.can_progress`, `neutral_defense_recovery_rate`. Якщо з Camp пішла остання Army поточного occupier, викликає `restore_owner_control(expected_occupier_player)`; **не змінює напряму computed** `is_occupied` чи `active_combat_situation` і не реєструє CombatSituation тільки через факт перерахунку присутності.
- `destroy_neutral_defense()`.

### Triggers

- `neutral_defense_recovery_complete` `[passive event trigger]` — boundary завершення поступового відновлення, без додаткової ігрової дії.

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

### Обчислювальні зв'язки

- Немає.

### Інші обчислювальні характеристики

- `wealth_balance` — необмежений баланс зростання Wealth, позитивний тільки коли `active_wealth_ratio == 1` і `region.city_wealth_growth_enabled == true`.
- `wealth_rate` — `wealth_balance` з допустимими межами `wealth` (зокрема без виходу в від’ємне значення).
- `active_wealth_ratio_balance` — баланс відновлення частки Wealth без обрізання на межі `1`; відновлення не залежить від ownership, Occupation або стану Coins Player.
- `active_wealth_ratio_rate` — `active_wealth_ratio_balance` з обмеженням діапазону `0..1`; при досягненні `1` додатна швидкість дорівнює `0`.

- `city_defense = CityDefense(wealth)` — залежить тільки від повного `wealth`.
- `raid_reward` — разовий Coin reward Raid.


- `effective_wealth = wealth * active_wealth_ratio`.
- `coin_income` — локальний recurring Coin income City на основі `effective_wealth`; `0` для Neutral/Occupied. Ця характеристика сама по собі не гарантує надходження Player: `Region.coin_balance_for_owner` додатково перевіряє connection та блокування Castle, щоб City disconnected Owned Region не давало Coins.

### Domain methods

- `complete_raid(player)` — використовує `effective_wealth`/`raid_reward` до зміни ratio, викликає `Player.add_coins(reward)` для переданого attacker Player та атомарно скидає `active_wealth_ratio = 0`; `wealth` і `city_defense` не змінюються. Повторний виклик для того самого успішного Raid у межах однієї транзакції не допускається.

### Triggers

- `active_wealth_recovered` `[passive event trigger]` — boundary досягнення `active_wealth_ratio == 1`, після якого змінюється `wealth_balance`.

### Trigger methods

- `check_trigger_active_wealth_recovered()`.
- `on_trigger_active_wealth_recovered()`.

---
