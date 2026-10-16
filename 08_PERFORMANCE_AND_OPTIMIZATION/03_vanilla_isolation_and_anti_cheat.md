# 03. Ванильная изоляция и совместимость с серверами (Vanilla Isolation)

## 1. Зачем нужна ванильная изоляция?

Когда клиент с установленными модами подключается к чистому ванильному серверу или серверу с защитой античитом:
- Сервер может разорвать соединение из-за пакетов с кастомными датапаками (`ServerboundSelectKnownPacks`).
- Разница в индексах метаданных сущностей (`SynchedEntityData`) между сервером и клиентом может вызывать краш: `Invalid entity data item type`.
- Сдвиги глобальной палитры блоков приводят к рассинхронизации блоков.

---

## 2. Канонический снимок палитры ванильных блоков (`VanillaBlockStateSnapshot`)

Сразу после завершения `Bootstrap.bootStrap()` фиксируется неизменяемый снимок чистых ванильных состояний блоков:
```java
public class VanillaBlockStateSnapshot {
    private static IdList<BlockState> CANONICAL_PALETTE;

    public static void capture() {
        CANONICAL_PALETTE = new IdList<>();
        for (BlockState state : Block.BLOCK_STATE_REGISTRY) {
            if ("minecraft".equals(Registries.BLOCK.getId(state.getBlock()).getNamespace())) {
                CANONICAL_PALETTE.add(state);
            }
        }
    }

    public static BlockState getCanonicalState(int id) {
        return CANONICAL_PALETTE.get(id);
    }
}
```
При игре на ванильных серверах ID блоков разрешаются против этого снимка, что исключает сдвиги палитры.

---

## 3. Компенсация метаданных сущностей (`SynchedEntityData`)

> [!CAUTION]
> Сторонние моды могут добавлять дополнительные записи в трекер сущностей игрока (`SynchedEntityData`), сдвигая ванильные индексы на $+1, -1, +2$.
> Сервер присылает пакет с данными сущности, а клиент ожидает другой тип сериализатора, что приводит к крашу `Invalid entity data item type`.
> 
> **Решение (DataTrackerMixin):**
> Перехват чтения пакета данных сущности с защитной проверкой типа сериализатора перед применением значения.

---

## 4. Фильтрация пакетов конфигурации `SelectKnownPacks`

В фазе сетевой конфигурации клиент отправляет список известных пакетов:
```java
// Фильтрация пакета: отправляем только ванильные пакеты
ServerboundSelectKnownPacks filteredPacket = new ServerboundSelectKnownPacks(
    originalPacket.knownPacks().stream()
        .filter(pack -> "minecraft".equals(pack.namespace()))
        .toList()
);
```
Это полностью исключает кики и дисконнекты со стороны ванильных серверов из-за объявления клиентом неизвестных модовых датапаков.
