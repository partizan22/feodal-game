# Backend models

Робочий опис моделей. На цьому етапі для кожної моделі фіксуються:

- прямі характеристики;
- динамічні характеристики;
- тільки ті обчислювальні характеристики, які безпосередньо використовуються для обчислення динамічних;
- user actions / майбутні `user_*` methods.

Для user actions root model поки не визначена, тому вони не прив'язуються до конкретної моделі як `user_*` method. Нижче вони наведені окремо у форматі `? -> LogicModel(s)`.

Рівні всіх Building одного Castle зберігаються як одна характеристика `buildings`.

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

- `coordinates` — положення Region на hex map. -не впевнений, що вона взагалі потрібна, а якщо й так - то це скоріщ буде system-computet.
- `player` — формальний owner; `null` для Neutral Region. - взагалі-то з Player пов'язується через Castle. Може бути обчислювальна характеристика, але можна залишити й так. Подумай.
- `castle` — Castle, до якого Region приєднана; `null`, якщо не належить Castle.
- `resource_sites` — структура ResourceSite за типами ресурсів і рівнями.
- `allow_transit` — правило Transit для Owned Region.
- `is_connection_valid` — чи має Owned Region чинний зв'язок зі своїм Castle.
- `neutral_defense` — актуальний стан Neutral Defense.
- `neutral_defense_recovery_started_at` — Game Time початку очікування повного відновлення Neutral Defense, якщо recovery активний. - можливо варто  виділити окрему модель

### Динамічні характеристики

- немає зафіксованих на цей момент.

### Обчислювальні характеристики для dynamic

- немає.

---

## 4. `City`

### Прямі характеристики

- `region`

### Динамічні характеристики

- `wealth`

### Обчислювальні характеристики для dynamic

- `wealth_balance` — effective rate зміни Wealth з урахуванням поточного стану Region/Player та правил City.

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

- `regrouping_progress_rate` — effective rate Regrouping progress. - навіщо ця характеристика і від чого вона залежить? Час regrouping фіксований

---

# Persistent interaction/process models

## 7. `CombatSituation`

### Прямі характеристики

- `region`
- `attacker` — зафіксований набір сторони attacker.  - атакуючий один
- `defenders` — зафіксований набір сторони defender.
- `combat_type` / context — потрібний контекст combat.
- `defenders_retreat_decisions` / інші зафіксовані pre-battle decisions.
- `locked_combat_parameters` — параметри, які за правилами фіксуються до resolve і не повинні змінитися від пізніших user changes.
- `status`

### Динамічні характеристики

- `battle_start_progress` — progress до моменту Battle Start / Resolve, якщо застосовується Combat Start Delay.

### Обчислювальні характеристики для dynamic

- `battle_start_progress_rate` — effective rate progress до battle start. - навіщо ця характеристика і від чого вона залежить? Час до початку бою від входу в region фіксований.

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

---

## 10. `Movement`

### Прямі характеристики

- `army` - здається, логічніше в Army мати посилання на її Movement як пряму характеристику, а тут - system-computet. Подумай, як краще
- `route` — запланована послідовність Region.
- `current_region`
- `next_region`
- `phase` — поточна локальна фаза Movement, зокрема transit / рух до Camp.
- `status`

### Динамічні характеристики

- `progress` — progress поточної фази / поточного проходження Region.

### Обчислювальні характеристики для dynamic

- `progress_rate` — effective rate Movement progress.

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

---

## 12. `BuildingUpgrade`

Один instance = один конкретний процес upgrade однієї Building.

Після завершення instance не видаляється, а переходить у terminal status.

### Прямі характеристики

- `castle`
- `building_type`
- `from_level`
- `target_level` - навіщо окрема характеристика? ми оновлюєм на один рівень. Хоча логічніше, якраз мати target_level а не from.
- `status`

### Динамічні характеристики

- `progress`

### Обчислювальні характеристики для dynamic

- `progress_rate` — effective rate Building upgrade progress.

---

## 13. `ResourceSiteUpgrade`

Один instance = один конкретний процес upgrade вибраної кількості ResourceSite одного типу/рівня.

Після завершення instance не видаляється, а переходить у terminal status.

### Прямі характеристики

- `region`
- `resource_type`
- `from_level` - аналогічно
- `target_level`
- `quantity`
- `status`

### Динамічні характеристики

- `progress`

### Обчислювальні характеристики для dynamic

- `progress_rate` — effective rate ResourceSite upgrade progress.

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

- `progress_rate` — effective rate Knight replacement progress.

---

# User actions / майбутні `user_*` methods

Root model для user GameEvent поки не визначена до проєктування взаємодії з frontend/API. Тому цей список не означає, що `user_*` method уже належить першій LogicModel після `->`.

- `user_start_movement` — `? -> Army, Movement`
- `user_change_planned_route` — `? -> Movement`
- `user_enter_castle` — `? -> Knight, Castle`
- `user_leave_castle` — `? -> Knight, Castle`
- `user_stop_transit_for_defense` — `? -> CombatSituation, Movement, Army`
- `user_merge_armies` — `? -> Army`
- `user_split_army` — `? -> Army`
- `user_change_commander` — `? -> Army`
- `user_change_unit_composition` — `? -> Castle, Knight`
- `user_change_combat_thresholds` — `? -> Army`
- `user_change_allow_transit` — `? -> Region`
- `user_set_pre_battle_retreat_decision` — `? -> CombatSituation`
- `user_attack_neutral_defense` — `? -> RegionControl, CombatSituation`
- `user_annex_region` — `? -> RegionControl, Region, Castle`
- `user_start_castle_founding` — `? -> Player, CastleFounding`
- `user_raid_city` — `? -> RegionControl, City`
- `user_start_building_upgrade` — `? -> Castle, BuildingUpgrade`
- `user_start_resource_site_upgrade` — `? -> Region, ResourceSiteUpgrade`
- `user_add_recruitment_order` — `? -> Castle, Recruitment`
