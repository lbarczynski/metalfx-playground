# TASK-12: Configuring the Spatial Scaler Descriptor

## 1. Concept & Mental Model
`MTLFX::SpatialScalerDescriptor` is the configuration specification for the spatial upscaling pipeline, establishing exact input/output dimensions, texture pixel formats, and the color processing domain. Analogous to compiling an immutable pipeline state or ML inference graph, `descriptor->newSpatialScaler(device)` compiles a hardware-optimized GPU compute/raster kernel tailored to the exact input-output resolution pair.

## 2. Mathematical Foundations & Algorithms
### Input Dimension Calculation & Half-Texel Phase Alignment
Given output presentation dimensions $(W_{\text{out}}, H_{\text{out}})$ and scale factor $s \in [0.33, 1.0]$:
$$W_{\text{raw}} = \lfloor W_{\text{out}} \cdot s \rfloor, \quad H_{\text{raw}} = \lfloor H_{\text{out}} \cdot s \rfloor$$

To prevent subpixel sampling jitter and phase offsets, input dimensions **must be even integers**:
$$W_{\text{in}} = \max(16, W_{\text{raw}} \ \& \ \sim 1)$$
$$H_{\text{in}} = \max(16, H_{\text{raw}} \ \& \ \sim 1)$$

### Color Processing Modes
1. `MTLFX::SpatialScalerColorProcessingModePerceptual`:
   Input pixels are assumed to be in non-linear perceptual space (gamma-encoded sRGB or tonemapped SDR). Edge gradients evaluate perceptual contrast, matching the human eye's response to shadows and highlights:
   $$Y = 0.2126 R + 0.7152 G + 0.0722 B$$
2. `MTLFX::SpatialScalerColorProcessingModeLinear`:
   Input pixels are linear photometric HDR values (e.g. `RGBA16Float`). MetalFX internally converts to perceptual space for edge detection and converts back to linear space for output.

## 3. Metal & C++ Blueprint

### 1. Descriptor Initialization & Scaler Instantiation
```cpp
#include <Metal/Metal.hpp>
#include <MetalFX/MetalFX.hpp>
#include <stdexcept>

MTLFX::SpatialScaler* createSpatialScaler(
    MTL::Device* pDevice,
    uint32_t inW, uint32_t inH,
    uint32_t outW, uint32_t outH,
    MTL::PixelFormat colorFormat,
    MTL::PixelFormat outputFormat,
    MTLFX::SpatialScalerColorProcessingMode mode
) {
    // 1. Allocate and initialize descriptor
    MTLFX::SpatialScalerDescriptor* pDesc = MTLFX::SpatialScalerDescriptor::alloc()->init();
    
    pDesc->setInputWidth(inW);
    pDesc->setInputHeight(inH);
    pDesc->setOutputWidth(outW);
    pDesc->setOutputHeight(outH);
    pDesc->setColorTextureFormat(colorFormat);
    pDesc->setOutputTextureFormat(outputFormat);
    pDesc->setColorProcessingMode(mode);

    // 2. Validate descriptor configuration
    if (!MTLFX::SpatialScalerDescriptor::supportsDevice(pDevice)) {
        pDesc->release();
        throw std::runtime_error("MetalFX Spatial Scaler unsupported on device.");
    }

    // 3. Instantiate compiled scaler object
    MTLFX::SpatialScaler* pScaler = pDesc->newSpatialScaler(pDevice);
    
    // Descriptors are transient and should be released after compilation
    pDesc->release();

    if (!pScaler) {
        throw std::runtime_error("Failed to compile MTLFX::SpatialScaler.");
    }

    return pScaler;
}
```

### 2. Texture Allocations Matching Descriptor Contract
```cpp
// 1. Low-Resolution Render Target (Offscreen Scene Buffer)
MTL::TextureDescriptor* inTexDesc = MTL::TextureDescriptor::texture2DDescriptor(
    MTL::PixelFormatRGBA8Unorm,
    inW,
    inH,
    false
);
// Scaler reads from this texture; Scene pass writes to it
inTexDesc->setUsage(MTL::TextureUsageRenderTarget | MTL::TextureUsageShaderRead);
inTexDesc->setStorageMode(MTL::StorageModePrivate);
MTL::Texture* pLowResTexture = pDevice->newTexture(inTexDesc);
inTexDesc->release();

// 2. Output Target is typically the MTKView's currentDrawable()->texture()
// with matching sRGB format (e.g. MTL::PixelFormatBGRA8Unorm if input is RGBA8Unorm, or both _sRGB)
// Note: MetalFX disallows mixing non-sRGB input with sRGB output.
```

## 4. Step-by-Step Implementation Checklist
- [ ] Determine output resolution from `pView->drawableSize()` (e.g. $1920 \times 1080$).
- [ ] Compute input resolution for a 50% scale factor ($960 \times 540$) enforced to even integers with `& ~1`.
- [ ] Allocate `MTLFX::SpatialScalerDescriptor`.
- [ ] Populate input and output dimensions on the descriptor.
- [ ] Configure `colorTextureFormat` (`MTL::PixelFormatRGBA8Unorm`) and `outputTextureFormat` (`pView->colorPixelFormat()`).
- [ ] Set `setColorProcessingMode(MTLFX::SpatialScalerColorProcessingModePerceptual)`.
- [ ] Instantiate `MTLFX::SpatialScaler* pScaler = pDesc->newSpatialScaler(pDevice)`.
- [ ] Call `pDesc->release()` immediately after instantiation.

## 5. Common Pitfalls & Debugging Checklist
- **Metal Validation Layer Error: Scaler / Texture Dimension Mismatch**:
  - *Error*: `Input texture width (959) does not match spatial scaler input width (960)`.
  - *Fix*: The texture bound at runtime must match the descriptor's width and height **exactly**.
- **Pixel Format Mismatch Crash**:
  - *Error*: `outputTextureFormat does not match output texture pixel format`.
  - *Fix*: Ensure `pDesc->setOutputTextureFormat()` matches `pView->colorPixelFormat()` (often `BGRA8Unorm_sRGB` on macOS).
- **Compiling Scaler Every Frame**:
  - *Performance Pitfall*: Calling `newSpatialScaler()` inside `drawInMTKView()` incurs massive CPU pipeline compilation stalls ($50\text{–}200\text{ ms}$). Compile once during initialization or resize.
