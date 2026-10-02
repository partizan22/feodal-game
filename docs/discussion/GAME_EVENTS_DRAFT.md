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
   - Наслідок: змінюється майбутня частина маршруту.
   - Модель: `? -> Movement`

3. **Movement Direction Revealed**
   - Тип: trigger.
   - Умова: Movement проходить встановлену частку шляху через Region.
   - Наслідок: backend game state не змінюється; виконується frontend update та notification для тих, хто має право бачити напрямок.
   - Модель: `Movement`

4. **Next Region Reached**
   - Тип: trigger.
   - Умова: Movement доходить до межі наступної Region.
   - Наслідок: Army входить у наступну Region; оновлюється рух і локальні процеси.
   - Модель: `Movement -> Army, RegionControl`

5. **Camp Reached**
   - Тип: trigger.
   - Умова: Army завершує локальний рух до Camp.
   - Наслідок: Army переходить у Camp; оновлюються локальна присутність і пов'язані процеси.
   - Модель: `Movement -> Army, RegionControl`

6. **Enter Castle**
   - Тип: дія користувача.
   - Умова: Unit перебуває у відповідній Castle Region і є місце.
   - Наслідок: Unit переходить із Camp до Castle housing.
   - Модель: `? -> Knight, Castle`

7. **Leave Castle**
   - Тип: дія користувача.
   - Умова: Unit перебуває у Castle.
   - Наслідок: Unit переходить із Castle housing до Camp.
   - Модель: `? -> Knight, Castle`

8. **Stop Transit for Defense**
   - Тип: дія користувача.
   - Умова: Transit Army може зупинитися для захисту до battle start.
   - Наслідок: Army припиняє Transit і додається до оборони.
   - Модель: `? -> CombatSituation, Movement, Army`

## Army / Unit organization

9. **Merge Armies**
   - Тип: дія користувача.
   - Умова: Armies можна об'єднати.
   - Наслідок: створюється об'єднана Army.
   - Модель: `? -> Army`

10. **Split Army**
   - Тип: дія користувача.
   - Умова: поточний стан Army дозволяє поділ.
   - Наслідок: Army розділяється на кілька Armies.
   - Модель: `? -> Army`

11. **Change Commander**
   - Тип: дія користувача.
   - Умова: зміна Commander-in-Chief дозволена.
   - Наслідок: змінюється Commander-in-Chief Army.
   - Модель: `? -> Army`

12. **Change Unit Composition**
   - Тип: дія користувача.
   - Умова: Unit перебуває у home Castle і зміна складу дозволена.
   - Наслідок: солдати переводяться `Castle reserve <-> Knight`.
   - Модель: `? -> Castle, Knight`

13. **Change Combat Thresholds**
   - Тип: дія користувача.
   - Умова: поточний стан дозволяє змінити thresholds.
   - Наслідок: змінюються loss/retreat thresholds Army.
   - Модель: `? -> Army`

14. **Change Allow Transit**
   - Тип: дія користувача.
   - Умова: власник Region змінює правило проходу.
   - Наслідок: змінюється `allow_transit` Region.
   - Модель: `? -> Region`

## Combat

15. **Set Pre-Battle Retreat Decision**
   - Тип: дія користувача.
   - Умова: battle ще не почався.
   - Наслідок: фіксується або змінюється рішення Army про відступ.
   - Модель: `? -> CombatSituation`

16. **Battle Start / Resolve**
   - Тип: trigger.
   - Умова: досягнуто battle start і CombatSituation досі актуальна.
   - Наслідок: розв'язується бій і застосовуються його наслідки; Castle створює KnightReplacement для Knights, які потребують replacement.
   - Модель: `CombatSituation -> Army, Knight, Castle, RegionControl, KnightReplacement`

17. **Attack Neutral Defense**
   - Тип: дія користувача.
   - Умова: війська Player у Camp Neutral Region можуть атакувати Neutral Defense.
   - Наслідок: RegionControl створює CombatSituation і запускається бій.
   - Модель: `? -> RegionControl, CombatSituation`

18. **Regrouping Complete**
   - Тип: trigger.
   - Умова: завершився період Camp-Regrouping.
   - Наслідок: Army виходить із Regrouping.
   - Модель: `Army`

## Territory / RegionControl

19. **RegionControl Ready**
   - Тип: trigger.
   - Умова: control progress досягає потрібного порога.
   - Наслідок: RegionControl стає готовим до annexation.
   - Модель: `RegionControl`

20. **Annex Region**
   - Тип: дія користувача.
   - Умова: виконані умови annexation і вибрано допустимий Castle.
   - Наслідок: Region переходить у власність Player і приєднується до Castle.
   - Модель: `? -> RegionControl, Region, Castle`

## Castle founding

21. **Start Castle Founding**
   - Тип: дія користувача.
   - Умова: виконані умови founding.
   - Наслідок: Player створює CastleFounding і починається founding progress.
   - Модель: `? -> Player, CastleFounding`

22. **Castle Founding Complete**
   - Тип: trigger.
   - Умова: founding progress завершений і процес валідний.
   - Наслідок: створюється Castle, Region стає Castle Region, founder переходить до нового Castle.
   - Модель: `CastleFounding -> Player, Castle, Region, Knight`

## City

23. **Raid City**
   - Тип: дія користувача.
   - Умова: війська Player можуть здійснити raid City.
   - Наслідок: застосовується raid reward і наслідки для City/Region.
   - Модель: `? -> RegionControl, City`

## Buildings і Castle development

24. **Start Building Upgrade**
   - Тип: дія користувача.
   - Умова: upgrade дозволений і є потрібні ресурси.
   - Наслідок: списуються витрати і створюється окремий BuildingUpgrade для цього будівництва.
   - Модель: `? -> Castle, BuildingUpgrade`

25. **Building Upgrade Complete**
   - Тип: trigger.
   - Умова: BuildingUpgrade досягає завершення.
   - Наслідок: рівень будівлі збільшується; BuildingUpgrade переходить у завершений стан.
   - Модель: `BuildingUpgrade -> Castle`

26. **Knight Replacement Complete**
   - Тип: trigger.
   - Умова: KnightReplacement досягає завершення.
   - Наслідок: створюється новий Knight у відповідному Castle; KnightReplacement переходить у завершений стан.
   - Модель: `KnightReplacement -> Castle, Knight`

Початок Knight replacement не є окремою root GameEvent: це наслідок **Battle Start / Resolve**. За створення відповідного `KnightReplacement` відповідає `Castle`.

## Resource Sites

27. **Start ResourceSite Upgrade**
   - Тип: дія користувача.
   - Умова: ResourceSite можна підвищити і є потрібні ресурси.
   - Наслідок: списуються витрати і створюється окремий ResourceSiteUpgrade для цього будівництва.
   - Модель: `? -> Region, ResourceSiteUpgrade`

28. **ResourceSite Upgrade Complete**
   - Тип: trigger.
   - Умова: ResourceSiteUpgrade досягає завершення.
   - Наслідок: рівень ResourceSite збільшується; ResourceSiteUpgrade переходить у завершений стан.
   - Модель: `ResourceSiteUpgrade -> Region`

## Recruitment

29. **Add Recruitment Order**
   - Тип: дія користувача.
   - Умова: recruitment дозволений і є потрібні ресурси.
   - Наслідок: order додається до recruitment queue; за потреби створюється Recruitment.
   - Модель: `? -> Castle, Recruitment`

30. **Current Recruit Finished**
   - Тип: trigger.
   - Умова: progress поточного recruit досяг завершення.
   - Наслідок: recruit завершується і queue переходить до наступного елемента.
   - Модель: `Recruitment -> Castle`

## Economy boundaries

31. **Food Empty Boundary**
   - Тип: двонаправлений trigger.
   - Умова: стан `empty_food` змінюється.
   - Наслідок: перераховуються залежні rates/processes.
   - Модель: `Castle`

32. **Coins Empty Boundary**
   - Тип: двонаправлений trigger.
   - Умова: стан `empty_coins` змінюється.
   - Наслідок: перераховуються залежні rates/processes.
   - Модель: `Player`

33. **Storage Capacity Boundary**
   - Тип: trigger.
   - Умова: ресурс досягає storage capacity.
   - Наслідок: подальший effective growth стає нульовим до зміни умов.
   - Модель: `Castle`

## Neutral Defense recovery

34. **Neutral Defense Recovery Complete**
   - Тип: trigger.
   - Умова: виконані умови повного відновлення Neutral Defense.
   - Наслідок: Neutral Defense відновлюється до повного значення.
   - Модель: `Region`

## Події, які зараз не виділяються в окрему GameEvent

Наступні зміни відбуваються як наслідки перелічених вище GameEvent і окремими root-подіями не вважаються:

- створення/завершення RegionControl після movement/combat;
- відновлення owner control після виходу occupier;
- перерахунок `is_connection_valid` після зміни occupation/ownership;
- створення KnightReplacement після загибелі Knight у battle;
- інші внутрішні виклики між моделями в межах поточної GameEvent.
