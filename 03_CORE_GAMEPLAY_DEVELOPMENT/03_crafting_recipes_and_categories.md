# 03. Рецепты крафта, категории и алгоритмы сопоставления

## 1. Революция формата рецептов в Minecraft 1.21.2+ и 1.21.4

> [!CAUTION]
> **КРИТИЧЕСКИЙ СБОЙ ПАРСИНГА СТАРЫХ РЕЦЕПТОВ:**
> В 1.21.2+ Mojang полностью переписали кодек `Ingredient`:
> - Старый формат объектов `{"item": "minecraft:stick"}` или `{"tag": "minecraft:planks"}` **БОЛЬШЕ НЕ ПОДДЕРЖИВАЕТСЯ**!
> - Попытка использовать старый формат вызывает фатальную ошибку при загрузке датапака:
>   `DataResult.Error['Input does not contain a key [fabric:type]: MapLike[...]']`
> 
> **Новые правила кодека:**
> 1. Ингредиент-предмет: строка ID напрямую: `"minecraft:stick"`.
> 2. Ингредиент-тег: строка с решеткой: `"#minecraft:planks"`.
> 3. Список допустимых предметов: массив строк: `["minecraft:iron_ingot", "minecraft:copper_ingot"]`.
> 4. Результат (`result`): объект с ключами `"id"` и `"count"`.

---

## 2. Примеры рабочих рецептов 1.21.4

### 2.1. Верстак: Форменный крафт (`data/<ns>/recipe/copper_spear.json`):
```json
{
  "type": "minecraft:crafting_shaped",
  "category": "equipment",
  "pattern": [
    "  X",
    " # ",
    "#  "
  ],
  "key": {
    "#": "minecraft:stick",
    "X": "minecraft:copper_ingot"
  },
  "result": {
    "id": "examplemod:copper_spear",
    "count": 1
  }
}
```

### 2.2. Верстак: Бесформенный крафт (`data/<ns>/recipe/copper_blend.json`):
```json
{
  "type": "minecraft:crafting_shapeless",
  "category": "misc",
  "ingredients": [
    "minecraft:copper_ingot",
    "#minecraft:coals"
  ],
  "result": {
    "id": "examplemod:copper_blend",
    "count": 2
  }
}
```

### 2.3. Стол кузнеца (`data/<ns>/recipe/netherite_spear_smithing.json`):
```json
{
  "type": "minecraft:smithing_transform",
  "template": "minecraft:netherite_upgrade_smithing_template",
  "base": "examplemod:diamond_spear",
  "addition": "minecraft:netherite_ingot",
  "result": {
    "id": "examplemod:netherite_spear",
    "count": 1
  }
}
```

### 2.4. Переплавка и категории:
> [!IMPORTANT]
> Для `minecraft:smoking` (коптильня) категория **обязана быть `"category": "food"`**! Использование `"misc"` приводит к конфликту с рецептурной книгой `RecipeBookCategories.SMOKER_FOOD`. Для печи обычно используется `"category": "blocks"` или `"misc"`.

```json
{
  "type": "minecraft:smelting",
  "category": "misc",
  "ingredient": "examplemod:copper_ore",
  "result": {
    "id": "minecraft:copper_ingot",
    "count": 1
  },
  "experience": 0.7,
  "cookingtime": 200
}
```

---

## 3. Условия безопасной загрузки (`fabric:load_conditions`)

Если рецепт использует предметы стороннего мода, защитите его Fabric Load Conditions, чтобы избежать краша `Unknown registry key`:

```json
{
  "fabric:load_conditions": [
    {
      "condition": "fabric:all_mods_loaded",
      "values": ["optional_mod_id"]
    }
  ],
  "type": "minecraft:crafting_shaped",
  "pattern": [" # ", "#X#", " # "],
  "key": {
    "#": "minecraft:iron_ingot",
    "X": "optional_mod_id:custom_core"
  },
  "result": {
    "id": "examplemod:advanced_gadget",
    "count": 1
  }
}
```

---

## 4. Алгоритмы сопоставления в коде (Engine Matching)

### 4.1. Скользящее окно со смещением (Sliding Window для $2 \times 2$ или $1 \times 2$ в $3 \times 3$):
Позволяет крафтам малого размера работать в любом месте сетки $3 \times 3$:
```java
public boolean matchesShaped(CraftingRecipeInput input, Pattern pattern) {
    int inputW = input.getWidth();
    int inputH = input.getHeight();
    int patternW = pattern.width;
    int patternH = pattern.height;

    if (inputW < patternW || inputH < patternH) return false;

    // Перебираем возможные смещения по X и Y
    for (int dx = 0; dx <= inputW - patternW; dx++) {
        for (int dy = 0; dy <= inputH - patternH; dy++) {
            if (checkMatch(input, pattern, dx, dy, false)) return true;
            // Проверка горизонтального зеркального отражения
            if (checkMatch(input, pattern, dx, dy, true)) return true;
        }
    }
    return false;
}
```

### 4.2. Рекурсивный бэктрекинг для бесформенных крафтов:
Предотвращает ложные совпадения при пересекающихся тегах без аллокации временных коллекций:
```java
private boolean matchShapelessBacktrack(List<ItemStack> inputs, List<Ingredient> ingredients, boolean[] used, int ingIndex) {
    if (ingIndex >= ingredients.size()) return true;

    Ingredient ingredient = ingredients.get(ingIndex);
    for (int i = 0; i < inputs.size(); i++) {
        if (!used[i] && ingredient.test(inputs.get(i))) {
            used[i] = true;
            if (matchShapelessBacktrack(inputs, ingredients, used, ingIndex + 1)) {
                return true;
            }
            used[i] = false; // Откат состояния
        }
    }
    return false;
}
```
