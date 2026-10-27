# TASK-14: Dynamic Window Resizing & Scaling Presets

## 1. Concept & Mental Model

### Handling Window Resizing in Real-Time
When the user drags the corner of the window to resize it, the presentation surface dimensions ($W_{\text{out}}, H_{\text{out}}$) change. Because an `MTLFX::SpatialScaler` is compiled for **fixed input and output dimensions**, resizing the window mandates reallocating your render targets and recompiling the scaler.

### Industry-Standard Scaling Presets
Upscaling presets maintain a fixed ratio between the internal rendering resolution and the target window display:

$$\text{Scale Factor } S = \frac{W_{\text{in}}}{W_{\text{out}}} = \frac{H_{\text{in}}}{H_{\text{out}}}$$

| Preset Mode | Scale Factor ($S$) | Effective Upscale Multiplier | 1080p Target Input Size | Fill-Rate Savings |
|---|---|---|---|---|
| **Quality** | $0.67$ ($67\%$) | $1.50\times$ | $1280 \times 720$ | **$55.6\%$** |
| **Balanced** | $0.58$ ($58\%$) | $1.72\times$ | $1114 \times 626$ | **$66.3\%$** |
| **Performance** | $0.50$ ($50\%$) | $2.00\times$ | $960 \times 540$ | **$75.0\%$** |

---

## 2. Resource Reallocation Protocol

To prevent crashes and GPU memory leaks during continuous resizing:
1. **Drain GPU Work**: Ensure the GPU is not currently reading from the textures being destroyed.
2. **Release Old Resources**: Call `->release()` on old offscreen color and depth textures, and the old `MTLFX::SpatialScaler`.
3. **Recompute Input Dimensions**:
$$W_{\text{in}} = \max(64, \lfloor W_{\text{out}} \times S \rfloor), \quad H_{\text{in}} = \max(64, \lfloor H_{\text{out}} \times S \rfloor)$$
4. **Reallocate & Compile**: Allocate new textures and create a new `SpatialScaler`.
5. **Update Projection**: Update camera perspective aspect ratio $a = \frac{W_{\text{out}}}{H_{\text{out}}}$.

---

## 3. Metal & C++ Blueprint

### Preset Manager (`src/UpscaleManager.hpp`)
```cpp
#pragma once
#include <Metal/Metal.hpp>
#include <MetalFX/MetalFX.hpp>
#include <algorithm>

enum class ScalingPreset {
    Quality,     // 67%
    Balanced,    // 58%
    Performance  // 50%
};

class UpscaleManager {
public:
    ScalingPreset currentPreset = ScalingPreset::Quality;

    float scaleFactor() const {
        switch (currentPreset) {
            case ScalingPreset::Quality:     return 0.67f;
            case ScalingPreset::Balanced:    return 0.58f;
            case ScalingPreset::Performance: return 0.50f;
        }
    }

    void onResize(MTL::Device* pDevice, uint32_t outWidth, uint32_t outHeight) {
        if (outWidth == 0 || outHeight == 0) return;

        _outputWidth = outWidth;
        _outputHeight = outHeight;

        uint32_t inWidth = std::max(64u, static_cast<uint32_t>(_outputWidth * scaleFactor()));
        uint32_t inHeight = std::max(64u, static_cast<uint32_t>(_outputHeight * scaleFactor()));

        // Release old textures and scaler
        if (_pScaler) { _pScaler->release(); _pScaler = nullptr; }
        if (_pColorTexture) { _pColorTexture->release(); _pColorTexture = nullptr; }
        if (_pDepthTexture) { _pDepthTexture->release(); _pDepthTexture = nullptr; }

        // Reallocate color texture
        MTL::TextureDescriptor* colorDesc = MTL::TextureDescriptor::texture2DDescriptor(
            MTL::PixelFormatBGRA8Unorm_sRGB, inWidth, inHeight, false
        );
        colorDesc->setUsage(MTL::TextureUsageRenderTarget | MTL::TextureUsageShaderRead);
        colorDesc->setStorageMode(MTL::StorageModePrivate);
        _pColorTexture = pDevice->newTexture(colorDesc);
        colorDesc->release();

        // Reallocate depth texture
        MTL::TextureDescriptor* depthDesc = MTL::TextureDescriptor::texture2DDescriptor(
            MTL::PixelFormatDepth32Float, inWidth, inHeight, false
        );
        depthDesc->setUsage(MTL::TextureUsageRenderTarget);
        depthDesc->setStorageMode(MTL::StorageModePrivate);
        _pDepthTexture = pDevice->newTexture(depthDesc);
        depthDesc->release();

        // Recompile scaler
        MTLFX::SpatialScalerDescriptor* desc = MTLFX::SpatialScalerDescriptor::alloc()->init();
        desc->setInputWidth(inWidth);
        desc->setInputHeight(inHeight);
        desc->setOutputWidth(_outputWidth);
        desc->setOutputHeight(_outputHeight);
        desc->setColorTextureFormat(MTL::PixelFormatBGRA8Unorm_sRGB);
        desc->setOutputTextureFormat(MTL::PixelFormatBGRA8Unorm_sRGB);
        desc->setColorProcessingMode(MTLFX::SpatialScalerColorProcessingModePerceptual);

        _pScaler = desc->newSpatialScaler(pDevice);
        desc->release();
    }

    MTLFX::SpatialScaler* scaler() const { return _pScaler; }
    MTL::Texture* colorTexture() const { return _pColorTexture; }
    MTL::Texture* depthTexture() const { return _pDepthTexture; }

private:
    uint32_t _outputWidth = 1280;
    uint32_t _outputHeight = 720;
    MTLFX::SpatialScaler* _pScaler = nullptr;
    MTL::Texture* _pColorTexture = nullptr;
    MTL::Texture* _pDepthTexture = nullptr;
};
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Connect `Window::ResizeCallback` to trigger `UpscaleManager::onResize()`.
- [ ] Implement hotkeys ('1' for Quality, '2' for Balanced, '3' for Performance) that trigger scaler recompilation.
- [ ] Test rapid window dragging: Confirm there are no memory leaks in Activity Monitor and no assertion crashes.
- [ ] Confirm that aspect ratio remains undistorted across arbitrary window shapes.

---

## 5. Definition of Done & Commit Message
- **DoD**: Window can be dynamically resized and scaling presets toggled between Quality, Balanced, and Performance in real-time without crashes or memory growth.
- **Commit**:
  ```bash
  git add src/UpscaleManager.hpp src/Renderer.cpp
  git commit -m "feat(metalfx): implement dynamic window resizing and scaling presets"
  ```
