# 01. Словарь частых крашей и фатальных исключений

## 1. `NullPointerException: Item id not set`
- **Лог:** `java.lang.NullPointerException: Item id not set at java.base/java.util.Objects.requireNonNull`
- **Причина:** В 1.21.4 конструктор `Item(Item.Settings)` проверяет наличие `registryKey`.
- **Решение:** Добавить `.registryKey(key)` в `new Item.Settings().registryKey(...)`.

---

## 2. `IllegalStateException: No display name set for ItemGroup`
- **Лог:** `java.lang.IllegalStateException: No display name set for ItemGroup at knot//net.fabricmc.fabric.impl.itemgroup.FabricItemGroupBuilderImpl.build`
- **Причина:** В Fabric Item Group API v1 не вызван `.displayName(...)` при создании вкладки.
- **Решение:** Добавить `.displayName(Text.translatable("itemGroup.<modid>.<name>"))`.

---

## 3. `IllegalStateException: Missing elements in vertex: UV1, UV2, Normal`
- **Лог:** `java.lang.IllegalStateException: Missing elements in vertex: UV1, UV2, Normal`
- **Причина:** При построении вершин в формате `NEW_ENTITY` пропущен один из обязательных атрибутов конвейера.
- **Решение:** Вызывать полную цепочку методов: `.addVertex() -> .setColor() -> .setUv() -> .setOverlay() -> .setLight() -> .setNormal()`.

---

## 4. `Input does not contain a key [fabric:type]: MapLike[...]`
- **Лог:** `com.mojang.serialization.DataResult$Error: Input does not contain a key [fabric:type]: MapLike[...]`
- **Причина:** В файле рецепта использован устаревший синтаксис объекта `{"item": "..."}` вместо нового синтаксиса 1.21.4.
- **Решение:** Заменить объект на прямую строку ID: `"minecraft:iron_ingot"` или `"#minecraft:planks"`.

---

## 5. `java.lang.module.ResolutionException: Modules export package net.fabricmc`
- **Лог:** `java.lang.module.ResolutionException: Modules ... and ... export package net.fabricmc to the unnamed module`
- **Причина:** В JAR мода для NeoForge попали стабы классов из пакета `net.fabricmc.*`, конфликтующие с JPMS.
- **Решение:** Исключить стабы `exclude "net/fabricmc/**"` из таска `jarNeoForge` в `build.gradle`.

---

## 6. `UnsupportedClassVersionError: class file version 66.0`
- **Лог:** `java.lang.UnsupportedClassVersionError: ... has been compiled by a more recent version of the Java Runtime (class file version 66.0)`
- **Причина:** Байткод мода скомпилирован под Java 22+ (66.0) вместо Java 21 (65.0) для 1.21.4.
- **Решение:** Установить `targetCompatibility = 21` и `options.release = 21` в `JavaCompile`.

---

## 7. `Couldn't load structure ... GZIPInputStream.readHeader`
- **Лог:** `Couldn't load structure <ns>:<id> java.util.zip.ZipException: Not in GZIP format`
- **Причина:** Шаблон структуры NBT сохранен в несжатом формате (начинается с байта `0A`).
- **Решение:** Сжать файл NBT в GZIP (первые два байта обязаны быть `1F 8B`).

---

## 8. `Invalid entity data item type`
- **Лог:** `java.lang.IllegalStateException: Invalid entity data item type for ...`
- **Причина:** Рассинхронизация индексов в `SynchedEntityData` между сервером и клиентом.
- **Решение:** Использовать защитный миксин в `DataTracker` / `SynchedEntityData` с валидацией типа сериализатора.

---

## 9. `DataResult.Error['No key pack_format in MapLike[...]']`
- **Лог:** `com.mojang.serialization.DataResult$Error: No key pack_format in MapLike[...]`
- **Причина:** В файле `pack.mcmeta` отсутствует или неверно названо поле `pack_format`.
- **Решение:** Указать `"pack_format": 61` (датапак 1.21.4) или `46` (ресурспак 1.21.4).

---

## 10. `JsonSyntaxException: Not a boolean: "false"`
- **Лог:** `com.google.gson.JsonSyntaxException: Not a boolean: "false"`
- **Причина:** Булево поле в JSON компонента предмета записано в кавычках как строка.
- **Решение:** Записать без кавычек: `"can_always_eat": false`.

---

## 11. `SignatureException: Bad signature length: 0` / `Invalid signature length`
- **Лог:** `java.security.SignatureException: Bad signature length: 0` при входе на сервер или обновлении скина в `com.mojang.authlib.yggdrasil.YggdrasilServicesKeyInfo.validateProperty`.
- **Причина:** Сервер (оффлайн-режим, BungeeCord/Velocity, пиратский прокси или плагин кастомных скинов) отправляет свойство профиля с пустой строкой в качестве подписи (`"signature": ""`). Метод `Property.hasSignature()` проверяет только `signature != null` и возвращает `true`, после чего валидатор пытается проверить пустой массив байт через RSA-подпись, выбрасывая фатальный `SignatureException` и разрывая соединение.
- **Решение:** 
  1. Переопределить `Property.hasSignature()` через `@Overwrite`: возвращать `signature != null && !signature.isEmpty()`.
  2. Внедрить защитный `@Inject` в `HEAD` метода `validateProperty`: если `property.signature() == null || property.signature().isEmpty()`, сразу возвращать `false` (`cir.setReturnValue(false)`). Для Authlib < 10 миксин вешается на `com.mojang.authlib.yggdrasil.YggdrasilServicesKeyInfo`, а для Authlib >= 10 (Minecraft 26.3+) — на `com.mojang.authlib.services.MinecraftServicesKeyInfo`.

