# TASK-18: Screen-Space Motion Vectors (Velocity Buffer)

## 1. Concept & Mental Model
Motion vectors track per-pixel 2D displacement from the current frame to the previous frame, allowing the temporal reconstructor to project previous history samples to their corresponding locations in the current frame. In video encoding (HEVC/AV1), motion vectors are approximated via search blocks; in real-time Metal rendering, motion vectors are calculated analytically per-vertex in the vertex shader using both current and previous unjittered model-view-projection matrices and written to an `RG16Float` render target.

## 2. Mathematical Foundations & Algorithms
### Screen-Space Motion Vector Calculation
Let $\mathbf{v}_{\text{model}}$ be a 3D vertex position. Compute current and previous clip space coordinates using **unjittered** matrices:
$$\mathbf{p}_{\text{curr, clip}} = \mathbf{P}_{\text{curr, unjittered}} \cdot \mathbf{V}_{\text{curr}} \cdot \mathbf{M}_{\text{curr}} \cdot \mathbf{v}_{\text{model}}$$
$$\mathbf{p}_{\text{prev, clip}} = \mathbf{P}_{\text{prev, unjittered}} \cdot \mathbf{V}_{\text{prev}} \cdot \mathbf{M}_{\text{prev}} \cdot \mathbf{v}_{\text{model}}$$

Perform perspective division to obtain Normalized Device Coordinates (NDC):
$$\mathbf{p}_{\text{curr, ndc}} = \frac{\mathbf{p}_{\text{curr, clip}}.xy}{\mathbf{p}_{\text{curr, clip}}.w}, \quad \mathbf{p}_{\text{prev, ndc}} = \frac{\mathbf{p}_{\text{prev, clip}}.xy}{\mathbf{p}_{\text{prev, clip}}.w}$$

> [!IMPORTANT]
> **MetalFX Motion Vector Invariant**:
> Per Apple's `MTLFXTemporalScaler` specification, a motion vector points **from the current pixel's location BACK TO where that same content was located in the previous frame's color texture**:
> $$\mathbf{v} = \mathbf{p}_{\text{prev}} - \mathbf{p}_{\text{curr}}$$

Converted to input pixel units:
$$\mathbf{v}_x = \left(\frac{\mathbf{p}_{\text{prev, ndc}}.x - \mathbf{p}_{\text{curr, ndc}}.x}{2.0}\right) \cdot W_{\text{in}}$$
$$\mathbf{v}_y = -\left(\frac{\mathbf{p}_{\text{prev, ndc}}.y - \mathbf{p}_{\text{curr, ndc}}.y}{2.0}\right) \cdot H_{\text{in}}$$

*(The negative sign on $v_y$ accounts for the inversion between NDC $+Y$ up and screen/texture space $+Y$ down).*

## 3. Metal & C++ Blueprint

### 1. C++ Host Struct Layout
```cpp
#include <simd/simd.h>

struct alignas(16) MotionUniforms {
    simd::float4x4 currentJitteredMVP;
    simd::float4x4 currentUnjitteredMVP;
    simd::float4x4 previousUnjitteredMVP;
    simd::float2   renderDimensions; // (W_in, H_in)
    simd::float2   _pad;              // Explicit 16-byte struct padding
};
```

### 2. MRT Pipeline Descriptor Setup
```cpp
// Render Pipeline with Multiple Render Targets (MRT)
pPipeDesc->colorAttachments()->object(0)->setPixelFormat(MTL::PixelFormatRGBA16Float); // RT0: Color
pPipeDesc->colorAttachments()->object(1)->setPixelFormat(MTL::PixelFormatRG16Float);   // RT1: Motion
pPipeDesc->setDepthAttachmentPixelFormat(MTL::PixelFormatDepth32Float);
```

### 3. MSL Shader Writing Color and Motion Vectors
```metal
#include <metal_stdlib>
using namespace metal;

struct VertexInput {
    float3 position [[attribute(0)]];
    float3 normal   [[attribute(1)]];
};

struct MotionUniforms {
    float4x4 currentJitteredMVP;
    float4x4 currentUnjitteredMVP;
    float4x4 previousUnjitteredMVP;
    float2   renderDimensions; // (W_in, H_in)
    float2   _pad;
};

struct VertexToFragment {
    float4 position [[position]]; // Jittered position for rasterization
    float4 currUnjitteredPos;
    float4 prevUnjitteredPos;
};

struct FragmentOutput {
    float4 color        [[color(0)]];
    float2 motionVector [[color(1)]];
};

vertex VertexToFragment motionVertexShader(VertexInput in [[stage_in]],
                                           constant MotionUniforms& u [[buffer(1)]]) {
    VertexToFragment out;
    float4 modelPos = float4(in.position, 1.0);

    // Rasterization must be jittered for temporal anti-aliasing
    out.position = u.currentJitteredMVP * modelPos;

    // Motion calculation must be unjittered
    out.currUnjitteredPos = u.currentUnjitteredMVP * modelPos;
    out.prevUnjitteredPos = u.previousUnjitteredMVP * modelPos;

    return out;
}

fragment FragmentOutput motionFragmentShader(VertexToFragment in [[stage_in]],
                                             constant MotionUniforms& u [[buffer(1)]]) {
    FragmentOutput out;

    // 1. Output base color
    out.color = float4(0.8, 0.4, 0.2, 1.0);

    // 2. Compute NDC positions
    float2 currNDC = in.currUnjitteredPos.xy / in.currUnjitteredPos.w;
    float2 prevNDC = in.prevUnjitteredPos.xy / in.prevUnjitteredPos.w;

    // 3. Compute delta pointing from CURRENT back to PREVIOUS
    float2 deltaNDC = prevNDC - currNDC; // (p_prev - p_curr)
    float2 pixelVelocity;
    pixelVelocity.x =  (deltaNDC.x * 0.5f) * u.renderDimensions.x;
    pixelVelocity.y = -(deltaNDC.y * 0.5f) * u.renderDimensions.y;
    out.motionVector = pixelVelocity;

    return out;
}
```

### 3. Render Pass Attachment Configuration
```cpp
passDesc->colorAttachments()->object(0)->setTexture(pLowResColorTexture);
passDesc->colorAttachments()->object(0)->setLoadAction(MTL::LoadActionClear);
passDesc->colorAttachments()->object(0)->setStoreAction(MTL::StoreActionStore);

passDesc->colorAttachments()->object(1)->setTexture(pLowResMotionTexture);
passDesc->colorAttachments()->object(1)->setLoadAction(MTL::LoadActionClear);
passDesc->colorAttachments()->object(1)->setClearColor(MTL::ClearColor::Make(0.0, 0.0, 0.0, 0.0));
passDesc->colorAttachments()->object(1)->setStoreAction(MTL::StoreActionStore);
```

## 4. Step-by-Step Implementation Checklist
- [ ] Allocate `MTL::Texture` with `MTL::PixelFormatRG16Float` for motion vectors.
- [ ] Update pipeline descriptor to declare two color attachments (RT0: `RGBA16Float`, RT1: `RG16Float`).
- [ ] Store previous frame's unjittered `modelViewProjection` matrix per object.
- [ ] In vertex shader, compute `currUnjitteredPos` and `prevUnjitteredPos`.
- [ ] In fragment shader, perform perspective divide and output pixel-space motion delta to `[[color(1)]]`.
- [ ] At end of frame, copy current unjittered transform to `previousTransform`.
- [ ] Verify stationary objects write `(0.0, 0.0)` velocity, while moving objects show bright green/red direction vectors.

## 5. Common Pitfalls & Debugging Checklist
- **Ghosting Smears on Stationary Objects**:
  - *Cause*: Uninitialized `previousMVP` matrix on frame 0 containing zeros, generating huge velocities across the entire screen.
  - *Fix*: Initialize `previousMVP = currentMVP` on the first frame or trigger `pScaler->setReset(true)`.
- **Y-Axis Velocity Inversion**:
  - *Symptom*: Objects streak and blur during vertical camera movements.
  - *Fix*: Ensure $v_y$ includes a negative sign to account for NDC to texture coordinate flipping.
- **Accidentally Jittering Previous MVP**:
  - *Fix*: Both current and previous matrices must strictly be **unjittered** for velocity calculations.
