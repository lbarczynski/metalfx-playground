# TASK-12: Configuring the Spatial Scaler Descriptor

## 1. Concept & Mental Model

### Scaler Descriptor vs Compiled Scaler
In MetalFX, setting up upscaling follows a two-stage factory pattern:
1. **`MTLFX::SpatialScalerDescriptor`**: A configuration blueprint specifying input/output dimensions, pixel formats, and color processing modes.
2. **`MTLFX::SpatialScaler`**: A compiled GPU filter object compiled by `descriptor->newSpatialScaler(pDevice)`.

Creating a compiled scaler takes several milliseconds because Metal compiles internal compute kernels for your exact dimension ratio and pixel format. Therefore, **scalers should be created during initialization or window resize, never every frame**.

---

## 2. Invariants & Texture Usage Requirements

A compiled scaler exposes the exact usage flags required for its input and output textures:
- `scaler->colorTextureUsage()`: Typically requires `MTL::TextureUsageShaderRead`.
- `scaler->outputTextureUsage()`: Typically requires `MTL::TextureUsageShaderWrite` (or `ShaderRead | ShaderWrite`).

> [!CAUTION]
> **The `framebufferOnly` Trap**:
> By default, `CAMetalLayer.framebufferOnly` is `YES`. Textures with `framebufferOnly` cannot be written to by compute shaders (`ShaderWrite` is disallowed).
> If you bind `drawable->texture()` directly as the scaler's output texture, you **must set `metalLayer.framebufferOnly = NO;`** in your window setup! Otherwise, Metal validation will crash with:
> `validateSpatialScaler: Output texture must support MTLTextureUsageShaderWrite`.

> [!IMPORTANT]
> **sRGB Format Consistency Rule**:
> In MetalFX, input color format and output format must share the exact same sRGB transfer characteristics.
> Pairing an sRGB input (`RGBA8Unorm_sRGB`) with a linear output (`RGBA8Unorm`) triggers a MetalFX runtime driver diagnostic:
> `mixed sRGB inputs and outputs is not supported`. Always keep both as `_sRGB` or both as linear.
> Also, when computing even dimensions for scalers, use unsigned bitwise masks: `W_in = std::max(16u, W_raw & ~1u);` to avoid `-Wconversion` compiler warnings.

---

## 3. Metal & C++ Blueprint

### Initializing the Spatial Scaler (`src/SpatialUpscaler.hpp`)
```cpp
#pragma once
#include <Metal/Metal.hpp>
#include <MetalFX/MetalFX.hpp>
#include <iostream>

class SpatialUpscaler {
public:
    SpatialUpscaler(
        MTL::Device* pDevice,
        uint32_t inputWidth,
        uint32_t inputHeight,
        uint32_t outputWidth,
        uint32_t outputHeight,
        MTL::PixelFormat format
    ) : _pDevice(pDevice) {
        // 1. Create Descriptor
        MTLFX::SpatialScalerDescriptor* desc = MTLFX::SpatialScalerDescriptor::alloc()->init();
        desc->setInputWidth(inputWidth);
        desc->setInputHeight(inputHeight);
        desc->setOutputWidth(outputWidth);
        desc->setOutputHeight(outputHeight);
        desc->setColorTextureFormat(format);
        desc->setOutputTextureFormat(format);
        desc->setColorProcessingMode(MTLFX::SpatialScalerColorProcessingModePerceptual);

        // 2. Instantiate Compiled Scaler
        _pScaler = desc->newSpatialScaler(pDevice);
        desc->release();

        if (!_pScaler) {
            std::cerr << "CRITICAL: Failed to create MTLFX::SpatialScaler!\n";
            return;
        }

        std::cout << ">>> MetalFX Spatial Scaler created successfully: "
                  << inputWidth << "x" << inputHeight << " -> "
                  << outputWidth << "x" << outputHeight << "\n";
    }

    ~SpatialUpscaler() {
        if (_pScaler) {
            _pScaler->release();
        }
    }

    MTLFX::SpatialScaler* scaler() const { return _pScaler; }
    MTL::TextureUsage requiredInputUsage() const { return _pScaler->colorTextureUsage(); }
    MTL::TextureUsage requiredOutputUsage() const { return _pScaler->outputTextureUsage(); }

private:
    MTL::Device* _pDevice = nullptr;
    MTLFX::SpatialScaler* _pScaler = nullptr;
};
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] In `src/platform/WindowBridge.mm`, set `metalLayer.framebufferOnly = NO;`.
- [ ] Configure `MTLFX::SpatialScalerDescriptor` for 540p ($960 \times 540$) input and 1080p ($1920 \times 1080$) output.
- [ ] Compile the scaler using `desc->newSpatialScaler(pDevice)`.
- [ ] Verify `_pScaler != nullptr`.
- [ ] Log `scaler->colorTextureUsage()` and `scaler->outputTextureUsage()` to confirm required flags.

---

## 5. Definition of Done & Commit Message
- **DoD**: `MTLFX::SpatialScaler` compiles successfully for low-res input to target window dimensions, with `framebufferOnly = NO` configured.
- **Commit**:
  ```bash
  git add src/SpatialUpscaler.hpp src/platform/WindowBridge.mm src/Renderer.cpp
  git commit -m "feat(metalfx): configure spatial scaler descriptor and instantiate scaler"
  ```
