# 06. Датапаки, генерация данных (Datagen) и pack.mcmeta

## 1. Автоматическая генерация данных (Data Generation)

Вместо ручного создания сотен однотипных JSON-файлов для моделей, рецептов и таблиц лута используется встроенная система генерации данных Fabric Data Generator или NeoForge GatherData.

### Настройка в Fabric (`FabricDataOutput`):
```java
public class ExampleModDataGenerator implements DataGeneratorEntrypoint {
    @Override
    public void onInitializeDataGenerator(FabricDataGenerator fabricDataGenerator) {
        FabricDataGenerator.Pack pack = fabricDataGenerator.createPack();

        pack.addProvider(ModModelGenerator::new);
        pack.addProvider(ModRecipeGenerator::new);
        pack.addProvider(ModLootTableGenerator::new);
        pack.addProvider(ModBlockTagGenerator::new);
    }
}
```

Генерация запускается командой Gradle:
```bash
./gradlew runData
```

---

## 2. Строгие требования к `pack.mcmeta` в 1.21.4

> [!CAUTION]
> Отсутствие файла `pack.mcmeta` или ключа `pack_format` приводит к отказу загрузки ресурсов:
> `DataResult.Error['No key pack_format in MapLike[...]']`

### Таблица актуальных форматов (`pack_format`):
| Версия Minecraft | Формат Ресурспака (`assets/`) | Формат Датапака (`data/`) |
| :--- | :--- | :--- |
| **1.20.1** | `15` | `15` |
| **1.20.4** | `22` | `26` |
| **1.21 - 1.21.1** | `34` | `48` |
| **1.21.4** | **`46`** | **`61`** |
| **26.x** | `48+` | `65+` |

### Пример эталонного `pack.mcmeta`:
```json
{
  "pack": {
    "pack_format": 61,
    "description": "Example Mod Data & Assets"
  }
}
```

---

## 3. Строгая типизация полей в JSON (GSON / Mojang Codecs)

> [!WARNING]
> В версиях 1.21+ кодеки Mojang строго проверяют примитивные типы данных:
> - Поля вроде `can_always_eat` или `has_consume_particles` в компонентах еды обязаны быть JSON-булевыми (`true` или `false`).
> - Запись строками `"true"` или `"false"` вызывает мгновенное исключение:
>   `com.google.gson.JsonSyntaxException: Not a boolean: "false"`
