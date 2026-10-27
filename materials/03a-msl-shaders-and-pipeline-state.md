# TASK-03A: MSL Shaders & Pipeline State Compilation

## 1. Concept & Mental Model

### The Kotlin / Android Bridge
In Android graphics (OpenGL ES or Skia/Compose runtime), shaders were historically compiled at runtime and state was set on a global state machine (`glBindTexture`, `glUseProgram`, `glEnable(GL_BLEND)`). This caused unpredictable driver stalls during gameplay.

In Metal:
- **Immutable Pipeline State Objects (PSO)**: An `MTL::RenderPipelineState` bakes your vertex shader, fragment shader, and pixel formats directly into GPU hardware machine code ahead of time.
- **Metal Shading Language (MSL)**: MSL is based on **C++14**, supporting templates, structs, and operator overloading.
- **Zero Run-Time Stalls**: Once an `MTL::RenderPipelineState` is created, binding it during a render pass is a virtually zero-cost operation.

---

## 2. Metal Shading Language (MSL) Fundamentals

```metal
#include <metal_stdlib>
using namespace metal;

struct VertexOut {
    float4 position [[position]];
    float3 color;
};

vertex VertexOut vertexMain(
    uint vertexID [[vertex_id]],
    constant float2* positions [[buffer(0)]],
    constant float3* colors    [[buffer(1)]]
) {
    VertexOut out;
    out.position = float4(positions[vertexID], 0.0, 1.0);
    out.color = colors[vertexID];
    return out;
}

fragment float4 fragmentMain(VertexOut in [[stage_in]]) {
    return float4(in.color, 1.0);
}
```

### Qualifiers Explained:
- `[[vertex_id]]`: Hardware-generated vertex index (0, 1, 2...).
- `[[position]]`: Built-in attribute telling the rasterizer this output is the homogeneous clip-space coordinate.
- `[[stage_in]]`: Informs the fragment shader to accept interpolated per-vertex attributes from the rasterizer.

---

## 3. Metal & C++ Blueprint

### Compiling Shaders from Source at Runtime (`src/ShaderCompiler.cpp`)
```cpp
#include <Metal/Metal.hpp>
#include <iostream>

MTL::RenderPipelineState* createTrianglePipeline(
    MTL::Device* pDevice,
    const char* shaderSource,
    MTL::PixelFormat colorFormat
) {
    NS::Error* pError = nullptr;

    // 1. Create MTLLibrary from source string
    NS::String* sourceStr = NS::String::string(shaderSource, NS::UTF8StringEncoding);
    MTL::CompileOptions* options = MTL::CompileOptions::alloc()->init();
    MTL::Library* pLibrary = pDevice->newLibrary(sourceStr, options, &pError);
    options->release();

    if (!pLibrary) {
        std::cerr << "Shader Compilation Failed: " 
                  << pError->localizedDescription()->utf8String() << "\n";
        return nullptr;
    }

    // 2. Extract vertex and fragment entry points
    auto vertName = NS::String::string("vertexMain", NS::UTF8StringEncoding);
    auto fragName = NS::String::string("fragmentMain", NS::UTF8StringEncoding);
    MTL::Function* pVertFunc = pLibrary->newFunction(vertName);
    MTL::Function* pFragFunc = pLibrary->newFunction(fragName);

    // 3. Configure Render Pipeline Descriptor
    MTL::RenderPipelineDescriptor* desc = MTL::RenderPipelineDescriptor::alloc()->init();
    desc->setVertexFunction(pVertFunc);
    desc->setFragmentFunction(pFragFunc);
    desc->colorAttachments()->object(0)->setPixelFormat(colorFormat);

    // 4. Compile into immutable GPU pipeline state
    MTL::RenderPipelineState* pState = pDevice->newRenderPipelineState(desc, &pError);
    if (!pState) {
        std::cerr << "Pipeline State Creation Failed: " 
                  << pError->localizedDescription()->utf8String() << "\n";
    }

    // Cleanup transient objects
    pVertFunc->release();
    pFragFunc->release();
    pLibrary->release();
    desc->release();

    return pState;
}
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Write MSL source string containing `vertexMain` and `fragmentMain`.
- [ ] Create `MTL::Library` using `device->newLibrary()`.
- [ ] Extract vertex and fragment functions by name.
- [ ] Configure `MTL::RenderPipelineDescriptor` matching your window's pixel format (`BGRA8Unorm_sRGB`).
- [ ] Compile `MTL::RenderPipelineState`.
- [ ] Verify error handling: Introduce an intentional syntax error in MSL and confirm informative compiler output in the console.

---

## 5. Definition of Done & Commit Message
- **DoD**: MSL shader source compiles successfully into an `MTL::RenderPipelineState` without compiler warnings or linker errors.
- **Commit**:
  ```bash
  git add src/ShaderCompiler.hpp src/ShaderCompiler.cpp
  git commit -m "feat(shader): compile msl vertex and fragment pipeline state"
  ```
