# Shader Compilation, Pipeline Prebuilding, and Cache Architecture

This note documents how this fork translates Xbox Series X|S console shaders,
prebuilds and caches host Direct3D 12 pipelines, and separates compilation
stalls from runtime memory and synchronization bottlenecks.

All repository behavior below is verified against `main` at `7759288c`.
Background on console hardware, compiler backends, and PC pipeline caching is
grounded in primary vendor specifications and engineering literature listed in
[Primary references](#primary-references). Closed-source internals (PC driver
user-mode compilers and NDA `d3d12_xs.dll` paths) are marked as outside public
verification.

---

## 1. Console AOT vs. PC JIT: why even AMD-on-AMD requires lifting and recompilation

On Xbox Series S and Series X, shader compilation happens ahead of time (AOT)
on the studio's build machines. Both consoles share one target GPU
microarchitecture — AMD RDNA 2 (`GFX10.3`) — and each title runs on a versioned
Game OS linked to its matching `d3d12_xs.dll` runtime. Because the ISA and
graphics ABI are fixed for the life of the title build, GDK tools
(`DxCompiler_xs.dll` / `DXC.exe`, and experimentally the Mesa ACO-based
`aco_xbox.dll` / `xdxc.exe` backend in GDK 2610) emit native RDNA 2 microcode
(and optionally offline-serialized PSOs) directly into the read-only `G:\`
package. Console OS updates do not invalidate those binaries.

Running the same title on Windows PC — even on an AMD Radeon GPU such as an
RX 7800 XT (RDNA 3) — inverts that lifecycle for three structural reasons:

1. **ISA and microarchitecture divergence:** GPU ISAs are not binary-compatible
   across vendors (AMD vs. NVIDIA vs. Intel) or across generations of the same
   vendor (RDNA 2 vs. RDNA 3 vs. RDNA 4). The host D3D12 driver does not accept
   console RDNA 2 shader containers directly; it expects portable SM 6.6 DXIL.
2. **State-coupled host PSO compilation:** On D3D12, `ID3D12PipelineState`
   couples shader bytecode with the Root Signature, input layout, render-target
   formats, blend, depth-stencil, and rasterizer state. On RDNA hardware,
   vertex-attribute fetch prologs and pixel-export packing epilogs depend on
   those fixed states, so translating shaders into DXIL alone does not produce
   ready-to-bind host pipelines.
3. **WDDM driver lifecycle and memory split:** Windows drivers compile DXIL to
   native host ISA (RDNA 3 ISA, NVIDIA SASS, etc.) JIT inside the vendor
   user-mode driver (UMD), under WDDM (`dxgkrnl.sys`, `VidMm`, `VidSch`, TDR),
   with separate host RAM and GPU VRAM across PCIe instead of the console's
   coherent unified GDDR6 memory.

As a result, this fork runs a two-stage pipeline:

```text
Console container (GFX10.3 / RDNA 2 microcode in title files or memory)
  │
  ├─► Stage 1: Reverse lifting (victory-shader)
  │     GFX10 → SSA IR → structurization → resource tracking → DXIL (SM 6.6)
  │     Persisted in: <cache>/shaders  (crates/victory-gpu/src/shader_cache.rs)
  │
  └─► Stage 2: Host PSO compilation (host D3D12 UMD + ID3D12PipelineLibrary1)
        DXIL + Root Signature + GraphicsState → host native ISA + PSO object
        Persisted in: <cache>/pipelines/host-pipelines.bin
        Used-set list: <cache>/pipelines/early-pipelines.txt
        (crates/victory-gpu/src/pipeline/{library,make,host}.rs)
```

---

## 2. Two-layer persistence model in this fork

Keeping the DXIL translation cache separate from the D3D12 Pipeline Library
mirrors standard multi-layer caching practice: portable IR (DXIL) survives
host driver updates, while native machine-code PSOs are tightly coupled to the
adapter and driver ABI.

| Property | Layer 1: DXIL translation cache | Layer 2: Host PSO library & early list |
|---|---|---|
| Location | `<cache>/shaders` | `<cache>/pipelines/host-pipelines.bin` + `early-pipelines.txt` |
| Stored payload | Serialized `HostShader` (`VSH1`: DXIL + binding metadata) or deterministic `ShaderError::Unsupported` refusal (`VSHE`), prefixed with a 32-byte BLAKE3 digest | D3D12 `ID3D12PipelineLibrary1` blob of driver-compiled PSOs, plus up to `EARLY_MAX` (`8192`) title pipeline keys used in prior runs |
| Implementation | `crates/victory-gpu/src/shader_cache.rs`, `crates/victory-shader/build.rs`, `crates/victory-pipeline/src/cache.rs` | `crates/victory-gpu/src/pipeline/library.rs`, `crates/victory-gpu/src/pipeline/make.rs` |
| Identity key | `CacheKey::from_parts("shader.dxil", 2, &[build, wave, view_limit, canonical_container])` | Library entry: `library::name(kind, root, code, state)` (32-hex BLAKE3). Early key: `make::key(shaders, graphics)` |
| Survives rebuilds outside `victory-shader`? | **Yes** (`build()` keys on `victory_shader::SOURCE_HASH`, not `victory.exe`) | **Yes** (unless translated DXIL, root signature, or host state changes) |
| Survives GPU driver updates (same GPU)? | **Yes**, provided `host_max_wave()` and `view_heap_limit()` are unchanged | **No** (D3D12 rejects the blob with driver mismatch; starts an empty library) |
| Survives GPU swap? | Only if `host_max_wave()` and `view_heap_limit()` match | **No** (adapter mismatch) |

### Layer 1 details (`shader_cache.rs` and `victory-shader/build.rs`)

- **Translator identity (`victory_shader::SOURCE_HASH`):** computed at build
  time in `crates/victory-shader/build.rs` with BLAKE3 key derivation over
  `rustc -vV`, `TARGET`, `CARGO_CFG_TARGET_FEATURE`, sorted `CARGO_FEATURE_*`
  flags of `victory-shader`, `Cargo.toml`, `build.rs`, workspace `Cargo.toml`
  and `Cargo.lock`, and every file under `crates/victory-shader/src/` (by
  relative path so identical checkouts can share a cache). The outer
  `victory.exe` binary is **not** hashed into the key.
- **Runtime configuration included in `build()`:** `loop_limit()`
  (`VICTORY_SHADER_LOOP_LIMIT`, default `4096`), `group_sync()`
  (`VICTORY_GROUP_SYNC`), `shader_opt()` (`VICTORY_SHADER_OPT`), and seven
  explicit translator switches: `VICTORY_WAVE_WIDE`, `VICTORY_WAVE_DEEP_SKIPS`,
  `VICTORY_WAVE_OWN_BITS`, `VICTORY_F16_NEAREST`, `VICTORY_WAVE_LOCAL`,
  `VICTORY_MESH_OUTPUT_STRICT`, and `VICTORY_STRICT_JOINS`.
- **Host GPU facts (`target()`):** `[host_max_wave(), view_heap_limit()]`,
  probed via `probe_view_limit()` and checked before and after translation in
  `lookup()` so an entry is never stored under mismatched host capabilities.
- **Canonical container slicing (`canonical_container()`):** trims DXBC inputs
  to the declared length at header bytes `0x18..0x1c` so trailing allocation
  padding never splits cache keys between offline warm-up and runtime (while
  pipeline identity retains the title's original container bytes).
- **Atomic writes and cached refusals:** entries are published atomically via
  `tempfile::NamedTempFile::persist` (`crates/victory-pipeline/src/cache.rs`).
  Deterministic `ShaderError::Unsupported` results are stored as `VSHE`
  records so repeated lookups skip re-running the translator; malformed inputs
  (e.g. `ShaderError::NotDxbc`) are not cached. Corrupted records fail the
  32-byte BLAKE3 digest check in `decode_record()` and are transparently
  retranslated.

### Layer 2 details (`pipeline/library.rs`)

PC pipeline cache engineering literature highlights three practical failure
modes for large PSO caches: lock serialization across worker threads, excessive
RAM retention of warm caches (which can grow by gigabytes), and torn or
corrupted files across crashes and concurrent saves. `library.rs` addresses
each explicitly:

- **Striped load locking (`same_name()`):** D3D12 documents
  `ID3D12PipelineLibrary` as thread-safe except for concurrent loads of the
  same pipeline name. Instead of a global lock (which serialized workers in
  earlier profiling, leaving them waiting 87–96 of 110 samples), `same_name()`
  hashes the name across 64 mutexes (`VICTORY_LIBRARY_LOCK=1` restores a single
  lock for diagnostics).
- **Memory-mapped read/write and working-set trimming:** by default,
  `host-pipelines.bin` is mapped read-only via `MapViewOfFile` (`Blob::Mapped`,
  disabled with `VICTORY_LIBRARY_MAP=0`) and serialized through a writable
  mapping so saves avoid allocating a duplicate ~1+ GB heap buffer. After
  loads go quiet, `trim()` releases resident pages from the process working set
  with `VirtualUnlock` (`VICTORY_LIBRARY_TRIM=0` disables trimming).
- **Safe in-place replacement (`holds()`, `one_save()`, `host-pipelines.next`):**
  keeping the mapped file handle open (`holds()`, disabled with
  `VICTORY_LIBRARY_HANDLE=0`) prevents Windows from zero-filling evicted pages
  when a save renames a new library over the mapped path — a failure mode that
  otherwise hands zeroed PSO bytes to the driver and triggers
  `DXGI_ERROR_DRIVER_INTERNAL_ERROR`. Saves are serialized with `one_save()`,
  staged in `tmp-<pid>` files, and fall back to `host-pipelines.next` (promoted
  on the next startup) if the OS refuses an in-place rename.

---

## 3. Offline warm-up and prebuilding framed by AOT compilation sets

Graphics compiler literature distinguishes five sets when evaluating any
ahead-of-time compilation strategy:

1. $S_{\text{static}}$ — static source shaders;
2. $S_{\text{perm}}$ — all known compiled shader permutations shipped in the title;
3. $P_{\text{obs}}$ — the subset of complete pipelines empirically observed in
   prior runs;
4. $P_{\text{theor}}$ — the full Cartesian product of shader permutations and
   fixed states (combinatorially intractable);
5. $P_{\text{real}}$ — the exact set of pipelines that an arbitrary future play
   session will actually request (undecidable in general because it depends on
   runtime scene state and player choices).

The offline commands in `apps/victory/src/translate.rs` and
`apps/victory/src/prebuild.rs` map directly onto these sets:

| Command | Target set | Layer populated | Trade-off |
|---|---|---|---|
| `victory translate` (without `--cache`) | $S_{\text{perm}}$ (analysis only) | None | Reports translator / validator / host-PSO coverage without writing caches |
| `victory translate --cache` | $S_{\text{perm}}$ | `<cache>/shaders` | Translates every distinct DXBC/SGD5 container found in the supplied paths (trimmed to header size) into DXIL using the host D3D12 device's `host_max_wave()` and `view_heap_limit()` |
| `victory translate --cache --pipelines` | $S_{\text{perm}}$ + all reconstructable pack records | `<cache>/shaders` + `<cache>/pipelines/host-pipelines.bin` | Reconstructs every pipeline record from the title's shader pack (`victory_game::pipelines`) through `victory_gpu::pipeline::Prebuild` (prioritizing `early-pipelines.txt` first); high CPU time and driver-compiler RAM cost |
| `victory translate --cache --pipelines --used-pipelines` | $S_{\text{perm}}$ + $P_{\text{obs}}$ | `<cache>/shaders` + `<cache>/pipelines/host-pipelines.bin` | Filters pack prebuilding strictly to keys recorded in `early-pipelines.txt` from prior runs — the same $P_{\text{obs}}$ strategy used by Unreal Engine Bundled PSO Caches, Valve Fossilize `.foz` replay, and DirectX Advanced Shader Delivery (ASD) SODBs |

Why `--used-pipelines` works well in practice — and what it cannot eliminate:

- `Prebuild::make()` (`crates/victory-gpu/src/pipeline.rs`) runs through the
  exact same `HostPipeline::new()` path as runtime and preserves
  `early-pipelines.txt` (`library::keep_early_list()`), while skipping the same
  `left_out` shaders and saving batches periodically so interrupted runs keep
  progress. On the maintainer's test machine, a run after prebuilding compiled
  only 2 pipelines and loaded 2,770 from the library.
- Because $P_{\text{obs}} \subsetneq P_{\text{real}}$, visiting new areas,
  changing console profile/settings, or triggering shader containers generated
  or loaded only in memory still produces occasional cold PSO misses at
  runtime. Pack records whose fixed states cannot be named (`Unnamed::*` in
  `prebuild.rs`) or that use unsupported stages are also skipped and reported
  by cause (`VICTORY_PREBUILD_KEYS=<file>` dumps all pack keys and skip reasons
  without compiling).
- For wider ecosystem context: newer PC APIs mitigate monolithic PSO explosion
  via pipeline decoupling (`VK_EXT_graphics_pipeline_library`,
  `VK_EXT_shader_object`, `VK_KHR_pipeline_binary`, and D3D12 Partial Graphics
  Programs) or cloud-precompiled driver databases (DirectX ASD PSDBs). This
  fork currently targets standard D3D12 monolithic PSOs backed by a local
  `ID3D12PipelineLibrary1` and offline pack prebuilding.

---

## 4. Runtime scheduling, pathological compiles, and stutter taxonomy

### Background worker pool and frame-budget gating (`pipeline/make.rs`)

Because D3D12 drivers do not maintain an application-facing compile thread
pool, AMD's performance guidance recommends application-managed worker threads
with deduplication and bounded concurrency so background compilation neither
starves main/render threads nor exhausts RAM under LLVM-based backends (a
failure mode also documented in Valve Fossilize issue #312).

`crates/victory-gpu/src/pipeline/make.rs` implements this with:

- **Bounded worker count and priority:** `pipeline_worker_count()` spawns
  `(cores / 2).clamp(2, 12)` threads at `THREAD_PRIORITY_BELOW_NORMAL`, with
  two queue priorities (`REQUESTED` ahead of `EARLY`) so a draw needing a
  pipeline right now promotes ahead of speculative `early-pipelines.txt` loads.
- **Shared per-frame wait budget:** `VICTORY_PIPELINE_WAIT_MS` (default
  `500 ms`) is shared across all recording threads (`VICTORY_PIPELINE_WAIT_SHARED=0`
  reverts to per-thread counting, which multiplied waits across five recording
  threads to ~2.5 s/frame). Past the budget, draws whose pipeline is still
  compiling are dropped rather than stalling the title's 30 s GPU watchdog.
- **Loading-phase grace window for one-off passes:** during loading, one-off
  compute passes (such as geometry/model unpacking) are never re-dispatched if
  dropped. `make.rs` therefore extends per-frame waits up to `LOADING_FRAME`
  (`4 s`) for graphics and `LOADING_FRAME_COMPUTE` (`12 s`) for compute,
  bounded across the whole run by `VICTORY_PIPELINE_GATE_S` (default `120 s`;
  `0` disables).
- **Slow-compile cutoff (`slow()`):** once a pipeline has been compiling for
  more than `10 s` (compute) or `3 s` (graphics), frames stop waiting on it.

### Pathological driver compiles (`pipeline/left_out.rs`)

Middle-end optimization passes and selective omission directly control worst-case
driver compiler time and memory:

- `RAY_TRACING_RESOLVES` (`348` compute container hashes) shade ray-tracing
  hits that do not exist while the BVH structure builder is off. On the
  RX 7800 XT, each takes the AMD driver compiler **5 to 9 s** on one core and
  all bind on the first world frame; `make::left_out()` skips them unless
  `VICTORY_RT_RESOLVE=1` or `VICTORY_BVH_BUILDER=1` is set.
- `ACTIVATE_RAYS` (`39f0e9aeb156a61c`) contains 66,501 console instructions and
  9,948 branches. With `emit::shader_opt` clean-up passes enabled, the AMD
  compiler finishes once in **85 s / 1.3 GB**; without clean-up passes, it grew
  to **47 GB after 8 minutes** without finishing on the RX 7800 XT. It can be
  omitted with `VICTORY_ACTIVATE_RAYS=0`, and arbitrary extra compute hashes can
  be excluded with `VICTORY_LEFT_OUT=<hash>,...`.

### Separating compilation stutter from remaining runtime stalls

PC frame-pacing literature separates **pipeline compilation stutter** (one-time
synchronous stalls on first encounter that disappear with warm caches) from
recurring runtime stalls. Once `--cache --pipelines --used-pipelines` warms
both layers, remaining frame-time costs in this fork come from distinct
mechanisms:

1. **Loss of coherent unified memory (UMA) and hardware LZ/BCPack:** on Xbox
   Series X|S, CPU and GPU share coherent GDDR6 with hardware decompression; on
   PC, dirty-page tracking (`watch`), shadow-buffer uploads, and shader-written
   copy-backs cross PCIe. This accounts for roughly two-thirds of frame time in
   steady open-world views (`README.md` reports `135 ms preparing / 94 ms GPU /
   78 ms copy-back`, dropping to `63 / 51 / 36 ms` with upstream speed
   defaults).
2. **VRAM allocation and residency pressure:** desktop GPUs with 8 GB VRAM
   (or 16 GB cards near saturation in the open world, ~13.3–14.0 GB of a
   14.7 GB budget) experience deferred or skipped texture uploads and WDDM
   `VidMm` paging — a memory-budget issue, not shader compilation.
3. **Async asset streaming vs. title clock:** geometry arriving 10–25 s after a
   cutscene starts reflects host upload/decompression throughput lagging behind
   the console's hardware NVMe/BCPack schedule rather than missing PSOs.

---

## 5. Host-GPU hardware divergences relevant to translation

Even after lifting GFX10.3 to SM 6.6 DXIL, several hardware differences between
the console's RDNA 2 GPU and desktop GPUs directly affect shader translation
and cache compatibility:

- **Wavefront width (`Wave32` vs. `Wave64`, `host_max_wave()`):** RDNA 2/3
  supports both native Wave32 (1 cycle per SIMD32) and Wave64 (2 cycles),
  whereas NVIDIA GPUs execute 32-lane warps exclusively. Console Wave64 compute
  shaders must be lowered differently when `WaveLaneCountMax == 32`
  (`crates/victory-gpu/src/host.rs` calls `set_host_max_wave()`). That is why
  `host_max_wave()` is part of `shader_key()`.
- **Pixel 2×2 quads and helper lanes:** pixel shaders execute in 2×2 quads so
  derivatives (`ddx`/`ddy`) and mip selection work across triangle edges where
  neighboring lanes are inactive helper pixels. On AMD hardware, wave-level
  cross-lane reads return `0` from helper lanes; translating pixel-shader quad
  swizzles and DPP quad permutations to explicit DXIL quad reads (commit
  `e086bda7`, `crates/victory-shader/src/emit/lower.rs`) fixes hand-computed
  derivatives along triangle edges.
- **Shader-visible descriptor heap limits (`view_heap_limit()`):** D3D12
  resource binding tiers 1–2 cap shader-visible CBV/SRV/UAV heaps at
  `PORTABLE_VIEWS = 1_000_000` (NVIDIA), whereas AMD tier 3 heaps accept
  `UNCAPPED_VIEWS = 1 << 21` (`2,097,152`). `probe_view_limit()`
  (`crates/victory-gpu/src/descriptor/host.rs`) probes this once and clamps
  out-of-bounds view indices to the final null slot on every host, entering
  `shader_key()` so translations built for one cap are never served on the
  other.
- **VGPR granularity and middle-end optimization:** in RDNA 2, each SIMD32 has
  1,024 physical 32-bit VGPRs shared across up to 16 wavefronts (allocated in
  granules of 16 VGPRs in Wave32 or 8 in Wave64) plus 128 KiB LDS per WGP.
  Running `emit::shader_opt` clean-up passes reduces IR bloat before the host
  driver's register allocator runs — critical both for occupancy and for
  keeping driver compile time/RAM bounded (as seen on `ACTIVATE_RAYS`).

---

## 6. Verifying both cache layers in logs

Check the two layers separately in a run's `INFO` log:

### Layer 1: DXIL translation cache (`<cache>/shaders`)

| Log message (`INFO`) | Fields | Source & how to read it |
|---|---|---|
| `shaders from the cache and translated so far` (emitted on power-of-two presented frames `>= 256`: `256`, `512`, `1024`, ...) | `hits`, `misses` | Cumulative counts since startup (`crates/victory-gpu/src/queue.rs`, `shader_cache::counts()`). After `translate --cache`, `hits` should dominate and `misses` should stay small (only shaders not present in the warmed files or changed config) |
| `host wave lanes` | `min`, `max` | `crates/victory-gpu/src/host.rs`; `max` enters `shader_key()` as `host_max_wave()` |
| `host view heaps hold <views> views; view indices are clamped to the null slot at the end from the first shader on` | `views` | `crates/victory-gpu/src/descriptor/host.rs`; `views - 1` enters `shader_key()` as `view_heap_limit()` |
| `shader translated` | `stage`, `hash`, `user_data`, `samplers` | `crates/victory-gpu/src/pipeline.rs`. **Does not indicate a cache miss:** logged once per distinct shader when `host_shader()` succeeds, whether served from disk (`Hit`) or translated on demand (`Stored`) |

Before launching the title, running `victory translate --cache ...` a second
time directly confirms reuse in its summary line (`0 translated`, remainder
under `already cached` and `cached refusals`).

### Layer 2: Host PSO library (`<cache>/pipelines`)

| Log message (`INFO`) | Fields | Source & how to read it |
|---|---|---|
| `host pipeline library loaded` | `bytes`, `mapped` | `crates/victory-gpu/src/pipeline/library.rs`. Confirms the driver accepted `host-pipelines.bin` (`mapped=true` when `MapViewOfFile` succeeded). First runs log `no host pipeline library yet; starting an empty one`; driver/GPU changes or corruption log `host pipeline library rejected; starting an empty one` (`error`) |
| `title pipelines to start early` | `pipelines` | Count of prior-run pipeline keys loaded from `early-pipelines.txt` |
| `host pipelines all made` (after a batch with peak waiting `>= 10`) | `made`, `loaded`, `started_early` | `crates/victory-gpu/src/pipeline/make.rs`. Cumulative run counters: `made` = newly driver-compiled PSOs (library miss); `loaded` = PSOs loaded from `host-pipelines.bin` (library hit); `started_early` = title pipelines queued at creation from `early-pipelines.txt` |
| `host pipeline library saved` (at most every `20 s` when changed, plus on shutdown) | `bytes`, `early`, `made`, `loaded`, `started_early` | `crates/victory-gpu/src/pipeline/library.rs`. Same cumulative counters plus `bytes` serialized to `host-pipelines.bin` (`0` if only `early-pipelines.txt` changed) and total `early` entries |
| `host pipelines left out so far` (at count `1` and powers of two) | `count`, `shaders`, `why` | `crates/victory-gpu/src/pipeline/make.rs`. Confirms `left_out.rs` / `VICTORY_LEFT_OUT` exclusions |

After a `--cache --pipelines --used-pipelines` warm-up on the same GPU and
driver, a run should log `host pipeline library loaded`, high `loaded` and
`started_early`, and low `made`.

---

## Primary references

1. Grossman, M., & Andrews, J. (2021). *The Xbox Series X System Architecture*,
   IEEE Micro 41(2), 22–28; and Hot Chips 32 presentation (Aug. 2020):
   https://hc32.hotchips.org/assets/program/conference/day1/HotChips2020_GPU_Microsoft_Jeff_Andrews_v2.pdf
2. Microsoft Learn — *Xbox GDK Samples: HlslCompile* (`DxCompiler_xs.dll`,
   `aco_xbox.dll`, `AcoOfflineSerializePipelineStateXEx`,
   `DeserializeGraphicsPipelineStateX`):
   https://learn.microsoft.com/en-us/samples/microsoft/xbox-gdk-samples/hlslcompile/
3. Microsoft Learn — *Introduction to the Microsoft Game Development Kit
   (GDK)* (Game OS versioning, `D3D12.x`, `XMemAlloc`):
   https://learn.microsoft.com/en-us/gaming/gdk/docs/gdk-dev/intro/introduction?view=gdk-2604
4. Microsoft Learn — *Local Storage on Xbox Consoles* (`G:\`, `T:\`, PLS,
   XR-133):
   https://learn.microsoft.com/en-us/gaming/gdk/docs/features/console/storage/local-storage?view=gdk-2610
5. Microsoft Learn — *Windows Display Driver Model (WDDM) Architecture*:
   https://learn.microsoft.com/en-us/windows-hardware/drivers/display/windows-vista-and-later-display-driver-model-architecture
6. Microsoft Learn — *Direct3D 12 API Reference*
   (`ID3D12Device1::CreatePipelineLibrary`,
   `D3D12_GRAPHICS_PIPELINE_STATE_DESC`, Root Signatures):
   https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device1-createpipelinelibrary
7. Microsoft DirectX-Specs — *D3D12 Shader Cache APIs* and *D3D Shader Cache
   Registration (D3DSCR) API*:
   https://microsoft.github.io/DirectX-Specs/d3d/ShaderCache.html and
   https://microsoft.github.io/DirectX-Specs/d3d/ShaderCacheRegistrationAPI.html
8. Microsoft DirectX Developer Blog — *DirectX Advanced Shader Delivery* and
   *Partial Graphics Programs*:
   https://devblogs.microsoft.com/directx/introducing-advanced-shader-delivery/
   and https://devblogs.microsoft.com/directx/partial-graphics-programs/
9. AMD — *"RDNA 2" Instruction Set Architecture: Reference Guide* (Doc. ID
   70648):
   https://docs.amd.com/v/u/en-US/rdna2-shader-instruction-set-architecture
10. AMD GPUOpen — *RDNA Performance Guide* and *Occupancy Explained*:
    https://gpuopen.com/learn/rdna-performance-guide/ and
    https://gpuopen.com/learn/occupancy-explained/
11. LLVM Project — *User Guide for AMDGPU Backend* (`gfx1030` VGPR allocation
    granularity and occupancy):
    https://llvm.org/docs/AMDGPUUsage.html
12. Mesa 3D — *RADV Driver Documentation* (NIR / ACO compiler architecture):
    https://docs.mesa3d.org/drivers/radv.html
13. Khronos Group — *Reducing Draw Time Hitching with
    `VK_EXT_graphics_pipeline_library`* and *Bringing Explicit Pipeline Caching
    Control to Vulkan (`VK_KHR_pipeline_binary`)*:
    https://www.khronos.org/blog/reducing-draw-time-hitching-with-vk-ext-graphics-pipeline-library
    and
    https://www.khronos.org/blog/bringing-explicit-pipeline-caching-control-to-vulkan
14. Epic Games — *Game Engines and Shader Stuttering: Unreal Engine's Solution
    to the Problem* and *PSO Precaching*:
    https://www.unrealengine.com/tech-blog/game-engines-and-shader-stuttering-unreal-engines-solution-to-the-problem
    and
    https://dev.epicgames.com/documentation/unreal-engine/pso-precaching-for-unreal-engine
15. ValveSoftware/Fossilize — *README* and parallel replay memory-usage report
    (#312):
    https://github.com/ValveSoftware/Fossilize/blob/master/README.md and
    https://github.com/ValveSoftware/Fossilize/issues/312
16. Roblox Engineering — *Creating a Robust Pipeline Cache with Vulkan*
    (header validation and atomic persistence):
    https://about.roblox.com/newsroom/2021/02/creating-a-robust-pipeline-cache-with-vulkan
17. Pettineo, M. (TheRealMJP) — *Shader Permutations Part 1 & Part 2*:
    https://therealmjp.github.io/posts/shader-permutations-part1/ and
    https://therealmjp.github.io/posts/shader-permutations-part2/
