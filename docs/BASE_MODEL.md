# Base Model — контракт базового класу

Цей документ описує спільну інфраструктуру **базового класу `Model`**, від якого успадковуються всі доменні моделі. Він деталізує `docs/FEUDALS_BACKEND_ARCHITECTURE_CURRENT.md`, не замінюючи його. Базовий клас не містить правил конкретних ігрових сутностей.

## 1. Відповідальність і межі

Базовий `Model` відповідає за:
- ідентичність моделі, зв'язок з `EventContext` (із внутрішнім реєстром моделей) і завантаження persisted state;
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

Runtime-поля не є game characteristics і не потрапляють до serialized game state. Один instance `(model_type, id)` існує в межах однієї GameEvent завдяки реєстру всередині `EventContext`. Жоден mutable runtime state не переноситься між подіями.

## 3. Ініціалізація

`EventContext::get(type, id)` перевіряє власний реєстр `(model_type, id)`, створює та реєструє новий instance, після чого викликає `Model::initialize()`. Повторне звернення повертає той самий instance. Саме **Model виконує ініціалізацію**, `EventContext` керує її життєвим циклом. Об'єкт реєструється до `initialize()`, але до завершення ініціалізації не може використовуватися як повноцінна Model.

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

Getter повертає relationship-об'єкти через внутрішній реєстр `EventContext`, а вкладені `_class`-структури — як wrappers, прив'язані до parent Model і JSON path. Не можна видавати mutable arrays/references, що обходять контроль state.

Computed characteristic може читати характеристики безпосередньо пов'язаних моделей, але **не може явно переходити через другий relationship** (`$this->relation->other_relation->value`). **Очікуваний механізм** для такої залежності — звернення до обчислювальної характеристики безпосередньо пов'язаної моделі (`$this->relation->computed_value`): саме ця модель відповідає за подальші залежності та їх обчислення. Транзитивні залежності через computed characteristics не заборонені. Dynamic advance не читає жодних relationships. Ці обмеження стосуються формул характеристик, а не domain methods.

## 5. Relationships і lazy loading

Всі relationships оголошуються в **централізованій декларації зв'язків**. Для кожної характеристики декларація задає related Model type, cardinality (одиничний/колекція), спосіб отримання значення (прямий чи system-computed), напрямки графа/зворотний зв'язок та SQL metadata. **Немає правила**, що одиничні зв'язки обов'язково прямі, а колекції обов'язково system-computed: можливі всі чотири комбінації.

- Прямий relationship зберігає посилання/посилання у контрольованому state; system-computed relationship визначається інфраструктурою (наприклад, через reverse SQL lookup).
- Одиничний getter повертає Model або `null`; колекційний getter повертає `LazyCollection<Model>`. Об'єкти завантажуються через `EventContext::get(type, id)`, який використовує свій реєстр; повторної ініціалізації того самого instance немає.
- Для **кожного** оголошеного relationship доступний віртуальний accessor `<relationship_name>_id`: одиничний повертає ID/`null`, колекційний — масив ID. Наприклад `$army->player_id` та `$player->armies_id`. Суфікс завжди `_id` — не `_ids`, без спроб перетворювати назву колекції на однину.
- `_id` не ініціалізує пов'язані моделі. Для system-computed relationships getter ID виконує актуальний lookup.
- Звичайні явно оголошені scalar/array характеристики з назвами `player_id`, `blocking_camp_presence_player_ids`, `active_knight_replacement_id` тощо **можуть бути самостійними direct/computed ID-values без relationship**. Getter спочатку шукає явно оголошену характеристику, і лише за її відсутності розпізнає віртуальний `_id`. Потрібно уникати конфлікту імен між явно оголошеною характеристикою та віртуальним accessor іншої: така неоднозначність має виявлятися при перевірці декларацій.
- Старі окремі computed `player_id`, які тільки дублюють оголошений relationship `player`, замінюються віртуальним accessor, а не дублюють формулу.

**Актуальність колекцій:** кожне звернення до `$model->armies` повертає lazy collection із правилами пошуку, а не кешованим складом. На початку **кожної** ітерації `foreach ($model->armies as $army)` `LazyCollection::getIterator()` виконує новий lookup актуальних ID і фіксує їхній список на час цієї ітерації. Зміни зв'язків усередині циклу не змінюють список поточної ітерації; наступна ітерація бачить зміни. Елементи ініціалізуються через `EventContext` лише коли ітератор доходить до них. Кожне читання `$model->armies_id` так само отримує **актуальний** список ID без ініціалізації елементів. Це правило стосується **і прямих, і system-computed колекцій**: для прямих склад береться з актуального state, а SQL запит виконується тільки якщо джерело складу — SQL; не потрібно зайвого SQL для вже наявного прямого state. Жоден отриманий раніше `LazyCollection` не повинен назавжди заморожувати склад — його нова ітерація теж робить актуальний lookup.

**Запис:** setter прямого одиничного relationship приймає Model/`null` або, через `_id`, ID/`null`; обидва маршрути ведуть до одного внутрішнього setter. Для прямої колекції можливі `$model->units = [Model, ...]` і `$model->units_id = [id, ...]`; присвоєння **повністю замінює** склад колекції. Валідуються related type, структура і допустимість змін. System-computed relationships та їхні віртуальні `_id` **read-only**; їхній результат змінюється опосередковано через прямі зв'язки інших Model. Мутація колекції через `[]` або модифікація повернутого масиву не обходить setter; для зміни складу слід присвоювати нову колекцію через контрольований setter.

**SQL-проєкції:** при кожній зміні прямого relationship відповідні FK/junction projections синхронізуються **негайно всередині поточної транзакції GameEvent**, щоб наступний reverse SQL lookup уже бачив зміну. Domain state лишається authoritative, а projection — його відображення. Транзакція охоплює `event_action()`, DFS, trigger checks і фінальний commit; при помилці весь SQL запис відкочується, а інші транзакції не бачать незавершених змін. Якщо зв'язок не має FK/junction projection, зайвий запис SQL не потрібний.

DFS отримує ID сусідів із тієї самої централізованої декларації, після чого за потреби ініціалізує відповідні Model через `EventContext`. Окремого механізму завантаження для DFS немає.

## 6. Setter і контроль змін

Зміни direct characteristics і допустимі зміни dynamic state проходять через контрольований setter або wrapper, який делегує setter parent Model. Базовий клас:
- перевіряє declaration, тип і допустимість запису;
- позначає зміну для UnitOfWork / dirty tracking;
- при зміні direct relationship негайно оновлює відповідні SQL FK/junction projections у відкритій транзакції;
- не дозволяє записувати computed characteristics як звичайні direct values;
- не допускає прямого запису характеристик іншої Model: викликається її публічний domain method.

Базовий setter не запускає DFS, trigger actions або окремий commit. Всі зміни під час `event_action()` завершуються до фази DFS.

## 7. `external_signature`

Кожна конкретна Model визначає semantic `external_signature`: лише характеристики, від яких залежать **computed characteristics безпосередньо пов'язаних моделей**. Не включати автоматично весь state.

- `initial_external_signature` фіксується один раз під час initialization.
- Поточний `external_signature` обчислюється на момент перевірки за поточним state; після initialization computed getters завжди виконують актуальні формули.
- Signature не має окремого persisted hash та не кешується як computed value.

Базовий клас порівнює baseline і поточну signature. Конкретні моделі визначають її склад; базовий клас не вгадує залежності.

## 8. Універсальний DFS

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

## 9. Trigger checks

Після DFS worker перевіряє triggers **усіх initialized моделей**, включно з ініціалізованими під час DFS та trigger checks. Базовий клас може надати generic `check_all_triggers()`, який перебирає declarations і викликає `check_trigger_*` конкретної Model.

`check_trigger_*` нічого не змінює в game state. Результат: `false/null/-1` — немає прогнозу, `N>0` — перевірка через `N` одиниць Game Time, `N=0` — окрема перевірка на тому самому `Te`. Планування виконує infrastructure; `on_trigger_*` виконується тільки як `event_action()` окремої GameEvent.

## 10. Commit / persistence

Після DFS та trigger checks UnitOfWork завершує відкриту протягом GameEvent транзакцію атомарним commit. Базовий клас надає підготовку persisted representation своєї Model:

1. Поточні direct characteristics.
2. Dynamic characteristics, доведені до `Te`, і нову temporal anchor `T0 = Te`.
3. **Актуальні computed characteristics:** формули викликаються під час підготовки до збереження, їхні результати записуються як persisted values для наступної GameEvent. Це не кеш поточної події.
4. Typed references і вкладені структури в серіалізованому вигляді.
5. Перевірку узгодженості FK/junction projections, які вже актуалізувалися під час setter-ів прямих relationships; це не відкладання їхнього першого оновлення до commit.
6. History snapshot, якщо ввімкнено.

Збереження всіх змінених моделей, projection fields, scheduled trigger checks, статусу GameEvent і history snapshots — **одна узгоджена транзакція**. Не допускається частковий commit. Базова Model не виконує незалежний SQL commit усередині domain method.

## 11. Рекомендований інтерфейс (псевдокод)

Це перелік відповідальностей, а не жорстко зафіксовані PHP-сигнатури:

```text
abstract Model:
    id, context, state, lifecycle_phase
    initial_external_signature, dfs_visited

    initialize(persisted_state, Te)
    get_characteristic(name)
    set_characteristic(name, value)
    get_relationship(name)         # Model/null або LazyCollection
    get_relationship_id(name)      # ID/null або ID[] без завантаження моделей
    set_direct_relationship(name, value_or_ids)
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

## 12. Перевірки для реалізації

Мінімальні тести базового класу:
- один instance та один temporal advance для повторного завантаження тієї самої Model в EventContext;
- computed getter повертає persisted value під час initialization і **нове обчислене** після initialization;
- після двох послідовних змін залежності два читання computed getter повертають відповідні актуальні значення без кешу;
- dynamic advance використовує лише власні збережені параметри, не читає relationships;
- `initial_external_signature` незмінна після `event_action()`;
- DFS обходить залежні Model лише за зміни signature, не змінює їхній game state та не викликає actions;
- тригери, створені при `N=0`, не виконуються у поточній GameEvent;
- після commit persisted computed values доступні наступній GameEvent як saved parameters;
- setter direct relationship доступний через Model та через `_id`, із негайним SQL projection update;
- reverse lookup у тій самій GameEvent бачить нові FK/junction, а rollback скасовує їх;
- колекція після повторного звернення або нового `foreach` відображає актуальний склад, але поточний `foreach` обходить фіксований ID snapshot;
- `_id` повертає ID/ID[] без ініціалізації пов'язаних Model, а явно оголошені ID-характеристики не плутаються із relationships;
- system-computed relationship read-only; direct collection setter замінює всю колекцію;
- помилка в будь-якій фазі до commit не залишає частково записаних моделей чи scheduled checks.

Поза цим документом залишаються точний SQL DDL, формат relationship declarations, locking/retry, arbitration черги та бізнес-правила конкретних моделей.
