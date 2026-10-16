# 02. Мастерство инъекций байткода (Sponge Mixins)

## 1. Конфигурация `mixins.json` и ловушка секции `"server"`

> [!CAUTION]
> **СМЕРТЕЛЬНАЯ ЛОВУШКА СЕКЦИИ `"server": [...]`:**
> В конфигурационном файле `*.mixins.json` массив `"server": [...]` активируется **ИСКЛЮЧИТЕЛЬНО** на выделенном сервере (Dedicated Server)!
> В одиночной игре (Singleplayer / Integrated Server) среда выполнения является клиентской (`client`), и все миксины из массива `"server"` **ПОЛНОСТЬЮ ИГНОРИРУЮТСЯ**.
> 
> **Строгое правило:**
> - Если миксин модифицирует серверную логику (`ServerPlayerEntity`, `ServerWorld`, `BedBlock`, рецепты, выпадение лута), он **ОБЯЗАН** находиться в общем корневом массиве `"mixins": [...]`!
> - Массив `"client": [...]` используется только для клиентских классов (`MinecraftClient`, `DrawContext`, `InGameHud`).

```json
{
  "required": true,
  "minVersion": "0.8",
  "package": "com.example.mod.mixin",
  "compatibilityLevel": "JAVA_21",
  "mixins": [
    "ServerWorldMixin",
    "PlayerInventoryMixin"
  ],
  "client": [
    "InGameHudMixin",
    "ItemRendererMixin"
  ],
  "injectors": {
    "defaultRequire": 1
  }
}
```

---

## 2. Основные типы инъекций

### 2.1. `@Inject` с перехватом входа или выхода:
```java
@Mixin(ServerPlayerEntity.class)
public abstract class ServerPlayerEntityMixin {
    @Inject(method = "tick", at = @At("HEAD"))
    private void onPlayerTick(CallbackInfo ci) {
        ServerPlayerEntity player = (ServerPlayerEntity) (Object) this;
        // Логика на каждом тике игрока
    }

    @Inject(method = "damage", at = @At("HEAD"), cancellable = true)
    private void onDamage(ServerWorld world, DamageSource source, float amount, CallbackInfoReturnable<Boolean> cir) {
        if (source.isOf(DamageTypes.FALL)) {
            cir.setReturnValue(false); // Полная отмена урона от падения
        }
    }
}
```

### 2.2. Замена вызова через `@WrapOperation` (MixinExtras):
`@WrapOperation` значительно безопаснее и совместимее старого `@Redirect`:
```java
@Mixin(HeldItemRenderer.class)
public class HeldItemRendererMixin {
    @WrapOperation(
        method = "renderFirstPersonItem",
        at = @At(value = "INVOKE", target = "Lnet/minecraft/client/render/item/HeldItemRenderer;swingArm(FFLnet/minecraft/client/util/math/MatrixStack;ILnet/minecraft/util/Arm;)V")
    )
    private void wrapSwingArm(HeldItemRenderer instance, float swingProgress, float equipProgress, 
                              MatrixStack matrices, int armX, Arm arm, Operation<Void> original) {
        // Кастомная модификация анимации взмаха (например, для копий или кинжалов)
        original.call(instance, swingProgress, equipProgress, matrices, armX, arm);
    }
}
```

---

## 3. Аксессоры (`@Accessor` и `@Invoker`)

Аксессоры позволяют безопасно читать или вызывать приватные поля без медленной Java Reflection:

```java
@Mixin(CreativeInventoryScreen.class)
public interface CreativeInventoryScreenAccessor {
    @Accessor("selectedTab")
    static ItemGroup getSelectedTab() {
        throw new AssertionError();
    }

    @Invoker("setSelectedTab")
    void callSetSelectedTab(ItemGroup tab);
}
```

---

## 4. Мягкие миксины совместимости (Soft Compat Mixins)

> [!TIP]
> При модификации классов сторонних модов (которые могут отсутствовать в сборке) **всегда** используйте связку `@Pseudo` и `require = 0`:
```java
@Pseudo
@Mixin(targets = "com.othermod.client.gui.OtherModScreen", remap = false)
public class OtherModScreenCompatMixin {
    @Inject(method = "init", at = @At("TAIL"), require = 0)
    private void injectCustomButton(CallbackInfo ci) {
        // Выполняется только если сторонний мод установлен
    }
}
```
Если стороннего мода нет, Fabric Loader тихо пропустит миксин без ошибок в логе.
