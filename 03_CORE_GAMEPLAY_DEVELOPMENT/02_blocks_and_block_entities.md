# 02. Блоки, BlockEntity и графические слои (Render Layers)

## 1. Регистрация блоков и BlockItem

В 1.21.4 ключ реестра `registryKey` обязателен и для блоков, и для связанных с ними предметов:

```java
public class ModBlocks {
    public static final RegistryKey<Block> RUBY_BLOCK_KEY = 
        RegistryKey.of(RegistryKeys.BLOCK, Identifier.of("examplemod", "ruby_block"));
    public static final RegistryKey<Item> RUBY_BLOCK_ITEM_KEY = 
        RegistryKey.of(RegistryKeys.ITEM, Identifier.of("examplemod", "ruby_block"));

    public static final Block RUBY_BLOCK = new Block(
        AbstractBlock.Settings.create()
            .registryKey(RUBY_BLOCK_KEY)
            .mapColor(MapColor.COLOR_RED)
            .strength(5.0f, 6.0f)
            .sounds(BlockSoundGroup.METAL)
            .requiresTool()
    );

    public static void register() {
        Registry.register(Registries.BLOCK, RUBY_BLOCK_KEY, RUBY_BLOCK);
        Registry.register(
            Registries.ITEM, 
            RUBY_BLOCK_ITEM_KEY, 
            new BlockItem(RUBY_BLOCK, new Item.Settings().registryKey(RUBY_BLOCK_ITEM_KEY).useBlockPrefixedTranslationKey())
        );
    }
}
```

---

## 2. BlockEntity и сериализация данных

```java
public class GemInfuserBlockEntity extends BlockEntity {
    private int progress = 0;

    public GemInfuserBlockEntity(BlockPos pos, BlockState state) {
        super(ModBlockEntities.GEM_INFUSER, pos, state);
    }

    // Сохранение NBT (в 1.21.4 перегружается writeNbt / saveAdditional)
    @Override
    protected void writeNbt(NbtCompound nbt, RegistryWrapper.WrapperLookup registries) {
        super.writeNbt(nbt, registries);
        nbt.putInt("Progress", this.progress);
    }

    // Загрузка NBT
    @Override
    protected void readNbt(NbtCompound nbt, RegistryWrapper.WrapperLookup registries) {
        super.readNbt(nbt, registries);
        this.progress = nbt.getInt("Progress");
    }

    // Серверный тик
    public static void tick(World world, BlockPos pos, BlockState state, GemInfuserBlockEntity be) {
        if (!world.isClient()) {
            be.progress++;
            be.markDirty();
        }
    }
}
```

---

## 3. Регистрация прозрачных слоев (`BlockRenderLayerMap`)

> [!CAUTION]
> **Черные квадраты вокруг листвы, травы и саженцев:**
> По умолчанию блоки в Fabric рендерятся на слое `solid` (непрозрачный). Если у блока текстура содержит альфа-канал или прозрачность, в клиентском инициализаторе `ClientModInitializer` **ОБЯЗАТЕЛЬНО** зарегистрировать слой:

```java
public class ModRenderLayers {
    public static void register() {
        // Cutout: растения, саженцы, двери, люки, цепочки, паутина
        BlockRenderLayerMap.INSTANCE.putBlocks(
            RenderType.cutout(),
            ModBlocks.RUBY_SAPLING,
            ModBlocks.CRYSTAL_DOOR,
            ModBlocks.CRYSTAL_TRAPDOOR
        );

        // CutoutMipped: листва деревьев
        BlockRenderLayerMap.INSTANCE.putBlocks(
            RenderType.cutoutMipped(),
            ModBlocks.CRYSTAL_LEAVES
        );

        // Translucent: полупрозрачное цветное стекло, лед, жидкости
        BlockRenderLayerMap.INSTANCE.putBlocks(
            RenderType.translucent(),
            ModBlocks.TINTED_CRYSTAL_GLASS
        );
    }
}
```

---

## 4. Регистрация взаимодействия с ванильными системами

Кастомные блоки должны подключаться к ванильным игровым циклам:

```java
public class ModVanillaCompat {
    public static void register() {
        // 1. Топливо для печей (FuelRegistryEvents.BUILD в Fabric API 1.21.4+)
        FuelRegistryEvents.BUILD.register((builder, context) -> {
            builder.add(ModBlocks.CRYSTAL_WOOD, 300);     // 300 тиков (1.5 предмета)
            builder.add(ModBlocks.CRYSTAL_PLANKS, 300);
            builder.add(ModBlocks.CRYSTAL_SAPLING, 100);
        });

        // 2. Компостер (шанс добавления слоя удобрения)
        CompostingChanceRegistry.INSTANCE.add(ModBlocks.CRYSTAL_LEAVES, 0.30f);
        CompostingChanceRegistry.INSTANCE.add(ModBlocks.CRYSTAL_SAPLING, 0.65f);

        // 3. Воспламеняемость и горение (FlammableBlockRegistry)
        FlammableBlockRegistry flammable = FlammableBlockRegistry.getDefaultInstance();
        flammable.add(ModBlocks.CRYSTAL_LOG, 5, 5);      // burnChance, spreadChance
        flammable.add(ModBlocks.CRYSTAL_PLANKS, 5, 20);
        flammable.add(ModBlocks.CRYSTAL_LEAVES, 30, 60);
    }
}
```
