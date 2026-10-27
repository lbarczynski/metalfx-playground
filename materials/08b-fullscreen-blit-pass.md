# TASK-08B: Fullscreen Quad Blit Pass

## 1. Concept & Mental Model

### The Procedural Fullscreen Triangle Trick
To display an offscreen texture on the window, you need to draw a quad across the entire screen. Traditional approaches allocate a 4-vertex quad with a vertex buffer.

High-performance real-time renderers use a clever optimization: **The Single Fullscreen Triangle**.
By generating 3 vertices directly in the vertex shader using `[[vertex_id]]`:
- No vertex buffer memory allocation or CPU-to-GPU bandwidth needed.
- A single large triangle covers the entire $[-1, 1]$ screen area.
- Eliminates the diagonal seam where two triangles meet, preventing rasterizer precision artifacts.

```
       (-1, 3) [UV: 0, -1]
          ▲
          │ ╲
          │   ╲
          │     ╲
       ( -1, 1)───(1, 1) [UV: 1, 0]  <── Screen Viewport Area [-1, 1]
          │   |     │ ╲
          │   |     │   ╲
       (-1,-1)───(1,-1)───(3, -1) [UV: 2, 1]
```

---

## 2. MSL Fullscreen Shaders

```metal
#include <metal_stdlib>
using namespace metal;

struct BlitVertexOut {
    float4 position [[position]];
    float2 uv;
};

// Generates an oversized triangle covering NDC [-1, 1]
vertex BlitVertexOut fullscreenVertexMain(uint vertexID [[vertex_id]]) {
    BlitVertexOut out;
    // Map vertexID (0, 1, 2) to (x, y) coordinates
    float2 grid = float2((vertexID << 1) & 2, vertexID & 2);
    out.position = float4(grid * 2.0f - 1.0f, 0.0f, 1.0f);
    // Metal UV coordinates: top-left (0, 0), bottom-right (1, 1)
    out.uv = float2(grid.x, 1.0f - grid.y);
    return out;
}

fragment float4 fullscreenFragmentMain(
    BlitVertexOut in [[stage_in]],
    texture2d<float> sourceTexture [[texture(0)]],
    sampler blitSampler [[sampler(0)]]
) {
    return sourceTexture.sample(blitSampler, in.uv);
}
```

---

## 3. Metal & C++ Blueprint

### 1. Blit Pipeline Initialization
```cpp
// Create pipeline with no vertex descriptor (procedural vertex generation)
MTL::RenderPipelineDescriptor* blitDesc = MTL::RenderPipelineDescriptor::alloc()->init();
blitDesc->setVertexFunction(pFullscreenVertFunc);
blitDesc->setFragmentFunction(pFullscreenFragFunc);
blitDesc->colorAttachments()->object(0)->setPixelFormat(MTL::PixelFormatBGRA8Unorm_sRGB);
MTL::RenderPipelineState* pBlitPipeline = pDevice->newRenderPipelineState(blitDesc, &pError);
blitDesc->release();
```

### 2. Multi-Pass Frame Orchestration in `renderFrame()`
```cpp
// ========================================================
// PASS 1: Scene Render (Offscreen 540p)
// ========================================================
MTL::RenderCommandEncoder* enc1 = cmd->renderCommandEncoder(offscreenPassDesc);
enc1->setRenderPipelineState(p3DPipeline);
// [Draw 3D spinning cube...]
enc1->endEncoding(); // FLUSH PASS 1

// ========================================================
// PASS 2: Fullscreen Presentation (Offscreen 540p -> Window)
// ========================================================
MTL::RenderPassDescriptor* presentPass = MTL::RenderPassDescriptor::renderPassDescriptor();
auto attach = presentPass->colorAttachments()->object(0);
attach->setTexture(pDrawable->texture());
attach->setLoadAction(MTL::LoadActionDontCare); // Overwrite entire screen
attach->setStoreAction(MTL::StoreActionStore);

MTL::RenderCommandEncoder* enc2 = cmd->renderCommandEncoder(presentPass);
enc2->setRenderPipelineState(pBlitPipeline);
enc2->setFragmentTexture(pLowResColorTexture, 0);
enc2->setFragmentSamplerState(pLinearSampler, 0);

// Draw 3 procedural vertices with NO vertex buffer bound
enc2->drawPrimitives(MTL::PrimitiveTypeTriangle, 0, 3);
enc2->endEncoding();

cmd->presentDrawable(pDrawable);
cmd->commit();
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Add `fullscreenVertexMain` and `fullscreenFragmentMain` to MSL.
- [ ] Create `MTL::RenderPipelineState` for the blit pass (no vertex descriptor).
- [ ] In `renderFrame()`, sequence Pass 1 (Scene $\to$ LowRes), close encoder with `endEncoding()`.
- [ ] Sequence Pass 2 (LowRes $\to$ `pDrawable->texture()`), calling `drawPrimitives(MTL::PrimitiveTypeTriangle, 0, 3)`.
- [ ] Run application: The spinning cube renders at 540p and is displayed scaled up to the full native window.

---

## 5. Definition of Done & Commit Message
- **DoD**: 3D scene renders to a low-res offscreen texture and is presented to the window via a procedural fullscreen blit pass in a single command buffer.
- **Commit**:
  ```bash
  git add src/Renderer.cpp src/Renderer.hpp src/Shaders.metal
  git commit -m "feat(renderer): implement fullscreen blit pass for offscreen target"
  ```
