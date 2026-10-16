# 01. Схемы генерации мира (Worldgen Schemas в 1.21.4)

## 1. Структура каталогов датапака генерации мира

В Minecraft 1.21+ генерация мира строго стандартизирована в каталоге `data/<namespace>/worldgen/`:

```
data/<namespace>/worldgen/
├── biome/                      # Определение биомов
├── configured_carver/          # Каньоны и пещеры (вместо устаревшего carver/)
├── configured_feature/         # Базовые фичи (жилы руд, деревья, кристаллы)
├── placed_feature/             # Размещение фич (высоты, частота, фильтры)
├── density_function/           # Функции плотности рельефа
└── noise_settings/             # Параметры шума измерений
```

---

## 2. Спецификация `configured_feature`

> [!CAUTION]
> **Провайдер состояния блока:**
> В 1.21.4 тип простого стейт-провайдера называется **`"type": "minecraft:simple_state_provider"`** (а НЕ `"minecraft:simple"` или `"minecraft:simple_block_state_provider"`).

Пример рудной жилы (`data/examplemod/worldgen/configured_feature/ruby_ore.json`):
```json
{
  "type": "minecraft:ore",
  "config": {
    "discard_chance_on_air_exposure": 0.0,
    "size": 8,
    "targets": [
      {
        "state": {
          "type": "minecraft:simple_state_provider",
          "state": {
            "Name": "examplemod:ruby_ore"
          }
        },
        "target": {
          "predicate_type": "minecraft:tag_match",
          "tag": "minecraft:stone_ore_replaceables"
        }
      },
      {
        "state": {
          "type": "minecraft:simple_state_provider",
          "state": {
            "Name": "examplemod:deepslate_ruby_ore"
          }
        },
        "target": {
          "predicate_type": "minecraft:tag_match",
          "tag": "minecraft:deepslate_ore_replaceables"
        }
      }
    ]
  }
}
```

---

## 3. Спецификация `placed_feature`

> [!CAUTION]
> **Модификатор смещения:**
> Вместо устаревшего `"minecraft:offset"` в 1.21.4 используется **`"type": "minecraft:random_offset"`** с обязательными объектами `xz_spread` и `y_spread`.

Пример размещения руды (`data/examplemod/worldgen/placed_feature/ruby_ore.json`):
```json
{
  "feature": "examplemod:ruby_ore",
  "placement": [
    {
      "type": "minecraft:count",
      "count": 7
    },
    {
      "type": "minecraft:in_square"
    },
    {
      "type": "minecraft:height_range",
      "height": {
        "type": "minecraft:trapezoid",
        "max_inclusive": { "absolute": 16 },
        "min_inclusive": { "absolute": -64 }
      }
    },
    {
      "type": "minecraft:biome"
    }
  ]
}
```

---

## 4. Спецификация `density_function`

В 1.21.4 в математических операторах функций плотности (`minecraft:add`, `minecraft:mul`, `minecraft:min`, `minecraft:max`):
- Аргументы именуются **`"argument1"`** и **`"argument2"`** (вместо устаревших `"left"` и `"right"`).
- Градиенты по высоте используют тип **`"minecraft:y_clamped_gradient"`** с полями `from_y`, `to_y`, `from_value`, `to_value`.
