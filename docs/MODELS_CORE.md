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

### Обчислювальні характеристики

- `*regions[]` — Region, формальним owner яких є цей Player.
- `*castles[]` — Castle цього Player.
- `*armies[]` — Army цього Player за прямим `Army.player`.
- `*camps[]` — active CampInRegion цього Player за прямим `CampInRegion.player`.
- `coin_balance` — effective rate зміни Coins. Складає `Region.coin_balance_for_owner` із `regions[]`, `Castle.coin_balance` і `Castle.food_coin_compensation` із `castles[]`, звичайний `Army.coin_upkeep` і `Army.food_coin_compensation` для Army у Movement із `armies[]`, а також `CampInRegion.food_coin_compensation` із `camps[]` для звичайних Region **та Occupied Castle Region**. У неокупованій Castle Region локальна Camp-компенсація дорівнює `0`, бо Food усіх власних Camp Unit враховується у Castle. Кожен дефіцит Food компенсується рівно один раз.
- `gold_balance` — effective rate зміни Gold; агрегує `Region.gold_income_for_owner` з `regions[]`, тому Occupied/disconnected Region не дають Gold формальному owner.
- `silver_balance` — effective rate зміни Silver; агрегує `Region.silver_income_for_owner` з `regions[]`, тому Occupied/disconnected Region не дають Silver формальному owner.
- `empty_coins` — `coins == 0 && coin_balance <= 0`.

### User methods

- немає зафіксованих на цей момент.

### Domain methods

- `can_pay_global_cost(cost)` — перевіряє одноразову вартість у Coins/Gold/Silver.
- `pay_global_cost(cost)` — списує одноразову глобальну частину вартості після validation.
- `add_coins(amount)` — зараховує разовий Coin reward, зокрема City Raid reward.
- `create_castle(region, name, founder_knight)` — після завершення Founding створює Castle з `Warehouse = 1`, `Granary = 1`, `Palace = 1`; викликає `Region.become_castle_region()` і `Knight.change_home_castle()` атомарно. Використовує `Player.castles[]`, `Region.player/castle`, `Knight.castle` та Palace slots обох Castle. Founder займає перший slot нового Castle; старий home Castle запускає звичайний replacement для звільненого slot. Додатковий Knight у новому Castle не створюється.

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
- `ready_knights_awaiting_name` — збережена FIFO-черга готових безіменних Knight; додається Palace upgrade та KnightReplacement, вилучається після надання імені.

### Динамічні характеристики

- `resources` — один клас із `wood`, `stone`, `iron`.
- `food`

### Обчислювальні характеристики

- `*regions[]` — Region, приєднані до цього Castle, включно з Castle Region.
- `*knights[]` — Knight, для яких цей Castle є home Castle.
- `*stationed_knights[]` — живі Knight з `Knight.stationed_castle == this Castle` незалежно від home Castle; reverse relationship для фактичного розміщення Unit у Barracks.
- `*building_upgrades[]` — BuildingUpgrade цього Castle, включно з history instances; active визначається їх `status`.
- `*recruitment` — active Recruitment Queue Castle або `null`.
- `*knight_replacements[]` — KnightReplacement цього Castle.
- `resource_balance` — effective rate для `wood`, `stone`, `iron`; агрегує `Region.resource_surplus_to_castle` з `regions[]` окремо для кожного ресурсу. Якщо відповідний ресурс досяг `warehouse_capacity`, його позитивний effective rate стає `0` (надлишок production втрачається), без впливу на інші два ресурси. Одноразові витрати застосовуються методами, а не як від'ємний production rate.
- `food_income` — сума позитивних `Region.food_surplus_to_castle` з `regions[]`. Негативний Food balance звичайних Region до Castle не передається.
- `non_military_food_consumption` — Castle-level Food consumption від Population/Building effects.
- `castle_stationed_food_consumption` — Food consumption `soldier_reserve` та всіх живих `stationed_knights[]` (Knight разом зі своїм Unit); включає війська всередині Castle навіть під час Occupation.
- `castle_region_army_food_consumption` — `region.castle_owner_outside_camp_food_consumption` для неокупованої Castle Region, інакше `0`; Knight/Unit всередині Castle уже враховані в `castle_stationed_food_consumption`. Після Occupation зовнішні Camp Army формального owner та occupier використовують локальне Food, а не Food Castle.
- `food_balance` — `food_income - non_military_food_consumption - castle_stationed_food_consumption - castle_region_army_food_consumption`, з урахуванням межі `granary_capacity`: якщо `food >= granary_capacity` і raw balance позитивний, effective `food_balance = 0` (надлишок втрачається); від'ємний balance не обрізається, щоб запас Food міг зменшуватися. Цей effective rate використовується для `empty_food` і компенсації дефіциту.
- `food_coin_compensation` — якщо `food == 0 && food_balance < 0`, дорівнює `(-food_balance) * coins_per_food`, інакше `0`.
- `coin_balance` — Coin income/upkeep самого Castle, включно з регулярним Coin upkeep `soldier_reserve`. Coin-producing Buildings Castle (включно з Bank effect) продовжують працювати при Occupation. City income сюди не входить: він враховується у `Region.coin_balance_for_owner`. Coin upkeep Unit у складі Army враховується в `Player.coin_balance` через `Army.coin_upkeep` і вдруге тут не нараховується.
- `player_empty_coins` — `player.empty_coins`; проміжна характеристика для залежних computed characteristics інших Model без ланцюга relationships.
- `warehouse_capacity`, `granary_capacity` — незалежні межі: одна Warehouse Capacity застосовується **окремо** до Wood, Stone та Iron; Granary Capacity — до Food.
- `storage_full` — структуровані прапорці `wood / stone / iron / food` за фактичним досягненням відповідних capacity; заповнення одного ресурсу не блокує інші.
- `is_blocked` — `region.is_occupied`; визначає блокування потоків із усіх прив'язаних Region, але не припиняє власні Coin-producing Buildings Castle.
- `empty_food` — `food == 0 && food_balance <= 0`.
- `barracks_capacity`.
- `barracks_used` — `soldier_reserve` + сумарна кількість Soldier усіх `stationed_knights[]` незалежно від їх home Castle; 1 Soldier будь-якого Type = 1 Capacity, Knight сам Capacity не займає.
- `barracks_free_capacity = barracks_capacity - barracks_used`.
- `governor_capacity`, `external_region_count`.
- `palace_capacity`, `active_knight_replacement_id`.
- `palace_reserved_slots` — кількість живих `knights[]` + елементів `ready_knights_awaiting_name` + `knight_replacements[]` зі status queued/active (timer ще не завершено). Кожний Palace slot обліковується рівно один раз; `palace_reserved_slots <= palace_capacity`. Completed replacement уже представлений ready-entry, а не одночасно двома reservations.


### User methods

- `user_name_next_ready_knight(name)` — user action власника Castle: перевіряє доступність head спільної FIFO-черги та ім'я, викликає `name_next_ready_knight(name)`.

### Domain methods

- `can_pay_local_cost(cost)`, `pay_local_cost(cost)` — перевіряють/списують Wood, Stone, Iron і Food з фактичних запасів Castle; від'ємний effective balance не забороняє upfront payment, якщо запасу достатньо.
- `can_house_unit(knight)` — перевіряє, що Castle **не заблокований**, Knight живий, належить власнику цього Castle, наразі розміщений **поза Castle** (`location_state = Camp`), його Army має Camp-presence у Region цього Castle, а Barracks має Capacity для **всіх Soldier** Unit; home Castle може бути іншим. При Occupation зовнішня Army не може перейти до внутрішнього складу заблокованого Castle. Вхід атомарний, partial entry немає; Knight з 0 Soldier не потребує Capacity.
- `move_soldiers_between_reserve_and_knight(knight, composition_delta)` — перевіряє `knight.castle == this`, `knight.stationed_castle == this`, відсутність combat command-lock, достатність `soldier_reserve` для додавання та допустимий результат за типами. Атомарно змінює `soldier_reserve` і викликає `Knight.set_soldiers(new_composition)`; сумарна Barracks occupancy не змінюється від перенесення.
- `add_recruited_soldier(type)`.
- `can_start_building_upgrade(building_type)` — перевіряє відсутність active upgrade цієї Building, повну upfront cost (локальну через `can_pay_local_cost()`, глобальну через `player.can_pay_global_cost()`) і, тільки для `0 -> 1`, усі config-driven minimum-level prerequisites. Самі по собі `empty_food`/`player_empty_coins` не забороняють BuildingUpgrade, якщо фактичної upfront cost достатньо.
- `apply_building_upgrade(building_type, target_level)` — застосовує level; Palace level increase додає один ready Knight у FIFO `ready_knights_awaiting_name` без replacement delay.
- `enqueue_knight_replacement()`, `complete_knight_replacement(replacement)` — використовують `knight_replacements[]`, `active_knight_replacement_id` та `ready_knights_awaiting_name`. Звільнення Palace slot створює queued replacement; якщо active timer немає, перший queued стає active. Completion обробляє тільки active head: додає одного ready Knight у спільну FIFO name queue, переводить replacement у terminal `timer_complete` і запускає наступний queued timer. Повторний виклик completion для terminal instance нічого не додає.
- `name_next_ready_knight(name)` — бере тільки head спільної FIFO-черги, створює живого Knight з указаним Player name у home Castle, `location_state = Castle`, `stationed_castle = this Castle`, без Soldier; **одночасно створює окрему Army** цього Player з цим Knight як Commander у Castle Region. У неокупованому Castle Army отримує `state = Camp`, `camp = Region.get_or_create_camp(player)`; в окупованому Castle — `state = BlockedInCastle`, `camp = null`. Нового Palace slot не займає: ready-entry замінюється живим Knight без зміни `palace_reserved_slots`.
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
- `occupier_player` — nullable relationship до Player за `occupier_player_id`; `null` поза Occupation. Потрібний `Movement.determine_target_opponent()`, `CombatSituation.start()` і `Region.restore_owner_control()`, де потрібен Player object, а не лише ID.
- `valid_connected_neighbor_player_ids[]` — `player_id` сусідніх Owned Region з валідним connection; використовує `neighbors[].player_id`, а не `neighbors[] -> player`.
- `is_occupied`, `is_castle_region`, `has_active_founding`.
- `distance_to_castle` — пряма стандартна hex-grid distance між цією Region і її Castle Region (`0` для Castle Region); не є path length і використовується DistanceEfficiency/territorial distance rules.
- `has_any_troops` — чи є в Region хоча б одна фізично присутня Army незалежно від її Camp/Transit/Regrouping context, включно з Army у Movement та `BlockedInCastle`; використовується для паузи відновлення Neutral Defense.
- `food_production`.
- `camp_food_consumption` — сумарне Food consumption Army у Camp/Regrouping цієї Region; для Castle Region агрегує `Army.outside_castle_food_consumption`, бо розміщені всередині Castle Unit враховуються у `Castle.castle_stationed_food_consumption`.
- `castle_owner_outside_camp_food_consumption` — для Castle Region сумарне `Army.outside_castle_food_consumption` власних (`Army.player == Region.player`) Army у Camp/Regrouping; використовується `Castle.castle_region_army_food_consumption`, щоб Castle не обходив `Region -> Army -> Knight`.
- `food_balance` — для звичайної Region `food_production - camp_food_consumption`; для **неокупованої** Castle Region дорівнює `food_production`, бо споживання власних Camp Army поза Castle віднімається на рівні Castle. Для **Occupied Castle Region** дорівнює `food_production - camp_food_consumption`: усі Army у Camp поза Castle (і occupier, і formal owner) використовують локальне Food за звичайними правилами, а Unit всередині заблокованого Castle продовжують споживати його запаси.
- `resource_production`, `gold_production`, `silver_production`.
- `resource_surplus_to_castle` — передає позитивну Resource production до Castle тільки для Owned non-Occupied Region із `is_connection_valid == true` та `castle.is_blocked == false`, із distance efficiency; інакше `0`.
- `food_surplus_to_castle` — для звичайної Owned non-Occupied Region з `is_connection_valid == true` і `castle.is_blocked == false` передає до Castle тільки позитивний `food_balance` з distance efficiency; для неокупованої Castle Region із незаблокованим Castle передає весь позитивний `food_production` з coefficient `1`. Негативний balance звичайної Region до Castle не передається; Occupied/invalid-connected Region та Region із заблокованим Castle нічого не передають.
- `gold_income_for_owner`, `silver_income_for_owner` — для Owned non-Occupied Region з `is_connection_valid == true` і `castle.is_blocked == false` дорівнюють відповідній production без distance penalty; інакше `0`.
- `coin_balance_for_owner` — recurring Coin balance Region для formal owner. City income (`city.coin_income`, якщо City є) враховується тільки для Owned non-Occupied Region із `is_connection_valid == true` та `castle.is_blocked == false`; Region upkeep продовжує нараховуватися при Occupation, disconnection і блокуванні Castle до втрати ownership. Після переходу Region у Neutral upkeep припиняється. Відсутність City дає нульовий City income.
- `owner_empty_coins` — `player.empty_coins` для Owned/Occupied Region; для Neutral Region `false`. Проміжна computed characteristic для City та інших залежностей без обходу `City -> Region -> Player`.
- `city_wealth_growth_enabled` — `true` тільки для Owned non-Occupied Region, якщо `owner_empty_coins == false`; territorial connection не потрібен. Для Neutral/Occupied Region `false`.
- `neutral_defense_full_strength`.
- `neutral_defense_recovery_rate` — ненульовий тільки для Neutral Region, коли `neutral_defense < neutral_defense_full_strength` і в Region немає жодних troops; будь-яка фізична присутність Army ставить recovery rate в `0`, а після виходу останньої Army recovery продовжується від поточного значення.
- `active_combat_situation` — єдина Active player-vs-player CombatSituation у Region або `null`.
- `next_queued_combat` — найраніше зареєстрована `Registered` CombatSituation у єдиній FIFO-черзі Region. Усі player-vs-player CombatSituation цієї Region, включно з Neutral Region і будь-якими парами Player, використовують одну чергу. Attack на Neutral Defense та City Raid у цю чергу не входять.

### User methods

- `user_change_allow_transit(value)` — змінює fallback Transit rule власної Owned non-Occupied Region; для Occupied Region `allow_transit` не використовується.
- `user_abandon_region()` — миттєво й без cost переводить власну ordinary Region у Neutral. Castle Region і Region з `active_combat_situation != null` заборонені. Не потребує Army/Knight presence; дозволена для disconnected або Occupied Region. Після neutralization запускає connectivity recalculation; downstream Region, що остаточно втратили connection до Castle, також стають Neutral. City Wealth, ResourceSite levels і active ResourceSiteUpgrade зберігаються; Army залишаються фізично на місці; уже active Annexation іншого Player не скасовується.

### Domain methods

- `find_active_camp(player)`, `get_or_create_camp(player)`.
- `register_combat(attacker, registration_reason, defender_player = null)` — реєструє player-vs-player CombatSituation. На Registration фіксуються attacking Army, Region, причина, порядок реєстрації та `CombatSituation.attacker_entry_from_region = Army.movement.entry_from_region` для Movement/Retreat entry (для explicit Neutral Camp attack — `null`). Для explicit Neutral Camp attack переданий `defender_player` фіксується одразу; для територіальних registration reasons він має бути `null` і визначається тільки на Start. `potential_defenders[]` і `transit_defender_candidates[]` на Registration не фіксуються. Одна CombatSituation завжди має рівно одну attacking Army. Якщо в Region уже є Active або earlier Registered CombatSituation, нова стає в єдину FIFO-чергу Region. Attacker отримує combat lock очікування, але його фізичний стан не змінюється: Camp Army лишається Camp-presence, Movement Army лишається у своєму Movement context з paused progress.
- `start_next_combat_if_possible()` — якщо Active CombatSituation немає, запускає найстарішу Registered CombatSituation Region. Викликається **синхронно після terminal transition** попередньої situation, уже після застосування наслідків Camp/Transit/Occupation/Retreat; окремого тригера чи очікування між завершенням попередньої ситуації та Start наступної немає. Якщо наступна situation одразу завершується без battle, обробляє наступну у FIFO в тому самому послідовному циклі. Для situation, що лишається Active, від Start починається повний `Dt` до Battle Start.
- `resolve_arrival(army, movement)` — визначає Camp/Transit context і реєструє CombatSituation лише за умовами §16: Camp-bound entry у чужу Owned/Occupied Region, **включно з player-vs-player і pre-battle Retreat**, створює situation за наявності чужої `Camp`/Regrouping або раніше введеної для Camp `EnteringCamp` Army; сама лише чужа Transit-presence не є підставою. Вхід formal owner до своєї ще non-Occupied Region також створює situation за наявності чужої `EnteringCamp` Army. Без registration Army продовжує `entry -> Camp`/`retreat-local` за `Dt`, після фактичного Camp arrival застосовуються Occupation/control consequences та для Retreat починається Regrouping. Transit у чужу Owned non-Occupied Region використовує окремі Transit registration rules; Transit через Occupied Region не створює situation. Army, що входить після CombatSituation Start, не може стати defender цієї CombatSituation. У Neutral Region сам вхід у Camp не запускає combat із Neutral Defense або City Defense.
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

### Обчислювальні характеристики

- `wealth_balance` — позитивний тільки коли `active_wealth_ratio == 1` і `region.city_wealth_growth_enabled == true`.
- `active_wealth_ratio_balance` — recovery rate при `active_wealth_ratio < 1`; recovery не залежить від ownership, Occupation або стану Coins Player.
- `effective_wealth = wealth * active_wealth_ratio`.
- `city_defense = CityDefense(wealth)` — залежить тільки від повного `wealth`.
- `coin_income` — локальний recurring Coin income City на основі `effective_wealth`; `0` для Neutral/Occupied. Ця характеристика сама по собі не гарантує надходження Player: `Region.coin_balance_for_owner` додатково перевіряє connection та блокування Castle, щоб City disconnected Owned Region не давало Coins.
- `raid_reward` — разовий Coin reward Raid.

### Domain methods

- `complete_raid(player)` — використовує `effective_wealth`/`raid_reward` до зміни ratio, викликає `Player.add_coins(reward)` для переданого attacker Player та атомарно скидає `active_wealth_ratio = 0`; `wealth` і `city_defense` не змінюються. Повторний виклик для того самого успішного Raid у межах однієї транзакції не допускається.

### Triggers

- `active_wealth_recovered` `[passive event trigger]` — boundary досягнення `active_wealth_ratio == 1`, після якого змінюється `wealth_balance`.

### Trigger methods

- `check_trigger_active_wealth_recovered()`.
- `on_trigger_active_wealth_recovered()`.

---
