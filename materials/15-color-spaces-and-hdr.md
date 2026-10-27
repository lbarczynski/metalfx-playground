# TASK-15: Color Spaces (sRGB vs Linear) & HDR Precision

## 1. Concept & Mental Model

### Color Spaces in Upscaling Pipelines
Human eyes do not perceive brightness linearly: we are much more sensitive to subtle differences in dark shadows than in bright highlights.
- **sRGB (Non-Linear)**: Standard display space encoded with an approximate $\gamma \approx 2.2$ power curve.
- **Linear Space**: Physically correct lighting computations in shaders must strictly occur in linear photometric space:
$$C_{\text{linear}} \approx (C_{\text{sRGB}})^{2.2}$$

If an upscaler filters images in the wrong color space, noticeable visual defects emerge:
- Dark ringing borders along high-contrast silhouettes.
- Color shifts during interpolation (e.g. red and green blending into a muddy dark brown instead of yellow).

---

## 2. MetalFX Color Processing Modes

`MTLFX::SpatialScalerDescriptor` provides three explicit color processing modes:

| Mode | Enumeration | Expected Input Data | Use Case |
|---|---|---|---|
| **Perceptual** | `SpatialScalerColorProcessingModePerceptual` | sRGB encoded non-linear colors | Standard SDR tone-mapped images |
| **Linear** | `SpatialScalerColorProcessingModeLinear` | Physically linear photometric data | Pre-tonemapping upscale passes |
| **HDR** | `SpatialScalerColorProcessingModeHDR` | Extended Dynamic Range (EDR) FP16 | High-dynamic range rendering ($L > 1.0$) |

---

## 3. High Dynamic Range (HDR) & 16-Bit Float Textures

Standard 8-bit formats (`BGRA8Unorm`) clamp color components to $[0.0, 1.0]$ with only 256 quantization levels per channel, causing visible color banding in dark gradients.

Switching offscreen render targets to **16-bit half-precision float (`MTL::PixelFormatRGBA16Float`)**:
- Allocates 16 bits per channel (1 sign bit, 5 exponent bits, 10 mantissa bits).
- Supports HDR specular highlights that exceed $1.0$ without clamping.
- Supported directly by MetalFX Spatial and Temporal scalers.

```cpp
// Configuring HDR pipeline
desc->setColorTextureFormat(MTL::PixelFormatRGBA16Float);
desc->setOutputTextureFormat(MTL::PixelFormatBGRA8Unorm_sRGB); // Tonemapped to SDR display
desc->setColorProcessingMode(MTLFX::SpatialScalerColorProcessingModeHDR);
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Configure `pLowResColorTexture` format to `MTL::PixelFormatRGBA16Float`.
- [ ] Update `SpatialScalerDescriptor` with `colorTextureFormat = RGBA16Float` and `colorProcessingMode = Perceptual` or `HDR`.
- [ ] Update scene pipeline color attachment pixel format to match `RGBA16Float`.
- [ ] Inspect output: Verify that smooth gradients show zero color banding, and edges are crisp without dark fringe halos.

---

## 5. Definition of Done & Commit Message
- **DoD**: Pipeline handles sRGB and linear color spaces accurately using 16-bit float textures without color crushing or fringe artifacts.
- **Commit**:
  ```bash
  git add src/SpatialUpscaler.hpp src/Renderer.cpp
  git commit -m "feat(color): handle linear and srgb color spaces in metalfx pipeline"
  ```
