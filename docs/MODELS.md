# Backend models

Робочий опис backend game-logic моделей. Для кожної Model тут фіксуються:

- прямі характеристики;
- динамічні характеристики;
- обчислювальні характеристики;
- user methods, що реалізують основну логіку відповідних user actions;
- внутрішні domain methods, які використовуються іншими game methods;
- triggers;
- `check_trigger_*()` та `on_trigger_*()` для кожного trigger-а.

Root model для user GameEvent поки не визначена до проєктування взаємодії з frontend/API. Тому `user_*` тут означає Model, яка реалізує основну логіку дії, а не обов'язково майбутню root model. Після визначення root model префікси методів за потреби будуть змінені.

Обчислювальна характеристика, позначена `*` перед назвою, є **system-computed**: її актуальне значення не обчислюється прямо в рамках game logic самої Model, а надається infrastructure. Це не окремий persistence type: як і інші computed characteristics, system-computed значення зберігається при `commit()`. Типові приклади — reverse relationships та топологічні зв'язки карти. `[]` у назві означає колекцію; для system-computed relationship це список посилань на Model, для звичайної computed characteristic це може бути список scalar values.

Для computed characteristics діє правило: якщо `A.computed` потрібне значення з `A -> B -> C`, відповідна інформація зазвичай піднімається в computed characteristic Model `B`, а `A` читає вже її. Прямий system-computed зв'язок `A -> C` додається лише коли він сам є природною характеристикою `A`.

Це обмеження не поширюється на domain/user methods: method може обходити потрібні relationships і читати характеристики пов'язаних Model без створення проміжних computed characteristics лише заради такого обходу.

Рівні всіх Building одного Castle зберігаються як одна характеристика `levels`.

Wood / Stone / Iron зберігаються як одна характеристика `resources` — один value object / helper class із трьома значеннями.

Food consumption одного Soldier не залежить від Soldier Type. Один Knight також рахується як одна food-consumption unit. Компенсація нестачі Food у Coins використовує один глобальний конфігураційний курс `coins_per_food`.

`Dt` — одна базова game-time константа для фіксованих просторових і бойових інтервалів: вхід у Region -> Camp, Camp -> вхід у сусідню Region, початок CombatSituation -> Battle Start (включно з player-vs-player атакою з Camp у Neutral Region; це не окремий додатковий interval), звичайний player-vs-player Retreat рух до Camp-Regrouping (включно з pre-battle Retreat), Regrouping та мінімальний інтервал між послідовними player-vs-player боями в одній Region. Базовий Transit однієї Region також дорівнює `Dt`, але в майбутньому тільки звичайний Transit без CombatSituation може отримати speed coefficient. Attack на Neutral Defense та City Raid відбуваються без `Dt`-черги; поразка від Neutral Defense або City Defense не має окремого post-defeat `Dt` до Regrouping.

У секціях methods:

- **Використовує** — характеристики цієї Model та relationships/характеристики пов'язаних Model, які потрібні method.
- **Викликає** — інші domain methods, якщо вони потрібні. Характеристики іншої Model напряму не змінюються.

## Player-specific projections

Між backend game-world models і frontend вводиться окремий шар **player-specific projections**. Frontend не отримує domain models напряму: projection збирає з них інформацію, доступну конкретному Player, застосовує правила видимості/доступу та є джерелом даних і оновлень для frontend subscriptions.

Конкретний набір projection models, їх characteristics, lifecycle, dependencies та формат frontend subscriptions/updates буде визначено окремо під час проєктування frontend/API.

---

# Model files

Опис моделей розділений за логічними блоками. Нумерація секцій моделей збережена, щоб старі посилання на номери не змінювали зміст.

- [MODELS_CORE.md](MODELS_CORE.md) — `Player`, `Castle`, `Region`, `City`.
- [MODELS_MILITARY.md](MODELS_MILITARY.md) — `Knight`, `Army`, `CampInRegion`, `Movement`.
- [MODELS_COMBAT.md](MODELS_COMBAT.md) — `CombatSituation` і combat lifecycle.
- [MODELS_PROCESSES.md](MODELS_PROCESSES.md) — `CastleFounding`, `Recruitment`, `BuildingUpgrade`, `ResourceSiteUpgrade`, `KnightReplacement`.

Цей файл є entry point і містить спільні conventions; детальний опис конкретних Model знаходиться у файлах вище.
