# TASK-15: Color Spaces, Precision & Tonemapping

## 1. Concept & Mental Model
Visual artifacts in spatial upscaling (such as ringing, haloing, and dark edge blur) stem from evaluating edge detection in linear radiometric space rather than human perceptual space. Tonemapping and color space transformations dictate whether upscaling operates on non-linear SDR color (Perceptual Mode) or full floating-point HDR values (Linear Mode with post-upscale tonemapping).

## 2. Mathematical Foundations & Algorithms
### Radiometric Linear vs Perceptual sRGB Space
Physical light transport is linear: doubling photon count doubles radiance. However, human visual sensitivity follows an exponential power law (Weber-Fechner Law):
$$C_{\text{srgb}} \approx C_{\text{linear}}^{1 / 2.2}$$

If edge detection is evaluated directly in linear space:
- In dark regions ($C_{\text{linear}} \in [0.0, 0.05]$), gradients are minuscule ($\Delta I \approx 0$). Edge detection fails, causing blurry, aliased edges in shadows.
- In bright highlights ($C_{\text{linear}} > 1.0$), gradients are enormous, causing excessive ringing and dark haloing around light sources.

Evaluating edge detection in **Perceptual Space** distributes contrast evenly across the dynamic range:
$$Y_{\text{perceptual}} = 0.2126 R_{\text{srgb}} + 0.7152 G_{\text{srgb}} + 0.0722 B_{\text{srgb}}$$

### ACES Tonemapping Curve (Narkowicz 2015 Fit)
Transforms high dynamic range linear values $x \in [0, \infty)$ into bounded display values $[0, 1]$:
$$f(x) = \frac{x(2.51x + 0.03)}{x(2.43x + 0.59) + 0.14}$$

### Pipeline Architecture Comparison
```
Option A: Perceptual SDR Pipeline (Standard)
[Scene HDR] -> [Tonemap + sRGB Encode] -> LowRes SDR -> [MetalFX Perceptual Mode] -> HighRes SDR Display

Option B: Linear HDR Pipeline (Studio Grade)
[Scene HDR] -> LowRes FP16 Linear -> [MetalFX Linear Mode] -> HighRes FP16 Linear -> [Tonemap] -> Display
```

## 3. Metal & C++ Blueprint

### 1. Studio-Grade Linear HDR Scaler Descriptor
```cpp
MTLFX::SpatialScalerDescriptor* pDesc = MTLFX::SpatialScalerDescriptor::alloc()->init();
pDesc->setInputWidth(inW);
pDesc->setInputHeight(inH);
pDesc->setOutputWidth(outW);
pDesc->setOutputHeight(outH);

// 16-bit floating point high dynamic range format
pDesc->setColorTextureFormat(MTL::PixelFormatRGBA16Float);
pDesc->setOutputTextureFormat(MTL::PixelFormatRGBA16Float);

// Instruct MetalFX to convert to perceptual space internally for edge detection
// and return linear FP16 output
pDesc->setColorProcessingMode(MTLFX::SpatialScalerColorProcessingModeLinear);

MTLFX::SpatialScaler* pHDRScaler = pDesc->newSpatialScaler(pDevice);
pDesc->release();
```

### 2. High-Res Tonemapping MSL Shader (Post-Upscale)
```metal
#include <metal_stdlib>
using namespace metal;

struct VertexOut {
    float4 position [[position]];
    float2 texCoord;
};

// ACES Film Tone Mapping
float3 acesTonemap(float3 x) {
    const float a = 2.51f;
    const float b = 0.03f;
    const float c = 2.43f;
    const float d = 0.59f;
    const float e = 0.14f;
    return saturate((x * (a * x + b)) / (x * (c * x + d) + e));
}

fragment float4 tonemapFragment(VertexOut in [[stage_in]],
                                texture2d<float> upscaledHDRTexture [[texture(0)]],
                                sampler smp [[sampler(0)]]) {
    float4 hdrColor = upscaledHDRTexture.sample(smp, in.texCoord);
    
    // 1. Apply ACES tonemapping
    float3 ldrColor = acesTonemap(hdrColor.rgb);
    
    // 2. Hardware sRGB output handles gamma encoding when target is BGRA8Unorm_sRGB
    return float4(ldrColor, 1.0);
}
```

### 3. Format Comparison Matrix
| Configuration | Color Texture Format | Scaler Mode | Output Format | Pros / Cons |
| :--- | :--- | :--- | :--- | :--- |
| **SDR Perceptual** | `RGBA8Unorm` | `Perceptual` | `BGRA8Unorm_sRGB` | Minimum memory footprint; low bandwidth; SDR only. |
| **HDR Linear** | `RGBA16Float` | `Linear` | `RGBA16Float` | Preserves HDR specular details; requires post-tonemap pass. |

## 4. Step-by-Step Implementation Checklist
- [ ] Determine color pipeline: SDR Perceptual or HDR Linear.
- [ ] For SDR: Ensure the scene fragment shader outputs tonemapped SDR colors or uses `RGBA8Unorm_sRGB`.
- [ ] Set `setColorProcessingMode(MTLFX::SpatialScalerColorProcessingModePerceptual)`.
- [ ] For HDR: Allocate low-res color target as `MTL::PixelFormatRGBA16Float`.
- [ ] Set `setColorProcessingMode(MTLFX::SpatialScalerColorProcessingModeLinear)` and output format as `RGBA16Float`.
- [ ] Implement a fullscreen blit pass with ACES tonemapping to map upscaled FP16 HDR pixels to the presentation drawable.
- [ ] Verify that high-contrast edges in shadows remain sharp without dark halo artifacts.

## 5. Common Pitfalls & Debugging Checklist
- **Washed Out or Over-Saturated Colors**:
  - *Cause*: Applying gamma encoding twice (e.g. manual `pow(c, 1/2.2)` in shader while writing to an `sRGB` pixel format target, which applies gamma a second time in hardware).
  - *Fix*: If writing to `MTL::PixelFormatBGRA8Unorm_sRGB`, output linear values from the shader and let hardware write logic handle gamma encoding.
- **Dark Ringing Halos around Highlights**:
  - *Cause*: Using `ColorProcessingModeLinear` on a texture that already has gamma correction baked in.
  - *Fix*: Only use `ColorProcessingModeLinear` on linear FP16/FP32 HDR render targets.
