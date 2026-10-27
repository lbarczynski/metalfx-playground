# MetalFX & Modern C++ Learning Curriculum

This curriculum is designed to build mastery in **Modern C++ (C++20)**, **Apple Metal**, and **MetalFX** upscaling through structured **30–45 minute daily micro-sessions**. 

Every session has a sharp, focused scope: you build and verify one concrete milestone, finish with a working state, and conclude your session with a clean git commit.

Each session is paired with a technical guide in [`materials/`](./materials/) containing conceptual models, Kotlin-to-C++ architectural bridges, mathematical foundations, and step-by-step checklists.

---

## AI Assistant Skills (Workflow Helpers)

Four workspace skills in `.agents/skills/` assist your daily workflow without writing the code for you:

| Command / Trigger | Skill | Purpose |
|---|---|---|
| `Start next task` / `Start task 02A` | [start-task](./.agents/skills/start-task/SKILL.md) | Identifies the next uncompleted task, extracts the core math & C++ mental model, and guides Step 1. |
| `Review my change` / `Check my code` | [review-change](./.agents/skills/review-change/SKILL.md) | Audits your `git diff` for struct alignments, Metal invariants, RAII lifetimes, and DoD compliance. |
| `Debug Metal` / `Validation error` | [debug-metal](./.agents/skills/debug-metal/SKILL.md) | Decodes Metal validation logs and diagnoses visual glitches (black screens, leaks, ghosting). |
| `Complete task` / `Task done` | [complete-task](./.agents/skills/complete-task/SKILL.md) | Marks task `[x]` in this plan, generates git commit metadata, and previews tomorrow's session. |

---

## Progress Tracker (33 Focused Sessions)

### Stage #0: Professional Engineering Infrastructure (Prerequisite)
- [x] **TASK-00: Tooling, Presets, Formatting, Tests & CI/CD Pipeline**
  - *Infrastructure*: CMake 3.25+ with Ninja generator, `CMakePresets.json` (Debug, Release, CI, Xcode).
  - *Automated Fetch*: Catch2 v3 test framework and Apple `metal-cpp` CDN ingestion.
  - *Quality*: Strict `.clang-format` (C++20), `.clang-tidy`, and `.githooks/pre-commit`.
  - *DoD*: `cmake --preset macos-debug` configures cleanly and build files generate without errors.

---

### Phase 1: Foundations & Modern C++ (Sessions 01–09)

- [ ] **[TASK-01: CMake, metal-cpp Integration & Device Discovery](./materials/01-project-setup-and-metal-cpp.md)** *(35 min)*
  - *C++*: Native compilation vs JVM bytecode, header/source separation, zero-overhead abstraction.
  - *Metal*: Querying Apple Silicon GPU capabilities (`MTL::CreateSystemDefaultDevice`, UMA detection).
  - *DoD*: Executable prints GPU device name and verified Apple Silicon feature set.
  - *Commit*: `feat(infra): setup cmake and query metal device`

- [ ] **[TASK-02A: Platform Window Bridge](./materials/02a-platform-window-bridge.md)** *(40 min)*
  - *C++*: Deterministic object lifetimes without GC. Objective-C++ bridging (`.mm` to `.hpp`).
  - *Platform*: Native Cocoa window lifecycle (`NSApplication`, `NSWindow`, event dispatch loop).
  - *DoD*: Native macOS window opens titled "MetalFX Playground" and cleanly handles close/exit.
  - *Commit*: `feat(platform): implement native cocoa window bridge`

- [ ] **[TASK-02B: Metal Render Loop & Clear Pass](./materials/02b-metal-render-loop.md)** *(45 min)*
  - *C++*: `NS::AutoreleasePool` RAII wrapper to prevent per-frame autorelease memory growth.
  - *Metal*: `CAMetalLayer` / `MTK::View`, `MTL::CommandQueue`, `MTL::CommandBuffer`, `MTL::RenderPassDescriptor`.
  - *DoD*: Window opens and animates an oscillating clear color cycle via Metal render pass.
  - *Commit*: `feat(renderer): implement metal render loop with clear pass`

- [ ] **[TASK-03A: MSL Shaders & Pipeline State Compilation](./materials/03a-msl-shaders-and-pipeline-state.md)** *(40 min)*
  - *C++*: Runtime compilation and error inspection (`NS::Error`), immutable state objects.
  - *Metal*: MSL shader syntax (`vertex`, `fragment`, qualifiers `[[stage_in]]`), `MTL::RenderPipelineState`.
  - *DoD*: Shaders compile into a valid pipeline state without compiler warnings or runtime errors.
  - *Commit*: `feat(shader): compile msl vertex and fragment pipeline state`

- [ ] **[TASK-03B: Vertex Buffer Layout & Triangle Draw Call](./materials/03b-vertex-buffers-and-draw-call.md)** *(45 min)*
  - *C++*: Contiguous memory layout (`struct Vertex`), memory offsets vs heap object pointers.
  - *Metal*: `MTL::Buffer`, vertex attribute layout, barycentric rasterization, `drawPrimitives`.
  - *DoD*: A vibrant colored triangle renders to the screen using custom vertex and fragment shaders.
  - *Commit*: `feat(renderer): render hello triangle with custom msl shaders`

- [ ] **[TASK-04A: C++ SIMD Math & Hardware Alignment](./materials/04a-simd-math-and-alignment.md)** *(40 min)*
  - *C++*: Vector/matrix math with `<simd/simd.h>`, `alignas(16)`, `static_assert` layout verification, Catch2 unit tests.
  - *Graphics*: Homogeneous coordinates and 2D transformation matrices.
  - *DoD*: CTest runs Catch2 suite verifying matrix multiplication and struct alignments.
  - *Commit*: `test(math): implement simd matrix math with catch2 verification`

- [ ] **[TASK-04B: Dynamic Uniforms & Ring Buffer Concurrency](./materials/04b-dynamic-uniforms-ring-buffer.md)** *(45 min)*
  - *C++*: Multi-buffering concurrency without GC pauses: triple-buffering via `dispatch_semaphore_t`.
  - *Metal*: 256-byte constant buffer dynamic offset alignment rule, `setVertexBuffer:offset:atIndex:`.
  - *DoD*: Shape rotates smoothly on screen with dynamic CPU-to-GPU uniform buffer updates.
  - *Commit*: `feat(uniforms): implement dynamic triple-buffered uniform ring buffer`

- [ ] **[TASK-05A: Procedural Texture Generation & Memory Stride](./materials/05a-procedural-texture-and-stride.md)** *(35 min)*
  - *C++*: Raw image byte manipulation (`std::vector<uint32_t>`), row pitch (`bytesPerRow`).
  - *Metal*: `MTL::TextureDescriptor`, `MTL::Texture`, uploading host memory via `replaceRegion`.
  - *DoD*: Procedural checkerboard texture allocated and populated in GPU memory.
  - *Commit*: `feat(texture): generate procedural texture and upload to gpu`

- [ ] **[TASK-05B: Sampler States & Textured Quad Rendering](./materials/05b-sampler-states-and-textured-quad.md)** *(45 min)*
  - *C++*: Type-safe enum class configurations, UV coordinate mapping $[0, 1]$.
  - *Metal*: `MTL::SamplerDescriptor`, Nearest vs Linear filtering, MSL `texture2d::sample`.
  - *DoD*: Textured quad rendered on screen with interactive switching between Nearest and Linear sampling.
  - *Commit*: `feat(renderer): implement sampler states and textured quad rendering`

---

### Phase 2: 3D Graphics & Compute (Sessions 10–17)

- [ ] **[TASK-06A: 3D Perspective Projection & Index Buffers](./materials/06a-3d-projection-and-index-buffers.md)** *(40 min)*
  - *C++*: Fixed-width types (`uint16_t`) for index buffers, frustum matrix math.
  - *Graphics*: Coordinate pipelines (Model $\to$ World $\to$ View $\to$ Clip Space), Metal $[0, 1]$ clip bounds.
  - *DoD*: Perspective projection math verified in Catch2; indexed 3D cube vertex buffer initialized.
  - *Commit*: `feat(math): implement perspective projection and indexed cube geometry`

- [ ] **[TASK-06B: Depth Buffer & Spinning 3D Cube](./materials/06b-depth-buffer-and-3d-cube.md)** *(45 min)*
  - *C++*: RAII resource encapsulation for depth textures.
  - *Metal*: `MTL::PixelFormatDepth32Float`, `MTL::DepthStencilDescriptor`, back-face culling.
  - *DoD*: A rotating 3D colored cube renders with correct depth occlusion.
  - *Commit*: `feat(renderer): implement depth testing and render spinning 3d cube`

- [ ] **[TASK-07: Metal Compute Pipeline & CPU Readback](./materials/07-metal-compute-pipeline.md)** *(45 min)*
  - *C++*: Shared struct headers between C++ and MSL (`#ifdef __METAL_VERSION__`).
  - *Metal*: `MTL::ComputePipelineState`, `MTL::ComputeCommandEncoder`, threadgroup geometry (`MTL::Size`).
  - *DoD*: Compute kernel executes on GPU with verified numeric readback on CPU.
  - *Commit*: `feat(compute): implement metal compute kernel with cpu readback`

- [ ] **[TASK-08A: Low-Resolution Offscreen Render Target](./materials/08a-low-res-offscreen-target.md)** *(40 min)*
  - *C++*: Decoupling internal render resolution from display dimensions.
  - *Metal*: Texture usage `MTL::TextureUsageRenderTarget | MTL::TextureUsageShaderRead`, `StoreActionStore`.
  - *DoD*: 3D scene renders into an offscreen low-res texture (e.g. 540p) in GPU memory.
  - *Commit*: `feat(offscreen): create low-resolution offscreen render target`

- [ ] **[TASK-08B: Fullscreen Quad Blit Pass](./materials/08b-fullscreen-blit-pass.md)** *(40 min)*
  - *C++*: Multi-pass command buffer sequencing and encoder boundaries.
  - *Metal*: Fullscreen procedural triangle pass, presenting low-res texture scaled to high-res window.
  - *DoD*: 540p scene is scaled up to the full-size native window via blit presentation.
  - *Commit*: `feat(renderer): implement fullscreen blit pass for offscreen target`

- [ ] **[TASK-09A: Orbit Camera Controller & Frame Delta Time](./materials/09a-orbit-camera-controller.md)** *(40 min)*
  - *C++*: High-resolution timers (`std::chrono`), frame rate delta time ($\Delta t$).
  - *Graphics*: Spherical orbit camera coordinates (yaw, pitch, radius), Look-At view matrix.
  - *DoD*: Interactive camera navigates around the 3D scene with mouse/keyboard input.
  - *Commit*: `feat(camera): implement interactive orbit camera controller`

- [ ] **[TASK-09B: Blinn-Phong Shading in MSL](./materials/09b-blinn-phong-shading-msl.md)** *(45 min)*
  - *C++*: Struct alignment for directional light parameters.
  - *Metal*: Surface normals, Lambertian diffuse, halfway vector specular reflection in MSL.
  - *DoD*: Scene geometry illuminated with realistic directional lighting and specular highlights.
  - *Commit*: `feat(shader): implement blinn-phong lighting model in msl`

- [ ] **[TASK-10: GPU Profiling & Metal Capture Infrastructure](./materials/10-profiling-and-gpu-capture.md)** *(35 min)*
  - *C++*: Moving average frame time metrics, formatted diagnostics.
  - *Metal*: `MTL::CaptureManager`, GPU timestamp queries (`gpuStartTime`, `gpuEndTime`).
  - *DoD*: Console logs live microsecond GPU frame time; keypress triggers programmatic `.gputrace`.
  - *Commit*: `feat(profile): integrate metal gpu profiling and programmatic capture`

---

### Phase 3: MetalFX Spatial Upscaling (Sessions 18–22)

- [ ] **[TASK-11: MetalFX Spatial Support & Device Capabilities](./materials/11-metalfx-spatial-foundations.md)** *(30 min)*
  - *C++*: Feature capability detection, error handling patterns.
  - *MetalFX*: Linking `MetalFX.framework`, `MTLFX::SpatialScalerDescriptor::supportsDevice`.
  - *DoD*: Application queries and prints hardware support for MetalFX Spatial and compatible formats.
  - *Commit*: `feat(metalfx): query hardware support for metalfx spatial scaler`

- [ ] **[TASK-12: Configuring the Spatial Scaler Descriptor](./materials/12-metalfx-spatial-descriptor.md)** *(40 min)*
  - *C++*: Dimension separation: input size ($W_{\text{in}}, H_{\text{in}}$) vs presentation size ($W_{\text{out}}, H_{\text{out}}$).
  - *MetalFX*: `MTLFX::SpatialScalerDescriptor`, color formats, `newSpatialScaler(device)`.
  - *DoD*: `MTLFX::SpatialScaler` successfully instantiated for 540p input to target window size.
  - *Commit*: `feat(metalfx): configure spatial scaler descriptor and instantiate scaler`

- [ ] **[TASK-13: Encoding the Spatial Scaler Pass](./materials/13-encoding-spatial-scaler.md)** *(45 min)*
  - *C++*: Command buffer sequencing, tile cache memory hazard boundaries (`endEncoding()`).
  - *MetalFX*: Texture binding, `scaler->outputTextureUsage()`, encoding pass via `encodeToCommandBuffer`.
  - *DoD*: Low-resolution 3D scene is upscaled in real-time via MetalFX Spatial to the display.
  - *Commit*: `feat(metalfx): encode spatial upscaler pass into command buffer`

- [ ] **[TASK-14: Dynamic Window Resizing & Scaling Presets](./materials/14-dynamic-resizing-and-quality-modes.md)** *(45 min)*
  - *C++*: Safe resource reallocation upon window resize without memory leaks.
  - *MetalFX*: Quality presets: Quality (67%), Balanced (58%), Performance (50%).
  - *DoD*: Window dynamically resizes and presets toggle seamlessly without crashes or leaks.
  - *Commit*: `feat(metalfx): implement dynamic window resizing and scaling presets`

- [ ] **[TASK-15: Color Spaces (sRGB vs Linear) & HDR Precision](./materials/15-color-spaces-and-hdr.md)** *(40 min)*
  - *C++*: Precision considerations: FP16 (`half`) vs FP32 vs UNORM.
  - *MetalFX*: Color processing modes (`Perceptual`, `Linear`, `HDR`), `MTL::PixelFormatRGBA16Float`.
  - *DoD*: Clean spatial upscaling without color crushing, dark edge ringing, or gamma banding.
  - *Commit*: `feat(color): handle linear and srgb color spaces in metalfx pipeline`

---

### Phase 4: MetalFX Temporal Upscaling & Capstone (Sessions 23–33)

- [ ] **[TASK-16: Temporal Scaler Architecture & Requirements](./materials/16-metalfx-temporal-foundations.md)** *(40 min)*
  - *C++*: Temporal history state management.
  - *MetalFX*: Required attachments (Color, Depth, Motion, History), `MTLFX::TemporalScalerDescriptor`.
  - *DoD*: Temporal scaler hardware support queried and `MTLFX::TemporalScaler` initialized.
  - *Commit*: `feat(metalfx): configure temporal scaler descriptor and verify inputs`

- [ ] **[TASK-17A: Halton(2,3) Jitter Generator & Unit Tests](./materials/17a-halton-generator-and-tests.md)** *(35 min)*
  - *C++*: Van der Corput radical inverse, `constexpr` sequence generation, Catch2 test suite.
  - *Graphics*: Low-discrepancy sampling vs aliasing, phase distribution across $[-0.5, 0.5]$.
  - *DoD*: Catch2 tests verify 16-sample Halton(2,3) generation within unit pixel bounds.
  - *Commit*: `feat(math): implement halton23 subpixel jitter generator with unit tests`

- [ ] **[TASK-17B: Subpixel Camera Projection Jittering](./materials/17b-subpixel-projection-jittering.md)** *(40 min)*
  - *C++*: Applying subpixel camera phase shifts to the projection matrix.
  - *Metal*: Modifying projection elements $P[2][0]$ and $P[2][1]$ per frame in pixel units.
  - *DoD*: Camera projection jitters across consecutive frames; low-res render shows subpixel phase shifts.
  - *Commit*: `feat(renderer): apply subpixel jitter to camera projection matrix`

- [ ] **[TASK-18A: Velocity Buffer & Multiple Render Targets (MRT)](./materials/18a-velocity-buffer-mrt-setup.md)** *(45 min)*
  - *C++*: Caching previous frame's unjittered MVP matrix per object.
  - *Metal*: Multiple Render Targets (MRT): Color (RT0) + Screen-Space Velocity (RT1: `RG16Float`).
  - *DoD*: Pipeline configured with 2 color attachments; scene writes to color and motion buffers simultaneously.
  - *Commit*: `feat(pipeline): configure mrt render pass for color and motion targets`

- [ ] **[TASK-18B: Analytical Motion Vector Calculation in MSL](./materials/18b-msl-velocity-calculation.md)** *(45 min)*
  - *C++*: Dual unjittered transformation feeding vertex shader.
  - *Metal*: MetalFX velocity convention: motion vector points **from current back to previous** ($\mathbf{p}_{\text{prev}} - \mathbf{p}_{\text{curr}}$).
  - *DoD*: Moving objects generate accurate color-coded velocity vectors in screen space.
  - *Commit*: `feat(shader): implement analytical motion vector calculation in msl`

- [ ] **[TASK-19: Reversed-Z Depth Precision for Temporal](./materials/19-depth-precision-for-temporal.md)** *(40 min)*
  - *C++*: Floating-point precision distribution across depth ranges.
  - *Metal*: Reversed-Z setup ($n \to 1.0, f \to 0.0$), `Depth32Float`, `GreaterEqual`, `setDepthReversed(true)`.
  - *DoD*: High-precision depth buffer configured and verified for temporal reconstruction.
  - *Commit*: `feat(depth): configure reversed-z depth buffer for temporal reconstruction`

- [ ] **[TASK-20A: History & Jitter Attachment Configuration](./materials/20a-temporal-attachments-and-jitter.md)** *(40 min)*
  - *C++*: Camera cut detection logic ($|\Delta \mathbf{P}| > \text{threshold}$) triggering `setReset(true)`.
  - *MetalFX*: Passing jitter offsets in input pixel units ($j_x, j_y$) via `setJitterOffsetX/Y`.
  - *DoD*: Color, depth, motion, output textures and jitter parameters bound to scaler.
  - *Commit*: `feat(metalfx): wire jitter offsets and history reset logic to temporal scaler`

- [ ] **[TASK-20B: Encoding the Temporal Scaler Pass](./materials/20b-encoding-temporal-scaler-pass.md)** *(45 min)*
  - *C++*: Multi-pass frame synchronization (Scene MRT $\to$ Barrier $\to$ Temporal Scaler $\to$ Present).
  - *MetalFX*: Active temporal reconstruction accumulating subpixel details across frames.
  - *DoD*: MetalFX Temporal Scaler upscales 540p scene to high-res with temporal stability.
  - *Commit*: `feat(metalfx): encode temporal upscaler pass with active accumulation`

- [ ] **[TASK-21: Temporal Scaler Tuning & Artifact Mitigation](./materials/21-artifact-mitigation-tuning.md)** *(40 min)*
  - *C++*: Dynamic resolution scaling via `setInputContentWidth/Height`.
  - *MetalFX*: Tuning `setMotionVectorScaleX/Y`, handling disocclusion, ghosting mitigation on camera cuts.
  - *DoD*: Stable temporal reconstruction without ghosting trails or edge shimmering during motion.
  - *Commit*: `feat(metalfx): tune motion vector scales and temporal stability parameters`

- [ ] **[TASK-22A: Interactive Multi-Mode State Machine](./materials/22a-multi-mode-state-machine.md)** *(40 min)*
  - *C++*: State Pattern in modern C++, polymorphic mode dispatcher.
  - *Architecture*: 5 Switchable Modes: 1. Native Low-Res, 2. Bilinear, 3. Spatial, 4. Temporal, 5. Native High-Res.
  - *DoD*: Application toggles between all 5 rendering pipelines smoothly via hotkeys.
  - *Commit*: `feat(app): implement interactive upscaling mode state machine`

- [ ] **[TASK-22B: Split-Screen Comparison Playground & GPU Metrics](./materials/22b-split-screen-and-live-profiling.md)** *(45 min)*
  - *C++*: Polished clean code adhering to `-Wall -Wextra -Wpedantic` and zero warnings.
  - *Metal / MetalFX*: Scissor rect (`setScissorRect`) split-screen divider with live GPU microsecond timings.
  - *DoD*: Interactive split-screen playground showing side-by-side visual and performance comparison.
  - *Commit*: `feat(capstone): complete split-screen comparison playground with gpu metrics`
