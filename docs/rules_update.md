# Game Rules — pending update

Цей файл містить погоджені зміни до `01_GAME_RULES_V1`. Після перевірки зміни мають бути інтегровані в основний файл правил.

## 1. Food consumption та утримання військ

### Загальні правила

- Один Soldier незалежно від типу споживає однакову кількість Food.
- Один Knight також рахується як одна одиниця споживання Food.
- Soldier мають окреме постійне споживання Coins, яке не залежить від Food.
- Компенсація нестачі Food у Coins додається до звичайного Soldier Coin upkeep і не замінює його.
- Курс компенсації `Coins / Food` є фіксованим і задається в game configuration.

### Army у Camp

Army у Camp споживають Food, вироблену в Region, де знаходиться Camp.

Якщо локального виробництва достатньо для всіх військ у Camp, усе їх Food consumption покривається локальним виробництвом.

Якщо локального виробництва недостатньо, доступна Food розподіляється між Player пропорційно Food consumption їхніх військ у Camp. Жоден Player або Army не має пріоритету. Непокрита частина Food consumption кожного Player компенсується Coins за фіксованим курсом.

Розподіл виконується між Player, а не між окремими Army. Food consumption одного Player у Region визначається сумарною кількістю його Soldier та Knight у Camp.

### Castle Region

Для Region, у якій знаходиться Castle, діє виняток.

Food consumption військ у Camp Castle Region не віднімається від локального production самої Region. Уся позитивна Food production Castle Region надходить у Castle.

Food consumption усіх військ у Camp Castle Region враховується безпосередньо в Food balance Castle. Воно може покриватися поточним запасом та надходженнями Food Castle.

Якщо Food у Castle недостатньо і запас вичерпаний, непокрита частина consumption компенсується Coins власника Castle за тим самим фіксованим курсом.

Knight `location_state` (`Castle` / `Camp`) не впливає на це правило: з точки зору Food усі війська Army, що знаходиться в Camp Castle Region, споживають Food через Castle.

### Інші Owned Region

Castle не постачає Food військам у приєднаних Region.

Позитивний залишок Food production Region після локального Camp consumption може надходити до Castle з урахуванням distance coefficient. Негативний баланс Region до Castle не передається: нестача Food військ у цій Region компенсується Coins.

Таким чином, distance coefficient застосовується до надлишку Food, що доставляється з Region у Castle, але не використовується для постачання військ із Castle.

### Neutral / foreign Region

Food consumption Camp покривається локальним production Region. Якщо production недостатньо, дефіцит розподіляється між Player пропорційно Food consumption їхніх військ і для кожного Player компенсується Coins.

### Army у Movement

Army у Movement не споживає Food з жодної Region або Castle.

Усе її Food consumption повністю переводиться у Coin consumption за тим самим фіксованим курсом незалежно від того, через власну, чужу чи Neutral Region вона рухається.

---

## 2. City Wealth після Raid

City має дві складові Wealth:

- `wealth` — повний економічний потенціал City;
- `active_wealth_ratio` — активна частка Wealth від `0` до `100%`.

У нормальному стані `active_wealth_ratio = 100%`. Поки Region належить Player і City повністю відновлене, `wealth` росте за звичайними правилами.

Після будь-якого успішного Raid:

- `active_wealth_ratio` скидається до `0` незалежно від його попереднього значення;
- `wealth` не зменшується;
- поки `active_wealth_ratio < 100%`, `wealth` не росте;
- `active_wealth_ratio` поступово відновлюється до `100%` зі своїм recovery rate;
- recovery `active_wealth_ratio` не залежить від того, чи належить Region Player.

Коли `active_wealth_ratio` досягає `100%`, City автоматично повертається до звичайного режиму росту `wealth`; окремий стан City для цього не потрібний.

Поточний економічний ефект City та ефективність Raid визначаються активним Wealth, тобто `wealth × active_wealth_ratio`.

---

## 3. City Defense

City має окрему силу захисту — **City Defense**.

`City Defense = f(wealth)`, де конкретна функція задається в game configuration.

City Defense:

- залежить від повного `wealth`, а не від `active_wealth_ratio`;
- не зменшується внаслідок combat;
- не залежить від попередніх Raid;
- існує як у Neutral City, так і в City, Region якого належить Player.

Тому City одразу після Raid має ту саму City Defense, що й перед Raid, хоча його активний Wealth і потенційна здобич від наступного Raid можуть бути значно меншими.

---

## 4. City Raid

Raid є локальною дією: атакуючі війська повинні перебувати в Camp тієї самої Region, де знаходиться City.

При Raid defender — тільки City Defense. Neutral Defense Region не бере участі в захисті City.

Для City Defense у Raid використовується фіксований loss threshold, заданий у game configuration (орієнтовно 30–50%).

Після успішного Raid:

- City Defense не змінюється;
- `wealth` не змінюється;
- `active_wealth_ratio` скидається до `0`;
- Player отримує Raid reward, величина якого визначається активним Wealth до Raid.

У Neutral Region можна зайти в Camp без бою з Neutral Defense, після чого окремо атакувати City Defense з метою Raid.

У Region іншого Player спочатку потрібно перемогти війська owner та отримати можливість перебувати в Camp. Лише після цього City можна атакувати окремою Raid-дією. City Defense не бере участі в попередньому бою за Region.

---

## 5. Neutral Defense та City

При звичайному вході Army у Camp Neutral Region автоматичного бою немає.

Neutral Defense атакується окремою дією, якщо Player хоче виконати умову для подальшої Annexation Neutral Region.

Якщо Neutral Region містить City, defensive strength при такій атаці складається з:

- поточної Neutral Defense;
- повної City Defense.

Для **обох** складових у цьому combat використовується loss threshold `100%`.

Якщо attacker перемагає:

- Neutral Defense вважається знищеною;
- поточна Neutral Defense стає `0`;
- City Defense не змінюється;
- City Defense не потрібно перемагати повторно для Annexation;
- подальша готовність до Annexation визначається звичайними правилами контролю Region.

City Defense додається до Neutral Defense саме як додаткова складність захоплення Neutral Region із City. Вона не стає окремою перешкодою для Annexation після перемоги в цьому combat.

Для Region, що вже належить Player, City Defense не бере участі в обороні Region. При вторгненні її захищають війська owner за звичайними combat rules. Для подальшої Annexation чужої Owned Region окремо перемагати City Defense не потрібно.

---

## 6. Neutral Defense recovery

Neutral Defense після знищення відновлюється **поступово**, а не одразу після завершення періоду recovery.

Recovery починається тільки після того, як війська **всіх Player** залишили Region.

Поки в Region присутні війська хоча б одного Player, Neutral Defense не відновлюється.

Після виходу останніх військ Neutral Defense починає рости від `0` до свого нормального значення. Відновлення є неперервним: одразу після початку recovery і проходження ненульового Game Time Neutral Defense вже стає більшою за `0`.

Якщо війська знову входять у Region до повного відновлення:

- recovery припиняється;
- зберігається вже відновлене поточне значення Neutral Defense;
- нова атака на Neutral Defense використовує саме це поточне значення;
- якщо в Region є City, до нього знову додається повна City Defense.

Після наступного виходу всіх військ recovery продовжується від поточного значення до нормальної сили Neutral Defense.

Нормальна сила Neutral Defense та її recovery rate визначаються game configuration / параметрами Region.