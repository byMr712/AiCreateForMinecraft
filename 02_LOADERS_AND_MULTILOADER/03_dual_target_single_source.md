# 03. Архитектура Dual-Target Single-Source (Loom + NeoForge Stubs)

## 1. Концепция: Зачем это нужно?

Традиционный подход к мультилоадерам (Architectury, MultiLoader-Template) дробит репозиторий на 3+ подпроекта (`common`, `fabric`, `neoforge`), требует дублирования классов реализации платформы (`@ExpectPlatform`), увеличивает время компиляции до 2–5 минут и создает конфликты демонов Gradle.

**Dual-Target Single-Source** решает эту проблему элегантно:

1. **Единый исходный код** на официальных маппингах Mojang.
2. **Одна Gradle-сборка** (Fabric Loom) с двумя целевыми артефактами:
   - `jar` — **Fabric JAR**;
   - `jarNeoForge` — **NeoForge JAR** из тех же классов плюс NeoForge-точка входа.
3. **Время сборки:** менее **10 секунд** на оба загрузчика сразу (на типовом клиентском моде — около 6 секунд с `clean`).
4. **Zero-Dependencies:** не требует обязательного Fabric API или громоздких сторонних библиотек.

> [!IMPORTANT]
> Начиная с Minecraft 26.x и Loom 1.17 задачи `remapJar` в проекте **нет**. Fabric-артефакт собирает обычная задача `jar`, а отдельного шага ремапа в пайплайне не выполняется: в JAR остаются имена Mojang (`net/minecraft/client/...`), refmap-файлы не генерируются. Старые инструкции про «Intermediary» относятся к версиям до 26.x и к Loom ≤ 1.16.

---

## 2. Структура проекта

```
MyMod-MinecraftMod/
├── src/main/java/com/example/mymod/          # Общий код (может быть пустым)
├── src/main/resources/
│   ├── fabric.mod.json                      # Манифест Fabric
│   ├── mymod.mixins.json                    # Общий конфиг миксинов
│   └── assets/mymod/                        # Текстуры, иконка
├── src/client/java/com/example/mymod/       # Клиентский код (splitEnvironmentSourceSets)
│   ├── MyModClient.java                     # Точка входа Fabric ClientModInitializer
│   └── ...
├── src/client/resources/
│   ├── mymod.client.mixins.json             # Клиентские миксины
│   └── assets/mymod/lang/                   # Локализация (ванильные lang-файлы)
├── src/neoforge/java/
│   ├── com/example/mymod/
│   │   └── MyModNeoForge.java               # Точка входа NeoForge с @Mod("mymod")
│   └── net/neoforged/fml/common/
│       └── Mod.java                         # Легковесный compile-time стаб аннотации @Mod
├── src/neoforge/resources/
│   └── META-INF/neoforge.mods.toml          # Манифест NeoForge
├── gradle.properties
└── build.gradle                             # Двойная сборка (Fabric + NeoForge)
```

> [!NOTE]
> `src/client` появляется только при `loom { splitEnvironmentSourceSets() }`. Это рекомендуемый режим для клиентских модов: клиентские классы физически не попадают в серверную часть Fabric-мода. NeoForge-сборка при этом включает и `main`, и `client`.

---

## 3. Легковесный стаб NeoForge (`src/neoforge/java/net/neoforged/fml/common/Mod.java`)

Этот класс нужен **только** для того, чтобы компилятор Java успешно собрал аннотацию `@Mod` во время билда. При упаковке в NeoForge JAR стаб исключается (`exclude "net/**"`), поэтому в рантайме игра использует настоящий FML из NeoForge:

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

Пример для Minecraft 26.x / Loom 1.17:

```groovy
plugins {
    id 'net.fabricmc.fabric-loom' version '1.17-SNAPSHOT'
    id 'maven-publish'
}

version = project.mod_version
group = project.maven_group

base {
    archivesName = project.archives_base_name
}

loom {
    splitEnvironmentSourceSets()

    mods {
        "mymod" {
            sourceSet sourceSets.main
            sourceSet sourceSets.client
        }
    }
}

dependencies {
    minecraft "com.mojang:minecraft:${project.minecraft_version}"
    implementation "net.fabricmc:fabric-loader:${project.loader_version}"
}

sourceSets {
    neoforge {
        java {
            srcDirs = ['src/neoforge/java']
        }
        resources {
            srcDirs = ['src/neoforge/resources']
        }
        compileClasspath += sourceSets.main.output + sourceSets.client.output + configurations.compileClasspath
        runtimeClasspath += sourceSets.main.output + sourceSets.client.output + configurations.runtimeClasspath
    }
}

def artifactVersion = "v${project.version}"

// 1. Fabric JAR. Для Loom 1.17+ это и есть готовый артефакт: remapJar отсутствует.
jar {
    archiveFileName = "${project.archives_base_name}-${artifactVersion}.jar"
    from("LICENSE") {
        rename { "${it}_${project.archives_base_name}" }
    }
}

// Совместимость с Loom 1.18, где remapJar может присутствовать.
// tasks.matching безопасно настраивает задачу, только если она существует.
tasks.matching { it.name == 'remapJar' }.configureEach {
    archiveFileName = "${project.archives_base_name}-${artifactVersion}.jar"
}

// 2. NeoForge JAR из тех же классов.
task jarNeoForge(type: Jar) {
    dependsOn compileNeoforgeJava, processNeoforgeResources, clientClasses, classes
    archiveFileName = "${project.archives_base_name.replace('-Fabric-', '-NeoForge-')}-${artifactVersion}.jar"
    destinationDirectory = layout.buildDirectory.dir("libs")
    duplicatesStrategy = DuplicatesStrategy.EXCLUDE

    from(sourceSets.main.output) {
        exclude "fabric.mod.json"
    }
    from(sourceSets.client.output) {
        // Fabric-only entrypoint; NeoForge использует MyModNeoForge.
        exclude "com/example/mymod/MyModClient.class"
    }
    from(sourceSets.neoforge.output) {
        // Compile-time стабы не должны попасть в рантайм.
        exclude "net/**"
    }
    from("src/client/resources/assets/mymod/icon.png") {
        into ""
        rename { "icon.png" }
    }
    from("LICENSE")
}

// ВАЖНО: build, а не assemble.
build.dependsOn(jarNeoForge)
```

Для версий до 26.x (Loom 1.10–1.16, Yarn или официальные маппинги) добавьте явные маппинги и собирайте Fabric-артефакт через `remapJar`:

```groovy
dependencies {
    minecraft "com.mojang:minecraft:${project.minecraft_version}"
    mappings loom.officialMojangMappings()
    modImplementation "net.fabricmc:fabric-loader:${project.loader_version}"
}

remapJar {
    archiveFileName = "${project.archives_base_name}-${artifactVersion}.jar"
    exclude "META-INF/neoforge.mods.toml"
}
```

---

## 5. Имена артефактов

Загрузчик и версия Minecraft держатся в `archives_base_name`, а `jarNeoForge` меняет только токен загрузчика:

```properties
# gradle.properties
archives_base_name=ExampleMod-Fabric-26.2
mod_version=1.1.1
```

```
ExampleMod-Fabric-26.2-v1.1.1.jar
ExampleMod-NeoForge-26.2-v1.1.1.jar
```

Внутри `fabric.mod.json` и `neoforge.mods.toml` версия остаётся без префикса `v` (`1.1.1`).

---

## 6. Что важно не перепутать

| Ошибка | Последствие | Правильно |
|---|---|---|
| `assemble.dependsOn jarNeoForge` | `./gradlew assemble` соберёт только Fabric | `build.dependsOn jarNeoForge` |
| В `jarNeoForge` нет `client.output` | клиентские классы отсутствуют в NeoForge JAR | `from(sourceSets.client.output)` |
| В NeoForge JAR попал `fabric.mod.json` | два манифеста, конфликт загрузчиков | `exclude "fabric.mod.json"` |
| В NeoForge JAR попал Fabric-entrypoint | FML пытается загрузить Fabric-класс | `exclude ".../MyModClient.class"` |
| В NeoForge JAR попал `net/fabricmc/**` или стаб `net/neoforged/**` | краш JPMS при установленном Sinytra Connector | `exclude "net/**"` |
| Ожидание задачи `remapJar` на Loom 1.17 | `Task 'remapJar' not found` | собирать `jar` |

---

## 7. Предотвращение сбоев модульной системы JPMS (Split Packages)

> [!CAUTION]
> Никогда не включайте в `jarNeoForge` стабы из пространства имен `net.fabricmc.*`!
> Если другой установленный в NeoForge мод содержит адаптер Fabric (например, `Sinytra Connector`), наличие дублирующихся пакетов `net.fabricmc` приведет к немедленному фатальному крашу JVM:
> `java.lang.module.ResolutionException: Modules ... and ... export package net.fabricmc to the unnamed module`
>
> **Решение:** исключайте весь пакет `net/**` из NeoForge-артефакта и используйте безопасную рефлексию для проверок окружения:

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

Если в проекте нет рантайм-ветвлений по загрузчику (разные entrypoint-классы), `LoaderHelper` не нужен.

---

## 8. Проверка собранных артефактов

```bash
# 1. Оба JAR собраны и лежат рядом
ls build/libs

# 2. Fabric JAR
jar tf build/libs/ExampleMod-Fabric-26.2-v1.1.1.jar | grep -E "fabric.mod.json|neoforge.mods.toml|^net/|^icon.png"

# 3. NeoForge JAR
jar tf build/libs/ExampleMod-NeoForge-26.2-v1.1.1.jar | grep -E "fabric.mod.json|neoforge.mods.toml|^net/|^icon.png|MyModClient.class|MyModNeoForge.class"
```

Ожидаемый результат:

- **Fabric:** есть `fabric.mod.json` и Fabric-entrypoint; нет `neoforge.mods.toml`, нет `net/`, нет `icon.png` в корне.
- **NeoForge:** есть `META-INF/neoforge.mods.toml`, `icon.png` в корне и NeoForge-entrypoint; нет `fabric.mod.json`, нет `net/`, нет Fabric-entrypoint.

Дополнительно:

```bash
./gradlew clean jar jarNeoForge   # оба артефакта с нуля
./gradlew tasks --all | grep -i remap   # на Loom 1.17 вывод пустой
```
