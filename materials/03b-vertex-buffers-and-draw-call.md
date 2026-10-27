# TASK-03B: Vertex Buffer Layout & Triangle Draw Call

## 1. Concept & Mental Model

### The Kotlin / Android Bridge
In Kotlin, an `Array<Vertex>` is actually an array of object references (pointers) scattered across heap memory. Each object has an 8–16 byte JVM object header, causing cache misses and pointer-chasing overhead.

In Modern C++ and Metal:
- **Value Semantics & Contiguous Memory**: `struct Vertex` has zero header overhead. An array `std::vector<Vertex>` occupies a single continuous block of physical memory.
- **Unified Memory Buffer**: On Apple Silicon, an `MTL::Buffer` allocated with `MTL::ResourceStorageModeShared` allows the CPU to write vertex coordinates directly into memory that the GPU reads with zero PCIe copying.

---

## 2. Geometric Layout & Normalized Device Coordinates (NDC)

Metal clip space uses the following Normalized Device Coordinates (NDC):
$$x \in [-1.0, 1.0], \quad y \in [-1.0, 1.0], \quad z \in [0.0, 1.0]$$
The center of the screen is $(0, 0)$. Top-left is $(-1, 1)$, bottom-right is $(1, -1)$.

```
            (0.0, 0.5) [Red]
                ▲
               / \
              /   \
             /     \
            /       \
 (-0.5, -0.5)       (0.5, -0.5)
   [Green]             [Blue]
```

### The Interleaved Vertex Struct (`src/Renderer.hpp`)
```cpp
#include <simd/simd.h>

struct Vertex {
    simd::float2 position; // x, y in NDC
    simd::float3 color;    // r, g, b in [0, 1]
};
```

---

## 3. Metal & C++ Blueprint

### 1. Creating the Vertex Buffer
```cpp
const Vertex triangleVertices[] = {
    { {  0.0f,  0.5f }, { 1.0f, 0.0f, 0.0f } }, // Top (Red)
    { { -0.5f, -0.5f }, { 0.0f, 1.0f, 0.0f } }, // Bottom-Left (Green)
    { {  0.5f, -0.5f }, { 0.0f, 0.0f, 1.0f } }  // Bottom-Right (Blue)
};

MTL::Buffer* pVertexBuffer = pDevice->newBuffer(
    triangleVertices,
    sizeof(triangleVertices),
    MTL::ResourceStorageModeShared
);
```

### 2. Updating MSL Vertex Shader for Interleaved Layout
```metal
struct VertexInput {
    float2 position;
    float3 color;
};

struct VertexOut {
    float4 position [[position]];
    float3 color;
};

vertex VertexOut vertexMain(
    uint vertexID [[vertex_id]],
    constant VertexInput* vertices [[buffer(0)]]
) {
    VertexOut out;
    out.position = float4(vertices[vertexID].position, 0.0, 1.0);
    out.color = vertices[vertexID].color;
    return out;
}
```

### 3. Encoding the Draw Call in `renderFrame()`
```cpp
MTL::RenderCommandEncoder* enc = cmd->renderCommandEncoder(passDesc);

// Bind pipeline state and vertex buffer
enc->setRenderPipelineState(pPipelineState);
enc->setVertexBuffer(pVertexBuffer, 0, 0);

// Draw 3 vertices as 1 triangle
enc->drawPrimitives(MTL::PrimitiveTypeTriangle, 0, 3);

enc->endEncoding();
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Define `Vertex` struct with `simd::float2` position and `simd::float3` color.
- [ ] Allocate `MTL::Buffer` with `MTL::ResourceStorageModeShared` containing the 3 triangle vertices.
- [ ] Update the vertex shader in MSL to accept `constant VertexInput* vertices [[buffer(0)]]`.
- [ ] In your render loop, bind the pipeline state and vertex buffer.
- [ ] Call `enc->drawPrimitives(MTL::PrimitiveTypeTriangle, 0, 3)`.
- [ ] Launch application: Verify a vibrant triangle appears in the center of the window with smoothly interpolated colors.

---

## 5. Definition of Done & Commit Message
- **DoD**: A smoothly interpolated RGB triangle renders in the center of the window with correct vertex color interpolation.
- **Commit**:
  ```bash
  git add src/Renderer.cpp src/Renderer.hpp src/Shaders.metal
  git commit -m "feat(renderer): render hello triangle with custom msl shaders"
  ```
