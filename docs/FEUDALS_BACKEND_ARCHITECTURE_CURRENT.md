# Феодали — актуальна backend-архітектура

Цей документ зводить в один актуальний опис прийняті рішення щодо реалізації backend першого прототипу. Він замінює попередні часткові архітектурні конспекти.

Документ не містить конкретного переліку моделей, їх характеристик, зв'язків і trigger-ів, а також не описує UI та систему користувацьких сповіщень.

## 1. Стек і загальний напрямок

Перший прототип backend реалізується на PHP. Основна причина — швидкість розробки й можливість легко змінювати доменну логіку під час активного проектування гри.

Backend будується навколо:
- довгоживучого CLI Game Worker;
- доменних Model з основною ігровою логікою;
- SQL persistence;
- окремого API/realtime шару;
- єдиного Game Time;
- послідовної обробки GameEvent.

Game Worker і realtime/WebSocket service є окремими довгоживучими процесами.

Архітектура першого прототипу свідомо оптимізується насамперед під коректність, прозорість поведінки, зручність debugging і простоту зміни правил. Передчасна оптимізація кількості завантажених моделей, SQL-запитів або cascade-викликів не є пріоритетом.

## 2. Основний принцип доменної архітектури

Основна ігрова логіка знаходиться в Model.

Model не обов'язково відповідає окремому видимому об'єкту гри. Модель може представляти:
- сутність ігрового світу;
- довготривалий процес;
- взаємодію кількох сутностей;
- persistent state конкретного відношення між двома або кількома сутностями.

Якщо певне відношення між моделями має власний persistent state, progress або lifecycle, для нього може створюватися окрема Model.

Якщо значення є лише похідним фактом поточного стану інших моделей, окрема relation-model не потрібна.

Не кожне поняття gameplay зобов'язане мати окрему backend Model. І навпаки, backend Model може не мати окремого gameplay-об'єкта.

## 3. Відповідальність Model

Model:
- зберігає власний game state;
- реалізує свої game rules;
- реалізує actions;
- може читати безпосередньо пов'язані Model;
- може викликати публічні game-logic methods інших Model;
- не змінює характеристики іншої Model прямим записом;
- описує або надає свої triggers;
- бере участь у DFS через спільну логіку базового класу;
- commit-ить свій persistent state.

Правило міжмодельної зміни: характеристика іншої Model не змінюється прямим присвоєнням. Потрібно викликати domain method самої цієї Model. Таким чином Model сама контролює інваріанти власного state.

Не вся залежність світу повинна виражатися через computed characteristics і triggers. Структурні зміни, для яких природно виконати явний перерахунок у момент конкретної події, можуть оброблятися прямо в `event_action()` / domain action.

Наприклад, складна графова властивість може зберігатися як пряма характеристика та централізовано перераховуватись відповідною моделлю при подіях, що можуть її змінити.

## 4. GameEvent

GameEvent — одна логічно миттєва зміна світу.

Одна ітерація Game Worker обробляє рівно одну GameEvent.

Кожна GameEvent має фіксований Game Time `Te`.

У межах однієї GameEvent:
- Game Time не змінюється;
- всі Model працюють з одним і тим самим `Te`;
- одна GameEvent може змінити багато Model;
- реальний час виконання PHP-коду не впливає на ігровий час;
- trigger-actions інших trigger-ів не виконуються каскадно в цій же GameEvent.

`event_action()` — кореневий game-logic method конкретної GameEvent. Після завершення `event_action()` game state цієї події вже вважається сформованим. Подальший DFS не є "стабілізацією світу"; він розповсюджує необхідність актуалізації на пов'язані Model.

## 5. EventContext

Кожна GameEvent має власний EventContext / UnitOfWork, який живе лише одну ітерацію worker-а.

Контекст щонайменше містить:
- `Te`;
- IdentityMap;
- список initialized Model;
- список dirty / changed Model;
- службовий стан DFS;
- службові дані, потрібні для commit;
- ліміти/діагностику ланцюга Actions.

IdentityMap:

`(model_type, model_id) -> один instance Model`

Один і той самий instance в межах GameEvent ініціалізується лише один раз.

Стан EventContext не переноситься як mutable state між різними GameEvent довгоживучого worker-а.

## 6. Типи характеристик

### 6.1. Пряма характеристика

Пряма характеристика — збережений game state, який не виводиться автоматично з інших характеристик.

Вона змінюється через game actions / domain methods.

### 6.2. Обчислювальна характеристика

Обчислювальна характеристика логічно визначається з інших характеристик та дозволених relationships.

В актуальній схемі всі обчислювальні характеристики зберігаються при `commit()`. Окремого типу "materialized computed" більше немає: materialization є стандартною властивістю всіх computed characteristics.

Обчислювальна характеристика може використовувати:
- прямі характеристики своєї Model;
- dynamic characteristics своєї Model;
- інші computed characteristics своєї Model;
- характеристики безпосередньо пов'язаних Model.

Архітектурне обмеження: обчислювальна характеристика Model A не повинна переходити через relationship пов'язаної Model B до третьої Model C.

Дозволено: `A.computed -> B.characteristic`.

Не дозволено: `A.computed -> B.related_C -> C.characteristic`.

Якщо A потрібна інформація, що походить від C через B, B повинна надати відповідну власну computed characteristic, і A читає її з B.

Це правило потрібне для коректного propagation через graph relationships і `external_signature`.

Поки це розглядається як архітектурне правило. Жорсткий runtime-control через getter не є обов'язковим; його можна додати пізніше, якщо порушення правила стане практичною проблемою.

### 6.3. Динамічна характеристика

Dynamic characteristic безперервно змінюється з Game Time між GameEvent.

Загальна форма:

`X[Te] = F(X[T0], Te - T0, own saved direct characteristics, own saved computed characteristics)`

Під час advance dynamic characteristic не читає relationships і не звертається до інших Model.

Всі значення, які вона використовує для інтервалу `T0 -> Te`, були зафіксовані на попередньому `commit()`.

Якщо з dynamic characteristic пов'язаний trigger, бажано спростити закон зміни до одного effective parameter `P`:

`X[Te] = F(X[T0], dt, P)`

а де можливо:

`X[Te] = X[T0] + P * dt`

Це рекомендація, а не жорстке обмеження. Якщо один `P` принципово недостатній, дозволені кілька збережених параметрів.

Умовну логіку, яку можна винести з `F`, бажано виносити в computed parameter(s), щоб `F` була простою, момент trigger-а було легко прогнозувати, а часову поведінку — легко перевіряти.

## 7. Межі dynamic characteristics

Dynamic calculation не повинна самостійно непомітно проходити через точку, в якій змінюються правила її подальшої динаміки.

Такі точки забезпечуються trigger-механізмом.

Типовий принцип:
- до порога діє попередній saved rate/режим;
- у точці порога виникає окрема GameEvent;
- після цієї точки computed characteristics зберігають новий режим;
- наступний часовий інтервал рахується вже за ним.

Тому clamp до 0/capacity або подібна умовна логіка за можливості задається через rate/parameter, а не ускладнює саму `F`.

## 8. Ініціалізація Model

При першому використанні Model в конкретній GameEvent:

1. Model завантажує persistent state.
2. Завантажуються збережені прямі, dynamic та computed characteristics попереднього commit.
3. Dynamic characteristics один раз доводяться від `T0` до `Te`, використовуючи тільки збережені власні характеристики.
4. До будь-яких змін, викликаних поточною GameEvent, Model фіксує локальний baseline, зокрема `initial_external_signature`.
5. Збережені computed values після initialization більше не вважаються автоматично актуальними для нового state `Te`; при потребі актуальне computed value перераховується за поточним state і дозволеними direct relationships.
6. Повторне temporal advance цієї Model у тій самій GameEvent не виконується.

Ключова вимога: baseline для поточної GameEvent повинен бути сформований до domain changes цієї GameEvent для даного instance.

## 9. Зв'язки між Model

Domain relationship — логічний зв'язок між Model, а не SQL foreign key.

Зв'язок може бути:
- двостороннім;
- одностороннім.

Напрямок означає, чи повинна зміна однієї Model розглядатися як потенційно значуща для іншої при propagation.

Domain relationship і фізичне представлення в SQL — різні рівні.

## 10. Централізований опис relationships

Relationships не повинні бути розкидані по окремих реалізаціях DFS конкретних Model.

План:
- існує один централізований declaration/config relationships;
- він описує graph domain relationships;
- базовий `Model` має універсальний method, який за declaration повертає суміжні Model для поточного instance;
- `dfs()` реалізовано в базовому Model;
- конкретні Model не дублюють generic DFS logic.

Declaration може також містити persistence metadata:
- related model type;
- direction;
- inverse relationship;
- JSON path typed reference;
- чи потрібна SQL FK projection;
- як отримати id для FK;
- junction mapping, якщо потрібно.

## 11. Зберігання state всередині Model

Game characteristics зберігаються в контрольованому внутрішньому state, а не як довільні mutable PHP properties.

Для простих top-level values може використовуватись magic access, але реальна операція проходить через базову getter/setter infrastructure.

Це потрібно для:
- dirty tracking;
- validation;
- type handling;
- dynamic/computed lifecycle;
- централізованого контролю state mutations.

Технічні поля (`id`, context, caches, flags тощо) не є game characteristics і можуть бути звичайними PHP properties.

## 12. Вкладені структури characteristics

Для складних вкладених структур використовуються helper/wrapper classes.

У serialized state структура може містити службовий ключ `_class`.

Generic getter, побачивши `_class`, повертає відповідний wrapper, прив'язаний до parent Model і JSON path.

Wrapper:
- не є Model;
- не має власного lifecycle;
- не має окремого persistence;
- не має незалежної копії state;
- читає і змінює дані тільки через parent Model.

Назовні не видаються mutable array references, через які можна обійти контрольований Model state API.

Ключі з `_` зарезервовані для infrastructure metadata.

## 13. Typed references

Посилання на Model у JSON state повинно містити не лише id, а й model type.

Конкретний serialization format ще може бути обраний, наприклад:

`{"Castle": 12}`

або:

`{"model_type": "Castle", "id": 12}`

Мета — generic loader повинен однозначно знати, Model якого класу завантажувати.

## 14. Persistence у SQL

Для кожного persistent Model type передбачається окрема основна SQL table.

Основна таблиця містить:
- `id`;
- службові fields;
- JSON/state field;
- FK projection fields, необхідні для ефективного reverse lookup relationships.

Основна частина game characteristics може зберігатися у JSON state.

Треба розрізняти:
1. domain state Model;
2. SQL projection relationships;
3. history snapshots.

FK/junction fields — не незалежне джерело game state. Вони є persistence projection domain relationship.

Під час `commit()` infrastructure синхронізує projection fields зі state/relationship values.

Для relationship, де reverse lookup не потрібний, typed reference може залишатися лише в JSON без окремого SQL FK.

Для many-to-many та аналогічних випадків використовуються junction tables.

## 15. Commit

`commit()` виконується наприкінці GameEvent після DFS і trigger checks.

Commit повинен:
- зберегти змінений direct state;
- зберегти актуальні dynamic characteristics;
- перерахувати та зберегти актуальні computed characteristics;
- синхронізувати FK/junction projections relationships;
- записати history snapshot, якщо history mode увімкнений;
- зафіксувати службові записи GameEvent/scheduled events в одній узгодженій операції.

Computed characteristics є частиною persisted state після commit і використовуються як зафіксовані параметри наступного часового інтервалу.

Точні DB transaction boundaries та retry/idempotency strategy можуть бути деталізовані вже у ТЗ реалізації, але GameEvent не повинна залишати частково застосований game state.

## 16. `external_signature`

`external_signature` — semantic representation тільки тієї частини Model state, зміна якої може змінити computed characteristics інших безпосередньо пов'язаних Model.

У `external_signature` не входить автоматично:
- весь state Model;
- все, що інша Model може коли-небудь прочитати в action;
- всі характеристики, цікаві frontend.

Критерій включення:

якщо інша Model використовує характеристику M у своїй computed characteristic, зміна цієї характеристики повинна відобразитися в `M.external_signature`.

Signature може складатися як із raw values, так і з semantic aggregates.

`external_signature` не зберігається в БД окремим persisted hash.

Для кожного instance під час initialization фіксується `initial_external_signature`.

Пізніше DFS порівнює його з поточним `external_signature`.

## 17. DFS

Після `event_action()` worker робить snapshot Model, які вже initialized на цей момент.

Це фіксований список roots. Model, які будуть initialized вже під час DFS, до root snapshot не додаються.

Worker:

```text
roots = snapshot(initialized_models)
for M in roots:
    M.dfs()
```

`dfs()` реалізований у базовому Model.

Алгоритм:

```text
dfs(M):
    if M.dfs_visited:
        return

    M.dfs_visited = true

    if M.external_signature == M.initial_external_signature:
        return

    for U in domain_related_models(M):
        initialize(U) if needed
        U.dfs()
```

`dfs_visited` — окремий event-local flag.

IdentityMap відповідає за single initialization, а `dfs_visited` — за graph traversal. Це різні задачі.

DFS може ініціалізувати додаткові Model. Їх dynamic state доводиться до того ж `Te`.

DFS не виконує trigger actions і не є другою фазою domain stabilization.

## 18. Пряма domain logic замість DFS/computed, коли це природно

Не всі залежності повинні автоматично "випливати" з graph cascade.

Якщо зміна має складну структурну семантику, її можна явно обробити в domain action.

Типова схема:

```text
Model A змінила структурний state
-> викликає domain method Model B
-> Model B перераховує/оновлює свої прямі характеристики
-> ці зміни далі вже видимі через звичайний external_signature/DFS
```

Особливо це корисно для graph/connectivity calculations, масового оновлення набору пов'язаних сутностей і state, який простіше і надійніше зберігати явно, ніж рекурсивно виводити з computed chain.

## 19. Worker: одна ітерація

Актуальний pipeline:

1. Вибрати наступну GameEvent.
2. Зафіксувати її `Te`.
3. Створити EventContext / IdentityMap.
4. Ініціалізувати root Model на `Te`.
5. Виконати відповідний `event_action()`.
6. Зробити snapshot поточного `initialized_models`.
7. Для кожної Model зі snapshot викликати `dfs()`.
8. Після DFS взяти новий повний `initialized_models`.
9. Для кожної initialized Model перевірити її triggers і створити/оновити потрібні technical scheduled events.
10. Виконати commit.
11. Завершити GameEvent.

Якщо під час trigger check ліниво ініціалізується додаткова Model лише для читання, вона також повинна отримати trigger check у цій фазі. Сам факт read-only initialization не запускає новий DFS.

Worker не містить game business logic. Він лише вибирає подію, задає `Te`, створює context, завантажує root Model, викликає один entry point, запускає generic post-action pipeline і commit-ить результат.

## 20. Детермінований порядок GameEvent

GameEvent обробляються послідовно.

Якщо кілька подій мають однаковий `Te`, порядок повинен бути детермінованим, наприклад:

`ORDER BY game_time, sequence/id`

Наступна GameEvent з тим самим `Te` бачить уже committed result попередньої.

Це особливо важливо для trigger events з `N = 0`.

## 21. Actions і naming methods

Action — логічно значущий domain method, який може змінити game state.

Не кожен private/helper setter є окремою Action.

Поточні prefixes:

### `api_{name}()`

Безпосередня точка входу з API.

Не кожен `api_*` зобов'язаний бути GameEvent: чисто описові зміни, які не впливають на gameplay, можуть оброблятися синхронно.

### `user_{name}()`

Gameplay action користувача, що виконується через worker як GameEvent.

Колишній окремий prefix `async_user_*` не використовується.

### `check_trigger_{name}()`

Перевірка та прогноз trigger-а. Не змінює game state.

### `on_trigger_{name}()`

Game action при фактичному спрацюванні trigger-а. Виконується лише в окремій GameEvent trigger-а.

### `on_timer_{name}()`

Обробник звичайної scheduled system event, яка не є trigger-check event.

Міжмодельні game methods окремого обов'язкового prefix не мають.

## 22. Trigger: призначення

Trigger — не synonym для "якщо умова true, встановити flag".

Головні задачі trigger engine:
- прогнозувати важливу часову межу;
- не дозволити dynamic calculation перескочити цю межу;
- створити окрему GameEvent точно у прогнозований момент;
- за потреби виконати domain action в окремій GameEvent.

Актуальний game state визначається characteristics/domain logic, а не stored flags, які trigger вручну перемикає.

## 23. Види trigger

Поки підтримуються два види.

### 23.1. `[однонаправлений]`

Спрацьовує при виконанні умови без необхідності відстежувати її попередній boolean state.

### 23.2. `[двонаправлений]`

Trigger engine пам'ятає попередній boolean state умови та реагує на:
- `false -> true`;
- `true -> false`.

Для обох переходів використовується один `on_trigger_{name}()`.

Якщо поведінка залежить від напрямку, method визначає актуальний state/transition.

Technical previous-state trigger-а не є domain characteristic Model.

## 24. Семантика `check_trigger_*()`

`check_trigger_*()` не виконує Action.

Він повертає прогноз:
- `false`, `null` або `-1` — trigger не прогнозується;
- `N > 0` — trigger очікується через `N` одиниць Game Time;
- `N = 0` — trigger уже повинен бути перевірений/спрацьовувати на поточному `Te`.

`N` — відносний час від поточного `Te`.

Навіть `N = 0` не запускає `on_trigger_*()` у поточній GameEvent.

Замість цього створюється technical scheduled event на той самий `Te`, яка буде окремою наступною ітерацією worker-а.

## 25. Trigger scheduled event

Scheduled record trigger-а означає: у цей Game Time створити GameEvent і повторно перевірити trigger.

Він не означає: безумовно виконати trigger action.

Тому якщо до прогнозованого моменту умови змінилися, scheduled event не треба обов'язково видаляти. При її обробці `check_trigger_*()` може показати, що action більше не актуальна.

Це дозволяє обійтися без складного механізму скасування trigger timers.

Усі trigger-и поки працюють однаково. Поділу на "активні" й "пасивні" немає.

## 26. Trigger action як окрема GameEvent

Схема:

1. GameEvent A змінює світ.
2. Після DFS `check_trigger_X()` повертає `0`.
3. Створюється scheduled event X на `Te`.
4. GameEvent A commit-иться.
5. Worker окремою наступною ітерацією бере X.
6. Model ініціалізується на тому ж `Te`.
7. Trigger повторно перевіряється.
8. Якщо trigger актуальний — виконується `on_trigger_X()`.
9. Далі ця GameEvent проходить звичайний pipeline DFS -> trigger checks -> commit.

Це прибирає необхідність другого trigger cascade в одній worker iteration і значно зменшує залежність результату від порядку одночасно готових trigger-actions.

## 27. Gameplay timer і technical scheduled event

Треба чітко розділяти два поняття.

У game rules "таймер" може означати просто тривалість ігрового процесу для гравця.

У backend technical scheduler — інфраструктурний механізм, який створює майбутню GameEvent або wake-up/check.

Це не обов'язково одна і та сама сутність.

У реалізації бажано використовувати окрему назву на кшталт `ScheduledEvent`, `ScheduledAction` або `TimeoutEvent`, щоб не змішувати її з gameplay-поняттям "таймер". Точну назву можна зафіксувати в ТЗ.

## 28. Довготривалі процеси

Якщо майбутній момент завершення процесу може змінитися, pause/resume або стати неактуальним, бажаний патерн:
- dynamic progress;
- computed current rate;
- trigger на boundary/completion;
- technical scheduled event лише як wake-up для trigger check.

Якщо подія справді безумовна після її постановки, допускається звичайний `on_timer_*` / scheduled system event.

## 29. Cycle protection

DFS не зациклюється завдяки `dfs_visited`.

Triggers не виконують cascade-actions у межах тієї ж GameEvent.

Основний ризик рекурсивного циклу лишається в domain Actions:

`A.action() -> B.action() -> A.action() -> ...`

EventContext повинен мати аварійний guard:
- max action calls per GameEvent;
- за потреби max nested depth;
- diagnostic call chain.

При перевищенні guard GameEvent повинна бути aborted без partial commit, а diagnostic information — записана.

## 30. Game Time

Game Time зберігається як integer у мікросекундах.

Real time для sync — integer у мілісекундах.

Таблиця синхронізацій містить приблизно:
- `real_time`;
- `game_time`;
- `speed`.

`speed` — скільки мікросекунд Game Time проходить за одну мілісекунду real time.

Приклади:
- `1000` -> x1;
- `500` -> x0.5;
- `10000` -> x10;
- `0` -> pause.

Поточний Game Time:

`game_time_now = last_sync.game_time + (real_now - last_sync.real_time) * last_sync.speed`

Зміна speed створює нову sync point.

Перемотка Game Time вперед також створює нову sync point з більшим `game_time`.

Normal GameClock не рухається назад.

## 31. Історія станів

Для debugging першого прототипу передбачається можливість зберігати повну історію committed state Model.

StateSnapshot містить щонайменше:
- `game_event_id`;
- `game_time`;
- `model_type`;
- `model_id`;
- повний serialized game state.

Snapshot не містить runtime caches, EventContext, службові transient objects та інші process-local поля.

Оскільки кілька GameEvent можуть мати однаковий `game_time`, одного timestamp недостатньо; `game_event_id`/sequence є обов'язковим для точного порядку.

Історія лінійна. Паралельних history branches немає.

## 32. Debug rewind

Debug rewind — окремий інструмент поверх history, а не звичайна операція GameClock.

При rewind недостатньо відновити лише JSON states Model.

Необхідно узгоджено відновити весь persistent game state, зокрема:
- states Model;
- lifecycle created/deleted Model;
- active scheduled events;
- черги gameplay actions, якщо вони persistent;
- persistent process/interaction state;
- Game Time sync state;
- RNG state/seed, якщо потрібне точне replay.

Потрібно підтримувати відновлення видалених/створених entity через lifecycle/tombstone semantics.

Точна техніка поводження з history після точки rewind може бути визначена в ТЗ: видалити майбутнє або позначити його неактуальним. Але нові паралельні history branches не створюються.

## 33. Replay і randomness

Для точного debugging/replay випадкові результати повинні бути відтворюваними.

Для GameEvent, де використовується randomness, треба зберігати достатній deterministic context, наприклад RNG seed/state або фактично використані random values.

При однаковому вході й однаковому RNG context replay повинен давати той самий game result.

## 34. Persistence consistency і transaction

Одна GameEvent — логічна атомарна операція.

У разі exception або action-cycle guard failure:
- не можна залишати partial model state;
- не можна частково commit-ити тільки частину взаємопов'язаних змін;
- scheduled records, snapshots і model states повинні лишитися узгодженими.

Точна SQL transaction strategy, lock strategy, retry та idempotency визначаються в ТЗ реалізації.

## 35. User actions і черга worker

Gameplay user action проходить через worker і стає GameEvent.

API method `api_*` відповідає за зовнішню точку входу, validation/auth/request handling і постановку gameplay request на виконання.

`user_*` — domain entry point самої GameEvent.

Точний arbitration між user request queue і system scheduled events, а також точний момент присвоєння `Te` user request можна визначити при реалізації worker queue, але всі GameEvent у підсумку мають єдиний детермінований порядок.

## 36. Realtime / API — тільки архітектурна межа

Моделі не повинні знати:
- хто зараз підключений;
- що відкрито у frontend;
- як виглядає transport message;
- як працює WebSocket/HTTP layer.

Gameplay state змінюється в domain layer, а зовнішня доставка state — окрема infrastructure responsibility.

Конкретний frontend protocol, subscriptions, frontend representation і UI в цьому документі не визначаються.

## 37. Конфігурація game parameters

Числові gameplay constants та функції балансу не повинні бути hardcoded у domain methods.

Вони зберігаються в окремих JSON configurations:
- starting parameters;
- global game/balance parameters.

Для level/distance-dependent values використовується декларативний rule format: table / function / piecewise ranges.

Domain code читає параметри конфігурації, але сама структура event processing, relationships, triggers, Game Time та persistence не залежить від конкретних балансних чисел.

## 38. Підсумковий pipeline

```text
Worker:
    event = get_next_event()
    Te = event.game_time

    context = new EventContext(Te)

    root = context.load(event.model)
    root.event_action(event.payload)

    roots = snapshot(context.initialized_models)

    for M in roots:
        M.dfs()

    checked = set()
    while exists initialized Model not in checked:
        M = next_unchecked_initialized_model()
        M.check_all_triggers()
        checked.add(M)

    context.commit()
```

Initialization:

```text
load persisted state
advance dynamic characteristics to Te
    using only own saved direct/computed values
capture initial_external_signature
mark computed cache/state for current-Te recalculation as needed
```

DFS:

```text
if dfs_visited:
    return

dfs_visited = true

if external_signature == initial_external_signature:
    return

for related_model in relationship_graph.neighbors(this):
    initialize related_model if needed
    related_model.dfs()
```

Trigger check:

```text
false/null/-1 -> no predicted event
N > 0         -> schedule check at Te + N
N = 0         -> schedule separate check GameEvent at Te
```

Commit:

```text
persist direct state
persist dynamic state
recompute + persist computed state
sync relationship projections
persist scheduler/GameEvent changes
write history snapshot where enabled
commit atomically
```

## 39. Ключові інваріанти

1. Одна worker iteration = одна GameEvent.
2. Всі Model однієї GameEvent використовують один `Te`.
3. Model temporal state advance виконується максимум один раз за GameEvent.
4. Dynamic characteristic не читає relationships.
5. Computed characteristic не переходить через relationship пов'язаної Model до третьої Model.
6. Всі computed characteristics зберігаються при commit.
7. Характеристики Model змінюються тільки methods самої Model.
8. `external_signature` містить тільки міжмодельні computed dependencies.
9. `initialized` і `dfs_visited` — різні стани.
10. DFS не виконує trigger actions.
11. Trigger action завжди є окремою GameEvent.
12. `check_trigger() == 0` означає окрему GameEvent на тому самому `Te`.
13. Worker не містить game business logic.
14. SQL FK/junction — projection domain relationships, а не друге джерело game state.
15. GameEvent повинна commit-итися атомарно або не commit-итися взагалі.
16. Складні структурні залежності дозволено явно оновлювати через domain actions, а не насильно виражати через computed/trigger cascade.
