# Актуальний перелік тригерів моделей

Джерела: `docs/MODELS_CORE.md`, `docs/MODELS_MILITARY.md`, `docs/MODELS_COMBAT.md`, `docs/MODELS_PROCESSES.md` (гілка `main`). Цей файл — інвентаризація наявних декларацій, а не нові правила. Назви тригерів локальні для відповідної моделі.

| Модель | Тригер | Тип | Призначення / дія |
| --- | --- | --- | --- |
| Player | `empty_coins` | passive state | Межа вичерпання монет; перерахунок залежних rates без окремої ігрової дії. |
| Castle | `empty_food` | passive state | Межа вичерпання їжі. |
| Castle | `storage_capacity` | passive state | Межа заповнення ресурсного сховища. |
| Region | `neutral_defense_recovery_complete` | passive event | Завершення відновлення нейтральної оборони; без додаткової ігрової дії. |
| City | `active_wealth_recovered` | passive event | Відновлення `active_wealth_ratio` до 1; зміна `wealth_balance`. |
| Army | `regrouping_complete` | event | Завершення перегрупування через `finish_regrouping()`. |
| CampInRegion | `can_progress` | state | Зміна можливості накопичувати контроль; перерахунок `control_progress_rate`. |
| CampInRegion | `ready_for_annexation` | event | Досягнута готовність до приєднання; саме приєднання залишається дією гравця. |
| Movement | `direction_revealed` | state | Розкриття напрямку руху та оновлення видимості. |
| Movement | `next_region_reached` | event | Досягнення наступної області; перехід до наступної фази руху. |
| Movement | `camp_reached` | event | Завершення прибуття до табору з урахуванням бойового очікування. |
| CombatSituation | `battle_start` | event | Настання Battle Start активної БС; передбойові рішення, розрахунок і завершення. |
| CastleFounding | `founder_invalid` | state | Засновник більше не відповідає умовам; скасування заснування. |
| CastleFounding | `can_progress` | state | Зміна можливості прогресу; перерахунок швидкості/планування. |
| CastleFounding | `complete` | event | Завершення заснування замку. |
| Recruitment | `can_progress` | state | Зміна можливості рекрутингу; перерахунок швидкості/планування. |
| Recruitment | `current_recruit_finished` | event | Завершення рекрутингу чергового солдата за наявності місткості. |
| BuildingUpgrade | `complete` | event | Завершення покращення будівлі. |
| ResourceSiteUpgrade | `complete` | event | Завершення покращення ресурсного об'єкта. |
| KnightReplacement | `complete` | event | Завершення заміни лицаря. |

**Разом: 20 тригерів у 12 моделях.** У `Knight` тригерів немає.

Для кожного запису в документах моделей визначено пару `check_trigger_<name>()` / `on_trigger_<name>()`. Повний перелік методів разом із тригерами: [MODEL_METHODS_AND_TRIGGERS.md](MODEL_METHODS_AND_TRIGGERS.md).

**Примітка щодо БС:** перехід до наступної БС у FIFO після завершення попередньої виконується синхронно через `Region.start_next_combat_if_possible()`, **не окремим тригером**. Транзитне долучення до оборони у V1 не вводиться.
