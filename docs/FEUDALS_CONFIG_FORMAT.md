# Феодали — формат конфігурації параметрів

Конфігурація розділена на два JSON-файли:

1. `feudals_starting_parameters.json` — початковий стан нового гравця / стартового Castle: стартові ресурси, рівні будівель, кількість лицарів та інші параметри, які задають саме стартову позицію.
2. `feudals_game_parameters.json` — глобальні параметри правил і балансу: вартості, тривалості, upkeep, production, capacities, коефіцієнти, combat-параметри та функції залежності значень від level/distance тощо.

У стартовий файл не слід дублювати загальні правила. Наприклад, capacity Warehouse визначається `feudals_game_parameters.json`, а стартовий файл лише задає `warehouse: 1`.

## 1. Загальні принципи

- JSON містить лише дані, без PHP/JS-коду.
- Формули задаються декларативними об'єктами, а не рядками з executable expression.
- Для значень, що ще не визначені, використовується `null`.
- Одиниці виміру по можливості вказуються в назві параметра: `_minutes`, `_hours`, `_coins_per_hour` тощо.
- Якщо значення залежить від level/distance/іншої числової змінної, використовується `value rule`.
- Один параметр може комбінувати точні значення та формули на різних діапазонах.

## 2. Value rule

Просте значення може залишатися звичайним JSON number:

```json
{
  "dt_minutes": 5
}
```

Для залежності використовується об'єкт із `type`.

### `table`

Точні значення для дискретних аргументів:

```json
{
  "type": "table",
  "variable": "level",
  "values": {
    "1": 100,
    "2": 150,
    "3": 220
  }
}
```

### `constant`

```json
{
  "type": "constant",
  "value": 100
}
```

### `linear`

`value(x) = value_at_origin + per_step * (x - origin)`

```json
{
  "type": "linear",
  "variable": "level",
  "origin": 20,
  "value_at_origin": 120,
  "per_step": 6
}
```

### `polynomial`

Коефіцієнти йдуть від степеня 0 вгору:

`a0 + a1*x + a2*x^2 + ...`

```json
{
  "type": "polynomial",
  "variable": "level",
  "coefficients": [420, 90, 18, 0.8]
}
```

### `exponential`

`value(x) = value_at_origin * factor^(x - origin)`

```json
{
  "type": "exponential",
  "variable": "level",
  "origin": 1,
  "value_at_origin": 1800,
  "factor": 1.38
}
```

### `points`

Для кривої, заданої контрольними точками. Метод між точками задається явно:

```json
{
  "type": "points",
  "variable": "level",
  "interpolation": "linear",
  "points": {
    "1": 1.1,
    "10": 2.4,
    "20": 3.8
  }
}
```

Якщо спосіб інтерполяції ще не визначений, `interpolation` може бути `null`. Це краще, ніж неявно припускати конкретний алгоритм.

### `piecewise`

Комбінація різних правил:

```json
{
  "type": "piecewise",
  "variable": "level",
  "segments": [
    {
      "from": 1,
      "to": 1,
      "rule": { "type": "constant", "value": 100 }
    },
    {
      "from": 2,
      "to": 2,
      "rule": { "type": "constant", "value": 200 }
    },
    {
      "from": 3,
      "to": 10,
      "rule": {
        "type": "polynomial",
        "variable": "level",
        "coefficients": [0, 80, 5]
      }
    },
    {
      "from": 11,
      "to": 20,
      "rule": {
        "type": "linear",
        "variable": "level",
        "origin": 11,
        "value_at_origin": 900,
        "per_step": 120
      }
    },
    {
      "from": 21,
      "to": null,
      "rule": {
        "type": "exponential",
        "variable": "level",
        "origin": 21,
        "value_at_origin": 2200,
        "factor": 1.08
      }
    }
  ]
}
```

`from` і `to` включні. `to: null` означає необмежений верхній діапазон.

Це основний формат для випадків на кшталт «level 1 і 2 задані явно, 3–10 однією функцією, 11–20 іншою, 21+ третьою».


### `$ref`

Щоб не дублювати однакове правило, дозволяється посилання на інший параметр цього самого файла:

```json
{
  "$ref": "resources.source_upgrade.time_minutes"
}
```

`$ref` завжди означає повну заміну поточного value rule значенням за вказаним шляхом; поверх reference не накладаються додаткові поля.


## 3. Вкладені параметри

Вартість може складатися з кількох ресурсів, причому кожен компонент має власний value rule:

```json
{
  "building_upgrade_cost": {
    "warehouse": {
      "wood": { "...": "value rule" },
      "stone": { "...": "value rule" },
      "iron": 0,
      "coins": { "...": "value rule" }
    }
  }
}
```

Так само описуються Soldier Type, Building Type, Source Type тощо.

## 4. Параметри, які ще не зафіксовані

`null` означає: параметр передбачений структурою, але числове значення ще не погоджене.

Наприклад:

```json
{
  "time": {
    "dt_minutes": null
  }
}
```

Це не означає нуль і не повинно трактуватися runtime як робоче значення. Валідатор конфігурації має відмовитися запускати режим, який реально потребує `null`-параметра.

У V1 окремі `combat_start_delay` / `regrouping_time` / `retreat_local_time` не задаються: ці фіксовані інтервали використовують одну `dt_minutes`. У майбутньому окремий speed coefficient може змінювати лише звичайний Transit.

## 5. Що перенесено зі старого simulation config

У нові файли перенесені значення, що є параметрами самої гри: стартові запаси, productivity Source, storage growth, upgrade costs/time, distance efficiency, Bank curve, Coin income profiles, Governor's House multiplier, territorial upkeep, стартова кількість Knight та neutral source level.

Не перенесені як gameplay config параметри, що стосувалися лише старої симуляції/поведінки AI: `simulation_days`, `step_minutes`, `active_hours_local`, `decision_window_each_hour`, reserve fractions, `civilian_coin_floor`, `max_actions_per_decision_cycle`, `resource_value`.

Стара одноразова `annexation` cost також не переноситься, оскільки в актуальних правилах Annexation не має окремої одноразової ціни.

## 6. Поточний статус чисел

Числа в цих файлах — початкові балансні значення з попереднього моделювання там, де вони існували. Вони не вважаються остаточним балансом.

Для новіших механік, для яких у старому моделюванні значення не задавалися, структура створена, а значення залишені `null`.
