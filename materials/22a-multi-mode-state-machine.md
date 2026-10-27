# TASK-22A: Interactive Multi-Mode State Machine

## 1. Concept & Mental Model

### The 5 Evaluation Modes
To truly evaluate upscaling technology, you need instant, side-by-side visual and performance comparisons against native baselines. The playground architecture supports 5 distinct real-time pipelines:

```
[ 3D Scene Simulation (Camera, Geometry, Lighting) ]
                         │
        ┌────────────────┼────────────────┬────────────────┐
        ▼                ▼                ▼                ▼
 1. Native Low-Res  2. Bilinear     3. MetalFX       4. MetalFX      5. Native High-Res
    (540p Nearest)     (540p Linear)   Spatial          Temporal        (1080p Native)
        │                │                │                │                │
        └────────────────┴────────────────┼────────────────┴────────────────┘
                                          ▼
                               [ Presentation Target ]
```

1. **Native Low-Res (540p Nearest)**: Shows raw unscaled pixels; reveals baseline aliasing and pixel grid.
2. **Bilinear Upscale (540p Linear)**: Standard baseline hardware interpolation; reveals silhouette blurriness.
3. **MetalFX Spatial**: Edge-adaptive spatial filter; demonstrates edge preservation from a single frame.
4. **MetalFX Temporal**: Multi-frame temporal reconstruction; demonstrates subpixel accumulation and temporal anti-aliasing.
5. **Native High-Res (1080p Native)**: Ground-truth unscaled rasterization; baseline quality and maximum GPU load.

---

## 2. Modern C++ State Machine (`src/PlaygroundMode.hpp`)

```cpp
#pragma once
#include <string_view>

enum class PlaygroundMode {
    NativeLowRes = 1,
    BilinearUpscale = 2,
    MetalFXSpatial = 3,
    MetalFXTemporal = 4,
    NativeHighRes = 5
};

inline std::string_view getModeName(PlaygroundMode mode) {
    switch (mode) {
        case PlaygroundMode::NativeLowRes:    return "1. Native Low-Res (540p Nearest)";
        case PlaygroundMode::BilinearUpscale: return "2. Bilinear Upscale (540p Linear)";
        case PlaygroundMode::MetalFXSpatial:  return "3. MetalFX Spatial";
        case PlaygroundMode::MetalFXTemporal: return "4. MetalFX Temporal";
        case PlaygroundMode::NativeHighRes:   return "5. Native High-Res (1080p Native)";
    }
}
```

---

## 3. Metal & C++ Blueprint

### Mode Dispatcher in `Renderer::renderFrame()`
```cpp
switch (_currentMode) {
    case PlaygroundMode::NativeLowRes:
        renderScenePass(kLowResWidth, kLowResHeight, false /* unjittered */);
        renderFullscreenBlit(pNearestSampler);
        break;

    case PlaygroundMode::BilinearUpscale:
        renderScenePass(kLowResWidth, kLowResHeight, false /* unjittered */);
        renderFullscreenBlit(pLinearSampler);
        break;

    case PlaygroundMode::MetalFXSpatial:
        renderScenePass(kLowResWidth, kLowResHeight, false /* unjittered */);
        pSpatialScaler->encodeToCommandBuffer(cmd);
        break;

    case PlaygroundMode::MetalFXTemporal:
        renderScenePass(kLowResWidth, kLowResHeight, true /* jittered */);
        pTemporalScaler->encodeToCommandBuffer(cmd);
        break;

    case PlaygroundMode::NativeHighRes:
        renderScenePass(_windowWidth, _windowHeight, false /* unjittered */);
        // Direct render to drawable
        break;
}
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Define `PlaygroundMode` enum with the 5 rendering modes.
- [ ] Hook keyboard number keys ('1' through '5') to switch modes instantly.
- [ ] Implement conditional execution in `renderFrame()` routing to the appropriate pipeline.
- [ ] Test mode switching while camera is moving: Verify smooth transitions with zero GPU device stalls or memory leaks.

---

## 5. Definition of Done & Commit Message
- **DoD**: Application toggles seamlessly between all 5 rendering modes via hotkeys ('1' through '5'), printing active mode status to the console.
- **Commit**:
  ```bash
  git add src/PlaygroundMode.hpp src/Renderer.cpp
  git commit -m "feat(app): implement interactive upscaling mode state machine"
  ```
