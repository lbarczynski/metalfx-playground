# TASK-20B: Encoding the Temporal Scaler Pass

## 1. Concept & Mental Model

### Active Temporal Reconstruction in Action
When `pTemporalScaler->encodeToCommandBuffer(cmd)` executes:
1. The scaler samples the current frame's color, depth, and motion vector textures.
2. It uses motion vectors to look up where each pixel was located in its internal **history texture** from the previous frame.
3. It performs depth rejection to detect disocclusions (surfaces that were previously hidden).
4. It blends the jittered current sample with reprojected history, accumulating subpixel detail.
5. It writes the reconstructed high-resolution image to the output texture and stores the result in its private history buffer.

The result is astounding: a $960 \times 540$ render reconstructed to $1920 \times 1080$ displays razor-sharp edges and intricate textural detail that matches or exceeds native rendering!

---

## 2. Multi-Pass Frame Orchestration

```
[ Command Buffer ]
├── MRT Scene Pass (Pass 1: 540p)
│   ├── Color RT0 (pLowResColor)     ──> StoreActionStore
│   ├── Motion RT1 (pLowResMotion)   ──> StoreActionStore
│   ├── Depth (pLowResDepth)         ──> StoreActionStore, Clear 0.0 (Rev-Z)
│   ├── Draw Scene Geometry
│   └── endEncoding()  <── CRITICAL FLUSH BARRIER
│
├── MetalFX Temporal Scaler Pass (Pass 2)
│   ├── Bind 4 Textures (Color, Depth, Motion, Drawable/Intermediate)
│   ├── Set Jitter (Jx, Jy) & Reset Flag
│   └── encodeToCommandBuffer(cmd)
│
└── presentDrawable(pDrawable) & commit()
```

---

## 3. Metal & C++ Blueprint

### Full Render Loop (`src/Renderer.cpp`)
```cpp
void renderTemporalFrame() {
    NS::AutoreleasePool* pool = NS::AutoreleasePool::alloc()->init();

    CA::MetalDrawable* pDrawable = pLayer->nextDrawable();
    if (!pDrawable) {
        pool->release();
        return;
    }

    MTL::CommandBuffer* cmd = pQueue->commandBuffer();

    // ========================================================
    // PASS 1: Scene Rasterization (MRT: Color + Motion + Depth)
    // ========================================================
    MTL::RenderPassDescriptor* passDesc = MTL::RenderPassDescriptor::renderPassDescriptor();
    
    // RT0: Color
    passDesc->colorAttachments()->object(0)->setTexture(pLowResColorTexture);
    passDesc->colorAttachments()->object(0)->setLoadAction(MTL::LoadActionClear);
    passDesc->colorAttachments()->object(0)->setStoreAction(MTL::StoreActionStore);
    passDesc->colorAttachments()->object(0)->setClearColor(MTL::ClearColor::Make(0.05, 0.05, 0.08, 1.0));

    // RT1: Motion Vectors
    passDesc->colorAttachments()->object(1)->setTexture(pLowResMotionTexture);
    passDesc->colorAttachments()->object(1)->setLoadAction(MTL::LoadActionClear);
    passDesc->colorAttachments()->object(1)->setStoreAction(MTL::StoreActionStore);
    passDesc->colorAttachments()->object(1)->setClearColor(MTL::ClearColor::Make(0.0, 0.0, 0.0, 0.0));

    // Reversed-Z Depth
    passDesc->depthAttachment()->setTexture(pLowResDepthTexture);
    passDesc->depthAttachment()->setLoadAction(MTL::LoadActionClear);
    passDesc->depthAttachment()->setStoreAction(MTL::StoreActionStore);
    passDesc->depthAttachment()->setClearDepth(0.0); // Reversed-Z far plane

    MTL::RenderCommandEncoder* sceneEnc = cmd->renderCommandEncoder(passDesc);
    sceneEnc->setRenderPipelineState(pMRT3DPipeline);
    sceneEnc->setDepthStencilState(pReversedZDepthState);
    sceneEnc->setCullMode(MTL::CullModeBack);
    // [Encode draw calls...]
    sceneEnc->endEncoding(); // FLUSH PASS 1

    // ========================================================
    // PASS 2: MetalFX Temporal Scaler Pass
    // ========================================================
    prepareTemporalFrame(
        pTemporalScaler,
        pLowResColorTexture,
        pLowResDepthTexture,
        pLowResMotionTexture,
        pDrawable->texture(),
        currentJitter,
        g_ResetHistoryThisFrame
    );

    pTemporalScaler->encodeToCommandBuffer(cmd);
    g_ResetHistoryThisFrame = false; // Reset one-shot flag

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
- [ ] Connect Scene Pass attachments with `StoreActionStore` on Color, Motion, and Depth.
- [ ] Ensure `sceneEnc->endEncoding()` is called before temporal scaler encoding.
- [ ] Call `pTemporalScaler->encodeToCommandBuffer(cmd)`.
- [ ] Present drawable and commit command buffer.
- [ ] Observe temporal accumulation: Within 8–16 frames of the camera coming to rest, observe all pixelation vanish into clean, temporally stable high-resolution imagery.

---

## 5. Definition of Done & Commit Message
- **DoD**: MetalFX Temporal Scaler actively reconstructs high-resolution images from a 540p input target with verified subpixel accumulation and zero Metal validation warnings.
- **Commit**:
  ```bash
  git add src/Renderer.cpp src/TemporalUpscaler.hpp
  git commit -m "feat(metalfx): encode temporal upscaler pass with active accumulation"
  ```
