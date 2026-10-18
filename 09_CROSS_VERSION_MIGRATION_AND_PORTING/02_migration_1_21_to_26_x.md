# 02. Руководство по миграции: с 1.21.x на 26.x

## Главная таблица изменений

| Компонент / Подсистема | В Minecraft 1.21.x | В Minecraft 26.x | Решение при миграции |
| :--- | :--- | :--- | :--- |
| **Java Runtime** | Java 21 LTS | **Java 25 LTS** | JDK 25 компилятор и рантайм |
| **Библиотека ввода** | GLFW (0-based кнопки мыши) | **SDL3 (1-based кнопки мыши)** | Левый клик = 1, Правый клик = 3, Средний = 2 |
| **Смена экрана** | `minecraft.setScreen(...)` | **`minecraft.setScreenAndShow(...)`** | Использовать новый метод отображения экранов |
| **Инициализация компонентов на титульном экране** | Прямой вызов `build()` | **Требует `safeProvider`** | Обернуть `BuiltInRegistries.createWrapperLookup()` во избежание краша на титульном экране |
| **Менеджер рецептов** | Единый `RecipeManager` | **`ServerRecipeManager`** | Разделение на серверный менеджер и `NetworkRecipeId` |
| **Пакеты Netty** | Классические POJO | **Java Records** | Доступ к полям пакетов через методы records: `packet.particle()` вместо `packet.getParticle()` |
| **Синхронизация сущностей** | `DataWatcher` / `DataTracker` | **`SynchedEntityData` + `DataValue<?>`** | Использование массива `itemsById` и records данных |
| **Mojang Authlib** | Authlib 6.x – 9.x (`YggdrasilServicesKeyInfo`) | **Authlib 10.x (`MinecraftServicesKeyInfo`)** | Рефакторинг пакета в `com.mojang.authlib.services.*` |
| **Открытие внешних ссылок** | `Util.getPlatform().openUri(uri)` | **`Blaze3D.openUri(uri)` (26.3+)** | Использовать `Blaze3D.openUri` для перехода в браузер |
| **Стек трансформаций GUI** | `MatrixStack` (1.21 – 1.21.5) | **`Matrix3x2fStack` (1.21.6 – 1.21.8)** | `matrices.pushMatrix()` / `matrices.translate(x, y)` |
| **Маппинги исходников** | Yarn (до 1.21.9 включительно) | **Mojang Official (начиная с 1.21.10)** | `Button`, `Component`, `GuiGraphics`, `Font` вместо Yarn |
| **Рендеринг GUI** | Прямой `DrawContext.drawText(...)` | **`GuiGraphicsExtractor` (начиная с 26.1)** | Экстракция состояния через `extractRenderState`, `centeredText`, `text` |

---

## Защита инициализации компонентов на титульном экране (`safeProvider`)

В Minecraft 26.x при открытии модового GUI с предметами из главного меню (через ModMenu):
- Динамические датапак-реестры (`RegistryAccess`) еще не существуют.
- Ванильные инициализаторы предметов пытаются разрешить динамические теги (например, огнестойкость брони, типы урона), выбрасывая `IllegalStateException: Missing tag`.

**Решение:**
Использовать безопасную обертку `safeProvider`, возвращающую пустые ссылки `HolderSet.emptyNamed(...)` при отсутствии реестров мира.

---

## Эволюция Mojang Authlib (Minecraft 26.3+ / Authlib 10.0.77+)

В версиях до 26.3 класс валидации подписи скинов находился по пути:
`com.mojang.authlib.yggdrasil.YggdrasilServicesKeyInfo`

Начиная с Authlib 10.x:
- Класс полностью удалён и заменён на `com.mojang.authlib.services.MinecraftServicesKeyInfo`, реализующий интерфейс `ServicesKeyInfo`.
- Любые миксины перехвата `validateProperty(Property)` обязаны учитывать смену целевого класса при переходе на 26.3+.
- Для сборки 26.3+ через Fabric Loom 1.18+ сам процесс сборки Gradle требует рантайм **Java 25** (`JAVA_HOME` должен указывать на JDK 25, иначе Loom завершит таск с ошибкой несовместимости версий JVM).

