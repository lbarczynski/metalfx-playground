# TASK-05B: Sampler States & Textured Quad Rendering

## 1. Concept & Mental Model

### The Kotlin / Android Bridge
In Android graphics (Canvas or RenderNode), scaling bitmaps uses `Paint.isFilterBitmap` (boolean flag). 

In Metal:
- **Decoupled Sampler State**: Textures contain raw texels, while `MTL::SamplerState` determines *how* the hardware samples them (filtering mode, wrapping behavior, anisotropy).
- **Filtering Modes**:
  - `Nearest`: Samples the closest single texel. Sharp, pixelated, retro aesthetic.
  - `Linear`: Performs weighted bilinear interpolation across 4 adjacent texels. Smooth transitions.
- **UV Coordinates**: Normalized texture space runs from $(0, 0)$ at the top-left to $(1, 1)$ at the bottom-right.

---

## 2. Geometry & UV Mapping

A 2D quad is formed by two adjacent triangles sharing an edge (6 vertices):

```
(-0.5, 0.5) [UV: 0, 0] ──────── (0.5, 0.5) [UV: 1, 0]
       │       ╲                     │
       │         ╲                   │
       │           ╲                 │
       │             ╲               │
(-0.5, -0.5) [UV: 0, 1] ─────── (0.5, -0.5) [UV: 1, 1]
```

### Vertex Definition (`src/Renderer.hpp`)
```cpp
struct TexturedVertex {
    simd::float2 position;
    simd::float2 uv;
};
```

---

## 3. Metal & C++ Blueprint

### 1. Creating Sampler States
```cpp
MTL::SamplerDescriptor* samplerDesc = MTL::SamplerDescriptor::alloc()->init();

// Configure Nearest Sampler
samplerDesc->setMinFilter(MTL::SamplerMinMagFilterNearest);
samplerDesc->setMagFilter(MTL::SamplerMinMagFilterNearest);
samplerDesc->setSAddressMode(MTL::SamplerAddressModeClampToEdge);
samplerDesc->setTAddressMode(MTL::SamplerAddressModeClampToEdge);
MTL::SamplerState* pNearestSampler = pDevice->newSamplerState(samplerDesc);

// Configure Linear Sampler
samplerDesc->setMinFilter(MTL::SamplerMinMagFilterLinear);
samplerDesc->setMagFilter(MTL::SamplerMinMagFilterLinear);
MTL::SamplerState* pLinearSampler = pDevice->newSamplerState(samplerDesc);

samplerDesc->release();
```

### 2. MSL Textured Shaders
```metal
struct TexturedVertexInput {
    float2 position;
    float2 uv;
};

struct TexturedVertexOut {
    float4 position [[position]];
    float2 uv;
};

vertex TexturedVertexOut texturedVertexMain(
    uint vertexID [[vertex_id]],
    constant TexturedVertexInput* vertices [[buffer(0)]]
) {
    TexturedVertexOut out;
    out.position = float4(vertices[vertexID].position, 0.0, 1.0);
    out.uv = vertices[vertexID].uv;
    return out;
}

fragment float4 texturedFragmentMain(
    TexturedVertexOut in [[stage_in]],
    texture2d<float> colorTexture [[texture(0)]],
    sampler colorSampler [[sampler(0)]]
) {
    return colorTexture.sample(colorSampler, in.uv);
}
```

### 3. Encoding in `renderFrame()`
```cpp
enc->setRenderPipelineState(pTexturedPipeline);
enc->setVertexBuffer(pQuadVertexBuffer, 0, 0);
enc->setFragmentTexture(pTexture, 0);
enc->setFragmentSamplerState(s_useLinear ? pLinearSampler : pNearestSampler, 0);

enc->drawPrimitives(MTL::PrimitiveTypeTriangle, 0, 6);
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Define quad vertices with 2D positions and UV coordinates in $[0, 1]$.
- [ ] Create both Nearest and Linear `MTL::SamplerState` objects.
- [ ] Write MSL shaders sampling from `texture(0)` via `sampler(0)`.
- [ ] Bind vertex buffer, texture, and sampler state to the render encoder.
- [ ] Add an interactive key toggle (e.g. Spacebar) to switch between Nearest and Linear filtering.
- [ ] Observe the difference: Nearest shows sharp pixel blocks, while Linear blurs the transitions smoothly.

---

## 5. Definition of Done & Commit Message
- **DoD**: A textured quad renders on screen displaying the checkerboard pattern, with interactive switching between Nearest-neighbor and Bilinear filtering.
- **Commit**:
  ```bash
  git add src/Renderer.cpp src/Renderer.hpp src/Shaders.metal
  git commit -m "feat(renderer): implement sampler states and textured quad rendering"
  ```
