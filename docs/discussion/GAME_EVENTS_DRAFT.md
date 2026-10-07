# Феодали — повний робочий список ігрових подій

Робочий список GameEvent: user actions і triggers, які змінюють game state або мають окремий системний ефект (notification / frontend update). Окремого типу timer більше немає: scheduled wake-up використовується тільки для перевірки trigger-а.

Для user-подій root model поки не визначена до проєктування взаємодії з frontend/API, тому використовується запис `? -> LogicModel(s)`. Для trigger-подій першою вказана модель, метод якої запускає worker; після `->` — моделі, яким вона передає основну логіку.

## Movement і присутність військ

1. **Start Movement**
   - Тип: дія користувача.
   - Умова: Army може почати рух і задано допустимий маршрут.
   - Наслідок: створюється Movement і починається рух.
   - Модель: `? -> Army, Movement`

2. **Change Planned Route**
   - Тип: дія користувача.
   - Умова: Army рухається і маршрут ще можна змінити.
   - Наслідок: змінюється майбутня частина маршруту; вже зафіксований exit поточної Region та `Movement.target_opponent` самі не переобчислюються.
   - Модель: `? -> Movement`

3. **Refresh Target Opponent**
   - Тип: дія користувача.
   - Умова: Army має active Movement.
   - Наслідок: за актуальним станом final Region обчислюється pending `Movement.target_opponent`; значення застосовується при вході Army в наступну Region і не змінює вже зареєстровану CombatSituation.
   - Модель: `? -> Movement`

4. **Movement Direction Revealed**
   - Тип: trigger.
   - Умова: Movement проходить встановлену частку шляху через Region.
   - Наслідок: `Movement.direction_revealed` переходить у true; projection/frontend може оновити доступний наступний exit direction і notification.
   - Модель: `Movement`

5. **Next Region Reached**
   - Тип: trigger.
   - Умова: Movement доходить до межі наступної Region.
   - Наслідок: Army входить у наступну Region; оновлюється рух і локальні процеси.
   - Модель: `Movement -> Army, Region`

6. **Camp Reached**
   - Тип: trigger.
   - Умова: Army завершує локальний рух до Camp.
   - Наслідок: Army переходить у Camp; оновлюються локальна присутність і пов'язані процеси.
   - Модель: `Movement -> Army, Region, CampInRegion`

7. **Enter Castle**
   - Тип: дія користувача.
   - Умова: Unit перебуває у відповідній Castle Region і є місце.
   - Наслідок: Unit переходить із Camp до Castle housing.
   - Модель: `? -> Knight, Castle`

8. **Leave Castle**
   - Тип: дія користувача.
   - Умова: Unit перебуває у Castle.
   - Наслідок: Unit переходить із Castle housing до Camp.
   - Модель: `? -> Knight, Castle`

9. **Stop Transit for Defense**
   - Тип: дія користувача.
   - Умова: Transit Army може зупинитися для захисту до battle start.
   - Наслідок: active Movement/старий Route завершується, Army переходить у Camp і стає potential defender цієї CombatSituation.
   - Модель: `? -> CombatSituation, Movement, Army`

## Army / Unit organization

10. **Merge Armies**
   - Тип: дія користувача.
   - Умова: Armies у звичайному Camp можна об'єднати; Regrouping merge забороняє.
   - Наслідок: створюється об'єднана Army.
   - Модель: `? -> Army`

11. **Split Army**
   - Тип: дія користувача.
   - Умова: Army у звичайному Camp дозволяє поділ; Regrouping split забороняє.
   - Наслідок: Army розділяється на кілька Armies.
   - Модель: `? -> Army`

12. **Change Commander**
   - Тип: дія користувача.
   - Умова: зміна Commander-in-Chief дозволена.
   - Наслідок: змінюється Commander-in-Chief Army.
   - Модель: `? -> Army`

13. **Change Unit Composition**
   - Тип: дія користувача.
   - Умова: Unit перебуває у home Castle і зміна складу дозволена.
   - Наслідок: солдати переводяться `Castle reserve <-> Knight`.
   - Модель: `? -> Castle, Knight`

14. **Change Combat Thresholds**
   - Тип: дія користувача.
   - Умова: поточний стан дозволяє змінити thresholds.
   - Наслідок: змінюються три persistent combat loss thresholds Army.
   - Модель: `? -> Army`

15. **Change Allow Transit**
   - Тип: дія користувача.
   - Умова: власник Region змінює правило проходу.
   - Наслідок: змінюється `allow_transit` Region.
   - Модель: `? -> Region`

## Combat

16. **Attack Player in Neutral Region**
   - Тип: дія користувача.
   - Умова: attacking Army перебуває у `state == Camp` Neutral Region, не command-locked; target Player має в цій самій Region хоча б одну Army у `state == Camp`. Самі лише Regrouping або entered-for-Camp Army target Player недостатні для ініціації.
   - Наслідок: реєструється CombatSituation з цією Army як єдиним attacker і target Player як зафіксованим defender; Movement `target_opponent` тут не використовується.
   - Модель: `? -> Army, Region, CombatSituation`

17. **Set Transit Decision**
   - Тип: дія користувача.
   - Умова: Active CombatSituation є NonAggressive Transit і Battle Start ще не настав.
   - Наслідок: defender один раз фіксує `Allow` або `Fight`; ручне рішення immutable.
   - Модель: `? -> CombatSituation`

18. **Set Pre-Battle Retreat Decision**
   - Тип: дія користувача.
   - Умова: CombatSituation Active, Army є potential defender і Battle Start ще не настав.
   - Наслідок: локально для цієї CombatSituation effective Defense Loss Threshold Army стає `0`; persistent threshold не змінюється. На Battle Start Retreat виконається тільки після side-level перевірки legal Retreat.
   - Модель: `? -> CombatSituation`

19. **Battle Start / Resolve**
   - Тип: trigger.
   - Умова: досягнуто battle start і CombatSituation досі актуальна.
   - Наслідок: спочатку перевіряється Retreat availability, застосовуються zero-threshold Retreat, потім за потреби розв'язується player-vs-player battle. Загиблий Knight створює KnightReplacement; якщо загинув Commander, після всіх casualties новим стає surviving Knight з найбільшим Experience.
   - Модель: `CombatSituation -> Army, Knight, Castle, Region, CampInRegion, Movement, KnightReplacement`

20. **Attack Neutral Defense**
   - Тип: дія користувача.
   - Умова: війська Player у Camp Neutral Region можуть атакувати Neutral Defense.
   - Наслідок: миттєво розраховується спеціальний combat поза player-vs-player CombatSituation queue; за наявності City до current Neutral Defense додається full City Defense.
   - Модель: `? -> Army, Region, City, Knight, Castle, KnightReplacement`

21. **Regrouping Complete**
   - Тип: trigger.
   - Умова: завершився період Camp-Regrouping.
   - Наслідок: Army виходить із Regrouping.
   - Модель: `Army`

## Territory / CampInRegion

22. **Annexation Ready**
   - Тип: trigger.
   - Умова: `CampInRegion.control_progress` досягає потрібного порога.
   - Наслідок: CampInRegion стає ready для manual Annexation.
   - Модель: `CampInRegion`

23. **Annex Region**
   - Тип: дія користувача.
   - Умова: виконані умови annexation і вибрано допустимий Castle.
   - Наслідок: Region переходить у власність Player і приєднується до Castle.
   - Модель: `? -> CampInRegion, Region, Castle`

## Castle founding

24. **Start Castle Founding**
   - Тип: дія користувача.
   - Умова: виконані умови founding.
   - Наслідок: Player створює CastleFounding і починається founding progress.
   - Модель: `? -> Player, CastleFounding`

25. **Castle Founding Complete**
   - Тип: trigger.
   - Умова: founding progress завершений і процес валідний.
   - Наслідок: створюється Castle, Region стає Castle Region, founder переходить до нового Castle.
   - Модель: `CastleFounding -> Player, Castle, Region, Knight`

## City

26. **Raid City**
   - Тип: дія користувача.
   - Умова: війська Player можуть здійснити raid City.
   - Наслідок: застосовується raid reward і наслідки для City/Region.
   - Модель: `? -> Army, City, Knight, Castle, KnightReplacement`

## Buildings і Castle development

27. **Start Building Upgrade**
   - Тип: дія користувача.
   - Умова: upgrade дозволений і є потрібні ресурси.
   - Наслідок: списуються витрати і створюється окремий BuildingUpgrade для цього будівництва.
   - Модель: `? -> Castle, BuildingUpgrade`

28. **Building Upgrade Complete**
   - Тип: trigger.
   - Умова: BuildingUpgrade досягає завершення.
   - Наслідок: рівень будівлі збільшується; BuildingUpgrade переходить у завершений стан.
   - Модель: `BuildingUpgrade -> Castle`

29. **Knight Replacement Complete**
   - Тип: trigger.
   - Умова: KnightReplacement досягає завершення.
   - Наслідок: створюється новий Knight у відповідному Castle; KnightReplacement переходить у завершений стан.
   - Модель: `KnightReplacement -> Castle, Knight`

Початок Knight replacement не є окремою root GameEvent: будь-який `Knight.die()` викликає створення відповідного `KnightReplacement` у home Castle, незалежно від того, чи death сталася в player-vs-player battle, Neutral Defense attack або City Raid.

## Resource Sites

30. **Start ResourceSite Upgrade**
   - Тип: дія користувача.
   - Умова: ResourceSite можна підвищити і є потрібні ресурси.
   - Наслідок: списуються витрати і створюється окремий ResourceSiteUpgrade для цього будівництва.
   - Модель: `? -> Region, ResourceSiteUpgrade`

31. **ResourceSite Upgrade Complete**
   - Тип: trigger.
   - Умова: ResourceSiteUpgrade досягає завершення.
   - Наслідок: рівень ResourceSite збільшується; ResourceSiteUpgrade переходить у завершений стан.
   - Модель: `ResourceSiteUpgrade -> Region`

## Recruitment

32. **Add Recruitment Order**
   - Тип: дія користувача.
   - Умова: recruitment дозволений і є потрібні ресурси.
   - Наслідок: order додається до recruitment queue; за потреби створюється Recruitment.
   - Модель: `? -> Castle, Recruitment`

33. **Current Recruit Finished**
   - Тип: trigger.
   - Умова: progress поточного recruit досяг завершення.
   - Наслідок: recruit завершується і queue переходить до наступного елемента.
   - Модель: `Recruitment -> Castle`

## Economy boundaries

34. **Food Empty Boundary**
   - Тип: `[state trigger]`.
   - Умова: boolean-стан `empty_food` змінюється (`false <-> true`).
   - Наслідок: перераховуються залежні rates/processes.
   - Модель: `Castle`

35. **Coins Empty Boundary**
   - Тип: `[state trigger]`.
   - Умова: boolean-стан `empty_coins` змінюється (`false <-> true`).
   - Наслідок: перераховуються залежні rates/processes.
   - Модель: `Player`

36. **Storage Capacity Boundary**
   - Тип: trigger.
   - Умова: ресурс досягає storage capacity.
   - Наслідок: подальший effective growth стає нульовим до зміни умов.
   - Модель: `Castle`

## Neutral Defense recovery

37. **Neutral Defense Recovery Complete**
   - Тип: trigger.
   - Умова: виконані умови повного відновлення Neutral Defense.
   - Наслідок: Neutral Defense відновлюється до повного значення.
   - Модель: `Region`

## Події, які зараз не виділяються в окрему GameEvent

Наступні зміни відбуваються як наслідки перелічених вище GameEvent і окремими root-подіями не вважаються:

- створення/завершення CampInRegion після Camp presence;
- відновлення owner control після виходу occupier;
- перерахунок `is_connection_valid` після зміни occupation/ownership;
- створення KnightReplacement після будь-якого `Knight.die()`;
- інші внутрішні виклики між моделями в межах поточної GameEvent.
