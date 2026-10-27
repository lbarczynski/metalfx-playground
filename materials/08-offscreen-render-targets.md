# TASK-08: Offscreen Render Targets (Low-Res Framebuffer)

## 1. Concept & Mental Model
Offscreen rendering decouples simulation/rendering resolution from final display resolution by rendering the scene into an offscreen intermediate texture, followed by a post-process blit pass that samples the texture onto the presentation drawable. In OpenGL, this requires creating and binding a Framebuffer Object (`glGenFramebuffers`, `glBindFramebuffer`); in Metal, it simply involves setting an offscreen `MTL::Texture` as the color attachment of an initial render pass, then binding that texture as an input to a secondary presentation pass.

## 2. Mathematical Foundations & Algorithms
### Dynamic Resolution Scaling Factor ($s$)
Given native display resolution $(W_{\text{target}}, H_{\text{target}})$ and scale factor $s \in (0.0, 1.0]$:
$$W_{\text{render}} = \max(1, \lfloor W_{\text{target}} \cdot s \rfloor)$$
$$H_{\text{render}} = \max(1, \lfloor H_{\text{target}} \cdot s \rfloor)$$
$$\text{Pixel Ratio} = \frac{W_{\text{render}} \cdot H_{\text{render}}}{W_{\text{target}} \cdot H_{\text{target}}} = s^2$$

*(At $s = 0.5$, rendering requires only $25\%$ of full-resolution fill-rate, cutting shading cost by $4\times$).*

### Fullscreen Triangle Trick (Zero Vertex Buffer)
Instead of allocating a 6-vertex quad buffer, a single triangle covering the entire $[-1.0, 3.0]$ NDC space can be generated purely from `vertex_id` in MSL:
$$x_{\text{ndc}} = (id == 1) \ ? \ 3.0 : -1.0$$
$$y_{\text{ndc}} = (id == 2) \ ? \ 3.0 : -1.0$$
$$u = (id == 1) \ ? \ 2.0 : 0.0, \quad v = (id == 2) \ ? \ 2.0 : 0.0$$

## 3. Metal & C++ Blueprint

### 1. Offscreen Texture Descriptors
```cpp
#include <Metal/Metal.hpp>

// Low-resolution offscreen render target
const uint32_t lowResW = 960;  // 50% of 1920
const uint32_t lowResH = 540;  // 50% of 1080

MTL::TextureDescriptor* offscreenDesc = MTL::TextureDescriptor::texture2DDescriptor(
    MTL::PixelFormatRGBA8Unorm,
    lowResW,
    lowResH,
    false
);
// Must have RenderTarget (Pass 1) and ShaderRead (Pass 2)
offscreenDesc->setUsage(MTL::TextureUsageRenderTarget | MTL::TextureUsageShaderRead);
offscreenDesc->setStorageMode(MTL::StorageModePrivate); // Best performance on Apple Silicon

MTL::Texture* pOffscreenTexture = pDevice->newTexture(offscreenDesc);
offscreenDesc->release();
```

### 2. Fullscreen Blit MSL Shaders
```metal
#include <metal_stdlib>
using namespace metal;

struct BlitVertexOut {
    float4 position [[position]];
    float2 texCoord;
};

// Generates fullscreen triangle without vertex buffer
vertex BlitVertexOut blitVertex(uint vertexID [[vertex_id]]) {
    BlitVertexOut out;
    float2 grid = float2((vertexID << 1) & 2, vertexID & 2);
    out.position = float4(grid * float2(2.0, -2.0) + float2(-1.0, 1.0), 0.0, 1.0);
    out.texCoord = grid;
    return out;
}

fragment float4 blitFragment(BlitVertexOut in [[stage_in]],
                             texture2d<float> lowResTexture [[texture(0)]],
                             sampler smp [[sampler(0)]]) {
    return lowResTexture.sample(smp, in.texCoord);
}
```

### 3. Dual-Pass Command Buffer Encoding
```cpp
// ==========================================
// PASS 1: Render 3D Scene into Low-Res Target
// ==========================================
MTL::RenderPassDescriptor* pass1Desc = MTL::RenderPassDescriptor::renderPassDescriptor();
pass1Desc->colorAttachments()->object(0)->setTexture(pOffscreenTexture);
pass1Desc->colorAttachments()->object(0)->setLoadAction(MTL::LoadActionClear);
pass1Desc->colorAttachments()->object(0)->setStoreAction(MTL::StoreActionStore); // Retain pixels for Pass 2
pass1Desc->colorAttachments()->object(0)->setClearColor(MTL::ClearColor::Make(0.1, 0.2, 0.3, 1.0));

MTL::RenderCommandEncoder* enc1 = cmd->renderCommandEncoder(pass1Desc);
// [Encode 3D scene geometry here...]
enc1->endEncoding();

// ==========================================
// PASS 2: Blit / Upscale to Native Drawable
// ==========================================
MTL::RenderPassDescriptor* pass2Desc = pView->currentRenderPassDescriptor();
CA::MetalDrawable* drawable = pView->currentDrawable();

if (pass2Desc && drawable) {
    pass2Desc->colorAttachments()->object(0)->setLoadAction(MTL::LoadActionDontCare);
    pass2Desc->colorAttachments()->object(0)->setStoreAction(MTL::StoreActionStore);

    MTL::RenderCommandEncoder* enc2 = cmd->renderCommandEncoder(pass2Desc);
    enc2->setRenderPipelineState(pBlitPipelineState);
    enc2->setCullMode(MTL::CullModeNone); // CRITICAL: Prevent CW procedural triangle from being culled
    enc2->setFragmentTexture(pOffscreenTexture, 0);
    enc2->setFragmentSamplerState(pBilinearSampler, 0);
    
    // Draw 1 triangle (3 vertices generated procedurally in shader)
    enc2->drawPrimitives(MTL::PrimitiveTypeTriangle, NS::UInteger(0), NS::UInteger(3));
    enc2->endEncoding();

    cmd->presentDrawable(drawable);
}
cmd->commit();
```

## 4. Step-by-Step Implementation Checklist
- [ ] Allocate offscreen `MTL::Texture` with `RenderTarget | ShaderRead` usage and `StorageModePrivate`.
- [ ] Compile blit vertex shader using procedural fullscreen triangle indexing (`[[vertex_id]]`).
- [ ] Create secondary blit `MTL::RenderPipelineState` targeting `pView->colorPixelFormat()`.
- [ ] Create a linear `MTL::SamplerState` for smooth scaling.
- [ ] In `drawInMTKView()`:
  - Configure Pass 1 descriptor pointing to `pOffscreenTexture` with `StoreActionStore`.
  - Encode scene rendering commands in Pass 1.
  - Configure Pass 2 descriptor pointing to native drawable.
  - Bind `pOffscreenTexture` at fragment texture index 0 and linear sampler at index 0.
  - Draw fullscreen triangle: `enc2->drawPrimitives(MTL::PrimitiveTypeTriangle, 0, 3)`.
- [ ] Run application; verify scene renders at 50% scale and seamlessly magnifies to fill window.

## 5. Common Pitfalls & Debugging Checklist
- **Missing `StoreActionStore` on Offscreen Attachment**:
  - *Symptom*: Pass 2 samples black or corrupt pixels from `pOffscreenTexture`.
  - *Cause*: `setStoreAction(MTL::StoreActionDontCare)` discards tile cache data at the end of Pass 1.
  - *Fix*: Set `StoreActionStore` on Pass 1's color attachment.
- **Missing `MTL::TextureUsageShaderRead`**:
  - *Error*: Validation layer asserts: `Texture usage (MTLTextureUsageRenderTarget) does not permit shader read`.
  - *Fix*: Bitwise OR flags: `TextureUsageRenderTarget | TextureUsageShaderRead`.
- **Memory Allocation on Every Frame**:
  - *Cause*: Reallocating `pOffscreenTexture` inside `drawInMTKView()`.
  - *Fix*: Cache the offscreen texture and only reallocate when window size changes.
