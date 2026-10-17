# Sodiumex low-end performance proposal

**Status:** Implementation is in place but awaits a Java-enabled build and in-game verification. Fabric and NeoForge builds now nest upstream Lithium 0.26.2 (game-logic), FerriteCore 9.0.0 (memory), and ImmediatelyFast 1.17.1 (immediate-mode GUI/HUD/text/entity rendering) for Minecraft 26.3 as independent mods. Their source archives, checksums, license texts, and notices are in `thirdparty/`. Fabric Loader is pinned at 0.19.5. Sodiumex also has these original client changes: (1) an opt-in setting to bypass automatic AFK/minimized/menu FPS caps, default off; (2) chunk-mesh cancellation checks each Z row so stale work yields sooner; (3) an opt-in maximum render distance cap that limits section traversal and mesh scheduling without changing world simulation distance or loaded world data; (4) a section-local block-state lookup path that removes redundant bounds and section-index work from each meshing scan. Static metadata/archive checks pass; compile and launch are unverified because this environment has no Java runtime.
**Repository snapshot inspected:** Minecraft 26.3, Sodium 0.9.3-beta.1 (from `buildSrc/src/main/kotlin/BuildConfig.kt`); Java toolchain 25 and Gradle wrapper 9.7.1. README build requirements now reflect those versions.

## 1. Problem statement

Improve frame-time consistency and usable performance on low-end computers, while preserving visual correctness, mod compatibility, and Sodium's supported Fabric/NeoForge platforms. The requested “around 60 FPS on a Core i3 with 2 GB RAM” is a goal to evaluate—not a guarantee that a mod can meet on every machine.

This repository is a Java Minecraft rendering mod. It already has rendering optimizations and exposes performance controls. `SodiumConfigBuilder` includes an inactivity FPS-limit setting sourced from Minecraft's vanilla options, as well as render/simulation distance, entity culling, and other settings. I found no separate Dynamic FPS implementation in the searched Sodium sources. Dynamic FPS is commonly a separate mod; verify which mod/profile is installed before attributing active-game FPS oscillation to Sodium. Inactivity throttling while unfocused and fluctuating FPS while actively playing are different behaviors.

## 2. Device-specific assessment: Fujitsu LIFEBOOK LH532

The supplied `fastfetch` output describes an Intel Core i3-3110M (Ivy Bridge, 2 cores / 4 threads), Intel HD Graphics 4000 integrated GPU, 5.64 GiB total system RAM, Arch Linux x86_64, KDE Wayland, and a 1366×768 60 Hz display. It shows 2.61 GiB memory in use and about 774 MiB of swap in use at capture time. This is **not a 2 GB total-RAM machine**; it does show some swap use, so memory pressure should be measured while the game runs. Fastfetch's `@ 2.40 GHz` is the CPU's reported clock at capture time, not a promise of sustained clock under gaming load.

As of 17 October 2026, the latest stable Minecraft Java release is **26.3**, released 15 September 2026 ([official release notes](https://www.minecraft.net/en-us/article/minecraft-java-edition-26-3)). This checkout also targets 26.3. Java 25 is the project toolchain, while the README still lists JDK 21; reconcile that documentation and use the runtime required by the actual 26.3 launcher/mod setup.

**Important compatibility gate:** this repository's README says Sodium officially supports graphics drivers with OpenGL 4.5 or newer and notes that older hardware may work but is unsupported. HD Graphics 4000 commonly reports OpenGL 4.2 under Mesa on Linux, which would put this laptop below Sodium's stated official support level; the exact driver output on this machine has not been provided. Before treating FPS tuning as the problem, run `glxinfo -B` (install Arch's `mesa-utils` if needed) and confirm direct rendering, renderer, and OpenGL version. If it reports `llvmpipe`/software rendering, or the game fails the required GL context/features, resolve that driver/backend compatibility first—software rendering will be extremely slow, and a Rust port cannot fix it.

A reasonable first benchmark target is 60 FPS at the native 1366×768 resolution in a simple, reproducible world with no shaders, a low render distance (start at 4–6 chunks), low simulation distance (start at 4), and a modest entity/particle load. These are test starting points, not universal optimal settings. If frame times cannot consistently stay below 16.67 ms, compare a stable 30 or 40 FPS cap instead of chasing an unstable 60; a cap can improve consistency, but cannot remove CPU/GPU stalls. Keep the laptop on AC and check for thermal/power throttling during sustained runs.

### Device-specific test steps

1. Record `java -version`, `glxinfo -B`, active launcher JVM arguments, and the complete mod list. Confirm the game is using hardware acceleration and the intended Mesa driver.
2. Benchmark a clean 26.3 instance first, then Sodiumex, then the same setup with the suspected FPS-related mod toggled. Keep world, camera path, resolution, settings, and focus state unchanged.
3. Capture frametime percentiles/1% lows, GPU/CPU utilization, Java heap use/GC pauses, system memory, and swap-in/out while traversing and rotating in a fixed area. On Linux, `MangoHud` can optionally provide an in-game overlay; do not treat its displayed average FPS alone as evidence.
4. For a conservative manual profile, begin with shaders off, default/low graphics quality, render distance 4–6, simulation distance 4, particles low, and FPS capped to 60; test 40/30 caps if 60 is uneven. Change one option at a time and retain visual-quality settings the user values.
5. Do not assume “more heap is faster.” Set heap based on measured peak use and system headroom; watch for swap churn and desktop/game memory contention. The available data does not establish a universally safe `-Xmx` value.

**Assessment:** 60 FPS may be reachable in light scenes if the HD 4000 is hardware-accelerated and compatible, but it cannot be responsibly promised for current Minecraft across worlds/mods. The CPU/GPU generation and official Sodium OpenGL support floor are the main risks. First verify GL support and collect a baseline; the best useful deliverable may be stable frame pacing at a lower cap rather than a universal 60-FPS claim.

### Scope and feasibility of the expanded goals

This checkout is explicitly client-side (`fabric.mod.json` declares `environment: client`; publishing metadata marks client required and server not required). So the following goals do not all belong to the existing Sodium renderer:

| Goal | What Sodiumex can reasonably do | Boundary / trade-off |
|---|---|---|
| Loading/resource reload | Profile and optimize client renderer/model preparation and resource handling where measured. | World creation, datapack/mod loading, storage speed, and other mods can dominate; do not promise a universal loading time. |
| Chunk generation vs. chunk meshing | Optimize client-side render-mesh building, uploads, and scheduling; bundled Lithium supplies supported game-logic optimizations to integrated single-player. | Remote world generation is still server work: the remote server must install compatible Lithium. Neither path makes generation cost-free. |
| World rendering/FPS | Cull invisible work, optimize renderer scheduling and GPU submissions, and report frame-time tails. | The GPU/driver floor and scene complexity set hard limits. A renderer cannot provide high FPS on unsupported/software GL. |
| Dynamic lighting | Potentially implement or integrate an explicitly optional, cached client visual feature. | Light updates add CPU/GPU work and may conflict with shaders/resource packs. “No lag” is not a valid guarantee; disabling it must restore vanilla-like behavior. |
| Mob AI/single-player backend | Bundled Lithium can apply upstream-supported game-logic optimizations to the local integrated server; FerriteCore also reduces applicable memory overhead. The client renderer remains separate. | The remote dedicated server must run compatible server-side mods itself. Optimizations preserve vanilla semantics by intent but still require mod-compatibility testing; no arbitrary load guarantee. |
| Thousands of TNT / players | Client culling can reduce drawing cost for off-screen entities; Lithium may reduce some supported tick costs on the integrated server or a server where installed. | Explosion/block updates, network traffic, and player simulation still have workload proportional to activity. Hard caps, batching, or delayed explosions change gameplay and need explicit approval. No-lag at arbitrary counts is impossible to guarantee. |
| PvP | Reduce client render/input overhead and measure client-side frame-time/input latency. | Server tick rate, network latency, hit registration, and fairness are outside a renderer-only mod. No gameplay automation or hidden advantage should be added. |
| Higher render/simulation distances | Improve efficiency at tested distances and expose user controls. | More visible terrain raises rendering/mesh cost; more simulation distance raises ticking/entity cost. Holding performance constant while increasing both without limit is not achievable without quality or simulation trade-offs. |

**Existing invisible-geometry work:** This checkout already performs chunk/section visibility culling (`RenderSectionManager` and `OcclusionCuller`), builds section occlusion visibility during `ChunkBuilderMeshingTask`, selects visible directional mesh slices in `DefaultChunkRenderer`, and exposes block-face culling. The render tree gates mesh-task scheduling for sections it rejects, helping avoid mesh work on fully hidden underground terrain; it does not unload world data, and terrain revealed through caves/portals remains visible and eligible for work. Minecraft still needs loaded world state for boundaries, updates, and gameplay. “Not visible” means “not submitted/rendered,” not “remove it from the loaded world.” Any further underground/occlusion optimization must preserve caves, translucent blocks, modded renderers, block entities, and correct reveal after world changes.

**Implementation rule:** Preserve vanilla/gameplay semantics by default. Do not silently hide players, suppress mob AI, cap entities/TNT, delay explosions, or reduce simulation. Any such quality/gameplay trade-off must be an opt-in, clearly described setting with multiplayer fairness and compatibility review.

## 3. Recommendation

**Do not rewrite Sodium in Rust as the first optimization.** Keep the renderer and game integration in Java, profile representative workloads, and make the smallest measured changes. Consider Rust/native code only for a demonstrated, self-contained CPU hotspot that cannot be improved adequately in Java, with an explicit maintenance and compatibility case.

A native rewrite is not automatically faster: Minecraft, its mod APIs, and OpenGL calls remain Java/game-thread constrained; JNI or a foreign-function boundary adds data transfer and lifecycle complexity; and native libraries must be built and tested per OS/architecture. The JVM JIT can optimize hot Java paths, so benchmark evidence—not language labels—must justify any native code.

Likewise, “all optimizations from every mod in one jar” is not a safe or well-defined goal. Sodium primarily optimizes chunk rendering; Lithium targets game logic, FerriteCore targets memory, and ImmediatelyFast targets immediate-mode rendering. Their upstream projects are bundled as independent nested mods after version/license review, not merged into Sodiumex source. Features can still overlap or conflict in modpacks, so keep the dependency pins explicit and verify both loaders before release. Entity Culling is not bundled because its published protective license prohibits redistributing its jar; see `docs/OPTIMIZATION_STACK.md`.

## 4. Functional requirements and acceptance criteria

1. Keep the inactivity FPS limit understandable and user-controllable; explicitly distinguish inactive/background limiting from active-play performance.
2. Provide a documented low-end preset that uses supported game settings (for example, modest render and simulation distances, a reasonable FPS cap, and optional visual trade-offs) without silently changing settings or reducing correctness.
3. Identify active-play stutter causes through repeatable frame-time and workload measurements before changing render or chunk scheduling.
4. Preserve supported-loader behavior and compatibility, with an option or safe fallback for any newly introduced optimization that can cause visual or mod conflicts.
5. Evaluate the 60-FPS goal on a precisely specified test machine and scene. At 60 FPS, each frame has a 16.67 ms budget. Report average FPS, frame-time percentiles/1% lows, stutter, memory use, and visual correctness; do not report only a peak FPS number.
6. Never claim a universal 60-FPS guarantee. Record failures and hardware limitations, including integrated GPU, driver, resolution, world, resource packs, and background load.

### Given / When / Then

- **Given** the game is unfocused, **when** the inactivity limit is set to unlimited/disabled (if offered by the vanilla option) or a chosen cap, **then** Sodiumex must preserve that configured behavior and must not introduce a separate, hidden limiter.
- **Given** a documented low-end benchmark setup, **when** a release candidate is measured against the same baseline, **then** the report includes repeatable frame-time and memory results, visual checks, software/hardware details, and any regressions.
- **Given** a scene where the CPU, GPU, memory, or chunk-build worker is the bottleneck, **when** a proposed optimization is enabled, **then** it must improve its stated target without introducing rendering errors, deadlocks, unbounded queues, or unacceptable compatibility failures.
- **Given** an unsupported or unstable native backend, **when** it fails to load, **then** the game must fail safely with an actionable diagnostic or use the Java path; it must not crash the game through an unsafe fallback.

## 5. Scope and non-functional requirements

- **Performance:** Optimize measured bottlenecks; track frame-time distributions and chunk-build latency, not just average FPS. Avoid adding per-frame allocations, synchronization, or polling without evidence.
- **Reliability/correctness:** Preserve vanilla visual output and resource lifecycles. Any asynchronous work must be bounded, cancellable where appropriate, and safely shut down.
- **Security:** Treat native binaries, resource-pack/model data, and mod-provided data as trust boundaries. Do not introduce downloaded executable code or unsafe native input handling.
- **Compatibility:** Keep Fabric and NeoForge support. Test with representative renderer, shader, resource-pack, and gameplay mods; document known conflicts.
- **Accessibility/UX:** Keep performance controls discoverable and clearly explain visual/quality trade-offs; do not silently reduce settings.
- **Observability/privacy:** Offer opt-in local diagnostics or benchmark output; do not upload world, username, or system data by default.
- **Maintainability/portability:** Prefer Java and existing project patterns. Any native option needs a narrow API, fallback, per-platform build/test plan, and ownership commitment.

## 6. Risk analysis and mitigations

| Risk | Why it matters | Mitigation / verification |
|---|---|---|
| Unfounded attribution to Dynamic FPS | A background limiter does not normally explain active-play spikes; changing the wrong subsystem can regress performance. | Reproduce with a clean mod profile; capture frame times, CPU/GPU load, GC pauses, and focus state. |
| 60 FPS is impossible on some target hardware | “Core i3” spans generations and does not specify GPU, thermal state, resolution, world, or RAM available to Minecraft. | Specify exact test systems/scenes and classify results; state hardware-dependent limits explicitly. |
| Low RAM causes paging or crashes | The operating system, JVM, native graphics driver, and game all consume memory; assigning most of 2 GB to Java can make the system worse. | Measure whole-system and process memory; avoid promising a JVM heap size without testing; document minimum practical OS/runtime requirements. |
| Aggressive culling or reduced work hides visible content | Incorrect occlusion/culling can cause visual artifacts or break modded blocks/entities. | Add scene-based correctness tests and a setting/fallback for risky changes. |
| More bundled features increase conflicts and support burden | Multiple mods can patch the same game code or assume different ordering/invariants. | Feature-by-feature compatibility review, mod matrix, opt-in rollout, and disable/fallback path. |
| Native code introduces crashes and packaging burden | ABI, memory safety, platform-specific builds, driver/JVM interaction, and library loading are additional failure modes. | Keep Java as default; require isolated API, signed/reproducible artifacts where feasible, platform CI, sanitizers/fuzzing as relevant, and Java fallback before considering release. |
| FPS cap hides instability rather than fixing it | A cap can smooth GPU-bound behavior but cannot resolve CPU stalls, chunk rebuild spikes, or paging. | Compare capped and uncapped frame-time traces; retain bottleneck labels and tail percentiles. |

## 7. Proposed design and delivery phases

### Phase A — Baseline and diagnosis (no behavior change)

1. Record exact Minecraft/Sodiumex/loader/Java versions and enabled mods.
2. Build a small reproducible benchmark set: static view, rapid camera rotation, chunk traversal, entity-heavy area, and a representative modded scene; include cold and warmed runs.
3. Capture frame-time percentiles, 1% lows, CPU/GPU utilization, GC pauses/allocation, chunk build/upload queue depth and latency, memory pressure, render distance, resolution, VSync/FPS cap, and focus state.
4. Compare Sodiumex with the same setup without it and with the suspect FPS-related mod removed/disabled. Keep all other conditions fixed.

**Exit criterion:** A written profile identifies the top bottlenecks and demonstrates repeatable baseline variance small enough to evaluate changes.

### Phase B — First Minecraft-internal change (one measured bottleneck)

Changing Minecraft internals is in scope: Sodium already transforms/integrates with Minecraft rendering. We should do it through narrowly scoped mixins and Sodium's existing renderer/chunk systems—not by patching game jars or changing unrelated gameplay behavior.

1. **Gate on graphics support:** verify the actual GL renderer/version and that the client is hardware accelerated. If the current Minecraft/Sodium combination cannot create the required GL context on HD 4000, stop and document that blocker; do not disguise software rendering as a mod optimization problem.
2. **Select one internal path from evidence:**
   - If GPU-bound: test existing frustum/occlusion/block-face/entity culling and draw submission in `SodiumWorldRenderer`, `RenderSectionManager`, and the chunk renderer. Reduce submitted work only when conservative visibility rules preserve all visible geometry.
   - If CPU-bound during exploration: profile chunk meshing and upload in `ChunkBuilderMeshingTask` and `RenderSectionManager`; inspect task priorities, stale-job cancellation, queue growth, and main-thread upload budget. Keep queues bounded and avoid adding worker threads by assumption—the i3-3110M has only two physical cores, and extra workers can worsen contention.
   - If memory/GC-bound: measure chunk slice, mesh, staging-buffer and pool lifetimes; reduce allocations or retained data only where traces show a real cost. Do not set a fixed heap size or disable GC as a supposed optimization.
   - If FPS varies only while unfocused: audit vanilla inactivity limit mapping. Do not alter active rendering or add another limiter to solve background behavior.
3. Add a small profiling/diagnostic path that reports aggregate stage timings and queue/worker state only when explicitly enabled; do not log every frame or collect/upload private world data.
4. Change one behavior behind an explicit option if visual or compatibility trade-offs are plausible. Keep the prior behavior as a rollback path until correctness and performance are established.

**Phase B exit criterion:** on a reproducible benchmark, the change improves its targeted metric beyond run-to-run noise, has no incorrect missing/extra geometry, and does not worsen frame-time tails or memory pressure materially. Otherwise revert it.

### Phase C — Further internal optimizations and single-mod scope

The current integration selectively nests the exact 26.3 builds of Lithium, FerriteCore, and ImmediatelyFast as independent upstream mods; do not duplicate or bulk-copy their source. After loader packaging is verified, evaluate further optimizations only against a measured bottleneck. Any new game simulation/ticking behavior is a distinct, higher-compatibility-risk scope and needs its own license review, gameplay invariants, benchmark, compatibility tests, documented option/default, and rollback.

### Phase D — Native experiment (only if profiling justifies it)

Prototype one isolated CPU-only hotspot behind an interface, with the Java implementation as reference and fallback. Benchmark end-to-end benefit including conversion/call overhead. Check memory ownership, bounds, cancellation, crash behavior, platform packaging, license, and loader interactions. Do not move OpenGL/game-thread operations or broad game logic into Rust merely to call it “low level.” Reject the prototype if improvement is not material, stable, and reproducible.

## 8. Test and verification plan

A repository scan found no existing `src/test` sources, so adding focused automated coverage is part of the implementation plan rather than assuming regression tests already exist.

- **FPS-toggle regression:** With the option off (including when absent from an older config), verify vanilla tracker results are passed through unchanged. With it on, verify the configured FPS limit is used while AFK, minimized, and in menus; verify VSync still applies, changing the option takes effect, and the setting persists after restart. Verify active-play behavior is not otherwise modified.
- **Unit/regression:** Deterministic tests for any changed scheduling, culling, cache, or configuration behavior; verify legacy config defaults and migration behavior.
- **Integration:** Run both Fabric and NeoForge launch/test paths for the supported Minecraft version; verify options persist and apply correctly.
- **Visual correctness:** Fixed-world screenshots or image comparisons for terrain, fluids, block entities, entities, resource packs, and supported shader/renderer combinations; review differences rather than blindly accepting pixel variance.
- **Performance:** Repeat benchmark runs after warm-up; report median and tail frame times, 1% lows, memory, GC, chunk latency, and run-to-run variance. Compare against baseline and record machine and settings.
- **Compatibility:** Test clean profile and a documented representative mod matrix, including the suspected FPS mod and relevant rendering/gameplay mods.
- **Native prototype, if ever approved:** Cross-platform CI, bounds/failure tests, sanitizers/fuzzing where applicable, library-load failure test, and Java fallback test.

No performance target should be considered passed from a single machine or a single run. Release evidence should include exact commands/setup, artifacts, and the raw results needed to reproduce the comparison.

## 9. Deployment, rollback, and operational notes

Ship performance changes incrementally and keep them independently revertible. For risky or opt-in changes, persist a user-visible setting and provide a safe default. If a release causes crashes, missing geometry, hangs, or worse frame-time tails, revert that feature or release and publish a compatibility note; do not silently delete user configuration. Native code, if ever accepted, must not be required to start the game and must have tested Java fallback behavior.

## 10. Remaining verification before release

The target is Minecraft Java 26.3. Before release, the outstanding tasks are:

1. Run Gradle builds for Fabric and NeoForge using JDK 25, and inspect the outputs to ensure all three nested jars are present exactly once with valid loader metadata.
2. Smoke-test launches on both loaders, including a chest/inventory screen, text/HUD, block entities, particles, and a cave/occlusion scene; check for missing visuals, mixin conflicts, or duplicate-mod warnings.
3. On the target LIFEBOOK LH532, record `glxinfo -B`, exact JVM/mod list, and repeatable frametime/memory traces. Verify the FPS-throttling toggle in-game and compare the performance stack against baseline.
4. Keep user-facing claims hardware-dependent. Do not promise 60 FPS or begin a Rust rewrite until graphics compatibility and benchmark data support it.

**Recommended next step:** Java-enabled CI build plus a two-loader smoke test. Then profile on the target hardware and only add further changes against a measured bottleneck.
