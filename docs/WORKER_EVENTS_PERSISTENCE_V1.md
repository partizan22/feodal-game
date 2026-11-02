# Worker, GameEvent, rollback і persistence моделей — погоджені рішення (V1)

Цей документ фіксує останні рішення для підготовки задачі Codex на реалізацію. Він доповнює `FEUDALS_BACKEND_ARCHITECTURE_CURRENT.md` і `BASE_MODEL.md`. Якщо старі описи черг, знімків стану або відкату суперечать цьому документу, для цих питань актуальним є цей документ. Конкретна реалізація SQL-міграцій і PHP-класів — наступний етап.

## 1. Загальна модель виконання

- Є один довгоживучий Game Worker, який послідовно обробляє GameEvent; одна ітерація — одна GameEvent.
- Зовнішні джерела GameEvent: **дія користувача** (`user_action`) або **запланована перевірка тригера** (`trigger`). Окремого третього типу зовнішньої черги для відкладених domain actions у V1 немає.
- GameEvent має один фіксований ігровий час `Te`. Усі моделі в її межах працюють з тим самим `Te`; реальний час виконання коду не змінює `Te`.
- Worker вибирає подію, визначає `Te`, створює EventContext, ініціалізує root Model, викликає відповідний entry point, проводить загальний post-action pipeline і commit. Ігрова логіка залишається в моделях.
- Вся обробка однієї GameEvent, включно зі станами моделей, проєкціями зв'язків, записами черг і `event_log`, фіксується **атомарно в одній транзакції**.
- Scheduled trigger — це перевірка в заданий час, а не безумовний виклик trigger action. При виконанні умова перевіряється повторно; перевірка без domain-змін теж вважається виконаною GameEvent.

### Pipeline GameEvent

1. Worker вибирає запис джерела і визначає `Te`.
2. Створюється EventContext / IdentityMap; root Model ініціалізується на `Te`.
3. Виконується `user_*` або перевірка тригера з умовним викликом `on_trigger_*`.
4. Після кореневої дії береться snapshot поточного списку initialized моделей; для них виконується DFS за правилами основної архітектури.
5. Береться оновлений список initialized моделей (включно з lazy-loaded під час DFS); перевіряються їхні тригери, оновлюються прогнози.
6. Виконується commit потрібних моделей, їхніх повних знімків стану та проєкцій зв'язків; фіксується факт виконання джерела і запис `event_log`.
7. Транзакція завершується. Інші trigger actions не виконуються каскадно в цій GameEvent.

## 2. Хронологічний вибір подій із двох черг

Є дві незалежні черги: `user_actions` і `trigger_events`. Обидві мають `scheduled_time`; GameEvent завжди виконується з `Te = scheduled_time` вибраного запису. Прострочені тригери **не зсуваються** на поточний час.

Перед **кожною** ітерацією вибору події worker:

1. Якщо `game_clock.paused = true`, не призначає час новим діям і не виконує жодних GameEvent.
2. Зчитує поточний Game Time за `game_clock`.
3. Усім невиконаним `user_actions` з `scheduled_time IS NULL`, які вже є в черзі на момент цього кроку, **присвоює однаковий поточний Game Time**. Це окрема атомарна операція, яка не виконує дій; порядок таких дій надалі визначається `id`.
4. Знаходить по одному найранішому готовому кандидату з кожної черги (`scheduled_time <= current_game_time`), порівнює їх та вибирає одну GameEvent.
5. Сортування спільне: **`scheduled_time ASC`, потім тригер перед дією користувача, потім `id ASC`**.
6. Якщо готових записів немає, очікує. Інакше виконує одну GameEvent та повторює цикл.

Початкове додавання user action в API: `scheduled_time = NULL`. Час дії призначає **сам worker**, не API і не під час виконання дії. Дії, що з'явилися після кроку призначення часу, отримають час на наступній ітерації. Якщо тригер з часом `T` створює інший тригер на `T`, наступний вибір знову порівнює обидві черги, а при рівності часу пріоритет має тригер.

Приклад SQL вибору наступної готової події (кожна гілка повертає щонайбільше один запис):

```sql
SELECT type, id, scheduled_time
FROM (
    (
        SELECT 'trigger' AS type, id, scheduled_time, 0 AS priority
        FROM trigger_events
        WHERE cancelled_event_id IS NULL
          AND executed_event_id IS NULL
          AND scheduled_time <= :current_time
        ORDER BY scheduled_time, id
        LIMIT 1
    )
    UNION ALL
    (
        SELECT 'user_action' AS type, id, scheduled_time, 1 AS priority
        FROM user_actions
        WHERE executed_event_id IS NULL
          AND scheduled_time IS NOT NULL
          AND scheduled_time <= :current_time
        ORDER BY scheduled_time, id
        LIMIT 1
    )
) q
ORDER BY scheduled_time, priority, id
LIMIT 1;
```

Цей порядок усуває ситуацію, коли дія користувача випереджає вже прострочені тригери через затримку worker-а. При однаковому часі всі готові trigger checks обробляються перед user actions цього часу. Спеціального правила «закінчити групу тригерів» не потрібно.

## 3. Ігровий час і точність

- Один tick Game Time = 1 мс ігрового часу; зберігається як `bigint`.
- `game_time = sync_game_time + round((real_time_ms - sync_real_time_ms) * speed / 10)`.
- `speed = 10` означає 1:1; зміна швидкості використовує нову пару опорних точок, без накопичення похибки округлення.
- Кожна GameEvent (тригер або дія користувача) виконується на її `scheduled_time`, навіть якщо worker фактично обробив її пізніше.
- Для динамічних значень зберігається раніше погоджена цілочисельна точність: 1 ігрова одиниця = 36 000 000 000 технічних одиниць. Динамічні значення зберігаються як цілі числа.

### Таблиця `game_clock` — поточна опорна точка часу

Є один рядок `game_clock` (`id = 1`):

| Колонка | Тип | Значення |
| --- | --- | --- |
| `id` | `smallint PRIMARY KEY` | Завжди `1` |
| `sync_game_time` | `bigint NOT NULL` | Ігровий час в опорній точці |
| `sync_real_time` | `bigint NOT NULL` | Unix timestamp опорної точки, мс |
| `speed` | `integer NOT NULL` | Швидкість; `10` = 1:1 |
| `paused` | `boolean NOT NULL` | Пауза обробки подій |
| `last_event_id` | `bigint NULL` | ID останньої виконаної GameEvent |

Початково: `sync_game_time = 0`, `sync_real_time = час запуску/ініціалізації`, `speed = 10`, `paused = false`, `last_event_id = NULL`.

Якщо `paused = false`, поточний Game Time розраховується як:

`sync_game_time + round((real_time_ms - sync_real_time) * speed / 10)`.

При `paused = true` ігровий час зафіксований у `sync_game_time`, worker не обробляє події. При `speed = 0`, `paused = false` ігровий час також не рухається, **але worker продовжує призначати час user actions і виконувати всі готові події**, включно з тригерами, час яких уже настав.

При зміні швидкості спочатку фіксується поточний ігровий час за старою швидкістю, далі `sync_game_time` і `sync_real_time` оновлюються до поточних значень, після чого записується нова швидкість. При встановленні паузи так само фіксується поточний час; при знятті паузи оновлюється `sync_real_time` без зміни `sync_game_time`. Ці операції виконуються атомарно з узгодженням worker-а.

`last_event_id` оновлюється в транзакції commit GameEvent. Він дозволяє отримати останній зафіксований ігровий час через `event_log`, але не замінює поточний час за формулою годинника.

## 4. SQL-таблиці

Назви типів моделей у посиланнях — **текстові** (`model text`), без числового довідника типів.

### 4.1. Таблиці екземплярів моделей

Кожен тип Model має власну таблицю (наприклад, `armies`, `regions`, `castles`, `combat_situations`). Спільні колонки:

| Колонка | Тип | Значення |
| --- | --- | --- |
| `id` | `bigint PRIMARY KEY` | ID екземпляра |
| `created_event_id` | `bigint NULL` | Подія створення; `NULL` для початкових екземплярів |
| `deleted_event_id` | `bigint NULL` | Подія логічного видалення; `NULL` — екземпляр не видалений |
| `current_state_id` | `bigint NULL` | Посилання на актуальний знімок у `model_states.id` |

Крім цих колонок, таблиця моделі має лише потрібні SQL-проєкції relationships (FK/junction та інші індексовані поля для пошуку). Ігрові характеристики не розкладаються на колонки цих таблиць; джерело їхніх значень — JSON-стан.

**Ці спільні поля застосовуються також до тимчасових моделей** (процесів, бойових ситуацій тощо). Видалення під час гри логічне, доки історія потрібна для rewind.

`current_state_id` дозволяє швидко завантажити актуальний стан без пошуку останнього snapshot. Він оновлюється в тій самій транзакції, що й вставка знімка. `NULL` допускається під час створення екземпляра до запису його першого знімка в межах транзакції; після успішного commit активний екземпляр повинен мати актуальний стан. Необхідно контролювати, що вказаний знімок належить саме цьому типу та ID екземпляра.

### 4.2. `model_states` — одна спільна таблиця повних станів

| Колонка | Тип | Значення |
| --- | --- | --- |
| `id` | `bigint PRIMARY KEY` | ID знімка |
| `model` | `text NOT NULL` | Тип моделі |
| `instance_id` | `bigint NOT NULL` | ID екземпляра |
| `event_id` | `bigint NOT NULL` | ID GameEvent, після якої записано знімок |
| `game_time` | `bigint NOT NULL` | Ігровий час знімка |
| `direct` | `jsonb NOT NULL` | Повний набір прямих характеристик |
| `dynamic` | `jsonb NOT NULL` | Повний набір динамічних характеристик |
| `computed` | `jsonb NOT NULL` | Повний набір обчислювальних характеристик, включно із system-computed |

Обмеження: `UNIQUE (model, instance_id, event_id)`.

Індекс читання історії: `(model, instance_id, event_id DESC)`.

Кожен commit зміненого екземпляра записує **повний знімок** усіх трьох категорій, а не diff і не попередні значення. Знімки незмінні; поточний стан вибирається через `current_state_id` відповідної таблиці моделі. `game_time` є спільною часовою опорою для динамічних значень у цьому знімку. Знімок містить актуалізовані на час commit обчислювальні характеристики.

Початковий знімок: **`event_id = 0`, `game_time = 0`**. `event_log` починається з реальних виконаних подій; запис у `event_log` з ID 0 не потрібний. Тому `model_states.event_id = 0` є спеціальним початковим значенням, а не звичайним FK на `event_log`.

Для одного екземпляра в одній GameEvent записується щонайбільше один підсумковий snapshot.

### 4.3. `event_log` — журнал виконаних GameEvent

| Колонка | Тип | Значення |
| --- | --- | --- |
| `id` | `bigint PRIMARY KEY` | Глобальний монотонний ID виконаної події |
| `game_time` | `bigint NOT NULL` | Ігровий час виконання |
| `type` | `text NOT NULL` | `user_action` або `trigger` |
| `source_id` | `bigint NOT NULL` | ID джерела у відповідній черзі |

Запис створюється навіть тоді, коли перевірка тригера не призвела до domain-змін. ID цієї події використовується для історії станів, lifecycle екземплярів і прогнозів. Одна GameEvent може змінити кілька моделей.

### 4.4. `user_actions` — черга дій користувача

| Колонка | Тип | Значення |
| --- | --- | --- |
| `id` | `bigint PRIMARY KEY` | ID дії |
| `model` | `text NOT NULL` | Тип root Model |
| `instance_id` | `bigint NOT NULL` | ID root Model |
| `method` | `text NOT NULL` | Назва `user_*` |
| `args` | `jsonb NOT NULL` | Аргументи |
| `executed_event_id` | `bigint NULL` | ID виконаної GameEvent |
| `scheduled_time` | `bigint NULL` | Ігровий час, призначений worker-ом; `NULL` до призначення |

`executed_event_id IS NULL` — дія очікує виконання. При реєстрації дії `scheduled_time IS NULL`; worker перед вибором наступної GameEvent встановлює час усім таким діям, використовуючи одне значення поточного Game Time. Після цього дії беруть участь у спільному хронологічному сортуванні з тригерами. Окремий `executed_at_time` не потрібний: ігровий час виконання доступний через `event_log`. Дії користувача **не replay-яться** після відкату.

Індекси:

```sql
CREATE INDEX idx_pending_user_actions
ON user_actions (scheduled_time, id)
WHERE executed_event_id IS NULL AND scheduled_time IS NOT NULL;

CREATE INDEX idx_unscheduled_user_actions
ON user_actions (id)
WHERE executed_event_id IS NULL AND scheduled_time IS NULL;
```

### 4.5. `trigger_events` — заплановані перевірки тригерів

| Колонка | Тип | Значення |
| --- | --- | --- |
| `id` | `bigint PRIMARY KEY` | ID прогнозу |
| `model` | `text NOT NULL` | Тип Model |
| `instance_id` | `bigint NOT NULL` | ID Model |
| `trigger` | `text NOT NULL` | Назва тригера |
| `scheduled_time` | `bigint NOT NULL` | Ігровий час перевірки |
| `created_event_id` | `bigint NULL` | Подія створення прогнозу |
| `cancelled_event_id` | `bigint NULL` | Подія скасування прогнозу |
| `executed_event_id` | `bigint NULL` | Подія виконання перевірки |

Активний прогноз: `cancelled_event_id IS NULL AND executed_event_id IS NULL`.

При зміні прогнозу попередній **логічно скасовується**, а новий створюється окремим рядком. Фізично старий прогноз не видаляємо, щоб rewind міг відновити його. `created_event_id = NULL` допускається для початкових прогнозів, створених під час ініціалізації гри.

Індекс черги: `(scheduled_time, id) WHERE cancelled_event_id IS NULL AND executed_event_id IS NULL`. Для відкату потрібні індекси на lifecycle-поля подій. При рівному `scheduled_time` тригери мають пріоритет над діями користувача.

## 5. Відкат (rewind) до події `E`

Маємо **лінійну історію без гілок**. Відкат остаточно відкидає майбутні події. Повторного виконання user actions немає; скасувати сам відкат неможливо.

У **єдиній транзакції**, зупинивши конкурентне виконання worker-а та приймання/виконання нових дій на час rewind:

1. Для кожного екземпляра відновити останній `model_states` з `event_id <= E`; видалити знімки з `event_id > E`.
2. Фізично видалити екземпляри моделей із `created_event_id > E`; для решти скинути `deleted_event_id = NULL`, якщо він був `> E`.
3. Встановити `current_state_id` на відновлений знімок; відновити SQL-проєкції relationships із його JSON-стану.
4. `DELETE FROM trigger_events WHERE created_event_id > E`.
5. `UPDATE trigger_events SET cancelled_event_id = NULL WHERE cancelled_event_id > E`.
6. `UPDATE trigger_events SET executed_event_id = NULL WHERE executed_event_id > E`.
7. `DELETE FROM user_actions WHERE executed_event_id > E OR executed_event_id IS NULL`.
8. `DELETE FROM event_log WHERE id > E`.
9. Оновити `game_clock`: `sync_game_time = event_log[E].game_time` (або `0` для `E = 0`), `sync_real_time = поточний Unix timestamp у мс`, `last_event_id = E` (або `NULL` для `E = 0`). `speed` і `paused` залишаються без змін. Узгодити відновлення worker-а з новою опорною точкою.

Фізичний порядок SQL-операцій може відрізнятися для дотримання FK; наведено **логічний** порядок. Усі пов'язані зміни атомарні.

ID з послідовностей PostgreSQL **не скидаємо**: після відкату можливі прогалини. Нові ID повинні залишатися більшими за всі вже видані ID, щоб порівняння lifecycle-полів з `E` було коректним.

FK/каскади не повинні непомітно очищати `created_event_id`, `cancelled_event_id`, `executed_event_id` до того, як rewind застосує власну логіку. Так само треба врахувати циклічний зв'язок `current_state_id` між таблицями екземплярів і `model_states`.

## 6. Що ще деталізувати перед Codex

Ці питання не змінюють затвердженої схеми, але їх треба реалізаційно визначити:

- SQL-міграції, правила FK, початкове створення екземплярів і контроль відповідності `current_state_id` його власнику.
- Надійне блокування/claim черг, recovery після падіння worker-а, retry/idempotency та транзакційна ізоляція.
- Відновлення технічного previous-state для state triggers, якщо цей стан зберігається окремо від уже описаних snapshot-ів.
- Узгодження проєкцій relationships при відкаті та видаленні екземплярів.

Інші правила Model, DFS, trigger checks, lazy relationships та EventContext залишаються в основних архітектурних документах.
