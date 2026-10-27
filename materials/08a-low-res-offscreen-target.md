# TASK-08A: Low-Resolution Offscreen Render Target

## 1. Concept & Mental Model

### Why Offscreen Render Targets?
Modern displays run at high pixel densities (e.g. 4K, 5K, or Apple Retina displays with $3024 \times 1964$ pixels). Rasterizing complex 3D geometry and shading millions of pixels at native resolution stresses the GPU and drains battery life.

The solution used by modern game engines is **Offscreen Rendering**:
1. Render the 3D scene at a lower internal resolution (e.g. $960 \times 540$ or $1280 \times 720$).
2. Store the result in an offscreen texture in GPU memory.
3. Scale the image up to the full window size in a subsequent pass.

This is the architectural prerequisite for **MetalFX Spatial and Temporal upscaling**!

---

## 2. Metal Texture Usage & Storage Invariants

For a texture to act as both a **render destination** for Pass 1 and an **input image** for Pass 2, it requires specific usage flags:

| Setting | Required Value | Purpose |
|---|---|---|
| **Pixel Format** | `MTL::PixelFormatBGRA8Unorm_sRGB` or `RGBA16Float` | Color channel layout |
| **Usage Flags** | `MTL::TextureUsageRenderTarget \| MTL::TextureUsageShaderRead` | Write in Pass 1, Read in Pass 2 |
| **Storage Mode** | `MTL::StorageModePrivate` | Fastest on-chip GPU tile memory |
| **Store Action** | `MTL::StoreActionStore` | Flushes tile cache lines to VRAM |

> [!CAUTION]
> If you leave `setStoreAction(MTL::StoreActionDontCare)`, the Apple Silicon tile memory controller discards the rendered pixels at the end of Pass 1, resulting in a black screen in Pass 2!

---

## 3. Metal & C++ Blueprint

### 1. Allocating Low-Resolution Offscreen Targets
```cpp
// Allocate 540p offscreen color texture
constexpr uint32_t kLowResWidth = 960;
constexpr uint32_t kLowResHeight = 540;

MTL::TextureDescriptor* colorDesc = MTL::TextureDescriptor::texture2DDescriptor(
    MTL::PixelFormatBGRA8Unorm_sRGB,
    kLowResWidth,
    kLowResHeight,
    false
);
colorDesc->setUsage(MTL::TextureUsageRenderTarget | MTL::TextureUsageShaderRead);
colorDesc->setStorageMode(MTL::StorageModePrivate);
MTL::Texture* pLowResColorTexture = pDevice->newTexture(colorDesc);
colorDesc->release();

// Allocate matching 540p depth texture
MTL::TextureDescriptor* depthDesc = MTL::TextureDescriptor::texture2DDescriptor(
    MTL::PixelFormatDepth32Float,
    kLowResWidth,
    kLowResHeight,
    false
);
depthDesc->setUsage(MTL::TextureUsageRenderTarget);
depthDesc->setStorageMode(MTL::StorageModePrivate);
MTL::Texture* pLowResDepthTexture = pDevice->newTexture(depthDesc);
depthDesc->release();
```

### 2. Scene Pass Targeting Offscreen Textures
```cpp
// In renderFrame():
MTL::RenderPassDescriptor* offscreenPass = MTL::RenderPassDescriptor::renderPassDescriptor();

// Color Attachment 0
auto colorAttachment = offscreenPass->colorAttachments()->object(0);
colorAttachment->setTexture(pLowResColorTexture);
colorAttachment->setLoadAction(MTL::LoadActionClear);
colorAttachment->setStoreAction(MTL::StoreActionStore); // CRITICAL: Retain for next pass
colorAttachment->setClearColor(MTL::ClearColor::Make(0.1, 0.1, 0.15, 1.0));

// Depth Attachment
auto depthAttachment = offscreenPass->depthAttachment();
depthAttachment->setTexture(pLowResDepthTexture);
depthAttachment->setLoadAction(MTL::LoadActionClear);
depthAttachment->setStoreAction(MTL::StoreActionDontCare); // Depth not needed after scene pass
depthAttachment->setClearDepth(1.0);

// Encode 3D scene geometry
MTL::RenderCommandEncoder* enc = cmd->renderCommandEncoder(offscreenPass);
// [Draw 3D spinning cube...]
enc->endEncoding(); // Closes pass and flushes tile cache
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Define low-res dimensions: $W = 960, H = 540$.
- [ ] Create color texture with `MTL::TextureUsageRenderTarget | MTL::TextureUsageShaderRead`.
- [ ] Create matching depth texture with dimensions $960 \times 540$.
- [ ] Configure `offscreenPass` with `colorAttachment->setStoreAction(MTL::StoreActionStore)`.
- [ ] Encode the 3D spinning cube pass into `pLowResColorTexture`.
- [ ] Capture a frame in Xcode Frame Debugger to verify the scene renders at $960 \times 540$.

---

## 5. Definition of Done & Commit Message
- **DoD**: 3D scene renders cleanly into a $960 \times 540$ offscreen render target in GPU private memory with `StoreActionStore` verified via Xcode capture.
- **Commit**:
  ```bash
  git add src/Renderer.cpp src/Renderer.hpp
  git commit -m "feat(offscreen): create low-resolution offscreen render target"
  ```
