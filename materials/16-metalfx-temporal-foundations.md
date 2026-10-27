# TASK-16: Temporal Scaler Architecture & Requirements

## 1. Concept & Mental Model

### Why Temporal Upscaling Outperforms Spatial
Spatial upscaling (MetalFX Spatial, FSR 1.0) inspects only a single frame: it has no way of knowing what details exist between two low-resolution pixels, so it can only sharpen existing edges.

**Temporal Super-Resolution (MetalFX Temporal, DLSS 2+, FSR 2+)** accumulates subpixel details over time:
- In each frame, the camera projection matrix is shifted slightly by a subpixel fraction (subpixel camera jitter).
- Each new frame captures a unique phase of the 3D scene geometry.
- By tracking how pixels moved since the previous frame (using **Motion Vectors**) and verifying surface occlusions (using **Depth**), MetalFX Temporal reconstructs an image with sharpness and fidelity rivaling or exceeding native high-resolution rendering!

---

## 2. The 4 Required Attachments & Invariants

| Attachment | Recommended Format | Purpose in Temporal Reconstruction | Required Usage Flags |
|---|---|---|---|
| **Color Texture** | `RGBA16Float` or `BGRA8Unorm_sRGB` | Current frame low-res shaded scene | `RenderTarget \| ShaderRead` |
| **Depth Texture** | `MTL::PixelFormatDepth32Float` | Identifies surface occlusions and disocclusions | `RenderTarget \| ShaderRead` |
| **Motion Texture** | `MTL::PixelFormatRG16Float` | 2D velocity vectors tracking pixel displacement | `RenderTarget \| ShaderRead` |
| **Output Texture** | `BGRA8Unorm_sRGB` or `RGBA16Float` | High-res reconstructed destination | `scaler->outputTextureUsage()` |

> [!IMPORTANT]
> Both the depth and motion vector textures **must** be allocated with `MTL::TextureUsageRenderTarget | MTL::TextureUsageShaderRead`. If `ShaderRead` is omitted, the MetalFX compute kernel cannot read them, triggering a Metal validation abort.

---

## 3. Metal & C++ Blueprint

### Querying Support & Creating the Temporal Scaler (`src/TemporalUpscaler.hpp`)
```cpp
#pragma once
#include <Metal/Metal.hpp>
#include <MetalFX/MetalFX.hpp>
#include <iostream>

class TemporalUpscaler {
public:
    TemporalUpscaler(
        MTL::Device* pDevice,
        uint32_t inWidth,
        uint32_t inHeight,
        uint32_t outWidth,
        uint32_t outHeight
    ) : _pDevice(pDevice) {
        if (!MTLFX::TemporalScalerDescriptor::supportsDevice(pDevice)) {
            std::cerr << "CRITICAL: MetalFX Temporal Scaler unsupported on this device!\n";
            return;
        }

        // Configure Descriptor
        MTLFX::TemporalScalerDescriptor* desc = MTLFX::TemporalScalerDescriptor::alloc()->init();
        desc->setInputWidth(inWidth);
        desc->setInputHeight(inHeight);
        desc->setOutputWidth(outWidth);
        desc->setOutputHeight(outHeight);

        desc->setColorTextureFormat(MTL::PixelFormatBGRA8Unorm_sRGB);
        desc->setDepthTextureFormat(MTL::PixelFormatDepth32Float);
        desc->setMotionTextureFormat(MTL::PixelFormatRG16Float);
        desc->setOutputTextureFormat(MTL::PixelFormatBGRA8Unorm_sRGB);

        // Compile Scaler
        _pScaler = desc->newTemporalScaler(pDevice);
        desc->release();

        if (!_pScaler) {
            std::cerr << "CRITICAL: Failed to create MTLFX::TemporalScaler!\n";
            return;
        }

        // Configure Reversed-Z depth mode
        _pScaler->setDepthReversed(true);

        std::cout << ">>> MetalFX Temporal Scaler initialized: "
                  << inWidth << "x" << inHeight << " -> "
                  << outWidth << "x" << outHeight << "\n";
    }

    ~TemporalUpscaler() {
        if (_pScaler) {
            _pScaler->release();
        }
    }

    MTLFX::TemporalScaler* scaler() const { return _pScaler; }

private:
    MTL::Device* _pDevice = nullptr;
    MTLFX::TemporalScaler* _pScaler = nullptr;
};
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Call `MTLFX::TemporalScalerDescriptor::supportsDevice(pDevice)` to verify hardware support.
- [ ] Configure `MTLFX::TemporalScalerDescriptor` with the 4 required pixel formats.
- [ ] Compile the scaler via `desc->newTemporalScaler(pDevice)`.
- [ ] Call `_pScaler->setDepthReversed(true)`.
- [ ] Verify `_pScaler != nullptr` and log successful initialization to the console.

---

## 5. Definition of Done & Commit Message
- **DoD**: MetalFX Temporal Scaler hardware capability is queried, and an `MTLFX::TemporalScaler` instance is successfully initialized and configured with Reversed-Z enabled.
- **Commit**:
  ```bash
  git add src/TemporalUpscaler.hpp src/main.cpp
  git commit -m "feat(metalfx): configure temporal scaler descriptor and verify inputs"
  ```
