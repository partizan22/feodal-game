# Base Model — контракт базового класу

Цей документ описує спільну інфраструктуру **базового класу `Model`**, від якого успадковуються всі доменні моделі. Він деталізує `docs/FEUDALS_BACKEND_ARCHITECTURE_CURRENT.md`, не замінюючи його. Базовий клас не містить правил конкретних ігрових сутностей.

## 1. Відповідальність і межі

Базовий `Model` відповідає за:
- ідентичність моделі, зв'язок з `EventContext` / `IdentityMap` і завантаження persisted state;
- контрольований доступ до характеристик (getter/setter, типи, вкладені структури);
- одноразове просування dynamic characteristics до часу події `Te`;
- розрізнення збережених та актуально обчислених computed characteristics;
- облік змін, `initial_external_signature`, універсальний DFS;
- підготовку та збереження state, relationship projections і snapshots через persistence infrastructure.

Конкретні класи визначають власні характеристики, формули computed/dynamic, `external_signature`, domain methods та `check_trigger_*` / `on_trigger_*`. Базовий клас не приймає ігрових рішень. Worker керує порядком фаз GameEvent; `Model` не обирає наступну подію та не виконує trigger actions під час DFS.

## 2. Runtime-поля базового класу

Мінімальний набір:
- `id`, `model_type`, посилання на `EventContext`;
- контрольований внутрішній `state` із direct, dynamic та persisted computed values;
- стан ініціалізації: `loading / advancing / initialized` (назви довільні, семантика обов'язкова);
- `initial_external_signature` — незмінний baseline поточної GameEvent;
- `dfs_visited` — прапорець тільки для поточної GameEvent;
- облік dirty state, створення/видалення, потрібних relationship projection changes.

Runtime-поля не є game characteristics і не потрапляють до serialized game state. Один instance `(model_type, id)` існує в межах однієї GameEvent завдяки `IdentityMap`. Жоден mutable runtime state не переноситься між подіями.

## 3. Ініціалізація

Ініціалізація запускається при першому зверненні до Model в EventContext і виконується **рівно один раз**:

1. Завантажити persisted state: direct characteristics, **збережені** computed characteristics, dynamic values на `T0`, службовий `T0` і необхідні references.
2. Перейти в режим initialization. У цьому режимі getter computed characteristic повертає **її збережене значення**, а не виконує формулу.
3. Довести всі dynamic characteristics від `T0` до `Te` за їхнім законом зміни, використовуючи лише власні збережені direct/computed parameters та dynamic state. Жодних звернень до пов'язаних моделей. Повторний advance у цій GameEvent заборонений.
4. Зафіксувати `initial_external_signature` **до перемикання getter-ів у звичайний режим**, відповідно до baseline, визначеного архітектурою. Значення signature повинне відображати стан моделі після temporal advance, але до змін `event_action()`; при цьому computed getters усе ще повертають persisted computed values.
5. Завершити initialization: перемкнути getter computed characteristics у звичайний режим. З цього моменту їхні збережені значення **не використовуються при читанні**. Жодного computed cache немає.

Якщо під час initialization потрібно читати computed characteristic, використовується її persisted value; якщо вона відсутня для нової Model, створення повинне надати визначені початкові значення до використання dynamic advance. Не підміняти відсутні значення довільними нулями.

## 4. Getter characteristics

Універсальний getter визначає категорію характеристики за declaration конкретної Model:

- **Direct:** повертає поточне значення з контрольованого state.
- **Dynamic:** повертає вже доведене до `Te` поточне значення; getter самостійно не запускає повторний temporal advance.
- **Computed, під час initialization:** повертає persisted computed value.
- **Computed, після initialization:** **щоразу** виконує формулу за поточним state та дозволеними relationships. Не читає persisted computed value як поточне, не кешує результат.

Перемикання computed getter автоматичне, за фазою lifecycle базового класу. Domain methods не повинні вручну вибирати «старе» чи «нове» computed value.

Getter повертає typed references як Model instances через EventContext / IdentityMap, а вкладені `_class`-структури — як wrappers, прив'язані до parent Model і JSON path. Не можна видавати mutable arrays/references, що обходять контроль state.

Computed characteristic може читати характеристики безпосередньо пов'язаних моделей, але не переходити через зв'язок цієї моделі до третьої (`A -> B -> C`). Dynamic advance не читає жодних relationships. Ці обмеження стосуються формул характеристик, а не domain methods.

## 5. Setter і контроль змін

Зміни direct characteristics і допустимі зміни dynamic state проходять через контрольований setter або wrapper, який делегує setter parent Model. Базовий клас:
- перевіряє declaration, тип і допустимість запису;
- позначає зміну для UnitOfWork / dirty tracking;
- не дозволяє записувати computed characteristics як звичайні direct values;
- не допускає прямого запису характеристик іншої Model: викликається її публічний domain method.

Базовий setter не запускає DFS, trigger actions або окремий commit. Всі зміни під час `event_action()` завершуються до фази DFS.

## 6. `external_signature`

Кожна конкретна Model визначає semantic `external_signature`: лише характеристики, від яких залежать **computed characteristics безпосередньо пов'язаних моделей**. Не включати автоматично весь state.

- `initial_external_signature` фіксується один раз під час initialization.
- Поточний `external_signature` обчислюється на момент перевірки за поточним state; після initialization computed getters завжди виконують актуальні формули.
- Signature не має окремого persisted hash та не кешується як computed value.

Базовий клас порівнює baseline і поточну signature. Конкретні моделі визначають її склад; базовий клас не вгадує залежності.

## 7. Універсальний DFS

DFS запускає worker **після завершення `event_action()`**, для snapshot усіх моделей, ініціалізованих до початку DFS.

```text
dfs(M):
    if M.dfs_visited:
        return
    M.dfs_visited = true
    if M.external_signature() == M.initial_external_signature:
        return
    for U in relationship_graph.neighbors(M):
        context.initialize(U)   # лише якщо ще не initialized
        U.dfs()
```

- `dfs_visited` живе тільки одну GameEvent; це не те саме, що initialized.
- Сусідів визначає централізований relationship declaration, а не індивідуальний код обходу кожної Model.
- DFS **не змінює ігровий state, не виконує actions, не перераховує та не зберігає computed values, не запускає triggers**. Він лише проходить граф для актуалізації залучених моделей.
- Нові моделі, ініціалізовані під час DFS, не додаються до початкового списку roots, але беруть участь у рекурсивному обході.
- Порядок обходу не повинен впливати на значення характеристик: computed getters не мають side effects або кешу.

## 8. Trigger checks

Після DFS worker перевіряє triggers **усіх initialized моделей**, включно з ініціалізованими під час DFS та trigger checks. Базовий клас може надати generic `check_all_triggers()`, який перебирає declarations і викликає `check_trigger_*` конкретної Model.

`check_trigger_*` нічого не змінює в game state. Результат: `false/null/-1` — немає прогнозу, `N>0` — перевірка через `N` одиниць Game Time, `N=0` — окрема перевірка на тому самому `Te`. Планування виконує infrastructure; `on_trigger_*` виконується тільки як `event_action()` окремої GameEvent.

## 9. Commit / persistence

Після DFS та trigger checks UnitOfWork здійснює атомарний commit. Базовий клас надає підготовку persisted representation своєї Model:

1. Поточні direct characteristics.
2. Dynamic characteristics, доведені до `Te`, і нову temporal anchor `T0 = Te`.
3. **Актуальні computed characteristics:** формули викликаються під час підготовки до збереження, їхні результати записуються як persisted values для наступної GameEvent. Це не кеш поточної події.
4. Typed references і вкладені структури в серіалізованому вигляді.
5. Дані для FK/junction projections, узгоджені з domain relationships.
6. History snapshot, якщо ввімкнено.

Збереження всіх змінених моделей, projection fields, scheduled trigger checks, статусу GameEvent і history snapshots — **одна узгоджена транзакція**. Не допускається частковий commit. Базова Model не виконує незалежний SQL commit усередині domain method.

## 10. Рекомендований інтерфейс (псевдокод)

Це перелік відповідальностей, а не жорстко зафіксовані PHP-сигнатури:

```text
abstract Model:
    id, context, state, lifecycle_phase
    initial_external_signature, dfs_visited

    initialize(persisted_state, Te)
    get_characteristic(name)
    set_characteristic(name, value)
    get_relationship(name)
    external_signature()          # визначає нащадок
    domain_related_models()       # за централізованим declaration
    dfs()                         # спільна реалізація
    check_all_triggers()          # спільний dispatcher
    serialize_for_commit(Te)
    mark_dirty(path)

ConcreteModel extends Model:
    declare_characteristics()
    calculate_computed(name)
    calculate_dynamic(name, T0, Te, saved_parameters)
    external_signature()
    user_*/domain methods
    check_trigger_*/on_trigger_*
```

## 11. Перевірки для реалізації

Мінімальні тести базового класу:
- один instance та один temporal advance для повторного завантаження тієї самої Model в EventContext;
- computed getter повертає persisted value під час initialization і **нове обчислене** після initialization;
- після двох послідовних змін залежності два читання computed getter повертають відповідні актуальні значення без кешу;
- dynamic advance використовує лише власні збережені параметри, не читає relationships;
- `initial_external_signature` незмінна після `event_action()`;
- DFS обходить залежні Model лише за зміни signature, не змінює їхній game state та не викликає actions;
- тригери, створені при `N=0`, не виконуються у поточній GameEvent;
- після commit persisted computed values доступні наступній GameEvent як saved parameters;
- помилка в будь-якій фазі до commit не залишає частково записаних моделей чи scheduled checks.

Поза цим документом залишаються точний SQL DDL, формат relationship declarations, locking/retry, arbitration черги та бізнес-правила конкретних моделей.
