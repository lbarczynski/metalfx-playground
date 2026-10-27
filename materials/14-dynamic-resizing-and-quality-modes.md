# TASK-14: Dynamic Window Resizing & Scaling Presets

## 1. Concept & Mental Model
Modern real-time engines allow users to freely resize application windows and toggle between performance-optimized scaling presets. Unlike standard texture sampling where linear filters stretch arbitrarily, `MTLFX::SpatialScaler` is compiled for a fixed $(W_{\text{in}}, H_{\text{in}}) \rightarrow (W_{\text{out}}, H_{\text{out}})$ resolution pair; any dimension change requires tearing down the old scaler, adjusting texture allocations, and recompiling a new scaler instance.

## 2. Mathematical Foundations & Algorithms
### Scaling Presets & Fill-Rate Ratios
Presets balance frame rate gains against visual fidelity:

$$\text{Scale Ratio } s = \frac{W_{\text{in}}}{W_{\text{out}}}$$

| Preset Mode | Scale Factor ($s$) | Upscale Factor ($R$) | Input from 1440p ($2560 \times 1440$) | Render Cost |
| :--- | :--- | :--- | :--- | :--- |
| **Native** | $1.00\times$ | $1.00\times$ | $2560 \times 1440$ | $100\%$ |
| **Ultra Quality** | $0.77\times$ | $1.30\times$ | $1970 \times 1108$ | $59.3\%$ |
| **Quality** | $0.67\times$ | $1.50\times$ | $1714 \times 964$ | $44.9\%$ |
| **Balanced** | $0.58\times$ | $1.72\times$ | $1484 \times 834$ | $33.6\%$ |
| **Performance** | $0.50\times$ | $2.00\times$ | $1280 \times 720$ | $25.0\%$ |

### Dynamic Aspect Ratio Invariance
To prevent geometric distortion when resizing windows:
$$\text{Aspect Ratio } A = \frac{W_{\text{out}}}{H_{\text{out}}}$$
$$W_{\text{in}} = \max\left(16, \lfloor W_{\text{out}} \cdot s \rfloor \ \& \ \sim 1\right)$$
$$H_{\text{in}} = \max\left(16, \lfloor H_{\text{out}} \cdot s \rfloor \ \& \ \sim 1\right)$$
$$\text{Perspective Aspect} = \frac{W_{\text{in}}}{H_{\text{in}}} \approx A$$

The camera's perspective projection matrix must be updated with $a = \frac{W_{\text{in}}}{H_{\text{in}}}$.

## 3. Metal & C++ Blueprint

### 1. Scaling Preset Enumeration
```cpp
enum class ScalingPreset {
    Native       = 0, // 1.00x
    UltraQuality = 1, // 0.77x
    Quality      = 2, // 0.67x
    Balanced     = 3, // 0.58x
    Performance  = 4  // 0.50x
};

inline float getScaleFactor(ScalingPreset preset) {
    switch (preset) {
        case ScalingPreset::Native:       return 1.00f;
        case ScalingPreset::UltraQuality: return 0.77f;
        case ScalingPreset::Quality:      return 0.67f;
        case ScalingPreset::Balanced:     return 0.58f;
        case ScalingPreset::Performance:  return 0.50f;
        default:                          return 1.00f;
    }
}
```

### 2. Scaler Manager Class with Lifecycle Management
```cpp
#include <Metal/Metal.hpp>
#include <MetalFX/MetalFX.hpp>

class SpatialUpscaleManager {
public:
    SpatialUpscaleManager(MTL::Device* pDevice, MTL::PixelFormat outputFormat)
        : _pDevice(pDevice->retain())
        , _outputFormat(outputFormat) {}

    ~SpatialUpscaleManager() {
        releaseResources();
        _pDevice->release();
    }

    void setPreset(ScalingPreset preset) {
        if (_preset != preset) {
            _preset = preset;
            rebuild(true);
        }
    }

    void resize(uint32_t outWidth, uint32_t outHeight) {
        if (_outWidth == outWidth && _outHeight == outHeight) return;
        _outWidth = outWidth;
        _outHeight = outHeight;
        rebuild(false);
    }

    MTLFX::SpatialScaler* getScaler() const { return _pScaler; }
    MTL::Texture* getColorTexture() const { return _pColorTexture; }
    MTL::Texture* getDepthTexture() const { return _pDepthTexture; }
    uint32_t getInputWidth() const { return _inWidth; }
    uint32_t getInputHeight() const { return _inHeight; }

private:
    void releaseResources() {
        if (_pScaler) { _pScaler->release(); _pScaler = nullptr; }
        if (_pColorTexture) { _pColorTexture->release(); _pColorTexture = nullptr; }
        if (_pDepthTexture) { _pDepthTexture->release(); _pDepthTexture = nullptr; }
    }

    void rebuild(bool force) {
        if (_outWidth == 0 || _outHeight == 0) return;

        float scale = getScaleFactor(_preset);
        uint32_t newInW = std::max<uint32_t>(16, static_cast<uint32_t>(_outWidth * scale) & ~1);
        uint32_t newInH = std::max<uint32_t>(16, static_cast<uint32_t>(_outHeight * scale) & ~1);

        if (!force && newInW == _inWidth && newInH == _inHeight) return;

        _inWidth = newInW;
        _inHeight = newInH;

        releaseResources();

        // 1. Allocate Low-Res Color Target
        MTL::TextureDescriptor* colorDesc = MTL::TextureDescriptor::texture2DDescriptor(
            MTL::PixelFormatRGBA8Unorm, _inWidth, _inHeight, false
        );
        colorDesc->setUsage(MTL::TextureUsageRenderTarget | MTL::TextureUsageShaderRead);
        colorDesc->setStorageMode(MTL::StorageModePrivate);
        _pColorTexture = _pDevice->newTexture(colorDesc);
        colorDesc->release();

        // 2. Allocate Low-Res Depth Target
        MTL::TextureDescriptor* depthDesc = MTL::TextureDescriptor::texture2DDescriptor(
            MTL::PixelFormatDepth32Float, _inWidth, _inHeight, false
        );
        depthDesc->setUsage(MTL::TextureUsageRenderTarget);
        depthDesc->setStorageMode(MTL::StorageModePrivate);
        _pDepthTexture = _pDevice->newTexture(depthDesc);
        depthDesc->release();

        // 3. Compile New Spatial Scaler
        MTLFX::SpatialScalerDescriptor* scalerDesc = MTLFX::SpatialScalerDescriptor::alloc()->init();
        scalerDesc->setInputWidth(_inWidth);
        scalerDesc->setInputHeight(_inHeight);
        scalerDesc->setOutputWidth(_outWidth);
        scalerDesc->setOutputHeight(_outHeight);
        scalerDesc->setColorTextureFormat(MTL::PixelFormatRGBA8Unorm);
        scalerDesc->setOutputTextureFormat(_outputFormat);
        scalerDesc->setColorProcessingMode(MTLFX::SpatialScalerColorProcessingModePerceptual);

        _pScaler = scalerDesc->newSpatialScaler(_pDevice);
        scalerDesc->release();
    }

    MTL::Device* _pDevice = nullptr;
    MTL::PixelFormat _outputFormat;
    ScalingPreset _preset = ScalingPreset::Quality;

    uint32_t _inWidth = 0;
    uint32_t _inHeight = 0;
    uint32_t _outWidth = 0;
    uint32_t _outHeight = 0;

    MTLFX::SpatialScaler* _pScaler = nullptr;
    MTL::Texture* _pColorTexture = nullptr;
    MTL::Texture* _pDepthTexture = nullptr;
};
```

## 4. Step-by-Step Implementation Checklist
- [ ] Implement `ScalingPreset` enum supporting Native, Quality, Balanced, and Performance.
- [ ] Connect `MTK::ViewDelegate::drawableSizeWillChange` to notify the upscale manager of window dimension changes.
- [ ] Release old textures and `MTLFX::SpatialScaler` before creating new ones.
- [ ] Align input dimensions to even integers with `& ~1`.
- [ ] Update camera perspective projection with the new aspect ratio $a = W_{\text{in}} / H_{\text{in}}$.
- [ ] Wire keyboard shortcuts (1, 2, 3, 4) to switch presets dynamically at runtime.
- [ ] Confirm in GPU trace that resizing and switching presets occurs smoothly without memory leaks.

## 5. Common Pitfalls & Debugging Checklist
- **Memory Leaks on Window Drag Resize**:
  - *Symptom*: Continuous memory accumulation while dragging the window corner.
  - *Cause*: Leaking the previous `MTLFX::SpatialScaler` or `MTL::Texture` instances on every resize tick.
  - *Fix*: Call `release()` on existing resources before allocating new ones.
- **Zero-Size Window Minimization Crash**:
  - *Cause*: Attempting to allocate textures or scalers with width or height equal to 0 when window is minimized.
  - *Fix*: Guard with `if (_outWidth == 0 || _outHeight == 0) return;`.
