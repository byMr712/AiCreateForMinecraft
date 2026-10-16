# 04. Кастомные измерения и интеграция с Distant Horizons

## 1. Архитектура файлов кастомного измерения

Для добавления нового измерения (Dimension) в Minecraft 1.21.4 требуются следующие JSON-файлы:

```
data/<namespace>/
├── dimension/
│   └── crystal_realm.json            # Связка типа измерения и генератора
├── dimension_type/
│   └── crystal_realm.json            # Физические свойства (свет, высота, эффекты)
└── worldgen/
    ├── biome/
    │   └── crystal_forest.json       # Описание биомов измерения
    └── noise_settings/
        └── crystal_noise.json        # Генератор рельефа и шума
```

---

## 2. Файл `dimension_type` (`data/<ns>/dimension_type/crystal_realm.json`)

```json
{
  "ultrawarm": false,
  "natural": true,
  "coordinate_scale": 1.0,
  "has_skylight": true,
  "has_ceiling": false,
  "ambient_light": 0.1,
  "fixed_time": false,
  "piglin_safe": false,
  "bed_works": true,
  "respawn_anchor_works": false,
  "has_raids": false,
  "logical_height": 384,
  "min_y": -64,
  "height": 384,
  "infiniburn": "#minecraft:infiniburn_overworld",
  "effects_location": "minecraft:overworld"
}
```

---

## 3. Файл `dimension` (`data/<ns>/dimension/crystal_realm.json`)

```json
{
  "type": "examplemod:crystal_realm",
  "generator": {
    "type": "minecraft:noise",
    "settings": "examplemod:crystal_noise",
    "biome_source": {
      "type": "minecraft:multi_noise",
      "biomes": [
        {
          "biome": "examplemod:crystal_forest",
          "parameters": {
            "temperature": 0.0,
            "humidity": 0.0,
            "continentalness": 0.0,
            "erosion": 0.0,
            "weirdness": 0.0,
            "depth": 0.0,
            "offset": 0.0
          }
        }
      ]
    }
  }
}
```

---

## 4. Мягкая интеграция терраформинга с Distant Horizons (DH LOD Updates)

> [!IMPORTANT]
> **Проблема «призрачного ландшафта» в Distant Horizons:**
> При процедурном создании миров или быстром терраформинге через прямое изменение блоков в секциях чанка (`ChunkSection`) в обход стандартных вызовов `world.setBlockState()`, мод **Distant Horizons** продолжает рендерить устаревшие дальние воксели (LOD), создавая артефакты фантомных гор или летающих островов.
> 
> **Решение (Безопасная рефлексия без жесткой зависимости):**
> Для немедленной постановки чанков в очередь запекания DH вызывается внутренняя система обновления:

```java
public class DistantHorizonsCompat {
    public static void markChunkForLodUpdate(ServerWorld level, WorldChunk chunk) {
        try {
            Class<?> levelWrapperClass = Class.forName("com.seibel.distanthorizons.common.wrappers.world.ServerLevelWrapper");
            Class<?> chunkWrapperClass = Class.forName("com.seibel.distanthorizons.common.wrappers.chunk.ChunkWrapper");
            Class<?> sharedApiClass = Class.forName("com.seibel.distanthorizons.common.SharedApi");

            Object levelWrapper = levelWrapperClass.getMethod("getWrapper", ServerWorld.class).invoke(null, level);
            Object chunkWrapper = chunkWrapperClass.getConstructor(WorldChunk.class, levelWrapperClass).newInstance(chunk, levelWrapper);

            Object sharedApi = sharedApiClass.getField("INSTANCE").get(null);
            sharedApiClass.getMethod("applyChunkUpdate", chunkWrapperClass).invoke(sharedApi, chunkWrapper);
        } catch (ClassNotFoundException ignored) {
            // Distant Horizons не установлен — пропускаем без ошибок
        } catch (Exception e) {
            // Защита от изменений во внутренних API DH
        }
    }
}
```
