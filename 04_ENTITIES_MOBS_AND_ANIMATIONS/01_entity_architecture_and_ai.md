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

## 4. Серверные боты (Fake Players) и изоляция подключения посреди матча

При реализации серверных ботов (Fake Players) на базе нативного NMS (`ServerPlayer`) без внешних мостов или клиентских ботов возникают специфические требования к жизненному циклу сущности.

### 4.1. Сетевые заглушки и динамический респавн
- **Сетевой пайплайн:** `ServerPlayer` создаётся с фиктивным сетевым соединением (`NoopConnection` / `FakeConnection`), чтобы отбрасывать исходящие пакеты и симулировать подключение в `PlayerList`.
- **Динамический геттер вместо кеширования:** В Spigot/Paper при респавне экземпляр `ServerPlayer` пересоздаётся. Ссылки на сущность нельзя кешировать в полях бота; геттеры `getBukkitPlayer()` и `getNmsPlayer()` обязаны запрашивать актуальный объект через `Bukkit.getPlayer(uuid)` и `PlayerList.getPlayer(uuid)` соответственно.
- **Очистка памяти (Zero Memory Leak):** При удалении бота необходимо разорвать GC-корни: отменить все таймеры AI, очистить данные игрока во всех менеджерах, вызвать `nmsPlayer.unRide()`, `nmsPlayer.closeContainer()`, `nmsPlayer.discard()`, `level.removePlayerImmediately(nmsPlayer, DISCARDED)` и отключить фиктивное соединение.

### 4.2. Изоляция подключения посреди матча (Mid-Game Join Isolation)
Когда новый бот (или живой игрок) подключается во время активного боя:
1. **Фаза активного раунда (PLAYING / COUNTDOWN):**
   - Бот не должен сразу появляться на боевой арене.
   - Сущность телепортируется в зону ожидания (лобби), переводится в режим `ADVENTURE` и наделяется неуязвимостью.
   - Игрок регистрируется в реестре ожидания следующего раунда (`pendingCardJoiner`) и отмечается как выбывший в текущем раунде.
   - Искусственный интеллект (`BotAI`) отключает поиск целей, стрельбу и маневры для сущностей с флагом `pendingCardJoiner`.
   - Живые участники боя и другие боты игнорируют ожидающих при поиске целей (`findVisibleTarget`).
2. **Критическая ошибка подсчёта выживших (`checkRoundEnd`):**
   - Если проверка окончания раунда проверяет только статус `isOnline()` или наличие в команде, ожидающий в лобби бот будет ошибочно засчитан как живой участник раунда.
   - В результате раунд никогда не сможет завершиться после победы над всеми противниками на арене.
   - **Правило:** Метод проверки условий победы в раунде обязан явно исключать ожидающих игроков:
     ```java
     if (isPendingCardJoiner(uuid)) {
         continue; // Ожидающий в лобби не участвует в текущем раунде
     }
     ```
3. **Фаза драфта карт (CARDS):**
   - Если бот добавлен во время выбора карт, он сразу телепортируется на точку спавна арены и получает интерфейс выбора карты для предстоящего раунда.
   - При переходе из активного боя в фазу драфта все ожидавшие в лобби боты автоматически переводятся в активный состав матча.
