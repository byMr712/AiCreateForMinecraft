# 07. Память просмотра по измерениям и метка «вы были тут»

> Как запомнить, где игрок оставил каждое измерение, восстановить его карту при переключении и показать некликабельную метку на месте выхода — без хранения лишних сущностей.

## 1. Данные

| Что | Где | Время жизни |
| :--- | :--- | :--- |
| `WorldMeta.dimPos: Map<String, double[]>` (x, z) | `meta.json` | навсегда, вместе с миром |
| `remoteView: Map<String, double[]>` | память (`MapManager`) | только текущий заход в своё измерение |
| `remoteZoom: Map<String, Float>` | память | там же |
| `curX/curZ` — позиция игрока на предыдущем тике | память | постоянно |

`dimPos` — это «точка, где вы были в измерении в момент выхода из него». `remoteView` — «где вы листали чужое измерение в этом заходе». Разделение важно: первое переживает перезапуск, второе — нет.

```java
public Map<String, double[]> dimPos = new LinkedHashMap<>();

public double[] posFor(String dim) { return dimPos == null ? null : dimPos.get(dim); }

public void setPos(String dim, double x, double z) {
    if (dimPos == null) dimPos = new LinkedHashMap<>();
    dimPos.put(dim, new double[]{x, z});
}
```

Gson сериализует такое поле в `meta.json` сам, никаких `TypeAdapter` не нужно.

## 2. Когда фиксируется точка выхода

В `MapManager.tick`, где и так отслеживается смена измерения:

```java
if (!dim.equals(liveDim)) {
    if (liveDim != null) {
        live.meta.setPos(liveDim, curX, curZ);   // позиция на предыдущем тике = где были
        live.saveMeta();
    }
    liveDim = dim;
    remoteView.clear();
    remoteZoom.clear();
    // ...
}
// в самом конце tick:
curX = mc.player.getX();
curZ = mc.player.getZ();
```

Важно: `curX/curZ` обновляются **в конце** тика. Тогда при смене измерения в текущем тике там ещё лежит позиция из старого измерения. Если обновлять в начале, запишется уже координата нового мира.

`closeLive()` (дисконнект, выход из мира) делает то же самое: `meta.setPos(liveDim, curX, curZ)` перед сохранением.

## 3. Восстановление позиции при переключении

Порядок в `switchDim` (первое совпадение выигрывает):

1. **своё измерение** → центр на игроке;
2. `remoteView(dim)` → позиция и зум из этого захода;
3. `meta.posFor(dim)` → точка выхода (переживает перезапуск), зум остаётся текущим;
4. среднее по маркерам измерения;
5. `session.layer(dim).center()` → центр исследованных тайлов;
6. `0,0`.

```java
double[] remembered = MapManager.remoteView(d);
double[] pos = remembered != null ? remembered : session.meta.posFor(d);
if (pos != null) {
    view.cx = pos[0];
    view.cz = pos[1];
    if (remembered != null) {
        Float z = MapManager.remoteZoom(d);   // зум только у сессионной памяти
        if (z != null) view.zoom = z;
    }
} else if (n > 0) { /* маркеры */ }
else { /* центр тайлов */ }
```

Запоминание панорамы: в `drawMap`, если `dim` не совпадает с измерением, где физически игрок, каждый кадр:

```java
if (session.isLive() && mc.level != null && !dim.equals(currentLevelDim)) {
    MapManager.rememberView(dim, view.cx, view.cz, view.zoom);
}
```

Так запоминаются и прокрутка, и зум, и это не требует отдельного обработчика мыши.

## 4. Центр исследованной области

Простое среднее всех тайлов уводит вид в пустоту, если игрок когда-то был далеко. Берём **самый свежий тайл** как якорь и усредняем только соседние (±4 тайла):

```java
long anchor = keys.iterator().next();
long newest = Long.MIN_VALUE;
for (long k : keys) {
    long stamp = onDisk.contains(k) ? Files.getLastModifiedTime(tileFile(k)).toMillis() : 0;
    if (stamp > newest) { newest = stamp; anchor = k; }
}
// среднее только по тайлам в радиусе 4 от якоря
```

Если время модификации недоступно (тайл только в памяти), он получает `0` и не станет якорем — это нормально.

## 5. Некликабельная метка «вы были тут»

- Рисуется только когда смотрите **чужое** измерение: `session.isLive()` и `dim != текущее измерение игрока`.
- Позиция — `meta.posFor(dim)`, то есть та же точка выхода. Отдельного объекта нет.
- Поэтому её нельзя удалить, перетащить или спрятать фильтром маркеров: это не `Marker`.
- В `hits`/`toolHits` она не регистрируется → клики проходят мимо.
- Ховер определяется вручную по прямоугольнику, и только тогда ставится тултип.

```java
public boolean drawLastSeen(GuiGraphics g, double px, double pz, int size, double mx, double my) {
    int x = Math.round(sx(px)) - size / 2;
    int y = Math.round(sy(pz)) - size / 2;
    Paper.icon(g, Paper.tool("target"), x, y, size);   // та же иконка, что у кнопки «к игроку»
    return Paper.inside(mx, my, x, y, size, size);
}
```

Вернулись в это измерение — условие перестаёт выполняться, метка исчезает сама.

### Почему `Paper.icon`, а не `blit`

`g.blit` в 1.21–26.x имеет **четыре** несовместимые сигнатуры:

| Версии | Сигнатура |
| :--- | :--- |
| 1.21, 1.21.1 | `blit(ResourceLocation, x, y, w, h, u, v, ...)` |
| 1.21.2–1.21.5 | `blit(Function<ResourceLocation, RenderType>, ...)` |
| 1.21.6–1.21.11 | `blit(RenderPipeline, ResourceLocation, ...)` |
| 26.x | `GuiGraphicsExtractor` + `RenderPipeline` |

Поэтому рисовать через свой версионно-нейтральный `Paper.icon(...)`: он в каждой ветке уже знает правильный вызов и сам вызывает `ensure(texture)`. Отличается только тип параметра (`GuiGraphics` / `GuiGraphicsExtractor`) — его подставляет скрипт переноса по содержимому файла.

## 6. Чек-лист

- [ ] `curX/curZ` обновляются в конце тика, иначе записывается позиция нового измерения.
- [ ] Точка выхода сохраняется при смене измерения **и** в `closeLive()`.
- [ ] `remoteView`/`remoteZoom` очищаются при входе в мир, выходе и смене измерения — но не при переключении карты.
- [ ] Фолбэк-цепочка: своё измерение → сессионная панорама → точка выхода → маркеры → тайлы → 0,0.
- [ ] Метка не попадает в хитбоксы и не хранится как маркер.
- [ ] Иконка рисуется через `Paper.icon`, а не напрямую `blit`.
