# TASK-07: Metal Compute Pipeline & CPU Readback

## 1. Concept & Mental Model

### The Kotlin / Android Bridge
In Android, computationally heavy algorithms (like particle physics or image filters) are either run on CPU worker threads (Coroutines / `Dispatchers.Default`), executed via RenderScript (now deprecated), or dispatched via Vulkan compute shaders or OpenCL.

In Modern C++ and Metal:
- **Massively Parallel GPGPU Compute**: Metal compute pipelines allow you to use the GPU's thousands of execution cores for non-graphics data processing.
- **Unified Address Space**: Because Apple Silicon has Unified Memory, the CPU can read the GPU's computed output directly from `MTL::Buffer::contents()` with zero PCIe transfer overhead once execution completes.
- **Explicit Command Encoding**: Unlike CUDA's custom language syntax (`kernel<<<grid, block>>>`), Metal encodes compute passes into standard `MTL::CommandBuffer` instances using `MTL::ComputeCommandEncoder`.

---

## 2. Mathematical Foundations & Algorithms

### Thread Hierarchy Mapping
| Concept | Metal Equivalent | Apple Silicon Behavior |
| :--- | :--- | :--- |
| **Individual Thread** | **Thread** | Single SIMD lane execution |
| **Warp (32 threads)** | **SIMD-group** | Strictly **32 lanes** on Apple Silicon |
| **Thread Block** | **Threadgroup** | Up to 1024 threads; shared on-chip SRAM |
| **Grid** | **Grid** | 1D, 2D, or 3D execution extent |

### Particle Euler Advection
For $N$ particles updated across time step $\Delta t$:
$$\mathbf{p}_i^{t + \Delta t} = \mathbf{p}_i^t + \mathbf{v}_i^t \Delta t$$
$$\text{Boundary Reflection}: \text{if } |p_{i,x}| > 1.0 \implies v_{i,x} = -v_{i,x}$$

---

## 3. Metal & C++ Blueprint

### 1. Shared Types Header (`src/ComputeTypes.hpp`)
```cpp
#pragma once
#include <simd/simd.h>
#include <cstdint>

struct Particle {
    simd::float2 position;
    simd::float2 velocity;
};

struct ComputeParams {
    float dt;
    uint32_t particleCount;
};
```

### 2. MSL Compute Kernel
```metal
#include <metal_stdlib>
using namespace metal;

struct Particle {
    float2 position;
    float2 velocity;
};

struct ComputeParams {
    float dt;
    uint particleCount;
};

kernel void updateParticles(
    device Particle* particles [[buffer(0)]],
    constant ComputeParams& params [[buffer(1)]],
    uint id [[thread_position_in_grid]]
) {
    if (id >= params.particleCount) return;

    Particle p = particles[id];
    p.position += p.velocity * params.dt;

    // Bounce off NDC boundaries
    if (abs(p.position.x) > 1.0f) { p.velocity.x *= -1.0f; }
    if (abs(p.position.y) > 1.0f) { p.velocity.y *= -1.0f; }

    particles[id] = p;
}
```

### 3. Compute Pipeline & Execution in C++
```cpp
// 1. Create compute pipeline state
MTL::Function* pKernelFunc = pLibrary->newFunction(
    NS::String::string("updateParticles", NS::UTF8StringEncoding)
);
MTL::ComputePipelineState* pComputePipeline = pDevice->newComputePipelineState(pKernelFunc, &pError);
pKernelFunc->release();

// 2. Allocate and initialize particles buffer
constexpr uint32_t kNumParticles = 10000;
MTL::Buffer* pParticleBuffer = pDevice->newBuffer(
    sizeof(Particle) * kNumParticles,
    MTL::ResourceStorageModeShared
);
// [Initialize particle positions and velocities...]

// 3. Encode compute dispatch
MTL::CommandBuffer* cmd = pQueue->commandBuffer();
MTL::ComputeCommandEncoder* compEnc = cmd->computeCommandEncoder();

compEnc->setComputePipelineState(pComputePipeline);
compEnc->setBuffer(pParticleBuffer, 0, 0);

ComputeParams params = { .dt = 0.016f, .particleCount = kNumParticles };
compEnc->setBytes(&params, sizeof(ComputeParams), 1);

// Threadgroup sizing
NS::UInteger threadgroupSize = std::min(
    static_cast<NS::UInteger>(256),
    pComputePipeline->maxTotalThreadsPerThreadgroup()
);
MTL::Size gridSize = MTL::Size::Make(kNumParticles, 1, 1);
MTL::Size tgSize = MTL::Size::Make(threadgroupSize, 1, 1);

compEnc->dispatchThreads(gridSize, tgSize);
compEnc->endEncoding();

cmd->commit();
cmd->waitUntilCompleted();

// Readback results directly on CPU
Particle* results = static_cast<Particle*>(pParticleBuffer->contents());
std::cout << "Particle 0 updated position: (" << results[0].position.x << ", " << results[0].position.y << ")\n";
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Define `Particle` and `ComputeParams` struct in `src/ComputeTypes.hpp`.
- [ ] Write `updateParticles` kernel in MSL.
- [ ] Create `MTL::ComputePipelineState` from the kernel function.
- [ ] Allocate and populate `MTL::Buffer` with $10\,000$ particles in shared memory.
- [ ] Encode compute command with `dispatchThreads` using 256 threads per threadgroup.
- [ ] Commit command buffer, wait for GPU completion, and assert that particle positions changed on the CPU.

---

## 5. Definition of Done & Commit Message
- **DoD**: Compute pipeline executes on GPU, updates particle positions, and CPU asserts valid mathematical advection upon buffer readback.
- **Commit**:
  ```bash
  git add src/ComputeTypes.hpp src/Shaders.metal src/Renderer.cpp
  git commit -m "feat(compute): implement metal compute kernel with cpu readback"
  ```

---

## 6. Common Pitfalls & Debugging Checklist
- **Threadgroup Dimension Exceeds Hardware Limits**:
  - *Error*: `threadgroupSize.width must be <= maxTotalThreadsPerThreadgroup`.
  - *Fix*: Always query `pComputePipeline->maxTotalThreadsPerThreadgroup()` before configuring threadgroup sizes.
- **Race Condition / Unsynchronized Reads**:
  - *Symptom*: CPU reads old data before the GPU finishes compute execution.
  - *Fix*: Call `cmd->waitUntilCompleted()` or use `addCompletedHandler` before accessing CPU pointer.
