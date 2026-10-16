# 03. Архитектура Dual-Target Single-Source (Loom + NeoForge Stubs)

## 1. Концепция: Зачем это нужно?

Традиционный подход к мультилоадерам (Architectury, MultiLoader-Template) дробит репозиторий на 3+ подпроекта (`common`, `fabric`, `neoforge`), требует дублирования классов реализации платформы (`@ExpectPlatform`), увеличивает время компиляции до 2–5 минут и создает конфликты демонов Gradle.

**Архитектура byMr712 (Dual-Target Single-Source)** решает эту проблему элегантно:
1. **Единый исходный код** на официальных маппингах Mojang (`loom.officialMojangMappings()`).
2. **Сборка через быстрый Fabric Loom** с двумя целевыми тасками:
   - `remapJar`: автоматически ремапит код в Intermediary для чистого **Fabric JAR**.
   - `jarNeoForge`: пакует исходный Mojang-байткод для прямого запуска под **NeoForge JAR**.
3. **Время сборки:** Менее **10 секунд** для обоих загрузчиков сразу!
4. **Zero-Dependencies:** Не требует обязательного Fabric API или громоздких сторонних библиотек.

---

## 2. Структура проекта

```
MyMod-MinecraftMod/
├── src/main/java/com/mr712/mymod/
│   ├── MyModClient.java                   # Точка входа Fabric ClientModInitializer
│   ├── config/MyModConfig.java            # Loader-Agnostic конфигурация
│   ├── mixin/                             # Универсальные Sponge-миксины
│   │   └── MyHandlerMixin.java
│   └── util/LoaderHelper.java             # Рефлексивные запросы к загрузчикам
├── src/main/resources/
│   ├── fabric.mod.json                    # Манифест Fabric
│   ├── mymod.mixins.json                  # Общий конфиг миксинов
│   └── assets/mymod/                      # Текстуры, локализация, иконка
├── src/neoforge/java/
│   ├── com/mr712/mymod/
│   │   └── MyModNeoForge.java             # Точка входа NeoForge с @Mod("mymod")
│   └── net/neoforged/fml/common/
│       └── Mod.java                       # Легковесный compile-time стаб аннотации @Mod
├── src/neoforge/resources/
│   └── META-INF/neoforge.mods.toml        # Манифест NeoForge
├── gradle.properties
└── build.gradle                           # Двойная сборка (Fabric + NeoForge)
```

---

## 3. Легковесный стаб NeoForge (`src/neoforge/java/net/neoforged/fml/common/Mod.java`)

Этот класс нужен **только** для того, чтобы компилятор Java успешно собрал аннотацию `@Mod` во время билда. При упаковке в NeoForge JAR стаб исключается (`exclude "net/neoforged/**"`), поэтому в рантайме игра использует настоящий FML из NeoForge:

```java
package net.neoforged.fml.common;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Mod {
    String value();
}
```

---

## 4. Конфигурация `build.gradle` для двойной сборки

```groovy
plugins {
    id 'fabric-loom' version '1.10.1'
    id 'maven-publish'
}

sourceSets {
    neoforge {
        compileClasspath += main.compileClasspath
        runtimeClasspath += main.runtimeClasspath
    }
}

dependencies {
    minecraft "com.mojang:minecraft:${project.minecraft_version}"
    mappings loom.officialMojangMappings() // Официальные маппинги Mojang!
    modImplementation "net.fabricmc:fabric-loader:${project.loader_version}"
}

// 1. Сборка чистого Fabric JAR
remapJar {
    archiveFileName = "${project.mod_id}-Fabric-${project.minecraft_version}-byMr712-v${project.mod_version}.jar"
    exclude "META-INF/neoforge.mods.toml"
}

// 2. Сборка чистого NeoForge JAR
tasks.register('jarNeoForge', Jar) {
    dependsOn classes, neoforgeClasses
    archiveFileName = "${project.mod_id}-NeoForge-${project.minecraft_version}-byMr712-v${project.mod_version}.jar"
    destinationDirectory = file("${project.buildDir}/libs")

    from sourceSets.main.output
    from sourceSets.neoforge.output

    // Исключаем стабы и специфичные файлы Fabric
    exclude "fabric.mod.json"
    exclude "com/mr712/mymod/MyModClient.class"
    exclude "net/neoforged/**"

    // Копируем иконку в корень JAR для NeoForge
    from("src/main/resources/assets/${project.mod_id}/icon.png") {
        into ""
    }
}

// Запуск обеих сборок одной командой ./gradlew build
assemble.dependsOn jarNeoForge
```

---

## 5. Предотвращение сбоев модульной системы JPMS (Split Packages)

> [!CAUTION]
> Никогда не включайте в `jarNeoForge` стабы из пространства имен `net.fabricmc.*`!
> Если другой установленный в NeoForge мод содержит адаптер Fabric (например, `Sinytra Connector`), наличие дублирующихся пакетов `net.fabricmc` приведет к немедленному фатальному крашу JVM:
> `java.lang.module.ResolutionException: Modules ... and ... export package net.fabricmc to the unnamed module`
> 
> **Решение:** Для всех проверок окружения в коде используйте безопасную рефлексию:
```java
public class LoaderHelper {
    public static boolean isFabric() {
        try {
            Class.forName("net.fabricmc.loader.api.FabricLoader");
            return true;
        } catch (ClassNotFoundException e) {
            return false;
        }
    }

    public static boolean isNeoForge() {
        try {
            Class.forName("net.neoforged.fml.loading.FMLLoader");
            return true;
        } catch (ClassNotFoundException e) {
            return false;
        }
    }
}
```
