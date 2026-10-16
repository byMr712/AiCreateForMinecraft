# 04. Графический интерфейс, экраны (GUI) и модель ввода

## 1. Архитектура экранов: Screen и HandledScreen

- **`Screen`**: Автономные меню без синхронизации серверного инвентаря (настройки, энциклопедии, редакторы).
- **`HandledScreen<T extends ScreenHandler>` (Fabric) / `AbstractContainerScreen<T extends AbstractContainerMenu>` (NeoForge)**: Контейнеры с привязкой к слотам и серверной синхронизацией предметов.

### Базовый экран Fabric:
```java
public class MyConfigScreen extends Screen {
    private final Screen parent;

    public MyConfigScreen(Screen parent) {
        super(Text.translatable("gui.examplemod.config.title"));
        this.parent = parent;
    }

    @Override
    protected void init() {
        this.addDrawableChild(ButtonWidget.builder(ScreenTexts.DONE, button -> {
            this.client.setScreen(this.parent);
        }).dimensions(this.width / 2 - 100, this.height - 30, 200, 20).build());
    }

    @Override
    public void render(DrawContext context, int mouseX, int mouseY, float delta) {
        super.render(context, mouseX, mouseY, delta);
        context.drawCenteredTextWithShadow(this.textRenderer, this.title, this.width / 2, 20, 0xFFFFFF);
    }
}
```

---

## 2. Разница моделей ввода: GLFW (1.20 - 1.21.x) против SDL3 (26.x)

> [!CAUTION]
> **КРИТИЧЕСКАЯ ОШИБКА ОБРАБОТКИ КЛИКОВ МЫШИ В 26.x:**
> В версиях Minecraft до 1.21.x (GLFW) индексы кнопок мыши начинались с `0`:
> - `0` = Левая кнопка (LMB)
> - `1` = Правая кнопка (RMB)
> - `2` = Средняя кнопка (MMB)
> 
> В Minecraft 26.x Mojang перешли на библиотеку **SDL3**, где индексы 1-based:
> - `1` = Левая кнопка (`InputConstants.MOUSE_BUTTON_LEFT`)
> - `2` = Средняя кнопка (`InputConstants.MOUSE_BUTTON_MIDDLE`)
> - `3` = Правая кнопка (`InputConstants.MOUSE_BUTTON_RIGHT`)
> 
> Если в 26.x проверить `button == 1` для правой кнопки, то левый клик начнет триггерить действие правого клика, а настоящий правый клик (кнопка 3) будет полностью проигнорирован!

### Универсальные помощники:
```java
public static boolean isLeftClick(int button) {
    // В 1.21: button == 0; в 26.x: button == 1
    return button == InputConstants.MOUSE_BUTTON_LEFT;
}

public static boolean isRightClick(int button) {
    // В 1.21: button == 1; в 26.x: button == 3
    return button == InputConstants.MOUSE_BUTTON_RIGHT;
}
```

---

## 3. Адаптивное масштабирование интерфейса (`uiScale`)

При разработке комплексных студийных интерфейсов (например, 460x380 px) координаты курсора мыши масштабируются относительно динамического разрешения окна:

```java
float uiScale = Math.min((float) width / 460f, (float) height / 380f);
int sX = (int) (mouseX / uiScale);
int sY = (int) (mouseY / uiScale);

// В методе render():
context.getMatrices().push();
context.getMatrices().scale(uiScale, uiScale, 1.0f);
// Отрисовка элементов в виртуальных координатах 460x380
...
context.getMatrices().pop();
```

---

## 4. Подключение к меню конфигурации

### Fabric (через ModMenu API):
```java
public class ModMenuIntegration implements ModMenuApi {
    @Override
    public ConfigScreenFactory<?> getModConfigScreenFactory() {
        return MyConfigScreen::new;
    }
}
```

### NeoForge (через `IConfigScreenFactory`):
```java
modContainer.registerExtensionPoint(
    IConfigScreenFactory.class,
    (mc, parent) -> new MyConfigScreen(parent)
);
```
Благодаря этой регистрации в интерфейсе списка модов NeoForge становится активной кнопка **«Config»**.
