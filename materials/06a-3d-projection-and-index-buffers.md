# TASK-06A: 3D Perspective Projection & Index Buffers

## 1. Concept & Mental Model

### The Coordinate Space Pipeline
In 3D computer graphics, vertices undergo a series of matrix transformations before reaching your display:

```
[ Model Space ] ── (Model Matrix M) ──> [ World Space ]
                                               │
                                       (View Matrix V)
                                               │
                                               ▼
[ Clip Space ] <── (Projection Matrix P) ── [ View / Eye Space ]
      │
(Perspective Divide: /w)
      ▼
[ Normalized Device Coordinates (NDC) ]
      │ (x, y in [-1, 1], z in [0, 1])
      ▼
[ Screen / Viewport Pixels ]
```

### Why Index Buffers?
A 3D cube has **8 unique corners (vertices)**, but 6 square faces made of 12 triangles (requiring 36 vertex lookups). 
- Without indices: You must allocate 36 duplicated vertices in memory ($36 \times 32 \text{ bytes} = 1152 \text{ bytes}$).
- With indices: You allocate only 8 vertices and an index buffer of 36 `uint16_t` values ($36 \times 2 \text{ bytes} = 72 \text{ bytes}$).
The GPU's post-transform vertex cache reuses transformed vertices across shared triangle edges, dramatically boosting performance.

---

## 2. Mathematical Foundations: Perspective Matrix

In Apple Metal, depth in clip space spans $[0.0, 1.0]$ (unlike OpenGL's $[-1.0, 1.0]$).
For field of view $\theta$, aspect ratio $a = \frac{W}{H}$, near plane $n$, and far plane $f$:

$$S = \frac{1}{\tan(\theta / 2)}$$
$$\mathbf{P}_{\text{persp}} = \begin{bmatrix}
\frac{S}{a} & 0 & 0 & 0 \\
0 & S & 0 & 0 \\
0 & 0 & \frac{f}{f - n} & \frac{-n \cdot f}{f - n} \\
0 & 0 & 1 & 0
\end{bmatrix}$$

---

## 3. Metal & C++ Blueprint

### 1. Matrix Helpers & Cube Geometry (`src/SimdMath.hpp`)
```cpp
inline simd::float4x4 makePerspective(float fovRadians, float aspect, float nearZ, float farZ) {
    float s = 1.0f / std::tan(fovRadians * 0.5f);
    simd::float4x4 m = { 0 };
    m.columns[0] = simd::make_float4(s / aspect, 0.0f, 0.0f, 0.0f);
    m.columns[1] = simd::make_float4(0.0f, s, 0.0f, 0.0f);
    m.columns[2] = simd::make_float4(0.0f, 0.0f, farZ / (farZ - nearZ), 1.0f);
    m.columns[3] = simd::make_float4(0.0f, 0.0f, (-nearZ * farZ) / (farZ - nearZ), 0.0f);
    return m;
}

// 8 Corner Vertices for a Unit Cube
struct Vertex3D {
    simd::float3 position;
    simd::float3 color;
};

inline const Vertex3D kCubeVertices[8] = {
    { { -0.5f, -0.5f, -0.5f }, { 1.0f, 0.0f, 0.0f } },
    { {  0.5f, -0.5f, -0.5f }, { 0.0f, 1.0f, 0.0f } },
    { {  0.5f,  0.5f, -0.5f }, { 0.0f, 0.0f, 1.0f } },
    { { -0.5f,  0.5f, -0.5f }, { 1.0f, 1.0f, 0.0f } },
    { { -0.5f, -0.5f,  0.5f }, { 1.0f, 0.0f, 1.0f } },
    { {  0.5f, -0.5f,  0.5f }, { 0.0f, 1.0f, 1.0f } },
    { {  0.5f,  0.5f,  0.5f }, { 1.0f, 1.0f, 1.0f } },
    { { -0.5f,  0.5f,  0.5f }, { 0.2f, 0.2f, 0.2f } }
};

// 36 Indices (12 Triangles, Counter-Clockwise Winding)
inline const uint16_t kCubeIndices[36] = {
    0, 2, 1, 0, 3, 2, // Front
    1, 6, 5, 1, 2, 6, // Right
    5, 7, 4, 5, 6, 7, // Back
    4, 3, 0, 4, 7, 3, // Left
    3, 6, 2, 3, 7, 6, // Top
    4, 1, 5, 4, 0, 1  // Bottom
};
```

### 2. Catch2 Verification (`tests/test_math.cpp`)
```cpp
TEST_CASE("Perspective Projection: Clip Coordinates at Near and Far", "[math]") {
    float nearZ = 0.1f;
    float farZ = 100.0f;
    auto proj = math::makePerspective(1.047f, 1.777f, nearZ, farZ);

    // Near point: z = nearZ -> NDC z should be 0.0
    simd::float4 pNear = simd::make_float4(0.0f, 0.0f, nearZ, 1.0f);
    simd::float4 clipNear = simd_mul(proj, pNear);
    float ndcNearZ = clipNear.z / clipNear.w;
    REQUIRE_THAT(ndcNearZ, Catch::Matchers::WithinAbs(0.0f, 1e-4f));

    // Far point: z = farZ -> NDC z should be 1.0
    simd::float4 pFar = simd::make_float4(0.0f, 0.0f, farZ, 1.0f);
    simd::float4 clipFar = simd_mul(proj, pFar);
    float ndcFarZ = clipFar.z / clipFar.w;
    REQUIRE_THAT(ndcFarZ, Catch::Matchers::WithinAbs(1.0f, 1e-4f));
}
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Implement `makePerspective()` in `src/SimdMath.hpp`.
- [ ] Define the 8 corner vertices and 36 `uint16_t` indices for the cube.
- [ ] Create Catch2 tests asserting that near plane maps to $0.0$ and far plane maps to $1.0$.
- [ ] Run CTest and verify all assertions pass: `ctest --preset macos-debug-test`.

---

## 5. Definition of Done & Commit Message
- **DoD**: Perspective projection matrix and 36 cube indices are mathematically verified with passing unit tests under Catch2.
- **Commit**:
  ```bash
  git add src/SimdMath.hpp tests/test_math.cpp
  git commit -m "feat(math): implement perspective projection and indexed cube geometry"
  ```
