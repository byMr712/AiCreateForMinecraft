# 05. Вкладки инвентаря и теги (Creative Tabs & Tags)

## 1. Кастомные вкладки в творческом режиме (Fabric ItemGroup)

> [!CAUTION]
> **КРИТИЧЕСКИЙ КРАШ ПРИ СОЗДАНИИ ВКЛАДКИ:**
> В Fabric Item Group API v1 метод билдера `FabricItemGroup.builder()` требует **ОБЯЗАТЕЛЬНОГО** вызова `.displayName(...)`!
> Если этот метод пропущен, игра крашится при старте с ошибкой:
> `java.lang.IllegalStateException: No display name set for ItemGroup at knot//net.fabricmc.fabric.impl.itemgroup.FabricItemGroupBuilderImpl.build`

### Рабочий эталон создания вкладки (1.21.4):
```java
public class ModItemGroups {
    public static final RegistryKey<ItemGroup> MAIN_GROUP_KEY = 
        RegistryKey.of(RegistryKeys.ITEM_GROUP, Identifier.of("examplemod", "main"));

    public static void register() {
        Registry.register(
            Registries.ITEM_GROUP,
            MAIN_GROUP_KEY,
            FabricItemGroup.builder()
                .displayName(Text.translatable("itemGroup.examplemod.main")) // ОБЯЗАТЕЛЬНО!
                .icon(() -> new ItemStack(ModItems.RUBY))
                .entries((displayContext, entries) -> {
                    entries.add(ModItems.RUBY);
                    entries.add(ModBlocks.RUBY_BLOCK.asItem());
                })
                .build()
        );
    }
}
```

---

## 2. Обязательные ванильные теги предметов и блоков

Чтобы кастомные блоки и предметы полноценно интегрировались в мир Minecraft, они должны быть добавлены в соответствующие JSON-теги.

### 2.1. Инструменты и добыча:
- `data/minecraft/tags/block/mineable/pickaxe.json`:
  ```json
  {
    "replace": false,
    "values": [
      "examplemod:ruby_block",
      "examplemod:ruby_ore"
    ]
  }
  ```
- `data/minecraft/tags/block/needs_iron_tool.json`: требует железную кирку или лучше.

### 2.2. Зачарования в 1.21.4 (`data/minecraft/tags/item/enchantable/`):
- `weapon.json`: базовые оружейные чары.
- `sharp_weapon.json`: острота, небесная кара, гибель насекомых.
- `mining.json`: эффективность, шелковое касание, удача.
- `durability.json`: прочность и починка.
- `fire_aspect.json`: заговор огня.

### 2.3. Дерево и топливо:
- `data/minecraft/tags/item/logs_that_burn.json`
- `data/minecraft/tags/item/wooden_fences.json`
- `data/minecraft/tags/item/wooden_slabs.json`
- `data/minecraft/tags/item/wooden_stairs.json`
- `data/minecraft/tags/item/wooden_buttons.json`

### 2.4. Маяки (Beacon Base):
Для блоков минералов, из которых можно строить пирамиду маяка:
`data/minecraft/tags/block/beacon_base_blocks.json`:
```json
{
  "replace": false,
  "values": [
    "examplemod:ruby_block"
  ]
}
```

### 2.5. Конвенционные теги (Conventional Tags `#c:`):
Для совместимости с другими модами в экосистеме всегда дублируйте материалы в пространство имен `c`:
- `#c:gems/ruby`
- `#c:ingots/copper`
- `#c:tools/spears`
