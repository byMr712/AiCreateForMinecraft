# 02. Кастомные структуры и Jigsaw (Structures в 1.21.4)

## 1. Два смертельных бага генерации структур в 1.21.4

> [!CAUTION]
> **1. ОШИБКА ИМЕНИ ПАПКИ NBT-ШАБЛОНОВ: `structure` (ЕДИНСТВЕННОЕ ЧИСЛО!):**
> В Minecraft 1.21.4 папка для NBT-файлов построек называется строго **`data/<namespace>/structure/`**, а **НЕ** `structures/` (как было в 1.20.x)!
> - **Симптом:** Если положить файл в `structures/` (множественное число), команда `/locate structure` найдет координаты (так как работает файл размещения `structure_set`), но при прилете на место чанк сгенерируется пустым, здание **не построится**, и в логе **НЕ БУДЕТ НИ ОДНОЙ ОШИБКИ**!

> [!CAUTION]
> **2. ФАЙЛЫ .NBT ОБЯЗАНЫ БЫТЬ СЖАТЫ GZIP (`1F 8B`):**
> Ванильный `StructureTemplateManager` в 1.21.4 всегда читает шаблоны через `NbtIo.readCompressed` (GZIP).
> - Если файл NBT сохранен несжатым (начинается с байта `0A`), игра выбрасывает ошибку:
>   `Couldn't load structure <namespace>:<id>` с трейсом в `GZIPInputStream.readHeader`.
> - **Проверка:** Первые два байта каждого файла `.nbt` обязаны быть шестнадцатеричными **`1F 8B`**!

### Сжатие NBT в GZIP через PowerShell:
```powershell
function Compress-NbtToGzip($sourcePath, $destPath) {
    $bytes = [System.IO.File]::ReadAllBytes($sourcePath)
    if ($bytes[0] -eq 0x1F -and $bytes[1] -eq 0x8B) {
        Write-Host "Файл уже сжат GZIP!"
        return
    }
    $ms = New-Object System.IO.MemoryStream
    $gz = New-Object System.IO.Compression.GZipStream($ms, [System.IO.Compression.CompressionMode]::Compress)
    $gz.Write($bytes, 0, $bytes.Length)
    $gz.Close()
    [System.IO.File]::WriteAllBytes($destPath, $ms.ToArray())
    Write-Host "Сжато успешно в $destPath"
}
```

---

## 2. Полная структура файлов датапака структуры

```
src/main/resources/data/<namespace>/
├── structure/                          <- Сжатые GZIP NBT-шаблоны (ЕДИНСТВЕННОЕ ЧИСЛО!)
│   └── ancient_shrine.nbt
├── tags/worldgen/biome/has_structure/
│   └── ancient_shrine.json             <- Биомы, где разрешен спавн
└── worldgen/
    ├── structure/ancient_shrine.json   <- Параметры генерации Jigsaw
    ├── structure_set/ancient_shrine.json <- Алгоритм размещения (random_spread)
    └── template_pool/ancient_shrine.json <- Пул элементов постройки
```

---

## 3. Эталонные JSON-файлы генерации структуры

### 3.1. Параметры Jigsaw (`data/<ns>/worldgen/structure/ancient_shrine.json`):
```json
{
  "type": "minecraft:jigsaw",
  "biomes": "#examplemod:has_structure/ancient_shrine",
  "step": "surface_structures",
  "terrain_adaptation": "beard_thin",
  "spawn_overrides": {},
  "start_pool": "examplemod:ancient_shrine",
  "size": 1,
  "start_height": { "absolute": 0 },
  "project_start_to_heightmap": "WORLD_SURFACE_WG",
  "max_distance_from_center": 80,
  "use_expansion_hack": false,
  "liquid_settings": "ignore_waterlogging"
}
```

### 3.2. Набор размещения (`data/<ns>/worldgen/structure_set/ancient_shrine.json`):
```json
{
  "structures": [
    { "structure": "examplemod:ancient_shrine", "weight": 1 }
  ],
  "placement": {
    "type": "minecraft:random_spread",
    "spacing": 34,
    "separation": 12,
    "spread_type": "linear",
    "salt": 94827104,
    "frequency_reduction_method": "default",
    "locate_offset": [0, 0, 0]
  }
}
```
*Примечание:* Параметр `salt` обязан быть уникальным большим числом, отличным от ванильных и других модов, чтобы избежать совмещения координат структур.

### 3.3. Пул шаблонов (`data/<ns>/worldgen/template_pool/ancient_shrine.json`):
```json
{
  "fallback": "minecraft:empty",
  "elements": [
    {
      "weight": 1,
      "element": {
        "element_type": "minecraft:single_pool_element",
        "projection": "rigid",
        "location": "examplemod:ancient_shrine",
        "processors": "minecraft:empty"
      }
    }
  ]
}
```

---

## 4. Диагностика и проверка в игре

1. **/locate structure `<namespace>:<id>`**:
   - Находит координаты структуры даже в том случае, если сам NBT-шаблон сломан (команда проверяет только placement из `structure_set`).
2. **/place structure `<namespace>:<id> ~ ~ ~`**:
   - **Главный тест шаблона:** принудительно строит структуру прямо перед игроком. Если постройка не появляется — поврежден NBT или указана неверная папка.
3. **Генерация только в новых чанках:**
   - Уже исследованные чанки существующего мира **никогда не перегенерируются**. Для проверки естественной генерации создавайте новый тестовый мир или телепортируйтесь на неисследованные координаты (`/tp @s 10000 ~ 10000`).
