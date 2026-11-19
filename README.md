> **Language:** Русский · [English](README.en.md)

# AiCreateForMinecraft

Практическая база знаний по созданию, портированию и оптимизации модов для **Minecraft: Java Edition** — Fabric и NeoForge.

Это не учебник по одной версии и не набор абстрактных примеров: здесь лежат решения, проверенные на реальных проектах, вместе с границами применимости. Общие принципы отделены от различий между версиями, поэтому база остаётся полезной при переходе с 1.21.x на 26.x и дальше.

---

## 🎯 Охват

| Поколение | Java | Loom | Что в базе |
| :--- | :---: | :--- | :--- |
| **1.21 – 1.21.11** | 21 | 1.10 – 1.18 | Полный набор: сборка, контент, рендер, миксины, миграции |
| **26.1 – 26.3** | 25 | 1.17+ | Полный набор + отличия 26.x: SDL3-ввод, `safeProvider`, Authlib 10.x |
| **1.20.x и ниже** | 17 / 21 | — | Только заметки о миграции на 1.21+ |

Точные версии маппингов, Gradle и Loom по конкретным веткам сведены в [09_CROSS_VERSION_MIGRATION_AND_PORTING/04](09_CROSS_VERSION_MIGRATION_AND_PORTING/04_multi_version_gui_input_and_tests.md).

Два загрузчика, один исходный код: Fabric и NeoForge собираются из одного дерева, без `common`/`fabric`/`neoforge` подпроектов. Подробности — [02/03](02_LOADERS_AND_MULTILOADER/03_dual_target_single_source.md).

---

## 🚀 С чего начать

| Задача | Документы |
| :--- | :--- |
| Настроить проект, Java, версии, манифесты | [01/01](01_FOUNDATIONS/01_toolchain_and_java_versions.md), [01/02](01_FOUNDATIONS/02_project_templates_and_buildscripts.md), [01/03](01_FOUNDATIONS/03_versioning_and_metadata_standards.md) |
| Собрать один мод под Fabric и NeoForge | [02/03](02_LOADERS_AND_MULTILOADER/03_dual_target_single_source.md) |
| Понять API загрузчиков и точки входа | [02/01](02_LOADERS_AND_MULTILOADER/01_fabric_architecture.md), [02/02](02_LOADERS_AND_MULTILOADER/02_neoforge_architecture.md) |
| Предметы, блоки, рецепты, GUI | [03/01](03_CORE_GAMEPLAY_DEVELOPMENT/01_items_and_data_components.md), [03/02](03_CORE_GAMEPLAY_DEVELOPMENT/02_blocks_and_block_entities.md), [03/04](03_CORE_GAMEPLAY_DEVELOPMENT/04_gui_menus_and_screens.md) |
| Мобы, AI, анимации, звук | [04/01](04_ENTITIES_MOBS_AND_ANIMATIONS/01_entity_architecture_and_ai.md), [04/02](04_ENTITIES_MOBS_AND_ANIMATIONS/02_geckolib_mobs_and_animations.md) |
| Генерация мира, структуры, измерения | [05/01](05_WORLDGEN_AND_STRUCTURES/01_worldgen_json_schemas.md), [05/02](05_WORLDGEN_AND_STRUCTURES/02_custom_structures_and_jigsaw.md) |
| Сеть и миксины | [06/01](06_NETWORKING_AND_BYTECODE_MIXINS/01_networking_custompayload.md), [06/02](06_NETWORKING_AND_BYTECODE_MIXINS/02_bytecode_mixins_mastery.md) |
| Рендер, шейдеры, HUD | [07/01](07_RENDERING_SHADERS_AND_OPENGL/01_modern_render_state_pipeline.md), [07/03](07_RENDERING_SHADERS_AND_OPENGL/03_opengl_lifecycle_and_state_safety.md) |
| Производительность и аллокации | [08/01](08_PERFORMANCE_AND_OPTIMIZATION/01_tick_loop_and_gc_optimization.md), [08/02](08_PERFORMANCE_AND_OPTIMIZATION/02_memory_caching_and_multithreading.md) |
| Портировать мод на новую версию | [09/05](09_CROSS_VERSION_MIGRATION_AND_PORTING/05_feature_porting_workflow.md), [09/02](09_CROSS_VERSION_MIGRATION_AND_PORTING/02_migration_1_21_to_26_x.md) |
| Клиентские запросы к миру и высотам | [09/06](09_CROSS_VERSION_MIGRATION_AND_PORTING/06_client_world_queries_and_heights.md) |
| Автотесты клиента | [09/08](09_CROSS_VERSION_MIGRATION_AND_PORTING/08_client_gametest_suite.md) |
| Разобрать краш | [10/01](10_TROUBLESHOOTING_AND_CRASH_CATALOG/01_runtime_exceptions_dictionary.md), чек-лист [10/02](10_TROUBLESHOOTING_AND_CRASH_CATALOG/02_universal_porting_checklist.md) |
| Опубликовать на CurseForge | [11/01](11_DISTRIBUTION_AND_PUBLISHING/01_curseforge_author_console.md) |

---

## 📁 Структура

| Раздел | О чём |
| :--- | :--- |
| **[01_FOUNDATIONS](01_FOUNDATIONS)** | Java 21/25, байткод, шаблоны `build.gradle` и `gradle.properties`, SemVer без префикса `v`, манифесты Fabric и NeoForge |
| **[02_LOADERS_AND_MULTILOADER](02_LOADERS_AND_MULTILOADER)** | Точки входа, реестры, Dual-Target Single-Source (два JAR из одного кода), Architectury, локализация без Fabric API |
| **[03_CORE_GAMEPLAY_DEVELOPMENT](03_CORE_GAMEPLAY_DEVELOPMENT)** | Предметы и Data Components, блоки и BlockEntity, актуальные схемы рецептов, GUI, креативные вкладки, datagen |
| **[04_ENTITIES_MOBS_AND_ANIMATIONS](04_ENTITIES_MOBS_AND_ANIMATIONS)** | Сущности и AI-цели, физика, серверные fake players, GeckoLib, анимации, звуки и партиклы |
| **[05_WORLDGEN_AND_STRUCTURES](05_WORLDGEN_AND_STRUCTURES)** | JSON-схемы генерации, структуры и jigsaw, GZIP-NBT, схематики, кастомные измерения и карверы |
| **[06_NETWORKING_AND_BYTECODE_MIXINS](06_NETWORKING_AND_BYTECODE_MIXINS)** | `CustomPayload`, миксины, аксессоры, `@WrapOperation`, безопасное размещение серверных миксинов |
| **[07_RENDERING_SHADERS_AND_OPENGL](07_RENDERING_SHADERS_AND_OPENGL)** | RenderState-конвейер, шейдеры, матрицы WorldRenderer, HUD без клиентских классов, слоты текстур, порталы и параллакс |
| **[08_PERFORMANCE_AND_OPTIMIZATION](08_PERFORMANCE_AND_OPTIMIZATION)** | Тики без лишних аллокаций, кэши и многопоточность, изоляция от ванильного кода, массовая 2D-отрисовка одним оверлеем |
| **[09_CROSS_VERSION_MIGRATION_AND_PORTING](09_CROSS_VERSION_MIGRATION_AND_PORTING)** | Чек-листы миграции 1.20 → 1.21.x → 26.x, бэкпортирование, мультиверсионный GUI/ввод, перенос фич, клиентские запросы к миру, client-gametest |
| **[10_TROUBLESHOOTING_AND_CRASH_CATALOG](10_TROUBLESHOOTING_AND_CRASH_CATALOG)** | Словарь runtime-исключений с причинами и исправлениями, универсальный чек-лист перед релизом |
| **[11_DISTRIBUTION_AND_PUBLISHING](11_DISTRIBUTION_AND_PUBLISHING)** | CurseForge Author Console, чанковая загрузка, метаданные версий, автоматизация через Selenium + Chrome |

---

## 🧭 Принципы

1. **Только проверенное.** Каждое решение взято из работающего мода, а не из предположений.
2. **Версия называется явно.** Если API отличается между 1.21.x и 26.x — это написано, а не спрятано за «в новых версиях».
3. **Примеры нейтральные.** Имена модов, пакетов и артефактов — условные (`mymod`, `com.example.*`, `<ModName>-<LoaderName>-<AuthorName>-v<version>.jar`).
4. **Сначала принцип, потом ветка.** Раздел объясняет механику, а конкретные сигнатуры и отличия — в разделе 09 и в таблице версий.

---

## 🤖 Правила для ИИ-ассистентов

1. **Сверяй версию, а не память.**
   - Не переноси код между ветками без проверки сигнатур: `Identifier` vs `ResourceLocation`, Yarn vs Mojang-маппинги, `Item.Settings.registryKey()`.
   - Если в ответе используется API, который менялся (модели предметов, Data Components, ввод, рендер), укажи версию, для которой решение верно.
2. **Мультилоадер — через Dual-Target Single-Source.**
   - Один проект, два артефакта, FML-стабы; не дроби репозиторий на три подмодуля.
3. **Соблюдай границы окружения.**
   - Никаких `net.minecraft.client.*` на выделенном сервере.
   - Серверные миксины — в общий массив `"mixins"`, не в `"server"` (в одиночной игре он игнорируется).
4. **Не оптимизируй вслепую.**
   - Никакого Stream API в горячих тиках; сначала измерение, потом кэш.
   - При кастомном рендере сохраняй текстурные слоты, иначе ломаются шрифты в Sodium/Iris.
5. **Портируй по процессу.**
   - Для перехода на новую версию используй [09/05](09_CROSS_VERSION_MIGRATION_AND_PORTING/05_feature_porting_workflow.md), а не точечные правки наугад.

---

## 🛠 Обновление базы

- Новая версия Minecraft → обновить таблицу охвата и добавить/дополнить документ в `09`.
- Устаревшее решение не удаляется молча: помечается версия, в которой оно перестало работать, и замена.
- Перед коммитом: примеры собираются, ссылки относительные, имена артефактов по шаблону.

---

## 📄 Лицензия

[Apache-2.0](LICENSE).
