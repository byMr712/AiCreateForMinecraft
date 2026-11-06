# 06. Клиентские запросы к миру: загруженность чанка, высоты и «-64»

> Почему `level.hasChunk(x, z)` не означает «чанк загружен», откуда берётся `-64` в высоте и как получить настоящую высоту поверхности на клиенте (1.21–26.x).

## 1. `hasChunk` — не проверка загруженности

На клиенте `ClientChunkCache` держит **один общий пустой чанк** (`emptyChunk`) и отдаёт его на любой запрос координат за пределами view distance, если запрос идёт с `requireChunk = true`. А `Level.hasChunk(int, int)` внутри как раз запрашивает чанк с этим флагом:

```java
// по факту:
public boolean hasChunk(int x, int z) {
    return this.getChunk(x, z) != null;   // → getChunk(x, z, FULL, true) → emptyChunk
}
```

Проверено байткодом 1.21.1: `ClientChunkCache.getChunk(int, int, ChunkStatus, boolean)` возвращает `emptyChunk`, если `bl == true`, и `null`, если `bl == false`. Поэтому `hasChunk(30_000_000, 30_000_000) == true`.

Следствия:

| Запрос | Результат для незагруженного чанка |
| :--- | :--- |
| `level.hasChunk(cx, cz)` | `true` (пустой чанк) |
| `level.getChunkSource().getChunkNow(cx, cz)` | `null` |
| `level.getChunkSource().getChunk(cx, cz, ChunkStatus.FULL, false)` | `null` |
| `level.getHeight(WORLD_SURFACE, x, z)` | `64` (или `32` в измерении с потолком) |
| высота непроинициализированного, но существующего чанка | `minY - 1` |

Именно поэтому «проверка `hasChunk`» пропускает мусорные значения: пустой чанк даёт `64`, недогруженный — `minY - 1` (в оверворлде `-64`).

## 2. Правильный шаблон

```java
private int surfaceY(int x, int z) {
    Minecraft mc = Minecraft.getInstance();
    if (session.isLive() && mc.level != null && dim.equals(mc.level.dimension().identifier().toString())
            && !mc.level.dimensionType().hasCeiling()
            && mc.level.getChunkSource().getChunkNow(x >> 4, z >> 4) != null) {
        int h = mc.level.getHeight(Heightmap.Types.WORLD_SURFACE, x, z);
        if (h > mc.level.getMinY()) {
            return h + 2;   // метка чуть выше поверхности
        }
    }
    return 70;              // чанк не загружен / данных нет
}
```

- `getChunkNow` есть у `ChunkSource` во всех версиях (1.21–26.x) и не требует `ChunkStatus`, который в 1.21.2 переехал из `net.minecraft.world.level.chunk` в `net.minecraft.world.level.chunk.status`.
- `h > mc.level.getMinY()` отсекает `minY - 1`.
- Не подставлять `player.getBlockY()` как «запасную высоту»: для чужого чанка это случайное число, которое потом уезжает в `/tp`.

## 3. Пересчёт при перетаскивании

`mouseDragged` меняет только `x/z`; `Y` пересчитывать **один раз на отпускании**, а не в каждом кадре:

```java
if (markerDrag) {
    markerDrag = false;
    if (working != null && selected != null
            && (working.x != selected.x || working.z != selected.z)) {
        working.y = surfaceY(working.x, working.z);
        yBox.setValue(Integer.toString(working.y));
    }
    return true;
}
```

Проверка «позиция реально изменилась» важна: клик по метке без перемещения иначе молча сбрасывал бы вручную заданный `Y`.

## 4. Как это тестировать (client gametest)

1. Открыть карту, `setView(x + 0.5, z + 0.5, 2f)` — центр экрана соответствует этой колонке.
2. ПКМ по центру создаёт метку (`onMapClick`).
3. Прочитать последнюю метку и сравнить с `level.getHeight(WORLD_SURFACE, m.x, m.z) + 2`.
4. **Выбирать колонку, где `h + 2 != 70`**, иначе тест не отличит вычисленное значение от запасного.
5. Для «незагруженной» территории: `setView(30_000_000, 30_000_000, 1f)` и ПКМ по центру → `Y == 70`.

```java
int[] spot = ctx.computeOnClient(mc -> {
    for (int r = 8; r <= 64; r += 8) for (int a = 0; a < 16; a++) {
        int x = px + (int)(Math.cos(a * Math.PI / 8) * r);
        int z = pz + (int)(Math.sin(a * Math.PI / 8) * r);
        if (mc.level.getChunkSource().getChunkNow(x >> 4, z >> 4) == null) continue;
        int h = mc.level.getHeight(Heightmap.Types.WORLD_SURFACE, x, z);
        if (h > mc.level.getMinY() && h + 2 != 70) return new int[]{x, z};
    }
    return new int[]{px, pz};
});
```

## 5. Чек-лист

- [ ] Ни одна высота из `getHeight` не используется без `h > level.getMinY()`.
- [ ] Загруженность чанка проверяется через `getChunkNow`/`getChunk(..., false)`, а не `hasChunk`.
- [ ] Для неизвестной территории — фиксированное значение (`70`), а не координата игрока.
- [ ] `Y` пересчитывается на отпускании мыши и только если позиция изменилась.
- [ ] Тест проверяет колонку, у которой `h + 2` не совпадает с запасным значением.
