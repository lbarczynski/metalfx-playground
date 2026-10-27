# TASK-04A: C++ SIMD Math & Hardware Alignment

## 1. Concept & Mental Model

### The Kotlin / Android Bridge
In Kotlin, doing 3D vector arithmetic typically uses classes like `Vector3` or `Matrix4f`, which can cause object allocations, GC pressure, or manual array unwrapping (`FloatArray`).

In Modern C++ on Apple Silicon:
- **Direct Hardware Vectorization**: Apple's `<simd/simd.h>` maps types like `simd::float4` directly to 128-bit ARM NEON hardware registers. Vector additions and matrix multiplications execute in single CPU clock cycles.
- **Strict Hardware Alignment**: Apple Silicon GPUs and MSL require uniform structs to align to **16-byte boundaries**. In C++, we enforce this using `alignas(16)`.
- **Compile-Time Guarantees**: We use C++20 `static_assert` to prove at compile time that structs will never cause misaligned GPU memory reads.

---

## 2. Mathematical Foundations & Algorithms

### 2D Rotation Matrix ($\mathbf{R}_z(\theta)$)
Operating on homogeneous coordinates:
$$\mathbf{R}_z(\theta) = \begin{bmatrix}
\cos\theta & -\sin\theta & 0 & 0 \\
\sin\theta &  \cos\theta & 0 & 0 \\
0          &  0          & 1 & 0 \\
0          &  0          & 0 & 1
\end{bmatrix}$$

### Alignment Calculation
Every struct passed to Metal shaders must satisfy:
$$\text{sizeof}(\text{Struct}) \equiv 0 \pmod{16}$$

---

## 3. Metal & C++ Blueprint

### 1. Math Header (`src/SimdMath.hpp`)
```cpp
#pragma once
#include <simd/simd.h>
#include <cmath>

namespace math {

struct alignas(16) FrameUniforms {
    simd::float4x4 modelMatrix;
    simd::float4   colorTint;
    float          time;
    float          _pad[3]; // Pad to 16 bytes
};

static_assert(sizeof(FrameUniforms) % 16 == 0, "FrameUniforms must be 16-byte aligned for MSL!");

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

} // namespace math
```

### 2. Unit Testing with Catch2 (`tests/test_math.cpp`)
```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/matchers/catch_matchers_floating_point.hpp>
#include "SimdMath.hpp"

using Catch::Matchers::WithinAbs;

TEST_CASE("SIMD Math: FrameUniforms Alignment", "[math]") {
    REQUIRE(sizeof(math::FrameUniforms) % 16 == 0);
    REQUIRE(alignof(math::FrameUniforms) >= 16);
}

TEST_CASE("SIMD Math: Z-Rotation Identity at 0 Radians", "[math]") {
    simd::float4x4 rot = math::makeZRotation(0.0f);
    simd::float4 v = simd::make_float4(1.0f, 0.0f, 0.0f, 1.0f);
    simd::float4 res = simd_mul(rot, v);

    REQUIRE_THAT(res.x, WithinAbs(1.0f, 1e-5f));
    REQUIRE_THAT(res.y, WithinAbs(0.0f, 1e-5f));
}

TEST_CASE("SIMD Math: Z-Rotation 90 Degrees", "[math]") {
    constexpr float kPiOver2 = 1.57079632679f;
    simd::float4x4 rot = math::makeZRotation(kPiOver2);
    simd::float4 v = simd::make_float4(1.0f, 0.0f, 0.0f, 1.0f);
    simd::float4 res = simd_mul(rot, v);

    REQUIRE_THAT(res.x, WithinAbs(0.0f, 1e-5f));
    REQUIRE_THAT(res.y, WithinAbs(1.0f, 1e-5f));
}
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Create `src/SimdMath.hpp` with `alignas(16)` `FrameUniforms` and `makeZRotation`.
- [ ] Add `static_assert` enforcing 16-byte alignment.
- [ ] Create `tests/test_math.cpp` verifying rotation and struct sizes.
- [ ] Run CTest: `ctest --preset macos-debug-test --output-on-failure`.
- [ ] Confirm all tests pass green.

---

## 5. Definition of Done & Commit Message
- **DoD**: Catch2 unit test suite compiles and passes under CTest, validating SIMD math functions and memory alignment constraints.
- **Commit**:
  ```bash
  git add src/SimdMath.hpp tests/test_math.cpp
  git commit -m "test(math): implement simd matrix math with catch2 verification"
  ```
