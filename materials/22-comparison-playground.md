# TASK-22: Capstone: The Multi-Mode Comparison Playground

## 1. Concept & Mental Model
The Capstone Playground synthesizes all curriculum modules (C++20 AppKit windowing, dynamic uniform triple-buffering, 3D Blinn-Phong lighting, offscreen rendering, MetalFX Spatial, MetalFX Temporal, and microsecond GPU profiling) into an interactive real-time multi-mode comparison application. Modeled after professional GPU SDK comparison tools (such as NVIDIA DLSS or AMD FidelityFX suites), the playground enables instant split-screen slider comparisons and runtime mode switching with live millisecond timing readouts.

## 2. Mathematical Foundations & Algorithms
### Interactive Split-Screen Separator
Let $x_{\text{split}} \in [0.0, 1.0]$ be the normalized horizontal slider position controlled via mouse dragging. For screen coordinate $(u, v) \in [0, 1]^2$ and separator bar width $\epsilon = \frac{2.0}{W_{\text{display}}}$:
$$M(u, v) = \begin{cases}
\text{Texture A}(u, v) & \text{if } u < x_{\text{split}} - \frac{\epsilon}{2} \\
\text{Divider Color (White)} & \text{if } |u - x_{\text{split}}| \le \frac{\epsilon}{2} \\
\text{Texture B}(u, v) & \text{if } u > x_{\text{split}} + \frac{\epsilon}{2}
\end{cases}$$

### Comparison Modes Spectrum
1. **Low-Res Nearest**: $540p$ render target displayed with nearest-neighbor magnification (Baseline aliasing).
2. **Bilinear Upscale**: $540p$ render target filtered with hardware linear sampler (Baseline blur).
3. **MetalFX Spatial**: $540p \rightarrow 1080p$ single-pass edge-adaptive reconstruction.
4. **MetalFX Temporal**: $540p \rightarrow 1080p$ multi-frame temporal super-sampling with Halton jitter.
5. **Native High-Res**: True $1080p$ rasterization (Ground Truth reference).

## 3. Metal & C++ Blueprint

### 1. Comparison Mode Enumeration & Engine State
```cpp
enum class UpscaleMode {
    LowResNearest   = 1,
    Bilinear        = 2,
    MetalFXSpatial  = 3,
    MetalFXTemporal = 4,
    NativeHighRes   = 5
};

struct PlaygroundState {
    UpscaleMode modeLeft  = UpscaleMode::MetalFXTemporal;
    UpscaleMode modeRight = UpscaleMode::NativeHighRes;
    float splitPosition   = 0.5f; // Center divider
    bool  isDraggingSplit = false;
    
    // Performance Metrics
    double sceneRenderTimeMs = 0.0;
    double upscalePassTimeMs = 0.0;
    double totalGpuTimeMs    = 0.0;
    double fps               = 60.0;
};
```

### 2. Split-Screen Comparison Blit Shader (MSL)
```metal
#include <metal_stdlib>
using namespace metal;

struct SplitUniforms {
    float splitPosition;
    float dividerWidth;
    float2 _pad; // 8-byte padding to satisfy 16-byte constant buffer alignment
};

struct VertexOut {
    float4 position [[position]];
    float2 texCoord;
};

fragment float4 splitScreenFragment(VertexOut in [[stage_in]],
                                    texture2d<float> texLeft  [[texture(0)]],
                                    texture2d<float> texRight [[texture(1)]],
                                    sampler smp [[sampler(0)]],
                                    constant SplitUniforms& u [[buffer(0)]]) {
    float uCoord = in.texCoord.x;
    float halfDiv = u.dividerWidth * 0.5f;

    // Draw white vertical divider bar
    if (abs(uCoord - u.splitPosition) <= halfDiv) {
        return float4(1.0, 1.0, 1.0, 1.0);
    }

    if (uCoord < u.splitPosition) {
        return texLeft.sample(smp, in.texCoord);
    } else {
        return texRight.sample(smp, in.texCoord);
    }
}
```

### 3. Application Interactive Controls
- Keys `1`–`5`: Switch primary left-side display mode.
- Keys `6`–`0`: Switch secondary right-side display mode.
- Mouse Drag: Move split-screen divider dynamically across the window.
- Key `Space`: Toggle camera animation.
- Key `C`: Programmatically capture single-frame Xcode `.gputrace`.

## 4. Step-by-Step Implementation Checklist
- [ ] Set up project structure linking `Metal`, `MetalFX`, `Foundation`, `QuartzCore`, and `MetalKit`.
- [ ] Initialize both `MTLFX::SpatialScaler` and `MTLFX::TemporalScaler`.
- [ ] Allocate low-resolution ($540p$) and native-resolution ($1080p$) render target textures.
- [ ] Implement the interactive split-screen blit pipeline.
- [ ] Hook mouse events (`mouseDragged`, `mouseDown`, `mouseUp`) to update `splitPosition`.
- [ ] Implement `GpuProfiler` to record separate timestamp intervals for scene rasterization and upscale passes.
- [ ] Format live console HUD reporting mode, GPU frame times, and speedup ratios.

## 5. Common Pitfalls & Debugging Checklist
- **Jitter Visible in Split-Screen Mode**:
  - *Symptom*: When comparing Temporal on Left with Native on Right, Native geometry appears to jitter.
  - *Cause*: Sharing the jittered camera matrix across both render passes.
  - *Fix*: Render Native High-Res pass with unjittered camera matrix; only apply jitter to low-res Temporal pass.
- **Drawable Texture Incompatibility**:
  - *Cause*: Attempting to sample directly from `CA::MetalDrawable` in the split-screen blit shader without `ShaderRead` usage.
  - *Fix*: Render modes into intermediate offscreen textures, then composite both into the drawable in a final split-screen pass.
- **Resource Exhaustion from Redundant Passes**:
  - *Optimization*: Only execute the two render/upscale passes currently selected on the left and right sides of the split.
