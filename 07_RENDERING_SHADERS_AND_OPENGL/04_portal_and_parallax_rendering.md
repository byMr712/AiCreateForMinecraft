# 04. Рендеринг порталов, эффект параллакса и глубина

## 1. Проблема просвечивания облаков и совместимость с Iris/Sodium

При создании полупрозрачных порталов (в стиле портала в Нижний мир) или небесных окон со звездным небом (в стиле портала в Край):
1. **Шейдеры Iris / Oculus:** Стандартный `RENDERTYPE_END_PORTAL_SHADER` перехватывается шейдерпаками и принудительно заменяется на ванильный скайбокс Края.
2. **Просвечивание облаков и частиц:** Если портал рисуется на полупрозрачном слое без записи в буфер глубины (Z-buffer), то облака, дождь и частицы, которые рендерятся игрой позже, рисуются **поверх портала**, портя погружение.

---

## 2. Архитектура кастомного слоя рендера с маской глубины

Для порталов создается кастомный `RenderType` с прозрачностью `TRANSLUCENT_TRANSPARENCY` и маской глубины **`COLOR_DEPTH_WRITE`**:

```java
public class ModRenderTypes {
    public static final RenderType CUSTOM_PORTAL = RenderType.create(
        "custom_portal",
        DefaultVertexFormat.POSITION_COLOR_TEX_LIGHTMAP,
        VertexFormat.DrawMode.QUADS,
        256,
        false,
        true,
        RenderType.CompositeState.builder()
            .setShader(RenderPhase.POSITION_COLOR_TEX_LIGHTMAP_SHADER)
            .setTexture(new RenderPhase.TextureStateShard(Identifier.of("examplemod", "textures/block/portal.png"), TriState.FALSE, false))
            .setTransparency(RenderPhase.TRANSLUCENT_TRANSPARENCY)
            .setDepthTest(RenderPhase.LEQUAL_DEPTH_TEST)
            .setWriteMaskState(RenderPhase.COLOR_DEPTH_WRITE) // Запись в Z-буфер!
            .createCompositeState(false)
    );
}
```

> [!IMPORTANT]
> Запись в Z-буфер (`COLOR_DEPTH_WRITE`) блокирует наложение облаков и дождя прямо поверх портала, сохраняя при этом видимость ландшафта за порталом.

---

## 3. Бесшовный маппинг текстур по мировым координатам (World-Space UV)

Чтобы портал из множества смежных блоков выглядел как единое окно в другое измерение, UV-координаты вычисляются на основе абсолютных мировых координат:

```java
public static void renderPortalFace(VertexConsumer buffer, MatrixStack.Entry entry, 
                                    BlockPos pos, Direction side, float x1, float y1, float z1, ...) {
    // Вычисление UV относительно глобальных координат мира
    float u1 = (pos.getX() + x1) * 0.125f;
    float v1 = (pos.getY() + y1) * 0.125f;
    // ...
}
```
Это исключает швы между блоками портала независимо от его размера и угла обзора игрока.
