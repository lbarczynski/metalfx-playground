# TASK-07: Metal Compute Pipeline (CUDA vs Metal Mental Model)

## 1. Concept & Mental Model
Metal compute provides direct General-Purpose GPU (GPGPU) computing for massively parallel data processing (e.g. physics simulations, particle advection, procedural synthesis) decoupled from the rasterizer. CUDA developers map CUDA grids, blocks, and warps directly to Metal grids, threadgroups, and SIMD-groups; however, Metal executes compute kernels via an explicit command encoder (`MTL::ComputeCommandEncoder`) without custom compiler syntax (`<<<grid, block>>>`).

## 2. Mathematical Foundations & Algorithms
### Thread Hierarchy Mapping (CUDA vs Metal)
| CUDA Concept | Metal Equivalent | Apple Silicon Value / Behavior |
| :--- | :--- | :--- |
| **Thread** | **Thread** | Single SIMD lane execution |
| **Warp (32 threads)** | **SIMD-group** | Strictly **32 lanes** on Apple Silicon |
| **Block / Thread Block** | **Threadgroup** | Up to 1024 threads; shared local memory |
| **Shared Memory (`__shared__`)** | **`threadgroup` memory** | 32 KB high-bandwidth on-chip SRAM |
| **Grid** | **Grid** | 1D, 2D, or 3D execution extent |

### Dispatch Dimension Formula
For $N$ linear elements and a 1D threadgroup size $T_x$ (typically 256):
$$\text{Threadgroups}_x = \left\lceil \frac{N}{T_x} \right\rceil = \frac{N + T_x - 1}{T_x}$$

In modern Metal (macOS 10.15+ / Metal 2+), `dispatchThreads` allows non-uniform grids without boundary padding:
$$\text{GridSize} = \text{MTL::Size::Make}(N, 1, 1), \quad \text{ThreadgroupSize} = \text{MTL::Size::Make}(T_x, 1, 1)$$

### Particle Euler Advection Algorithm
$$\mathbf{p}_i^{t + \Delta t} = \mathbf{p}_i^t + \mathbf{v}_i^t \Delta t$$
$$\text{Boundary Reflection}: \text{if } |p_{i,x}| > 1.0 \implies v_{i,x} = -v_{i,x}$$

## 3. Metal & C++ Blueprint

### 1. Shared Data Struct Layout (`SharedTypes.h`)
```cpp
#pragma once
#include <simd/simd.h>

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

kernel void updateParticles(device Particle* particles [[buffer(0)]],
                            constant ComputeParams& params [[buffer(1)]],
                            uint id [[thread_position_in_grid]]) {
    if (id >= params.particleCount) return;

    Particle p = particles[id];
    p.position += p.velocity * params.dt;

    // Boundary reflection
    if (abs(p.position.x) > 1.0f) { p.velocity.x *= -1.0f; }
    if (abs(p.position.y) > 1.0f) { p.velocity.y *= -1.0f; }

    particles[id] = p;
}
```

### 3. Compute Pipeline & Dispatch Execution (C++)
```cpp
#include <Metal/Metal.hpp>
#include <iostream>

// 1. Pipeline Creation
NS::Error* pError = nullptr;
MTL::Function* pKernelFunc = pLibrary->newFunction(
    NS::String::string("updateParticles", NS::UTF8StringEncoding)
);
MTL::ComputePipelineState* pComputePipeline = pDevice->newComputePipelineState(pKernelFunc, &pError);
pKernelFunc->release();

// 2. Buffer Allocation
constexpr uint32_t kNumParticles = 100000;
MTL::Buffer* pParticleBuffer = pDevice->newBuffer(
    sizeof(Particle) * kNumParticles,
    MTL::ResourceStorageModeShared
);

// 3. Command Encoding
MTL::CommandBuffer* cmd = pQueue->commandBuffer();
MTL::ComputeCommandEncoder* compEnc = cmd->computeCommandEncoder();

compEnc->setComputePipelineState(pComputePipeline);
compEnc->setBuffer(pParticleBuffer, 0, 0);

ComputeParams params = { .dt = 0.016f, .particleCount = kNumParticles };
// setBytes is optimized for small uniform data (<= 4 KB)
compEnc->setBytes(&params, sizeof(ComputeParams), 1);

// Threadgroup calculation
NS::UInteger maxThreads = pComputePipeline->maxTotalThreadsPerThreadgroup();
NS::UInteger threadgroupSize = std::min(static_cast<NS::UInteger>(256), maxThreads);

MTL::Size gridSize = MTL::Size::Make(kNumParticles, 1, 1);
MTL::Size tgSize = MTL::Size::Make(threadgroupSize, 1, 1);

// Modern non-uniform grid dispatch
compEnc->dispatchThreads(gridSize, tgSize);
compEnc->endEncoding();

// Synchronize and verify CPU readback
cmd->commit();
cmd->waitUntilCompleted();

Particle* results = static_cast<Particle*>(pParticleBuffer->contents());
std::cout << "First Particle Pos: (" << results[0].position.x << ", " << results[0].position.y << ")\n";
```

## 4. Step-by-Step Implementation Checklist
- [ ] Define `Particle` and `ComputeParams` struct in a shared header.
- [ ] Write the MSL compute kernel using `kernel void ... [[thread_position_in_grid]]`.
- [ ] Compile kernel function into `MTL::ComputePipelineState`.
- [ ] Allocate unified memory buffer (`MTL::ResourceStorageModeShared`) populated with initial particle positions.
- [ ] Create `MTL::ComputeCommandEncoder` via `cmd->computeCommandEncoder()`.
- [ ] Query `pipelineState->maxTotalThreadsPerThreadgroup()` to ensure threadgroup sizing fits within hardware limits.
- [ ] Dispatch threads using `compEnc->dispatchThreads(gridSize, threadgroupSize)`.
- [ ] End encoding, commit command buffer, and call `cmd->waitUntilCompleted()`.
- [ ] Inspect buffer contents on CPU to verify numerical updates.

## 5. Common Pitfalls & Debugging Checklist
- **Threadgroup Dimension Exceeds Hardware Limits**:
  - *Error*: Metal validation layer throws: `threadgroupSize.width (1024) must be <= maxTotalThreadsPerThreadgroup (512)`.
  - *Fix*: Always query `pComputePipeline->maxTotalThreadsPerThreadgroup()` before configuring threadgroup sizes.
- **Race Condition / Unsynchronized Reads**:
  - *Symptom*: CPU reads old data before the GPU finishes compute execution.
  - *Fix*: Wait for completion (`cmd->waitUntilCompleted()`) or register an asynchronous `addCompletedHandler`.
- **Out of Bounds Memory Writes**:
  - *Symptom*: GPU page fault or OS watchdog resets GPU driver.
  - *Fix*: Always guard thread indices in MSL: `if (id >= params.particleCount) return;`.
