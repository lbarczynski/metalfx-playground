# TASK-05: Textures, Samplers & UV Mapping

## 1. Concept & Mental Model
Textures store discrete 2D spatial pixel data (texels) on the GPU, sampled across continuous normalized UV coordinate space using hardware filtering units (samplers) that handle interpolation, wrapping, and mipmapping. In OpenGL, textures and sampling state are bound via `glActiveTexture`, `glBindTexture`, and `glBindSampler`; in Metal, `MTL::Texture` represents the GPU allocation and formatted view, while immutable `MTL::SamplerState` governs addressing and filtering modes.

## 2. Mathematical Foundations & Algorithms
### Normalized UV Coordinates
Metal texture coordinate space maps $(u, v) \in [0.0, 1.0]^2$ with $(0.0, 0.0)$ located at the **top-left** texel:
$$\text{Texel } X = u \cdot (W - 1)$$
$$\text{Texel } Y = v \cdot (H - 1)$$

*(Difference from OpenGL: OpenGL texture coordinates place $(0.0, 0.0)$ at the bottom-left).*

### Procedural Checkerboard Function
For an integer texel coordinate $(x, y) \in [0, W-1] \times [0, H-1]$ with square check size $S$:
$$\text{Check}(x, y) = \left( \left\lfloor \frac{x}{S} \right\rfloor + \left\lfloor \frac{y}{S} \right\rfloor \right) \bmod 2$$
$$\mathbf{C}(x, y) = \begin{cases} \mathbf{C}_{\text{primary}} & \text{if Check}(x, y) = 0 \\ \mathbf{C}_{\text{secondary}} & \text{if Check}(x, y) = 1 \end{cases}$$

### Filtering Algorithms
1. **Nearest Neighbor**:
   $$T(u, v) = T\left[\lfloor u W \rfloor, \lfloor v H \rfloor\right]$$
2. **Bilinear Filtering**:
   For fractional coordinate offsets $\alpha = \text{frac}(u W - 0.5)$, $\beta = \text{frac}(v H - 0.5)$:
   $$T(u, v) = (1-\alpha)(1-\beta)T_{00} + \alpha(1-\beta)T_{10} + (1-\alpha)\beta T_{01} + \alpha\beta T_{11}$$

### Row Pitch & Stride Alignment
$$\text{BytesPerRow} = \text{Width} \times \text{BytesPerPixel}$$
For `MTL::PixelFormatRGBA8Unorm`, $\text{BytesPerPixel} = 4$.

## 3. Metal & C++ Blueprint

### 1. Textured Vertex Format
```cpp
#include <simd/simd.h>

struct TexturedVertex {
    simd::float2 position; // NDC coordinate
    simd::float2 texCoord; // UV coordinate [0.0, 1.0]
};
```

### 2. Procedural Texture Generation & `std::span` Upload (C++)
```cpp
#include <Metal/Metal.hpp>
#include <span>
#include <vector>
#include <cstdint>

void uploadTexture2D(MTL::Texture* pTexture, std::span<const uint32_t> pixels, uint32_t width, uint32_t height) {
    MTL::Region region = MTL::Region::Make2D(0, 0, width, height);
    NS::UInteger bytesPerRow = width * sizeof(uint32_t);
    pTexture->replaceRegion(region, 0, pixels.data(), bytesPerRow);
}

MTL::Texture* createCheckerboardTexture(MTL::Device* pDevice, uint32_t width, uint32_t height) {
    MTL::TextureDescriptor* pDesc = MTL::TextureDescriptor::texture2DDescriptor(
        MTL::PixelFormatRGBA8Unorm,
        width,
        height,
        false // no mipmaps for base task
    );
    pDesc->setUsage(MTL::TextureUsageShaderRead);
    pDesc->setStorageMode(MTL::StorageModeShared);

    MTL::Texture* pTexture = pDevice->newTexture(pDesc);
    pDesc->release();

    // Generate 2D RGBA checkerboard data
    // On Apple Silicon (Little-Endian ARM64), 0xFF202020 in memory stores bytes [0x20, 0x20, 0x20, 0xFF],
    // perfectly matching [R, G, B, A] for MTL::PixelFormatRGBA8Unorm.
    std::vector<uint32_t> pixels(width * height);
    constexpr uint32_t checkSize = 16;
    constexpr uint32_t colorA = 0xFFFFFFFF; // White
    constexpr uint32_t colorB = 0xFF202020; // Dark Gray

    for (uint32_t y = 0; y < height; ++y) {
        for (uint32_t x = 0; x < width; ++x) {
            uint32_t mask = ((x / checkSize) + (y / checkSize)) & 1;
            pixels[y * width + x] = mask ? colorA : colorB;
        }
    }

    uploadTexture2D(pTexture, std::span<const uint32_t>(pixels), width, height);

    return pTexture;
}
```

### 3. Sampler State Construction
```cpp
MTL::SamplerState* createSampler(MTL::Device* pDevice, MTL::SamplerMinMagFilter filterMode) {
    MTL::SamplerDescriptor* pDesc = MTL::SamplerDescriptor::alloc()->init();
    pDesc->setMinFilter(filterMode);
    pDesc->setMagFilter(filterMode);
    pDesc->setSAddressMode(MTL::SamplerAddressModeRepeat);
    pDesc->setTAddressMode(MTL::SamplerAddressModeRepeat);

    MTL::SamplerState* pSampler = pDevice->newSamplerState(pDesc);
    pDesc->release();
    return pSampler;
}
```

### 4. MSL Texture Sampling Shader
```metal
#include <metal_stdlib>
using namespace metal;

struct VertexInput {
    float2 position [[attribute(0)]];
    float2 texCoord [[attribute(1)]];
};

struct RasterizerData {
    float4 position [[position]];
    float2 texCoord;
};

vertex RasterizerData vertexShader(VertexInput in [[stage_in]]) {
    RasterizerData out;
    out.position = float4(in.position, 0.0, 1.0);
    out.texCoord = in.texCoord;
    return out;
}

fragment float4 fragmentShader(RasterizerData in [[stage_in]],
                               texture2d<float> colorTexture [[texture(0)]],
                               sampler textureSampler [[sampler(0)]]) {
    return colorTexture.sample(textureSampler, in.texCoord);
}
```

### 5. Encoding Quad Draw with Texture & Sampler
```cpp
// Fullscreen or centered quad (2 triangles, 6 vertices)
const TexturedVertex quadVertices[6] = {
    // Triangle 1
    { { -0.8f,  0.8f }, { 0.0f, 0.0f } },
    { { -0.8f, -0.8f }, { 0.0f, 1.0f } },
    { {  0.8f, -0.8f }, { 1.0f, 1.0f } },
    // Triangle 2
    { { -0.8f,  0.8f }, { 0.0f, 0.0f } },
    { {  0.8f, -0.8f }, { 1.0f, 1.0f } },
    { {  0.8f,  0.8f }, { 1.0f, 0.0f } }
};

enc->setRenderPipelineState(pPipelineState);
enc->setVertexBuffer(pQuadVertexBuffer, 0, 0);
enc->setFragmentTexture(pTexture, 0);
enc->setFragmentSamplerState(pSamplerLinear, 0); // or pSamplerNearest
enc->drawPrimitives(MTL::PrimitiveTypeTriangle, NS::UInteger(0), NS::UInteger(6));
```

## 4. Step-by-Step Implementation Checklist
- [ ] Define `TexturedVertex` struct containing `simd::float2 position` and `simd::float2 texCoord`.
- [ ] Create 6-vertex textured quad buffer matching NDC $[-0.8, 0.8]$ and UV $[0.0, 1.0]$.
- [ ] Generate 256x256 RGBA8 procedural checkerboard texture in CPU memory.
- [ ] Instantiate `MTL::Texture` with `MTL::TextureUsageShaderRead`.
- [ ] Upload raw bytes to texture via `texture->replaceRegion()`.
- [ ] Construct two `MTL::SamplerState` objects: `Nearest` and `Linear`.
- [ ] Write MSL shader declaring `texture2d<float> [[texture(0)]]` and `sampler [[sampler(0)]]`.
- [ ] In `drawInMTKView()`, bind vertex buffer, texture at index 0, and sampler at index 0.
- [ ] Run application; verify crisp checkerboard rendering and test toggling between nearest and linear sampling.

## 5. Common Pitfalls & Debugging Checklist
- **Texture Diagonal Skew / Distorted Image**:
  - *Cause*: Mismatch between `bytesPerRow` and actual row stride (e.g. using 3 bytes per pixel for RGB without padding, rather than 4-byte RGBA).
  - *Fix*: Always use 4-byte aligned formats (`MTL::PixelFormatRGBA8Unorm` or `MTL::PixelFormatBGRA8Unorm`).
- **Upside-Down Texture Sampling**:
  - *Cause*: Mapping $v=0.0$ to the bottom edge (OpenGL convention) instead of the top edge (Metal convention).
  - *Fix*: Set top vertices to $v=0.0$ and bottom vertices to $v=1.0$.
- **Missing `MTL::TextureUsageShaderRead` Flag**:
  - *Symptom*: Sampler returns black or validation layer logs usage error.
  - *Fix*: Set `pDesc->setUsage(MTL::TextureUsageShaderRead)` prior to `device->newTexture(pDesc)`.
- **Resource Leaks of Descriptors**:
  - *Fix*: Call `pDesc->release()` immediately after creating `MTL::Texture` or `MTL::SamplerState`.
