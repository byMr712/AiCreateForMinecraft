# 02. Память, кэширование и многопоточность

## 1. Потокобезопасные структуры данных и быстрые кэши $O(1)$

В модах, активно изменяющих конфигурацию или обрабатывающих рецепты в фоновых потоках:

```java
// 1. Потокобезопасная коллекция рецептов
private final Map<String, CustomRecipeData> recipes = new ConcurrentHashMap<>();

// 2. Быстрый кэш O(1) для мгновенной проверки наличия модификаций
private final Set<String> enabledResultIds = ConcurrentHashMap.newKeySet();

public boolean hasCustomRecipe(String itemId) {
    return this.enabledResultIds.contains(itemId); // Мгновенный ответ без итерации
}
```

---

## 2. Монотонные счетчики версий (`configVersion++`)

> [!TIP]
> Вместо постоянного ресурсоемкого вычисления `hashCode()` для сотен объектов конфигурации, используйте монотонный атомарный инкремент версии:
```java
private volatile long configVersion = 0;

public synchronized void addOrUpdateRecipe(CustomRecipeData recipe) {
    this.recipes.put(recipe.getKey(), recipe);
    this.rebuildCache();
    this.configVersion++; // Сигнализирует всем миксинам и GUI об обновлении
}
```
Миксины и внешние оверлеи просто сравнивают `if (this.lastSeenVersion != config.getVersion())`, обновляя данные только при реальных изменениях.

---

## 3. Атомарное сохранение файлов конфигурации (Crash-Proof Saving)

Чтобы файл конфигурации не повредился в случае внезапного краша игры или отключения питания во время записи на диск:

```java
public void saveAtomic(Path configPath, String jsonContent) throws IOException {
    Path tempPath = configPath.resolveSibling(configPath.getFileName() + ".tmp");

    // 1. Полная запись во временный файл
    Files.writeString(tempPath, jsonContent, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING);

    // 2. Атомарное перемещение
    try {
        Files.move(tempPath, configPath, StandardCopyOption.ATOMIC_MOVE, StandardCopyOption.REPLACE_EXISTING);
    } catch (AtomicMoveNotSupportedException e) {
        // Fallback для файловых систем без поддержки ATOMIC_MOVE
        Files.move(tempPath, configPath, StandardCopyOption.REPLACE_EXISTING);
    }
}
```

---

## 4. Workstation Guard и Debounce для рецептурных интеграций (REI / JEI / EMI)

1. **Workstation Guard (Защита от фризов инвентаря):**
   - Перезагрузка кэша рецептов сторонних модов (REI, JEI) во время того, как у игрока открыт сундук, печь или верстак (`client.currentScreen instanceof HandledScreen`), вызывает подвисание GUI.
   - **Решение:** Откладывать перезагрузку до момента, пока игрок не закроет контейнер.
2. **Debounce (350 ms):**
   - Серия быстрых сохранений (например, перетаскивание предметов в редакторе) объединяется таймером задержки в один финальный вызов перезагрузки кэша.
