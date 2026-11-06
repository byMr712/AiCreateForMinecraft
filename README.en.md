> **Language:** [Русский](README.md) · English

# AiCreateForMinecraft

My personal knowledge base for creating, porting, and optimizing mods for **Minecraft (Java Edition)**.

In this repository, I have gathered my solutions, code templates, and notes for **Fabric** and **NeoForge**: from build configuration and content registration to mixins, custom shaders, structures, and fixes for common game engine crashes. The information is organized to be easy to navigate for both humans and AI.

---

## 📁 Repository Structure

| Section | Description |
| :--- | :--- |
| **[01_FOUNDATIONS](file:///E:/GitHub/AiCreateForMinecraft/01_FOUNDATIONS)** | Java versions (21/25), bytecode targets, template `build.gradle`, `gradle.properties`, SemVer `X.X` format (no `v` prefix), and manifests. |
| **[02_LOADERS_AND_MULTILOADER](file:///E:/GitHub/AiCreateForMinecraft/02_LOADERS_AND_MULTILOADER)** | Fabric, NeoForge, my Dual-Target build setup (two JARs from one codebase via Loom without subprojects), Architectury, and localization without Fabric API (key binding translations via `Language`). |
| **[03_CORE_GAMEPLAY_DEVELOPMENT](file:///E:/GitHub/AiCreateForMinecraft/03_CORE_GAMEPLAY_DEVELOPMENT)** | Items with `.registryKey()`, Data Components, Client Item Definitions, blocks, render layers, 1.21+ recipe schemas, and GUI screens. |
| **[04_ENTITIES_MOBS_AND_ANIMATIONS](file:///E:/GitHub/AiCreateForMinecraft/04_ENTITIES_MOBS_AND_ANIMATIONS)** | Entity creation, AI goals, water physics, Server-Side Fake Players, GeckoLib integration, animations, and sounds. |
| **[05_WORLDGEN_AND_STRUCTURES](file:///E:/GitHub/AiCreateForMinecraft/05_WORLDGEN_AND_STRUCTURES)** | Worldgen schemas, custom structures (`structure/` folder, mandatory GZIP NBT compression `1F 8B`), and schematic conversion. |
| **[06_NETWORKING_AND_BYTECODE_MIXINS](file:///E:/GitHub/AiCreateForMinecraft/06_NETWORKING_AND_BYTECODE_MIXINS)** | `CustomPayload` networking, mixins, accessors, `@WrapOperation`, and safe placement for server-side mixins. |
| **[07_RENDERING_SHADERS_AND_OPENGL](file:///E:/GitHub/AiCreateForMinecraft/07_RENDERING_SHADERS_AND_OPENGL)** | EntityRenderState, shader paths in 1.21.4, WorldRenderer matrices, Zero-Client HUD via fonts and negative spaces, 16 texture units, and portals. |
| **[08_PERFORMANCE_AND_OPTIMIZATION](file:///E:/GitHub/AiCreateForMinecraft/08_PERFORMANCE_AND_OPTIMIZATION)** | Optimizing ticks without Stream API allocations, `BlockPos.Mutable`, safe chunk access, and atomic config saving. Bulk 2D drawing: one overlay texture instead of thousands of draw calls, CPU rasterisation, view-keyed caching, and bbox culling. |
| **[09_CROSS_VERSION_MIGRATION_AND_PORTING](file:///E:/GitHub/AiCreateForMinecraft/09_CROSS_VERSION_MIGRATION_AND_PORTING)** | Checklists for updating mods from 1.20 to 1.21.4 and 1.21 to 26.x (SDL3, `safeProvider`), and native library backporting. Practical multi-version GUI/input porting for 1.21–1.21.11 and 26.x, mixins, client gametests, and porting large feature sets between branches. Client world queries: why `hasChunk` does not mean "loaded", where `-64` heights come from, and how to read the real surface height. Per-dimension view memory and a non-clickable "you were here" pin without extra entities. |
| **[10_TROUBLESHOOTING_AND_CRASH_CATALOG](file:///E:/GitHub/AiCreateForMinecraft/10_TROUBLESHOOTING_AND_CRASH_CATALOG)** | Breakdown of common runtime crashes with ready-to-use fixes and a 35-step pre-release verification checklist. |
| **[11_DISTRIBUTION_AND_PUBLISHING](file:///E:/GitHub/AiCreateForMinecraft/11_DISTRIBUTION_AND_PUBLISHING)** | CurseForge publishing: the new Author Console (`authors.curseforge.com/_api`), chunked uploads, version metadata, Selenium + Chrome automation, and getting past Cloudflare. |

---

## 🤖 Guidelines for AI Assistants

If you are an AI working on tasks in my repositories:

1. **Use the current patterns from this base:**
   - Avoid deprecated code (in 1.21.4, item models live in `items/<id>.json`, not `models/item/`; item NBT is replaced by Data Components; `Item.Settings` requires `.registryKey()`).
   - If multi-loader support is needed, use the Dual-Target Single-Source approach with FML compile-stubs instead of splitting the project into three submodules.
2. **Mind the environment:**
   - Never call client-only classes on a dedicated server.
   - Put server-side mixins in the main `"mixins": [...]` array, not in `"server": [...]` (it is ignored in singleplayer).
3. **Write clean, efficient code:**
   - Avoid Stream API allocations in hot tick methods (`tick()`).
   - Preserve and restore OpenGL texture units during custom rendering so you don't break fonts in Sodium/Iris.
