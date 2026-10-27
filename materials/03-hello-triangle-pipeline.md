# TASK-03: The Hello Triangle & Metal Shading Language

## 1. Concept & Mental Model
The graphics pipeline consumes structured vertex streams in GPU memory, transforms them through programmable vertex shading into Normalized Device Coordinates (NDC), rasterizes primitive geometries, and executes fragment shading to produce final framebuffer color values. In OpenGL, state is bound via `glUseProgram`, `glBindVertexArray`, and `glDrawArrays`; in Metal, all shader stages, pixel formats, and blend configurations are compiled upfront into an immutable `MTL::RenderPipelineState`, with vertex buffers bound directly via explicit buffer indices.

## 2. Mathematical Foundations & Algorithms
### Metal Clip Space & NDC Coordinate Systems
Metal defines Normalized Device Coordinates (NDC) in a **left-handed coordinate system with a $[0.0, 1.0]$ depth range**:
$$x_{ndc} \in [-1.0, 1.0] \quad (\text{left} \rightarrow \text{right})$$
$$y_{ndc} \in [-1.0, 1.0] \quad (\text{bottom} \rightarrow \text{top in NDC})$$
$$z_{ndc} \in [0.0, 1.0] \quad (\text{near} \rightarrow \text{far})$$

*(Critical difference: OpenGL uses $z_{ndc} \in [-1.0, 1.0]$, whereas Metal and DirectX 12 use $[0.0, 1.0]$.)*

### Barycentric Coordinates Interpolation
For triangle vertices $\mathbf{v}_0, \mathbf{v}_1, \mathbf{v}_2$ with associated vertex colors $\mathbf{c}_0, \mathbf{c}_1, \mathbf{c}_2$, any fragment point $\mathbf{p}$ on the primitive is parameterized by barycentric weights $\lambda_0, \lambda_1, \lambda_2$:
$$\mathbf{p} = \lambda_0 \mathbf{v}_0 + \lambda_1 \mathbf{v}_1 + \lambda_2 \mathbf{v}_2$$
$$\lambda_0 + \lambda_1 + \lambda_2 = 1.0, \quad \lambda_i \ge 0$$
$$\mathbf{c}(\mathbf{p}) = \lambda_0 \mathbf{c}_0 + \lambda_1 \mathbf{c}_1 + \lambda_2 \mathbf{c}_2$$

Hardware rasterization evaluates $\lambda_i$ and feeds smoothly interpolated colors into the fragment function.

## 3. Metal & C++ Blueprint

### 1. Interleaved Vertex Data Structure
```cpp
#include <simd/simd.h>

// Explicit memory alignment matching MSL float4 padding
struct alignas(16) Vertex {
    simd::float2 position; // 8 bytes: x, y
    simd::float2 _pad0;    // 8 bytes padding to align next member
    simd::float4 color;    // 16 bytes: r, g, b, a
};
```

### 2. Metal Shading Language (MSL) Shader Code
```metal
#include <metal_stdlib>
using namespace metal;

struct VertexInput {
    float2 position [[attribute(0)]];
    float4 color    [[attribute(1)]];
};

struct RasterizerData {
    float4 position [[position]];
    float4 color;
};

// Attribute inputs marked [[stage_in]] are fetched by hardware via MTLVertexDescriptor
vertex RasterizerData vertexShader(VertexInput in [[stage_in]]) {
    RasterizerData out;
    out.position = float4(in.position, 0.0, 1.0);
    out.color = in.color;
    return out;
}

fragment float4 fragmentShader(RasterizerData in [[stage_in]]) {
    return in.color;
}
```

### 3. Pipeline & Buffer Initialization (C++)
```cpp
// Triangle vertices in NDC
const Vertex triangleVertices[3] = {
    { .position = {  0.0f,  0.5f }, .color = { 1.0f, 0.0f, 0.0f, 1.0f } }, // Top (Red)
    { .position = { -0.5f, -0.5f }, .color = { 0.0f, 1.0f, 0.0f, 1.0f } }, // Bottom-Left (Green)
    { .position = {  0.5f, -0.5f }, .color = { 0.0f, 0.0f, 1.0f, 1.0f } }  // Bottom-Right (Blue)
};

// 1. Create shared vertex buffer
MTL::Buffer* pVertexBuffer = pDevice->newBuffer(
    triangleVertices,
    sizeof(triangleVertices),
    MTL::ResourceStorageModeShared
);

// 2. Compile shaders from string (or precompiled metallib)
NS::Error* pError = nullptr;
MTL::CompileOptions* pOptions = MTL::CompileOptions::alloc()->init();
MTL::Library* pLibrary = pDevice->newLibrary(
    NS::String::string(shaderSource, NS::UTF8StringEncoding),
    pOptions,
    &pError
);
pOptions->release();

if (!pLibrary) {
    std::cerr << "Shader compile error: " << pError->localizedDescription()->utf8String() << "\n";
    return;
}

MTL::Function* pVertFunc = pLibrary->newFunction(
    NS::String::string("vertexShader", NS::UTF8StringEncoding)
);
MTL::Function* pFragFunc = pLibrary->newFunction(
    NS::String::string("fragmentShader", NS::UTF8StringEncoding)
);

// 3. Configure Type-Safe Hardware Vertex Descriptor
MTL::VertexDescriptor* pVertexDesc = MTL::VertexDescriptor::vertexDescriptor();

// Attribute 0: position
pVertexDesc->attributes()->object(0)->setFormat(MTL::VertexFormatFloat2);
pVertexDesc->attributes()->object(0)->setOffset(offsetof(Vertex, position));
pVertexDesc->attributes()->object(0)->setBufferIndex(0);

// Attribute 1: color
pVertexDesc->attributes()->object(1)->setFormat(MTL::VertexFormatFloat4);
pVertexDesc->attributes()->object(1)->setOffset(offsetof(Vertex, color));
pVertexDesc->attributes()->object(1)->setBufferIndex(0);

// Layout 0: Stride & Step Rate
pVertexDesc->layouts()->object(0)->setStride(sizeof(Vertex));
pVertexDesc->layouts()->object(0)->setStepFunction(MTL::VertexStepFunctionPerVertex);

// 4. Create Render Pipeline State
MTL::RenderPipelineDescriptor* pPipeDesc = MTL::RenderPipelineDescriptor::alloc()->init();
pPipeDesc->setVertexFunction(pVertFunc);
pPipeDesc->setFragmentFunction(pFragFunc);
pPipeDesc->setVertexDescriptor(pVertexDesc);
pPipeDesc->colorAttachments()->object(0)->setPixelFormat(pView ? pView->colorPixelFormat() : MTL::PixelFormatRGBA8Unorm);

MTL::RenderPipelineState* pPipelineState = pDevice->newRenderPipelineState(pPipeDesc, &pError);

pPipeDesc->release();
pVertexDesc->release();
pVertFunc->release();
pFragFunc->release();
pLibrary->release();
```

### 4. Encoding the Draw Call
```cpp
MTL::CommandBuffer* cmd = pQueue->commandBuffer();
MTL::RenderCommandEncoder* enc = cmd->renderCommandEncoder(passDesc);

enc->setRenderPipelineState(pPipelineState);
enc->setVertexBuffer(pVertexBuffer, 0, 0); // buffer pointer, offset, index
enc->drawPrimitives(MTL::PrimitiveTypeTriangle, NS::UInteger(0), NS::UInteger(3)); // primitive, startVertex, vertexCount

enc->endEncoding();
cmd->presentDrawable(drawable);
cmd->commit();
```

## 4. Step-by-Step Implementation Checklist
- [ ] Define the `Vertex` struct with 16-byte alignment.
- [ ] Construct the 3 CCW vertices in NDC space.
- [ ] Allocate the vertex `MTL::Buffer` with `MTL::ResourceStorageModeShared`.
- [ ] Write the MSL shader string containing `vertexShader` and `fragmentShader`.
- [ ] Compile `MTL::Library` using `device->newLibrary()`. Check `pError` for compilation logs.
- [ ] Configure `MTL::RenderPipelineDescriptor`, ensuring the color attachment pixel format matches `view->colorPixelFormat()`.
- [ ] Build the `MTL::RenderPipelineState`.
- [ ] In `drawInMTKView()`, bind the pipeline state with `enc->setRenderPipelineState()`.
- [ ] Bind vertex buffer at buffer index 0: `enc->setVertexBuffer(pVertexBuffer, 0, 0)`.
- [ ] Issue draw call: `enc->drawPrimitives(MTL::PrimitiveTypeTriangle, 0, 3)`.
- [ ] Run application and verify smooth color gradient across the rendered triangle.

## 5. Common Pitfalls & Debugging Checklist
- **Metal Validation Layer Assertion: Pixel Format Mismatch**:
  - *Error*: `Pixel format (MTLPixelFormatBGRA8Unorm) does not match render pipeline state format (MTLPixelFormatRGBA8Unorm)`.
  - *Fix*: Match `pPipeDesc->colorAttachments()->object(0)->setPixelFormat(pView->colorPixelFormat())` explicitly.
- **Memory Layout & MSL Alignment Discrepancies**:
  - *Symptom*: Colors are black or coordinates are distorted.
  - *Cause*: In MSL, `float3` has a 16-byte alignment and size requirement. Packing `float2` (8 bytes) followed by `float3` (12 bytes in C++ standard struct) leads to offset misalignment.
  - *Fix*: Use `simd::float4` for color or add explicit padding bytes in C++.
- **Winding Order Culling**:
  - *Symptom*: Triangle disappears when culling is enabled.
  - *Cause*: Default winding is Counter-Clockwise (CCW). If vertices are listed in Clockwise (CW) order with back-face culling enabled, the triangle will be discarded.
