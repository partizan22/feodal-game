# Backend models

Робочий опис моделей. На цьому етапі для кожної моделі фіксуються:

- прямі характеристики;
- динамічні характеристики;
- тільки ті обчислювальні характеристики, які безпосередньо використовуються для обчислення динамічних;
- user methods, що реалізують основну логіку відповідних user actions.

Root model для user GameEvent поки не визначена до проєктування взаємодії з frontend/API. Тому `user_*` тут означає модель, яка реалізує основну логіку дії, а не обов'язково майбутню root model. Після визначення root model префікси методів за потреби будуть змінені.

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

### Обчислювальні характеристики для dynamic

- `coin_balance` — effective rate зміни Coins.
- `gold_balance` — effective rate зміни Gold.
- `silver_balance` — effective rate зміни Silver.

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

### Обчислювальні характеристики для dynamic

- `resource_balance` — один клас із effective rate для `wood`, `stone`, `iron`; враховує production/flows і межі Warehouse.
- `food_balance` — effective rate зміни Food; враховує production/flows, consumption і межі Granary/zero.

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

### Обчислювальні характеристики для dynamic

- немає.

### User methods

- `user_change_allow_transit()` — змінює правило Transit для Owned Region.

---

## 4. `City`

### Прямі характеристики

- `region`

### Динамічні характеристики

- `wealth`

### Обчислювальні характеристики для dynamic

- `wealth_balance` — effective rate зміни Wealth з урахуванням поточного стану Region/Player та правил City.

### User methods

- `user_raid_city()` — виконує raid City: застосовує наслідки до Wealth і пов'язаних результатів raid.

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

### Обчислювальні характеристики для dynamic

- `experience_balance` — effective passive rate зміни Experience.

### User methods

- `user_enter_castle()` — переводить Unit з Camp у Castle після перевірки можливості входу.
- `user_leave_castle()` — переводить Unit з Castle у Camp.
- `user_change_unit_composition()` — змінює склад Soldier Unit через обмін із reserve його Castle.

---

## 6. `Army`

### Прямі характеристики

- `current_region` — поточна Region Army.
- `commander` — Commander-in-Chief.
- `defense_loss_threshold`
- `target_combat_threshold`
- `incidental_combat_threshold`
- `state` — поточний operational state Army, зокрема Camp / Movement / Regrouping та інші потрібні V1 підстани.

### Динамічні характеристики

- `regrouping_progress` — progress Regrouping, коли Army перебуває у цьому стані.

### Обчислювальні характеристики для dynamic

- `regrouping_progress_rate` — rate Regrouping progress; у поточних правилах константа, визначена конфігурацією.

### User methods

- `user_merge_armies()` — об'єднує сумісні Army.
- `user_split_army()` — розділяє Army на окремі Army.
- `user_change_commander()` — змінює Commander-in-Chief.
- `user_change_combat_thresholds()` — змінює налаштовувані combat thresholds Army.

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

### Обчислювальні характеристики для dynamic

- `battle_start_progress_rate` — rate progress до battle start; у поточних правилах константа, визначена конфігурацією.

### User methods

- `user_stop_transit_for_defense()` — фіксує рішення зупинити Transit для участі в defense та застосовує пов'язані зміни до combat/movement state.
- `user_set_pre_battle_retreat_decision()` — задає або змінює pre-battle retreat decision.
- `user_attack_neutral_defense()` — створює/налаштовує combat проти Neutral Defense і запускає відповідний combat process.

---

## 8. `RegionControl`

Один instance представляє стан контролю конкретного `Player` у конкретній `Region`; control progress не прив'язаний до Castle.

### Прямі характеристики

- `player`
- `region`
- `status`

### Динамічні характеристики

- `control_progress`

### Обчислювальні характеристики для dynamic

- `control_progress_rate` — effective rate накопичення control progress; стає нульовим, коли умови накопичення не виконуються.

### User methods

- `user_annex_region()` — виконує Annexation після перевірки накопиченого control progress та інших актуальних умов, включно з вибором Castle.

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

### Обчислювальні характеристики для dynamic

- `progress_rate` — effective rate Founding progress; враховує pause conditions.

### User methods

- `user_start_castle_founding()` — створює та запускає процес Founding з вибраним founder Knight і параметрами нового Castle.

---

## 10. `Movement`

### Прямі характеристики

- `army`
- `route` — запланована послідовність Region.
- `current_region`
- `next_region`
- `phase` — поточна локальна фаза Movement, зокрема transit / рух до Camp.
- `status`

### Динамічні характеристики

- `progress` — progress поточної фази / поточного проходження Region.

### Обчислювальні характеристики для dynamic

- `progress_rate` — effective rate Movement progress.

### User methods

- `user_start_movement()` — створює та запускає Movement для Army за заданим route.
- `user_change_planned_route()` — змінює ще не пройдений запланований route Movement.

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

### Обчислювальні характеристики для dynamic

- `progress_rate` — effective recruitment rate; `0`, коли Recruitment paused через Food, Coins, Barracks capacity або іншу блокуючу умову.

### User methods

- `user_add_recruitment_order()` — додає нове замовлення в Recruitment Queue.

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

### Обчислювальні характеристики для dynamic

- `progress_rate` — rate Building upgrade progress; у поточних правилах визначається конфігурацією.

### User methods

- `user_start_building_upgrade()` — створює та запускає upgrade вибраної Building до наступного level.

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

### Обчислювальні характеристики для dynamic

- `progress_rate` — rate ResourceSite upgrade progress; у поточних правилах визначається конфігурацією.

### User methods

- `user_start_resource_site_upgrade()` — створює та запускає upgrade вибраної кількості ResourceSite.

---

## 14. `KnightReplacement`

Один instance = один конкретний процес створення replacement Knight для звільненого Palace slot.

Процес створює Castle як наслідок загибелі Knight. Після завершення instance не видаляється, а переходить у terminal status.

### Прямі характеристики

- `castle`
- `status`

### Динамічні характеристики

- `progress`

### Обчислювальні характеристики для dynamic

- `progress_rate` — effective rate Knight replacement progress; для процесів, що очікують своєї черги, дорівнює `0`.
