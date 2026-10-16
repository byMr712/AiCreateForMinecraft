# 01. Современный пайплайн рендеринга (EntityRenderState)

## 1. Революция EntityRenderState в Minecraft 1.21.4

В Minecraft 1.21.4 Mojang полностью разделили игровую симуляцию и рендеринг сущностей:
- Рендереры `EntityRenderer` и модели `EntityModel` **больше не принимают** экземпляр `LivingEntity` во время отрисовки!
- Вместо этого отрисовка происходит через легковесные структуры состояния кадра: **`EntityRenderState`** (например, `LivingEntityRenderState`, `BipedEntityRenderState`).

### 1.1. Проброс кастомных данных через Duck Typing:
Если вашей модели нужны кастомные данные (например, особый таймер натяжения лука, тип копья, заряд магии), создается интерфейс-аксессор:

```java
// Интерфейс расширения состояния кадра
public interface CustomWeaponRenderState {
    boolean examplemod$isSpearEquipped();
    void examplemod$setSpearEquipped(boolean value);
}
```

Инъекция в `BipedEntityRenderState` через Mixin:
```java
@Mixin(BipedEntityRenderState.class)
public class BipedEntityRenderStateMixin implements CustomWeaponRenderState {
    @Unique private boolean isSpear;

    @Override
    public boolean examplemod$isSpearEquipped() {
        return this.isSpear;
    }

    @Override
    public void examplemod$setSpearEquipped(boolean value) {
        this.isSpear = value;
    }
}
```

Заполнение состояния в `PlayerEntityRenderer.updateRenderState`:
```java
@Mixin(PlayerEntityRenderer.class)
public class PlayerEntityRendererMixin {
    @Inject(method = "updateRenderState", at = @At("TAIL"))
    private void updateCustomState(AbstractClientPlayerEntity player, PlayerEntityRenderState state, float delta, CallbackInfo ci) {
        if (state instanceof CustomWeaponRenderState accessor) {
            accessor.examplemod$setSpearEquipped(player.getMainHandStack().isOf(ModItems.COPPER_SPEAR));
        }
    }
}
```

---

## 2. Строгая валидация формата вершин `VertexConsumer`

> [!CAUTION]
> **КРАШ ПРИ РЕНДЕРЕ СУЩНОСТЕЙ (`NEW_ENTITY`):**
> При построении полигона сущности в формате `DefaultVertexFormat.NEW_ENTITY` (`POSITION_COLOR_TEX_OVERLAY_LIGHT_NORMAL`) конвейер Minecraft 1.21.4 строго валидирует вызовы.
> Если пропустить хотя бы один атрибут, игра крашится с фатальной ошибкой:
> `java.lang.IllegalStateException: Missing elements in vertex: UV1, UV2, Normal`

### Корректная цепочка вызовов:
```java
buffer.addVertex(matrixEntry, x, y, z)
    .setColor(red, green, blue, alpha)
    .setUv(u, v)
    .setOverlay(OverlayTexture.NO_OVERLAY) // UV1
    .setLight(packedLight)                 // UV2
    .setNormal(matrixEntry, nx, ny, nz);  // Normal
```
Все 6 методов должны вызываться в строгой последовательности для каждой вершины полигона.
