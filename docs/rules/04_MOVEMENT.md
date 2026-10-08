# Феодали — Переміщення: детальна специфікація V1

[Основні правила](../01_GAME_RULES_V1.md). Усі правила нижче збережені без змін із початкового документа.

# 13. Route, Movement, Dt і фіксація наміру

Player задає Army фізичний Route та кінцеву Region. Локальна мета Camp або Transit визначається самим movement order і після входу в поточну Region не переобчислюється через зміну ownership, Occupation або наявності військ. Під час combat command-lock заборонені звичайні movement-planning commands, включно зі зміною future route та explicit refresh `target_opponent`; дозволені лише прямо передбачені combat-specific decisions. Після зняття lock Player знову може refresh-нути target opponent до наступного border entry.

Проміжна Region маршруту проходиться як Transit. Якщо поточна Region є кінцевою для цього order, Army рухається до Camp. CombatSituation може pause-ити цей рух, але не змінює початкову локальну мету.

У V1 використовується одна базова константа часу Dt:

- Camp -> entry у сусідню Region: Dt;
- entry у Region -> Camp: Dt;
- базовий звичайний Transit однієї Region: Dt;
- CombatSituation Start -> Battle Start: Dt;
- звичайний Retreat entry -> Camp: Dt;
- Regrouping: Dt.

У майбутньому speed modifier може змінювати тільки звичайний Transit без CombatSituation; інші перелічені інтервали лишаються рівно Dt.

Якщо Player змінює Route під час Transit, зміна стосується тільки майбутньої частини маршруту. Уже зафіксований exit із поточної Region не змінюється. Army не може зупинитися посеред Neutral Transit і перетворити його на Camp; щоб зупинитися там, вона повинна вийти й зайти знову з відповідною метою. Виняток — Transit Army defender, яка явно приєднується до вже Active CombatSituation за правилами розділу 17.

Після половини поточного Transit іншим Player може бути відкритий тільки наступний exit direction, а не весь Route.

## 13.1 Target opponent Movement

При створенні Movement фіксується target_opponent за станом final Region:

- Neutral -> null;
- Owned non-Occupied Player B -> B;
- Owned Occupied Player C -> поточний occupier C.

Target opponent не дає формальному owner права на Transit через його Occupied Region.

target_opponent є характеристикою **Movement**, а не Army. Подальша зміна ownership/Occupation final Region автоматично його не змінює.

Player може явно виконати refresh target opponent. Нове значення обчислюється в момент refresh, зберігається як pending і набуває чинності тільки при вході Army в наступну Region. Уже зареєстрована CombatSituation від цього не змінюється.

---

# 14. Transit і Aggressive Transit

Звичайний Transit через Neutral Region не створює територіальної CombatSituation.

При вході чужої Army в Owned non-Occupied Region CombatSituation реєструється і для Camp, і для Transit.

Для Transit такої CombatSituation на Start використовується snapshot target_opponent, зафіксований при Registration:

- якщо formal owner поточної Region == target_opponent -> Aggressive Transit;
- інакше -> NonAggressive Transit.

Occupied Region є винятком: третій Player (не formal owner та не occupier) може Transit без CombatSituation, Allow Transit і Aggressive/NonAggressive classification. Formal owner не має права Transit через власну Occupied Region — для входу він мусить атакувати occupier з метою Camp. Чужий Transit через Capital Region заборонений. У не столичних Castle Region діють звичайні правила Owned/Occupied. Виняток: Transit, уже розпочатий фізичним входом у Neutral Region до completion Castle Founding, завершується без CombatSituation.

---

# 15. Allow Transit для Owned non-Occupied Region

Allow Transit є fallback rule тільки для NonAggressive Transit через Owned non-Occupied Region.

На CombatSituation Start battle context існує, якщо defender має Army у Camp/Regrouping або Army, що вже entered-for-Camp. Army, яка вже entered-for-Camp, до Battle Start гарантовано досягає Camp або переходить у Regrouping і для участі в battle еквівалентна звичайній Camp/Regrouping Army. Якщо таких Army немає, attacking Transit Army продовжує Route. Defender Transit Army самі по собі не створюють можливості interception.

Якщо Camp/Camp-bound defense context є, defender протягом Dt може один раз вручну обрати:

- Allow — CombatSituation завершується без battle, attacker продовжує Transit;
- Fight — situation доходить до Battle Start.

Ручний вибір immutable. Owner може змінювати `Allow Transit` до Battle Start за звичайними ownership rules. Якщо explicit `Allow/Fight` уже вибрано, fallback для цієї CombatSituation більше не використовується. Якщо ручного рішення немає до Battle Start, використовується актуальне на Battle Start значення Allow Transit: true -> Allow, false -> Fight.

Для Aggressive Transit Allow Transit не застосовується. Якщо defense context є, battle відбувається.

---

