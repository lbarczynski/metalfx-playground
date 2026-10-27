# TASK-11: MetalFX Spatial Support & Device Capabilities

## 1. Concept & Mental Model

### The Kotlin / Android Bridge
In Android, scaling bitmaps or surfaces is usually handled by `RenderNode`, bilinear hardware scaling in the compositor (`SurfaceView` hardware scaling), or custom shaders.

In Modern Apple Silicon Graphics:
- **MetalFX Spatial**: Apple's hardware-accelerated, single-frame edge-adaptive upscaler (analogous to AMD FSR 1.0 or NVIDIA NIS).
- **Edge-Adaptive Reconstruction**: Instead of isotropic bilinear filtering (which blurs silhouettes and fine lines), MetalFX detects local image gradients $\nabla I$ and dynamically aligns filtering kernels along edges.
- **Low Overhead**: Operates in a single pass without needing depth buffers, jitter, or motion vectors. It is the perfect introduction to GPU upscaling.

---

## 2. Mathematical Foundations & Algorithms

### Spatial Edge-Adaptive Filtering vs Bilinear
Standard bilinear interpolation averages neighboring texels isotropically:
$$I_{\text{bilinear}}(\mathbf{p}) = \sum_{i} w(\|\mathbf{p} - \mathbf{p}_i\|) I(\mathbf{p}_i)$$
This causes blurry silhouettes because the kernel blurs across high-contrast edges.

MetalFX Spatial computes image gradients:
$$\nabla I(x, y) = \left( \frac{\partial I}{\partial x}, \frac{\partial I}{\partial y} \right)$$
$$\theta_{\text{edge}} = \text{atan2}\left(\frac{\partial I}{\partial y}, \frac{\partial I}{\partial x}\right)$$
Filtering is directed parallel to the edge tangent $\mathbf{t} = (-\sin\theta, \cos\theta)$, preserving sharp geometric silhouettes while smoothing interior aliasing.

### Render Fill-Rate Workload Reduction
Given scaling ratio $R = \frac{W_{\text{out}}}{W_{\text{in}}}$:
$$\text{Fill-Rate Savings} = 1.0 - \frac{1}{R^2}$$

| Quality Mode | Input Resolution | Output Resolution | Workload | Shading Savings |
|---|---|---|---|---|
| **Quality ($1.5\times$)** | $1280 \times 720$ | $1920 \times 1080$ | $44.4\%$ of native | **$55.6\%$ savings** |
| **Performance ($2.0\times$)** | $960 \times 540$ | $1920 \times 1080$ | $25.0\%$ of native | **$75.0\%$ savings** |

---

## 3. Metal & C++ Blueprint

### Querying Hardware Support (`src/MetalFXUtils.hpp`)
```cpp
#pragma once
#include <Metal/Metal.hpp>
#include <MetalFX/MetalFX.hpp>
#include <iostream>

inline bool queryMetalFXSpatialSupport(MTL::Device* pDevice) {
    bool supported = MTLFX::SpatialScalerDescriptor::supportsDevice(pDevice);
    
    std::cout << "========================================\n";
    std::cout << "         METALLIC HARDWARE FX QUERY     \n";
    std::cout << "========================================\n";
    std::cout << "MetalFX Spatial Scaler Supported : " << (supported ? "YES" : "NO") << "\n";
    std::cout << "========================================\n";

    return supported;
}
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Ensure CMake links `-framework MetalFX`.
- [ ] Define `#define MTLFX_PRIVATE_IMPLEMENTATION` in `src/metal_impl.cpp`.
- [ ] In application startup, call `MTLFX::SpatialScalerDescriptor::supportsDevice(pDevice)`.
- [ ] Log device capability to the console.
- [ ] Confirm that your Apple Silicon device reports `YES` for MetalFX Spatial.

---

## 5. Definition of Done & Commit Message
- **DoD**: Program queries and prints hardware support for MetalFX Spatial Scaler on Apple Silicon, verifying framework linking.
- **Commit**:
  ```bash
  git add src/MetalFXUtils.hpp src/main.cpp
  git commit -m "feat(metalfx): query hardware support for metalfx spatial scaler"
  ```
