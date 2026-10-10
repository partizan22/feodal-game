# Зв'язки моделей — інвентаризація V1

Джерела: `MODELS_CORE.md`, `MODELS_MILITARY.md`, `MODELS_COMBAT.md`, `MODELS_PROCESSES.md` (стан на 2026-10-10).

Це інвентаризація **зв'язків, представлених характеристиками моделей**, а не остаточна декларація SQL-схеми чи напрямків DFS. Кожен зв'язок описаний **один раз**, незалежно від напрямку навігації. Якщо між тією самою парою типів моделей є кілька незалежних зв'язків, вони наведені підряд. Зв'язок моделі із самою собою допустимий.

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
- **Черга рекрутингу** — `1:N` у часі, максимум один active Recruitment на Castle. `Recruitment.castle` (П) ↔ `Castle.recruitment` (О, лише active, `0..1`).

## Castle ↔ BuildingUpgrade
- **Покращення будівель** — `1:N`. `BuildingUpgrade.castle` (П) ↔ `Castle.building_upgrades[]` (О, включно з історією).

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
- **Заснування замку в області** — `1:N` у часі. `CastleFounding.region` (П) ↔ `Region.castle_foundings[]` (О).

## Region ↔ ResourceSiteUpgrade
- **Покращення видобувних об'єктів** — `1:N`. `ResourceSiteUpgrade.region` (П) ↔ `Region.resource_site_upgrades[]` (О).

## Region ↔ CombatSituation
- **Бойові ситуації в області** — `1:N`. `CombatSituation.region` (П) ↔ `Region.combat_situations[]` (О, включно з історією).
- **Поточна активна бойова ситуація** — похідна вибірка з попереднього зв'язку, **не незалежний FK-зв'язок**: `Region.active_combat_situation` (О, `0..1`) ↔ `CombatSituation.region` (П).
- **Наступна ситуація в черзі** — похідна вибірка з того самого зв'язку, **не незалежний FK-зв'язок**: `Region.next_queued_combat` (О, `0..1`) ↔ `CombatSituation.region` (П).

## Region ↔ Movement
- **Поточна область руху** — `1:N`. `Movement.current_region` (П) ↔ —.
- **Наступна область руху** — `1:N`, nullable залежно від фази. `Movement.next_region` (П) ↔ —.
- **Область фізичного входу (snapshot)** — `1:N`, nullable. `Movement.entry_from_region` (П) ↔ —.
- **Локальна область виходу** — `1:N`, nullable. `Movement.local_exit_region` (П) ↔ —.
- **Маршрут (послідовність областей)** — упорядкований `M:N` із можливими повтореннями Region у маршруті: `Movement.route[]` (П, колекція) ↔ —. Це не звичайна множина; важливий порядок і, можливо, повторення.

## Region ↔ CombatSituation (додаткові ролі)
- **Область входу атакувальника (snapshot)** — `1:N`, nullable. `CombatSituation.attacker_entry_from_region` (П) ↔ —.
- **Область відступу атакувальника** — `1:N`, nullable. `CombatSituation.attacker_retreat_region` (П) ↔ —.
- **Область відступу захисників** — `1:N`, nullable. `CombatSituation.defender_retreat_region` (П) ↔ —.

## Knight ↔ Army
- **Армія лицаря** — `1:N`. `Knight.army` (П) ↔ `Army.knights[]` (О).
- **Командир армії** — `1:N` як потенційна роль лицаря у часі; для кожної Army один commander. `Army.commander` (П) ↔ —. Уточнити, чи один Knight може одночасно командувати більш ніж однією Army (ігровий інваріант).

## Knight ↔ CastleFounding
- **Лицар-засновник** — `1:N` у часі. `CastleFounding.founder_knight` (П) ↔ —.

## Army ↔ CampInRegion
- **Табір армії** — `1:N`, для Army `0..1`. `Army.camp` (П) ↔ `CampInRegion.armies[]` (О).

## Army ↔ Movement
- **Рух армії** — `1:N` у часі, максимум один active Movement на Army. `Movement.army` (П) ↔ `Army.movement` (О, лише active, `0..1`).

## Army ↔ CombatSituation
- **Атакувальна армія** — `1:N` у часі. `CombatSituation.attacker` (П) ↔ `Army.attacking_combat_situation` (О, максимум одна незавершена).
- **Потенційні захисники** — `M:N`. `CombatSituation.potential_defenders[]` (П, колекція, snapshot на Start) ↔ `Army.active_defender_combat_situation` (О, лише Active).
- **Кандидати на захист із транзиту** — `M:N`. `CombatSituation.transit_defender_candidates[]` (П, колекція) ↔ —. Цей зв'язок не тотожний участі у захисті.

## Додаткові зауваження до інвентаризації

1. **ID і похідні скаляри не є додатковими зв'язками.** Наприклад, `Army.player_id`, `CampInRegion.player_id`, `Region.player_id`, `Region.camp_player_ids[]`, `Region.blocking_camp_presence_player_ids[]`, `Castle.active_knight_replacement_id` не створюють самі по собі окремої пари навігаційних характеристик. Для `active_knight_replacement_id` може знадобитися окремо формалізувати семантику вибору конкретного KnightReplacement.
2. **Структуровані поля** на кшталт `CombatSituation.transit_defender_join_decisions`, `defenders_retreat_decisions` та `locked_combat_parameters` можуть містити ID Army, але специфікації не визначають їх як окремі навігаційні зв'язки. Вони не додані як самостійні зв'язки.
3. **Похідні проєкції** `Region.active_combat_situation`, `Region.next_queued_combat`, `Castle.recruitment` та інші filtered views треба врахувати при формалізації способу отримання ID, але не дублювати незалежними фізичними зв'язками.
4. **Самозв'язки** підтримуються форматом на рівних правах із міжмодельними. Поточний явний приклад — `Region.neighbors[]`.
5. **Не зафіксовано тут**: SQL-таблиці/колонки, authoritative side для зміни, тип direct/system-computed за майбутньою централізованою схемою, DFS-напрямки. Це окремий наступний крок; вигадувати їх на підставі лише назви характеристики не можна.
