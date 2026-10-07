# Backend models — persistent processes

Long-running construction, recruitment, founding and replacement processes.

Загальні conventions, system-computed semantics та спільні припущення див. у [MODELS.md](MODELS.md).

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

---

## 11. `Recruitment`

Один Castle має одну послідовну FIFO Recruitment Queue. Кожний Soldier проходить окремий повний recruitment cycle.

### Прямі характеристики

- `castle`.
- `queue` — orders `{soldier_type, remaining_quantity}` у незмінному FIFO order.
- `current_order_index`.
- `status`.

### Динамічні характеристики

- `progress` — progress **поточного одного Soldier**, а не всього order; при pause зберігається.

### Обчислювальні характеристики

- `current_order`.
- `can_progress` — є current order, `castle.empty_food == false`, `castle.player_empty_coins == false`, `castle.barracks_free_capacity > 0`.
- `progress_rate`, `required_progress`, `current_recruit_finished`. Якщо progress already complete, але Capacity зникла, completion залишається ready до появи місця.

### User methods

- `user_add_recruitment_order(soldier_type, quantity)` — тільки append; після validation атомарно списує повну upfront cost всього quantity і додає order у хвіст. Cancellation/reorder/priority change немає.

### Domain methods

- `validate_new_order(type, quantity)` — valid Type, `quantity > 0`, `!castle.empty_food`, `!castle.player_empty_coins`, `castle.barracks_free_capacity > 0` та достатньо фактичних local/global resources для **повної** order cost. Capacity для всього quantity не потрібна; artificial max quantity/queue length немає.
- `finish_current_recruit()` — за наявності одного вільного Barracks slot створює рівно 1 Soldier через `Castle.add_recruited_soldier()`, декрементує `remaining_quantity`, скидає per-Soldier progress; при `remaining_quantity == 0` переходить до наступного FIFO order. Якщо slot немає, нічого не створює й лишає completed progress ready.

### Triggers

- `can_progress` `[state trigger]`.
- `current_recruit_finished` `[event trigger]`.

### Trigger methods

- `check_trigger_can_progress()`, `on_trigger_can_progress()`.
- `check_trigger_current_recruit_finished()`, `on_trigger_current_recruit_finished()`.

---

---

## 12. `BuildingUpgrade`

Один instance = один process upgrade однієї Building. Різні Building одного Castle можуть upgrade-итися паралельно; для одного concrete `building_type` active instance може бути максимум один.

### Прямі характеристики

- `castle`, `building_type`, `target_level`, `status`.

### Динамічні характеристики

- `progress`.

### Обчислювальні характеристики

- `progress_rate` — після успішного start не pause-иться через `empty_food`, `empty_coins` або подальшу зміну prerequisite state.
- `required_progress`, `is_complete`.

### User methods

- `user_start_building_upgrade(castle, building_type)` — визначає `target_level = current + 1`, перевіряє відсутність active upgrade цієї Building, повну upfront cost та config prerequisites для `0 -> 1`, атомарно списує cost і запускає process. Manual cancellation/refund немає.

### Domain methods

- `validate_start()` — prerequisites є config-driven набором умов `other_building_level >= min_level`; у V1 всі вони застосовуються лише для first construction `0 -> 1` і мають виконуватися одночасно. `empty_food`/`empty_coins` окремо не блокують start, якщо фактичних ресурсів для upfront cost достатньо.
- `complete()` — застосовує target level через `Castle.apply_building_upgrade()`; Palace completion також додає одного ready unnamed Knight у спільну FIFO чергу.

### Triggers / Trigger methods

- `complete` `[event trigger]`.
- `check_trigger_complete()`, `on_trigger_complete()`.

---

---

## 13. `ResourceSiteUpgrade`

Один instance = один parallel process upgrade вибраної кількості ResourceSite одного `resource_type` з одного фактичного `from_level` у `target_level = from_level + 1`. Іменованих Site не потрібно; order резервує count із відповідного level bucket.

### Прямі характеристики

- `region`, `resource_type`, `from_level`, `target_level`, `quantity`, `status`.

### Динамічні характеристики

- `progress`.

### Обчислювальні характеристики

- `progress_rate` — після start постійний за config transition і не pause-иться через Occupation/disconnection/Neutral/owner change.
- `required_progress` — залежить від level transition, але **не** від `quantity`.
- `is_complete`.

### User methods

- `user_start_resource_site_upgrade(region, resource_type, quantity)` — дозволено тільки current owner для Owned non-Occupied valid-connected Region; pure foreign Transit не блокує. Резервує `quantity` eligible Site та атомарно списує upfront `per_site_cost × quantity`. Manual cancellation/refund немає.

### Domain methods

- `validate_start()` — перевіряє `quantity > 0`, ownership/non-Occupied/connection, layered rule за **фактично completed** site levels, достатню кількість незарезервованих Site current minimum level і повну upfront cost. Active reservations не вважаються completed level і не можуть бути зарезервовані повторно.
- `complete()` — незалежно від поточного owner/state Region переводить зарезервований count із `from_level` у `target_level`; до цього моменту ці Site виробляють як `from_level`.

### Triggers / Trigger methods

- `complete` `[event trigger]`.
- `check_trigger_complete()`, `on_trigger_complete()`.

---

---

## 14. `KnightReplacement`

Один instance = один replacement **timer** для Palace slot, звільненого death/іншим виходом active Knight із home Castle. Сам Knight при completion timer ще не створюється.

### Прямі характеристики

- `castle`, `status` — queued / active / timer_complete.

### Динамічні характеристики

- `progress`.

### Обчислювальні характеристики

- `is_queue_head` — визначається через `castle.active_knight_replacement_id` серед replacement, timer яких ще не завершився.
- `progress_rate` — ненульовий тільки для active queue head.
- `required_progress`, `is_complete`.

### Domain methods

- `complete()` — завершує timer, додає один ready unnamed Knight у `castle.ready_knights_awaiting_name`, звільняє replacement timer queue для наступного queued instance. Очікування Player name не блокує наступний replacement timer; усі ready unnamed Knight разом із ready Knight від Palace upgrade використовують одну спільну FIFO name queue.

### Triggers / Trigger methods

- `complete` `[event trigger]`.
- `check_trigger_complete()`, `on_trigger_complete()`.
