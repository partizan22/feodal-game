# Невизначеності правил — аудит 3

Тимчасовий review-файл. Не змінює чинні правила `docs/01_GAME_RULES_V1.md`. Пункти, що залежать від зміни ситуації через завершення Castle Founding, навмисно не пропонуються до затвердження зараз.

## 1. Створення Castle і вже розпочатий чужий Transit

**Питання.** Після Founding у Neutral Region чужа Army може перебувати у Transit. Відомо, що вона може завершити вже розпочатий Transit, але не визначено, чи може вона змінити маршрут, перейти в Camp або повернутися після завершення цього транзиту.

**Пропозиція.** Залишити відкритим до окремого рішення про зміну ситуації через Founding. Не переносити в RULES.

## 2. Доля власного Army/Unit у Castle Region після знищення всіх Knight

**Питання.** Army припиняє існування, якщо в ній не лишилося живих Knight. Але чи можуть Soldier пережити загибель усіх Knight, правила забороняють лише загибель Knight за наявності живих Soldier у *його Unit*. Для Army з кількома Unit теоретично може лишитися Soldier в інших Unit; не визначено, чи такі Soldier зникають, повертаються до reserve або стають окремою Army.

**Пропозиція.** Оскільки кожний Unit містить Knight і Knight не може загинути, доки в його Unit лишається Soldier, ситуація Army без живих Knight із живими Soldier неможлива. Прямо зафіксувати цю інваріанту, без нової механіки.

## 3. Knight replacement після Founding і ready unnamed slots

**Питання.** Founder переноситься з home Castle у новий Castle; у старому запускається KnightReplacement. Якщо в старому Castle вже є готові неназвані Knight, чи може replacement запускатися паралельно і як обліковується Palace capacity, щоб не створити більше Knight, ніж slots?

**Пропозиція.** Усі occupied living Knight, ready unnamed entries і active/pending replacement reservations враховуються як різні стани одного Palace slot; transfer Founder звільняє рівно один slot, replacement займає саме його. Нові ready entries не створюють додаткових slots.

## 4. Castle Food при кількох одночасних джерелах витрат

**Питання.** Коли Food запас = 0, виробництво недостатнє для Population, reserve Soldier і Camp Army, правила визначають загальну компенсацію Coins, але не визначають, як обчислюється дефіцит між цими групами і чи виникають окремі penalties.

**Пропозиція.** Підсумовувати всі Castle-level Food consumption і всі доступні Castle-level Food flows; непокритий загальний дефіцит конвертувати у Coins один раз. Пріоритетів і додаткових penalties між групами немає.

## 5. Upkeep Soldier при переході між Castle/Camp/Movement посеред інтервалу

**Питання.** Для Coin upkeep можливі різні ставки за location. Не визначено, як враховувати миттєву зміну location під час поточного періоду нарахування.

**Пропозиція.** Нарахування безперервне в Game Time за поточним location; до переходу діє стара ставка, після переходу нова, без перерахунку минулого.

## 6. Зміна складу Army з Unit у Castle та Camp одночасно

**Питання.** У Castle Region Unit в Castle і Camp можуть бути в одній Army. Якщо Player дає Movement order усій Army, чи застосовується до Unit у Castle окрема перевірка Barracks/перехід і чи можуть вони стартувати синхронно?

**Пропозиція.** Movement стартує синхронно для всіх Unit; Unit у Castle миттєво залишають Castle і звільняють Barracks capacity. Ніякого додаткового Dt чи окремої перевірки на вихід немає.

## 7. Casualty rounding і Soldier Type при надлишку budget

**Питання.** Визначено перерозподіл overkill між Unit, але не визначено, що робити з надлишком budget конкретного Soldier Type, якщо всі Soldier цього Type загинули, а в Unit лишилися інші Type. Втрата budget суперечила б загальному правилу нормалізації.

**Пропозиція.** Перерозподіляти надлишок між іншими Soldier Type того самого Unit; після вичерпання всіх Soldier залишок може застосовуватися до Knight. Якщо Unit повністю вичерпав доступну Casualty Health, надлишок переходить до інших Unit side.

## 8. Combat при обох threshold = 0 і legal Retreat

**Питання.** Правила передбачають pre-battle Retreat для Army/side із threshold 0, але не визначають порядок, коли і Attacker, і всі Defender мають threshold 0. Чи вважається хтось переможцем, чи обидві сторони відступають?

**Пропозиція.** Обидві сторони виконують pre-battle Retreat у межах одного Battle Start без combat calculation, casualties, XP і переможця. Destination кожної side обчислюється незалежно.

## 9. Combat, якщо всі Defender виконали pre-battle Retreat

**Питання.** Після виключення zero-threshold Army Defender side може стати порожньою. Не визначено, чи це перемога Attacker і чи виконується Occupation/Camp arrival без battle.

**Пропозиція.** Combat calculation не проводиться; Attacker продовжує свій початковий Camp/Transit context так, ніби opposition не залишилося. Для Camp-bound Attacker після Camp arrival застосовуються звичайні Occupation rules; XP за неіснуючий battle немає.

## 10. Combat tie reroll без гарантії завершення

**Питання.** При одночасному досягненні thresholds повторно генерується Luck Factor. За деяких threshold і функцій (наприклад, однакові threshold=100% та однакові темпи втрат) tie може бути неминучим для всіх допустимих Luck Factor. Тоді reroll нескінченний.

**Пропозиція.** Якщо tie неможливо усунути жодним допустимим Luck Factor, застосувати deterministic tie-breaker на основі повного combat seed; це дає одного переможця і програвшого без нескінченного циклу. Якщо tie усувний — звичайний reroll.

## 11. Player-vs-player Attack у Neutral Region при двох власних Army

**Питання.** Explicit Attack ініціює конкретна Army. Чи інші Army того самого attacking Player у цьому ж Camp автоматично беруть участь? Правила фіксують рівно одну attacking Army, але варто закріпити наслідок для Army-союзників.

**Пропозиція.** Лише Army, що ініціювала explicit Attack, є Attacker; інші Army того самого Player не беруть участі автоматично. Для їхньої атаки потрібна окрема CombatSituation у FIFO.

## 12. Retreat у Neutral Region із Camp третього Player

**Питання.** Neutral Retreat заборонено, якщо там є local presence opponent, якому сторона програла. Чи дозволено Retreat у Neutral Region з Camp третього Player, і чи створює це combat/блокує arrival?

**Пропозиція.** Дозволено: третій Player не є opponent цього battle. Retreat entry не створює CombatSituation, а Camp у Neutral Region допускає кілька Player одночасно.

## 13. Retreat destination для Defender у Neutral Region без entry direction

**Питання.** Для звичайного territorial combat геометрія Defender Retreat залежить від напряму входу Attacker. Якщо CombatSituation виникла через зміну Occupation, коли attacking Army уже перебувала в Region, не завжди очевидно, який entry vector використовувати.

**Пропозиція.** Зберігати для кожної Camp-bound Army останній border-entry direction до цієї Region і використовувати його для Retreat geometry відповідної CombatSituation. Якщо такого напряму принципово немає (наприклад, початкове розміщення), розглядати всі шість сусідів.

---

Пункти 1 та інші випадки, де нове Castle змінює вже наявні interactions, залишаються окремою відкритою темою; їх не слід переносити до чинних правил без нового рішення.
