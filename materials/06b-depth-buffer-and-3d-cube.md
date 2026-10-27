# TASK-06B: Depth Buffer & Spinning 3D Cube

## 1. Concept & Mental Model

### Why Do We Need a Depth Buffer (Z-Buffer)?
Without depth testing, the GPU relies on the **Painter's Algorithm**: whatever triangle happens to be drawn last overwrites previous pixels, regardless of whether it is physically in front or behind. As a 3D cube rotates, back faces draw over front faces, causing visual chaos and inside-out flickering.

A **Depth Buffer (Z-Buffer)** is a dedicated 2D floating-point texture (`MTL::PixelFormatDepth32Float`) that stores the distance from the camera to the closest fragment rendered so far at each pixel.

---

## 2. Metal Depth State & Pass Configuration

```
[ Fragment Candidate (Z_new) ]
              │
      (Compare with Z_buffer)
              │
        Z_new < Z_buffer ?
       ├── YES: Overwrite color, Z_buffer = Z_new (Depth Write)
       └── NO : Discard fragment (Occluded)
```

---

## 3. Metal & C++ Blueprint

### 1. Creating Depth Texture & Depth Stencil State
```cpp
// 1. Allocate Depth Texture matching window dimensions
MTL::TextureDescriptor* depthDesc = MTL::TextureDescriptor::texture2DDescriptor(
    MTL::PixelFormatDepth32Float,
    windowWidth,
    windowHeight,
    false
);
depthDesc->setUsage(MTL::TextureUsageRenderTarget);
depthDesc->setStorageMode(MTL::StorageModePrivate); // Fast on-tile GPU memory
MTL::Texture* pDepthTexture = pDevice->newTexture(depthDesc);
depthDesc->release();

// 2. Configure Depth Stencil State
MTL::DepthStencilDescriptor* dsDesc = MTL::DepthStencilDescriptor::alloc()->init();
dsDesc->setDepthCompareFunction(MTL::CompareFunctionLess);
dsDesc->setDepthWriteEnabled(true);
MTL::DepthStencilState* pDepthState = pDevice->newDepthStencilState(dsDesc);
dsDesc->release();
```

### 2. Updating Render Pass Pipeline & Draw Call
```cpp
// Update Pipeline Descriptor to declare depth attachment format
pipelineDesc->setDepthAttachmentPixelFormat(MTL::PixelFormatDepth32Float);

// In renderFrame():
passDesc->depthAttachment()->setTexture(pDepthTexture);
passDesc->depthAttachment()->setClearDepth(1.0); // 1.0 = Far plane in standard Z
passDesc->depthAttachment()->setLoadAction(MTL::LoadActionClear);
passDesc->depthAttachment()->setStoreAction(MTL::StoreActionDontCare);

MTL::RenderCommandEncoder* enc = cmd->renderCommandEncoder(passDesc);
enc->setRenderPipelineState(p3DPipeline);
enc->setDepthStencilState(pDepthState);
enc->setCullMode(MTL::CullModeBack); // Cull back-facing triangles

enc->setVertexBuffer(pCubeVertexBuffer, 0, 0);
enc->setVertexBuffer(pUniformBuffer, 0, 1);

// Draw 36 indices using 16-bit index buffer
enc->drawIndexedPrimitives(
    MTL::PrimitiveTypeTriangle,
    36,
    MTL::IndexTypeUInt16,
    pCubeIndexBuffer,
    0
);

enc->endEncoding();
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Create `MTL::Texture` with `MTL::PixelFormatDepth32Float`.
- [ ] Create `MTL::DepthStencilState` with `CompareFunctionLess` and `depthWriteEnabled = true`.
- [ ] Update `RenderPipelineDescriptor` to include depth attachment format.
- [ ] Attach depth texture to `passDesc->depthAttachment()` with `clearDepth = 1.0`.
- [ ] Call `enc->drawIndexedPrimitives()` with the 36 cube indices.
- [ ] Verify: The 3D cube rotates in full perspective with clean edges and zero depth sorting artifacts.

---

## 5. Definition of Done & Commit Message
- **DoD**: A 3D colored cube rotates on screen with accurate perspective projection, back-face culling, and flawless depth occlusion.
- **Commit**:
  ```bash
  git add src/Renderer.cpp src/Renderer.hpp src/Shaders.metal
  git commit -m "feat(renderer): implement depth testing and render spinning 3d cube"
  ```
