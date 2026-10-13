# Зв'язки моделей — інвентаризація V1

Джерела: `MODELS_CORE.md`, `MODELS_MILITARY.md`, `MODELS_COMBAT.md`, `MODELS_PROCESSES.md` (стан на 2026-10-10).

Це інвентаризація **зв'язків, представлених характеристиками моделей**, з урахуванням узгоджених змін; не остаточна декларація SQL-схеми чи напрямків DFS. **У зв'язках беруть участь лише актуальні моделі.** Історичні записи зберігаються виключно для відкату світу, не потрапляють у пошуки зв'язків або DFS і не впливають на кратність. Кожен зв'язок описаний **один раз**, незалежно від напрямку навігації. Якщо між тією самою парою типів моделей є кілька незалежних зв'язків, вони наведені підряд. Зв'язок моделі із самою собою допустимий.

Позначення: **П** — пряма характеристика; **О** — обчислювальна характеристика зв'язку (у поточних специфікаціях часто позначена `*`); **—** — зворотна характеристика не оголошена. Типи `1:N`, `1:1`, `M:N` описують логічну кратність; `0..1` позначає необов'язкову одиничну сторону. Позначка **уточнити** означає, що специфікація не фіксує однозначно відповідний інваріант. П/О наведено за чинними описами моделей; майбутня централізована декларація може визначати прямий/system-computed тип незалежно від того, чи це одиничний зв'язок, чи колекція.

## Player ↔ Castle
- **Власник замку** — `1:N`. `Castle.player` (П, одинична) ↔ `Player.castles[]` (О, колекція).

## Player ↔ Region
- **Формальний власник області** — `1:N` (для Region `0..1`). `Region.player` (П) ↔ `Player.regions[]` (О).
- **Окупант області** — `1:N` (для Region `0..1`). `Region.occupier_player` (О, одинична, через `occupier_player_id` — П, scalar ID) ↔ —. `occupier_player_id` є збереженим ID, не другою навігаційною характеристикою.

## Player ↔ Army
- **Власник армії** — `1:N`. `Army.player` (П) ↔ `Player.armies[]` (О).

## Player ↔ CampInRegion
- **Власник табору** — `1:N`. `CampInRegion.player` (П) ↔ `Player.camps[]` (О, лише active Camp).

## Player ↔ CastleFounding
- **Гравець-засновник** — `1:N`. `CastleFounding.player` (П) ↔ —.

## Player ↔ Movement
- **Зафіксований цільовий суперник** — `1:N`, nullable. `Movement.target_opponent` (П) ↔ —.
- **Очікувана заміна цільового суперника** — `1:N`, nullable. `Movement.pending_target_opponent` (П) ↔ —. Відрізняється від попереднього зв'язку; наявність pending update визначає окремий прапорець.

## Player ↔ CombatSituation
- **Сторона захисту** — `1:N`, nullable до Start. `CombatSituation.defender_player` (П) ↔ —.
- **Зафіксований цільовий суперник** — `1:N`, nullable. `CombatSituation.target_opponent` (П) ↔ —.

## Castle ↔ Region
- **Приєднані області** — `1:N`, для Region `0..1`. `Region.castle` (П) ↔ `Castle.regions[]` (О).
- **Область розташування замку** — `1:1` для Castle, для Region `0..1`. `Castle.region` (П) ↔ —. Це **інший** зв'язок, ніж `Region.castle`; відповідність обох задається ігровим інваріантом.

## Castle ↔ Knight
- **Домашній замок лицаря** — `1:N`. `Knight.castle` (П) ↔ `Castle.knights[]` (О).
- **Замок фактичного розміщення** — `1:N`, для Knight `0..1`. `Knight.stationed_castle` (П) ↔ `Castle.stationed_knights[]` (О). Може не збігатися з домашнім замком.

## Castle ↔ Recruitment
- **Черга рекрутингу** — `1:0..1`, максимум один актуальний Recruitment на Castle. `Recruitment.castle` (П) ↔ `Castle.recruitment` (О, лише active, `0..1`).

## Castle ↔ BuildingUpgrade
- **Покращення будівель** — `1:N` (можливі кілька одночасних актуальних процесів). `BuildingUpgrade.castle` (П) ↔ `Castle.building_upgrades[]` (О, тільки актуальні).

## Castle ↔ KnightReplacement
- **Заміни лицарів** — `1:N`. `KnightReplacement.castle` (П) ↔ `Castle.knight_replacements[]` (О).

## Region ↔ Region
- **Сусідство на hex-мапі** — самозв'язок `M:N`, симетричний, не більше шести сусідів. `Region.neighbors[]` (О) ↔ `Region.neighbors[]` (О). Це один самозв'язок, а не два протилежні зв'язки.

## Region ↔ City
- **Місто в області** — `1:0..1`. `City.region` (П) ↔ `Region.city` (О).

## Region ↔ Army
- **Поточне фізичне місце армії** — `1:N`. `Army.current_region` (П) ↔ `Region.armies[]` (О).

## Region ↔ CampInRegion
- **Табір в області** — `1:N`. `CampInRegion.region` (П) ↔ `Region.camps[]` (О, лише active).

## Region ↔ CastleFounding
- **Заснування замку в області** — `1:0..1` (не більше одного актуального процесу заснування в Region). `CastleFounding.region` (П) ↔ `Region.castle_foundings[]` (О).

## Region ↔ ResourceSiteUpgrade
- **Покращення видобувних об'єктів** — `1:N` (якщо одночасно існує кілька актуальних процесів). `ResourceSiteUpgrade.region` (П) ↔ `Region.resource_site_upgrades[]` (О).

## Region ↔ CombatSituation
- **Бойові ситуації в області** — `1:N`. `CombatSituation.region` (П) ↔ `Region.combat_situations[]` (О, тільки актуальні `Registered` та `Active`; історія не входить).
- **Поточна активна бойова ситуація** — похідна вибірка з попереднього зв'язку, **не незалежний FK-зв'язок**: `Region.active_combat_situation` (О, `0..1`) ↔ `CombatSituation.region` (П).
- **Хвіст черги** — `1:0..1` для Region: `Region.combat_situation_queue_tail` (П, остання актуальна БС або `null`) ↔ —. Це окреме посилання на елемент того самого набору БС.
- **Область входу атакувальника (snapshot)** — `1:N`, nullable. `CombatSituation.attacker_entry_from_region` (П) ↔ —.
- `Region.next_queued_combat` **прибрано**; чергою керує сама CombatSituation, а не Region.

## Region ↔ Movement
- **Поточна область руху** — `1:N`. `Movement.current_region` (П) ↔ —.
- **Наступна область руху** — `1:N`, nullable залежно від фази. `Movement.next_region` (П) ↔ —.
- **Кінцева область маршруту** — `1:N`. `Movement.final_region` (П) ↔ —.
- **Область фізичного входу (snapshot)** — `1:N`, nullable. `Movement.entry_from_region` (П) ↔ —.
- **Локальна область виходу** — `1:N`, nullable. `Movement.local_exit_region` (П) ↔ —.
- **Маршрут** — `Movement.route[]` (П, впорядкований масив ID Region, із можливими повтореннями). **Не окремий M:N зв'язок** і не junction table; Region за ID за потреби завантажуються через Model.

## CombatSituation ↔ CombatSituation
- **Попередня/наступна БС в FIFO-черзі однієї Region** — самозв'язок `1:0..1` у межах актуальної черги. `CombatSituation.previous_combat_situation` (П, `null` для голови) ↔ `CombatSituation.next_combat_situation` (О, system-computed зворотний пошук). Жодних DFS-залежностей через цей зв'язок.
- При завершенні БС вона сама визначає наступну **до** власного завершення/виключення з актуальних моделей та запускає її; наступна на старті перевіряє актуальність ситуації. Якщо наступної немає, очищується `Region.combat_situation_queue_tail`. Історичне посилання `previous_combat_situation` після завершення попередньої БС очищувати не потрібно. `registration_order` для цієї черги не потрібен.

## Knight ↔ Army
- **Армія лицаря** — `1:N`. `Knight.army` (П) ↔ `Army.knights[]` (О).
- **Командир армії** — `1:1` (один Knight не може одночасно командувати більш ніж однією Army). `Army.commander` (П) ↔ —.

## Knight ↔ CastleFounding
- **Лицар-засновник** — `1:0..1` (серед актуальних процесів). `CastleFounding.founder_knight` (П) ↔ —.

## Army ↔ CampInRegion
- **Табір армії** — `1:N`, для Army `0..1`. `Army.camp` (П) ↔ `CampInRegion.armies[]` (О).

## Army ↔ Movement
- **Рух армії** — `1:0..1`, максимум один актуальний Movement на Army. `Movement.army` (П) ↔ `Army.movement` (О, лише active, `0..1`).

## Army ↔ CombatSituation
- **Атакувальна армія** — `1:0..1` серед актуальних CombatSituation. `CombatSituation.attacker` (П) ↔ `Army.attacking_combat_situation` (О, максимум одна незавершена).
- **Потенційні захисники** — кратність зв'язку ще не визначена (потребує окремого рішення). `CombatSituation.potential_defenders[]` (П, колекція, snapshot на Start) ↔ `Army.active_defender_combat_situation` (О, лише Active).
- **Кандидати на захист із транзиту** — відкладено після V1: `CombatSituation.transit_defender_candidates[]` не заповнюється й не створює актуального зв'язку.

## Додаткові зауваження до інвентаризації

1. **ID і похідні скаляри не є додатковими зв'язками.** Наприклад, `Army.player_id`, `CampInRegion.player_id`, `Region.player_id`, `Region.camp_player_ids[]`, `Region.blocking_camp_presence_player_ids[]`, `Castle.active_knight_replacement_id` не створюють самі по собі окремої пари навігаційних характеристик. Для `active_knight_replacement_id` може знадобитися окремо формалізувати семантику вибору конкретного KnightReplacement.
2. **Структуровані поля** на кшталт `CombatSituation.transit_defender_join_decisions`, `defenders_retreat_decisions` та `locked_combat_parameters` можуть містити ID Army, але специфікації не визначають їх як окремі навігаційні зв'язки. Вони не додані як самостійні зв'язки.
3. **Похідні проєкції** `Region.active_combat_situation`, `Castle.recruitment` та інші filtered views треба врахувати при формалізації способу отримання ID, але не дублювати незалежними фізичними зв'язками.
4. **Самозв'язки** підтримуються форматом на рівних правах із міжмодельними. Поточний явний приклад — `Region.neighbors[]`.
5. **Не зафіксовано тут**: SQL-таблиці/колонки, authoritative side для зміни, тип direct/system-computed за майбутньою централізованою схемою, DFS-напрямки. Це окремий наступний крок; вигадувати їх на підставі лише назви характеристики не можна.
