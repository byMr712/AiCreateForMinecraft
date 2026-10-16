# 01. Архитектура Fabric (Fabric Loader & Fabric API)

## 1. Точки входа (Entrypoints)

В Fabric весь жизненный цикл инициализации мода управляется через интерфейсы точек входа, объявляемые в `fabric.mod.json`:

```java
// Общий инициализатор (Client + Server)
package com.example.mod;

import net.fabricmc.api.ModInitializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class ExampleMod implements ModInitializer {
    public static final String MOD_ID = "examplemod";
    public static final Logger LOGGER = LoggerFactory.getLogger(MOD_ID);

    @Override
    public void onInitialize() {
        LOGGER.info("Initializing {}", MOD_ID);
        ModBlocks.register();
        ModItems.register();
        ModItemGroups.register();
        ModEvents.register();
    }
}
```

```java
// Клиентский инициализатор (Only Client)
package com.example.mod.client;

import net.fabricmc.api.ClientModInitializer;
import net.fabricmc.api.EnvType;
import net.fabricmc.api.Environment;

@Environment(EnvType.CLIENT)
public class ExampleModClient implements ClientModInitializer {
    @Override
    public void onInitializeClient() {
        // Регистрация слоев блоков (прозрачность)
        ModRenderLayers.register();
        // Регистрация клиентских рендереров сущностей и блоков
        ModEntityRenderers.register();
        // Регистрация экранов GUI
        ModScreens.register();
        // Сетевые клиентские приемники
        ModClientNetworking.register();
    }
}
```

---

## 2. Статическая регистрация в реестрах (Vanilla Registries)

В Fabric объекты регистрируются напрямую в статических реестрах `net.minecraft.registry.Registries`:

```java
public class ModItems {
    public static final RegistryKey<Item> RUBY_KEY = RegistryKey.of(
        RegistryKeys.ITEM, 
        Identifier.of(ExampleMod.MOD_ID, "ruby")
    );

    public static final Item RUBY = new Item(new Item.Settings().registryKey(RUBY_KEY));

    public static void register() {
        Registry.register(Registries.ITEM, RUBY_KEY, RUBY);
    }
}
```

---

## 3. Событийная модель Fabric API

Fabric API использует шаблоны интерфейсных коллбэков `Event<T>` с методом `.register(...)`:

```java
// Реакция на подключение игрока к серверу
ServerPlayConnectionEvents.JOIN.register((handler, sender, server) -> {
    ServerPlayerEntity player = handler.getPlayer();
    player.sendMessage(Text.literal("Welcome to the server!"), false);
});

// Добавление предметов во вкладки креатива
ItemGroupEvents.modifyEntriesEvent(ItemGroups.INGREDIENTS).register(entries -> {
    entries.add(ModItems.RUBY);
});

// События тиков
ServerTickEvents.END_SERVER_TICK.register(server -> {
    // Выполняется в конце каждого серверного тика
});
```

---

## 4. Защита клиентского кода (`@Environment(EnvType.CLIENT)`)

> [!CAUTION]
> Любой класс, метод или поле, обращающиеся к коду MinecraftClient, DrawContext, звукам или рендерерам, **обязаны** быть изолированы в клиентской части. Попытка загрузить такой класс на выделенном сервере немедленно выбрасывает `ClassNotFoundException: net.minecraft.client.MinecraftClient`.
