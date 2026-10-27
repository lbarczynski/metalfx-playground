# TASK-04B: Dynamic Uniforms & Ring Buffer Concurrency

## 1. Concept & Mental Model

### The Kotlin / Android Bridge
In Android, passing values to shaders often involves calling `GLES20.glUniformMatrix4fv` or setting uniforms in a Compose `ShaderBrush`. 

In Metal:
- **CPU-GPU Concurrency**: The CPU encodes frame $N$ while the GPU is executing frame $N-1$ and the display is showing frame $N-2$. If the CPU modifies a shared buffer while the GPU reads it, visual tearing or corruption occurs.
- **The 256-Byte Offset Rule**: Apple Silicon hardware mandates that any dynamic offset passed to `setVertexBuffer:offset:atIndex:` must be aligned to a multiple of **256 bytes**:
$$\text{AlignedSize} = (\text{sizeof}(\text{Uniforms}) + 255) \ \& \ \sim 255$$
- **Triple-Buffering with Semaphores**: We allocate a single unified buffer holding 3 slots (one for each in-flight frame). A Grand Central Dispatch (GCD) semaphore ensures the CPU never overwrites a slot until the GPU finishes reading it.

---

## 2. Concurrency Architecture

```
Ring Buffer Slots: [ Slot 0 (Frame N) ] [ Slot 1 (Frame N-1) ] [ Slot 2 (Frame N-2) ]
                          ▲                        ▲
                    CPU Writing              GPU Reading
```

---

## 3. Metal & C++ Blueprint

### 1. Dynamic Uniform Manager (`src/DynamicUniformManager.hpp`)
```cpp
#pragma once
#include <Metal/Metal.hpp>
#include <dispatch/dispatch.h>
#include "SimdMath.hpp"

class DynamicUniformManager {
public:
    static constexpr size_t kMaxFramesInFlight = 3;

    DynamicUniformManager(MTL::Device* pDevice) {
        _semaphore = dispatch_semaphore_create(kMaxFramesInFlight);
        _alignedSize = (sizeof(math::FrameUniforms) + 255) & ~size_t(255);
        _pBuffer = pDevice->newBuffer(_alignedSize * kMaxFramesInFlight, MTL::ResourceStorageModeShared);
    }

    ~DynamicUniformManager() {
        _pBuffer->release();
    }

    void beginFrame() {
        dispatch_semaphore_wait(_semaphore, DISPATCH_TIME_FOREVER);
    }

    void update(const math::FrameUniforms& uniforms) {
        uint8_t* pDest = reinterpret_cast<uint8_t*>(_pBuffer->contents()) + (_frameIndex * _alignedSize);
        memcpy(pDest, &uniforms, sizeof(math::FrameUniforms));
    }

    void bind(MTL::RenderCommandEncoder* enc, uint32_t bufferIndex) {
        enc->setVertexBuffer(_pBuffer, _frameIndex * _alignedSize, bufferIndex);
    }

    void endFrame(MTL::CommandBuffer* cmd) {
        dispatch_semaphore_t sema = _semaphore;
        cmd->addCompletedHandler([sema](MTL::CommandBuffer*) {
            dispatch_semaphore_signal(sema);
        });
        _frameIndex = (_frameIndex + 1) % kMaxFramesInFlight;
    }

private:
    MTL::Buffer* _pBuffer = nullptr;
    dispatch_semaphore_t _semaphore;
    size_t _alignedSize = 0;
    size_t _frameIndex = 0;
};
```

### 2. MSL Shader Update
```metal
struct FrameUniforms {
    float4x4 modelMatrix;
    float4   colorTint;
    float    time;
};

vertex VertexOut vertexMain(
    uint vertexID [[vertex_id]],
    constant VertexInput* vertices [[buffer(0)]],
    constant FrameUniforms& uniforms [[buffer(1)]]
) {
    VertexOut out;
    float4 localPos = float4(vertices[vertexID].position, 0.0, 1.0);
    out.position = uniforms.modelMatrix * localPos;
    out.color = vertices[vertexID].color * uniforms.colorTint.rgb;
    return out;
}
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Implement `DynamicUniformManager` with 256-byte aligned stride and GCD semaphore.
- [ ] In MSL, bind `constant FrameUniforms& uniforms [[buffer(1)]]` and transform positions by `uniforms.modelMatrix`.
- [ ] In `Renderer::renderFrame()`:
  - Call `uniformManager.beginFrame()`.
  - Calculate `makeZRotation(angle)` and update uniforms.
  - Call `uniformManager.bind(enc, 1)`.
  - Attach `cmd->addCompletedHandler` to signal the semaphore and advance frame index.
- [ ] Run application: The triangle or geometry should rotate smoothly without stuttering or memory corruption.

---

## 5. Definition of Done & Commit Message
- **DoD**: Geometry rotates continuously on screen driven by dynamic per-frame CPU uniform updates using triple-buffering without race conditions or memory leaks.
- **Commit**:
  ```bash
  git add src/DynamicUniformManager.hpp src/Renderer.cpp src/Shaders.metal
  git commit -m "feat(uniforms): implement dynamic triple-buffered uniform ring buffer"
  ```
