# TASK-19: Depth Precision & Reversed-Z Configuration

## 1. Concept & Mental Model
Temporal reconstruction relies on subpixel depth accuracy along silhouette boundaries to prevent background colors from bleeding onto foreground objects during camera motion. In classical OpenGL, the near plane maps to $0.0$ and far to $1.0$, which wastes floating-point precision on geometry immediately adjacent to the camera; Reversed-Z maps the near plane to $1.0$ and far plane to $0.0$, exploiting IEEE 754 floating-point exponent distributions to achieve near-uniform precision across the entire view frustum.

## 2. Mathematical Foundations & Algorithms
### IEEE 754 Floating-Point Distribution
A 32-bit float distributes its $2^{24}$ discrete mantissa values exponentially: half of all representable floating-point numbers lie within $[0.0, 1.0]$, and half of *those* lie within $[0.5, 1.0]$.

In Standard Z ($z_{\text{ndc}} \in [0.0, 1.0]$, $z_{\text{near}} \rightarrow 0, z_{\text{far}} \rightarrow 1$):
$$z_{\text{ndc}} = \frac{f}{f - n} - \frac{f n}{(f - n) z_{\text{eye}}}$$
The non-linear $1/z$ curve compresses over $90\%$ of available float precision into the first $5\%$ of the camera distance.

In **Reversed-Z** ($z_{\text{ndc}} \in [1.0, 0.0]$, $z_{\text{near}} \rightarrow 1.0, z_{\text{far}} \rightarrow 0.0$):
$$z_{\text{ndc}} = \frac{n}{n - f} - \frac{f n}{(n - f) z_{\text{eye}}}$$
For an infinite far plane ($f \rightarrow \infty$):
$$z_{\text{ndc}} = \frac{n}{z_{\text{eye}}}$$
The reciprocal nature of perspective projection ($1/z$) directly cancels out the exponential concentration of IEEE 754 floats near $0.0$, yielding nearly constant relative depth precision across the entire frustum.

### Reversed-Z Perspective Matrix Layout (Metal Left-Handed, Eye Looking Down $-Z$)
Given vertical FOV $\theta$, aspect ratio $a = W/H$, near plane $n$, and far plane $f$:
$$f_t = \frac{1}{\tan(\theta / 2)}$$
$$\mathbf{P}_{\text{RevZ}} = \begin{bmatrix}
\frac{f_t}{a} & 0 & 0 & 0 \\
0 & f_t & 0 & 0 \\
0 & 0 & \frac{n}{f - n} & \frac{f \cdot n}{f - n} \\
0 & 0 & -1 & 0
\end{bmatrix}$$

*Derivation check*: When $z_{\text{eye}} = -n \implies z_{\text{ndc}} = 1.0$; when $z_{\text{eye}} = -f \implies z_{\text{ndc}} = 0.0$.

## 3. Metal & C++ Blueprint

### 1. Depth Stencil State with Inverted Comparison
```cpp
MTL::DepthStencilDescriptor* dsDesc = MTL::DepthStencilDescriptor::alloc()->init();
// Inverted from Less to GreaterEqual for Reversed-Z
dsDesc->setDepthCompareFunction(MTL::CompareFunctionGreaterEqual);
dsDesc->setDepthWriteEnabled(true);

MTL::DepthStencilState* pReversedZDepthState = pDevice->newDepthStencilState(dsDesc);
dsDesc->release();
```

### 2. Render Pass Depth Clear Configuration
```cpp
// Clear depth buffer to 0.0 (the furthest possible depth in Reversed-Z)
passDesc->depthAttachment()->setTexture(pLowResDepthTexture);
passDesc->depthAttachment()->setClearDepth(0.0); // NOT 1.0!
passDesc->depthAttachment()->setLoadAction(MTL::LoadActionClear);
passDesc->depthAttachment()->setStoreAction(MTL::StoreActionStore); // Retain for MetalFX
```

### 3. Temporal Scaler Reversed-Z Configuration
```cpp
pTemporalDesc->setDepthTextureFormat(MTL::PixelFormatDepth32Float);

// On compiled scaler instance:
pTemporalScaler->setDepthReversed(true);
```

## 4. Step-by-Step Implementation Checklist
- [ ] Implement `makeReversedZPerspective(fovy, aspect, near, far)`.
- [ ] Update `MTL::DepthStencilDescriptor` compare function to `MTL::CompareFunctionGreaterEqual`.
- [ ] Set `passDesc->depthAttachment()->setClearDepth(0.0)`.
- [ ] Call `pTemporalScaler->setDepthReversed(true)` on the compiled scaler instance.
- [ ] Verify in Xcode GPU capture that foreground geometry has depth values near $1.0$ and background objects approach $0.0$.

## 5. Common Pitfalls & Debugging Checklist
- **All Geometry Disappears (Black Screen)**:
  - *Cause*: Leaving `clearDepth` at $1.0$ while using `CompareFunctionGreaterEqual`. Everything fails the depth test against $1.0$.
  - *Fix*: Clear depth to `0.0`.
- **Z-Fighting on Distant Mountains/Backgrounds**:
  - *Cause*: Using `MTL::PixelFormatDepth16Unorm` with Reversed-Z.
  - *Fix*: Reversed-Z benefits require 32-bit floating-point depth (`MTL::PixelFormatDepth32Float`).
- **Temporal Scaler Blur on Edges**:
  - *Cause*: Mismatch between projection matrix Reversed-Z configuration and `pTemporalScaler->setDepthReversed()`.
  - *Fix*: Ensure both C++ projection matrix and MetalFX scaler instance agree on Reversed-Z.
