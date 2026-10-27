# TASK-04: Transformations, Dynamic Uniforms & Math Primitives

## 1. Concept & Mental Model
Dynamic uniforms transfer per-frame CPU simulation state (matrices, time, color tints) to GPU shaders via constant memory buffers. In OpenGL, uniform variables are set via `glUniformMatrix4fv` or uniform buffer objects (UBOs); in Vulkan/DirectX 12, dynamic descriptor offsets or push constants are used. In Metal, uniform buffers are bound using `setVertexBuffer:offset:atIndex:`, utilizing a triple-buffering ring-buffer pattern to avoid CPU-GPU race conditions and synchronization stalls.

## 2. Mathematical Foundations & Algorithms
### 2D Affine Transformation in Homogeneous Coordinates
Any sequence of 2D translation $\mathbf{T}$, rotation $\mathbf{R}$, and scale $\mathbf{S}$ can be represented as a $4 \times 4$ column-major matrix operating on homogeneous vectors:
$$\mathbf{v}' = \mathbf{M} \cdot \mathbf{v}$$
$$\mathbf{M} = \mathbf{T}(t_x, t_y) \cdot \mathbf{R}_z(\theta) \cdot \mathbf{S}(s_x, s_y)$$

### Rotation Matrix around Z-Axis ($\mathbf{R}_z(\theta)$)
$$\mathbf{R}_z(\theta) = \begin{bmatrix}
\cos\theta & -\sin\theta & 0 & 0 \\
\sin\theta &  \cos\theta & 0 & 0 \\
0          &  0          & 1 & 0 \\
0          &  0          & 0 & 1
\end{bmatrix}$$

### Metal Constant Buffer Offset Alignment Rule
Hardware memory controllers on Apple Silicon mandate that any dynamic buffer offset passed to `setVertexBuffer:offset:atIndex:` must be a multiple of **256 bytes**:
$$\text{Offset}_{\text{Uniform}} \equiv 0 \pmod{256}$$
$$\text{AlignedSize} = (\text{sizeof}(\text{Uniforms}) + 255) \ \& \ \sim 255$$

## 3. Metal & C++ Blueprint

### 1. Uniform Structure & Matrix Math Helpers
```cpp
#include <simd/simd.h>
#include <cmath>

struct alignas(16) FrameUniforms {
    simd::float4x4 modelMatrix;
    simd::float4   colorTint;
    float          time;
    float          _pad[3]; // Align to 16 bytes
};

inline simd::float4x4 makeZRotation(float angleRadians) {
    float c = std::cos(angleRadians);
    float s = std::sin(angleRadians);
    simd::float4x4 m = matrix_identity_float4x4;
    m.columns[0] = simd::make_float4( c, s, 0.0f, 0.0f);
    m.columns[1] = simd::make_float4(-s, c, 0.0f, 0.0f);
    m.columns[2] = simd::make_float4(0.0f, 0.0f, 1.0f, 0.0f);
    m.columns[3] = simd::make_float4(0.0f, 0.0f, 0.0f, 1.0f);
    return m;
}
```

### 2. Triple-Buffering Concurrency Architecture
```cpp
#include <dispatch/dispatch.h>

class DynamicUniformManager {
public:
    static constexpr size_t kMaxFramesInFlight = 3;

    DynamicUniformManager(MTL::Device* pDevice) {
        _semaphore = dispatch_semaphore_create(kMaxFramesInFlight);
        
        // Compute 256-byte aligned stride (preventing signed conversion warnings)
        _alignedUniformSize = (sizeof(FrameUniforms) + 255) & ~size_t(255);
        
        // Single unified buffer for all 3 frames
        _pUniformBuffer = pDevice->newBuffer(
            _alignedUniformSize * kMaxFramesInFlight,
            MTL::ResourceStorageModeShared
        );
    }

    ~DynamicUniformManager() {
        _pUniformBuffer->release();
    }

    void beginFrame() {
        // Wait for GPU to finish frame (N - 3)
        dispatch_semaphore_wait(_semaphore, DISPATCH_TIME_FOREVER);
    }

    void updateUniforms(const FrameUniforms& uniforms) {
        size_t currentOffset = _frameIndex * _alignedUniformSize;
        uint8_t* pContents = static_cast<uint8_t*>(_pUniformBuffer->contents()) + currentOffset;
        std::memcpy(pContents, &uniforms, sizeof(FrameUniforms));
    }

    void bind(MTL::RenderCommandEncoder* pEncoder, uint32_t bufferIndex) {
        size_t currentOffset = _frameIndex * _alignedUniformSize;
        pEncoder->setVertexBuffer(_pUniformBuffer, currentOffset, bufferIndex);
    }

    void endFrame(MTL::CommandBuffer* pCmd) {
        dispatch_semaphore_t sem = _semaphore;
        pCmd->addCompletedHandler(^(MTL::CommandBuffer*) {
            dispatch_semaphore_signal(sem);
        });

        _frameIndex = (_frameIndex + 1) % kMaxFramesInFlight;
    }

private:
    MTL::Buffer* _pUniformBuffer = nullptr;
    size_t _alignedUniformSize = 0;
    size_t _frameIndex = 0;
    dispatch_semaphore_t _semaphore;
};
```

### 3. MSL Shader with Dynamic Uniforms
```metal
#include <metal_stdlib>
using namespace metal;

struct VertexInput {
    float2 position [[attribute(0)]];
    float4 color    [[attribute(1)]];
};

struct FrameUniforms {
    float4x4 modelMatrix;
    float4   colorTint;
    float    time;
    float    _pad[3]; // Explicit padding matching C++ 96-byte layout
};

struct RasterizerData {
    float4 position [[position]];
    float4 color;
};

vertex RasterizerData vertexShader(VertexInput in [[stage_in]],
                                   constant FrameUniforms& uniforms [[buffer(1)]]) {
    RasterizerData out;
    float4 pos = float4(in.position, 0.0, 1.0);
    out.position = uniforms.modelMatrix * pos;
    out.color = in.color * uniforms.colorTint;
    return out;
}

fragment float4 fragmentShader(RasterizerData in [[stage_in]]) {
    return in.color;
}
```

## 4. Step-by-Step Implementation Checklist
- [ ] Add `-fblocks` to CMake compile options to enable Apple Clang block syntax (`^(MTL::CommandBuffer*)`).
- [ ] Define `FrameUniforms` struct with `alignas(16)`.
- [ ] Calculate `alignedUniformSize = (sizeof(FrameUniforms) + 255) & ~size_t(255)`.
- [ ] Allocate a single shared `MTL::Buffer` sized `alignedUniformSize * 3`.
- [ ] Initialize GCD semaphore with 3 permits: `dispatch_semaphore_create(3)`.
- [ ] In `drawInMTKView()`, ensure `beginFrame()` is called **only inside** `if (passDesc && drawable)` to prevent semaphore deadlock on minimized windows.
- [ ] Calculate rotation angle $\theta = \omega t$ and update uniform memory.
- [ ] Bind uniform buffer: `enc->setVertexBuffer(buffer, currentOffset, 1)`.
- [ ] Register `addCompletedHandler` to signal semaphore when GPU finishes execution.
- [ ] Run application; observe smooth continuous rotation without CPU-GPU stalls or tearing.

## 5. Common Pitfalls & Debugging Checklist
- **256-Byte Offset Alignment Assertion**:
  - *Error*: `setVertexBuffer:offset:atIndex: offset (X) must be a multiple of 256`.
  - *Fix*: Enforce bitmask rounding: `(size + 255) & ~255`.
- **CPU-GPU Data Races & Flickering**:
  - *Symptom*: Rendered geometry exhibits violent tearing or glitching.
  - *Cause*: Mutating the uniform buffer while the GPU is concurrently executing a previous frame.
  - *Fix*: Implement triple-buffering with semaphore synchronization.
- **Semaphore Deadlock**:
  - *Symptom*: Application hangs indefinitely on `dispatch_semaphore_wait`.
  - *Cause*: Failing to call `dispatch_semaphore_signal` when command buffer fails to commit or encounters an error.
  - *Fix*: Always register `addCompletedHandler` before `cmd->commit()`.
