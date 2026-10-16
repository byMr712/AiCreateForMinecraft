# 02. Мобы и анимации GeckoLib (GeckoLib 4.x / 5.x)

## 1. Архитектура сущности GeckoLib (`GeoEntity`)

Сущность реализует интерфейс `GeoEntity` и управляет кэшем анимаций:

```java
import software.bernie.geckolib.animatable.GeoEntity;
import software.bernie.geckolib.animatable.instance.AnimatableInstanceCache;
import software.bernie.geckolib.animation.*;
import software.bernie.geckolib.util.GeckoLibUtil;

public class PenguinEntity extends AnimalEntity implements GeoEntity {
    private final AnimatableInstanceCache cache = GeckoLibUtil.createInstanceCache(this);

    public PenguinEntity(EntityType<? extends AnimalEntity> type, World world) {
        super(type, world);
    }

    @Override
    public void registerControllers(AnimatableManager.ControllerRegistrar controllers) {
        // 1. Контроллер передвижения (базовые зацикленные анимации)
        controllers.add(new AnimationController<>(this, "move", 5, event -> {
            if (this.isSubmergedInWater()) {
                return event.setAndContinue(RawAnimation.begin().thenLoop("animation.penguin.swim"));
            }
            if (event.isMoving()) {
                return event.setAndContinue(RawAnimation.begin().thenLoop("animation.penguin.walk"));
            }
            return event.setAndContinue(RawAnimation.begin().thenLoop("animation.penguin.idle"));
        }));

        // 2. Контроллер триггерных действий (взмах крыльями, чистка перьев, клевание)
        controllers.add(new AnimationController<>(this, "action", 2, event -> PlayState.STOP)
            .triggerableAnim("preen", RawAnimation.begin().thenPlay("animation.penguin.preen"))
            .triggerableAnim("shake", RawAnimation.begin().thenPlay("animation.penguin.shake"))
        );
    }

    // Запуск триггерного действия с сервера
    public void triggerShake() {
        this.triggerAnim("action", "shake");
    }

    @Override
    public AnimatableInstanceCache getAnimatableInstanceCache() {
        return this.cache;
    }
}
```

---

## 2. Контракт именования костей и анимаций в Blockbench

> [!IMPORTANT]
> **API между Blockbench и кодом Java:**
> 1. **Имена анимаций:** Всегда строгий паттерн: `animation.<mob_id>.<action>` (например, `animation.penguin.walk`, `animation.penguin.swim`).
> 2. **Имена костей:**
>    - `head`: голова (для плавного поворота за игроком).
>    - `eyes` / `eyes_blink`: кость глаз для моргания.
>    - `body`: туловище.
>    - `left_wing` / `right_wing` или `left_arm` / `right_arm`.
>    - `left_leg` / `right_leg`.
> 3. Любой элемент, который код Java может скрывать или переключать (закрытые глаза, предмет в клюве, седло), **обязан быть отдельной костью**!

---

## 3. Трюки в рендерере GeckoLib (`GeoEntityRenderer`)

```java
public class PenguinRenderer extends GeoEntityRenderer<PenguinEntity> {
    public PenguinRenderer(EntityRendererFactory.Context context) {
        super(context, new PenguinModel());
    }

    @Override
    public void adjustModelBonesForRender(PenguinEntity entity, GeoModel<PenguinEntity> model, 
                                          MatrixStack matrices, VertexConsumerProvider bufferSource, 
                                          VertexConsumer buffer, float partialTick, int packedLight, int packedOverlay) {
        super.adjustModelBonesForRender(entity, model, matrices, bufferSource, buffer, partialTick, packedLight, packedOverlay);

        GeoBone head = model.getBone("head").orElse(null);
        if (head != null) {
            // Плавный поворот головы (не используйте hardcodedHeadRotation, он сбрасывает анимации!)
            head.setRotX(head.getRotX() + (entity.getPitch(partialTick) * ((float) Math.PI / 180F)));
            head.setRotY(head.getRotY() + ((entity.getYaw(partialTick) - entity.bodyYaw) * ((float) Math.PI / 180F)));
        }

        // Эффект моргания: сплющивание кости глаз по Y в 0 на 2 тика раз в 4-6 секунд
        GeoBone eyes = model.getBone("eyes").orElse(null);
        if (eyes != null) {
            long time = entity.getId() * 31L + entity.age;
            boolean blinking = (time % 80) < 3;
            eyes.setScaleY(blinking ? 0.0f : 1.0f);
        }

        // Масштабирование детеныша: уменьшенное тело, увеличенная голова
        if (entity.isBaby()) {
            matrices.scale(0.5f, 0.5f, 0.5f);
            if (head != null) {
                head.setScaleX(1.3f);
                head.setScaleY(1.3f);
                head.setScaleZ(1.3f);
            }
        }
    }
}
```

---

## 4. Размещение файлов моделей и анимаций

В GeckoLib 4.8.5+ пути к ресурсам обязаны быть строго согласованы:
- Геометрия: `assets/<namespace>/geo/entity/<id>.geo.json` (и дубликат в `geo/<id>.geo.json`).
- Анимации: `assets/<namespace>/animations/entity/<id>.animation.json` (и `animations/<id>.animation.json`).
- Текстура: `assets/<namespace>/textures/entity/<id>.png`.
