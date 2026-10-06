# Феодали — словник термінів

| English term | Український термін | Короткий опис |
|---|---|---|
| Region | область | Базова шестикутна одиниця карти. |
| Neutral Region | нейтральна область | Region без власника. |
| Owned Region | власна область | Region, що формально належить гравцю. |
| Occupied Region | окупована область | Чужа Region під військовим контролем окупанта без зміни формального власника. |
| Castle | замок | Локальний економічний і військовий центр. |
| Castle Region | область замку | Region, у якій розташований Castle. |
| Annexation | приєднання | Остаточне включення Region до володінь Castle. |
| Occupation | окупація | Тимчасовий військовий контроль чужої Region; закінчується, коли остання Army occupier залишає Camp. |
| Region Upkeep | утримання області | Регулярна плата за Owned Region. |
| Territorial Connectivity | територіальна зв'язність | Безперервний ланцюг Owned Region до Castle. |
| ResourceSite | джерело ресурсу | Природне джерело конкретного ресурсу в Region. |
| Resource Potential | ресурсний потенціал | Природна кількість/цінність ResourceSite у Region. |
| Wood | дерево | Базовий локальний ресурс. |
| Stone | камінь | Базовий локальний ресурс. |
| Iron | залізо | Базовий локальний ресурс, важливий для війська. |
| Food | їжа | Локальний ресурс виробництва, зберігання і постійного споживання. |
| Coins | монети | Глобальна валюта гравця. |
| Gold | золото | Глобальний рідкісний ресурс розвитку. |
| Silver | срібло | Глобальний видобувний ресурс, у V1 поки без витрат. |
| Warehouse | склад | Зберігає Wood, Stone та Iron. |
| Granary | амбар | Зберігає Food. |
| Building | будівля | Розвиваний об'єкт Castle. |
| Upgrade | покращення / апгрейд | Підвищення level Building або ResourceSite. |
| Palace | палац | Визначає кількість Knight slots Castle. |
| Governor's House | будинок губернатора | Обмежує кількість зовнішніх Region Castle. |
| Forge | кузня | У V1 передумова для Barracks. |
| Barracks | казарма | Recruitment і місткість для Soldier у Castle. |
| Bank | банк | Множить eligible Coin income. |
| Market | ринок | Економічна Building; повна торгівля відкладена. |
| Population | населення | Показник Castle, що створює Food consumption. |
| Knight | лицар | Індивідуальний військовий юніт і командир. |
| Soldier | солдат | Неіндивідуальна бойова одиниця певного Type. |
| Soldier Type | тип солдата | Набір бойових і економічних характеристик Soldier. |
| Light Infantry | легкий піхотинець | Дешевий універсальний Soldier. |
| Spearman | списник | Defense-oriented Soldier. |
| Swordsman | мечник | Attack-oriented Soldier. |
| Heavy Infantry | важкий піхотинець | Сильний універсальний Soldier. |
| Halberdier | алебардник | Елітний Defense-oriented Soldier. |
| Rider | вершник | Елітний Attack-oriented mounted Soldier. |
| Recruitment | найм | Поступове створення Soldier через Barracks. |
| Recruitment Queue | черга найму | Єдина послідовна черга Recruitment у Castle. |
| Unit | загін | Один Knight з його Soldier або Knight без Soldier. |
| Army | армія | Об'єднання одного або кількох Unit. |
| Commander-in-Chief | головнокомандувач | Knight, що командує всією Army. |
| Home Castle | рідний замок | Castle, до якого структурно прив'язаний Knight/Unit. |
| Camp | кемп / табір | Нерухомий стан війська у Region. |
| Movement | рух / переміщення | Стан проходження Route. |
| Route | маршрут | Кінцева Region і послідовність Region, через які проходить Army. |
| Transit | транзит | Локальний режим проходження проміжної Region без зупинки. |
| Aggressive Transit | агресивний транзит | Transit через Owned non-Occupied Region, formal owner якої збігається з `Movement.target_opponent`, зафіксованим для CombatSituation. |
| Direction | напрямок | Наступна сусідня Region, у яку виходить Army. |
| Allow Transit | дозволити транзит | Fallback rule Owned non-Occupied Region для NonAggressive Transit, якщо defender не зробив ручний Allow/Fight choice до Battle Start. |
| Combat | бій | Аналітична військова взаємодія двох сторін. |
| Dt | базовий часовий інтервал | Єдина V1-константа для Camp↔Region entry, Battle preparation, Retreat-local, Regrouping і базового Transit. |
| CombatSituation | бойова ситуація | Persistent player-vs-player interaction від Registration через Start до Resolved; одна situation має одну attacking Army. |
| CombatSituation Start | початок бойової ситуації | Момент, коли situation стає Active і фіксує актуальний defender context. |
| Battle Start | початок бою | Момент через `Dt` після CombatSituation Start, коли lock-яться combat parameters і, якщо battle досі актуальний, він розраховується. |
| Attacker | атакуючий | Сторона, що ініціює combat або входить у ворожу Region. |
| Defender | захисник | Сторона, що обороняється. |
| Target Opponent | цільовий суперник | Player або `null`, зафіксований у Movement за final Region; може бути явно refreshed і застосовується з наступного Region entry. |
| Target Combat | цільовий бій | Movement-combat, де фактичний defender збігається зі snapshot Target Opponent, а також explicit Camp attack у Neutral Region. |
| Incidental Combat | побічний бій | Movement-combat із Player, який не збігається зі snapshot Target Opponent. |
| Loss Threshold | поріг втрат | Межа втрат, після якої сила припиняє combat. |
| Defense Loss Threshold | оборонний поріг втрат | Persistent threshold окремої defender Army; participating Defender side використовує мінімальне значення. `0` сам по собі не є pre-battle Retreat command. |
| Target Combat Threshold | поріг цільового бою | Loss Threshold Attacker у Target Combat, explicit Camp attack у Neutral Region, атаці Neutral Defense та City Raid. |
| Incidental Combat Threshold | поріг побічного бою | Loss Threshold Attacker у Incidental Combat. |
| Combat Strength | бойова сила | Розрахункова сила Army/Unit у combat. |
| Experience | досвід | Параметр Knight, що впливає на combat coefficient. |
| Luck Factor | коефіцієнт удачі | Обмежена випадкова поправка до співвідношення сил. |
| Casualties | втрати | Фактично загиблі Soldier/Knight після combat. |
| Casualty Health | умовне здоров'я втрат | Внутрішня величина для розподілу casualties, не persistent HP. |
| Loss Budget | бюджет втрат | Загальний обсяг Casualty Health, який треба розподілити після combat. |
| Retreat | відступ | Side-level відхід у одну вибрану сусідню Region; Retreat entry не створює нову CombatSituation, далі `Dt` до Camp і Regrouping. |
| Regrouping | перегрупування | Camp-presence після Retreat/спецпоразки; сама Army не може Move або Attack, але для інших presence rules рахується як Camp. |
| Neutral Defense | нейтральний захист | Абстрактна оборонна сила Neutral Region. |
| City | місто | Об'єкт Region, що має Wealth і Coin economy. |
| Wealth | багатство міста | Параметр City, що визначає income та raid reward. |
| City Raid | рейд / пограбування міста | Локальна дія Army/Unit у Camp у Region з City для отримання Coins та виснаження City. |
| Founding | заснування замку | Процес створення нового Castle у Neutral Region або у власній уже Annexed Region. |
| Founding Knight | лицар-засновник | Knight без Soldier, прив'язаний до Founding до його завершення. |
| Founding Progress | прогрес заснування | Накопичений прогрес Founding до створення Castle. |
| Game Time | ігровий час | Єдина шкала часу всіх процесів гри. |
| AI Player | віртуальний гравець | Програмний гравець для тестування/прототипу. |
