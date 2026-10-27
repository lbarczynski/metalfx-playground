# TASK-17A: Halton(2,3) Jitter Generator & Unit Tests

## 1. Concept & Mental Model

### Why Low-Discrepancy Sequences?
To reconstruct high-frequency details across frames, the camera projection matrix must sample different subpixel locations within each low-resolution pixel.
- **Random Uniform Noise**: Clumps samples together and leaves large gaps, causing temporal flickering and noisy boiling artifacts.
- **Halton(2, 3) Sequence**: A quasi-random, low-discrepancy sequence that maximizes the physical distance between consecutive sample points, covering the unit square evenly.

---

## 2. Mathematical Foundations & Algorithms

### Van der Corput Radical Inverse
For an integer index $i$ represented in base $b$:
$$i = \sum_{k=0}^{M} d_k \cdot b^k$$
The radical inverse $\Phi_b(i)$ reflects the digits around the decimal point:
$$\Phi_b(i) = \sum_{k=0}^{M} d_k \cdot b^{-(k+1)}$$

### Halton(2, 3) Generator
For sample index $i \ge 1$:
$$\mathbf{H}(i) = (\Phi_2(i), \Phi_3(i)) \in [0.0, 1.0) \times [0.0, 1.0)$$
To center the jitter phase within the pixel:
$$j_x = \Phi_2(i) - 0.5, \quad j_y = \Phi_3(i) - 0.5 \quad \implies j_x, j_y \in [-0.5, 0.5)$$

---

## 3. Metal & C++ Blueprint

### Jitter Generator Class (`src/HaltonJitter.hpp`)
```cpp
#pragma once
#include <simd/simd.h>
#include <array>

class HaltonJitter {
public:
    static constexpr size_t kPhaseCount = 16;

    static float radicalInverse(uint32_t index, uint32_t base) {
        float result = 0.0f;
        float invBase = 1.0f / static_cast<float>(base);
        float fraction = invBase;
        while (index > 0) {
            result += static_cast<float>(index % base) * fraction;
            index /= base;
            fraction *= invBase;
        }
        return result;
    }

    static simd::float2 sample(size_t index) {
        uint32_t sampleIndex = static_cast<uint32_t>((index % kPhaseCount) + 1); // 1-indexed
        return simd::make_float2(
            radicalInverse(sampleIndex, 2) - 0.5f,
            radicalInverse(sampleIndex, 3) - 0.5f
        );
    }
};
```

### Catch2 Unit Test (`tests/test_math.cpp`)
```cpp
TEST_CASE("Halton(2,3) Jitter Generator: Subpixel Range Bounds", "[math]") {
    for (size_t i = 0; i < HaltonJitter::kPhaseCount; ++i) {
        simd::float2 j = HaltonJitter::sample(i);
        // Offsets must strictly lie within [-0.5, 0.5)
        REQUIRE(j.x >= -0.5f);
        REQUIRE(j.x < 0.5f);
        REQUIRE(j.y >= -0.5f);
        REQUIRE(j.y < 0.5f);
    }
}

TEST_CASE("Halton(2,3) First Sample Coordinates", "[math]") {
    simd::float2 j1 = HaltonJitter::sample(0); // index 1: 1/2 - 0.5 = 0.0, 1/3 - 0.5 = -0.166667
    REQUIRE_THAT(j1.x, Catch::Matchers::WithinAbs(0.0f, 1e-4f));
    REQUIRE_THAT(j1.y, Catch::Matchers::WithinAbs(1.0f / 3.0f - 0.5f, 1e-4f));
}
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Implement `HaltonJitter` with `radicalInverse` for bases 2 and 3.
- [ ] Center the sequence values around 0: subtract $0.5$.
- [ ] Add unit tests verifying that all 16 sample phases stay within $[-0.5, 0.5)$.
- [ ] Run CTest to verify all tests pass: `ctest --preset macos-debug-test`.

---

## 5. Definition of Done & Commit Message
- **DoD**: Catch2 unit test suite verifies that 16-phase Halton(2,3) subpixel offsets strictly lie within the unit subpixel domain $[-0.5, 0.5)$.
- **Commit**:
  ```bash
  git add src/HaltonJitter.hpp tests/test_math.cpp
  git commit -m "feat(math): implement halton23 subpixel jitter generator with unit tests"
  ```
