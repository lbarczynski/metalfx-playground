# TASK-11: MetalFX Architecture & Device Support Query

## 1. Concept & Mental Model
MetalFX Spatial is Apple's hardware-accelerated, single-frame edge-adaptive upscaler (analogous to AMD FSR 1.0 or NVIDIA NIS) that reconstructs high-resolution images from lower-resolution render targets using purely spatial color gradient analysis. Unlike temporal upscaling algorithms (DLSS 2+, FSR 2+, MetalFX Temporal) requiring camera jitter, depth buffers, and motion vectors, MetalFX Spatial operates as a low-overhead, single-pass filter with zero temporal history overhead and zero ghosting artifacts.

## 2. Mathematical Foundations & Algorithms
### Spatial Edge-Adaptive Filtering vs Linear Interpolation
Standard bilinear or bicubic filtering applies isotropic weight kernels $w(d)$, blurring sharp transitions because weights cross high-contrast boundaries:
$$I_{\text{bilinear}}(\mathbf{p}) = \sum_{i} w(\|\mathbf{p} - \mathbf{p}_i\|) I(\mathbf{p}_i)$$

MetalFX Spatial computes localized image gradients to detect high-frequency geometric and textural edges:
$$\nabla I(x, y) = \left( \frac{\partial I}{\partial x}, \frac{\partial I}{\partial y} \right), \quad \|\nabla I\| = \sqrt{\left(\frac{\partial I}{\partial x}\right)^2 + \left(\frac{\partial I}{\partial y}\right)^2}$$
$$\theta_{\text{edge}} = \text{atan2}\left(\frac{\partial I}{\partial y}, \frac{\partial I}{\partial x}\right)$$

The reconstruction filter is dynamically oriented along the edge tangent $\mathbf{t} = (-\sin\theta, \cos\theta)$ to preserve sharp boundaries while applying low-pass smoothing exclusively parallel to the edge, suppressing aliasing without blurring silhouettes.

### Render Fill-Rate Workload Reduction
Given scale ratio $R = \frac{W_{\text{out}}}{W_{\text{in}}} = \frac{H_{\text{out}}}{H_{\text{in}}}$:
$$\text{Pixel Ratio} = \frac{1}{R^2}$$
$$\text{Fill-Rate Workload Reduction} = 1.0 - \frac{1}{R^2}$$

| Scaling Ratio ($R$) | Input Resolution | Output Resolution | Pixel Reduction | Shading Savings |
| :--- | :--- | :--- | :--- | :--- |
| **$1.33\times$ (Ultra Quality)** | $1440 \times 810$ | $1920 \times 1080$ | $56.3\%$ of native | **$43.7\%$ savings** |
| **$1.50\times$ (Quality)** | $1280 \times 720$ | $1920 \times 1080$ | $44.4\%$ of native | **$55.6\%$ savings** |
| **$2.00\times$ (Performance)** | $960 \times 540$ | $1920 \times 1080$ | $25.0\%$ of native | **$75.0\%$ savings** |

## 3. Metal & C++ Blueprint

### 1. CMake Framework Linking
```cmake
target_link_libraries(task11_metalfx PRIVATE
    metal_cpp_interface
    "-framework Metal"
    "-framework MetalFX"
    "-framework Foundation"
    "-framework QuartzCore"
    "-framework MetalKit"
)
```

### 2. Implementation TU Macro Setup (`metal_impl.cpp`)
Include the MetalFX private implementation macro in your single implementation translation unit:
```cpp
#define NS_PRIVATE_IMPLEMENTATION
#define CA_PRIVATE_IMPLEMENTATION
#define MTL_PRIVATE_IMPLEMENTATION
#define MTLFX_PRIVATE_IMPLEMENTATION

#include <Foundation/Foundation.hpp>
#include <Metal/Metal.hpp>
#include <QuartzCore/QuartzCore.hpp>
#include <MetalFX/MetalFX.hpp>

#if __has_include(<MetalKit/MetalKit.hpp>)
#define MTK_PRIVATE_IMPLEMENTATION
#include <MetalKit/MetalKit.hpp>
#endif
```

### 3. Device Support & Format Query (C++)
```cpp
#include <Metal/Metal.hpp>
#include <MetalFX/MetalFX.hpp>
#include <iostream>

void queryMetalFXCapabilities(MTL::Device* pDevice) {
    std::cout << "========================================\n";
    std::cout << "      METALLIC HARDWARE FX QUERY        \n";
    std::cout << "========================================\n";

    // 1. Check hardware support for Spatial Scaler
    bool spatialSupported = MTLFX::SpatialScalerDescriptor::supportsDevice(pDevice);
    std::cout << "MetalFX Spatial Scaler Supported : " 
              << (spatialSupported ? "YES" : "NO") << "\n";

    // 2. Check hardware support for Temporal Scaler
    bool temporalSupported = MTLFX::TemporalScalerDescriptor::supportsDevice(pDevice);
    std::cout << "MetalFX Temporal Scaler Supported: " 
              << (temporalSupported ? "YES" : "NO") << "\n";

    if (!spatialSupported) {
        std::cerr << "CRITICAL: Current device does not support MetalFX Spatial Scaling.\n";
        return;
    }

    // 3. Supported pixel formats for Spatial Scaler
    // MetalFX Spatial supports RGBA8Unorm, BGRA8Unorm, RGBA16Float, BGRA8Unorm_sRGB
    std::cout << "----------------------------------------\n";
    std::cout << "Supported Primary Formats:\n";
    std::cout << "  - MTL::PixelFormatRGBA8Unorm       : Compatible\n";
    std::cout << "  - MTL::PixelFormatBGRA8Unorm_sRGB  : Compatible\n";
    std::cout << "  - MTL::PixelFormatRGBA16Float      : Compatible (HDR Pipeline)\n";
    std::cout << "========================================\n";
}
```

## 4. Step-by-Step Implementation Checklist
- [ ] Add `"-framework MetalFX"` to target link libraries in `CMakeLists.txt`.
- [ ] Define `#define MTLFX_PRIVATE_IMPLEMENTATION` in `metal_impl.cpp` before including `<MetalFX/MetalFX.hpp>`.
- [ ] In application startup, call `MTLFX::SpatialScalerDescriptor::supportsDevice(pDevice)`.
- [ ] Assert `supportsDevice == true` before initializing any spatial scaler descriptors.
- [ ] Log device name, UMA status, and MetalFX support flags to verify hardware capabilities on Apple Silicon (M1/M2/M3/M4).

## 5. Common Pitfalls & Debugging Checklist
- **Linker Error: Undefined symbols for architecture arm64 (`_OBJC_CLASS_$_MTLFXSpatialScalerDescriptor`)**:
  - *Cause*: Missing `-framework MetalFX` in CMake `target_link_libraries`.
  - *Fix*: Link `MetalFX.framework` explicitly in CMake.
- **Unsupported Hardware Return (`supportsDevice == false`)**:
  - *Cause*: Running in virtualized CI environments or older Intel Macs without Apple Silicon GPU family support (requires `GPUFamilyApple7` or `GPUFamilyMetal3`).
  - *Fix*: Implement a fallback code path to standard bilinear blitting when `supportsDevice` returns `false`.
