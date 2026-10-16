# 02. Универсальный 35-шаговый чек-лист портирования и релиза

Перед выпуском релиза любого мода для Minecraft Java Edition пройдите по этому списку:

- [ ] **1. Именование:** Создана директория проекта по стандарту `<ModName>-<Version>-MinecraftMod`.
- [ ] **2. Git:** Инициализирован Git-репозиторий (`git init`).
- [ ] **3. Gradle:** Настроен Gradle Wrapper (`gradle-8.12`) и файлы сборки.
- [ ] **4. Build script:** Создан скрипт быстрого запуска `build.bat`.
- [ ] **5. Gitignore:** Настроен `.gitignore` (исключены `.gradle/`, `build/`, `.idea/`).
- [ ] **6. Идентификаторы:** Все вызовы `Identifier` заменены на `Identifier.of(...)` (Fabric) или `ResourceLocation.fromNamespaceAndPath(...)` (NeoForge).
- [ ] **7. Client Item Definitions:** Созданы `assets/<ns>/items/*.json` для **всех** предметов и блоков. Контекст `on_shelf` удален.
- [ ] **8. RegistryKey Guard:** Все `Item.Settings` и `AbstractBlock.Settings` снабжены `.registryKey(...)` во избежание NPE.
- [ ] **9. Creative Tabs:** Вкладки `FabricItemGroup` содержат обязательный `.displayName(...)`.
- [ ] **10. Материалы:** `ToolMaterial` и `ArmorMaterial` обновлены под спецификацию 1.21.4. Удален неподдерживаемый слой `humanoid_baby`.
- [ ] **11. Рендер сущностей:** Модели и рендеринг адаптированы под `EntityRenderState` (`BipedEntityRenderState`).
- [ ] **12. GeckoLib:** Файлы геометрии и анимаций помещены в `geo/entity/` и `animations/entity/`.
- [ ] **13. Data Components:** NBT предметов переведен на `DataComponentTypes` (`CUSTOM_DATA`, `ATTRIBUTE_MODIFIERS`).
- [ ] **14. Сеть:** Сетевые пакеты переведены на `CustomPayload` + `PacketCodec`.
- [ ] **15. Рецепты:** Рецепты переведены на формат 1.21.4 (`{"id": "...", "count": 1}`, строковые ингредиенты, категории `food`/`blocks`).
- [ ] **16. Ванильные механики:** Зарегистрированы топливо (`FuelRegistryEvents.BUILD`), компостер (`CompostingChanceRegistry`), огонь (`FlammableBlockRegistry`).
- [ ] **17. Теги:** Созданы теги предметов и блоков (`logs_that_burn`, `mineable/*`, `beacon_base_blocks`, `enchantable/*`, `#c:`).
- [ ] **18. Локализация:** Синхронно заполнены `en_us.json` и `ru_ru.json`.
- [ ] **19. Документация:** Оформлены `README.md` и `README.en.md` для обычных игроков (без технического кода).
- [ ] **20. Чистая компиляция:** Выполнена сборка `./gradlew build` -> `BUILD SUCCESSFUL`, сгенерирован целевой JAR.
- [ ] **21. Структуры:** NBT шаблоны лежат в `data/<ns>/structure/` (в единственном числе!) и сжаты GZIP (`1F 8B`).
- [ ] **22. Сторонние схемы:** Снапшот-блоки (полок, медных сундуков) отфильтрованы при конвертации schematics.
- [ ] **23. Прозрачные блоки:** Растения, двери, саженцы зарегистрированы в `BlockRenderLayerMap` (`cutout` / `cutoutMipped`).
- [ ] **24. VertexConsumer:** При рендере полигонов `NEW_ENTITY` переданы все атрибуты (`Vertex`, `Color`, `Uv`, `Overlay`, `Light`, `Normal`).
- [ ] **25. Порталы:** Порталы используют world-space UV маппинг и запись в буфер глубины (`COLOR_DEPTH_WRITE`).
- [ ] **26. Лут-таблицы:** Используют строгий массив `conditions` (с `"condition": "minecraft:..."` внутри `terms`).
- [ ] **27. Достижения:** Предикаты сущностей оформлены списком `entity: [ { "condition": "minecraft:entity_properties", ... } ]`.
- [ ] **28. Таблички:** Модели используют текстурные переменные `"sign"`, `"board"`, `"planks"`.
- [ ] **29. Оптимизация тиков:** Исключен Stream API в `tick()`, применены `BlockPos.Mutable`.
- [ ] **30. Автономность:** Модули совместимости используют `@Pseudo` и `require = 0`.
- [ ] **31. Шейдеры:** Файлы шейдеров размещены в `assets/<ns>/shaders/` (а не в `shaders/core/`).
- [ ] **32. Матрицы мира:** Соблюден порядок в `WorldRenderer.render(...)`: 6-й — view, 7-й — proj.
- [ ] **33. 3D предметы:** Отрисовка 3D моделей предметов в руках переведена на `ItemRendererMixin`.
- [ ] **34. Обзор мыши:** Миксины обзора таргетируют `ClientPlayerEntity.changeLookDirection`.
- [ ] **35. Текстуры UI:** В `RenderLayer` используется `getGuiTextured(...)` вместо `getText(...)`.
