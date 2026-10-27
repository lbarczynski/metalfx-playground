# TASK-01: CMake, metal-cpp Integration & Device Discovery

## 1. Concept & Mental Model

### The Kotlin / Android Bridge
In Android, accessing the GPU typically occurs through high-level abstractions like Vulkan (`VkInstance`, `VkPhysicalDevice`), OpenGL ES (`EGLContext`), or Android's RenderThread. Hardware feature checks involve `PackageManager.hasSystemFeature()` or querying EGL configuration lists. Memory is governed by the JVM Garbage Collector.

In Modern C++ on Apple Silicon:
- **Zero Overhead**: `metal-cpp` is a header-only C++20 wrapper around native Objective-C Metal frameworks. Method calls inline directly to Objective-C runtime messages (`objc_msgSend`) without wrapper object overhead.
- **Explicit Lifetimes**: Objects returned from `CreateSystemDefaultDevice()` or `alloc()->init()` follow explicit reference counting. You must release them manually with `->release()` within an active `NS::AutoreleasePool`.
- **Direct Discovery**: Unlike Vulkan's verbose physical device enumeration loop (`vkEnumeratePhysicalDevices`), Metal gives you the system default GPU via a single function call: `MTL::CreateSystemDefaultDevice()`.

---

## 2. Mathematical Foundations & Architecture

### Unified Memory Architecture (UMA)
Apple Silicon integrates CPU and GPU execution cores on a single physical die sharing high-bandwidth, unified LPDDR memory:
$$\text{Memory Pool}_{\text{Total}} = \text{RAM}_{\text{Unified}}$$
$$\text{Bus Overhead} = 0 \text{ (zero PCIe copy overhead, cache-coherent physical address space)}$$

Unlike discrete GPUs where data must travel over PCIe buses ($16\text{–}64 \text{ GB/s}$), Apple Silicon allows both CPU and GPU to access identical physical address spaces simultaneously without staging buffers.

### GPU Family & Feature Hierarchy
Metal device capabilities are queried via feature sets and GPU families:
```
MTL::GPUFamily::GPUFamilyMetal3   --> Modern macOS 13+ tier (Hardware Raytracing, MetalFX, Mesh Shaders)
MTL::GPUFamily::GPUFamilyApple9    --> M3 / A17 Pro (Dynamic Caching, Hardware Mesh Shading)
MTL::GPUFamily::GPUFamilyApple8    --> M2 / A16
MTL::GPUFamily::GPUFamilyApple7    --> M1 / A14
```

> [!NOTE]
> In `metal-cpp` headers from `macOS15_iOS18`, `GPUFamilyApple9` is the highest enumerated Apple family. Queries for newer families can be checked using integer values or Metal 3 feature set flags.

---

## 3. Metal & C++ Blueprint

### 1. Translation Unit for Implementation Symbols (`src/metal_impl.cpp`)
Exactly **one** `.cpp` file in your application must define implementation macros before including `metal-cpp` headers:
```cpp
#define NS_PRIVATE_IMPLEMENTATION
#define CA_PRIVATE_IMPLEMENTATION
#define MTL_PRIVATE_IMPLEMENTATION
#define MTLFX_PRIVATE_IMPLEMENTATION

#include <Foundation/Foundation.hpp>
#include <Metal/Metal.hpp>
#include <QuartzCore/QuartzCore.hpp>
#include <MetalFX/MetalFX.hpp>
```

### 2. Device Discovery & Query (`src/main.cpp`)
```cpp
#include <Foundation/Foundation.hpp>
#include <Metal/Metal.hpp>
#include <iostream>

int main() {
    // Autorelease pool required for Objective-C reference lifecycle
    NS::AutoreleasePool* pool = NS::AutoreleasePool::alloc()->init();

    MTL::Device* device = MTL::CreateSystemDefaultDevice();
    if (!device) {
        std::cerr << "CRITICAL: No Metal-compatible GPU found.\n";
        pool->release();
        return 1;
    }

    std::cout << "========================================\n";
    std::cout << "         METAL DEVICE DISCOVERY         \n";
    std::cout << "========================================\n";
    std::cout << "Device Name          : " << device->name()->utf8String() << "\n";
    std::cout << "Registry ID          : " << device->registryID() << "\n";
    std::cout << "Unified Memory (UMA) : " << (device->hasUnifiedMemory() ? "YES" : "NO") << "\n";
    std::cout << "Low Power Device     : " << (device->lowPower() ? "YES" : "NO") << "\n";
    std::cout << "Removable GPU (eGPU) : " << (device->removable() ? "YES" : "NO") << "\n";
    std::cout << "Max Buffer Length    : " << (device->maxBufferLength() / (1024 * 1024)) << " MB\n";

    // Feature set verification
    std::cout << "----------------------------------------\n";
    std::cout << "Feature Set Support:\n";
    std::cout << "  - Metal 3 Hardware : " 
              << (device->supportsFamily(MTL::GPUFamily::GPUFamilyMetal3) ? "Supported" : "Unsupported") << "\n";
    std::cout << "  - Apple 9 (M3/A17) : " 
              << (device->supportsFamily(MTL::GPUFamily::GPUFamilyApple9) ? "Supported" : "Unsupported") << "\n";
    std::cout << "  - Apple 8 (M2/A16) : " 
              << (device->supportsFamily(MTL::GPUFamily::GPUFamilyApple8) ? "Supported" : "Unsupported") << "\n";
    std::cout << "  - Apple 7 (M1/A14) : " 
              << (device->supportsFamily(MTL::GPUFamily::GPUFamilyApple7) ? "Supported" : "Unsupported") << "\n";
    std::cout << "========================================\n";

    // Explicit release of allocated objects
    device->release();
    pool->release();

    return 0;
}
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Create `src/metal_impl.cpp` with private implementation defines.
- [ ] Create `src/main.cpp` with `NS::AutoreleasePool`, `MTL::CreateSystemDefaultDevice()`, and GPU capability queries.
- [ ] Run CMake presets to configure: `cmake --preset macos-debug`.
- [ ] Build the executable: `cmake --build --preset macos-debug-build`.
- [ ] Run `./build/macos-debug/src/metalfx_app` and observe your Apple Silicon GPU parameters.

---

## 5. Definition of Done & Commit Message
- **DoD**: Executable compiles without warnings (`-Wall -Wextra -Wpedantic -Wconversion`) and outputs GPU name, UMA status, and Metal 3 support to the terminal.
- **Commit**:
  ```bash
  git add src/metal_impl.cpp src/main.cpp
  git commit -m "feat(infra): setup cmake and query metal device"
  ```

---

## 6. Common Pitfalls & Debugging Checklist
- **Duplicate Symbol Linker Errors (`ld: duplicate symbol ...`)**:
  - *Cause*: Defining `MTL_PRIVATE_IMPLEMENTATION` in multiple `.cpp` files or inside a header (`.hpp`).
  - *Fix*: Keep implementation macros strictly inside `src/metal_impl.cpp`.
- **Memory Leaks from Unreleased Objects**:
  - *Cause*: Omitting `device->release()` or failing to wrap code inside `NS::AutoreleasePool`.
  - *Fix*: Any object created with `Create...`, `alloc()->init()`, or `new...` must have a matching `release()` call.
