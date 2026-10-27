# TASK-17: Subpixel Camera Jitter & Halton Sequence

## 1. Concept & Mental Model
Temporal super-sampling relies on sampling different subpixel locations within each texel on successive frames so that accumulation recovers detail lost to low-resolution rasterization. While MSAA samples multiple locations in a single frame using fixed hardware positions, temporal upscaling distributes sample locations across frames using low-discrepancy sequences like Halton(2, 3), applying a subpixel phase shift to the camera's perspective projection matrix.

## 2. Mathematical Foundations & Algorithms
### Radical Inverse Function $\phi_b(i)$
For an integer index $i \ge 0$ expressed in base $b$ as $i = \sum_{j=0}^{M} a_j b^j$:
$$\phi_b(i) = \sum_{j=0}^{M} a_j b^{-j-1}$$

The 2D Halton point for index $i$ is:
$$\mathbf{h}_i = \left( \phi_2(i), \phi_3(i) \right) \in [0.0, 1.0)^2$$

### Subpixel Texel Jitter Offsets
Map $[0, 1)$ to centered subpixel offsets $[-0.5, 0.5]$ in input texel units:
$$j_x(i) = \phi_2(i) - 0.5, \quad j_y(i) = \phi_3(i) - 0.5$$

For a scaling factor $R = \frac{W_{\text{out}}}{W_{\text{in}}}$, the Halton sequence cycles over $N = 8 \cdot R^2$ frames (e.g. 16 or 32 samples).

### Jittered Perspective Projection Matrix
To shift rasterization by subpixel offset $(j_x, j_y)$ in input space $(W_{\text{in}}, H_{\text{in}})$:
$$\Delta x_{\text{clip}} = \frac{2 \cdot j_x}{W_{\text{in}}}, \quad \Delta y_{\text{clip}} = -\frac{2 \cdot j_y}{H_{\text{in}}}$$

Modify the unjittered projection matrix $\mathbf{P}$ at column 2 (row 0 and row 1):
$$\mathbf{P}'_{0,2} = \mathbf{P}_{0,2} + \frac{2 \cdot j_x}{W_{\text{in}}}$$
$$\mathbf{P}'_{1,2} = \mathbf{P}_{1,2} - \frac{2 \cdot j_y}{H_{\text{in}}}$$

*(Perspective division by $-z_{\text{view}}$ translates NDC by $+\Delta x_{\text{clip}}$, perfectly shifting the image by $+j_x$ texels on the low-resolution raster grid).*

## 3. Metal & C++ Blueprint

### 1. `constexpr` Halton(2, 3) Generator (C++20)
```cpp
#include <simd/simd.h>
#include <array>

constexpr float radicalInverse(uint32_t index, uint32_t base) {
    float result = 0.0f;
    float fraction = 1.0f / static_cast<float>(base);
    while (index > 0) {
        result += static_cast<float>(index % base) * fraction;
        index /= base;
        fraction /= static_cast<float>(base);
    }
    return result;
}

template <size_t N>
inline std::array<simd::float2, N> generateHaltonJitter() noexcept {
    std::array<simd::float2, N> sequence{};
    for (uint32_t i = 0; i < N; ++i) {
        // Base 2 for X, Base 3 for Y, centered at [-0.5, 0.5]
        sequence[i] = simd::make_float2(
            radicalInverse(i + 1, 2) - 0.5f,
            radicalInverse(i + 1, 3) - 0.5f
        );
    }
    return sequence;
}

// 16-phase jitter table (note: simd::make_float2 is non-constexpr in Apple SDK)
inline const auto kHalton16 = generateHaltonJitter<16>();
```

### 2. Jittered Matrix Helper & Scaler Parameterization
```cpp
// Translation approach: Equivalent to shifting NDC clip space by (+2jx/W, -2jy/H)
inline simd::float4x4 applyJitter(
    simd::float4x4 unjitteredProj,
    float jx, float jy,
    uint32_t inW, uint32_t inH
) {
    simd::float4x4 jitterTranslation = matrix_identity_float4x4;
    // 2.0 / dimension converts pixel offsets to NDC [-1, 1] range
    jitterTranslation.columns[3].x =  (2.0f * jx) / static_cast<float>(inW);
    jitterTranslation.columns[3].y = -(2.0f * jy) / static_cast<float>(inH);
    return matrix_multiply(jitterTranslation, unjitteredProj);
}

// In Render Loop:
uint32_t jitterPhase = frameIndex % 16;
simd::float2 currentJitter = kHalton16[jitterPhase];

simd::float4x4 jitteredProj = applyJitter(
    baseProj,
    currentJitter.x, currentJitter.y,
    renderWidth, renderHeight
);

// Inform MetalFX of jitter offset in input pixels
pTemporalScaler->setJitterOffsetX(currentJitter.x);
pTemporalScaler->setJitterOffsetY(currentJitter.y);
```

## 4. Step-by-Step Implementation Checklist
- [ ] Implement C++20 `constexpr` Halton generator function for bases (2, 3).
- [ ] Precompute a 16-sample jitter sequence centered around $[-0.5, 0.5]$.
- [ ] Store both unjittered and jittered projection matrices in uniform buffer.
- [ ] Render scene geometry using the jittered projection matrix.
- [ ] Pass jitter offsets $(j_x, j_y)$ directly to `pTemporalScaler->setJitterOffsetX/Y()`.
- [ ] Verify that disabling upscaling reveals subpixel shimmering (phase shifts), and enabling MetalFX Temporal eliminates shimmering and produces a sharp, stable image.

## 5. Common Pitfalls & Debugging Checklist
- **Jittering Motion Vectors**:
  - *Critical Bug*: Calculating velocity using the jittered projection matrix causes objects to jitter violently in place.
  - *Fix*: Motion vectors must **always** be calculated using unjittered projection matrices.
- **Y-Axis Inversion Bug in Jitter**:
  - *Symptom*: Severe blur or ghosting during camera motion.
  - *Cause*: Screen-space $Y$ is inverted compared to NDC $Y$.
  - *Fix*: Ensure subtraction $\mathbf{P}'_{1,2} = \mathbf{P}_{1,2} - \frac{2 j_y}{H_{\text{in}}}$.
- **Jitter Units Mismatch**:
  - *Error*: Passing normalized UV offsets $[-\frac{0.5}{W}, \frac{0.5}{W}]$ to `setJitterOffsetX`.
  - *Fix*: MetalFX expects jitter offsets in **pixels** in input texture space $[-0.5, 0.5]$.
