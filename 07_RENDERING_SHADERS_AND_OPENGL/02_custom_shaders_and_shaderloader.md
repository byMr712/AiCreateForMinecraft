# 02. Кастомные шейдеры, ShaderLoader и матрицы мира

## 1. Размещение шейдеров в Minecraft 1.21.4

> [!CAUTION]
> **СМЕНА КАТАЛОГА ШЕЙДЕРОВ В 1.21.4:**
> В 1.21.4 загрузчик шейдеров `ShaderLoader` (`ShaderProgramKey`) ищет программы напрямую по пути:
> **`assets/<namespace>/shaders/<program_name>.json`** (а также `.vsh` и `.fsh`).
> 
> Помещение шейдеров в старый каталог `shaders/core/` приводит к тому, что ванильный движок не находит шейдер при старте, сбрасывает все пакеты ресурсов и вылетает в главное меню с ошибкой `missing shader`!

Пример:
- Конфигурация: `assets/examplemod/shaders/crystal_glow.json`
- Вершинный шейдер: `assets/examplemod/shaders/crystal_glow.vsh`
- Фрагментный шейдер: `assets/examplemod/shaders/crystal_glow.fsh`

---

## 2. Порядок матриц в `WorldRenderer.render(...)` (Minecraft 1.21.4)

> [!CAUTION]
> **КРИТИЧЕСКАЯ ОШИБКА РАСЧЕТА МИРОВЫХ КООРДИНАТ И ЛУЧЕЙ:**
> Сигнатура метода рендера мира в 1.21.4:
> ```java
> public void render(
>     ObjectAllocator allocator,
>     RenderTickCounter tickCounter,
>     boolean tick,
>     Camera camera,
>     GameRenderer gameRenderer,
>     Matrix4f positionMatrix,    // 6-й параметр: Матрица вида (View Matrix)
>     Matrix4f projectionMatrix  // 7-й параметр: Матрица проекции (Projection Matrix)
> )
> ```
> Если в миксине перепутать аргументы местами, матрицы инвертируются. Это полностью ломает расчет мировых координат по буферу глубины (`unproject`), лучевой рендеринг (raymarching) и шейдерные пост-эффекты!

---

## 3. Замена устаревшего `BuiltinItemRendererRegistry`

В Fabric 1.21.4 класс `BuiltinItemRendererRegistry` удален.
Для кастомного 3D-рендеринга предметов в руке используется миксин в `ItemRenderer`:

```java
@Mixin(ItemRenderer.class)
public class ItemRendererMixin {
    @Inject(
        method = "renderItem(Lnet/minecraft/item/ItemStack;Lnet/minecraft/item/ModelTransformationMode;ZLnet/minecraft/client/util/math/MatrixStack;Lnet/minecraft/client/render/VertexConsumerProvider;IILnet/minecraft/client/render/model/BakedModel;)V",
        at = @At("HEAD"),
        cancellable = true
    )
    private void renderCustom3DItem(ItemStack stack, ModelTransformationMode renderMode, boolean leftHanded, 
                                    MatrixStack matrices, VertexConsumerProvider vertexConsumers, 
                                    int light, int overlay, BakedModel model, CallbackInfo ci) {
        // В GUI рендеринг идет через стандартные JSON-модели
        if (renderMode == ModelTransformationMode.GUI || renderMode == ModelTransformationMode.NONE) {
            return;
        }

        if (stack.isOf(ModItems.MAGIC_SCEPTER)) {
            // Применяем кастомный динамический рендеринг
            MagicScepterRenderer.render(stack, renderMode, matrices, vertexConsumers, light, overlay);
            ci.cancel(); // Отменяем ванильную отрисовку
        }
    }
}
```

---

## 4. Слой рендера текстурированных UI-элементов

> [!NOTE]
> В 1.21.4 `RenderLayer.getText(...)` строго требует вершинного формата текста (`POSITION_COLOR_TEXTURE_LIGHT`).
> Для отрисовки произвольных UI спрайтов и текстур интерфейса используйте:
> **`RenderLayer.getGuiTextured(Identifier texture)`** или встроенные методы `DrawContext.drawTexture(...)`.
