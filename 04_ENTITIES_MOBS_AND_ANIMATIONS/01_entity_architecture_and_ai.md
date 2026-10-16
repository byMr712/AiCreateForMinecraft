# 01. Архитектура сущностей, искусственный интеллект (AI) и физика

## 1. Регистрация EntityType и базовый класс

```java
public class ModEntities {
    public static final EntityType<PenguinEntity> PENGUIN = Registry.register(
        Registries.ENTITY_TYPE,
        Identifier.of("examplemod", "penguin"),
        EntityType.Builder.create(PenguinEntity::new, SpawnGroup.CREATURE)
            .dimensions(0.6f, 0.9f)
            .eyeHeight(0.7f)
            .maxTrackingRange(10)
            .build(RegistryKey.of(RegistryKeys.ENTITY_TYPE, Identifier.of("examplemod", "penguin")))
    );

    public static void registerAttributes() {
        FabricDefaultAttributeRegistry.register(
            PENGUIN,
            AnimalEntity.createAnimalAttributes()
                .add(EntityAttributes.MAX_HEALTH, 10.0)
                .add(EntityAttributes.MOVEMENT_SPEED, 0.20)
                .add(EntityAttributes.WATER_MOVEMENT_EFFICIENCY, 0.8)
                .add(EntityAttributes.STEP_HEIGHT, 1.0) // Важно для выхода из воды!
        );
    }
}
```

---

## 2. Искусственный интеллект и системы целей (AI Goals)

### 2.1. Управление целями (`initGoals`):
```java
@Override
protected void initGoals() {
    this.goalSelector.add(0, new SwimGoal(this));
    this.goalSelector.add(1, new EscapeDangerGoal(this, 1.4));
    this.goalSelector.add(2, new AnimalMateGoal(this, 1.0));
    this.goalSelector.add(3, new TemptGoal(this, 1.1, stack -> stack.isOf(Items.COD), false));
    this.goalSelector.add(4, new FollowParentGoal(this, 1.1));
    this.goalSelector.add(5, new WanderAroundFarGoal(this, 1.0));
    this.goalSelector.add(6, new LookAtEntityGoal(this, PlayerEntity.class, 6.0f));
    this.goalSelector.add(7, new LookAroundGoal(this));
}
```

### 2.2. Критические уроки настройки AI мобов:
1. **Цели одинакового приоритета блокируют друг друга:**
   - Если две цели имеют одинаковый приоритет, они могут бесконечно соперничать за управление мобом, вызывая фризы или застревание.
   - Механики, которые должны работать всегда (например, наложение ауры, выдача эффекта, частицы), помещаются напрямую в `tick()` или `customServerAiStep()`, а **не в цель (Goal)**!
2. **Проблема плавного плавания (`SmoothSwimmingMoveControl`):**
   - Наземные/водные мобы с кастомным контролем плавания перестают прыгать на суше. Решение: установите базовый атрибут `STEP_HEIGHT = 1.0` (как у черепах и аксолотлей).
3. **Безопасный поиск пути:**
   - Всегда проверяйте доступность целевой точки: `path.canReach()`.
   - Если моб не продвинулся к цели за 5 секунд — сбрасывайте путь (`navigation.stop()`) во избежание циклического упора в стену.
4. **Стайное поведение (Flocking):**
   - Вожак стаи выбирается детерминированно: сущность с наименьшим `getId()` среди ближайших сородичей.
   - Последователи держат дистанцию 2–4 блока позади вожака.
5. **Коллизия с лодками и рассинхронизация ("Moved wrongly"):**
   - Сущности, находящиеся близко к лодкам с игроками, могут вызывать рассинхронизацию сервера и откат («rubber-banding»). Если моб плывет рядом с управляемой лодкой, делайте его временно непроницаемым (`isPushable() -> false`).

---

## 3. Сигнатуры нанесения урона в Minecraft 1.21.4+

> [!CAUTION]
> В 1.21.4 методы `damage` и `tryAttack` требуют обязательной явной передачи `ServerWorld` в качестве первого аргумента:
> ```java
> if (world instanceof ServerWorld serverWorld) {
>     target.damage(serverWorld, this.getDamageSources().mobAttack(this), damageAmount);
>     this.tryAttack(serverWorld, target);
> }
> ```
> Попытка вызвать устаревший `target.damage(damageSource, amount)` приведет к ошибке компиляции.
