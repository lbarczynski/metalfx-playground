# TASK-19: Reversed-Z Depth Precision for Temporal

## 1. Concept & Mental Model

### The Floating-Point Depth Problem
In IEEE 754 floating-point numbers, roughly half of all representable values exist between $[-1.0, 1.0]$, with density clustering dramatically near zero.

In standard perspective projection, depth is non-linear:
$$z_{\text{ndc}} = \frac{f}{f - n} - \frac{n \cdot f}{(f - n) \cdot z_{\text{eye}}}$$

When mapping near $n \to 0.0$ and far $f \to 1.0$:
- Over **$90\%$** of all float precision is wasted in the first $10\%$ of distance in front of the camera lens!
- Objects further away share identical depth bits, causing severe **Z-fighting** (flickering surfaces).
- Temporal upscalers rely on precise depth differences to detect geometry disocclusions. Standard Z causes false disocclusion triggers and severe ghosting.

---

## 2. Mathematical Foundations: The Reversed-Z Solution

By mapping near plane $n \to 1.0$ and far plane $f \to 0.0$, the logarithmic distribution of IEEE 754 floating-point exponents perfectly cancels out the non-linear $\frac{1}{z}$ perspective divide!

### Reversed-Z Perspective Matrix ($n \to 1, f \to 0$):
$$\mathbf{P}_{\text{revZ}} = \begin{bmatrix}
\frac{S}{a} & 0 & 0 & 0 \\
0 & S & 0 & 0 \\
0 & 0 & \frac{n}{n - f} & \frac{-f \cdot n}{n - f} \\
0 & 0 & 1 & 0
\end{bmatrix}$$
*(With an infinite far plane $f \to \infty$, this simplifies to $\mathbf{P}[2][2] = 0, \mathbf{P}[3][2] = n$.)*

---

## 3. Metal Configuration Rules

To switch a Metal pipeline to Reversed-Z:

| Property | Standard Z | Reversed-Z (Mandated by MetalFX) |
|---|---|---|
| **Depth Clear Value** | `1.0` | **`0.0`** (`passDesc->setClearDepth(0.0)`) |
| **Compare Function** | `CompareFunctionLess` | **`CompareFunctionGreaterEqual`** |
| **Depth Format** | `Depth16Unorm` / `Depth32Float` | Strictly **`MTL::PixelFormatDepth32Float`** |
| **MetalFX Flag** | `setDepthReversed(false)` | **`pScaler->setDepthReversed(true)`** |

---

## 4. Metal & C++ Blueprint

### 1. Reversed-Z Matrix Helper (`src/SimdMath.hpp`)
```cpp
inline simd::float4x4 makeReversedZPerspective(float fovRadians, float aspect, float nearZ, float farZ) {
    float s = 1.0f / std::tan(fovRadians * 0.5f);
    simd::float4x4 m = { 0 };
    m.columns[0] = simd::make_float4(s / aspect, 0.0f, 0.0f, 0.0f);
    m.columns[1] = simd::make_float4(0.0f, s, 0.0f, 0.0f);
    m.columns[2] = simd::make_float4(0.0f, 0.0f, nearZ / (nearZ - farZ), 1.0f);
    m.columns[3] = simd::make_float4(0.0f, 0.0f, (-farZ * nearZ) / (nearZ - farZ), 0.0f);
    return m;
}
```

### 2. Depth State & Pass Setup
```cpp
// 1. Configure Depth Stencil State with GreaterEqual
MTL::DepthStencilDescriptor* dsDesc = MTL::DepthStencilDescriptor::alloc()->init();
dsDesc->setDepthCompareFunction(MTL::CompareFunctionGreaterEqual);
dsDesc->setDepthWriteEnabled(true);
MTL::DepthStencilState* pReversedZDepthState = pDevice->newDepthStencilState(dsDesc);
dsDesc->release();

// 2. In renderFrame():
passDesc->depthAttachment()->setTexture(pLowResDepthTexture);
passDesc->depthAttachment()->setClearDepth(0.0); // 0.0 = Far plane in Reversed-Z
passDesc->depthAttachment()->setLoadAction(MTL::LoadActionClear);
passDesc->depthAttachment()->setStoreAction(MTL::StoreActionStore); // Retain for MetalFX

// 3. Configure Scaler
pTemporalScaler->setDepthReversed(true);
```

---

## 5. Step-by-Step Implementation Checklist
- [ ] Implement `makeReversedZPerspective()` in `src/SimdMath.hpp`.
- [ ] Configure `MTL::DepthStencilState` with `CompareFunctionGreaterEqual`.
- [ ] Set `clearDepth = 0.0` on render pass depth attachment.
- [ ] Set `setStoreAction(MTL::StoreActionStore)` on depth attachment so MetalFX can read it.
- [ ] Call `pTemporalScaler->setDepthReversed(true)`.
- [ ] Verify: 3D scene renders with flawless depth sorting, and near/far planes are properly preserved.

---

## 6. Definition of Done & Commit Message
- **DoD**: Reversed-Z depth buffer (`Depth32Float`) is configured with `GreaterEqual` compare, `clearDepth = 0.0`, and bound to the temporal pipeline with `setDepthReversed(true)`.
- **Commit**:
  ```bash
  git add src/SimdMath.hpp src/Renderer.cpp src/TemporalUpscaler.hpp
  git commit -m "feat(depth): configure reversed-z depth buffer for temporal reconstruction"
  ```
