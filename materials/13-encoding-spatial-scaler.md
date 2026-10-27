# TASK-13: Encoding the Spatial Scaler Pass

## 1. Concept & Mental Model

### Command Buffer Level Encoding
Unlike graphics passes that operate inside an `MTL::RenderCommandEncoder`, or compute kernels inside an `MTL::ComputeCommandEncoder`, MetalFX scalers encode **directly onto the `MTL::CommandBuffer`**:

```cpp
pScaler->encodeToCommandBuffer(cmd);
```

Under the hood, Apple Silicon inserts internal compute dispatches, memory fences, and cache synchronization barriers. 

> [!CAUTION]
> You **cannot** call `pScaler->encodeToCommandBuffer(cmd)` while any other command encoder is currently active on that command buffer. You must explicitly call `enc->endEncoding()` on Pass 1 before encoding the MetalFX pass.

---

## 2. Multi-Pass Frame Orchestration

```
[ Command Buffer ]
├── RenderPassEncoder (Pass 1: 3D Scene -> LowRes 540p Texture)
│   ├── setRenderPipelineState(3D)
│   ├── drawIndexedPrimitives(Cube)
│   └── endEncoding()  <── CRITICAL: FLUSHES TILE CACHE LINES TO VRAM
│
├── MetalFX Spatial Pass (Pass 2)
│   ├── pScaler->setColorTexture(LowResTexture)
│   ├── pScaler->setOutputTexture(DrawableTexture)
│   └── pScaler->encodeToCommandBuffer(cmd)
│
└── presentDrawable(pDrawable) & commit()
```

---

## 3. Metal & C++ Blueprint

### Frame Execution (`src/Renderer.cpp`)
```cpp
void renderFrame() {
    NS::AutoreleasePool* pool = NS::AutoreleasePool::alloc()->init();

    CA::MetalDrawable* pDrawable = pLayer->nextDrawable();
    if (!pDrawable) {
        pool->release();
        return;
    }

    MTL::CommandBuffer* cmd = pQueue->commandBuffer();

    // ========================================================
    // PASS 1: Render 3D Scene to Low-Res Offscreen Target
    // ========================================================
    MTL::RenderPassDescriptor* scenePass = MTL::RenderPassDescriptor::renderPassDescriptor();
    auto colorAttach = scenePass->colorAttachments()->object(0);
    colorAttach->setTexture(pLowResColorTexture);
    colorAttach->setLoadAction(MTL::LoadActionClear);
    colorAttach->setStoreAction(MTL::StoreActionStore); // Retain for MetalFX read
    colorAttach->setClearColor(MTL::ClearColor::Make(0.05, 0.05, 0.08, 1.0));

    auto depthAttach = scenePass->depthAttachment();
    depthAttach->setTexture(pLowResDepthTexture);
    depthAttach->setLoadAction(MTL::LoadActionClear);
    depthAttach->setStoreAction(MTL::StoreActionDontCare);
    depthAttach->setClearDepth(1.0);

    MTL::RenderCommandEncoder* sceneEnc = cmd->renderCommandEncoder(scenePass);
    sceneEnc->setRenderPipelineState(p3DPipeline);
    sceneEnc->setDepthStencilState(pDepthState);
    sceneEnc->setVertexBuffer(pCubeVertexBuffer, 0, 0);
    sceneEnc->setVertexBuffer(pUniformBuffer, 0, 1);
    sceneEnc->drawIndexedPrimitives(MTL::PrimitiveTypeTriangle, 36, MTL::IndexTypeUInt16, pCubeIndexBuffer, 0);
    
    // MUST end encoding before MetalFX starts
    sceneEnc->endEncoding();

    // ========================================================
    // PASS 2: MetalFX Spatial Upscaling Pass
    // ========================================================
    pSpatialScaler->setColorTexture(pLowResColorTexture);
    pSpatialScaler->setOutputTexture(pDrawable->texture());
    pSpatialScaler->encodeToCommandBuffer(cmd);

    // ========================================================
    // PRESENTATION
    // ========================================================
    cmd->presentDrawable(pDrawable);
    cmd->commit();

    pool->release();
}
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Connect `pLowResColorTexture` as color attachment 0 of Pass 1 with `StoreActionStore`.
- [ ] Draw the 3D lit cube into Pass 1 and call `sceneEnc->endEncoding()`.
- [ ] Bind `pLowResColorTexture` to `pSpatialScaler->setColorTexture(...)`.
- [ ] Bind `pDrawable->texture()` to `pSpatialScaler->setOutputTexture(...)`.
- [ ] Encode the upscaler: `pSpatialScaler->encodeToCommandBuffer(cmd)`.
- [ ] Present drawable and commit.
- [ ] Run application: The 3D scene renders at 540p and is upscaled cleanly in real-time to the native window.

---

## 5. Definition of Done & Commit Message
- **DoD**: 3D scene renders into a 540p texture and is upscaled in real-time via MetalFX Spatial to the display without validation layer warnings.
- **Commit**:
  ```bash
  git add src/Renderer.cpp
  git commit -m "feat(metalfx): encode spatial upscaler pass into command buffer"
  ```
