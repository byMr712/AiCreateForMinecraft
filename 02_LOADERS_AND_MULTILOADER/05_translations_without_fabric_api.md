# 05. Локализация мода без Fabric API и переводы клавиш

## 1. Проблема: сырые ключи в управлении

Если мод не зависит от Fabric API, файлы `assets/<modid>/lang/*.json` **не монтируются** в ванильный ресурс-менеджер автоматически. Собственный загрузчик переводов (например, `T`) может отлично работать в своих экранах, но ванильный экран управления показывает:

```
key.category.mymod.main
key.mymod.open_map
key.mymod.editor
```

Причина: `KeyBindsList` строит подпись через `Component.translatable(keyMapping.getName())`, а `Language.getInstance()` не знает ключей мода.

Fabric API решает это через `fabric-resource-loader`: он добавляет ресурсы мода как встроенный `ModPackResources` в `ResourceManager`. Без Fabric API есть два пути:

1. Реализовать собственный resource pack — сложно и избыточно.
2. Добавить переводы прямо в ванильный `Language` после каждой загрузки ресурсов — быстро и надёжно.

---

## 2. Обёртка `Language` (рекомендуемый способ)

Создаём `Language`, который делегирует всё ванильному языку, но сначала ищет ключ в карте мода.

```java
public final class ModLanguage extends Language {
    private final Language delegate;
    private Map<String, String> extra;

    public ModLanguage(Language delegate, Map<String, String> extra) {
        this.delegate = delegate;
        this.extra = extra;
    }

    public void setExtra(Map<String, String> extra) {
        this.extra = extra;
    }

    @Override
    public String getOrDefault(String key, String fallback) {
        String value = extra.get(key);
        return value != null ? value : delegate.getOrDefault(key, fallback);
    }

    @Override
    public boolean has(String key) {
        return extra.containsKey(key) || delegate.has(key);
    }

    @Override
    public boolean isDefaultRightToLeft() {
        return delegate.isDefaultRightToLeft();
    }

    @Override
    public FormattedCharSequence getVisualOrder(FormattedText text) {
        return delegate.getVisualOrder(text);
    }
}
```

Затем подменяем глобальный язык после перезагрузки ресурсов:

```java
@Mixin(LanguageManager.class)
public class LanguageManagerMixin {

    @Inject(method = "onResourceManagerReload", at = @At("TAIL"))
    private void addTranslations(ResourceManager manager, CallbackInfo ci) {
        Language current = Language.getInstance();
        String selected = ((LanguageManager) (Object) this).getSelected();
        Map<String, String> extra = MyTranslations.forLanguage(selected);

        if (current instanceof ModLanguage mod) {
            mod.setExtra(extra); // смена языка без вложенных обёрток
        } else {
            Language.inject(new ModLanguage(current, extra));
        }
    }
}
```

Обязательно добавляем миксин в **клиентский** массив:

```json
"client": [
  "LanguageManagerMixin"
]
```

---

## 3. Ключи перевода

В `en_us.json` и `ru_ru.json` должны быть не только тексты мода, но и ключи клавиш:

```json
{
  "key.category.mymod.main": "My Mod",
  "key.mymod.open_map": "Open map",
  "key.mymod.editor": "Drawing editor"
}
```

Названия клавиш создаются так:

```java
new KeyMapping("key.mymod.open_map", InputConstants.KEY_M, CATEGORY);
```

Категория:

```java
KeyMapping.Category.register(Identifier.fromNamespaceAndPath("mymod", "main"));
// подпись: key.category.mymod.main
```

---

## 4. Ловушки

> [!CAUTION]
> **Не мутируйте `ClientLanguage.storage` напрямую.**
> При наличии resource-loader'ов и других модов карта может быть неизменяемой. Попытка `storage.putAll(...)` даёт:
> `java.lang.UnsupportedOperationException` и ломает загрузку ресурсов.

> [!WARNING]
> **Не заворачивайте `ModLanguage` в `ModLanguage` повторно.**
> При каждом `onResourceManagerReload` проверяйте `instanceof ModLanguage` и обновляйте существующую карту через `setExtra`.

> [!IMPORTANT]
> **Делегируйте все абстрактные методы `Language`.**
> Иначе сломаются RTL, форматирование и `getVisualOrder`, а текст в интерфейсе начнёт отображаться некорректно.

> [!NOTE]
> Обёртка должна стоять в `"client"` массиве `mixins.json`: `LanguageManager` — клиентский класс.

---

## 5. Проверка

После перезагрузки ресурсов и смены языка:

```java
String name = I18n.get("key.mymod.open_map");
if (name.startsWith("key.")) {
    throw new IllegalStateException("Key binding translation is missing");
}
```

В экране **Настройки → Управление → Клавиши** категория и все клавиши должны отображаться нормальным текстом на выбранном языке.

---

## 6. Бэкпорт на несколько версий

Файлы `ModLanguage`, `LanguageManagerMixin` и метод `clientTranslations` не зависят от версии Minecraft в диапазоне 1.21–26.x. При переносе на другую ветку обычно конфликтует только `atlasmapextended.client.mixins.json`: в старых ветках HUD-миксин может называться `GuiMixin`, а в новых — `HudMixin`.

Правильное разрешение конфликта — **сохранить веточный HUD-миксин** и просто добавить рядом:

```json
"GuiMixin",
"LanguageManagerMixin",
```

Не заменяйте `GuiMixin` на `HudMixin` из другой ветки.
