# 01. Предметы, Data Components и Client Item Definitions

## 1. Регистрация предметов и критическое требование `.registryKey()`

> [!CAUTION]
> **КРИТИЧЕСКИЙ БАГ MINECRAFT 1.21.4+:**
> Конструктор `Item(Item.Settings)` проверяет:
> `Objects.requireNonNull(settings.getRegistryKey(), "Item id not set")`
> Если не вызвать `.registryKey(key)` в `Item.Settings`, игра падает с немедленным фатальным:
> `NullPointerException: Item id not set` прямо во время инициализации мода!

### Корректная регистрация в Fabric (1.21.4):
```java
public class ModItems {
    public static final RegistryKey<Item> COPPER_SPEAR_KEY = 
        RegistryKey.of(RegistryKeys.ITEM, Identifier.of("examplemod", "copper_spear"));

    public static final Item COPPER_SPEAR = new Item(
        new Item.Settings()
            .registryKey(COPPER_SPEAR_KEY)
            .maxCount(1)
            .maxDamage(250)
    );

    public static void register() {
        Registry.register(Registries.ITEM, COPPER_SPEAR_KEY, COPPER_SPEAR);
    }
}
```

---

## 2. Client Item Definitions (`assets/<namespace>/items/<id>.json`) — СТАНДАРТ 1.21.4+

> [!IMPORTANT]
> Начиная с версии 1.21.4, игра **больше не ищет** модели предметов напрямую в `models/item/<id>.json`.
> Каждый предмет обязан иметь файл-дефиницию в каталоге `assets/<namespace>/items/<id>.json`.

### 2.1. Стандартный плоский предмет:
`assets/examplemod/items/ruby.json`:
```json
{
  "model": {
    "type": "minecraft:model",
    "model": "examplemod:item/ruby"
  }
}
```

### 2.2. Разделение 2D спрайта в GUI и 3D модели в руке (`minecraft:display_context`):
Больше не нужны сложные кастомные миксины в рендерер предметов! Ванильный селектор решает это нативно:
`assets/examplemod/items/copper_spear.json`:
```json
{
  "model": {
    "type": "minecraft:select",
    "property": "minecraft:display_context",
    "cases": [
      {
        "when": ["gui", "ground", "fixed"],
        "model": {
          "type": "minecraft:model",
          "model": "examplemod:item/copper_spear_gui"
        }
      }
    ],
    "fallback": {
      "type": "minecraft:model",
      "model": "examplemod:item/copper_spear_in_hand"
    }
  }
}
```

> [!WARNING]
> Допустимые контексты `display_context`: `"gui"`, `"ground"`, `"fixed"`, `"head"`, `"firstperson_righthand"`, `"firstperson_lefthand"`, `"thirdperson_righthand"`, `"thirdperson_lefthand"`.
> Контекст `"on_shelf"` (снапшоты 25w/26.x) в 1.21.4 **не существует** и ломает парсинг!

### 2.3. Инструменты со стадиями поломки (`minecraft:range_dispatch`):
```json
{
  "model": {
    "type": "minecraft:range_dispatch",
    "property": "minecraft:damage",
    "scale": 1.0,
    "entries": [
      { "threshold": 0.75, "model": { "type": "minecraft:model", "model": "examplemod:item/damaged_3" } },
      { "threshold": 0.50, "model": { "type": "minecraft:model", "model": "examplemod:item/damaged_2" } },
      { "threshold": 0.25, "model": { "type": "minecraft:model", "model": "examplemod:item/damaged_1" } }
    ],
    "fallback": {
      "type": "minecraft:model",
      "model": "examplemod:item/copper_axe"
    }
  }
}
```

---

## 3. Data Components (Замена NBT на предметах в 1.21+)

В 1.20.5+ методы `stack.getNbt()` и `stack.getOrCreateNbt()` удалены. Данные на `ItemStack` хранятся строго через `DataComponentType`:

```java
// 1. Чтение кастомного NBT компаунда
CustomDataComponent customData = stack.get(DataComponentTypes.CUSTOM_DATA);
NbtCompound tag = customData != null ? customData.copyNbt() : new NbtCompound();
String myValue = tag.getString("MyKey");

// 2. Запись или обновление данных
stack.apply(DataComponentTypes.CUSTOM_DATA, CustomDataComponent.DEFAULT, comp ->
    comp.apply(nbt -> nbt.putString("MyKey", "NewValue"))
);

// 3. Установка атрибутов (урон, скорость, дальность)
stack.set(DataComponentTypes.ATTRIBUTE_MODIFIERS, 
    AttributeModifiersComponent.builder()
        .add(EntityAttributes.ATTACK_DAMAGE, new EntityAttributeModifier(...), EquipmentSlotGroup.MAINHAND)
        .build()
);
```

---

## 4. Спецификация ToolMaterial и ArmorMaterial в 1.21.4

### ToolMaterial:
```java
public class ModToolMaterials {
    public static final TagKey<Item> REPAIR_TAG = 
        TagKey.of(RegistryKeys.ITEM, Identifier.of("examplemod", "copper_ingots"));

    public static final ToolMaterial COPPER_TOOL_MATERIAL = new ToolMaterial(
        BlockTags.INCORRECT_FOR_IRON_TOOL, // Тег блоков, которые нельзя копать
        250,                                // Прочность
        6.0f,                               // Скорость копания
        2.0f,                               // Базовый дополнительный урон
        14,                                 // Зачаровываемость
        REPAIR_TAG                          // Тег предметов для починки на наковальне
    );
}
```

### ArmorMaterial и слои экипировки (`assets/<ns>/equipment/<id>.json`):
```json
{
  "layers": {
    "humanoid": [
      { "texture": "examplemod:copper" }
    ],
    "humanoid_leggings": [
      { "texture": "examplemod:copper" }
    ]
  }
}
```
> [!CAUTION]
> Слой `"humanoid_baby"` не существует в 1.21.4! Допустимы: `"humanoid"`, `"humanoid_leggings"`, `"wings"`, `"wolf_body"`, `"horse_body"`, `"llama_body"`.
