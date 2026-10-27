# TASK-16: Temporal Scaler Theory & Descriptor Requirements

## 1. Concept & Mental Model
MetalFX Temporal accumulates subpixel samples across successive frames into an internal history buffer to reconstruct high-frequency geometric detail and texture sharpness well beyond the Nyquist limit of the low-resolution render target. Analogous to NVIDIA DLSS 2+ and AMD FSR 2+, Apple MetalFX Temporal leverages Apple Silicon GPU and Neural Engine hardware, consuming jittered color, depth, and screen-space motion vectors to output a temporally stabilized, anti-aliased high-resolution frame.

## 2. Mathematical Foundations & Algorithms
### Nyquist-Shannon Sampling & Aliasing in Rasterization
A discrete raster grid of pixel spacing $\Delta x$ imposes a spatial Nyquist cutoff frequency:
$$f_N = \frac{1}{2 \Delta x}$$

Frequencies in geometric edges and fine textures higher than $f_N$ fold back into low frequencies, creating jagged edges and moiré patterns. By jittering camera subpixel offsets $\delta_k \in [-0.5, 0.5]\Delta x$ across $K$ consecutive frames, the effective spatial sampling rate increases by a factor of $K$:
$$f_{N,\text{effective}} = \frac{K}{2 \Delta x} = K \cdot f_N$$

### Exponential Temporal Accumulation
For pixel coordinate $\mathbf{x} = (x, y)$ with motion vector $\mathbf{v}(\mathbf{x})$:
$$\mathbf{C}_{\text{accum}}(\mathbf{x}) = \alpha \cdot \mathbf{C}_{\text{curr}}(\mathbf{x}) + (1.0 - \alpha) \cdot \mathbf{C}_{\text{history}}(\mathbf{x} - \mathbf{v}(\mathbf{x}))$$
where blend weight $\alpha \in [0.05, 0.10]$ allows the reconstructor to weigh new subpixel information against historical samples.

## 3. Metal & C++ Blueprint

### 1. Temporal Scaler Descriptor Configuration
```cpp
#include <Metal/Metal.hpp>
#include <MetalFX/MetalFX.hpp>
#include <stdexcept>

MTLFX::TemporalScaler* createTemporalScaler(
    MTL::Device* pDevice,
    uint32_t inW, uint32_t inH,
    uint32_t outW, uint32_t outH,
    MTL::PixelFormat colorFormat,
    MTL::PixelFormat outputFormat
) {
    if (!MTLFX::TemporalScalerDescriptor::supportsDevice(pDevice)) {
        throw std::runtime_error("MetalFX Temporal Scaler unsupported on device.");
    }

    MTLFX::TemporalScalerDescriptor* pDesc = MTLFX::TemporalScalerDescriptor::alloc()->init();
    
    pDesc->setInputWidth(inW);
    pDesc->setInputHeight(inH);
    pDesc->setOutputWidth(outW);
    pDesc->setOutputHeight(outH);
    
    pDesc->setColorTextureFormat(colorFormat);                      // e.g. MTL::PixelFormatRGBA16Float
    pDesc->setDepthTextureFormat(MTL::PixelFormatDepth32Float);      // Strict requirement: 32-bit float
    pDesc->setMotionTextureFormat(MTL::PixelFormatRG16Float);        // 2D Screen-space motion vectors
    pDesc->setOutputTextureFormat(outputFormat);                     // e.g. MTL::PixelFormatBGRA8Unorm_sRGB
    
    MTLFX::TemporalScaler* pScaler = pDesc->newTemporalScaler(pDevice);
    pDesc->release();

    if (!pScaler) {
        throw std::runtime_error("Failed to compile MTLFX::TemporalScaler.");
    }

    // Reversed-Z is strongly recommended for floating-point depth precision (configured on scaler instance)
    pScaler->setDepthReversed(true);

    return pScaler;
}
```

### 2. Texture Attachment Requirements Matrix
| Target Attachment | Format | Required Usage Flags | Storage Mode |
| :--- | :--- | :--- | :--- |
| **Color Texture** | `RGBA16Float` or `RGBA8Unorm` | `RenderTarget \| ShaderRead` | `StorageModePrivate` |
| **Depth Texture** | `Depth32Float` | `RenderTarget \| ShaderRead` | `StorageModePrivate` |
| **Motion Texture** | `RG16Float` | `RenderTarget \| ShaderRead` | `StorageModePrivate` |
| **Output Texture** | `BGRA8Unorm_sRGB` or `RGBA16Float` | `RenderTarget \| ShaderRead` | `StorageModePrivate` / Drawable |

## 4. Step-by-Step Implementation Checklist
- [ ] Add `MTLFX::TemporalScalerDescriptor::supportsDevice(pDevice)` query in startup diagnostics.
- [ ] Allocate `MTLFX::TemporalScalerDescriptor`.
- [ ] Set input dimensions (e.g. $960 \times 540$) and output dimensions (e.g. $1920 \times 1080$).
- [ ] Configure `colorTextureFormat` (`RGBA16Float`), `depthTextureFormat` (`Depth32Float`), and `motionTextureFormat` (`RG16Float`).
- [ ] Set `pScaler->setDepthReversed(true)`.
- [ ] Instantiate `MTLFX::TemporalScaler* pScaler = pDesc->newTemporalScaler(pDevice)`.
- [ ] Release descriptor and cache scaler instance.

## 5. Common Pitfalls & Debugging Checklist
- **`Depth32Float` Format Requirement**:
  - *Error*: Validation layer asserts invalid depth format if using `Depth16Unorm` or `Depth24Unorm_Stencil8`.
  - *Fix*: MetalFX Temporal strictly mandates `MTL::PixelFormatDepth32Float`.
- **Missing `ShaderRead` Usage on Depth/Motion Attachments**:
  - *Symptom*: Fatal pipeline assertion when encoding scaler.
  - *Fix*: Both depth and velocity textures must specify `MTL::TextureUsageRenderTarget | MTL::TextureUsageShaderRead`.
- **Frame History Smearing on Startup**:
  - *Cause*: Uninitialized history buffer on first frame.
  - *Fix*: Set `pScaler->setReset(true)` on the first frame of execution.
