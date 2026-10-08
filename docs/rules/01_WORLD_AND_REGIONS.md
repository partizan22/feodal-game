# Феодали — Світ і області

## Версія 1

# 2. Стани області та територіальний контроль

Region може бути в одному з трьох базових станів:

- нейтральна (Neutral);
- належить гравцю (Owned);
- окупована (Occupied).

## 2.1 Neutral

Neutral Region не має власника.

У ній можуть одночасно перебувати в Camp війська різних гравців. Сам факт такої присутності не запускає бій.

Транзит через Neutral Region завжди дозволений і не може бути заблокований іншими гравцями.

Neutral Region може містити ресурси, Food production, Neutral Defense та City.

## 2.2 Owned

Owned Region формально належить одному гравцю та прив'язана до конкретного його Castle.

Її економічні потоки надходять до цього Castle, крім глобальних ресурсів.

Власник сплачує регулярне утримання Region. Розмір утримання залежить від відстані до Castle через налаштовувану функцію.

У V1 `distance` між звичайною Region і її Castle — це пряма стандартна відстань на hex-grid між hex цієї Region і hex Castle Region, а не довжина територіального шляху. Для Castle Region `distance = 0`, для сусідньої Region `distance = 1`.


## 2.3 Occupied

Occupied Region формально продовжує належати попередньому власнику, але в її Camp є війська одного ворожого Player — поточного occupier.

Під час Occupation:

- формальний owner не змінюється і продовжує сплачувати утримання Region;
- Region не дає формальному owner нормального економічного потоку;
- occupier не отримує її економічний потік до Annexation;
- Occupation припиняється в момент, коли **остання Army occupier залишає Camp**, а не лише після її виходу за межі Region; контроль одразу повертається формальному owner.

У Region не може одночасно бути кілька occupier. Якщо третій Player входить у цю Region з локальною метою Camp, він взаємодіє з поточним occupier; після перемоги й переходу в Camp він стає новим occupier. Якщо перемагає формальний owner, Occupation припиняється й відновлюється звичайний Owned state.

Transit через Occupied Region не створює CombatSituation незалежно від того, хто проходить Region, включно з формальним owner. Для такого Transit не застосовуються Allow Transit та Aggressive/NonAggressive classification.

Якщо формальний owner входить у свою Occupied Region з локальною метою **Camp**, він атакує поточного occupier.

---

# 3. Територіальна зв'язність

Звичайна Region може бути Annexed до Castle лише тоді, коли вона межує принаймні з однією Region, яка вже належить цьому самому Castle. Таким чином усі звичайні володіння Castle повинні утворювати безперервний ланцюг до Castle Region.

Якщо проміжна Owned Region стає Occupied, усі Owned Region за нею, що втратили безперервний шлях до Castle, не втрачаються юридично, але їх Wood/Stone/Iron/Food потоки та Gold/Silver income припиняються, доки зв'язок не відновиться.

Якщо проміжна Region остаточно втрачається власником, усі його Region, які після цього не мають безперервного ланцюга власних Region до свого Castle, також автоматично втрачаються і стають Neutral.

При такій автоматичній втраті:

- рівні ResourceSite зберігаються;
- Wealth City, якщо City є, зберігається, але перестає зростати, поки Region Neutral;
- active ResourceSiteUpgrade не скасовуються і продовжуються;
- Neutral Defense надалі відновлюється за звичайними правилами Neutral Region;
- війська колишнього власника, які вже перебувають у Camp у цій Region, залишаються на місці й надалі вважаються військами в Camp у Neutral Region.

Player може миттєво й безкоштовно добровільно відмовитися від будь-якої своєї звичайної Region, але не від Castle Region. Для цього не потрібна присутність Army або Knight. Єдиний спеціальний blocker — CombatSituation зі status `Active` у самій Region. `Registered` situation існує в queue тільки тоді, коли інша situation цієї Region уже Active, тому окремого стану «queued без Active» у нормальному lifecycle немає.

Після відмови Region одразу стає Neutral, її City Wealth і рівні ResourceSite зберігаються, active ResourceSiteUpgrade продовжуються, а війська, що вже перебувають у ній, залишаються на місці за правилами Neutral Region. Якщо відмова остаточно розриває territorial connection інших Region цього Castle, вони також автоматично стають Neutral за правилами вище. Уже запущений Annexation іншого Player від самої відмови не скасовується. Власний active CastleFounding також не cancel-иться і не reset-иться лише через `Owned -> Neutral`, якщо founder лишається валідним; далі для progress застосовуються звичайні умови Founding у Neutral Region, включно з `Neutral Defense == 0` і pause через blocking foreign Camp-presence.

Castle Region є винятком із звичайних правил війни у V1: її не можна атакувати і через неї не можна прокладати новий ворожий Transit. Якщо чужа Army вже почала Transit через Region до моменту завершення Founding Castle, вона має право завершити цей уже розпочатий Transit.

---

# 26. Neutral Defense

Neutral Defense — абстрактна поточна сила Neutral Region, а не Army.

Вхід Army у Camp Neutral Region сам по собі не запускає battle. Neutral Defense атакується окремою миттєвою дією конкретної Army у Camp; Regrouping Army не може її ініціювати. Ця дія не створює CombatSituation і не входить у player-vs-player FIFO.

Якщо в Region є City, defender strength при атаці Neutral Defense дорівнює:

current Neutral Defense + full City Defense

Для обох складових defender threshold дорівнює 100%. Attacker використовує Target Combat Threshold.

При перемозі Attacker поточна Neutral Defense стає 0; City Defense не змінюється.

При поразці часткові втрати Neutral Defense не зберігаються: після combat вона лишається на тому самому pre-combat current value. City Defense також не змінюється. Attacker лишається у своєму Camp і одразу починає Regrouping на Dt.

Neutral Defense recovery є поступовим. Recovery rate ненульовий тільки коли Region Neutral, current strength нижча за full strength і в Region немає **жодної фізично присутньої Army** будь-якого Player.

Будь-яка Army у Camp, Regrouping або Movement всередині Region pause-ить recovery. Після виходу останньої Army recovery продовжується від уже досягнутого current value, а не починається заново.

Player не може ініціювати Attack Neutral Defense у Region, якщо він є attacker будь-якої unresolved CombatSituation у цій Region або potential defender Active CombatSituation.

---

# 29. City

City є об'єктом Region і має дві економічні характеристики:

- wealth — повний довгостроковий економічний потенціал;
- active_wealth_ratio — активна частка від 0 до 1.

effective_wealth = wealth × active_wealth_ratio.

Коли active_wealth_ratio == 1, Region є Owned non-Occupied і Player не має empty_coins, wealth поступово зростає. Territorial connection для самого Wealth growth не потрібен: disconnected Owned non-Occupied City продовжує локально нарощувати Wealth, хоча recurring Coin income з disconnected Region не надходить Player до відновлення connection.

Після успішного Raid wealth не зменшується, а active_wealth_ratio скидається до 0. Поки ratio < 1, wealth не росте.

Recovery active_wealth_ratio до 1 відбувається поступово **незалежно від ownership, Occupation та стану Coins Player**.

Recurring City Coin income і Raid reward базуються на effective_wealth. Neutral або Occupied City не дає recurring income формальному owner.

City має окрему City Defense = f(wealth). Вона залежить від повного wealth, не зменшується після Raid і не залежить від active ratio.

---

