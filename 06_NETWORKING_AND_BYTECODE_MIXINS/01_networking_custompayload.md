# 01. Сетевой стек и пакеты (CustomPayload API)

## 1. Архитектура CustomPayload (Minecraft 1.20.5+ и 1.21.4)

Старые сетевые системы с прямым манипулированием `PacketByteBuf` заменены на строго типизированный интерфейс `CustomPayload` с кодеками `PacketCodec`:

```java
package com.example.mod.network;

import net.minecraft.network.RegistryByteBuf;
import net.minecraft.network.codec.PacketCodec;
import net.minecraft.network.codec.PacketCodecs;
import net.minecraft.network.packet.CustomPayload;
import net.minecraft.util.Identifier;

public record SyncPlayerStatsPayload(int mana, int maxMana, boolean active) implements CustomPayload {
    public static final CustomPayload.Id<SyncPlayerStatsPayload> ID = 
        new CustomPayload.Id<>(Identifier.of("examplemod", "sync_stats"));

    // Кортежный кодек для упаковки/распаковки полей
    public static final PacketCodec<RegistryByteBuf, SyncPlayerStatsPayload> CODEC = PacketCodec.tuple(
        PacketCodecs.INTEGER, SyncPlayerStatsPayload::mana,
        PacketCodecs.INTEGER, SyncPlayerStatsPayload::maxMana,
        PacketCodecs.BOOLEAN, SyncPlayerStatsPayload::active,
        SyncPlayerStatsPayload::new
    );

    @Override
    public Id<? extends CustomPayload> getId() {
        return ID;
    }
}
```

---

## 2. Регистрация и обработка пакетов

### 2.1. Регистрация типа пакета (выполняется в общем коде `ModInitializer`):
```java
public class ModNetworking {
    public static void registerCommon() {
        // Регистрация пакета от сервера к клиенту (S2C)
        PayloadTypeRegistry.playS2C().register(SyncPlayerStatsPayload.ID, SyncPlayerStatsPayload.CODEC);
        
        // Регистрация пакета от клиента к серверу (C2S)
        // PayloadTypeRegistry.playC2S().register(...);
    }
}
```

### 2.2. Прием пакета на клиенте (`ClientPlayNetworking`):
```java
public class ModClientNetworking {
    public static void registerClient() {
        ClientPlayNetworking.registerGlobalReceiver(SyncPlayerStatsPayload.ID, (payload, context) -> {
            // КРИТИЧНО: Переключение в главный игровой поток клиента!
            context.client().execute(() -> {
                ClientStatsManager.updateStats(payload.mana(), payload.maxMana(), payload.active());
            });
        });
    }
}
```

### 2.3. Отправка пакета игроку с сервера:
```java
ServerPlayNetworking.send(player, new SyncPlayerStatsPayload(100, 100, true));
```

---

## 3. Защита сетевого буфера от сбоев и эксплойтов

> [!CAUTION]
> **Сетевой краш рассинхронизации буфера (`IndexOutOfBoundsException`):**
> При чтении динамических массивов с ограничением максимального размера:
> `int count = buf.readVarInt();`
> `int safeCount = Math.min(count, MAX_ALLOWED);`
> **ОБЯЗАТЕЛЬНО** вычитывать оставшиеся `(count - safeCount)` элементов из сетевого потока в холостую!
> Если этого не сделать, непрочитанные байты останутся в буфере Netty, сломают чтение заголовков следующего входящего пакета и мгновенно разорвут соединение игрока с ошибкой повреждения пакета данных.
