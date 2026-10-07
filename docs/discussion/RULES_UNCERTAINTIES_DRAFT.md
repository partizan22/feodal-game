# Невизначеності правил і моделей — на перевірку

Цей файл **не є джерелом чинних правил**. Тут зібрані лише місця, де на поточному етапі аудиту лишається неоднозначність або бракує явного правила для однозначної backend-реалізації.

Для кожного пункту наведено рекомендований варіант. Після затвердження рішення треба перенести у `01_GAME_RULES_V1.md` та/або `MODELS.md`, а цей пункт прибрати або позначити вирішеним.

---

## 1. Casualties для об'єднаної Defender side

### Невизначеність

Коли Defender складається з кількох Army, правила задають один side-level LossBudget і говорять про розподіл між Unit, але прямо не визначають, чи спочатку budget ділиться між Army, а потім між їх Unit, чи всі Unit defender side беруть участь в одному спільному розподілі.

### Пропоноване рішення

Не вводити окремий рівень розподілу по Army. Після визначення side-level LossBudget усі Unit усіх фактичних participating Army цієї side утворюють один набір. Budget розподіляється між Unit приблизно пропорційно їх Casualty Health з установленою randomness.

Army лишається організаційною структурою і впливає на strength через свого Commander, але не створює окремої "квоти" casualties.

---

## 2. Threshold 100% за відсутності legal Retreat

### Невизначеність

Якщо side не має legal Retreat, її effective threshold стає 100%. Водночас Knight death має probabilistic mortality coefficient. Теоретично side може досягти 100% LossFraction, програти, але після casualty resolution мати живого Knight. Немає визначеного post-combat стану такого survivor, бо Retreat неможливий.

### Пропоноване рішення

`100%` у ситуації без legal Retreat трактувати як fight-to-destruction: side, яка першою досягла такого threshold і програла, повністю знищується.

Для цього при terminal LossFraction = 100% усі Soldier і Knight losing side гарантовано гинуть; Knight mortality randomness у цьому окремому випадку не застосовується.

---

## 3. Battle Experience: хто його отримує

### Невизначеність

Правила кажуть, що Knight отримує Experience після боїв залежно від масштабу противника, але не визначають точний eligibility:
- чи отримують XP тільки Knight, що реально брали участь у combat calculation;
- чи отримують його Knight, які виконали pre-battle Retreat;
- чи отримує XP Knight, який загинув у цьому battle;
- чи отримують однаковий базовий XP усі Knight side.

### Пропоноване рішення

Battle XP отримують тільки **живі після battle Knight, які входили до фактичного combat calculation**.

Knight, що пішли pre-battle Retreat до calculation, XP не отримують. Загиблі Knight XP не потребують.

Базовий XP для кожного participating survivor визначати функцією від pre-combat strength противника та, за потреби, власної side; не залежати від того, скільки Soldier конкретно втратив цей Unit. 

### Зміна
Commander отримує більше досвіду за статус Commander.

---

## 4. Passive Experience з часом

### Невизначеність

За правилами Experience Knight повільно зростає з часом, але не визначено, в яких states це відбувається.

### Пропоноване рішення

Passive Experience нараховується всім існуючим живим Knight безперервно незалежно від `Castle / Camp / Movement / Regrouping` та combat lock.

Неназваний ready Knight ще не існує і Experience не накопичує. Dead Knight також не накопичує Experience.

---

## 5. Pre-battle Retreat decision: чи можна змінити рішення

### Невизначеність

Для `Allow/Fight` прямо зафіксовано immutable decision. Для індивідуального pre-battle Retreat defender Army такої вказівки немає.

### Пропоноване рішення

До Battle Start дозволити змінювати pre-battle Retreat decision; чинним є останнє значення.

Після Battle Start decision lock-иться разом з іншими combat parameters.

Це не змінює persistent `defense_loss_threshold`.

---

## 6. Regrouping, що завершується між CombatSituation Start і Battle Start

### Невизначеність

Тепер встановлено, що під час Regrouping effective Defense Loss Threshold = 0, а після завершення Regrouping знову діє persistent threshold. Треба явно визначити, який стан береться, якщо Army була в Regrouping на CombatSituation Start, але завершила його до Battle Start.

### Пропоноване рішення

Effective threshold визначається **на Battle Start за актуальним станом Army**.

Якщо Regrouping уже завершився — використовується persistent threshold. Якщо Army все ще Regrouping — effective threshold = 0, а при відсутності legal Retreat примусово 100%.

---

## 7. Retreat: що означає "Neutral Region ближча до власної territory"

### Невизначеність

Правила задають цей пріоритет, але не визначають точну metric і яку саме власну territory враховувати.

### Пропоноване рішення

Для кожної Neutral candidate Region рахувати мінімальну стандартну hex-grid distance до будь-якої **Owned non-Occupied Region** цього Player.

Перемагає candidate з меншою distance. При рівності використовується seeded random tie-breaker.

Occupied  власні Region не вважати опорними точками для цього priority.

### Зміна:
disconnected можна, якщо вона ще не втрачена остоточно 

---

## 8. Retreat: критерій вибору серед legal foreign Region

### Невизначеність

У правилах залишено лише "визначений географічний критерій".

### Зміна

Ні. Для чужих територій пріоритет - мінімальна відстань до найближчної нейтральної (армія намагається якомога швидще покинути чужу територію)

---

## 9. Reserve Soldier: Food consumption та Coin upkeep

### Невизначеність

Barracks містить reserve Soldier, але поточні правила Food/upkeep описані переважно через Army/Unit. Не визначено явно, чи Soldier у reserve продовжують споживати Food і Coins.

### Зміна

Reserve Soldier повністю зберігають звичайний Soldier upkeep:
- споживають Food Castle;
- мають Coin upkeep відповідно до Soldier Type;
- одразу починають це робити після завершення Recruitment.

Інакше Player міг би безкоштовно уникати upkeep, просто залишаючи Soldier у reserve.

---

## 10. Disconnected City: чи продовжує рости Wealth

### Невизначеність

Позитивний Coin income disconnected Region уже визначено як 0. Але City Wealth growth зараз залежить від Owned + non-Occupied + `!empty_coins`, без явної вимоги territorial connection.

### Зміна

Disconnected City **продовжує локально нарощувати Wealth**, якщо Region Owned, non-Occupied, `active_wealth_ratio == 1` і Player не має `empty_coins`.

Recurring Coin income при цьому не надходить Player до відновлення connection.

Occupation, як і зараз, зупиняє Wealth growth.

---

## 11. Voluntary abandonment при queued Registered CombatSituation

### Невизначеність

Встановлено, що abandon заборонений при active CombatSituation. Неясно, чи слово `active` означає тільки status `Active`, чи будь-яку unresolved situation, включно з `Registered` у FIFO queue.

### Пропоноване рішення

Трактувати правило буквально: blocker — тільки `Active` CombatSituation.

Якщо в Region є лише queued `Registered` situations, owner може abandon Region. Після зміни state вони не видаляються наперед, а кожна на своєму Start перевіряє актуальність interaction і за потреби завершується без battle за вже чинним загальним правилом queue.

### Уточнення

Якщо в Region є лише queued situations, то є і активна, інакше чого чикає queued?
---

## 12. Voluntary abandonment під час власного CastleFounding

### Невизначеність

Player може abandon будь-яку ordinary Region, а Founding дозволений і в Neutral Region, і у власній annexed Region. Прямо не сказано, що відбувається з уже active Founding, якщо owner добровільно робить Region Neutral.

### Пропоноване рішення

Founding **продовжується**, якщо founder лишається валідним.

Сам факт переходу Owned -> Neutral не cancel-ить і не reset-ить progress. Далі застосовуються звичайні Neutral Founding conditions: Neutral Defense має бути 0 для progress, foreign blocking Camp-presence pause-ить process тощо.

---

## 13. ResourceSiteUpgrade: звідки списується upfront cost

### Невизначеність

Визначено `per-site cost × quantity`, але не зафіксовано, з якого Castle беруться local resources і з якого Player — global resources.

### Пропоноване рішення

На Start:
- Wood / Stone / Iron / Food списуються з Castle, до якого **Region приєднана на момент Start**;
- Coins / Gold / Silver списуються з current owner Player;
- вся mixed cost перевіряється і списується атомарно.

Подальша зміна owner, Castle association або neutralization нічого не повертає й не переносить cost.

---

## 14. Allow Transit fallback: чи можна змінювати його під час Active CombatSituation

### Невизначеність

Fallback читається на Battle Start, але не сказано прямо, чи owner може змінити `Region.allow_transit` протягом Dt між Start і Battle Start.

### Пропоноване рішення

Дозволити зміну `allow_transit` до Battle Start за звичайними ownership rules.

Якщо defender уже зробив explicit immutable `Allow/Fight` decision, fallback більше не впливає на цю CombatSituation. Якщо explicit decision немає — на Battle Start використовується актуальне значення `allow_transit`.

---

## 15. Movement target-opponent refresh під час combat command-lock

### Невизначеність

Зміна route під command-lock заборонена. `user_refresh_target_opponent()` не змінює поточну CombatSituation, але може змінити майбутні interaction після виходу з неї. Не визначено, чи ця дія дозволена під час lock.

### Пропоноване рішення

Під час будь-якого combat command-lock заборонити і `user_refresh_target_opponent()`.

Після зняття lock Player може виконати refresh перед наступним border entry. Це тримає command-lock простим: під час unresolved CombatSituation Army не приймає звичайних movement-planning commands, крім прямо дозволених combat-specific decisions.

---

## 16. Ready unnamed Knight і Palace slot

### Невизначеність

Knight ще не існує до отримання імені, але треба явно визначити, чи ready unnamed entry вже резервує Palace capacity.

### Пропоноване рішення

Кожний елемент `ready_knights_awaiting_name` резервує рівно один конкретно доступний Palace slot у сенсі capacity, хоча сам Knight як entity ще не створений.

Тобто відкладання імені не створює додаткової вільної capacity і не може призвести до появи більшої кількості active Knight, ніж Palace level.

---

# Уже визначено під час поточного аудиту

Наступні питання більше не є відкритими і в список на затвердження не входять:

- casualty overkill конкретного Unit перерозподіляється між іншими Unit тієї самої side, а не губиться;
- Commander можна змінювати під час звичайного Movement, але не під час Regrouping;
- під час Regrouping effective Defense Loss Threshold = 0, а без legal Retreat = 100%; persistent value зберігається;
- готові неназвані Knight від Palace upgrade і KnightReplacement використовують одну FIFO-чергу;
- завершений KnightReplacement, що чекає імені, не блокує timer наступного replacement.
