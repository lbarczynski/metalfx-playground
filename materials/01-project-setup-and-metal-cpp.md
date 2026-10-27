# TASK-01: CMake, metal-cpp Integration & Device Discovery

## 1. Concept & Mental Model
Apple's `metal-cpp` is a zero-overhead, header-only C++20 wrapper around the native Objective-C Metal, Foundation, and QuartzCore frameworks, enabling direct GPU programming without writing Objective-C++ (`.mm`) code. In contrast to Vulkan's verbose physical device enumeration (`vkEnumeratePhysicalDevices`) or OpenGL's driver-managed context initialization, Metal provides immediate, explicit access to Apple Silicon GPU hardware via a single factory call (`MTL::CreateSystemDefaultDevice()`).

## 2. Mathematical Foundations & Algorithms
### Unified Memory Architecture (UMA)
Apple Silicon integrates CPU and GPU execution cores on a single physical die sharing high-bandwidth, unified LPDDR memory:
$$\text{Memory Pool}_{\text{Total}} = \text{RAM}_{\text{Unified}}$$
$$\text{Bus Transfer Overhead} = 0 \text{ (zero PCIe copy overhead, cache-coherent physical address space)}$$

Unlike discrete GPUs (PCIe transfer overhead: $\sim 16 \text{–} 64 \text{ GB/s}$), Apple Silicon allows both CPU and GPU to access identical physical address spaces simultaneously without staging buffers.

### GPU Family & Feature Hierarchy
Metal device capabilities are queried via feature sets and GPU families:
```
MTL::GPUFamily::GPUFamilyMetal3      --> Modern macOS 13+ tier (Hardware Raytracing, MetalFX, Mesh Shaders)
MTL::GPUFamily::GPUFamilyApple10     --> M4 / A18 (macOS 15+)
MTL::GPUFamily::GPUFamilyApple9      --> M3 / A17 Pro (Dynamic Caching, Hardware Mesh Shading)
MTL::GPUFamily::GPUFamilyApple8      --> M2 / A16
MTL::GPUFamily::GPUFamilyApple7      --> M1 / A14
```

## 3. Metal & C++ Blueprint

### 1. Project Directory Layout
```
├── CMakeLists.txt
├── external/
│   └── metal-cpp/          # Header-only metal-cpp directory from Apple
│       ├── Foundation/
│       ├── Metal/
│       ├── MetalKit/
│       └── QuartzCore/
└── src/
    ├── metal_impl.cpp      # Single translation unit for private implementations
    └── main.cpp            # Application entry point
```

### 2. Modern CMake Target Configuration (`CMakeLists.txt`)
```cmake
cmake_minimum_required(VERSION 3.25)
project(MetalDiscovery LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Enable compiler diagnostics
add_compile_options(-Wall -Wextra -Wpedantic -Wconversion)

# metal-cpp interface library (marked SYSTEM to suppress third-party header warnings under strict flags)
add_library(metal_cpp_interface INTERFACE)
target_include_directories(metal_cpp_interface SYSTEM INTERFACE
    ${CMAKE_CURRENT_SOURCE_DIR}/external/metal-cpp
)

# Executable target
add_executable(task01_discovery
    src/metal_impl.cpp
    src/main.cpp
)

target_link_libraries(task01_discovery PRIVATE
    metal_cpp_interface
    "-framework Metal"
    "-framework Foundation"
    "-framework QuartzCore"
    "-framework MetalKit"
)
```

### 3. Implementation Translation Unit (`src/metal_impl.cpp`)
Exactly one translation unit must define implementation macros before including `metal-cpp` headers to generate symbol definitions:
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

### 4. Device Discovery & Capability Query (`src/main.cpp`)
```cpp
#include <Foundation/Foundation.hpp>
#include <Metal/Metal.hpp>
#include <iostream>
#include <iomanip>

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
    std::cout << "Device Name             : " << device->name()->utf8String() << "\n";
    std::cout << "Registry ID             : " << device->registryID() << "\n";
    std::cout << "Unified Memory (UMA)    : " << (device->hasUnifiedMemory() ? "YES" : "NO") << "\n";
    std::cout << "Low Power Device        : " << (device->lowPower() ? "YES" : "NO") << "\n";
    std::cout << "Removable GPU (eGPU)    : " << (device->removable() ? "YES" : "NO") << "\n";
    std::cout << "Max Buffer Length       : " << (device->maxBufferLength() / (1024 * 1024)) << " MB\n";
    std::cout << "Max Threadgroup Dim     : ["
              << device->maxThreadsPerThreadgroup().width << ", "
              << device->maxThreadsPerThreadgroup().height << ", "
              << device->maxThreadsPerThreadgroup().depth << "]\n";

    // Feature set verification
    std::cout << "----------------------------------------\n";
    std::cout << "Feature Set Support:\n";
    std::cout << "  - Metal 3 Hardware    : " 
              << (device->supportsFamily(MTL::GPUFamily::GPUFamilyMetal3) ? "Supported" : "Unsupported") << "\n";
    std::cout << "  - Apple 9 (M3/A17)    : " 
              << (device->supportsFamily(MTL::GPUFamily::GPUFamilyApple9) ? "Supported" : "Unsupported") << "\n";
    std::cout << "  - Apple 8 (M2/A16)    : " 
              << (device->supportsFamily(MTL::GPUFamily::GPUFamilyApple8) ? "Supported" : "Unsupported") << "\n";
    std::cout << "  - Apple 7 (M1/A14)    : " 
              << (device->supportsFamily(MTL::GPUFamily::GPUFamilyApple7) ? "Supported" : "Unsupported") << "\n";
    std::cout << "========================================\n";

    // Cleanup resources
    device->release();
    pool->release();

    return 0;
}
```

## 4. Step-by-Step Implementation Checklist
- [ ] Create workspace directories: `mkdir -p external src build`.
- [ ] Download or vendor the official Apple `metal-cpp` headers into `external/metal-cpp`.
- [ ] Write `CMakeLists.txt` enforcing `CMAKE_CXX_STANDARD 20` and linking `-framework Metal`, `-framework Foundation`, `-framework QuartzCore`.
- [ ] Create `src/metal_impl.cpp` with `NS_PRIVATE_IMPLEMENTATION`, `CA_PRIVATE_IMPLEMENTATION`, and `MTL_PRIVATE_IMPLEMENTATION`.
- [ ] Create `src/main.cpp` using `NS::AutoreleasePool`, `MTL::CreateSystemDefaultDevice()`, and property logging.
- [ ] Generate build files: `cmake -B build -S . -DCMAKE_BUILD_TYPE=Debug`.
- [ ] Compile: `cmake --build build`.
- [ ] Execute `./build/task01_discovery` and verify Apple Silicon GPU architecture is detected with UMA enabled.

## 5. Common Pitfalls & Debugging Checklist
- **Duplicate Symbol Linker Errors (`ld: duplicate symbol ...`)**:
  - *Cause*: Defining `MTL_PRIVATE_IMPLEMENTATION` or `NS_PRIVATE_IMPLEMENTATION` in multiple `.cpp` files or inside a header (`.hpp`).
  - *Fix*: Ensure implementation macros exist in **only one** dedicated `.cpp` translation unit.
- **Undefined Reference Errors (`_MTLCreateSystemDefaultDevice`)**:
  - *Cause*: Missing framework linker flags.
  - *Fix*: In CMake, verify `-framework Metal`, `-framework Foundation`, and `-framework QuartzCore` are passed to `target_link_libraries`.
- **Memory Leaks from Unreleased Objects**:
  - *Cause*: Omitting `device->release()` or failing to wrap code inside `NS::AutoreleasePool`.
  - *Fix*: Follow metal-cpp ownership rules: any object created via `Create...`, `alloc()->init()`, or `new...` must have a matching `release()` call.
- **Null Device Return**:
  - *Cause*: Attempting to initialize Metal on unsupported hardware (e.g. legacy virtual machines without Metal acceleration).
  - *Fix*: Always assert `device != nullptr` before calling member methods.
