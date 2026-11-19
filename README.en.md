> **Language:** [Русский](README.md) · English

# AiCreateForMinecraft

A practical knowledge base for building, porting, and optimizing **Minecraft: Java Edition** mods with Fabric and NeoForge.

This is not a tutorial for a single version and not a set of abstract snippets: every entry comes from a mod that actually ships, together with the limits of its applicability. General principles are kept separate from version-specific details, so the base stays useful when moving from 1.21.x to 26.x and beyond.

---

## 🎯 Coverage

| Generation | Java | Loom | What the base covers |
| :--- | :---: | :--- | :--- |
| **1.21 – 1.21.11** | 21 | 1.10 – 1.18 | Full set: build, content, rendering, mixins, migrations |
| **26.1 – 26.3** | 25 | 1.17+ | Full set plus 26.x specifics: SDL3 input, `safeProvider`, Authlib 10.x |
| **1.20.x and older** | 17 / 21 | — | Migration notes to 1.21+ only |

Exact mappings, Gradle, and Loom versions per branch are collected in [09_CROSS_VERSION_MIGRATION_AND_PORTING/04](09_CROSS_VERSION_MIGRATION_AND_PORTING/04_multi_version_gui_input_and_tests.md).

Two loaders, one source tree: Fabric and NeoForge are built from the same code, with no `common`/`fabric`/`neoforge` subprojects. See [02/03](02_LOADERS_AND_MULTILOADER/03_dual_target_single_source.md).

---

## 🚀 Where to start

| Task | Documents |
| :--- | :--- |
| Set up the project, Java, versions, manifests | [01/01](01_FOUNDATIONS/01_toolchain_and_java_versions.md), [01/02](01_FOUNDATIONS/02_project_templates_and_buildscripts.md), [01/03](01_FOUNDATIONS/03_versioning_and_metadata_standards.md) |
| Build one mod for Fabric and NeoForge | [02/03](02_LOADERS_AND_MULTILOADER/03_dual_target_single_source.md) |
| Understand loader APIs and entrypoints | [02/01](02_LOADERS_AND_MULTILOADER/01_fabric_architecture.md), [02/02](02_LOADERS_AND_MULTILOADER/02_neoforge_architecture.md) |
| Items, blocks, recipes, GUI | [03/01](03_CORE_GAMEPLAY_DEVELOPMENT/01_items_and_data_components.md), [03/02](03_CORE_GAMEPLAY_DEVELOPMENT/02_blocks_and_block_entities.md), [03/04](03_CORE_GAMEPLAY_DEVELOPMENT/04_gui_menus_and_screens.md) |
| Mobs, AI, animations, audio | [04/01](04_ENTITIES_MOBS_AND_ANIMATIONS/01_entity_architecture_and_ai.md), [04/02](04_ENTITIES_MOBS_AND_ANIMATIONS/02_geckolib_mobs_and_animations.md) |
| Worldgen, structures, dimensions | [05/01](05_WORLDGEN_AND_STRUCTURES/01_worldgen_json_schemas.md), [05/02](05_WORLDGEN_AND_STRUCTURES/02_custom_structures_and_jigsaw.md) |
| Networking and mixins | [06/01](06_NETWORKING_AND_BYTECODE_MIXINS/01_networking_custompayload.md), [06/02](06_NETWORKING_AND_BYTECODE_MIXINS/02_bytecode_mixins_mastery.md) |
| Rendering, shaders, HUD | [07/01](07_RENDERING_SHADERS_AND_OPENGL/01_modern_render_state_pipeline.md), [07/03](07_RENDERING_SHADERS_AND_OPENGL/03_opengl_lifecycle_and_state_safety.md) |
| Performance and allocations | [08/01](08_PERFORMANCE_AND_OPTIMIZATION/01_tick_loop_and_gc_optimization.md), [08/02](08_PERFORMANCE_AND_OPTIMIZATION/02_memory_caching_and_multithreading.md) |
| Port a mod to a new version | [09/05](09_CROSS_VERSION_MIGRATION_AND_PORTING/05_feature_porting_workflow.md), [09/02](09_CROSS_VERSION_MIGRATION_AND_PORTING/02_migration_1_21_to_26_x.md) |
| Client world queries and heights | [09/06](09_CROSS_VERSION_MIGRATION_AND_PORTING/06_client_world_queries_and_heights.md) |
| Client gametests | [09/08](09_CROSS_VERSION_MIGRATION_AND_PORTING/08_client_gametest_suite.md) |
| Diagnose a crash | [10/01](10_TROUBLESHOOTING_AND_CRASH_CATALOG/01_runtime_exceptions_dictionary.md), checklist [10/02](10_TROUBLESHOOTING_AND_CRASH_CATALOG/02_universal_porting_checklist.md) |
| Publish to CurseForge | [11/01](11_DISTRIBUTION_AND_PUBLISHING/01_curseforge_author_console.md) |

---

## 📁 Structure

| Section | What it covers |
| :--- | :--- |
| **[01_FOUNDATIONS](01_FOUNDATIONS)** | Java 21/25, bytecode targets, `build.gradle` and `gradle.properties` templates, SemVer without a `v` prefix, Fabric and NeoForge manifests |
| **[02_LOADERS_AND_MULTILOADER](02_LOADERS_AND_MULTILOADER)** | Entrypoints, registries, Dual-Target Single-Source (two JARs from one codebase), Architectury, localization without Fabric API |
| **[03_CORE_GAMEPLAY_DEVELOPMENT](03_CORE_GAMEPLAY_DEVELOPMENT)** | Items and Data Components, blocks and BlockEntities, current recipe schemas, GUI, creative tabs, datagen |
| **[04_ENTITIES_MOBS_AND_ANIMATIONS](04_ENTITIES_MOBS_AND_ANIMATIONS)** | Entities and AI goals, physics, server-side fake players, GeckoLib, animations, sounds and particles |
| **[05_WORLDGEN_AND_STRUCTURES](05_WORLDGEN_AND_STRUCTURES)** | Worldgen JSON schemas, structures and jigsaw, GZIP NBT, schematics, custom dimensions and carvers |
| **[06_NETWORKING_AND_BYTECODE_MIXINS](06_NETWORKING_AND_BYTECODE_MIXINS)** | `CustomPayload`, mixins, accessors, `@WrapOperation`, safe placement of server-side mixins |
| **[07_RENDERING_SHADERS_AND_OPENGL](07_RENDERING_SHADERS_AND_OPENGL)** | RenderState pipeline, shaders, WorldRenderer matrices, client-free HUD, texture slots, portals and parallax |
| **[08_PERFORMANCE_AND_OPTIMIZATION](08_PERFORMANCE_AND_OPTIMIZATION)** | Tick loops without extra allocations, caches and multithreading, vanilla isolation, bulk 2D drawing with a single overlay |
| **[09_CROSS_VERSION_MIGRATION_AND_PORTING](09_CROSS_VERSION_MIGRATION_AND_PORTING)** | Migration checklists 1.20 → 1.21.x → 26.x, backporting, multi-version GUI/input, feature porting, client world queries, client gametests |
| **[10_TROUBLESHOOTING_AND_CRASH_CATALOG](10_TROUBLESHOOTING_AND_CRASH_CATALOG)** | Runtime exception dictionary with causes and fixes, a universal pre-release checklist |
| **[11_DISTRIBUTION_AND_PUBLISHING](11_DISTRIBUTION_AND_PUBLISHING)** | CurseForge Author Console, chunked uploads, version metadata, Selenium + Chrome automation |

---

## 🧭 Principles

1. **Only what has been verified.** Every solution comes from a working mod, not from guesswork.
2. **State the version.** If an API differs between 1.21.x and 26.x, the difference is written down instead of hiding behind "in newer versions".
3. **Keep examples neutral.** Mod names, packages, and artifacts are placeholders (`mymod`, `com.example.*`, `<Mod>-<Loader>-<MC>-v<version>.jar`).
4. **Principle first, branch second.** A section explains the mechanism; exact signatures and differences live in section 09 and the version table.

---

## 🤖 Guidelines for AI Assistants

1. **Check the version, not your memory.**
   - Do not move code between branches without checking signatures: `Identifier` vs `ResourceLocation`, Yarn vs Mojang mappings, `Item.Settings.registryKey()`.
   - When a solution uses an API that has changed (item models, Data Components, input, rendering), state which version it is valid for.
2. **Use Dual-Target Single-Source for multi-loader support.**
   - One project, two artifacts, FML compile-stubs; do not split the repository into three submodules.
3. **Respect environment boundaries.**
   - Never call `net.minecraft.client.*` on a dedicated server.
   - Put server-side mixins in the main `"mixins"` array, not in `"server"` (it is ignored in singleplayer).
4. **Do not optimize blindly.**
   - No Stream API in hot tick methods; measure first, cache second.
   - Preserve texture slots during custom rendering, or fonts break in Sodium/Iris.
5. **Port by process.**
   - For a new Minecraft version, follow [09/05](09_CROSS_VERSION_MIGRATION_AND_PORTING/05_feature_porting_workflow.md) instead of ad-hoc edits.

---

## 🛠 Keeping the Base Current

- New Minecraft version → update the coverage table and extend the matching document in `09`.
- Outdated guidance is not deleted silently: mark the version where it stopped working and add the replacement.
- Before committing: examples compile, links are relative, artifact names follow the template.

---

## 📄 License

[Apache-2.0](LICENSE).
