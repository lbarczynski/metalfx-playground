# MetalFX & Modern C++ Learning Curriculum

This curriculum is designed to build mastery in **Modern C++ (C++20)** and **Apple Metal / MetalFX** through daily 30–60 minute tasks. 

Every task is paired with a concise, high-density technical guide in the [`materials/`](./materials/) directory covering the required mathematical formulas, API blueprints, and implementation steps.

---

## AI Assistant Skills (Workflow Helpers)

Four specialized workspace skills in `.agents/skills/` are available to assist your daily workflow without writing the code for you:

| Command / Trigger | Skill | Purpose |
|---|---|---|
| `Start next task` / `Start task X` | [start-task](./.agents/skills/start-task/SKILL.md) | Locates the next uncompleted task, extracts the core math & concept, and sets up Step 1. |
| `Review my change` / `Check my code` | [review-change](./.agents/skills/review-change/SKILL.md) | Audits your `git diff` for struct alignments, Metal invariants, and DoD compliance. |
| `Debug Metal` / `Validation error` | [debug-metal](./.agents/skills/debug-metal/SKILL.md) | Decodes cryptic Metal validation logs and diagnoses visual glitches (black screens, leaks). |
| `Complete task` / `Task done` | [complete-task](./.agents/skills/complete-task/SKILL.md) | Checks off `[x]` in this plan, generates git commit metadata, and previews tomorrow. |

---

## Progress Tracker

### Stage #0: Professional Engineering Infrastructure (Prerequisite)
- [x] **TASK-00: Tooling, Presets, Formatting, Tests & CI/CD Pipeline**
  - *Dependencies*: Microsoft `vcpkg.json` manifest + automated Apple `metal-cpp` ingestion.
  - *Build*: `CMakePresets.json` (Ninja generator, Debug, Release, ASan/UBSan, CTest, Xcode).
  - *Quality*: Strict `.clang-format` (C++20), `.clang-tidy`, and `.githooks/pre-commit`.
  - *Testing*: Catch2 v3 unit test suite with CTest discovery (`test_math`, `test_metal_device`).
  - *Cloud CI*: `.github/workflows/ci.yml` running on Apple Silicon macOS-14 cloud runners.
  - *DoD*: CMake presets configure cleanly with Ninja, tests pass under CTest, and pre-commit checks succeed.

### Phase 1: Foundations & Modern C++ (Days 1–5)
- [ ] **[TASK-01: CMake, metal-cpp Integration & Device Discovery](./materials/01-project-setup-and-metal-cpp.md)** *(45 min)*
  - *C++*: Modern CMake target-based architecture, C++20 standard, header-only wrappers, compiler flags (`-Wall -Wextra -Wpedantic`).
  - *Metal*: Linking Metal, Foundation, QuartzCore, MetalKit; initializing `MTL::Device` and querying Apple Silicon GPU capabilities.
  - *DoD*: CMake project builds from CLI; executable prints GPU device name and feature set.

- [ ] **[TASK-02: Native Windowing & The Metal Render Loop](./materials/02-native-window-and-metal-loop.md)** *(60 min)*
  - *C++*: Native Cocoa lifecycle in C++ using `AppKit` wrappers (`NS::Application`, `NS::Window`), `NS::AutoreleasePool` RAII wrappers.
  - *Metal*: `MTK::View`, `MTK::ViewDelegate`, `MTL::CommandQueue`, `MTL::CommandBuffer`, `MTL::RenderPassDescriptor`.
  - *DoD*: Native window opens displaying a solid animated clear color cycle via Metal render pass.

- [ ] **[TASK-03: The Hello Triangle & Metal Shading Language](./materials/03-hello-triangle-pipeline.md)** *(60 min)*
  - *C++*: Interleaved vertex memory layout, `std::vector`, pointer arithmetic, type-safe vertex descriptors.
  - *Metal*: `MTL::RenderPipelineDescriptor`, `MTL::RenderPipelineState`, MSL vertex & fragment functions, `MTL::Buffer`.
  - *DoD*: A vibrant colored triangle rendered to the window using custom vertex and fragment shaders.

- [ ] **[TASK-04: Transformations, Dynamic Uniforms & Math Primitives](./materials/04-dynamic-uniforms-and-math.md)** *(60 min)*
  - *C++*: Custom vector/matrix types in C++20 with `<simd/simd.h>`, 16-byte MSL alignment, triple-buffering / ring buffer.
  - *Metal*: Constant buffer binding (`setVertexBuffer`), dynamic uniforms per frame.
  - *DoD*: A 2D geometric shape rotating smoothly on screen with dynamic CPU-to-GPU uniform updates.

- [ ] **[TASK-05: Textures, Samplers & UV Mapping](./materials/05-textures-and-samplers.md)** *(45 min)*
  - *C++*: Procedural texture generation (checkerboard/noise) in memory, `std::span`, row-pitch byte alignment.
  - *Metal*: `MTL::TextureDescriptor`, `MTL::Texture`, `replaceRegion`, `MTL::SamplerDescriptor`, MSL `texture2d::sample`.
  - *DoD*: A textured quad rendered on screen with configurable linear and nearest-neighbor sampling.

---

### Phase 2: 3D Graphics & Compute (Days 6–10)
- [ ] **[TASK-06: 3D Cube, Depth Testing & Camera Projection](./materials/06-depth-testing-and-3d-cube.md)** *(60 min)*
  - *C++*: 3D Math (Perspective projection, View matrix, Euler angles/quaternions), index buffers (`uint16_t`).
  - *Metal*: `MTL::DepthStencilDescriptor`, depth texture attachment, Metal clip space $[0, 1]$, back-face culling.
  - *DoD*: A spinning 3D colored cube with correct depth occlusion and perspective projection.

- [ ] **[TASK-07: Metal Compute Pipeline (CUDA vs Metal Mental Model)](./materials/07-metal-compute-pipeline.md)** *(45 min)*
  - *C++*: Shared memory structs between Host C++ and MSL compute kernels via shared header.
  - *Metal*: `MTL::ComputePipelineState`, `MTL::ComputeCommandEncoder`, `MTL::Size` (threadgroups and threads per threadgroup).
  - *DoD*: A GPGPU compute kernel executed on GPU (e.g. procedural image or particle physics) with verified CPU readback.

- [ ] **[TASK-08: Offscreen Render Targets (Low-Res Framebuffer)](./materials/08-offscreen-render-targets.md)** *(60 min)*
  - *C++*: Multi-pass architecture abstraction, render target texture lifecycle management.
  - *Metal*: Offscreen `MTL::Texture` with `RenderTarget` and `ShaderRead` flags, fullscreen quad blit pass.
  - *DoD*: Scene renders into an offscreen low-res texture (e.g. 540p) and blits to the high-res window drawable.

- [ ] **[TASK-09: Interactive Camera & Simple Lighting](./materials/09-camera-controls-and-lighting.md)** *(45 min)*
  - *C++*: High-resolution timer (`std::chrono`), frame rate delta time, basic keyboard/mouse input state.
  - *Metal*: MSL Blinn-Phong directional lighting with surface normals and specular reflection.
  - *DoD*: An illuminated 3D scene (multiple objects) navigable with an interactive camera.

- [ ] **[TASK-10: GPU Profiling & Metal Capture Infrastructure](./materials/10-profiling-and-gpu-capture.md)** *(30 min)*
  - *C++*: Moving average frame-time counter, programmatic capture trigger.
  - *Metal*: `MTL::CaptureManager`, GPU timestamp queries (`gpuStartTime`, `gpuEndTime`).
  - *DoD*: On-screen/console microsecond GPU timing logs and programmatic Xcode GPU capture trigger.

---

### Phase 3: MetalFX Spatial Upscaling (Days 11–15)
- [ ] **[TASK-11: MetalFX Architecture & Device Support Query](./materials/11-metalfx-spatial-foundations.md)** *(30 min)*
  - *C++*: Feature detection, `std::expected` / error handling patterns, enum wrappers.
  - *MetalFX*: Linking `MetalFX.framework`, `MTLFX::SpatialScalerDescriptor::supportsDevice`.
  - *DoD*: Application checks and logs MetalFX spatial scaler hardware support and compatible pixel formats.

- [ ] **[TASK-12: Configuring the Spatial Scaler Descriptor](./materials/12-metalfx-spatial-descriptor.md)** *(45 min)*
  - *C++*: Separation of low-res render dimensions and high-res presentation dimensions.
  - *MetalFX*: `MTLFX::SpatialScalerDescriptor`, input/output dimensions, `colorTextureFormat`, `outputTextureFormat`, `colorProcessingMode`.
  - *DoD*: `MTLFX::SpatialScaler` instantiated successfully for low-res input (e.g. 540p) to target window size.

- [ ] **[TASK-13: Encoding the Spatial Scaler Pass](./materials/13-encoding-spatial-scaler.md)** *(60 min)*
  - *C++*: Command buffer sequencing, hazard management.
  - *MetalFX*: `scaler->setColorTexture`, `scaler->setOutputTexture`, `scaler->encodeToCommandBuffer`.
  - *DoD*: 3D scene rendered at low resolution and upscaled in real-time via MetalFX to the display.

- [ ] **[TASK-14: Dynamic Window Resizing & Scaling Presets](./materials/14-dynamic-resizing-and-quality-modes.md)** *(45 min)*
  - *C++*: Resource reallocation on resize, scaling factor presets (Performance 50%, Balanced 67%, Quality 75%).
  - *MetalFX*: Scaler re-creation rules and cost considerations on window resize.
  - *DoD*: Window can be dynamically resized and scaler presets toggled seamlessly without memory leaks or crashes.

- [ ] **[TASK-15: Color Spaces, Precision & Tonemapping](./materials/15-color-spaces-and-hdr.md)** *(45 min)*
  - *C++*: Precision trade-offs (FP16 vs FP32 vs UNORM), bitwise flag configurations.
  - *MetalFX*: Perceptual color processing, `MTL::PixelFormatRGBA16Float`, sRGB vs Linear color pipelines.
  - *DoD*: Crisp, artifact-free spatial upscaling with proper linear/sRGB color space handling.

---

### Phase 4: MetalFX Temporal Upscaling & Capstone (Days 16–22)
- [ ] **[TASK-16: Temporal Scaler Theory & Descriptor Requirements](./materials/16-metalfx-temporal-foundations.md)** *(45 min)*
  - *C++*: Temporal history state abstraction.
  - *MetalFX*: `MTLFX::TemporalScalerDescriptor`, temporal inputs (Color, Depth, Motion, History).
  - *DoD*: Hardware support queried and `MTLFX::TemporalScaler` initialized.

- [ ] **[TASK-17: Subpixel Camera Jitter & Halton Sequence](./materials/17-subpixel-jitter-halton.md)** *(60 min)*
  - *C++*: Radical inverse function, Halton(2, 3) sequence generator, `constexpr` sequence lookup table.
  - *Metal*: Applying subpixel jitter offsets to perspective projection matrix.
  - *DoD*: Camera projection jitters across consecutive frames; low-res render shows subpixel phase shifts.

- [ ] **[TASK-18: Screen-Space Motion Vectors (Velocity Buffer)](./materials/18-motion-vectors-velocity-buffer.md)** *(60 min)*
  - *C++*: Caching previous frame's MVP matrix per object, dual-projection calculations.
  - *Metal*: Multiple Render Targets (MRT) in MSL, writing screen-space motion vectors `(curr_ndc - prev_ndc)` to 2D RG texture.
  - *DoD*: Scene outputs a valid velocity buffer; moving objects produce colored motion vectors in NDC space.

- [ ] **[TASK-19: Depth Precision & Reversed-Z Configuration](./materials/19-depth-precision-for-temporal.md)** *(45 min)*
  - *C++*: Floating-point precision distribution across view frustum.
  - *Metal*: MetalFX depth format requirements (`Depth32Float`), Reversed-Z configuration (near=1.0, far=0.0).
  - *DoD*: High-precision depth buffer bound and verified for temporal reconstruction.

- [ ] **[TASK-20: Wiring the Temporal Scaler Pass](./materials/20-temporal-scaler-wiring.md)** *(60 min)*
  - *C++*: Frame-to-frame state management, camera cut detection (`resetHistory`).
  - *MetalFX*: Binding color, depth, motion, output textures, jitter offsets, and encoding pass.
  - *DoD*: MetalFX Temporal Scaler successfully upscaling 3D scene with accumulated subpixel detail and temporal stability.

- [ ] **[TASK-21: Temporal Artifact Mitigation & Tuning](./materials/21-artifact-mitigation-tuning.md)** *(45 min)*
  - *C++*: Reactive mask generation, camera velocity thresholding.
  - *MetalFX*: Tuning motion vector scaling, handling disocclusion, ghosting mitigation on fast-moving geometry.
  - *DoD*: Rock-solid temporal reconstruction without excessive ghosting or edge shimmering during motion.

- [ ] **[TASK-22: Capstone: The Multi-Mode Comparison Playground](./materials/22-comparison-playground.md)** *(60 min)*
  - *C++*: Clean architectural structure, RAII resource wrappers, interactive toggle modes.
  - *Metal / MetalFX*: Real-time split-screen / side-by-side mode switcher:
    1. Native Low-Res (nearest)
    2. Bilinear Upscale
    3. MetalFX Spatial
    4. MetalFX Temporal
    5. Native High-Res
  - *DoD*: Polished interactive playground showing split-screen comparisons with live GPU timing metrics.
