# 01. Оптимизация циклов тиков и сборщика мусора (GC)

## 1. Отказ от Java Stream API в горячих тиках

> [!CAUTION]
> **НАГРУЗКА НА СБОРЩИК МУСОРА (GC PRESSURE):**
> Создание экземпляров стримов (`stream()`, `filter()`, `map()`, лямбда-выражений) внутри методов, вызываемых каждый тик (`tick()`, `isWaterlogged()`, `entityInside()`, `getFluidState()`), порождает миллионы короткоживущих объектов в секунду.
> Это приводит к частым паузам сборщика мусора (GC Stop-The-World) и микрофризам сервера.

### Плохо (GC-heavy):
```java
// Вызывает создание Stream-пайплайна 20 раз в секунду на каждого моба/блок
boolean hasProperty = state.getProperties().stream()
    .filter(p -> p.getName().equals("waterlogged"))
    .findFirst()
    .isPresent();
```

### Хорошо (Zero-Allocation):
```java
// Прямой обход массива/коллекции без промежуточных аллокаций
boolean hasProperty = false;
for (Property<?> p : state.getProperties()) {
    if ("waterlogged".equals(p.getName())) {
        hasProperty = true;
        break;
    }
}
```

---

## 2. Использование переиспользуемого `BlockPos.Mutable`

В циклах проверки блоков вокруг сущности или при сканировании чанка:

```java
// Создаем ровно один объект Mutable на стеке
BlockPos.Mutable mutablePos = new BlockPos.Mutable();

for (int dx = -radius; dx <= radius; dx++) {
    for (int dy = -radius; dy <= radius; dy++) {
        for (int dz = -radius; dz <= radius; dz++) {
            mutablePos.set(centerPos.getX() + dx, centerPos.getY() + dy, centerPos.getZ() + dz);
            BlockState blockState = world.getBlockState(mutablePos);
            // Обработка блока...
        }
    }
}
```

---

## 3. Неблокирующий доступ к чанкам (`getWorldChunk` vs `getChunk`)

> [!CAUTION]
> **ПРИЧИНА ТАЙМАУТОВ СЕРВЕРА (WATCHDOG TIMEOUT):**
> Вызов `world.getChunk(chunkX, chunkZ)` в главном потоке сервера принудительно запускает **синхронную генерацию или загрузку чанка с диска**, если он еще не был загружен.
> При масштабных заклинаниях или сканировании радиуса это замораживает сервер на сотни миллисекунд.
> 
> **Решение:** Используйте неблокирующий вызов с проверкой на `null`:
```java
WorldChunk chunk = world.getChunkManager().getWorldChunk(chunkX, chunkZ);
if (chunk != null) {
    // Чанк уже загружен в память — безопасно работаем
}
```

---

## 4. Монотонные серверные таймеры

Для таймеров кулдаунов и способностей всегда используйте монотонный серверный счетчик тиков:
```java
int currentTick = player.getServer().getTicks();
```
Вместо `world.getTime()`, так как время конкретного измерения может сбрасываться командами администратора (`/time set 0`) или рассинхронизироваться при переходе через порталы.
