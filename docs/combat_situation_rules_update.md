# Combat Situation rules update

Цей файл фіксує оновлені правила участі Army у player-vs-player CombatSituation, pre-battle decisions та черги CombatSituation. Це правила гри; деталі backend implementation тут не визначаються.

## 1. Базові поняття

У правилах нижче `Regrouping` Army для всіх перевірок presence прирівнюється до Army у `Camp`. Єдині спеціальні обмеження Regrouping Army: вона не може починати Movement і не може ініціювати Attack.

Одна CombatSituation завжди має рівно **одну attacking Army**. Декілька Army одного Player не об'єднуються в одну attacking side. Якщо дві Army одного Player послідовно створюють підставу для combat у тій самій Region, реєструються дві окремі CombatSituation.

Для defender side одна CombatSituation може включати декілька Army одного Player.

`Dt` — базовий час підготовки до player-vs-player battle. Від `CombatSituation Start` до `Battle Start` завжди проходить рівно `Dt`.

## 2. CombatSituation Registration, Start та Active state

### CombatSituation Registration

`CombatSituation Registration` — момент, коли виникає підстава для майбутнього player-vs-player combat.

CombatSituation реєструється, коли:

- Army Player A входить у Region Player B, незалежно від наявності там Army B або Army третіх Player;
- Army Player A входить у власну Region, яка Occupied іншим Player;
- Region Player A стає Occupied, коли Army A уже рухається в цій Region, увійшла до Occupation, але запізно для участі в defense попередньої CombatSituation;
- Army Player A, що знаходиться в Camp Neutral Region разом з Army Player B, ініціює Attack на Player B.

На етапі Registration фіксується тільки attacking Army. Не фіксуються:

- defending Player;
- potential defenders;
- defensive composition;
- aggressive/non-aggressive Transit classification.

Якщо attacking Army рухалась, її початкове завдання `Camp` або `Transit` зберігається.

### CombatSituation Start

`CombatSituation Start` — момент, коли зареєстрована CombatSituation починає фактичну pre-battle phase.

Якщо в Region немає іншої конкуруючої Active CombatSituation, Start відбувається одразу після Registration.

Якщо в Region уже є конкуруюча CombatSituation, нова CombatSituation стає в чергу. Її Start відбувається після завершення попередньої CombatSituation, у тому числі після її дострокового завершення без battle.

Саме на Start за актуальним станом Region визначаються:

- чи CombatSituation взагалі лишається чинною;
- defending Player;
- potential defenders;
- для Transit — aggressive або non-aggressive mode;
- pre-battle options сторін.

У всіх правилах участі defenders нижче формулювання «на момент входу attacker» слід читати як **«на момент CombatSituation Start»**.

### Active CombatSituation

`Active CombatSituation` — CombatSituation від її Start до завершення.

Від Start до Battle Start проходить `Dt`, якщо CombatSituation не завершується достроково.

## 3. Army у черзі CombatSituation

Якщо CombatSituation зареєстрована, але ще не почалася через наявність попередньої CombatSituation, attacking Army переходить у стан очікування combat.

Поки вона очікує:

- її рух призупиняється, навіть якщо початковим завданням був Transit;
- вона не може отримувати нові orders;
- вона не може стати attacking або defending participant іншої CombatSituation;
- defending Player і defenders для її CombatSituation ще не визначаються.

Після завершення попередньої CombatSituation queued CombatSituation переходить до Start і оцінює Region за актуальним станом.

Якщо на Start в Region більше немає підстав для player-vs-player combat — наприклад, немає іншого Player або Region стала Neutral — CombatSituation завершується без battle. Attacking Army продовжує своє початкове Movement/order.

## 4. Transit classification на CombatSituation Start

Початкове завдання Army `Camp` або `Transit` є фіксованим і під час очікування CombatSituation не змінюється.

Якщо початкове завдання — `Transit`, на CombatSituation Start Transit класифікується як aggressive або non-aggressive за актуальним станом Region.

Transit є **Aggressive Transit**, тільки якщо одночасно виконані обидві умови:

1. Final Region маршруту не Neutral, і в ній на цей момент або є війська Player A, або вона належить Player A та не Occupied.
2. Current Region не Neutral, і в ній є війська Player A.

Якщо хоча б одна умова не виконується, Transit є non-aggressive.

## 5. Вхід Army A у Region Player B: коли battle не відбувається

### 5.1. Transit без локального defense presence

Якщо attacking Army має завдання Transit, а на CombatSituation Start у Region немає Army Player B:

- у Camp/Regrouping;
- або тих, що вже увійшли в Region з метою Camp,

battle не відбувається.

Attacking Army продовжує Transit. Army B, які самі перебувають у Transit, також продовжують Transit. Player B не отримує pre-battle decision dialog.

### 5.2. Non-aggressive Transit при наявності Camp defense

Якщо attacking Army має **non-aggressive Transit**, а Player B має Army у Camp/Regrouping або Army, що вже увійшли в Region з метою Camp, Player B спочатку отримує вибір:

- `Пропустити`;
- `Прийняти бій`.

Після ручного вибору рішення фіксується і не може бути змінене.

Якщо вибрано `Пропустити`, CombatSituation завершується без battle. Attacking Army продовжує Transit; стан Army B не змінюється.

Якщо вибрано `Прийняти бій`, CombatSituation залишається Active.

Якщо Player B не зробив ручний вибір до Battle Start, використовується `allow_transit`:

- `allow_transit = true` → battle не відбувається, attacker продовжує Transit, усі Army B залишаються у своїх поточних станах;
- `allow_transit = false` → battle відбувається за звичайними правилами.

## 6. Potential defenders при attack у Owned Region

Якщо CombatSituation не завершилася за правилами розділу 5, potential defenders визначаються за станом Army Player B на CombatSituation Start.

### Army, які можуть бути defenders

- Army у Camp/Regrouping.
- Army, яка вже увійшла в Region з метою Camp. Вона гарантовано досягає Camp не пізніше Battle Start, оскільки `entry -> Camp = Dt` і `CombatSituation Start -> Battle Start = Dt`.

### Army, які не є defenders

- Army, яка до CombatSituation Start уже вийшла з Camp і почала Movement.
- Army, яка до CombatSituation Start увійшла в Region з метою Transit, якщо вона не була окремо залишена для defense за правилами Active CombatSituation.
- Army, яка входить у Region після CombatSituation Start.

Army, яка входить після CombatSituation Start, не може долучитися до вже Active CombatSituation незалежно від своєї початкової мети.

## 7. Defensive decisions до Battle Start

Для Army, які є potential defenders, Player B протягом `Dt` може визначити:

- які Army залишаються для battle;
- які Army retreat без battle;
- один спільний `Defense Loss Threshold` для всіх Army, які залишаються для battle.

Retreat задається для цілих Army. Не дозволяється ділити Army або змінювати її внутрішній склад для цього рішення.

Після CombatSituation Start potential defender Army не може:

- почати звичайний Movement;
- merge/split;
- змінювати Commander;
- змінювати склад Unit/Army.

Вона може тільки залишитися для battle або retreat за pre-battle decision.

Retreat не виконується в момент вибору. Він застосовується при завершенні/розв'язанні CombatSituation. Така Army не отримує combat casualties і після retreat проходить Regrouping за звичайними правилами.

## 8. Default behavior, якщо Defender не прийняв рішення

Якщо до Battle Start Player B не зробив потрібних ручних рішень:

- Army B, що перебувають у Transit, продовжують Transit і не беруть участі в battle;
- potential defender Army, для яких Defense Loss Threshold дорівнює `0%`, retreat без combat casualties і після цього проходять Regrouping;
- potential defender Army з Defense Loss Threshold `> 0%` беруть участь у battle.

Для non-aggressive Transit при `allow_transit = true` спочатку діє правило автоматичного пропуску з розділу 5.2, тому battle взагалі не починається.

## 9. Army Defender, які входять після CombatSituation Start

Якщо Army Player B входить у Region після CombatSituation Start, вона не може приєднатися до цієї CombatSituation.

Подальша поведінка залежить від початкового завдання attacker та результату battle:

### Attacker мав Transit

Незалежно від результату battle Army B, які увійшли після CombatSituation Start, продовжують свій запланований рух. Вони можуть у тому числі досягти Camp після завершення battle.

### Attacker мав Camp і програв

Army B, які увійшли після CombatSituation Start, продовжують свій запланований рух, у тому числі до Camp.

### Attacker мав Camp, виграв і Occupied Region

Для Army B, що входять у Region після Occupation, застосовується правило входу у власну Region, Occupied іншим Player: така Army реєструє нову CombatSituation проти occupier незалежно від свого початкового завдання.

Ця нова CombatSituation підпорядковується загальній черзі combat у Region.

## 10. Attack між Player у Neutral Region

Якщо Army Player A та Player B перебувають у Camp однієї Neutral Region і Army A ініціює Attack на B:

- реєструється CombatSituation з однією attacking Army A;
- якщо немає конкуруючої CombatSituation, вона одразу переходить до Start;
- Battle Start настає через `Dt`.

Potential defenders Player B визначаються на CombatSituation Start за тими самими правилами:

- Army B у Camp/Regrouping — potential defenders;
- Army B, яка до Start уже ввійшла в Region з метою Camp — potential defender;
- Army B у Transit або Army, яка вже вийшла з Camp, — не defender;
- Army B, що входить після Start, — не defender.

Player B може наказати retreat усім або вибраним Army та встановити один спільний Defense Loss Threshold для Army, що залишаються.

Після Start potential defender Army не може почати Movement, merge/split, змінювати Commander або склад Army до завершення CombatSituation.

## 11. Attack на occupier

Ті самі правила potential defenders, pre-battle decisions та блокування Army застосовуються до Army, що Occupy чужу Owned Region, коли на них атакує:

- formal owner Region;
- третій Player.

Regrouping Army occupier прирівнюються до Camp Army.

## 12. Мінімальний інтервал і FIFO-черга CombatSituation

У Region не можуть відбуватися дві конкуруючі player-vs-player CombatSituation одночасно.

Якщо нова CombatSituation Registration відбувається, коли в Region уже є Active CombatSituation, нова CombatSituation стає в FIFO-чергу.

Якщо під час очікування виникає ще одна CombatSituation, вона стає після вже зареєстрованих. Пріоритетів немає; порядок визначається порядком Registration.

Після завершення Active CombatSituation наступна queued CombatSituation отримує Start саме в цей момент. Її pre-battle interval знову становить повний `Dt`.

Таким чином мінімальний інтервал між Battle Start конкуруючих CombatSituation в одній Region забезпечується послідовною активацією ситуацій і повним `Dt` pre-battle interval для кожної.

Queued CombatSituation до Start не має defending Player або potential defenders; усе це визначається заново на Start.

## 13. Незалежні CombatSituation та спеціальні combat

Обмеження FIFO-черги не застосовується до незалежних combat у Neutral Region, якщо вони не конфліктують між собою, зокрема до player-vs-player combat між різними парами Player.

Attack на Neutral Defense та City Raid:

- відбуваються без `Dt` pre-battle preparation;
- не стають у FIFO-чергу player-vs-player CombatSituation;
- можуть відбуватися незалежно від інших не пов'язаних special combat.

Однак Player не може ініціювати Attack на Neutral Defense або City Raid Army, яка:

- є attacker будь-якої незавершеної CombatSituation, включно з queued;
- є potential defender Active CombatSituation.

До CombatSituation Start queued situation не має potential defenders, тому сама можливість стати defender у майбутньому не створює такого обмеження.

## 14. City Raid defeat

При програші City Raid attacking Army не retreat у сусідню Region.

Вона залишається в тому самому CampInRegion і переходить у Regrouping на `Dt`. Під час Regrouping вона продовжує рахуватися звичайною Camp presence для Annexation, Occupation, Food consumption та defense, але не може рухатися або атакувати.
