# 03. Стандарты версионирования и манифестов (Metadata Standards)

## 1. Правило чистого формата версии: `X.X` / `X.Y.Z` (Без префикса `v`)

> [!CAUTION]
> **Критическое требование NeoForge и Maven ArtifactVersion:**
> Внутри метаданных мода (`fabric.mod.json`, `neoforge.mods.toml`, `gradle.properties`) версия **ОБЯЗАНА** начинаться с цифры (например, `1.0`, `1.2.3`).
> Использование префикса `v` (например, `version="v1.0"`) в `neoforge.mods.toml` приводит к фатальному сбою загрузки:
> `net.neoforged.fml.loading.moddiscovery.InvalidModFileException: ... Invalid version string 'v1.0'`
> 
> **Стандарт:**
> - Внутренняя версия мода (`mod_version`): строго `1.0` или `1.0.0`.
> - Префикс `v` допустим **только в именах файлов** JAR: `archives_base_name=MyMod-Fabric-1.21.4-byMr712-v1.0`.

Защитная нормализация в `build.gradle`:
```groovy
def cleanVersion = project.mod_version.toString().replaceFirst(/^v/, '')
```

---

## 2. Манифест Fabric (`src/main/resources/fabric.mod.json`)

```json
{
  "schemaVersion": 1,
  "id": "examplemod",
  "version": "${version}",
  "name": "Example Mod",
  "description": "Excellence in Minecraft Modding.",
  "authors": [
    "Mr712"
  ],
  "contact": {
    "sources": "https://github.com/byMr712/ExampleMod-MinecraftMod",
    "issues": "https://github.com/byMr712/ExampleMod-MinecraftMod/issues"
  },
  "license": "Apache-2.0",
  "icon": "assets/examplemod/icon.png",
  "environment": "*",
  "entrypoints": {
    "main": [
      "com.mr712.examplemod.ExampleMod"
    ],
    "client": [
      "com.mr712.examplemod.client.ExampleModClient"
    ]
  },
  "mixins": [
    "examplemod.mixins.json"
  ],
  "depends": {
    "fabricloader": ">=0.16.10",
    "fabric-api": ">=0.115.0",
    "minecraft": "~1.21.4"
  },
  "suggests": {
    "modmenu": "*"
  }
}
```

> [!NOTE]
> Если мод чисто клиентский (HUD, кастомные подсказки, локальные улучшения рендера), укажите `"environment": "client"`. Это защитит выделенные серверы от случайной загрузки клиентских классов.

---

## 3. Манифест NeoForge (`src/main/resources/META-INF/neoforge.mods.toml`)

```toml
modLoader="javafml"
loaderVersion="[1,)"
license="Apache-2.0"
issueTrackerURL="https://github.com/byMr712/ExampleMod-MinecraftMod/issues"

[[mods]]
modId="examplemod"
version="${version}"
displayName="[MR] Example Mod"
description='''
Excellence in Minecraft Modding.
'''
logoFile="icon.png"
displayURL="https://github.com/byMr712/ExampleMod-MinecraftMod"

[[dependencies.examplemod]]
modId="neoforge"
type="required"
versionRange="[21.4,)"
ordering="NONE"
side="BOTH"

[[dependencies.examplemod]]
modId="minecraft"
type="required"
versionRange="[1.21.4,1.22)"
ordering="NONE"
side="BOTH"

[[mixins]]
config="examplemod.mixins.json"
```

### Ключевые моменты NeoForge:
- **`logoFile="icon.png"`**: Иконка мода ищется в корне архива JAR.
- **`displayURL`**: Обеспечивает кликабельность кнопки **«Homepage» / «Website»** в меню списка модов NeoForge и моде Catalogue.
- **`[[mixins]]`**: Секция подключения миксинов через FML.
