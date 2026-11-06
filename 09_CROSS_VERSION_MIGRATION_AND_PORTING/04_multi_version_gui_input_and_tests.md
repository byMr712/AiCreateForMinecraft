# 04. Практический мультиверсионный порт: GUI, ввод, миксины и автотесты

> Заметки из портирования клиентского мода с полноэкранным GUI (карта, миникарта, метки, рисование, скриншоты) на Fabric + NeoForge для веток 1.21–1.21.11 и 26.1–26.3.

## 1. Общая стратегия

1. **Одна ветка = одна версия Minecraft.** Имя ветки совпадает с версией (`1.21`, `1.21.4`, `26.3`).
2. **Dual-Target Single-Source:** общий код на официальных маппингах Mojang, два артефакта из одного дерева:
   - Fabric — `remapJar`/`jar` (Intermediary);
   - NeoForge — `jarNeoForge` (Mojang-байткод + `neoforge.mods.toml`).
3. **Ноль обязательных зависимостей.** Fabric API/NeoForge-библиотеки не нужны в рантайме: все хуки — обычные Sponge-миксины.
4. **Версия внутри манифестов — `X.Y` без префикса `v`**, префикс `v` только в имени файла: `AtlasMap-<Loader>-<MC>-v1.0.jar`.

## 2. Java, маппинги и сборка по версиям

| Ветка | Java | Маппинги | Loom / Gradle | Особенности сборки |
| :--- | :---: | :--- | :--- | :--- |
| 26.1–26.3 | 25 | официальные имена «из коробки» | Loom 1.17.x / Gradle 9.6.1 | `mappings` не указывается; `implementation "net.fabricmc:fabric-loader"` достаточно |
| 1.21–1.21.11 | 21 | `loom.officialMojangMappings()` | Loom 1.18.x / Gradle 9.7.0 | нужен `modImplementation` для Fabric Loader, иначе на compile classpath нет Sponge Mixin |
| 1.21.x | 21 | Mojang Official | Loom 1.18.x | Java-toolchain 21; `mixin.compatibilityLevel` = `JAVA_21` |
| 26.x | 25 | Mojang Official | Loom 1.17.x | `mixin.compatibilityLevel` = `JAVA_25` |

> [!CAUTION]
> `compatibilityLevel` в `*.mixins.json` должен совпадать с Java рантайма. Иначе при запуске:
> `The requested compatibility level JAVA_25 could not be set ... (Java 21.0)`.
> Для 1.21.x ставьте `JAVA_21`, для 26.x — `JAVA_25`.

## 3. GUI, HUD и рендеринг по версиям

| Версия | GUI-класс | HUD-хук | Screen | Трансформации | Текстуры/blit | Resource |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 1.21–1.21.1 | `GuiGraphics` | `Gui.render(GuiGraphics, DeltaTracker)` | `mc.screen` / `mc.setScreen` | `PoseStack` | `blit(ResourceLocation, ...)` | `ResourceLocation` |
| 1.21.2–1.21.5 | `GuiGraphics` | `Gui.render(...)` | `mc.screen` / `mc.setScreen` | `PoseStack` | `blit(Function<ResourceLocation,RenderType>, ...)` + `RenderType::guiTextured` | `ResourceLocation` |
| 1.21.6–1.21.8 | `GuiGraphics` | `Gui.render(...)` | `mc.screen` / `mc.setScreen` | `Matrix3x2fStack` | `blit(RenderPipeline, ...)` + `RenderPipelines.GUI_TEXTURED` | `ResourceLocation` |
| 1.21.9–1.21.10 | `GuiGraphics` | `Gui.render(...)` | `mc.screen` / `mc.setScreen` | `Matrix3x2fStack` | `blit(RenderPipeline, ...)` + `RenderPipelines.GUI_TEXTURED` | `ResourceLocation` |
| 1.21.11 | `GuiGraphics` | `Gui.render(...)` | `mc.screen` / `mc.setScreen` | `Matrix3x2fStack` | `blit(RenderPipeline, ...)` + `RenderPipelines.GUI_TEXTURED` | `Identifier` (как в 26.x) |
| 26.1 | `GuiGraphicsExtractor` | `Gui.extractRenderState(GuiGraphicsExtractor, DeltaTracker)` | `mc.screen` / `mc.setScreen` | `Matrix3x2fStack` | `RenderPipelines` | `Identifier` |
| 26.2 | `GuiGraphicsExtractor` | `Hud.extractRenderState(GuiGraphicsExtractor, DeltaTracker)` | `mc.gui.screen()` / `mc.gui.setScreen(...)` | `Matrix3x2fStack` | `RenderPipelines` | `Identifier` |
| 26.3 | `GuiGraphicsExtractor` | `Hud.extractRenderState(...)` | `mc.gui.screen()` / `mc.gui.setScreen(...)` | `Matrix3x2fStack` | `RenderPipelines` | `Identifier` |

Практические следствия:

- В 26.2 `Hud` уже есть, но SDL (`org.lwjgl.sdl`) и `com.mojang.renderpearl` ещё отсутствуют. Это «гибрид» 26.1 и 26.3.
- В 1.21.6 вернулись `RenderPipeline` и `Matrix3x2fStack` — код рендера становится почти как в 26.x.
- В 1.21.2–1.21.5 удобно использовать `RenderType::guiTextured`; в 1.21.6+ — `RenderPipelines.GUI_TEXTURED`.
- В 1.21.6+ у `GuiGraphics.blit` есть перегрузка с цветом (последний `int`), поэтому перекраску иконок можно делать без `setColor`.

### Порядок аргументов `GuiGraphics.blit` (важно!)

Порядок аргументов меняется между версиями. Если перепутать `width/height` (куда рисуем) и `uWidth/vHeight` (какую область текстуры берём), иконки рисуются размером 16×16 и наезжают на текст, а скриншоты показывают не ту область (тайлинг/обрезка).

| Версия | Порядок аргументов |
| :--- | :--- |
| 1.21–1.21.1 | `blit(id, x, y, width, height, u, v, uWidth, vHeight, texW, texH)` — **сначала destination** |
| 1.21.2–1.21.11 | `blit(renderType/pipeline, id, x, y, u, v, width, height, uWidth, vHeight, texW, texH)` — **сначала source** |
| 26.x | `blit(pipeline, id, x, y, u, v, width, height, uWidth, vHeight, texW, texH)` — как в 1.21.2+ |

Пример: нарисовать иконку `size x size` из текстуры 16×16.

```java
// 1.21–1.21.1 (destination first) — или 9-арг: blit(tex, x, y, 0, 0, size, size, 16, 16)
g.blit(tex, x, y, size, size, 0, 0, 16, 16, 16, 16);

// 1.21.2+ и 26.x (source first)
g.blit(pipeline, tex, x, y, 0, 0, size, size, 16, 16, 16, 16);
```

Для 9-арг/10-арг перегрузки без `uWidth/vHeight` (тайлинг, `drawTiles`) порядок одинаковый: `(x, y, u, v, width, height, texW, texH)`.

### Оверлеи и слои GUI (`nextStratum` / `flush`)

В 1.21.x GUI-отрисовка батчится, и `renderOutline`/`submitOutline`, а также отложенный текст могут попасть в отдельный слой. Из-за этого при открытии модального оверлея (например, выбора скриншотов) поверх него остаются тонкие контуры и текст предыдущего меню.

| Версия | Метод сброса слоёв |
| :--- | :--- |
| 1.21–1.21.5 | `g.flush()` |
| 1.21.6–1.21.11 | `g.nextStratum()` |
| 26.x | `g.nextStratum()` |

Практические правила:

1. Обводки рисуйте 4 вызовами `g.fill(...)`, а не `g.renderOutline`/`g.submitOutline`. Так они попадают в обычный батч и перекрываются оверлеем.
2. В 1.21–1.21.5 `flush()` не сбрасывает текстовый батч: надёжнее не рисовать базовые панели и виджеты, когда открыт модальный оверлей.
3. В 1.21.6+ и 26.x `nextStratum()` сбрасывает все слои, поэтому достаточно вызвать его перед оверлеем.

### Текстуры и изображения

| Версия | `DynamicTexture` | Пиксели `NativeImage` | Высота мира |
| :--- | :--- | :--- | :--- |
| 1.21–1.21.1 | `DynamicTexture(NativeImage)` | `getPixelRGBA` / `setPixelRGBA` — **ABGR**, R/B надо менять местами | `getMinBuildHeight()` |
| 1.21.2–1.21.4 | `DynamicTexture(NativeImage)` | `getPixel` / `setPixel` — ARGB | `getMinBuildHeight()` |
| 1.21.5+ | `DynamicTexture(Supplier<String>, NativeImage)` | `getPixel` / `setPixel` — ARGB | `getMinY()` |
| 26.x | `DynamicTexture(Supplier<String>, NativeImage)` | `getPixel` / `setPixel` — ARGB | `getMinY()` |

> [!CAUTION]
> В 1.21–1.21.1 методы называются `getPixelRGBA`/`setPixelRGBA`, но работают с **ABGR**. Если передать ARGB напрямую (как в 1.21.2+), красный и синий каналы меняются местами: трава становится бирюзовой, вода — коричневой, лёд — лавандовым. Перед `setPixelRGBA` и после `getPixelRGBA` меняйте R и B: `(c & 0xFF00) | (c & 0xFF) << 16 | (c >> 16) & 0xFF | c & 0xFF000000`.

> 1.21.9–1.21.10 по GUI совпадают с 1.21.6–1.21.8. В 1.21.9 `GuiGraphics.renderOutline` заменён на `submitOutline`, а ввод снова перешёл на event-API (`MouseButtonEvent`/`KeyEvent`/`CharacterEvent`). В 1.21.11 `ResourceLocation` переименован в `Identifier`, поэтому ветку 1.21.11 проще собирать от 26.x-кода, а не от 1.21.10.
>
> В 1.21.11 у `AbstractTexture` больше нет `setFilter()`: фильтр задаётся через `RenderSystem.getSamplerCache().getClampToEdge(FilterMode.LINEAR)` (как в 26.x).
>
> Также в 1.21.11: `GuiGraphics.renderOutline` (а не `submitOutline`, как в 1.21.9–1.21.10) и `ResourceKey.identifier()` (а не `location()`).

### Tooltip и текстовые поля

| Версия | Tooltip | Многострочное поле |
| :--- | :--- | :--- |
| 1.21–1.21.1 | `g.renderTooltip(font, component, x, y)` | `new MultiLineEditBox(...)` |
| 1.21.2–1.21.5 | `g.renderTooltip(...)` / `g.setTooltipForNextFrame(...)` | `new MultiLineEditBox(...)` |
| 1.21.6+ | `g.setTooltipForNextFrame(font, component, x, y)` | `MultiLineEditBox.builder()...build(font, w, h, title)` |
| 26.x | `g.setTooltipForNextFrame(...)` | `MultiLineEditBox.builder()` |

## 4. Ввод

| Версия | Клик мыши | Клавиша | Ввод символа | Категория `KeyMapping` | `InputConstants.Type` |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1.21–1.21.8 | `mouseClicked(double, double, int)` | `keyPressed(int keyCode, int scanCode, int modifiers)` | `charTyped(char, int)` | `String` (`"key.categories.<mod>.main"`) | `KEYSYM` |
| 1.21.9–1.21.11 | `MouseButtonEvent` + `boolean` | `KeyEvent` | `CharacterEvent` | `KeyMapping.Category.register(...)` | `KEYSYM` |
| 26.1–26.2 | `MouseButtonEvent` + `boolean` | `KeyEvent` | `CharacterEvent` | `KeyMapping.Category.register(Identifier)` | `KEYSYM` |
| 26.3 | `MouseButtonEvent` + `boolean` | `KeyEvent` | `CharacterEvent` | `KeyMapping.Category.register(Identifier)` | `KEYBOARD` |

В 1.21.x:

- `e.x()` → `mouseX`, `e.y()` → `mouseY`, `e.button()` → `button`.
- `e.isEscape()` → `keyCode == InputConstants.KEY_ESCAPE`.
- `e.hasControlDown()` → `hasControlDown()` (статический метод `Screen`).
- `ModKeys.X.matches(e)` → `ModKeys.X.matches(keyCode, scanCode)`.
- `super.keyPressed(e)` → `super.keyPressed(keyCode, scanCode, modifiers)`.
- `iconSearch.keyPressed(e)` → `iconSearch.keyPressed(keyCode, scanCode, modifiers)`.
- `charTyped(CharacterEvent e)` → `charTyped(char chr, int modifiers)`.

## 5. Миксины вместо Fabric API

| Хук | Цель миксина | Примечание |
| :--- | :--- | :--- |
| Клиентский тик | `Minecraft.tick()` TAIL | — |
| Выход из игры | `Minecraft.close()` HEAD | — |
| Вход в мир | `ClientPacketListener.handleLogin(...)` TAIL | — |
| Выход из мира | `ClientPacketListener.close()` HEAD | — |
| Загрузка чанка | `ClientChunkCache.replaceWithPacketData(...)` RETURN | сигнатура зависит от версии, см. ниже |
| HUD/миникарта | `Hud.extractRenderState(...)` или `Gui.extractRenderState(...)`/`Gui.render(...)` TAIL | по версии |
| Кнопки титульного экрана | `TitleScreen.init()` TAIL + `@Invoker Screen.addRenderableWidget` | — |
| Клавиши | `Options.load()` HEAD | `@Shadow @Final @Mutable KeyMapping[] keyMappings` + `System.arraycopy` |

### Ловушка: `ClientChunkCache.replaceWithPacketData`

| Версия | 4-й аргумент |
| :--- | :--- |
| 1.21–1.21.3 | `Map<Heightmap.Types, long[]>` |
| 1.21.4 | `CompoundTag` |
| 1.21.5–1.21.8 | `Map<Heightmap.Types, long[]>` |
| 26.1–26.2 | `Map<Heightmap.Types, long[]>` |
| 26.3 | `ClientboundLevelChunkPacketData` |

Mixin-аннотации не проверяются компилятором: успешная сборка **не гарантирует**, что инъекция применится. Ошибка проявляется только при запуске:
`Invalid descriptor ... Expected (...) but found (...)`.
Всегда сверяйте сигнатуру через `javap` по deobf-JAR нужной версии.

### Регистрация клавиш без Fabric API

```java
@Mixin(Options.class)
public class OptionsMixin {
    @Shadow @Final @Mutable public KeyMapping[] keyMappings;

    @Inject(method = "load", at = @At("HEAD"))
    private void mod$appendKeys(CallbackInfo ci) {
        KeyMapping[] custom = ModKeys.createAll(); // пустой массив при повторном вызове
        if (custom.length == 0) return;
        KeyMapping[] merged = Arrays.copyOf(keyMappings, keyMappings.length + custom.length);
        System.arraycopy(custom, 0, merged, keyMappings.length, custom.length);
        keyMappings = merged;
    }
}
```

`Options.load()` вызывается в конце конструктора `Options`, поэтому клавиши успевают попасть в список до применения сохранённых биндов и до экрана управления.

## 6. Автотесты (Fabric Client GameTest)

### 6.1. Когда доступны

Модуль `fabric-client-gametest-api-v1` входит в Fabric API **начиная с 1.21.5** (и во все 26.x). В 1.21–1.21.4 его нет — там клиентские screenshot-тесты недоступны.

### 6.2. Подключение

```groovy
fabricApi {
    configureTests {
        createSourceSet = true
        modId = "mapminecraft-test"
        enableGameTests = false
        enableClientGameTests = true
        eula = true
    }
}

dependencies {
    // Именно modImplementation, а не gametestImplementation:
    // gametestImplementation не ремапит Fabric API, и компилятор видит class_310 вместо Minecraft.
    modImplementation "net.fabricmc.fabric-api:fabric-api:${project.fabric_version}"
}
```

### 6.3. Различия API тестов

| Версия | Контекст мира | Ресайз GUI | Масштаб GUI |
| :--- | :--- | :--- | :--- |
| 26.1+ | `sp.getClientLevel().waitForChunksRender()` | `mc.resizeGui()` | `int` |
| 1.21.5 | `sp.getClientWorld().waitForChunksRender()` | `mc.resizeDisplay()` | `double` (нужен `(int)`) |

### 6.4. Network synchronizer

На 1.21.6+ тест может упасть с:
`Network synchronizer in invalid state` / `Detected interfacing with packets at a lower level`.
Решение — отключить синхронизатор для тестового запуска:

```groovy
tasks.matching { it.name == 'runClientGameTest' }.configureEach {
    jvmArgs '-Dfabric.client.gametest.disableNetworkSynchronizer=true'
}
```

### 6.5. Запуск

```bash
./gradlew clean build runClientGameTest
```

Скриншоты: `build/run/clientGameTest/screenshots/`.
Перед повторным прогоном удаляйте `build/run/clientGameTest`, чтобы не тянуть несовместимый мир от другой версии.

## 7. Имена артефактов

Требуемая схема:

```
AtlasMap-<Loader>-<MC>-v<version>.jar
AtlasMap-Fabric-1.21-v1.0.jar
AtlasMap-NeoForge-26.3-v1.0.jar
```

В `gradle.properties`:

```properties
mod_version=1.0
archives_base_name=AtlasMap-Fabric-1.21
```

В `build.gradle`:

```groovy
def artifactVersion = "v${project.version}"

jar {
    archiveFileName = "${project.archives_base_name}-${artifactVersion}.jar"
}

// Loom 1.17 не имеет метода remapJar(), Loom 1.18 — имеет.
// tasks.matching безопасно настраивает задачу, если она существует.
tasks.matching { it.name == 'remapJar' }.configureEach {
    archiveFileName = "${project.archives_base_name}-${artifactVersion}.jar"
}

task jarNeoForge(type: Jar) {
    archiveFileName = "${project.archives_base_name.replace('-Fabric-', '-NeoForge-')}-${artifactVersion}.jar"
}
```

Внутри `fabric.mod.json` и `neoforge.mods.toml` версия остаётся `1.0` (без `v`).

## 8. Чек-лист порта на новую версию

1. Создать ветку `<mc_version>` от ближайшей готовой.
2. Обновить `gradle.properties`: `minecraft_version`, `archives_base_name`, `fabric_version`.
3. Проверить `loom`/Gradle/Java-toolchain и `mappings`.
4. Собрать `compileClientJava` — исправить API-ошибки.
5. Сверить **все** mixin-сигнатуры через `javap` (особенно `ClientChunkCache` и HUD).
6. Проверить `compatibilityLevel` (`JAVA_21`/`JAVA_25`).
7. Подключить client-gametest, если версия его поддерживает.
8. Запустить `clean build runClientGameTest`, проверить скриншоты и отсутствие `Mixin apply` ошибок.
9. Обновить `README.md`/`README.en.md` и таблицу веток.

## 9. Ловушки автоматизации

- Не передавайте вложенные массивы пар в PowerShell-функцию без явной типизации: PowerShell может «сплющить» массив, и `.Replace($pair[0], $pair[1])` начнёт заменять отдельные символы (например, `l` → `e`), ломая Java-файлы. Для массовых правок безопаснее `edit`/`write` или явные последовательные `.Replace()`.
- После каждой массовой замены проверяйте `git diff --stat` и компиляцию, а не только успешный `build`.

## 10. Функциональные автотесты механик

Скриншоты проверяют внешний вид, но не логику. Для механик добавлен отдельный `MapMechanicsTest` (Fabric client gametest), который выполняет ~67 проверок:

- **Storage:** `sanitize`, `asciiId`, `dimFolder`, JSON round-trip маркера, очистка тестового мира.
- **Marker:** `copy()`/`apply()`, глубокая копия `screenshots` и `shotDrawings`.
- **Stroke:** `addPoint`, `pointCount`, координаты, типы.
- **Theme/T:** 8 тем, уникальные id, каталог иконок, поиск (`cave`, `дом`), переводы.
- **MapView:** world↔screen, `zoomAt`, `contains`.
- **ModConfig:** сохранение/чтение JSON, `clamp()` индексов, восстановление значений.
- **MapManager/MapLayer/MapTile:** live-сессия, измерение, скан чанков, пиксели тайла, `pack`.
- **MapSession:** сохранение/перезагрузка маркеров и рисунков.
- **ImageCache:** `importImage`, файл на диске, сохранение в маркере, удаление.

Тест входит в `fabric-client-gametest` entrypoint и запускается тем же:

```bash
./gradlew clean build runClientGameTest
```

> На 1.21–1.21.4 клиентского gametest API в Fabric API ещё нет, поэтому механики там проверяются только сборкой.

## 11. Текстовые поля: паритет с 26.3 на 1.21.x

В 26.3 у полей отключены тень и ванильный фон. В 1.21.x это доступно не сразу:

| Версия | `EditBox.setTextShadow` | `MultiLineEditBox` |
| :--- | :--- | :--- |
| 1.21–1.21.5 | нет | конструктор; тёмный фон, рамка и счётчик |
| 1.21.6+ | да | builder с `setTextShadow(false)`, `setShowBackground(false)`, `setShowDecorations(false)` |

Решения для 1.21–1.21.5:

- `EditBoxMixin` — `@Redirect` вызовов `GuiGraphics.drawString(...)` в `EditBox.renderWidget` на перегрузку без тени.
- `MultiLineEditBoxMixin` — в `renderContents` рисовать текст цветом `Paper.INK` без тени, в `renderDecorations` не рисовать счётчик.
- `AbstractScrollWidgetMixin` (1.21–1.21.3) или `AbstractTextAreaWidgetMixin` (1.21.4–1.21.5) — отменить `renderBackground`/`renderBorder` для `MultiLineEditBox`, чтобы был виден пергаментный фон мода.

Итог: поле описания на 1.21.x выглядит как в 26.3 — пергамент, тёмный текст, без счётчика.

## 12. Ресурсы мода без Fabric API

Ключевой факт: **Fabric Loader сам не монтирует ресурсы модов**. Пакеты `assets/<modid>` (текстуры, `lang`, `icons.json`) появляются в `ResourceManager` только через модуль `fabric-resource-loader-v0` из Fabric API. Если мод объявляет «ноль зависимостей», но рисует свои текстуры, на чистом Fabric он покажет magenta/black missing texture, а переводы — сырые ключи. Ошибка тихая: в логе только `Failed to load texture: <modid>:...`.

Решения для zero-dependency мода:

1. **Текстуры** — читать PNG из classpath и регистрировать в `TextureManager`:
   ```java
   var image = NativeImage.read(Paper.class.getResourceAsStream("/assets/" + id.getNamespace() + "/" + id.getPath()));
   var texture = new DynamicTexture(image); // 1.21–1.21.4
   var texture = new DynamicTexture(() -> "mymod texture", image); // 1.21.5+
   Minecraft.getInstance().getTextureManager().register(id, texture);
   ```
   Регистрировать лениво перед первым `blit`. `TextureManager.getTexture(id)` возвращает **missing-texture placeholder, а не null**, поэтому проверять «уже загружено» нужно по своему `Map<Identifier, DynamicTexture>` и сравнивать `getTexture(id) == свояТекстура`. После F3+T/рестарта ресурсов записи очищаются — при следующем кадре текстура загрузится заново.
2. **Переводы** — читать `en_us.json`/`ru_ru.json` из classpath в `Map<String,String>` и форматировать `%s`/`%d` самостоятельно (например, `String.format`). `Component.translatable` оставить только как fallback для случаев, когда ресурсы всё-таки смонтированы.
3. **Иконка мода** (`fabric.mod.json → icon`) без resource-loader может не отображаться в Mod Menu — это не влияет на игру.

Проверка: запуск на «пустом» Fabric-инстансе без Fabric API. В логе не должно быть `Failed to load texture: <modid>:`; на титульном экране иконки мода должны рисоваться, а не magenta/black.

> [!NOTE]
> На NeoForge такой проблемы нет: FML монтирует ресурсы мода сам.

## 13. Текст, обводка, EditBox и многострочные подсказки

Сводка различий, которые ловятся только компиляцией на конкретной ветке:

| Что | 1.21–1.21.10 | 1.21.11+ / 26.x |
| :--- | :--- | :--- |
| Текст | `g.drawString(font, s, x, y, color, shadow)` | `g.text(font, s, x, y, color, shadow)` |
| Обводка | `Paper.outline(g, x, y, w, h, color)` | `g.outline(x, y, w, h, color)` |
| Поле ввода | `editBox.render(g, mx, my, 0)` | `editBox.extractRenderState(g, mx, my, 0)` |
| `ResourceKey` | `.location()` | `.identifier()` |
| Текущий экран | `mc.screen` | `mc.gui.screen()` (с 26.2) |
| Клик мыши | `mouseClicked(double, double, int)` (до 1.21.8) | `MouseButtonEvent` (с 1.21.9) |

### Многострочные подсказки

`Component` с символом `\n` **не переносится** ванильным тултипом: `\n` рисуется как глиф (в игре выглядит как «LF» или мусорные символы поверх текста). Нужно разбивать строку и передавать список компонентов:

```java
String tip = tooltip.getString();
if (tip.indexOf('\n') >= 0) {
    List<Component> lines = new ArrayList<>();
    for (String line : tip.split("\n", -1)) {
        lines.add(Component.literal(line));
    }
    // 1.21–1.21.5
    g.renderComponentTooltip(font, lines, mx, my);
    // 1.21.6+ и 26.x
    // g.setComponentTooltipForNextFrame(font, lines, mx, my);
} else {
    // 1.21–1.21.5: g.renderTooltip(font, tooltip, mx, my)
    // 1.21.6+:     g.setTooltipForNextFrame(font, tooltip, mx, my)
}
```

### Хитбоксы кастомных иконок

Если иконки рисуются не через общий `hit(...)`, а по своим координатам (например, адаптивная панель, где иконка уже ячейки), нужен **отдельный список хитбоксов** и проверка в `mouseClicked` **до** `super.mouseClicked(...)`:

```java
if (runToolHits(mx, my, button)) return true;
if (super.mouseClicked(mx, my, button)) return true;
```

Важно: компиляция такой регресс **не ловит** — список заполняется, но обработчик не вызывается, и панель «мертва». После любого переноса проверять на каждой ветке наличие самого вызова:

```bash
grep -n "if (runToolHits(" src/client/java/.../MapScreen.java
```

На `26.x` левая кнопка мыши = `1` (`InputConstants.MOUSE_BUTTON_LEFT`), на `1.21.x` = `0`, поэтому сравнивать только с константой (`pressButton == InputConstants.MOUSE_BUTTON_LEFT`), а не с числом.

### Хитбокс должен совпадать с областью подсветки

Самая частая причина жалобы «кнопка не нажимается с первого раза» — визуальная область больше кликабельной. Пример из AtlasMapExtended: заголовок секции списка объектов подсвечивался при наведении на всю ширину, а клик ловился только на `w - 54`, чтобы не мешать иконкам сортировки и «глаза». Текст «Метки (12)» на узком списке (130–180 px) выходил за эту границу, и клик по последним буквам ничего не делал. Воспроизводилось «через раз»: по началу заголовка всё работало, по концу — нет.

Правило: область клика = область подсветки, а вложенные кнопки регистрируются **позже** — при обходе списка хитбоксов с конца они получают приоритет:

```java
hit(x, y, w, 18, toggle);          // вся вкладка
hit(sx, y + 1, 16, 16, next, prev); // иконки перекрывают её в своих 16x16
hit(ex, y + 1, 16, 16, eye);
```

Хитбоксы, которые регистрируются внутри скролла, нужно обрезать по видимой области, иначе прокрученный заголовок продолжает ловить клики там, где его уже не видно:

```java
int hy = Math.max(y, top);
int hh = Math.min(y + 18, bottom) - hy;
if (hh > 0) hit(x, hy, w, hh, toggle);
```

### Как проверить хитбоксы клиентским gametest'ом

Компиляция такие баги не ловит — нужен настоящий клик. Fabric client gametest (`fabric-client-gametest-api-v1`, 1.21.5+/26.x) это умеет:

```java
public class ListHeaderClickTest implements FabricClientGameTest {
    @Override
    public void runTest(ClientGameTestContext ctx) {
        TestInput input = ctx.getInput();
        input.resizeWindow(1280, 720);
        // ... открыть экран, посчитать координаты в GUI-пикселях ...
        int[] p = ctx.computeOnClient(mc -> {
            int s = mc.getWindow().getGuiScale();
            return new int[]{gx * s, gy * s};   // TestInput работает в пикселях окна
        });
        for (int mode : new int[]{10, 60, 200, -51, -6}) {
            // разные точки одной и той же вкладки
            input.setCursorPos(p[0], p[1]);
            ctx.waitTicks(2);                    // хитбоксы строятся в render()
            input.pressMouse(InputConstants.MOUSE_BUTTON_LEFT);
            ctx.waitTicks(2);
            // public boolean markersSectionOpen() — проверять состояние, а не скриншот
        }
    }
}
```

Что важно:

- координаты пересчитывать через `guiScale` — `TestInput` принимает пиксели окна, а GUI рисуется в скалированных;
- между `setCursorPos` и кликом ждать тик: список хитбоксов перестраивается в начале кадра, до этого он ещё от прошлого кадра;
- проверять состояние через маленький публичный аксессор (`public boolean markersSectionOpen()`), а не по скриншоту;
- кликать дважды в одну точку: первый клик включает, второй выключает — так сразу видно «мёртвую» зону;
- прогон до фикса полезен как доказательство: тест печатает, какие точки не сработали (`mode=200 opened=false`), и после фикса те же точки проходят.

### Вложенные области: побеждает самая маленькая

Если районы вложены друг в друга, «первый попавший» или «последний в списке» зависит от порядка создания: пользователь кликает внутри малого района, а открывается большой (или не открывается ничего). Правило, не зависящее от порядка: среди всех видимых областей, содержащих точку, брать **наименьшую по площади**.

```java
public static Region innermostAt(List<Region> regions, String dim, double x, double z) {
    Region best = null;
    double bestArea = Double.MAX_VALUE;
    for (Region r : regions) {
        if (r.hidden || !r.dim.equals(dim) || !r.contains(x, z)) continue;
        double area = r.area();               // площадь по формуле шнуровки
        if (best == null || area < bestArea) { best = r; bestArea = area; }
    }
    return best;
}
```

Тот же приём используется и для клика по карте (`regionAt`), и для уведомлений о входе/выходе.

Уведомления: нельзя останавливаться на первом найденном районе (`break`). Нужно хранить **множество** районов, внутри которых находится игрок, и сравнивать с предыдущим снимком: появился новый `id` — «вход», пропал — «выход». При вложенности игрок одновременно находится в нескольких районах, поэтому переход из малого в большой не должен выглядеть как «выход из большого».

Плашка одна, поэтому сообщения нужно ставить в очередь, иначе «выход из малого» мгновенно заменится на «вход в большой» и пользователь увидит только второе:

```java
public static void show(String heading, String regionName) {
    if (ticks > 0) {
        queue.addLast(new String[]{heading, regionName});   // показываем после текущей
        return;
    }
    ...
}
```

Важно: сравнивать районы по `id`, а не по имени — они могут называться одинаково.
