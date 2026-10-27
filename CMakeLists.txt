cmake_minimum_required(VERSION 3.25)
project(MetalFXPlayground LANGUAGES CXX OBJCXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# -----------------------------------------------------------------------------
# Compiler Diagnostics Interface Target
# -----------------------------------------------------------------------------
add_library(project_warnings INTERFACE)
target_compile_options(project_warnings INTERFACE
    -Wall
    -Wextra
    -Wpedantic
    -Wconversion
    -Wno-unused-parameter
    -fblocks
)

# -----------------------------------------------------------------------------
# Sanitizers (AddressSanitizer + UndefinedBehaviorSanitizer)
# -----------------------------------------------------------------------------
option(ENABLE_SANITIZERS "Enable ASan and UBSan" OFF)
if(ENABLE_SANITIZERS)
    message(STATUS "Enabling AddressSanitizer (ASan) and UndefinedBehaviorSanitizer (UBSan)")
    add_compile_options(-fsanitize=address,undefined -fno-omit-frame-pointer)
    add_link_options(-fsanitize=address,undefined)
endif()

# -----------------------------------------------------------------------------
# Dependency Management: Apple metal-cpp (Automated Ingestion)
# -----------------------------------------------------------------------------
include(FetchContent)

# Option to use local external/metal-cpp if provided, or download directly from Apple
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/external/metal-cpp/Metal/Metal.hpp")
    set(METAL_CPP_INCLUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/external/metal-cpp")
    message(STATUS "Using local metal-cpp from: ${METAL_CPP_INCLUDE_DIR}")
else()
    message(STATUS "Fetching Apple metal-cpp headers from Apple Developer CDN...")
    FetchContent_Declare(
        metal_cpp
        URL "https://developer.apple.com/metal/cpp/files/metal-cpp_macOS15_iOS18.zip"
        DOWNLOAD_EXTRACT_TIMESTAMP TRUE
    )
    FetchContent_MakeAvailable(metal_cpp)
    set(METAL_CPP_INCLUDE_DIR "${metal_cpp_SOURCE_DIR}")
endif()

add_library(metal_cpp_interface INTERFACE)
target_include_directories(metal_cpp_interface SYSTEM INTERFACE
    "${METAL_CPP_INCLUDE_DIR}"
)
target_link_libraries(metal_cpp_interface INTERFACE
    "-framework Metal"
    "-framework MetalFX"
    "-framework MetalKit"
    "-framework QuartzCore"
    "-framework Foundation"
    "-framework AppKit"
)

# -----------------------------------------------------------------------------
# Translation Unit for metal-cpp Implementation Symbols
# -----------------------------------------------------------------------------
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/src/metal_impl.cpp")
    add_library(metal_impl STATIC
        src/metal_impl.cpp
    )
    target_link_libraries(metal_impl PUBLIC metal_cpp_interface)
    target_compile_options(metal_impl PRIVATE -Wno-conversion -Wno-pedantic)
endif()

# -----------------------------------------------------------------------------
# Subdirectories: Application & Testing
# -----------------------------------------------------------------------------
add_subdirectory(src)

option(BUILD_TESTING "Build CTest test suite" ON)
if(BUILD_TESTING)
    enable_testing()
    add_subdirectory(tests)
endif()
