> **Language:** Русский · [English](README.en.md)

# AiCreateForMinecraft

Моя база знаний по созданию, портированию и оптимизации модов для **Minecraft (Java Edition)**.

В этом репозитории я собрал свои наработки, проверенные решения и шаблоны кода под **Fabric** и **NeoForge**: от настройки сборки и регистрации контента до миксинов, кастомных шейдеров, структур и разбора частых ошибок. Информация структурирована так, чтобы в ней легко ориентировался и человек, и ИИ.

---

## 📁 Структура базы знаний

| Раздел | Описание |
| :--- | :--- |
| **[01_FOUNDATIONS](file:///E:/GitHub/AiCreateForMinecraft/01_FOUNDATIONS)** | Версии Java (21/25), байткод, шаблоны `build.gradle`, `gradle.properties`, формат версий SemVer `X.X` (без префикса `v`) и манифесты. |
| **[02_LOADERS_AND_MULTILOADER](file:///E:/GitHub/AiCreateForMinecraft/02_LOADERS_AND_MULTILOADER)** | Fabric, NeoForge, моя сборка Dual-Target (два JAR из одного кода через Loom без лишних подпроектов) и Architectury. |
| **[03_CORE_GAMEPLAY_DEVELOPMENT](file:///E:/GitHub/AiCreateForMinecraft/03_CORE_GAMEPLAY_DEVELOPMENT)** | Предметы с `.registryKey()`, Data Components, Client Item Definitions, блоки, слои прозрачности, синтаксис рецептов 1.21+ и GUI. |
| **[04_ENTITIES_MOBS_AND_ANIMATIONS](file:///E:/GitHub/AiCreateForMinecraft/04_ENTITIES_MOBS_AND_ANIMATIONS)** | Создание мобов, настройка AI целей, физика воды, интеграция GeckoLib, анимации и требования к звукам. |
| **[05_WORLDGEN_AND_STRUCTURES](file:///E:/GitHub/AiCreateForMinecraft/05_WORLDGEN_AND_STRUCTURES)** | Схемы генерации мира, кастомные структуры (папка `structure/`, обязательное GZIP сжатие NBT `1F 8B`) и конвертация схем. |
| **[06_NETWORKING_AND_BYTECODE_MIXINS](file:///E:/GitHub/AiCreateForMinecraft/06_NETWORKING_AND_BYTECODE_MIXINS)** | Сеть на `CustomPayload`, миксины, аксессоры, `@WrapOperation` и правила безопасного размещения серверных миксинов. |
| **[07_RENDERING_SHADERS_AND_OPENGL](file:///E:/GitHub/AiCreateForMinecraft/07_RENDERING_SHADERS_AND_OPENGL)** | `EntityRenderState`, расположение шейдеров в 1.21.4, порядок матриц в `WorldRenderer`, сохранение 16 текстурных слотов и порталы. |
| **[08_PERFORMANCE_AND_OPTIMIZATION](file:///E:/GitHub/AiCreateForMinecraft/08_PERFORMANCE_AND_OPTIMIZATION)** | Оптимизация тиков без лишних аллокаций Stream API, `BlockPos.Mutable`, безопасная работа с чанками и атомарные конфиги. |
| **[09_CROSS_VERSION_MIGRATION_AND_PORTING](file:///E:/GitHub/AiCreateForMinecraft/09_CROSS_VERSION_MIGRATION_AND_PORTING)** | Чек-листы обновления модов с 1.20 на 1.21.4 и с 1.21 на 26.x (SDL3, `safeProvider`), бэкпортирование нативных библиотек. |
| **[10_TROUBLESHOOTING_AND_CRASH_CATALOG](file:///E:/GitHub/AiCreateForMinecraft/10_TROUBLESHOOTING_AND_CRASH_CATALOG)** | Разбор частых крашей с готовыми решениями и 35-шаговый чек-лист проверки мода перед релизом. |

---

## 🤖 Правила для ИИ-ассистентов

Если ты ИИ и выполняешь задачу в моих проектах:

1. **Используй актуальные решения из этой базы:**
   - Не используй устаревший код (в 1.21.4 модели предметов лежат в `items/<id>.json`, а не в `models/item/`; NBT на предметах заменен на Data Components; конструктор предметов требует `.registryKey()`).
   - Если нужен мультилоадер, используй подход Dual-Target Single-Source со стабами FML, а не дроби проект на три подмодуля.
2. **Следи за окружением:**
   - Не вызывай клиентские классы на сервере.
   - Серверные миксины клади в общий массив `"mixins": [...]`, а не в `"server": [...]` (он не работает в одиночной игре).
3. **Пиши аккуратный код:**
   - Не используй Stream API в горячих тиках (`tick()`).
   - Сохраняй текстурные слоты OpenGL при кастомном рендеринге, чтобы не ломать шрифты в Sodium/Iris.
