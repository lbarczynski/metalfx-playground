# TASK-06: 3D Cube, Depth Testing & Camera Projection

## 1. Concept & Mental Model
3D rendering projects world-space objects onto a 2D screen using perspective projection and resolves primitive visibility via hardware depth testing. In OpenGL, depth testing is enabled with `glEnable(GL_DEPTH_TEST)` using a driver-managed default depth buffer; in Metal, depth testing is configured as an immutable `MTL::DepthStencilState` object, and the depth attachment texture (`MTL::PixelFormatDepth32Float`) must be explicitly allocated and bound to the render pass descriptor.

## 2. Mathematical Foundations & Algorithms
### Metal Perspective Projection Matrix
Metal clip space uses a **left-handed coordinate system with depth range $[0.0, 1.0]$**:
$$x_{\text{ndc}} \in [-1.0, 1.0], \quad y_{\text{ndc}} \in [-1.0, 1.0], \quad z_{\text{ndc}} \in [0.0, 1.0]$$

Given vertical Field of View $\theta = \text{fovy}$, aspect ratio $a = \frac{W}{H}$, near plane $n$, and far plane $f$, with camera looking down $-Z$ (right-handed view space):
$$f_t = \frac{1}{\tan(\theta / 2)}$$
$$\mathbf{P}_{\text{Metal}} = \begin{bmatrix}
\frac{f_t}{a} & 0 & 0 & 0 \\
0 & f_t & 0 & 0 \\
0 & 0 & \frac{f}{n - f} & \frac{n \cdot f}{n - f} \\
0 & 0 & -1 & 0
\end{bmatrix}$$

*Derivation check*: When eye $z = -n \implies z_{\text{ndc}} = 0.0$; when eye $z = -f \implies z_{\text{ndc}} = 1.0$.

### Look-At View Matrix
Given eye position $\mathbf{E}$, target position $\mathbf{T}$, and world up $\mathbf{U}_{\text{world}}$:
$$\mathbf{f} = \frac{\mathbf{T} - \mathbf{E}}{\|\mathbf{T} - \mathbf{E}\|}, \quad \mathbf{r} = \frac{\mathbf{f} \times \mathbf{U}_{\text{world}}}{\|\mathbf{f} \times \mathbf{U}_{\text{world}}\|}, \quad \mathbf{u} = \mathbf{r} \times \mathbf{f}$$
$$\mathbf{V} = \begin{bmatrix}
r_x & r_y & r_z & -\mathbf{r} \cdot \mathbf{E} \\
u_x & u_y & u_z & -\mathbf{u} \cdot \mathbf{E} \\
-f_x & -f_y & -f_z & \mathbf{f} \cdot \mathbf{E} \\
0 & 0 & 0 & 1
\end{bmatrix}$$

### Index Buffer Deduplication
A cube has 6 faces $\times$ 2 triangles = 12 triangles = 36 vertices. Using an index buffer (`uint16_t`), only 8 unique corner vertices are allocated:
$$\text{Memory Savings} = 36 \times \text{sizeof(Vertex)} \longrightarrow 8 \times \text{sizeof(Vertex)} + 36 \times 2 \text{ bytes}$$

## 3. Metal & C++ Blueprint

### 1. Depth Texture & Depth-Stencil State Creation
```cpp
#include <Metal/Metal.hpp>

// 1. Allocate Depth Texture Attachment
MTL::TextureDescriptor* depthDesc = MTL::TextureDescriptor::texture2DDescriptor(
    MTL::PixelFormatDepth32Float,
    width,
    height,
    false
);
depthDesc->setUsage(MTL::TextureUsageRenderTarget);
depthDesc->setStorageMode(MTL::StorageModePrivate); // Fast on-tile GPU memory

MTL::Texture* pDepthTexture = pDevice->newTexture(depthDesc);
depthDesc->release();

// 2. Create Depth Stencil State
MTL::DepthStencilDescriptor* dsDesc = MTL::DepthStencilDescriptor::alloc()->init();
dsDesc->setDepthCompareFunction(MTL::CompareFunctionLess);
dsDesc->setDepthWriteEnabled(true);

MTL::DepthStencilState* pDepthStencilState = pDevice->newDepthStencilState(dsDesc);
dsDesc->release();
```

### 2. Pipeline Descriptor Depth Configuration
```cpp
// Render pipeline must specify matching depth pixel format
pPipelineDesc->setDepthAttachmentPixelFormat(MTL::PixelFormatDepth32Float);
pPipelineDesc->colorAttachments()->object(0)->setPixelFormat(pView ? pView->colorPixelFormat() : MTL::PixelFormatRGBA8Unorm);
```

### 3. Cube Vertex & Index Buffers
> [!IMPORTANT]
> **SIMD Memory Layout on Apple Silicon (ARM64)**: `simd::float3` has `sizeof=16` and `alignof=16`. The offset of `color` in `Vertex3D` is **16 bytes** (not 12). In `MTL::VertexDescriptor`, configure attribute 0 at offset 0, attribute 1 at offset 16, and layout stride at **32 bytes**.

```cpp
struct alignas(16) Vertex3D {
    simd::float3 position; // 16 bytes on AppleClang ARM64
    simd::float4 color;    // 16 bytes
};

// 8 unique cube corners
const Vertex3D cubeVertices[8] = {
    { { -0.5f, -0.5f,  0.5f }, { 1, 0, 0, 1 } },
    { {  0.5f, -0.5f,  0.5f }, { 0, 1, 0, 1 } },
    { {  0.5f,  0.5f,  0.5f }, { 0, 0, 1, 1 } },
    { { -0.5f,  0.5f,  0.5f }, { 1, 1, 0, 1 } },
    { { -0.5f, -0.5f, -0.5f }, { 1, 0, 1, 1 } },
    { {  0.5f, -0.5f, -0.5f }, { 0, 1, 1, 1 } },
    { {  0.5f,  0.5f, -0.5f }, { 1, 1, 1, 1 } },
    { { -0.5f,  0.5f, -0.5f }, { 0, 0, 0, 1 } }
};

// 36 indices for 12 triangles (CCW winding)
const uint16_t cubeIndices[36] = {
    0, 1, 2, 2, 3, 0, // Front
    1, 5, 6, 6, 2, 1, // Right
    5, 4, 7, 7, 6, 5, // Back
    4, 0, 3, 3, 7, 4, // Left
    3, 2, 6, 6, 7, 3, // Top
    4, 5, 1, 1, 0, 4  // Bottom
};

MTL::Buffer* pVertexBuffer = pDevice->newBuffer(cubeVertices, sizeof(cubeVertices), MTL::ResourceStorageModeShared);
MTL::Buffer* pIndexBuffer = pDevice->newBuffer(cubeIndices, sizeof(cubeIndices), MTL::ResourceStorageModeShared);
```

### 4. MSL Shader with MVP Transform & Vertex Attributes
```metal
#include <metal_stdlib>
using namespace metal;

struct VertexInput {
    float3 position [[attribute(0)]];
    float4 color    [[attribute(1)]];
};

struct CubeUniforms {
    float4x4 modelViewProjectionMatrix;
};

struct RasterizerData {
    float4 position [[position]];
    float4 color;
};

vertex RasterizerData vertexShader(VertexInput in [[stage_in]],
                                   constant CubeUniforms& uniforms [[buffer(1)]]) {
    RasterizerData out;
    out.position = uniforms.modelViewProjectionMatrix * float4(in.position, 1.0);
    out.color = in.color;
    return out;
}

fragment float4 fragmentShader(RasterizerData in [[stage_in]]) {
    return in.color;
}
```

### 5. Encoding Depth Pass & Indexed Draw
```cpp
// Set up depth attachment in pass descriptor
passDesc->depthAttachment()->setTexture(pDepthTexture);
passDesc->depthAttachment()->setClearDepth(1.0);
passDesc->depthAttachment()->setLoadAction(MTL::LoadActionClear);
passDesc->depthAttachment()->setStoreAction(MTL::StoreActionDontCare); // Don't care if not reused

MTL::RenderCommandEncoder* enc = cmd->renderCommandEncoder(passDesc);

enc->setRenderPipelineState(pPipelineState);
enc->setDepthStencilState(pDepthStencilState);
enc->setCullMode(MTL::CullModeBack);
enc->setFrontFacingWinding(MTL::WindingCounterClockwise);

enc->setVertexBuffer(pVertexBuffer, 0, 0);
enc->setVertexBuffer(pUniformBuffer, uniformOffset, 1);

enc->drawIndexedPrimitives(
    MTL::PrimitiveTypeTriangle,
    36,
    MTL::IndexTypeUInt16,
    pIndexBuffer,
    0
);

enc->endEncoding();
```

## 4. Step-by-Step Implementation Checklist
- [ ] Implement `makePerspective(fovy, aspect, near, far)` returning Metal-compliant matrix.
- [ ] Implement `makeLookAt(eye, target, up)`.
- [ ] Allocate `MTL::Texture` with `MTL::PixelFormatDepth32Float` matching window dimensions.
- [ ] Create `MTL::DepthStencilState` with `CompareFunctionLess` and `depthWriteEnabled = true`.
- [ ] Set `setDepthAttachmentPixelFormat(MTL::PixelFormatDepth32Float)` on `MTL::RenderPipelineDescriptor`.
- [ ] Allocate vertex buffer (8 vertices) and index buffer (36 `uint16_t` indices).
- [ ] Bind depth texture to `passDesc->depthAttachment()->setTexture(pDepthTexture)`.
- [ ] Bind `enc->setDepthStencilState()`, `enc->setCullMode(MTL::CullModeBack)`.
- [ ] Execute `enc->drawIndexedPrimitives(MTL::PrimitiveTypeTriangle, 36, MTL::IndexTypeUInt16, pIndexBuffer, 0)`.
- [ ] Handle window resize by releasing and recreating the depth texture at the new dimensions.

## 5. Common Pitfalls & Debugging Checklist
- **Black Screen or Pipeline Creation Crash**:
  - *Cause*: `pPipelineDesc->setDepthAttachmentPixelFormat()` omitted or set to `MTLPixelFormatInvalid` while pass descriptor provides a depth attachment.
  - *Fix*: Both pipeline descriptor and render pass attachment must agree on `MTL::PixelFormatDepth32Float`.
- **Z-Fighting or Disappearing Geometry**:
  - *Cause*: Near plane set to $0.0$ or negative value ($n \le 0.0$ generates division by zero / `NaN` in projection matrix).
  - *Fix*: Keep $n \ge 0.1$.
- **Cube Interior Rendered Instead of Exterior**:
  - *Cause*: Winding order mismatch (`WindingClockwise` vs `WindingCounterClockwise`) with `CullModeBack`.
  - *Fix*: Align index winding with `setFrontFacingWinding(MTL::WindingCounterClockwise)`.
