# 05. Практический перенос крупного набора фич между версиями

## 1. Главное правило: не черри-пикать диапазон между разошедшимися ветками

Если ветки `1.21.5` и `1.21.8` разошлись, команда

```bash
git cherry-pick a56d01c..d7353fc
```

может притащить **всю историю чужой ветки** (порты, переименования, старые коммиты), а не только нужные фичи. В результате получаются конфликты в `FileDialogs`, старых пакетах `mapminecraft` и десятках файлов.

Правильно:

1. Собрать список **точных хешей** фича-коммитов на эталонной ветке.
2. Черри-пикать их по одному.
3. Для следующей версии использовать уже адаптированную ветку как шаблон.

```powershell
$commits = @('d68bd52','acad859','6ef70ea','ab883b2')
foreach ($c in $commits) { git cherry-pick $c }
```

---

## 2. Схема «эталонная ветка → соседние версии»

Не нужно адаптировать каждую версию с нуля.

1. Портировать фичи на **ближайшую по API** версию (например, `1.21.8`).
2. Исправить компиляцию.
3. Запустить gametest.
4. Забрать фича-коммиты уже из этой ветки и перенести на соседние (`1.21.6`, `1.21.7`).
5. Для более старых версий сделать следующий шаблон (`1.21.5` → `1.21.4` → `1.21.3`).

Так количество ручных правок резко падает: большинство конфликтов уже разрешено в шаблоне.

---

## 3. Проекция мира в HUD до 1.21.6

В **1.21.6+** и 26.x есть готовый метод:

```java
Vec3 ndc = mc.gameRenderer.projectPointToScreen(new Vec3(x, y, z));
```

В **1.21–1.21.5** его нет. Нужно собрать матрицу самостоятельно:

```java
Camera camera = mc.gameRenderer.getMainCamera();
float fov = mc.options.fov().get().floatValue();

Matrix4f mvp = new Matrix4f(mc.gameRenderer.getProjectionMatrix(fov));
mvp.mul(new Matrix4f().rotation(new Quaternionf(camera.rotation()).conjugate()));

Vector3f p = new Vector3f(
        (float) (x - camera.getPosition().x),
        (float) (y - camera.getPosition().y),
        (float) (z - camera.getPosition().z));
mvp.transformProject(p);
```

Дальше `p.x/p.y` — NDC, `p.z` — глубина:

```java
if (p.z < -1 || p.z > 1) return; // за камерой / вне клипа
int sx = Math.round((p.x + 1) * 0.5f * guiWidth);
int sy = Math.round((1 - p.y) * 0.5f * guiHeight);
```

> [!CAUTION]
> Обязательно копируйте кватернион камеры перед `conjugate()`:
> `new Quaternionf(camera.rotation()).conjugate()`.
> Иначе можно изменить внутреннее состояние камеры.

---

## 4. Скриншоты: три разных API

### 26.3

Скриншот вызывается из `Minecraft.handleGlobalKeyPress`:

```java
Screenshot.grab(Minecraft mc, boolean panorama);
```

Перехват — `@Redirect` на этот вызов.

### 26.1–26.2 и 1.21.x

Вызов находится в `KeyboardHandler.keyPress`:

```java
Screenshot.grab(File directory, RenderTarget target, Consumer<Component> message);
```

Перехват — `@Redirect` на `Screenshot.grab(File, RenderTarget, Consumer)`.

### Уведомление в чате

- 1.21.x: `mc.gui.getChat().addMessage(component)`
- 26.1.2: `mc.gui.getChat().addClientSystemMessage(component)`

Если нужно отложить скриншот на один кадр (например, убрать world-маркеры), сохраняйте флаг и вызывайте `Screenshot.grab(...)` в `GameRenderer.render` **TAIL** — там кадр уже полностью отрисован.

---

## 5. Частые API-ловушки при переносе

| Что | Где отличается |
| :--- | :--- |
| `GuiGraphics` / `GuiGraphicsExtractor` | 1.21.x / 26.x |
| `mouseClicked(MouseButtonEvent, boolean)` | 1.21.9+ |
| `mouseClicked(double, double, int)` | 1.21–1.21.8 |
| `g.text(...)` | 26.x |
| `g.drawString(...)` | 1.21.x |
| `g.outline(...)` | 26.x |
| `Paper.outline(g, ...)` / `g.renderOutline` | 1.21.x |
| `setTextShadow(false)` | нет в 1.21.5 и ниже |
| `MultiLineEditBox.builder()` | нет в 1.21.5 и ниже, только конструктор |
| `g.nextStratum()` | нет в 1.21.5 и ниже |
| `setTooltipForNextFrame` | в 1.21.5 — `renderTooltip` |
| `g.blit(RenderPipelines.GUI_TEXTURED, ...)` | 1.21.6+ |
| `g.blit(RenderType::guiTextured, ...)` | 1.21.2–1.21.5 |
| `g.blit(ResourceLocation, ...)` | 1.21–1.21.1 |
| `Identifier` | 26.x и 1.21.11+ |
| `ResourceLocation` | 1.21–1.21.10 |
| `dimension().identifier()` | 26.x и 1.21.11+ |
| `dimension().location()` | 1.21–1.21.10 |

### Кнопки мыши в 26.x

В 26.x (SDL3) кнопки мыши **1-based**:

- `MOUSE_BUTTON_LEFT = 1`
- `MOUSE_BUTTON_MIDDLE = 2`
- `MOUSE_BUTTON_RIGHT = 3`

В 1.21.x левая кнопка — `0`. Поэтому код вида

```java
if (pressButton == 0) { /* рисование */ }
```

на 26.x не срабатывает: начинается панорамирование карты вместо рисования. Всегда сравнивайте с константой:

```java
if (pressButton == InputConstants.MOUSE_BUTTON_LEFT) { ... }
```

Это особенно легко пропустить при переносе фич с 1.21 на 26.x: на old-версии баг не проявляется.

> [!TIP]
> После переноса кода с 26.x на 1.21.x делайте пакетные замены:
> `GuiGraphicsExtractor → GuiGraphics`, `g.text → g.drawString`, `g.outline → Paper.outline(g, ...)`,
> `Identifier → ResourceLocation`, `.identifier() → .location()`.
> Затем компилируйте и исправляйте только оставшиеся точечные ошибки.

---

## 6. Проверка

- `./gradlew build` на каждой ветке.
- Если есть `runClientGameTest` — запускать его: он ловит несовпадение сигнатур миксинов и проблемы ввода.
- На старых ветках (1.21–1.21.4) задачи gametest может не быть — тогда минимум проверяйте запуск клиента и отсутствие ошибок миксинов.
- Сверять содержимое jar: наличие новых классов и версия в манифесте.
