# TASK-20: Wiring the Temporal Scaler Pass

## 1. Concept & Mental Model
Wiring MetalFX Temporal establishes the runtime binding between the low-resolution render targets (Jittered Color, Reversed-Z Depth, Screen-Space Motion Vectors), subpixel jitter coordinates, and the destination display texture. The engine must orchestrate texture lifetimes, synchronization fences, and camera cut detection flags to ensure historical image reprojection functions without ghosting trails or stalling the GPU.

## 2. Mathematical Foundations & Algorithms
### Temporal Disocclusion & Camera Cut Detection
When camera position jumps drastically (e.g. cutscenes or teleports), historical pixels have no spatial correspondence in the current view frustum:
$$\|\mathbf{T}_{\text{curr}} - \mathbf{T}_{\text{prev}}\| > D_{\text{threshold}} \quad \text{or} \quad \Delta \text{Angle} > \theta_{\text{threshold}}$$

If a camera cut occurs, historical accumulation must be immediately invalidated:
$$\text{setResetHistory}(\text{true})$$
This forces MetalFX Temporal to perform spatial reconstruction for that single frame, instantaneously rebuilding a fresh history baseline.

## 3. Metal & C++ Blueprint

### 1. Complete Temporal Scaler Execution Function
```cpp
void executeTemporalUpscale(
    MTL::CommandBuffer* cmd,
    MTLFX::TemporalScaler* pScaler,
    MTL::Texture* pLowResColor,
    MTL::Texture* pLowResDepth,
    MTL::Texture* pLowResMotion,
    MTL::Texture* pHighResOutput,
    float jitterOffsetX,
    float jitterOffsetY,
    bool resetHistory
) {
    // 1. Bind Input Attachments
    pScaler->setColorTexture(pLowResColor);
    pScaler->setDepthTexture(pLowResDepth);
    pScaler->setMotionTexture(pLowResMotion);
    pScaler->setOutputTexture(pHighResOutput);

    // 2. Set Subpixel Jitter Offsets (in input pixels)
    pScaler->setJitterOffsetX(jitterOffsetX);
    pScaler->setJitterOffsetY(jitterOffsetY);

    // 3. Set Camera Cut / History Invalidation Flag
    pScaler->setReset(resetHistory);

    // 4. Encode pass directly into active command buffer
    pScaler->encodeToCommandBuffer(cmd);
}
```

### 2. Multi-Pass Frame Orchestration
```cpp
void renderFrame() {
    NS::AutoreleasePool* pool = NS::AutoreleasePool::alloc()->init();

    CA::MetalDrawable* drawable = pView->currentDrawable();
    if (!drawable) { pool->release(); return; }

    MTL::CommandBuffer* cmd = pQueue->commandBuffer();

    // ========================================================
    // PASS 1: Scene Rasterization (MRT: Color + Motion + Depth)
    // ========================================================
    MTL::RenderPassDescriptor* passDesc = MTL::RenderPassDescriptor::renderPassDescriptor();
    
    // RT0: Color
    passDesc->colorAttachments()->object(0)->setTexture(pLowResColor);
    passDesc->colorAttachments()->object(0)->setLoadAction(MTL::LoadActionClear);
    passDesc->colorAttachments()->object(0)->setStoreAction(MTL::StoreActionStore);

    // RT1: Motion Vectors
    passDesc->colorAttachments()->object(1)->setTexture(pLowResMotion);
    passDesc->colorAttachments()->object(1)->setLoadAction(MTL::LoadActionClear);
    passDesc->colorAttachments()->object(1)->setClearColor(MTL::ClearColor::Make(0, 0, 0, 0));
    passDesc->colorAttachments()->object(1)->setStoreAction(MTL::StoreActionStore);

    // Depth: Reversed-Z
    passDesc->depthAttachment()->setTexture(pLowResDepth);
    passDesc->depthAttachment()->setClearDepth(0.0);
    passDesc->depthAttachment()->setLoadAction(MTL::LoadActionClear);
    passDesc->depthAttachment()->setStoreAction(MTL::StoreActionStore);

    MTL::RenderCommandEncoder* enc = cmd->renderCommandEncoder(passDesc);
    // [Draw scene geometry...]
    enc->endEncoding(); // Close encoder to flush cache lines

    // ========================================================
    // PASS 2: MetalFX Temporal Scaler Pass
    // ========================================================
    executeTemporalUpscale(
        cmd,
        pTemporalScaler,
        pLowResColor,
        pLowResDepth,
        pLowResMotion,
        drawable->texture(),
        currentJitter.x,
        currentJitter.y,
        g_ResetHistoryNextFrame
    );

    g_ResetHistoryNextFrame = false; // Clear one-shot flag

    cmd->presentDrawable(drawable);
    cmd->commit();

    pool->release();
}
```

## 4. Step-by-Step Implementation Checklist
- [ ] Connect Scene Pass outputs (Color, Depth, Motion) to `MTL::StoreActionStore`.
- [ ] Call `enc->endEncoding()` prior to encoding the temporal scaler.
- [ ] Call `setJitterOffsetX()` and `setJitterOffsetY()`.
- [ ] Bind all 4 textures (`Color`, `Depth`, `Motion`, `Output`).
- [ ] Set `pScaler->setReset(true)` on first frame or upon camera cut.
- [ ] Encode temporal scaler pass via `pScaler->encodeToCommandBuffer(cmd)`.
- [ ] Present drawable and commit command buffer.

## 5. Common Pitfalls & Debugging Checklist
- **Ghosting Trails on Camera Cuts**:
  - *Cause*: Failing to set `pScaler->setReset(true)` after camera jumps.
  - *Fix*: Detect large position deltas and call `pScaler->setReset(true)` for one frame.
- **Validation Failure: Unbound Textures**:
  - *Error*: `Temporal scaler requires all 4 textures to be bound before encoding`.
  - *Fix*: Ensure none of the 4 texture pointers are `nullptr`.
- **Command Encoder Overlap**:
  - *Fix*: Do not call `encodeToCommandBuffer()` while another encoder is open.
