# TASK-18A: Velocity Buffer & Multiple Render Targets (MRT)

## 1. Concept & Mental Model

### What are Motion Vectors?
In video compression (HEVC/AV1), motion vectors estimate where blocks of pixels moved. In real-time 3D graphics, motion vectors are calculated **analytically** per-vertex and per-pixel in shaders.

To avoid running two separate rasterization passes (which would double geometry vertex processing), modern GPUs use **Multiple Render Targets (MRT)**:
- A single draw call outputs both surface color and 2D velocity vectors into two separate textures at the same time:
  - **Color Attachment 0**: Shaded scene color (`RGBA16Float` or `BGRA8Unorm_sRGB`).
  - **Color Attachment 1**: 2D Screen-space velocity vectors (`RG16Float`).

---

## 2. Metal & C++ Blueprint

### 1. Allocating the Motion Vector Texture (`src/TemporalUpscaler.hpp`)
```cpp
MTL::TextureDescriptor* motionDesc = MTL::TextureDescriptor::texture2DDescriptor(
    MTL::PixelFormatRG16Float,
    inputWidth,
    inputHeight,
    false
);
motionDesc->setUsage(MTL::TextureUsageRenderTarget | MTL::TextureUsageShaderRead);
motionDesc->setStorageMode(MTL::StorageModePrivate);

MTL::Texture* pMotionTexture = pDevice->newTexture(motionDesc);
motionDesc->release();
```

### 2. Configuring Pipeline State for Dual Color Attachments
```cpp
MTL::RenderPipelineDescriptor* pipelineDesc = MTL::RenderPipelineDescriptor::alloc()->init();
pipelineDesc->setVertexFunction(pLitVertexFunc);
pipelineDesc->setFragmentFunction(pLitFragmentFunc);

// Attachment 0: Color
pipelineDesc->colorAttachments()->object(0)->setPixelFormat(MTL::PixelFormatBGRA8Unorm_sRGB);

// Attachment 1: Motion Vectors
pipelineDesc->colorAttachments()->object(1)->setPixelFormat(MTL::PixelFormatRG16Float);

// Depth
pipelineDesc->setDepthAttachmentPixelFormat(MTL::PixelFormatDepth32Float);

MTL::RenderPipelineState* pMRTPipeline = pDevice->newRenderPipelineState(pipelineDesc, &pError);
pipelineDesc->release();
```

### 3. Setting Up the MRT Render Pass Descriptor
```cpp
// In renderFrame():
MTL::RenderPassDescriptor* passDesc = MTL::RenderPassDescriptor::renderPassDescriptor();

// Attachment 0: Color
auto rt0 = passDesc->colorAttachments()->object(0);
rt0->setTexture(pLowResColorTexture);
rt0->setLoadAction(MTL::LoadActionClear);
rt0->setStoreAction(MTL::StoreActionStore);
rt0->setClearColor(MTL::ClearColor::Make(0.05, 0.05, 0.08, 1.0));

// Attachment 1: Motion Vectors
auto rt1 = passDesc->colorAttachments()->object(1);
rt1->setTexture(pLowResMotionTexture);
rt1->setLoadAction(MTL::LoadActionClear);
rt1->setStoreAction(MTL::StoreActionStore);
rt1->setClearColor(MTL::ClearColor::Make(0.0, 0.0, 0.0, 0.0)); // 0 velocity = stationary

// Depth Attachment
auto depthAttachment = passDesc->depthAttachment();
depthAttachment->setTexture(pLowResDepthTexture);
depthAttachment->setLoadAction(MTL::LoadActionClear);
depthAttachment->setStoreAction(MTL::StoreActionStore);
depthAttachment->setClearDepth(0.0); // Reversed-Z
```

---

## 3. Step-by-Step Implementation Checklist
- [ ] Allocate `pLowResMotionTexture` with `MTL::PixelFormatRG16Float` and `RenderTarget | ShaderRead`.
- [ ] Update `RenderPipelineDescriptor` declaring color attachment 0 and color attachment 1 (`RG16Float`).
- [ ] In `renderFrame()`, attach both color textures to `passDesc->colorAttachments()`.
- [ ] Set `setStoreAction(MTL::StoreActionStore)` on both color attachments.
- [ ] Execute draw call and inspect in Xcode Frame Debugger to verify two render targets are bound.

---

## 4. Definition of Done & Commit Message
- **DoD**: Scene rasterization executes under an MRT render pass outputting simultaneously to a color attachment and an `RG16Float` motion vector attachment.
- **Commit**:
  ```bash
  git add src/Renderer.cpp src/Renderer.hpp
  git commit -m "feat(pipeline): configure mrt render pass for color and motion targets"
  ```
