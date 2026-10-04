# Backend models

Робочий опис моделей. На цьому етапі для кожної моделі фіксуються:

- прямі характеристики;
- динамічні характеристики;
- обчислювальні характеристики;
- user methods, що реалізують основну логіку відповідних user actions;
- triggers.

Root model для user GameEvent поки не визначена до проєктування взаємодії з frontend/API. Тому `user_*` тут означає модель, яка реалізує основну логіку дії, а не обов'язково майбутню root model. Після визначення root model префікси методів за потреби будуть змінені.

Обчислювальна характеристика, позначена `*` перед назвою, є **system-computed**: її значення не зберігається прямо і не обчислюється game logic самої Model, а надається infrastructure. Типові приклади — reverse relations, вибірки пов'язаних Model та топологічні зв'язки карти. Характеристики, що містять список посилань на Model, мають `[]` у кінці назви.

Рівні всіх Building одного Castle зберігаються як одна характеристика `levels`.

Wood / Stone / Iron зберігаються як одна характеристика `resources` — один value object / helper class із трьома значеннями.

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
- `coin_balance` — effective rate зміни Coins; залежить від економічних характеристик `regions[]` і `castles[]`.
- `gold_balance` — effective rate зміни Gold; залежить від економічних характеристик `regions[]` і `castles[]`.
- `silver_balance` — effective rate зміни Silver; залежить від економічних характеристик `regions[]` і `castles[]`.
- `empty_coins` — `coins == 0 && coin_balance <= 0`.

### User methods

- немає зафіксованих на цей момент.

### Triggers

- `empty_coins` `[state trigger]` — спрацьовує при зміні стану `empty_coins`; потрібний як temporal boundary для залежних processes/rates і пов'язаних зовнішніх ефектів.

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
- `resource_balance` — один клас із effective rate для `wood`, `stone`, `iron`; залежить від `levels`, `regions[]`, їх production/flows і меж Warehouse.
- `food_balance` — effective rate зміни Food; залежить від `levels`, `regions[]`, `knights[]`, production/flows, consumption і меж Granary/zero.
- `warehouse_capacity` — Capacity Warehouse відповідно до `levels`.
- `granary_capacity` — Capacity Granary відповідно до `levels`.
- `empty_food` — `food == 0 && food_balance <= 0`.

### User methods

- немає зафіксованих на цей момент.

### Triggers

- `empty_food` `[state trigger]` — спрацьовує при зміні стану `empty_food`; потрібний як temporal boundary для Recruitment та інших залежних rates/processes.
- `storage_capacity` `[state trigger]` — окремо для Food і кожного компонента `resources` відстежує стан досягнення відповідної Capacity; при заповненні storage припиняється його effective growth, а коли місце знову з'являється — growth може відновитися.

---

## 3. `Region`

### Прямі характеристики

- `player` — формальний owner; `null` для Neutral Region.
- `castle` — Castle, до якого Region приєднана; `null`, якщо не належить Castle.
- `resource_sites` — структура ResourceSite за типами ресурсів і рівнями.
- `allow_transit` — правило Transit для Owned Region.
- `is_connection_valid` — чи має Owned Region чинний зв'язок зі своїм Castle.
- `neutral_defense` — актуальний стан Neutral Defense.
- `neutral_defense_recovery_started_at` — Game Time початку очікування повного відновлення Neutral Defense, якщо recovery активний.

### Динамічні характеристики

- немає зафіксованих на цей момент.

### Обчислювальні характеристики

- `*neighbors[]` — шість сусідніх Region, визначені топологією карти.
- `*armies[]` — Army, що поточно рахуються фізично присутніми в Region за правилами presence.
- `camp_player_ids[]` — ID Player, чиї Army зараз перебувають у Camp у цій Region; це список scalar ID, а не зв'язків з Player.
- `valid_connected_neighbor_player_ids[]` — ID Player, для яких серед `neighbors[]` є Owned Region з `is_connection_valid == true`; це список scalar ID, а не зв'язків з Player.
- `has_any_troops` — чи є в `armies[]` будь-які війська, що блокують початок/завершення Neutral Defense recovery.
- `neutral_defense_recovery_at` — Game Time повного відновлення Neutral Defense, якщо recovery активний.

### User methods

- `user_change_allow_transit()` — змінює правило Transit для Owned Region.

### Triggers

- `neutral_defense_recovery_complete` `[event trigger]` — спрацьовує, коли активний recovery досягає `neutral_defense_recovery_at` і умови recovery все ще виконуються; Neutral Defense відновлюється повністю.

---

## 4. `City`

### Прямі характеристики

- `region`

### Динамічні характеристики

- `wealth`

### Обчислювальні характеристики

- `*player` — поточний Player-owner Region міста; `null` для Neutral Region.
- `wealth_balance` — effective rate зміни Wealth з урахуванням характеристик `region`, `player` та правил City.

### User methods

- `user_raid_city()` — виконує raid City: застосовує наслідки до Wealth і пов'язаних результатів raid.

### Triggers

- немає зафіксованих на цей момент.

---

## 5. `Knight`

Knight разом зі своїми Soldier представляє gameplay Unit; окремої backend-моделі Unit немає.

### Прямі характеристики

- `castle` — home Castle.
- `army` — поточна Army; `null`, якщо Knight не входить до окремої multi-Unit Army / відповідно до остаточного представлення одиночного Unit.
- `soldiers` — склад Soldier за типами.
- `location_state` — локальний стан Unit, зокрема Castle / Camp там, де це має окремий сенс.
- `status` — життєвий стан Knight.

### Динамічні характеристики

- `experience`

### Обчислювальні характеристики

- `experience_balance` — effective passive rate зміни Experience; залежить від характеристик поточного `army`/стану Knight за правилами Experience.

### User methods

- `user_enter_castle()` — переводить Unit з Camp у Castle після перевірки можливості входу.
- `user_leave_castle()` — переводить Unit з Castle у Camp.
- `user_change_unit_composition()` — змінює склад Soldier Unit через обмін із reserve його Castle.

### Triggers

- немає зафіксованих на цей момент.

---

## 6. `Army`

### Прямі характеристики

- `current_region` — поточна Region Army.
- `camp` — поточний `CampInRegion`, якщо Army перебуває в Camp; встановлюється при вході в Camp і очищується при виході.
- `commander` — Commander-in-Chief.
- `defense_loss_threshold`
- `target_combat_threshold`
- `incidental_combat_threshold`
- `state` — поточний operational state Army, зокрема Camp / Movement / Regrouping та інші потрібні V1 підстани.

### Динамічні характеристики

- `regrouping_progress` — progress Regrouping, коли Army перебуває у цьому стані.

### Обчислювальні характеристики

- `*knights[]` — Knight, які поточно входять до Army.
- `*movement` — поточний active Movement цієї Army; визначається reverse lookup за `Movement.army`.
- `regrouping_progress_rate` — rate Regrouping progress; у поточних правилах константа, визначена конфігурацією.

### User methods

- `user_merge_armies()` — об'єднує сумісні Army.
- `user_split_army()` — розділяє Army на окремі Army.
- `user_change_commander()` — змінює Commander-in-Chief.
- `user_change_combat_thresholds()` — змінює налаштовувані combat thresholds Army.

### Triggers

- `regrouping_complete` `[event trigger]` — спрацьовує, коли `regrouping_progress` досягає завершення; Army виходить із Regrouping.

---

# Persistent interaction/process models

## 7. `CombatSituation`

### Прямі характеристики

- `region`
- `attacker` — зафіксований attacker.
- `defenders` — зафіксований набір сторони defender.
- `combat_type` / context — потрібний контекст combat.
- `defenders_retreat_decisions` / інші зафіксовані pre-battle decisions.
- `locked_combat_parameters` — параметри, які за правилами фіксуються до resolve і не повинні змінитися від пізніших user changes.
- `status`

### Динамічні характеристики

- `battle_start_progress` — progress до моменту Battle Start / Resolve, якщо застосовується Combat Start Delay.

### Обчислювальні характеристики

- `battle_start_progress_rate` — rate progress до battle start; у поточних правилах константа, визначена конфігурацією.
- `is_battle_valid` — чи CombatSituation все ще має актуальні сторони/умови для Battle Start / Resolve; залежить від зафіксованих `attacker`, `defenders` і їх актуального стану.

### User methods

- `user_stop_transit_for_defense()` — фіксує рішення зупинити Transit для участі в defense та застосовує пов'язані зміни до combat/movement state.
- `user_set_pre_battle_retreat_decision()` — задає або змінює pre-battle retreat decision.
- `user_attack_neutral_defense()` — створює/налаштовує combat проти Neutral Defense і запускає відповідний combat process.

### Triggers

- `battle_start` `[event trigger]` — спрацьовує, коли `battle_start_progress` досягає завершення і `is_battle_valid`; виконується Battle Start / Resolve.

---

## 8. `CampInRegion`

Один active instance представляє один безперервний епізод присутності Army конкретного `Player` у Camp конкретної `Region`. Модель використовується для Camp-level interaction незалежно від того, чи Region є власною, чужою або Neutral, і бере на себе функції накопичення можливості Annexation.

Модель створюється при вході першої Army цього Player у Camp. Коли остання Army залишає Camp, instance логічно видаляється / деактивується, але фізичний запис зберігається для history/rollback. Наступна поява Army цього Player у Camp створює новий instance.

### Прямі характеристики

- `player`
- `region`
- `status`

### Динамічні характеристики

- `control_progress` — progress можливості Annexation; не накопичується, якщо немає Army цього Player у Camp.

### Обчислювальні характеристики

- `*armies[]` — Army, для яких цей active `CampInRegion` є прямою характеристикою `camp`.
- `has_valid_adjacent_owned_region` — чи `region.valid_connected_neighbor_player_ids[]` містить ID `player`.
- `can_progress` — чи зараз виконані всі умови накопичення `control_progress`; присутність чужих Camp Army визначається через `region.camp_player_ids[]`, а географічна умова — через `has_valid_adjacent_owned_region`.
- `control_progress_rate` — effective rate накопичення control progress; ненульовий лише коли `can_progress == true`.
- `required_control_progress` — необхідний control progress для поточного типу Region/контролю за конфігурацією.
- `is_ready_for_annexation` — чи `control_progress` досяг `required_control_progress`.

### User methods

- `user_annex_region()` — виконує Annexation після перевірки накопиченого control progress та інших актуальних умов, включно з вибором Castle.

### Triggers

- `can_progress` `[state trigger]` — спрацьовує при зміні стану `can_progress`; при втраті умов progress зупиняється, при їх відновленні — продовжується.
- `ready_for_annexation` `[event trigger]` — спрацьовує при досягненні `required_control_progress`; `CampInRegion` переходить у стан готовності до Annexation.

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

- `founder_valid` — founder Knight живий, залишається у потрібному стані/Region та не має Soldier.
- `can_progress` — чи зараз виконані всі умови продовження Founding; присутність foreign Camp Army визначається через `region.camp_player_ids[]`, а решта умов — через `founder_valid` та прямі характеристики процесу.
- `progress_rate` — effective rate Founding progress; ненульовий лише коли `can_progress == true`.
- `required_progress` — progress, потрібний для завершення Founding за конфігурацією.
- `is_complete` — чи `progress` досяг `required_progress`.

### User methods

- `user_start_castle_founding()` — створює та запускає процес Founding з вибраним founder Knight і параметрами нового Castle.

### Triggers

- `can_progress` `[state trigger]` — спрацьовує при зміні стану `can_progress`; при втраті умов Founding призупиняється, при їх відновленні — продовжується.
- `complete` `[event trigger]` — спрацьовує при завершенні progress, якщо Founding все ще валідний; створюється новий Castle та застосовуються наслідки Founding.

---

## 10. `Movement`

### Прямі характеристики

- `army`
- `route` — запланована послідовність Region.
- `current_region`
- `next_region`
- `phase` — поточна локальна фаза Movement, зокрема transit / рух до Camp.
- `status`
- `direction_revealed` — чи вже був пройдений reveal boundary поточної transit-фази.

### Динамічні характеристики

- `progress` — progress поточної фази / поточного проходження Region.

### Обчислювальні характеристики

- `progress_rate` — effective rate Movement progress; залежить від характеристик `army` та поточної Movement phase/Region.
- `direction_reveal_progress` — progress boundary, після якого відкривається напрямок виходу з поточної Region.
- `phase_completion_progress` — progress boundary завершення поточної Movement phase.

### User methods

- `user_start_movement()` — створює та запускає Movement для Army за заданим route.
- `user_change_planned_route()` — змінює ще не пройдений запланований route Movement.

### Triggers

- `direction_revealed` `[state trigger]` — відстежує стан досягнення `direction_reveal_progress`; при вході в revealed-state фіксує `direction_revealed = true` та запускає frontend/notification effect, а при переході до нової transit-фази стан trigger-а повертається назад.
- `next_region_reached` `[event trigger]` — спрацьовує при завершенні transit-фази; Army входить у `next_region`, Movement переходить до наступної фази.
- `camp_reached` `[event trigger]` — спрацьовує при завершенні локального руху до Camp/interaction point; Army завершує цю Movement phase, отримує пряме посилання `camp` на відповідний `CampInRegion` і переходить до Camp state; якщо active `CampInRegion` для цієї пари Player/Region ще немає, він створюється.

---

## 11. `Recruitment`

Один Castle має одну послідовну Recruitment Queue.

### Прямі характеристики

- `castle`
- `queue` — замовлення Recruitment у порядку виконання.
- `current_order` / позиція всередині поточного order.
- `status`

### Динамічні характеристики

- `progress` — progress поточного Soldier.

### Обчислювальні характеристики

- `can_progress` — чи зараз виконані всі умови Recruitment поточного Soldier: є активний order і немає блокуючих умов; залежить від характеристик `castle` і поточного order.
- `progress_rate` — effective recruitment rate; ненульовий лише коли `can_progress == true`.
- `required_progress` — progress, потрібний для завершення поточного Soldier відповідно до його Type/configuration.
- `current_recruit_finished` — чи `progress` досяг `required_progress`.

### User methods

- `user_add_recruitment_order()` — додає нове замовлення в Recruitment Queue.

### Triggers

- `can_progress` `[state trigger]` — спрацьовує при зміні стану `can_progress`; при появі блокуючої умови Recruitment призупиняється, при її зникненні — продовжується.
- `current_recruit_finished` `[event trigger]` — спрацьовує при завершенні поточного Soldier; Soldier додається до Castle reserve, queue переходить до наступного елемента, progress скидається для наступного Soldier.

---

## 12. `BuildingUpgrade`

Один instance = один конкретний процес upgrade однієї Building.

Після завершення instance не видаляється, а переходить у terminal status.

### Прямі характеристики

- `castle`
- `building_type`
- `target_level`
- `status`

### Динамічні характеристики

- `progress`

### Обчислювальні характеристики

- `progress_rate` — rate Building upgrade progress; у поточних правилах визначається конфігурацією.
- `required_progress` — progress, потрібний для завершення upgrade до `target_level`.
- `is_complete` — чи `progress` досяг `required_progress`.

### User methods

- `user_start_building_upgrade()` — створює та запускає upgrade вибраної Building до наступного level.

### Triggers

- `complete` `[event trigger]` — спрацьовує при завершенні progress; Castle отримує `target_level`, а BuildingUpgrade переходить у terminal status.

---

## 13. `ResourceSiteUpgrade`

Один instance = один конкретний процес upgrade вибраної кількості ResourceSite одного типу/рівня.

Після завершення instance не видаляється, а переходить у terminal status.

### Прямі характеристики

- `region`
- `resource_type`
- `target_level`
- `quantity`
- `status`

### Динамічні характеристики

- `progress`

### Обчислювальні характеристики

- `progress_rate` — rate ResourceSite upgrade progress; у поточних правилах визначається конфігурацією.
- `required_progress` — progress, потрібний для завершення upgrade до `target_level` для заданої `quantity`.
- `is_complete` — чи `progress` досяг `required_progress`.

### User methods

- `user_start_resource_site_upgrade()` — створює та запускає upgrade вибраної кількості ResourceSite.

### Triggers

- `complete` `[event trigger]` — спрацьовує при завершенні progress; Region оновлює відповідні ResourceSite, а ResourceSiteUpgrade переходить у terminal status.

---

## 14. `KnightReplacement`

Один instance = один конкретний процес створення replacement Knight для звільненого Palace slot.

Процес створює Castle як наслідок загибелі Knight. Після завершення instance не видаляється, а переходить у terminal status.

### Прямі характеристики

- `castle`
- `status`

### Динамічні характеристики

- `progress`

### Обчислювальні характеристики

- `*castle_replacements[]` — усі KnightReplacement цього Castle, потрібні для визначення черги процесів.
- `progress_rate` — effective rate Knight replacement progress; залежить від стану `castle_replacements[]`; для процесів, що очікують своєї черги, дорівнює `0`.
- `required_progress` — progress, потрібний для створення replacement Knight за конфігурацією.
- `is_complete` — чи `progress` досяг `required_progress`.

### User methods

- немає: створення KnightReplacement є наслідком battle і виконується через Castle domain logic.

### Triggers

- `complete` `[event trigger]` — спрацьовує при завершенні progress; Castle створює replacement Knight, поточний KnightReplacement переходить у terminal status, а наступний waiting process за потреби стає активним.