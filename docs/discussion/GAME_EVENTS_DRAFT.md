# Феодали — повний робочий список ігрових подій

Це робочий список подій, які запускаються дією користувача, timer/scheduled event або trigger і змінюють game state. Внутрішні helper-виклики всередині тієї ж GameEvent окремими подіями тут не вважаються.

## Movement і присутність військ

1. **Start Movement**
   - Тип: дія користувача.
   - Умова: Army може почати рух і задано допустимий маршрут.
   - Наслідок: створюється Movement і починається рух.
   - Модель: `Movement`

2. **Change Planned Route**
   - Тип: дія користувача.
   - Умова: Army вже рухається, а незаблоковану частину маршруту ще можна змінити.
   - Наслідок: змінюється майбутня частина маршруту.
   - Модель: `Movement`

3. **Movement Direction Revealed**
   - Тип: trigger.
   - Умова: Movement проходить встановлену частку шляху через поточну Region.
   - Наслідок: напрямок виходу з Region стає відкритим для тих, хто має право його бачити.
   - Модель: `Movement`

4. **Next Region Reached**
   - Тип: trigger.
   - Умова: Movement доходить до межі наступної Region.
   - Наслідок: Army входить у наступну Region; оновлюється фаза руху і запускаються локальні наслідки входу.
   - Модель: `Movement`

5. **Camp Reached**
   - Тип: trigger.
   - Умова: Army завершує локальний рух усередині Region до Camp.
   - Наслідок: Army переходить у Camp; оновлюються локальна присутність, territorial processes та можливі взаємодії.
   - Модель: `Movement`

6. **Enter Castle**
   - Тип: дія користувача.
   - Умова: Unit/Army перебуває у відповідній Castle Region і для всіх потрібних солдатів є місце.
   - Наслідок: відповідні війська переходять зі стану Camp до Castle housing.
   - Модель: `Army`

7. **Leave Castle**
   - Тип: дія користувача.
   - Умова: війська перебувають у Castle.
   - Наслідок: війська переходять із Castle housing до Camp.
   - Модель: `Army`

8. **Stop Transit for Defense**
   - Тип: дія користувача.
   - Умова: Transit Army має право зупинитися для захисту до battle start.
   - Наслідок: Army припиняє Transit і додається до оборони.
   - Модель: `CombatSituation`

## Army / Unit organization

9. **Merge Armies**
   - Тип: дія користувача.
   - Умова: Armies співвласні, сумісні за станом і знаходяться разом у допустимому місці.
   - Наслідок: старі Army замінюються новою об'єднаною Army.
   - Модель: `Army`

10. **Split Army**
   - Тип: дія користувача.
   - Умова: Army перебуває в стані, де поділ дозволений.
   - Наслідок: вихідна Army замінюється кількома новими Army.
   - Модель: `Army`

11. **Change Commander**
   - Тип: дія користувача.
   - Умова: у Army є допустимі Knights і зміна дозволена поточним станом.
   - Наслідок: змінюється Commander-in-Chief.
   - Модель: `Army`

12. **Reorganize Unit Composition**
   - Тип: дія користувача.
   - Умова: Knight/Units перебувають у home Castle і реорганізація дозволена.
   - Наслідок: солдати перерозподіляються між Knights.
   - Модель: `Knight`

13. **Change Combat Thresholds**
   - Тип: дія користувача.
   - Умова: поточний стан дозволяє змінювати loss/retreat thresholds.
   - Наслідок: для Army зберігаються нові пороги.
   - Модель: `Army`

14. **Change Allow Transit**
   - Тип: дія користувача.
   - Умова: власник змінює правило проходу через свою територію.
   - Наслідок: змінюється характеристика Allow Transit.
   - Модель: `Player`

## Combat

15. **Set Pre-Battle Retreat Decision**
   - Тип: дія користувача.
   - Умова: CombatSituation ще не перейшла до battle start.
   - Наслідок: фіксується або змінюється рішення Army відступити без бою.
   - Модель: `CombatSituation`

16. **Battle Start / Resolve**
   - Тип: timer / scheduled event.
   - Умова: настав battle start і CombatSituation досі актуальна.
   - Наслідок: визначаються учасники, відступи, результат бою, втрати та післябойовий стан.
   - Модель: `CombatSituation`

17. **Attack Neutral Defense**
   - Тип: дія користувача.
   - Умова: Army знаходиться в Camp тієї самої Neutral Region і атака дозволена.
   - Наслідок: миттєво розв'язується бій із Neutral Defense.
   - Модель: `CombatSituation`

18. **Regrouping Complete**
   - Тип: timer / trigger.
   - Умова: сплив встановлений час Camp-Regrouping.
   - Наслідок: Army виходить із Regrouping і знову може виконувати дозволені дії.
   - Модель: `Army`

## Territory / RegionControl

19. **RegionControl Ready**
   - Тип: trigger.
   - Умова: накопичений control progress Player у Region досягає потрібного порога.
   - Наслідок: RegionControl переходить у стан, у якому Region може бути приєднана за виконання інших умов.
   - Модель: `RegionControl`

20. **Annex Region**
   - Тип: дія користувача.
   - Умова: RegionControl готовий; вибрано допустимий Castle; є Knight цього Castle і виконані territorial/capacity умови.
   - Наслідок: Region переходить у власність Player і прив'язується до вибраного Castle.
   - Модель: `RegionControl`

## Castle founding

21. **Start Castle Founding**
   - Тип: дія користувача.
   - Умова: Knight і Region відповідають умовам founding; витрати можуть бути сплачені.
   - Наслідок: створюється процес CastleFounding і починається накопичення progress.
   - Модель: `CastleFounding`

22. **Castle Founding Complete**
   - Тип: trigger.
   - Умова: founding progress досяг завершення і процес лишається валідним.
   - Наслідок: створюється новий Castle, Region стає Castle Region, founder переходить до нового Castle.
   - Модель: `CastleFounding`

## City

23. **Raid City**
   - Тип: дія користувача.
   - Умова: Army знаходиться в Camp тієї самої Region і raid дозволений.
   - Наслідок: нараховується raid reward і змінюється стан City/Region відповідно до правил raid.
   - Модель: `City`

## Buildings і Castle development

24. **Start Building Upgrade**
   - Тип: дія користувача.
   - Умова: виконані prerequisites, є ресурси і для цієї будівлі не йде інший upgrade.
   - Наслідок: списуються витрати і починається upgrade progress.
   - Модель: `Castle`

25. **Building Upgrade Complete**
   - Тип: trigger / timer.
   - Умова: upgrade progress конкретної будівлі завершився.
   - Наслідок: рівень будівлі збільшується і upgrade process завершується.
   - Модель: `Castle`

26. **Knight Replacement Complete**
   - Тип: timer / trigger.
   - Умова: сплив час відновлення вільного Palace slot після загибелі Knight.
   - Наслідок: створюється новий Knight у відповідному Castle.
   - Модель: `Castle`

## Resource Sources

27. **Start Resource Source Upgrade**
   - Тип: дія користувача.
   - Умова: Source може бути підвищене за правилом мінімального рівня і є потрібні ресурси.
   - Наслідок: списуються витрати і починається upgrade process Source.
   - Модель: `Region`

28. **Resource Source Upgrade Complete**
   - Тип: trigger / timer.
   - Умова: upgrade process Source завершився.
   - Наслідок: рівень Source збільшується.
   - Модель: `Region`

## Recruitment

29. **Add Recruitment Order**
   - Тип: дія користувача.
   - Умова: recruitment дозволений, є ресурси і немає blocking `empty_food` / `empty_coins`.
   - Наслідок: витрати списуються, order додається до recruitment queue.
   - Модель: `Recruitment`

30. **Current Recruit Finished**
   - Тип: trigger.
   - Умова: progress поточного recruit досяг завершення і Recruitment може завершити його.
   - Наслідок: створюється солдат, queue переходить до наступного елемента.
   - Модель: `Recruitment`

## Economy boundaries

31. **Food Empty Boundary**
   - Тип: двонаправлений trigger.
   - Умова: стан `empty_food` змінюється між активним і неактивним.
   - Наслідок: фіксується точна temporal boundary і перераховуються залежні economic/recruitment rates.
   - Модель: `Castle`

32. **Coins Empty Boundary**
   - Тип: двонаправлений trigger.
   - Умова: стан `empty_coins` змінюється між активним і неактивним.
   - Наслідок: фіксується точна temporal boundary і перераховуються залежні economic processes.
   - Модель: `Player`

33. **Storage Capacity Boundary**
   - Тип: trigger.
   - Умова: bulk resource або Food досягає відповідної storage capacity.
   - Наслідок: запас фіксується на capacity, а подальший effective growth для переповненого storage стає нульовим до зміни умов.
   - Модель: `Castle`

## Neutral Defense recovery

34. **Neutral Defense Recovery Complete**
   - Тип: trigger / timer.
   - Умова: після повного знищення Neutral Defense Region лишалася без військ увесь recovery period.
   - Наслідок: Neutral Defense відновлюється одразу до повного значення.
   - Модель: `Region`

## Події, які зараз не виділяються в окрему GameEvent

Наступні зміни відбуваються як наслідки перелічених вище GameEvent і окремими root-подіями не вважаються:

- створення/завершення occupation після movement/combat;
- відновлення owner control після виходу occupier;
- перерахунок `is_connection_valid` у Castle після territorial change;
- автоматичне перетворення permanently disconnected Region на Neutral;
- pause/reset RegionControl через зміну локальної присутності;
- pause/cancel CastleFounding через зміну стану founder або Region;
- запуск recovery period Neutral Defense після виходу останніх військ;
- створення retreat Movement після battle resolution;
- смерть Knight, розпуск Army або зміна occupation як безпосередній результат combat;
- створення/видалення process-model, якщо це лише внутрішній наслідок іншої GameEvent.
