# 01. Тулчейны, версии Java и байткода

## 1. Таблица соответствия версий Minecraft, Java и байткода

| Версия Minecraft | Рекомендуемый JDK | Версия байткода | Gradle | Инструмент сборщика | Маппинги по умолчанию |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **1.20.1** | Java 17 | `61.0` | 8.1 - 8.5 | Fabric Loom / ForgeGradle | Mojang / Yarn / Parchment |
| **1.20.4 - 1.20.6** | Java 17 / 21 | `61.0` / `65.0` | 8.6 - 8.8 | Fabric Loom / NeoGradle | Mojang / Yarn |
| **1.21.0 - 1.21.3** | Java 21 LTS | `65.0` | 8.10+ | Fabric Loom 1.8+ / ModDevGradle | Mojang / Yarn |
| **1.21.4** | Java 21 LTS | `65.0` | 8.10 - 8.12 | Fabric Loom 1.10+ | Mojang / Yarn |
| **26.1 - 26.3** | Java 25 LTS | `69.0` | 8.12+ | Fabric Loom 1.18-SNAPSHOT | Mojang Official |

> [!WARNING]
> Если при запуске мода в логе появляется ошибка:
> `java.lang.UnsupportedClassVersionError: ... has been compiled by a more recent version of the Java Runtime (class file version 66.0)`
> это означает, что один из модов или вложенных JAR (Jar-in-Jar) был собран под более новую версию Java (например, Java 22 = 66.0, Java 25 = 69.0), чем запущенная игра (Java 21 = 65.0).
> Для 1.21.4 целевой байткод компилятора **строго `targetCompatibility = 21` / `release = 21`**.

---

## 2. Настройка Java Toolchain в Gradle

Чтобы сборка не зависела от системного JDK разработчика, в `build.gradle` всегда настраивается Java Toolchain:

```groovy
def targetJavaVersion = 21

tasks.withType(JavaCompile).configureEach {
    it.options.encoding = "UTF-8"
    if (targetJavaVersion >= 10 || JavaVersion.current().isJava10Compatible()) {
        it.options.release = targetJavaVersion
    }
}

java {
    def javaVersion = JavaVersion.toVersion(targetJavaVersion)
    if (JavaVersion.current() < javaVersion) {
        toolchain.languageVersion = JavaLanguageVersion.of(targetJavaVersion)
    }
}
```

Для Minecraft 26.x:
```groovy
def targetJavaVersion = 25
// Сборка на JDK 25 с release = 25
```

---

## 3. Маппинги: Yarn против Official Mojang Mappings

### Official Mojang Mappings (Рекомендуется для универсальности):
- В среде выполнения NeoForge игра работает **исключительно** на официальных маппингах Mojang.
- Использование Yarn в NeoForge приводит к немедленному `NoClassDefFoundError`.
- Подключение в Fabric Loom:
  ```groovy
  dependencies {
      minecraft "com.mojang:minecraft:${project.minecraft_version}"
      mappings loom.officialMojangMappings()
  }
  ```
- **Преимущество:** Единый исходный код может компилироваться под Fabric и напрямую исполняться под NeoForge без промежуточного ремаппинга исходников.

### Fabric Yarn Mappings:
- Традиционные маппинги сообщества Fabric.
- Подключение:
  ```groovy
  dependencies {
      minecraft "com.mojang:minecraft:${project.minecraft_version}"
      mappings "net.fabricmc:yarn:${project.yarn_mappings}:v2"
  }
  ```
- Для запуска Yarn-кода на NeoForge требуется двухпроходный ремаппинг байткода (`TinyRemapper`: `Yarn -> Intermediary -> Mojang`).
