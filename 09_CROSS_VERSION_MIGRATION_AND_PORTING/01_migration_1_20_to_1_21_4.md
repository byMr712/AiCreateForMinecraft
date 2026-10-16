# 01. Руководство по миграции: с 1.20.x на 1.21.4

## Главная таблица изменений

| Компонент / Подсистема | В Minecraft 1.20.x | В Minecraft 1.21.4 | Решение при миграции |
| :--- | :--- | :--- | :--- |
| **Java Runtime** | Java 17 | **Java 21 LTS** | Обновить JDK и `toolchain.languageVersion` в `build.gradle` |
| **Хранение данных на предметах** | `stack.getNbt()` / `getOrCreateNbt()` | **Data Components** (`DataComponentTypes`) | Использовать `stack.get(...)` и `stack.apply(...)` |
| **Модели предметов** | `assets/<ns>/models/item/<id>.json` | **Client Item Definitions** (`assets/<ns>/items/<id>.json`) | Создать JSON-дефиниции в `items/` для каждого предмета |
| **Конструктор предметов** | `new Item.Settings()` | Требует **`.registryKey(key)`** | Обязательно вызвать `.registryKey()` во избежание NPE |
| **Фабрика Identifier** | `new Identifier("ns", "path")` | **`Identifier.of("ns", "path")`** | Заменить все `new Identifier` на статический метод `.of` |
| **Кастомные вкладки креатива** | `FabricItemGroup.builder()` | Требует **`.displayName(...)`** | Обязательно передать транслируемое имя во вкладку |
| **Схема рецептов** | `{"item": "minecraft:stick"}` | **`"minecraft:stick"`** (прямая строка) | Убрать объекты, оставить прямые строки ID или `#теги` |
| **Результат рецептов** | `{"item": "...", "count": 1}` | **`{"id": "...", "count": 1}`** | Заменить ключ `"item"` на `"id"` |
| **Папка NBT структур** | `data/<ns>/structures/` | **`data/<ns>/structure/` (ЕД. ЧИСЛО!)** | Переименовать папку в единственное число |
| **Сжатие NBT структур** | Допускался несжатый NBT | **Строго сжат GZIP (`1F 8B`)** | Проверить первые 2 байта каждого NBT (`1F 8B`) |
| **Рендеринг сущностей** | `EntityRenderer<LivingEntity>` | **`EntityRenderer<T extends EntityRenderState>`** | Перевести модели и рендереры на RenderState |
| **Сетевые пакеты** | `PacketByteBuf` | **`CustomPayload` + `PacketCodec`** | Оформить пакеты как Java record с кодеками |
| **Топливо в печи** | `FuelRegistry.INSTANCE.add(...)` | **`FuelRegistryEvents.BUILD`** | Перейти на событие билдера Fabric API |
| **3D рендер предметов** | `BuiltinItemRendererRegistry` | **Удален** | Использовать `ItemRendererMixin` |
