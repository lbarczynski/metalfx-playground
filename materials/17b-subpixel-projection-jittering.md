# TASK-17B: Subpixel Camera Projection Jittering

## 1. Concept & Mental Model

### Applying Subpixel Jitter to Perspective Matrices
In clip space, the viewport spans $[-1.0, 1.0]$ in both $X$ and $Y$ (total span = $2.0$).
Given input render dimensions $W_{\text{in}} \times H_{\text{in}}$ and a subpixel offset $(j_x, j_y)$ in pixel units:

$$\Delta x_{\text{ndc}} = \frac{2.0 \cdot j_x}{W_{\text{in}}}, \quad \Delta y_{\text{ndc}} = \frac{2.0 \cdot j_y}{H_{\text{in}}}$$

To shift the entire 3D scene by this subpixel offset prior to rasterization, we adjust elements $P[2][0]$ and $P[2][1]$ in our column-major perspective projection matrix:

$$\mathbf{P}_{\text{jittered}}[2][0] = \mathbf{P}_{\text{unjittered}}[2][0] + \Delta x_{\text{ndc}}$$
$$\mathbf{P}_{\text{jittered}}[2][1] = \mathbf{P}_{\text{unjittered}}[2][1] + \Delta y_{\text{ndc}}$$

*(This works because $x_{\text{clip}} = P[0][0] \cdot x_{\text{eye}} + P[2][0] \cdot z_{\text{eye}}$. Dividing by $w_{\text{clip}} = z_{\text{eye}}$ during perspective divide directly adds $P[2][0]$ to $x_{\text{ndc}}$!)*

---

## 2. Invariants: Jittered vs Unjittered Matrices

> [!CAUTION]
> **The Jitter Invariant**:
> - **Scene Rasterization (Color & Depth)**: MUST use the **Jittered** projection matrix so the rasterizer samples unique subpixel phases every frame.
> - **Motion Vector Calculation (Velocity Buffer)**: MUST strictly use the **Unjittered** projection matrix! If you accidentally compute motion vectors using jittered matrices, the subpixel camera shifts will leak into the velocity buffer, causing violent full-screen jitter and blurring.

---

## 3. Metal & C++ Blueprint

### Jittering the Camera Projection (`src/Renderer.cpp`)
```cpp
// 1. Retrieve current frame subpixel offset
simd::float2 jitter = HaltonJitter::sample(_frameNumber);

// 2. Compute NDC offsets based on low-res input dimensions
float deltaX = (2.0f * jitter.x) / static_cast<float>(kLowResWidth);
float deltaY = (2.0f * jitter.y) / static_cast<float>(kLowResHeight);

// 3. Create unjittered and jittered projection matrices
simd::float4x4 unjitteredProj = math::makePerspective(1.047f, aspect, 0.1f, 100.0f);
simd::float4x4 jitteredProj = unjitteredProj;

// Apply subpixel offset to column 2 (column-major)
jitteredProj.columns[2].x += deltaX;
jitteredProj.columns[2].y += deltaY;

// 4. Update uniforms for scene rasterization
_frameUniforms.viewProjection = simd_mul(jitteredProj, _camera.viewMatrix());
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Compute subpixel NDC offsets $\Delta x$ and $\Delta y$ per frame.
- [ ] Add the offsets to columns `[2].x` and `[2].y` of the perspective projection matrix.
- [ ] Pass the jittered projection to the scene pass.
- [ ] Render the scene at 540p without upscaling: Observe subtle, rhythmic pixel shimmering across geometric edges as the phase shifts across consecutive frames.

---

## 5. Definition of Done & Commit Message
- **DoD**: Camera projection matrix jitters subpixel offsets every frame based on Halton(2,3) sequence, creating visible geometric subpixel phase shifts in the low-res render target.
- **Commit**:
  ```bash
  git add src/Renderer.cpp src/SimdMath.hpp
  git commit -m "feat(renderer): apply subpixel jitter to camera projection matrix"
  ```
