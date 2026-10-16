# 03. Жизненный цикл OpenGL и безопасность графических состояний

## 1. Сохранение и восстановление 16 текстурных слотов (Sodium / Iris / ImmediatelyFast)

> [!CAUTION]
> **ПРИЧИНА ЧЕРНОГО ШРИФТА И ЧЕРНЫХ ПРЕДМЕТОВ В ИНВЕНТАРЕ:**
> Моды оптимизации рендеринга (**Sodium**, **Iris**, **ImmediatelyFast**) кэшируют атласы шрифтов, спрайты предметов и текстуры GUI на различных текстурных юнитах видеокарты (`GL_TEXTURE0` ... `GL_TEXTURE15`).
> 
> Если кастомный шейдер или процедурный рендерер вашего мода сбрасывает все текстурные юниты в 0 (`glBindTextureUnit(i, 0)`), кэш графического движка сбивается.
> - **Симптом:** При открытии чата весь текст рендерится сплошными черными квадратами, а иконки предметов становятся черными силуэтами!
> 
> **Золотое правило:** Перед выполнением кастомного рендеринга сохраняйте привязки активных текстур и сэмплеров, а после рендеринга восстанавливайте их:

```java
int[] boundTextures = new int[16];
int[] boundSamplers = new int[16];

// 1. Сохранение привязок
for (int i = 0; i < 16; i++) {
    boundTextures[i] = GL11.glGetInteger(GL13.GL_TEXTURE_BINDING_2D);
    boundSamplers[i] = GL11.glGetInteger(GL33.GL_SAMPLER_BINDING);
}

// ... Ваш кастомный рендеринг ...

// 2. Восстановление всех 16 слотов
for (int i = 0; i < 16; i++) {
    GL13.glActiveTexture(GL13.GL_TEXTURE0 + i);
    GL11.glBindTexture(GL11.GL_TEXTURE_2D, boundTextures[i]);
    GL33.glBindSampler(i, boundSamplers[i]);
}
GL13.glActiveTexture(GL13.GL_TEXTURE0);
```

---

## 2. Синхронизация с кэшем Blaze3D (`GlStateManager`)

Прямые вызовы OpenGL вроде `glBindTexture` или `glUseProgram` работают в обход кэша состояний ванильного движка Minecraft (`GlStateManager` / `RenderSystem`).
- Вместо прямых `glBindTexture` используйте вызовы `GlStateManager._bindTexture(...)`.
- После завершения работы кастомных шейдеров всегда сбрасывайте активную шейдерную программу:
  ```java
  GL20.glUseProgram(0);
  ```

---

## 3. Очистка нативных ресурсов при отключении (VRAM Leaks)

При частых перезаходах в одиночные миры или повторных подключениях к серверам не освобожденные буферы FBO, VAO, текстуры и шейдеры вызывают утечку видеопамяти (VRAM):

```java
ClientPlayConnectionEvents.DISCONNECT.register((handler, client) -> {
    // Освобождение нативных ресурсов OpenGL
    if (customFboId != 0) {
        GL30.glDeleteFramebuffers(customFboId);
        customFboId = 0;
    }
    if (customTextureId != 0) {
        GL11.glDeleteTextures(customTextureId);
        customTextureId = 0;
    }
});
```

---

## 4. Устранение падений FPS из-за `glFinish()`

> [!TIP]
> Никогда не используйте вызовы `GL11.glFinish()` в регулярном коде отрисовки кадров!
> Вызов `glFinish()` принудительно замораживает процессор (CPU) до тех пор, пока видеокарта (GPU) не закончит все запланированные команды конвейера, вызывая микрофризы и сильное падение фреймрейта.
