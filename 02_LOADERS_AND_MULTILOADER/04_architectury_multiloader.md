# 04. Мультилоадер Architectury (@ExpectPlatform)

## 1. Концепция Architectury

Framework **Architectury** предоставляет кроссплатформенный слой абстракций над Fabric API и NeoForge.

### Структура подпроектов:
- `common/`: Общий код. Не содержит прямых импортов Fabric или NeoForge.
- `fabric/`: Реализация специфичного для Fabric функционала + точка входа `ModInitializer`.
- `neoforge/`: Реализация специфичного для NeoForge функционала + точка входа `@Mod`.

---

## 2. Паттерн `@ExpectPlatform`

Если методу в `common` требуется разная реализация на разных платформах, метод помечается аннотацией `@ExpectPlatform`:

```java
// common/src/main/java/com/example/mod/platform/PlatformHelper.java
package com.example.mod.platform;

import dev.architectury.injectables.annotations.ExpectPlatform;
import java.nio.file.Path;

public class PlatformHelper {
    @ExpectPlatform
    public static Path getConfigDirectory() {
        throw new AssertionError();
    }
}
```

Реализация под Fabric:
```java
// fabric/src/main/java/com/example/mod/platform/PlatformHelperImpl.java
package com.example.mod.platform;

import net.fabricmc.loader.api.FabricLoader;
import java.nio.file.Path;

public class PlatformHelperImpl {
    public static Path getConfigDirectory() {
        return FabricLoader.getInstance().getConfigDir();
    }
}
```

Реализация под NeoForge:
```java
// neoforge/src/main/java/com/example/mod/platform/PlatformHelperImpl.java
package com.example.mod.platform;

import net.neoforged.fml.loading.FMLPaths;
import java.nio.file.Path;

public class PlatformHelperImpl {
    public static Path getConfigDirectory() {
        return FMLPaths.CONFIGDIR.get();
    }
}
```

---

## 3. Декаплинг (Упрощение) Architectury проектов при портировании

> [!TIP]
> **Урок портирования (из практики Port-1.21.4-MinecraftMod):**
> Если исходный мод был написан на Architectury с подмодулями `common`, `fabric`, `neoforge`, но в вашей сборке требуется только версия под Fabric:
> 1. Объедините `common/src/main` и `fabric/src/main` в единый плоский каталог `src/main`.
> 2. Замените тяжелый мультимодульный `build.gradle` на одиночный `fabric-loom`.
> 3. Замените вызовы `@ExpectPlatform` на прямые реализации из Fabric.
> 
> Это сокращает время компиляции проекта с нескольких минут до **нескольких секунд** и полностью исключает рассинхронизацию Gradle Daemon.
