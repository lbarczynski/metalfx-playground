# TASK-13: Encoding the Spatial Scaler Pass

## 1. Concept & Mental Model
Executing MetalFX Spatial involves binding the low-resolution color texture and high-resolution destination drawable onto the scaler instance and encoding its internal compute kernel into the active command buffer. Unlike graphics passes that open and close an `MTL::RenderCommandEncoder`, MetalFX encodes directly at the `MTL::CommandBuffer` level, managing internal GPU memory barriers and execution scheduling automatically.

## 2. Mathematical Foundations & Algorithms
### GPU Pipeline Sequencing & Hazard Management
The low-resolution color texture experiences two distinct GPU access phases during a single frame:
$$\text{Phase 1: Scene Rasterization} \longrightarrow \text{Write access to } T_{\text{lowres}} \text{ via Tile Color Attachment}$$
$$\text{Execution Boundary} \longrightarrow \text{Encoder End establishes tile writeback fence}$$
$$\text{Phase 2: MetalFX Upscale} \longrightarrow \text{Read access to } T_{\text{lowres}} \text{ via Texture Sampling Units}$$

Calling `pRenderEncoder->endEncoding()` prior to `pScaler->encodeToCommandBuffer(cmd)` signals the Metal driver to flush tile cache lines, preventing Read-After-Write (RAW) data hazards.

```
[ Command Buffer ]
├── RenderPassEncoder (Scene -> LowResTexture)
│   ├── setRenderPipelineState
│   ├── drawIndexedPrimitives
│   └── endEncoding()  <--- HAZARD BARRIER FLUSH
├── MetalFX Spatial Pass
│   ├── scaler->setColorTexture(LowResTexture)
│   ├── scaler->setOutputTexture(DrawableTexture)
│   └── scaler->encodeToCommandBuffer()
└── presentDrawable(Drawable) & commit()
```

## 3. Metal & C++ Blueprint

### 1. Complete Multi-Pass Render Loop
```cpp
void renderFrame(
    MTK::View* pView,
    MTL::CommandQueue* pQueue,
    MTLFX::SpatialScaler* pScaler,
    MTL::Texture* pLowResColorTexture,
    MTL::Texture* pLowResDepthTexture,
    MTL::RenderPipelineState* pScenePipeline,
    MTL::DepthStencilState* pDepthState,
    MTL::Buffer* pVertexBuffer,
    MTL::Buffer* pIndexBuffer,
    MTL::Buffer* pUniformBuffer
) {
    NS::AutoreleasePool* pool = NS::AutoreleasePool::alloc()->init();

    CA::MetalDrawable* pDrawable = pView->currentDrawable();
    if (!pDrawable) {
        pool->release();
        return;
    }

    MTL::CommandBuffer* cmd = pQueue->commandBuffer();

    // ========================================================
    // PASS 1: Render 3D Scene into Low-Resolution Render Target
    // ========================================================
    MTL::RenderPassDescriptor* scenePass = MTL::RenderPassDescriptor::renderPassDescriptor();
    
    // Color attachment configuration
    auto colorAttach = scenePass->colorAttachments()->object(0);
    colorAttach->setTexture(pLowResColorTexture);
    colorAttach->setLoadAction(MTL::LoadActionClear);
    colorAttach->setStoreAction(MTL::StoreActionStore); // CRITICAL: Retain for MetalFX read
    colorAttach->setClearColor(MTL::ClearColor::Make(0.05, 0.05, 0.08, 1.0));

    // Depth attachment configuration
    auto depthAttach = scenePass->depthAttachment();
    depthAttach->setTexture(pLowResDepthTexture);
    depthAttach->setLoadAction(MTL::LoadActionClear);
    depthAttach->setStoreAction(MTL::StoreActionDontCare);
    depthAttach->setClearDepth(1.0);

    MTL::RenderCommandEncoder* enc = cmd->renderCommandEncoder(scenePass);
    enc->setRenderPipelineState(pScenePipeline);
    enc->setDepthStencilState(pDepthState);
    enc->setVertexBuffer(pVertexBuffer, 0, 0);
    enc->setVertexBuffer(pUniformBuffer, 0, 1);
    
    enc->drawIndexedPrimitives(MTL::PrimitiveTypeTriangle, 36, MTL::IndexTypeUInt16, pIndexBuffer, 0);
    
    // MUST end encoding before MetalFX encoding begins
    enc->endEncoding();

    // ========================================================
    // PASS 2: Encode MetalFX Spatial Upscaler
    // ========================================================
    pScaler->setColorTexture(pLowResColorTexture);
    pScaler->setOutputTexture(pDrawable->texture());
    pScaler->encodeToCommandBuffer(cmd);

    // ========================================================
    // PRESENTATION
    // ========================================================
    cmd->presentDrawable(pDrawable);
    cmd->commit();

    pool->release();
}
```

## 4. Step-by-Step Implementation Checklist
- [ ] Configure scene render pass descriptor using `pLowResColorTexture` as color attachment 0.
- [ ] Set `setStoreAction(MTL::StoreActionStore)` on the scene color attachment.
- [ ] Encode all geometry draw calls into the scene render pass.
- [ ] Explicitly call `enc->endEncoding()` to finalize the scene pass.
- [ ] Retrieve `pView->currentDrawable()`.
- [ ] Call `pScaler->setColorTexture(pLowResColorTexture)`.
- [ ] Call `pScaler->setOutputTexture(pDrawable->texture())`.
- [ ] Call `pScaler->encodeToCommandBuffer(cmd)`.
- [ ] Present drawable and commit command buffer.
- [ ] Verify that the scene is rendered at low resolution and upscaled to the full display window.

## 5. Common Pitfalls & Debugging Checklist
- **Black Screen from Discarded Pixels**:
  - *Cause*: Leaving color attachment store action as `StoreActionDontCare`. The GPU discards tile contents on pass completion.
  - *Fix*: Set `colorAttach->setStoreAction(MTL::StoreActionStore)`.
- **Active Encoder Conflict Assertion**:
  - *Error*: `Encoding to command buffer while another encoder (MTLRenderCommandEncoder) is active`.
  - *Fix*: Ensure `enc->endEncoding()` is called before `pScaler->encodeToCommandBuffer(cmd)`.
- **Texture Usage Assertion Failure**:
  - *Error*: `Color texture must have MTLTextureUsageShaderRead enabled`.
  - *Fix*: Ensure the input texture was allocated with `MTL::TextureUsageRenderTarget | MTL::TextureUsageShaderRead`.
