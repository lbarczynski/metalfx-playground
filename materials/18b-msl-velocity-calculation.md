# TASK-18B: Analytical Motion Vector Calculation in MSL

## 1. Concept & Mental Model

### The Vector Direction Invariant
In video encoding or optical flow, motion vectors often point forward from past to present.
In Apple MetalFX, the temporal reconstructor looks up historical samples from the previous frame to blend into the current frame. Therefore:

> [!IMPORTANT]
> **MetalFX Motion Vector Direction Rule**:
> Motion vectors strictly point **from the CURRENT pixel BACK TO where that content was located in the PREVIOUS frame**:
> $$\mathbf{v} = \mathbf{p}_{\text{prev}} - \mathbf{p}_{\text{curr}}$$
> If your vectors point in the opposite direction ($\mathbf{p}_{\text{curr}} - \mathbf{p}_{\text{prev}}$), moving geometry will display inverted smear trails and severe ghosting artifacts!

---

## 2. Mathematical Foundations & Algorithms

For each vertex $\mathbf{v}_{\text{model}}$, compute current and previous positions using **unjittered** matrices:

$$\mathbf{p}_{\text{curr, clip}} = \mathbf{P}_{\text{curr, unjittered}} \cdot \mathbf{V}_{\text{curr}} \cdot \mathbf{M}_{\text{curr}} \cdot \mathbf{v}_{\text{model}}$$
$$\mathbf{p}_{\text{prev, clip}} = \mathbf{P}_{\text{prev, unjittered}} \cdot \mathbf{V}_{\text{prev}} \cdot \mathbf{M}_{\text{prev}} \cdot \mathbf{v}_{\text{model}}$$

Perform perspective divide to obtain Normalized Device Coordinates (NDC):
$$\mathbf{p}_{\text{curr, ndc}} = \frac{\mathbf{p}_{\text{curr, clip}}.xy}{\mathbf{p}_{\text{curr, clip}}.w}, \quad \mathbf{p}_{\text{prev, ndc}} = \frac{\mathbf{p}_{\text{prev, clip}}.xy}{\mathbf{p}_{\text{prev, clip}}.w}$$

Compute delta pointing from current to previous:
$$\Delta \mathbf{p}_{\text{ndc}} = \mathbf{p}_{\text{prev, ndc}} - \mathbf{p}_{\text{curr, ndc}}$$

Convert to input pixel units ($W_{\text{in}} \times H_{\text{in}}$):
$$\mathbf{v}_x = \left(\frac{\Delta \mathbf{p}_{\text{ndc}}.x}{2.0}\right) \cdot W_{\text{in}}$$
$$\mathbf{v}_y = -\left(\frac{\Delta \mathbf{p}_{\text{ndc}}.y}{2.0}\right) \cdot H_{\text{in}} \quad \text{(Negated because Metal UV Y runs downwards)}$$

---

## 3. Metal & C++ Blueprint

### 1. Uniforms Definition (`src/SimdMath.hpp`)
```cpp
struct alignas(16) TemporalUniforms {
    simd::float4x4 currentJitteredMVP;
    simd::float4x4 currentUnjitteredMVP;
    simd::float4x4 previousUnjitteredMVP;
    float          inputWidth;
    float          inputHeight;
    float          _pad[2];
};
```

### 2. MSL Vertex and Fragment Shaders
```metal
#include <metal_stdlib>
using namespace metal;

struct TemporalUniforms {
    float4x4 currentJitteredMVP;
    float4x4 currentUnjitteredMVP;
    float4x4 previousUnjitteredMVP;
    float    inputWidth;
    float    inputHeight;
};

struct VertexInput {
    float3 position [[attribute(0)]];
    float3 normal   [[attribute(1)]];
    float2 uv       [[attribute(2)]];
};

struct VertexOutput {
    float4 position [[position]];       // Jittered for rasterization
    float4 currUnjitteredPos;           // Unjittered current
    float4 prevUnjitteredPos;           // Unjittered previous
    float2 uv;
};

struct FragmentOutput {
    float4 color  [[color(0)]];         // Color RT0
    float2 motion [[color(1)]];         // Motion Vectors RT1
};

vertex VertexOutput velocityVertexMain(
    uint vertexID [[vertex_id]],
    constant VertexInput* vertices [[buffer(0)]],
    constant TemporalUniforms& u   [[buffer(1)]]
) {
    VertexOutput out;
    float4 modelPos = float4(vertices[vertexID].position, 1.0);

    // 1. Rasterization position MUST be jittered
    out.position = u.currentJitteredMVP * modelPos;

    // 2. Velocity positions MUST be strictly unjittered
    out.currUnjitteredPos = u.currentUnjitteredMVP * modelPos;
    out.prevUnjitteredPos = u.previousUnjitteredMVP * modelPos;
    out.uv = vertices[vertexID].uv;
    return out;
}

fragment FragmentOutput velocityFragmentMain(
    VertexOutput in [[stage_in]],
    texture2d<float> colorTexture [[texture(0)]],
    sampler colorSampler          [[sampler(0)]],
    constant TemporalUniforms& u  [[buffer(0)]]
) {
    FragmentOutput out;
    out.color = colorTexture.sample(colorSampler, in.uv);

    // Perspective divide to obtain NDC coordinates
    float2 currNDC = in.currUnjitteredPos.xy / in.currUnjitteredPos.w;
    float2 prevNDC = in.prevUnjitteredPos.xy / in.prevUnjitteredPos.w;

    // Motion vector pointing from CURRENT back to PREVIOUS
    float2 deltaNDC = prevNDC - currNDC;

    // Convert to pixel space
    float2 pixelVelocity;
    pixelVelocity.x = (deltaNDC.x * 0.5f) * u.inputWidth;
    pixelVelocity.y = -(deltaNDC.y * 0.5f) * u.inputHeight;

    out.motion = pixelVelocity;
    return out;
}
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Store `previousUnjitteredMVP` in C++ and copy current to previous at the end of every frame.
- [ ] Initialize `previousUnjitteredMVP = currentUnjitteredMVP` on frame 0 to avoid huge velocity spikes on startup.
- [ ] In MSL vertex shader, calculate both `currUnjitteredPos` and `prevUnjitteredPos`.
- [ ] In MSL fragment shader, compute $(\mathbf{p}_{\text{prev}} - \mathbf{p}_{\text{curr}})$ and write to `color(1)`.
- [ ] Capture frame in Xcode: Inspect the `RG16Float` motion texture. Moving objects should show bright color tints (red/green) indicating vector direction, while stationary backgrounds remain pure black $(0, 0)$.

---

## 5. Definition of Done & Commit Message
- **DoD**: Analytical screen-space motion vectors are calculated in MSL pointing strictly from current to previous frame and outputted to the `RG16Float` velocity buffer.
- **Commit**:
  ```bash
  git add src/Renderer.cpp src/SimdMath.hpp src/Shaders.metal
  git commit -m "feat(shader): implement analytical motion vector calculation in msl"
  ```
