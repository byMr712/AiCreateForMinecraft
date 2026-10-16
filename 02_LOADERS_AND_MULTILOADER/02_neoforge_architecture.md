# 02. Архитектура NeoForge (1.21+)

## 1. Точка входа `@Mod` и шины событий (Event Buses)

В NeoForge загрузка мода начинается с конструктора класса, помеченного `@Mod`:

```java
package com.example.mod;

import net.neoforged.bus.api.IEventBus;
import net.neoforged.fml.common.Mod;
import net.neoforged.fml.event.lifecycle.FMLCommonSetupEvent;
import net.neoforged.neoforge.common.NeoForge;

@Mod(ExampleMod.MOD_ID)
public class ExampleMod {
    public static final String MOD_ID = "examplemod";

    public ExampleMod(IEventBus modEventBus) {
        // 1. Регистрация реестров через DeferredRegister на шине мода
        ModBlocks.BLOCKS.register(modEventBus);
        ModItems.ITEMS.register(modEventBus);
        ModTabs.CREATIVE_MODE_TABS.register(modEventBus);

        // 2. Листенеры жизненного цикла на шине мода
        modEventBus.addListener(this::commonSetup);

        // 3. Регистрация на глобальной игровой шине NeoForge
        NeoForge.EVENT_BUS.register(new GameEventHandler());
    }

    private void commonSetup(FMLCommonSetupEvent event) {
        event.enqueueWork(() -> {
            // Потокобезопасная настройка
        });
    }
}
```

### Разделение шин событий:
- **Mod Event Bus (`modEventBus`)**: События инициализации, регистрации, настройки датагена (`GatherDataEvent`), настройки клиентских рендереров (`EntityRenderersEvent`). Вызываются один раз при старте игры.
- **Game Event Bus (`NeoForge.EVENT_BUS`)**: Игровые события рантайма (`PlayerTickEvent`, `LivingDamageEvent`, `ServerStartingEvent`). Вызываются во время игрового процесса.

---

## 2. Система отложенной регистрации (`DeferredRegister`)

В NeoForge объекты регистрируются через `DeferredRegister`, который гарантирует регистрацию в нужный момент жизненного цикла движка:

```java
public class ModBlocks {
    public static final DeferredRegister.Blocks BLOCKS = 
        DeferredRegister.createBlocks(ExampleMod.MOD_ID);

    public static final DeferredBlock<Block> RUBY_BLOCK = BLOCKS.registerSimpleBlock(
        "ruby_block",
        BlockBehaviour.Properties.of()
            .mapColor(MapColor.COLOR_RED)
            .strength(5.0f, 6.0f)
            .sound(SoundType.METAL)
            .requiresCorrectToolForDrops()
    );
}

public class ModItems {
    public static final DeferredRegister.Items ITEMS = 
        DeferredRegister.createItems(ExampleMod.MOD_ID);

    public static final DeferredItem<BlockItem> RUBY_BLOCK_ITEM = 
        ITEMS.registerSimpleBlockItem(ModBlocks.RUBY_BLOCK);

    public static final DeferredItem<Item> RUBY = ITEMS.registerSimpleItem(
        "ruby", 
        new Item.Properties().stacksTo(64)
    );
}
```

---

## 3. Регистрация вкладок творческого режима

```java
public class ModTabs {
    public static final DeferredRegister<CreativeModeTab> CREATIVE_MODE_TABS =
        DeferredRegister.create(Registries.CREATIVE_MODE_TAB, ExampleMod.MOD_ID);

    public static final DeferredHolder<CreativeModeTab, CreativeModeTab> EXAMPLE_TAB =
        CREATIVE_MODE_TABS.register("example_tab", () -> CreativeModeTab.builder()
            .title(Component.translatable("itemGroup." + ExampleMod.MOD_ID))
            .icon(() -> new ItemStack(ModItems.RUBY.get()))
            .displayItems((parameters, output) -> {
                output.accept(ModItems.RUBY.get());
                output.accept(ModBlocks.RUBY_BLOCK.get());
            })
            .build()
        );
}
```

---

## 4. Система способностей (Capabilities API)

Вместо устаревших методов инвентарей NeoForge использует типизированные `Capabilities`:
- `Capabilities.ItemHandler.BLOCK`: Инвентари блоков (`IItemHandler`).
- `Capabilities.EnergyStorage.BLOCK`: Энергия блоков (`IEnergyStorage`).
- `Capabilities.FluidHandler.BLOCK`: Жидкости (`IFluidHandler`).

Регистрация через событие `RegisterCapabilitiesEvent`:
```java
@SubscribeEvent
public static void registerCapabilities(RegisterCapabilitiesEvent event) {
    event.registerBlockEntity(
        Capabilities.ItemHandler.BLOCK,
        ModBlockEntities.MY_CONTAINER.get(),
        (blockEntity, side) -> blockEntity.getItemHandler(side)
    );
}
```
