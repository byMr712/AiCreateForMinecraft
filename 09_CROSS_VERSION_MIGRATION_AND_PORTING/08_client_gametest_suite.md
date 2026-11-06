# 08. Полный набор Fabric client gametest: логика, ввод, высоты и стресс-тесты

> Практика AtlasMapExtended: 8 клиентских gametest'ов, которые проверяют не только «картинка похожа», но и поведение — вложенные районы, фильтр списка по измерению, память вида, высоту метки, хитбоксы заголовков и производительность на 300/500/1000 районах.

Этот файл — подробный разбор конкретного набора. Общая настройка `fabric-client-gametest-api-v1` (зависимости, `configureTests`, скриншоты, отличия API 26.x и 1.21.5) осталась в [04_multi_version_gui_input_and_tests.md](04_multi_version_gui_input_and_tests.md), §6. Здесь — что именно запускается, что каждый тест доказывает и на чём эти тесты обычно падают.

## 1. Состав набора

Ветка `26.3` содержит 8 тестов:

| Тест | Тип | Что проверяет | Скриншоты |
| :--- | :--- | :--- | :--- |
| `MapClientGameTest` | сценарный / визуальный | весь пользовательский путь: карта, биом, список, редактор метки, иконки и цвет, скриншоты, рисование, линейка, темы, телепорт, офлайн-атлас | 30+ (`01_title` … `16_offline_map`, `theme_*`) |
| `ThumbnailShots` | постановочный / визуальный | красивые кадры 1920×1080 для обложки: исследованный мир, метки на суше и воде, маршруты, clean mode, панели, выбор иконок | `thumb_*` |
| `MapMechanicsTest` | функциональный | 89 проверок данных и механик без опоры на пиксели | нет |
| `ListHeaderClickTest` | ввод / хитбоксы | клик в любую точку подсвеченного заголовка «Метки» открывает и закрывает секцию | `list_header_before/after` |
| `MarkerHeightTest` | мир / высоты | метка получает `surface + 2`, а в незагруженном чанке — ровно `70` | `marker_heights` |
| `ListDimensionTest` | UI / состояние | список объектов показывает только выбранное измерение и переключается вместе с картой | `list_dim_overworld/nether` |
| `DimensionMemoryTest` | UI / состояние | вид чужого измерения восстанавливается, панорамирование запоминается, метка «вы были тут» стоит в точке выхода | `dim_memory_pin`, `dim_memory_pin_tooltip` |
| `RegionPerfTest` | стресс / метрики | 300/500/1000 районов по 200 точек, время `drawRegions` и запекания, нормальный вид на 3 крупных районах | `region_perf_*`, `region_normal` |

Важно: `RegionPerfTest` **не падает** по порогу времени. Он печатает метрики и делает скриншоты; решение «стало хуже» принимает человек, сравнивая прогоны на одной машине. Остальные тесты падают через `AssertionError`.

## 2. Где лежат тесты и как они подключаются

```
src/gametest/
├── java/org/webtrade/atlasmapextended/test/
│   ├── MapClientGameTest.java
│   ├── ThumbnailShots.java
│   ├── MapMechanicsTest.java
│   ├── ListHeaderClickTest.java
│   ├── MarkerHeightTest.java
│   ├── ListDimensionTest.java
│   ├── DimensionMemoryTest.java
│   └── RegionPerfTest.java
└── resources/fabric.mod.json
```

`src/gametest/resources/fabric.mod.json` — отдельный тестовый мод:

```json
{
  "id": "atlasmapextended-test",
  "environment": "client",
  "entrypoints": {
    "fabric-client-gametest": [
      "org.webtrade.atlasmapextended.test.MapClientGameTest",
      "org.webtrade.atlasmapextended.test.ThumbnailShots",
      "org.webtrade.atlasmapextended.test.MapMechanicsTest",
      "org.webtrade.atlasmapextended.test.ListHeaderClickTest",
      "org.webtrade.atlasmapextended.test.MarkerHeightTest",
      "org.webtrade.atlasmapextended.test.ListDimensionTest",
      "org.webtrade.atlasmapextended.test.DimensionMemoryTest",
      "org.webtrade.atlasmapextended.test.RegionPerfTest"
    ]
  },
  "depends": {
    "atlasmapextended": "*"
  }
}
```

Правила:

1. Каждый класс реализует `net.fabricmc.fabric.api.client.gametest.v1.FabricClientGameTest` и имеет единственный метод `runTest(ClientGameTestContext ctx)`.
2. Порядок классов в массиве entrypoint — это порядок запуска. `MapClientGameTest` стоит первым, потому что он самый длинный и лучше всего показывает, что клиент вообще жив.
3. `depends.atlasmapextended` обязателен: тестовый мод не соберётся в рантайме без основного.
4. Тестовый исходник **не попадает** в релизный JAR: это отдельный source set, а не `src/client`.
5. Все тесты выполняются **последовательно в одном клиентском процессе**, а не в изолированных JVM. Общий `ModConfig`, общий каталог мира и общий `ImageCache` — поэтому уборка после теста обязательна.

## 3. Конфигурация по веткам

Набор развивался вместе с модом, поэтому на разных ветках включены разные тесты:

| Ветки | Source set | Зависимость Fabric API | `disableNetworkSynchronizer` | Какие тесты реально запускаются |
| :--- | :--- | :--- | :--- | :--- |
| 1.21–1.21.4 | **нет** | — | — | никакие; файлы лежат как заготовка для переноса |
| 1.21.5 | есть | `modImplementation` | нет | `MapClientGameTest`, `MapMechanicsTest`, `ThumbnailShots` |
| 1.21.6–1.21.11 | есть | `modImplementation` | да | те же три |
| 26.1–26.3 | есть | `gametestImplementation` | нет | на `26.3` — все 8 |

### 3.1. Почему на 1.21–1.21.4 тестов нет

Модуль `fabric-client-gametest-api-v1` появился в Fabric API только с **1.21.5**. На `1.21`–`1.21.4` в `build.gradle` нет блока `fabricApi { configureTests { ... } }`, поэтому каталог `src/gametest` не подключён как source set и не компилируется. Файлы там оставлены, чтобы при переносе не начинать с нуля, но запустить их на этих ветках нельзя. Это тот случай, когда «файл есть» не значит «тест работает».

### 3.2. `modImplementation` против `gametestImplementation`

На `1.21.5`–`1.21.11` используется:

```groovy
modImplementation "net.fabricmc.fabric-api:fabric-api:${project.fabric_version}"
```

На `26.1+` — короче:

```groovy
gametestImplementation "net.fabricmc.fabric-api:fabric-api:${project.fabric_version}"
```

Причина: `gametestImplementation` не ремапит Fabric API, и компилятор видит `class_310` вместо `Minecraft`. На старых ветках это ломает компиляцию тестов, поэтому там нужен именно `modImplementation`. На `26.x` метаданные Loom уже позволяют `gametestImplementation`.

Версия `fabric_version` в `gradle.properties` обязана соответствовать ветке Minecraft: `0.128.2+1.21.5`, `0.136.1+1.21.8`, `0.165.0+26.3` и т.д. Нельзя брать версию от `26.3` на `1.21.x`.

### 3.3. Полный блок `configureTests` на 26.3

```groovy
fabricApi {
    configureTests {
        createSourceSet = true
        modId = "atlasmapextended-test"
        enableGameTests = false          // серверные game tests не нужны
        enableClientGameTests = true     // нужны клиентские
        eula = true                      // автоматически принимает EULA для тестового мира
    }
}
```

`createSourceSet = true` создаёт `src/gametest` и задачу `runClientGameTest`. `enableGameTests = false` отключает серверный вариант, чтобы не тянуть лишний entrypoint.

### 3.4. Network synchronizer на 1.21.6+

На `1.21.6+` клиентский тест может упасть с:

```
Network synchronizer in invalid state
Detected interfacing with packets at a lower level
```

Лечится отключением синхронизатора только для тестового запуска:

```groovy
tasks.matching { it.name == 'runClientGameTest' }.configureEach {
    jvmArgs '-Dfabric.client.gametest.disableNetworkSynchronizer=true'
}
```

На `26.x` этот флаг обычно не нужен. Если добавить его без необходимости — вреда нет, но и смысла тоже.

## 4. Запуск

### 4.1. Полный прогон

```powershell
$env:JAVA_HOME = '<путь к JDK 25>'   # для 1.21.x — JDK 21
.\gradlew clean build runClientGameTest --console=plain
```

`build` перед тестом важен: он собирает и основной мод, и тестовый. На `26.x` без JDK 25 сборка упадёт на toolchain, даже если тесты не при чём.

### 4.2. Где смотреть результат

- Лог: ищите строки с префиксами `[AtlasMapExtended test]`, `[AtlasMapExtended mechanics]`, `[list header]`, `[marker height]`, `[list dim]`, `[dim memory]`, `[region perf]`.
- Скриншоты: `build/run/clientGameTest/screenshots/`.
- Падение теста — это `AssertionError` в логе и ненулевой код возврата Gradle. Сообщение обычно содержит имя проверки, например `[AtlasMapExtended mechanics] config size clamped`.

### 4.3. Повторный прогон

Перед повторным запуском удаляйте рабочий каталог теста:

```powershell
Remove-Item -Recurse -Force build\run\clientGameTest
```

Иначе можно получить мир от другой версии Minecraft или от предыдущего прогона с другими настройками. Особенно это заметно на `1.21.x`, где формат мира меняется между минорными версиями.

### 4.4. Фильтр по моду

Fabric поддерживает системное свойство:

```
-Dfabric.client.gametest.modid=atlasmapextended-test
```

Это фильтр по **id мода**, а не по имени класса. Удобно, когда в тестовом клиенте есть чужие gametest'ы; в нашем случае обычно запускается весь набор.

### 4.5. Свой seed для скриншотов

`ThumbnailShots` читает переменную окружения `MM_SEED`:

```powershell
$env:MM_SEED = '123456789'
.\gradlew runClientGameTest --console=plain
```

Без неё берётся случайный seed, и кадры отличаются от прогона к прогону. Для сравнения «до/после» по картинке seed нужно фиксировать. Для обычных тестов это не требуется: они не сравнивают пиксели.

## 5. Модель выполнения и общие помощники

### 5.1. Два потока

`runTest` выполняется на **потоке gametest**, а игра живёт на **render thread**. Поэтому:

- `ctx.runOnClient(mc -> ...)` — выполнить действие на render thread и дождаться;
- `ctx.computeOnClient(mc -> value)` — то же, но вернуть значение;
- напрямую трогать `Minecraft.getInstance()` из `runTest` нельзя;
- `ctx.waitTick()`, `ctx.waitTicks(n)`, `ctx.waitForScreen(...)`, `ctx.waitFor(...)` можно вызывать только на потоке gametest.

`ctx.waitFor(predicate)` полезнее фиксированного `waitTicks`, когда неизвестно, за сколько загрузится чанк или откроется экран. Но для UI после изменения состояния лучше `waitTicks(2)`: один тик — на применение ввода, второй — на перестройку хитбоксов в начале кадра.

### 5.2. Координаты: GUI-пиксели и пиксели окна

Самая частая ошибка. `TestInput.setCursorPos` принимает **физические пиксели окна**, а `Screen.width/height` — **GUI-пиксели** (окно / `guiScale`). Пересчёт обязателен:

```java
int[] p = ctx.computeOnClient(mc -> {
    int s = mc.getWindow().getGuiScale();
    return new int[]{guiX * s, guiY * s};
});
input.setCursorPos(p[0], p[1]);
```

Если этого не сделать, на `guiScale = 2` курсор окажется вдвое ближе к левому верхнему углу, чем нужно, и клик уйдёт мимо цели. В тестах есть готовые помощники `clickGui`, `moveGui`, `drag`, `cursor`, которые делают пересчёт сами.

### 5.3. Хитбоксы строятся в кадре

В `MapScreen` список хитбоксов пересобирается в начале `render()`. Поэтому после `setCursorPos` и после смены состояния экрана нужно подождать минимум один, лучше два тика:

```java
input.setCursorPos(x, y);
ctx.waitTicks(2);
input.pressMouse(InputConstants.MOUSE_BUTTON_LEFT);
ctx.waitTicks(2);
```

Без ожидания клик проверяет хитбоксы от **прошлого** кадра. Это главная причина «тест кликнул, а ничего не произошло».

### 5.4. Клавиши — через реальные `KeyMapping`

Экран открывается не прямым `setScreen`, а нажатием бинда:

```java
input.pressKey(ModKeys.OPEN_MAP);
ctx.waitForScreen(MapScreen.class);
```

Так тест заодно проверяет, что бинд зарегистрирован и что обработчик клавиши не сломан. Прямой `ctx.setScreen(...)` оставьте для случаев, когда нужно открыть экран, у которого нет бинда (например, `WorldsScreen` из главного меню).

`pressKey` сам ждёт тик после нажатия. Если бинд не зарегистрирован, `pressKey` бросит исключение — это тоже полезная проверка.

### 5.5. Мир для теста

Стандартная заготовка:

```java
try (TestSingleplayerContext sp = ctx.worldBuilder()
        .setUseConsistentSettings(false)
        .adjustSettings(s -> s.setAllowCommands(true))
        .create()) {
    sp.getConnection().waitForChunksRender();
    sp.getServer().runCommand("time set day");
    sp.getServer().runCommand("gamemode creative @a");
    ctx.waitTicks(80);
    ...
}
```

- `setUseConsistentSettings(false)` — обычный мир с дефолтным пресетом, а не плоский «тестовый». Так проверяются настоящие высоты, биомы и тайлы карты.
- `allowCommands(true)` — иначе `runCommand` не сработает.
- `waitForChunksRender()` — дождаться, пока клиент отрисует чанки. Без него карта может быть пустой, а `getChunkNow` — `null`.
- `waitTicks(80–400)` — дать карте просканировать чанки и появиться тайлам. Для `MapClientGameTest` используется `400`, для маленьких тестов хватает `60–80`.

`ctx.worldBuilder().create()` возвращает `AutoCloseable`; `try-with-resources` закрывает мир и корректно выходит из него. Не оставляйте мир открытым — следующий тест получит чужое состояние.

### 5.6. Скриншоты

```java
ctx.takeScreenshot("03_map");
```

Файл попадает в `screenshots/` рабочего каталога. Имена должны быть уникальными внутри прогона: одинаковые имена перезаписывают друг друга, и потом невозможно понять, какой кадр к какому шагу относится.

Скриншоты — это **артефакт для человека**, а не утверждение. Набор не использует `assertScreenshotEquals`, потому что рендер GUI отличается между 1.21.x и 26.x. Если когда-нибудь понадобится пиксельный эталон, его придётся делать отдельно для каждой ветки и каждого `guiScale`.

### 5.7. Что фреймворк проверяет после каждого теста

Это важно знать до написания первого теста. Fabric после `runTest` сам проверяет:

1. все клавиши отпущены (`clearKeysDown`);
2. размер окна сброшен, курсор поставлен в центр;
3. **ни один сервер не запущен**;
4. **клиент не подключён к миру** (`client.level == null`);
5. текущий экран — `TitleScreen`.

Если мир остался открытым или на экране висит `MapScreen`, тест упадёт уже **после** своей последней проверки, с сообщением вида:

```
Client gametest ... finished while still connected to a server
Client gametest ... did not finish on the title screen
```

Поэтому:

- мир создавайте только через `try (TestSingleplayerContext sp = ...)`, чтобы он закрылся автоматически;
- если тест закончился на экране мода, вернитесь на титульный: `ctx.setScreen(TitleScreen::new)` и `ctx.waitTicks(5)`;
- не оставляйте открытый `MapScreen` «на память» — это не ошибка самой механики, но формальное падение теста.

Перед каждым тестом фреймворк также вызывает `restoreDefaultGameOptions()`. Это сбрасывает ванильные настройки (язык, `guiScale`, render distance), но **не трогает** `atlasmapextended.json`: тему, миникарту и сортировку тест обязан восстановить сам.

## 6. Разбор тестов

Ниже — версия набора из `26.3`. На `1.21.5`–`26.2` из него есть только `MapClientGameTest`, `MapMechanicsTest` и `ThumbnailShots`; остальные тесты появились позже.

### 6.1. `MapMechanicsTest` — 89 проверок без пикселей

Тест состоит из двух частей: проверки, которым мир не нужен (выполняются сразу), и проверки живого мира.

```java
ctx.runOnClient(mc -> {
    storage();
    markerCopy();
    strokePoints();
    regions();
    themes();
    translations();
    viewTransforms();
    configPersistence();
});

try (TestSingleplayerContext sp = ...) {
    ctx.waitTicks(400);
    ctx.runOnClient(mc -> liveMap());
    ctx.runOnClient(mc -> persistence());
    ctx.runOnClient(mc -> screenshots());
}
```

Что именно проверяется:

| Группа | Проверки |
| :--- | :--- |
| `storage()` | `sanitize` мира (латиница и кириллица), формат `asciiId`, `dimFolder`, JSON round-trip маркера, удаление тестового каталога |
| `markerCopy()` | `copy()` сохраняет `id`, но делает глубокую копию `screenshots` и `shotDrawings`; `apply()` переносит изменения |
| `strokePoints()` | пустой штрих, `addPoint`, координаты, константы `pen/line/rect/circle` |
| `regions()` | площадь, `contains` для квадрата и треугольника, `copy`, `innermostAt` при разном порядке создания, скрытые районы, другое измерение, `regionAt` на карте и приоритет подписи над площадью |
| `themes()` | ровно 8 тем, уникальные id, `byId`, fallback на `papyrus`, цвет чернил |
| `translations()` | каталог ≥100 иконок, поиск по `cave` и `дом`, имя иконки, имя измерения, загруженный перевод кнопки |
| `viewTransforms()` | world↔screen, `zoomAt` с сохранением точки под курсором, `contains` |
| `configPersistence()` | запись/чтение JSON, `clamp()` индексов размера/зума/угла, восстановление значений |
| `liveMap()` | живая сессия, измерение, `meta.id`, слой, наличие тайлов, round-trip пикселя, `hasData`, `MapLayer.pack` |
| `persistence()` | маркер, штрих и район переживают `MapSession.open`; `knownDims` |
| `screenshots()` | `ImageCache.importImage`, файл на диске, сохранение имени в маркере, удаление |

Почему это важно: скриншотный тест покажет, что карта нарисовалась, но не покажет, что `copy()` сделал мелкую копию или что `clamp()` не пустил индекс `99`. Такие регрессии ловятся только проверками.

Два практических правила из этого теста:

1. **Не сравнивайте `List` по ссылке.** `copy()` должен создать новый список; проверка `a.screenshots == b.screenshots` ловит именно эту ошибку.
2. **Убирайте за собой.** Тест создаёт маркеры, штрихи, районы и файлы, а в конце удаляет их и вызывает `Storage.IO` flush. Иначе следующий прогон начнётся с грязными данными, а `MapClientGameTest` будет ждать ровно 8 маркеров и упадёт.

`Storage.IO` асинхронный. Перед повторным `MapSession.open` нужно дождаться записи:

```java
Storage.IO.submit(() -> {}).get(15, TimeUnit.SECONDS);
```

Без этого можно прочитать старый файл и получить «плавающий» тест.

### 6.2. `MapClientGameTest` — сквозной пользовательский сценарий

Этот тест — «пройти по моду как игрок» и попутно снять скриншоты. Он не пытается проверить всё; его задача — не дать сломать основной путь.

Настройка:

```java
input.resizeWindow(1600, 900);
mc.options.languageCode = "ru_ru";
mc.getLanguageManager().setSelected("ru_ru");
mc.options.guiScale().set(3);
mc.resizeGui();
mc.reloadResourcePacks();
ctx.waitTicks(60);
```

Затем создаётся мир, и тест проходит шаги:

1. `02_ingame_minimap` — миникарта в игре.
2. Создаёт 8 маркеров с разными иконками, именами и описаниями вокруг игрока.
3. `03_map` — открывает карту биндом `OPEN_MAP`.
4. Проверяет **позитивный** и **негативный** биом-подсказку:
   - на исследованном центре `hoveredBiome()` не пустой;
   - после `setView(30_000_000, 30_000_000, 1f)` подсказка `null`.
5. `04/05` — зум колесом.
6. `06` — список, `07` — просмотр маркера, `08` — редактор и ввод текста.
7. `09` — выбор иконки, поиск «дом», правый клик открывает палитру, `09c/09d` — перекраска. При этом проверяется, что цвет применился к редактируемой копии, а не к оригиналу (`w.color == 0`).
8. `10` — выбор скриншота, прикрепление, `10b` — редактор со скриншотом, `10c` — рисование поверх, `10d` — просмотр с рисунком.
9. `11` — линейка: три клика и движение курсора.
10. `12/13` — свободное рисование и выход из редактора.
11. `13a` — настройки; клик по плитке темы проверяет, что `ModConfig.get().theme` реально изменился, а не только подсветилась плитка.
12. Перебирает все 8 тем и снимает `theme_<id>`.
13. `13c` — телепорт к метке. Проверка не «экран открылся», а расстояние:

```java
double d = Math.hypot(mc.player.getX() - m.x - 0.5, mc.player.getZ() - m.z - 0.5);
if (d > 2) throw new AssertionError("teleport did not move the player, distance " + d);
```

14. Выходит из мира, открывает `WorldsScreen`, проверяет, что мир сохранился, читает его через `MapManager.openForViewing(meta)` и ждёт 8 маркеров. Затем `16_offline_map` — карта без запущенного мира.

Чему этот тест учит:

- **Проверяйте состояние, а не картинку.** «Плитка темы нажалась» — не проверка. Проверка — `theme.equals("modern_dark")`.
- **Телепорт проверяется расстоянием**, а не фактом закрытия экрана.
- **Офлайн-режим — отдельный путь.** `MapManager.openForViewing` читает мир с диска, и именно там ломаются пути к тайлам и маркерам.
- **Не полагайтесь на абсолютные координаты** элементов: они зависят от `width/height` и `guiScale`. В тесте координаты вычисляются из `mc.gui.screen().width/height` и текущих формул панели.

### 6.3. `ThumbnailShots` — постановочные кадры

Это не проверка механик, а генератор обложки. Он не должен падать от того, что мир выглядит неидеально: все «найти место» возвращают `null`, если не нашли, и просто пропускают метку.

Ключевые отличия от остальных тестов:

- окно **1920×1080**, `guiScale = 4` (потом 5 и 3 для разных кадров), `bobView = false`;
- мир создаётся с `MM_SEED`, если переменная задана;
- время, погода и `advance_time/advance_weather` фиксируются командами, чтобы кадры были стабильны;
- игрок телепортируется по 9 точкам вокруг спавна (`±170`), чтобы карта просканировала большой участок;
- маркеры ставятся **по пикселям уже нарисованной карты**: `findSpot` ищет сушу или воду по ARGB тайла, а не по координатам биома;
- вода дополнительно проверяется `nearLand`, чтобы «Порт» не оказался посреди океана;
- между метками выдерживается расстояние ≥30 блоков;
- рисуются маршрут, круг опасности и прямоугольник фермы;
- `setCleanMode(true)` убирает панели для «чистых» кадров;
- один из кадров импортируется как скриншот и прикрепляется к маркеру дома вместе с нарисованным кругом.

Список кадров: `thumb_ingame_raw`, `thumb_ingame_minimap`, `thumb_clean_big`, `thumb_clean_low`, `thumb_clean_bigger`, `thumb_clean_wide`, `thumb_map_overview`, `thumb_map_close`, `thumb_map_panels`, `thumb_icons_top`, `thumb_icons_shapes`.

Если кадр нужен для сравнения версий, фиксируйте `MM_SEED` и не меняйте `guiScale`/разрешение: даже один шаг `guiScale` меняет компоновку панелей.

### 6.4. `ListHeaderClickTest` — хитбокс = подсветка

Тест проверяет правило: **любая точка заголовка, которая подсвечивается при наведении, должна реагировать на клик.**

Он кликает в 7 точек одной и той же вкладки:

```java
for (int mode : new int[]{10, 40, TEXT_TAIL, 60, 200, -51, -6}) { ... }
```

- `10`, `40`, `60`, `200` — пиксели от левого края;
- `TEXT_TAIL` — конец текста «Метки (8)»;
- `-51`, `-6` — смещения от правого края.

В каждой точке выполняется двойной клик: первый должен открыть секцию, второй — закрыть. Проверяется публичное состояние `markersSectionOpen()`, а не скриншот:

```java
click(ctx, input, p);
boolean opened = ctx.computeOnClient(mc -> ((MapScreen) mc.gui.screen()).markersSectionOpen());
click(ctx, input, p);
boolean closed = !ctx.computeOnClient(mc -> ((MapScreen) mc.gui.screen()).markersSectionOpen());
```

Все неудачные точки собираются в список и выводятся одним `AssertionError`. Это удобнее, чем падать на первой: сразу видно, что сломано — начало, конец текста или правый край.

Почему тест появился: визуально заголовок подсвечивался на всю ширину, а клик ловился только до `w - 54`, чтобы не мешать иконкам сортировки. На узком списке текст выходил за границу, и клик по последним буквам не работал. Компиляция такое не ловит.

### 6.5. `MarkerHeightTest` — `surface + 2` и честный `70`

Тест проверяет два режима:

1. **Чанк загружен.** Найти колонку, где `getChunkNow` не `null`, высота `> minY` и `h + 2 != 70`, поставить метку правым кликом и проверить `m.y == h + 2`.
2. **Чанк не загружен.** Уйти на `30_000_000, 30_000_000`, поставить метку и проверить `m.y == 70`.

Тонкость: если проверять загруженный чанк на высоте 68, то `68 + 2 == 70`, и тест «пройдёт» даже если мод всегда ставит fallback. Поэтому координата подбирается заранее:

```java
if (h > mc.level.getMinY() && h + 2 != 70) {
    return new int[]{x, z};
}
```

Второй важный момент — `getChunkNow`, а не `hasChunk`. `hasChunk` может быть `true` для чанка, который ещё не загружен на клиенте; высота в таком случае будет мусорной. Подробнее — в [06_client_world_queries_and_heights.md](06_client_world_queries_and_heights.md).

Если тест упал с `marker fell back to 70`, это почти всегда значит, что `waitForChunksRender()` не дождался нужного чанка или координата попала на границу. Лечение — сначала `waitForChunksRender`, потом ещё несколько тиков, и выбирать точку из загруженного чанка.

### 6.6. `ListDimensionTest` — список живёт в выбранном измерении

Тест создаёт объекты в двух измерениях:

- Обычный мир: 3 метки + район `Village`;
- Нижний мир: 2 метки + районы `Fortress`, `Lava sea`.

Открывает карту, список и снимает `list_dim_overworld`. Затем переключает измерение не кнопкой «глобус», а через `showMarker(...)` на метке из Нижнего мира — это проверяет, что переход по объекту тоже меняет выбранное измерение. После этого:

```java
String nether = ctx.computeOnClient(mc -> ((MapScreen) mc.gui.screen()).shownDimension());
if (!NETHER.equals(nether)) {
    throw new AssertionError("dimension did not switch: " + nether);
}
```

Скриншот `list_dim_nether` показывает, что в списке только объекты Нижнего мира, а счётчики и заголовок совпадают с содержимым.

Тест не проверяет пиксели списка. Он проверяет **состояние** `shownDimension()`, а визуальную часть смотрит человек по двум скриншотам. Это осознанный компромисс: проверять текст в списке через OCR/пиксели хрупко.

### 6.7. `DimensionMemoryTest` — память вида и метка выхода

Тест проверяет три вещи:

1. **Восстановление из `meta.json`.** Записывает `MapManager.rememberView(NETHER, 1234, -4321, 2f)` и `meta.setPos(...)`, затем `showDimension(NETHER)` и проверяет `viewCenter()`.
2. **Память в рамках сессии.** Ставит вид `(52, 62, 3)`, уходит в Обычный мир и возвращается в Нижний; вид должен сохраниться.
3. **Метка «вы были тут».** Ставит `meta.setPos(NETHER, 62, 72)`, центрирует вид на `(60, 70)` и снимает `dim_memory_pin` + `dim_memory_pin_tooltip` (курсор наводится на точку метки).

Проверки — только по `viewCenter()`; положение иконки на экране не проверяется математически. Поэтому тест устойчив к смене темы и размеров панелей, но и не ловит «метка нарисована не там» — это смотрят на скриншоте.

Важно: тест вызывает `showDimension`, а не жмёт кнопку. Так проверяется логика памяти, а не ввод. Если сломается кнопка «глобус», это поймает `ListDimensionTest`/`MapClientGameTest`, а не этот тест.

### 6.8. `RegionPerfTest` — стресс 300/500/1000

Тест создаёт районы одного размера в сетке:

```java
for (int count : new int[]{300, 500, 1000}) {
    ctx.runOnClient(mc -> fill(count));
    ctx.waitTicks(10);
    double[] sample = ctx.computeOnClient(mc -> new double[]{
            MapView.lastRegionsNanos / 1_000_000.0,
            MapView.lastBakeNanos / 1_000_000.0,
            MapView.lastRegionsDrawn,
            MapView.lastRegionsPoints
    });
    long t0 = System.nanoTime();
    ctx.waitTicks(20);
    double frameMs = (System.nanoTime() - t0) / 1_000_000.0 / 20.0;
    ...
    ctx.takeScreenshot("region_perf_" + count);
}
```

- каждый район — 200 точек;
- районы укладываются в сетку с шагом 12 блоков;
- вид фиксируется на `(0.5, 0.5, zoom 1)`, чтобы все районы были на экране;
- после стресса районы очищаются и создаются 3 крупных «нормальных» района (`Village`, `Forest`, `Lake`) для скриншота `region_normal`.

Метрики:

| Поле | Что означает |
| :--- | :--- |
| `lastRegionsNanos` | полное время `drawRegions` за последний кадр |
| `lastBakeNanos` | время последнего пересчёта оверлея (запекания) |
| `lastRegionsDrawn` | сколько районов попало в слой после отсечения |
| `lastRegionsPoints` | сколько точек осталось после экранного упрощения |

На тестовой машине исторический прогон дал:

```
до:   300 районов ≈ 3160 мс на кадр, 500 ≈ 8773 мс
после: drawRegions ≈ 0.007–0.026 мс, запекание ≈ 12.9–16.5 мс
```

Это не эталон: цифры зависят от CPU, разрешения, `guiScale` и софтверного рендерера. Сравнивать можно только два прогона **на одной машине с одинаковыми настройками**. Подробный разбор оптимизации и таблица `drawRegions` — в [08/04_bulk_2d_overlays.md](../08_PERFORMANCE_AND_OPTIMIZATION/04_bulk_2d_overlays.md).

Что важно при чтении вывода:

- `tick=` — это грубая оценка полного кадра через `waitTicks(20)`. В неё входит ожидание тиков, поэтому абсолютное значение мало о чём говорит; смотрите на относительную разницу между 300/500/1000.
- `bake` — разовый пересчёт. Он может быть 10–20 мс и это нормально; важно, чтобы `perFrame` после запекания не рос с числом районов.
- `drawn` и `simplifiedPoints` показывают, что отсечение и упрощение реально работают. Если `drawn` равно общему числу районов при крошечном экране — bbox-отсечение сломано.
- Если `perFrame` снова начал расти пропорционально `count`, скорее всего оверлей не запекается и на каждый район снова идёт `g.fill`.

## 7. Приёмы, без которых тесты становятся флапающими

1. **Пересчитывайте GUI → окно через `guiScale`.** Иначе клики уходят мимо.
2. **Ждите 2 тика после движения курсора и после смены состояния.** Хитбоксы строятся в кадре.
3. **Не полагайтесь на координаты игрока.** Спавн и мир могут отличаться. Берите `mc.player.getBlockX()/getBlockZ()` и стройте точки относительно них.
4. **Не полагайтесь на абсолютную высоту 70.** Для загруженного чанка проверяйте `h + 2` и специально избегайте колонок, где `h + 2 == 70`.
5. **Проверяйте `getChunkNow`, а не `hasChunk`.** Иначе получите `-64` или мусорную высоту.
6. **Давайте карте время.** 60–80 тиков для маленьких тестов, 400 — для сквозного. Меньше — тайлы ещё пустые.
7. **Изолируйте данные.** Каждый тест начинается с `markers.clear()` или уникальных имён и убирает за собой. Тесты выполняются в одном клиентском процессе и видят файлы друг друга.
8. **Восстанавливайте конфиг.** `MapClientGameTest` меняет тему и `teleport`; после себя возвращает `papyrus` и сохраняет. Иначе следующий тест стартует в другой теме.
9. **Сбрасывайте `ImageCache`.** Кэш картинок статический; без `ImageCache.clear()` следующий тест может получить чужую текстуру.
10. **Дожидайтесь `Storage.IO`.** Асинхронная запись без flush даёт «то прочиталось, то нет».
11. **Не сравнивайте пиксели между версиями.** Даже одинаковый код рисует по-разному на 1.21.5 и 26.3.
12. **Удаляйте `build/run/clientGameTest` между прогонами.** Старый мир другой версии — источник загадочных падений.

## 8. Типовые падения и что они значат

| Симптом | Причина | Решение |
| :--- | :--- | :--- |
| `no live map session` | мир не успел загрузиться или `MapManager` не инициализирован | `waitForChunksRender()` + `waitTicks(80–400)`; проверить, что тест внутри `TestSingleplayerContext` |
| `biome hint did not appear` | курсор не над исследованной областью или карта ещё пустая | двигать курсор в центр после `waitTicks`, дать карте просканировать чанки |
| `biome hint appeared over unscanned map` | проверка негативного случая, карта успела дорисоваться | увести вид дальше (`30_000_000`) и подождать |
| `header click did not toggle at: ...` | клик мимо хитбокса, неверный `guiScale`, хитбоксы не перестроились | пересчитать координаты, `waitTicks(2)`, проверить `markersSectionOpen()` |
| `loaded chunk marker fell back to 70` | чанк не загружен или `getChunkNow` вернул `null` | `waitForChunksRender`, выбирать колонку из загруженного чанка |
| `marker y is not above minY` | высота взята из пустого чанка | то же |
| `dimension did not switch` | `showMarker` не переключил измерение или список смотрит не туда | проверить `showDimension`/`showMarker`, `shownDimension()` |
| `remembered view was not restored` | `meta.json` не записался или `rememberView` перетёрт `showDimension` | проверить порядок вызовов и `dimPos` |
| `panned view was not remembered` | вид не сохраняется при уходе из измерения | проверить `rememberView` в `switchDim`/`showDimension` |
| `Network synchronizer in invalid state` | 1.21.6+ не отключён синхронизатор | `-Dfabric.client.gametest.disableNetworkSynchronizer=true` |
| `class_310` / `cannot find symbol Minecraft` | `gametestImplementation` не ремапит Fabric API на старой ветке | заменить на `modImplementation` |
| тест не найден / не запускается | класс не добавлен в `fabric-client-gametest` entrypoint или source set не создан | проверить `fabric.mod.json` и `configureTests` |
| мир открывается, но всё пустое | старый `build/run/clientGameTest` от другой версии | удалить каталог и перезапустить |
| сырые ключи переводов | ресурсы мода не смонтированы (нет resource-loader) | см. [04_multi_version_gui_input_and_tests.md](04_multi_version_gui_input_and_tests.md), §12 |
| `Mixin apply` ошибка на старте | сигнатура миксина не совпала с версией | сверить через `javap`; тест тут не виноват, клиент не стартует |
| скриншоты чёрные/пустые | тест снял кадр до отрисовки или окно не того размера | `waitTicks`, `resizeWindow`, проверять `guiScale` |
| падение после успешного одиночного прогона | тест оставил маркеры/районы/конфиг | добавить cleanup и `flushIo()` |
| `did not finish on the title screen` | тест оставил открытый `MapScreen`/`WorldsScreen` | `ctx.setScreen(TitleScreen::new)` в конце |
| `finished while still connected to a server` | мир не закрыт | `try-with-resources` вокруг `TestSingleplayerContext` |

## 9. Чего этот набор не проверяет

- **NeoForge-рантайм.** Все тесты запускаются на Fabric-клиенте. NeoForge-сборка проверяется только компиляцией и ручным запуском.
- **Мультиплеер и реальную сеть.** Используется одиночный мир; `Network synchronizer` для этого и отключают.
- **Звук, шейдеры, ресурспаки, Sodium/Iris.** Визуальные модификаторы могут менять картинку, но не логику.
- **Пиксельное равенство.** Нет эталонных изображений и `assertScreenshotEquals`; регрессии внешнего вида ловит человек по скриншотам.
- **Производительность как pass/fail.** `RegionPerfTest` только печатает цифры. Порогов нет, потому что они зависят от машины.
- **Ветки 1.21–1.21.4.** Там клиентского gametest API нет; проверка только сборкой.
- **Долгие сессии и утечки памяти.** Нет прогона на сотни открытий/закрытий карты.

## 10. Чек-лист нового теста

Минимальный скелет:

```java
public class MyFeatureTest implements FabricClientGameTest {
    @Override
    public void runTest(ClientGameTestContext ctx) {
        TestInput input = ctx.getInput();
        input.resizeWindow(1280, 720);
        ctx.runOnClient(mc -> {
            mc.options.guiScale().set(2);
            mc.resizeGui();
        });

        try (TestSingleplayerContext sp = ctx.worldBuilder()
                .setUseConsistentSettings(false)
                .adjustSettings(s -> s.setAllowCommands(true))
                .create()) {
            sp.getConnection().waitForChunksRender();
            ctx.waitTicks(80);

            input.pressKey(ModKeys.OPEN_MAP);
            ctx.waitForScreen(MapScreen.class);
            ctx.waitTicks(5);

            boolean ok = ctx.computeOnClient(mc -> ((MapScreen) mc.gui.screen()).shownDimension() != null);
            if (!ok) {
                throw new AssertionError("dimension is not shown");
            }

            ctx.takeScreenshot("my_feature");
        }

        ctx.waitForScreen(TitleScreen.class);
        ctx.waitTicks(5);
    }
}
```

1. Создать класс в `src/gametest/java/.../test/`, реализовать `FabricClientGameTest`.
2. Добавить класс в `fabric-client-gametest` в `src/gametest/resources/fabric.mod.json`.
3. Задать окно и `guiScale` в начале, если тест кликает; иначе координаты будут нестабильны.
4. Создать мир через `ctx.worldBuilder()`; включить `allowCommands`, если нужны команды.
5. Дождаться `waitForChunksRender()` и 60–400 тиков.
6. Использовать `ModKeys` + `waitForScreen` для открытия экранов, если есть бинд.
7. Пересчитывать GUI-координаты в пиксели окна.
8. После `setCursorPos` и смены состояния ждать 2 тика.
9. Проверять **состояние** через публичный аксессор, а не по скриншоту.
10. Если проверка может упасть по причине «не успело» — использовать `waitFor`, а не фиксированный `waitTicks`.
11. Уникальные имена скриншотов; снимать ключевые шаги, а не каждый кадр.
12. Убирать за собой маркеры, районы, штрихи, файлы, конфиг; вызывать `flushIo()` и `ImageCache.clear()`.
13. Заканчивать тест на `TitleScreen`, без открытого мира и запущенного сервера.
14. Запустить `clean build runClientGameTest`, проверить лог и скриншоты.
15. Прогнать тест дважды подряд без ручной очистки данных между запусками — он должен оставаться зелёным. Каталог `build/run/clientGameTest` удаляйте только при смене версии Minecraft.

## 11. Связанные материалы

- [04_multi_version_gui_input_and_tests.md](04_multi_version_gui_input_and_tests.md) — общая настройка client gametest, отличия API, ввод и хитбоксы.
- [06_client_world_queries_and_heights.md](06_client_world_queries_and_heights.md) — `hasChunk` против `getChunkNow`, высоты и `-64`.
- [07_per_dimension_view_memory.md](07_per_dimension_view_memory.md) — логика памяти вида, которую проверяет `DimensionMemoryTest`.
- [08/04_bulk_2d_overlays.md](../08_PERFORMANCE_AND_OPTIMIZATION/04_bulk_2d_overlays.md) — оптимизация районов и метрики, которые читает `RegionPerfTest`.
- [05_feature_porting_workflow.md](05_feature_porting_workflow.md) — место gametest в чек-листе переноса фичи.
