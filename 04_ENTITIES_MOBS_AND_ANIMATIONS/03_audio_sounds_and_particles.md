# 03. Звуки, аудиоэффекты и частицы

## 1. Регистрация звуковых событий (`SoundEvent`)

В Minecraft 1.21+ звуковые события регистрируются через метод `SoundEvent.of(...)`:

```java
public class ModSounds {
    public static final Identifier PENGUIN_AMBIENT_ID = Identifier.of("examplemod", "entity.penguin.ambient");
    public static final SoundEvent PENGUIN_AMBIENT = SoundEvent.of(PENGUIN_AMBIENT_ID);

    public static final Identifier PENGUIN_HURT_ID = Identifier.of("examplemod", "entity.penguin.hurt");
    public static final SoundEvent PENGUIN_HURT = SoundEvent.of(PENGUIN_HURT_ID);

    public static final Identifier PENGUIN_DEATH_ID = Identifier.of("examplemod", "entity.penguin.death");
    public static final SoundEvent PENGUIN_DEATH = SoundEvent.of(PENGUIN_DEATH_ID);

    public static void register() {
        Registry.register(Registries.SOUND_EVENT, PENGUIN_AMBIENT_ID, PENGUIN_AMBIENT);
        Registry.register(Registries.SOUND_EVENT, PENGUIN_HURT_ID, PENGUIN_HURT);
        Registry.register(Registries.SOUND_EVENT, PENGUIN_DEATH_ID, PENGUIN_DEATH);
    }
}
```

---

## 2. Конфигурация `assets/<namespace>/sounds.json`

```json
{
  "entity.penguin.ambient": {
    "subtitle": "subtitles.examplemod.penguin.ambient",
    "sounds": [
      "examplemod:entity/penguin/ambient1",
      "examplemod:entity/penguin/ambient2",
      "examplemod:entity/penguin/ambient3"
    ]
  },
  "entity.penguin.hurt": {
    "subtitle": "subtitles.examplemod.penguin.hurt",
    "sounds": [
      "examplemod:entity/penguin/hurt1",
      "examplemod:entity/penguin/hurt2"
    ]
  },
  "entity.penguin.death": {
    "subtitle": "subtitles.examplemod.penguin.death",
    "sounds": [
      "examplemod:entity/penguin/death"
    ]
  }
}
```

Файлы `.ogg` размещаются в `assets/examplemod/sounds/entity/penguin/ambient1.ogg`.

---

## 3. Требования к аудиофайлам и обработка

1. **Формат:** Строго **Mono, 44.1 kHz, Ogg Vorbis (q4 ~ 128 kbps)**.
   - Стереозвуки в Minecraft **не имеют пространственного позиционирования** (звучат одинаково громко в обоих наушниках независимо от положения моба в мире)! Только моно-файлы имеют 3D-звучание.
2. **Команда конвертации через ffmpeg:**
   ```bash
   ffmpeg -i input.wav -ac 1 -ar 44100 -c:a libvorbis -qscale:a 4 output.ogg
   ```
3. **Нормализация громкости (`getSoundVolume`):**
   - Реальные записи звуков животных и природы значительно громче ванильных звуков Minecraft. В классе сущности переопределяйте:
     ```java
     @Override
     public float getSoundVolume() {
         return 0.6F; // Снижение громкости до ванильного уровня
     }
     ```
4. **Звуки детенышей:**
   - Если нет отдельных сэмплов для детеныша, ваниль автоматически повышает питч звука. Если же вы используете реальную запись писка детеныша, отключите повторное повышение питча.

---

## 4. Стандарты лицензирования аудио (Open Source)

> [!CAUTION]
> Для публичных модов допустимы **исключительно** свободные лицензии:
> - **CC0** (Public Domain)
> - **CC-BY** (с указанием автора в `CREDITS.md`)
> 
> **Категорически запрещено** использовать аудио с лицензиями **NC** (Non-Commercial), **ND** (No-Derivatives) или **SA** (Share-Alike), так как они несовместимы с лицензиями мода (MIT, Apache-2.0, LGPL-3.0) и распространением на Modrinth/CurseForge.
