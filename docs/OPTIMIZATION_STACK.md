# Sodiumex optimization stack (Minecraft 26.3)

Sodiumex remains a **client-side** mod. Its renderer and these nested mods can improve the local client (and integrated single-player where applicable); they do not make an unmodified remote server generate chunks, tick entities, or process packets faster.

## Bundled, independent upstream mods

All three projects are embedded as their original, independent loader mods. Sodiumex does not merge or fork their code. Install the Sodiumex jar; do not also install duplicate copies of these three mods.

| Mod | Version / license | What it targets | Implementation in Sodiumex |
| --- | --- | --- | --- |
| [Lithium](https://github.com/CaffeineMC/lithium/tree/mc26.3-0.26.2) | 0.26.2, LGPL-3.0-only | Vanilla game-logic hot paths (world/entity/block behavior); client-side and integrated-server performance, not remote-server ticks | Nested Fabric and NeoForge jars |
| [FerriteCore](https://github.com/malte0811/FerriteCore/tree/build-9.0.0) | 9.0.0, MIT | Memory overhead; its optimizations share/deduplicate repeated structures such as block-state data and cache predicates instead of keeping redundant copies | Nested Fabric and NeoForge jars |
| [ImmediatelyFast](https://github.com/RaphiMC/ImmediatelyFast/tree/v1.17.1) | 1.17.1 for 26.3, LGPL-3.0-or-later | Client immediate-mode rendering; batches drawing and GPU uploads for GUI/HUD, text, entities, block entities, particles, and maps | Nested Fabric and NeoForge jars |

The pinned Modrinth versions are recorded in `buildSrc/src/main/kotlin/BuildConfig.kt`. The Fabric build uses Loom's nested-mod inclusion; NeoForge uses Jar-in-Jar. Corresponding upstream source archives, checksums, attribution, and license texts are kept in `thirdparty/` and `thirdparty/NOTICE.txt`, and the build scripts package them under `META-INF/thirdparty/` inside each distributable mod jar.

These optimizations are complementary: Sodium focuses on chunk rendering, ImmediatelyFast reduces overhead in other immediate-mode draw paths, FerriteCore targets RAM consumption, and Lithium targets game-logic work. Gains depend on hardware, world, modpack, and workload; no fixed FPS or memory reduction is promised.

## Original Sodiumex client-side work

In addition to those upstream mods, this branch adds:

- **Crowd visibility fast path:** entity AABBs that fit within one 16-block section now use a single visibility-bit lookup instead of two nested section scans. Since most players and mobs occupy one section, this reduces per-entity CPU overhead in dense lobbies without changing the culling decision.
- **Render-distance ceiling:** a client preference can cap the effective distance used by Sodium's section traversal without changing the Minecraft slider. This is a predictable way to reduce the amount of nearby world considered for rendering on weak devices. It is a radius cap, **not** ray/path-tracing visibility: it does not mean Minecraft never loads unseen chunks or that every underground block is skipped.
- **Section-local block-state reads:** the chunk mesher fetches the center section's cloned state array once, computes each row base once, and indexes the 4096 states directly. This avoids per-block world-bounds checks, section-index calculations, and accessor calls. It is confined to the known center section; output semantics are unchanged. This is a targeted hot-loop reduction, not a measured FPS claim yet.
- **Faster stale-mesh cancellation:** chunk meshing workers check cancellation per horizontal row rather than waiting for an entire 16-block-high slice, limiting CPU spent finishing obsolete work after a section is invalidated or the camera moves.
- **Automatic-FPS-throttle override:** a client preference bypasses Minecraft's automatic low-FPS/background throttle while retaining the user's configured frame cap. It can raise power use and heat when unfocused; it is not an FPS increase by itself.

These are conservative client-side controls, not claims of novel server optimizations. Sodium's existing section-visibility graph also gates mesh-task scheduling: sections rejected by the current visibility tree are not queued as ordinary deferred mesh work, which is the safe route for avoiding work on fully hidden underground terrain. This is section-level occlusion, not chunk unloading or entity path tracing; terrain exposed through a cave opening or other visible portal remains eligible to render. Entity visibility should continue to use Sodium/Minecraft's established path unless a separately tested, safe implementation is added.

## Why Entity Culling is not bundled

[Entity Culling](https://modrinth.com/mod/entityculling?version=26.3) is the most directly targeted optional add-on for a crowded lobby: it uses asynchronous visibility checks to skip drawing entities and block entities hidden behind opaque geometry. Its [published license](https://tr7zw.github.io/project/entityculling/) prohibits redistribution of its jar, so Sodiumex does **not** nest its binary or copy its code; install it separately from its official project page if you want that extra wall-level culling. It only changes client rendering, not server mob/player simulation. As with any occlusion culler, review its settings for modded entities/block entities and test your visual setup.

## Verification status

The dependency pins, source archives, checksums, and license notices are present in the repository. A Gradle compile and Minecraft launch have **not** been verified in this environment because Java is unavailable. Treat packaging/runtime compatibility as pending a Java-enabled CI build and smoke test.
