# TASK-21: Temporal Artifact Mitigation & Tuning

## 1. Concept & Mental Model
While temporal upscaling achieves remarkable subpixel sharpness, imperfect velocity buffers and disocclusions can introduce temporal artifacts such as ghosting trails, edge shimmering, and thin line flickering. Mitigation techniques stabilize the reconstruction pipeline through neighborhood color clamping (AABB clipping in YCoCg space), velocity dilation along silhouette edges, and motion vector scaling.

## 2. Mathematical Foundations & Algorithms
### Neighborhood Color Clamping (AABB Clipping in YCoCg Space)
To eliminate ghosting when an object moves and reveals previously hidden background (disocclusion), the reprojected historical sample $\mathbf{C}_{\text{hist}}$ must be clamped within the color bounding box of the current frame's $3 \times 3$ neighborhood.

Transform RGB into decorrelated YCoCg space:
$$\begin{bmatrix} Y \\ C_o \\ C_g \end{bmatrix} = \begin{bmatrix} 0.25 & 0.50 & 0.25 \\ 0.50 & 0.00 & -0.50 \\ -0.25 & 0.50 & -0.25 \end{bmatrix} \begin{bmatrix} R \\ G \\ B \end{bmatrix}$$

Compute neighborhood mean $\mu$ and standard deviation $\sigma$:
$$\mu = \frac{1}{9} \sum_{i \in 3 \times 3} \mathbf{c}_i, \quad \sigma = \sqrt{\frac{1}{9} \sum_{i \in 3 \times 3} (\mathbf{c}_i - \mu)^2}$$
$$\mathbf{c}_{\text{min}} = \mu - \gamma \sigma, \quad \mathbf{c}_{\text{max}} = \mu + \gamma \sigma \quad (\text{typically } \gamma \in [1.0, 1.5])$$
$$\mathbf{C}_{\text{clamped}} = \text{clamp}(\mathbf{C}_{\text{hist}}, \mathbf{c}_{\text{min}}, \mathbf{c}_{\text{max}})$$

### Velocity Dilation (Nearest Depth Search)
Thin geometry (such as ropes, antennas, and wireframes) suffers from velocity bleeding when motion vectors are bilinearly sampled across background edges. Velocity dilation searches the $3 \times 3$ neighborhood and adopts the velocity of the closest depth sample (in Reversed-Z, maximum depth):
$$\mathbf{v}_{\text{dilated}}(\mathbf{p}) = \mathbf{v}\left( \arg\max_{\mathbf{q} \in \mathcal{N}_{3 \times 3}(\mathbf{p})} Z(\mathbf{q}) \right)$$

## 3. Metal & C++ Blueprint

### 1. Velocity Dilation Pre-Pass Kernel (MSL Compute)
```metal
#include <metal_stdlib>
using namespace metal;

kernel void dilateMotionAndDepth(
    texture2d<float, access::read> inDepth    [[texture(0)]],
    texture2d<float, access::read> inMotion   [[texture(1)]],
    texture2d<float, access::write> outMotion [[texture(2)]],
    uint2 gid [[thread_position_in_grid]]
) {
    if (gid.x >= inDepth.get_width() || gid.y >= inDepth.get_height()) return;

    float maxDepth = -1.0f; // In Reversed-Z, closest depth is maximum
    float2 bestMotion = float2(0.0f);

    // 3x3 neighborhood search for closest surface
    for (int dy = -1; dy <= 1; ++dy) {
        for (int dx = -1; dx <= 1; ++dx) {
            uint2 coord = uint2(clamp(int2(gid) + int2(dx, dy), int2(0), int2(inDepth.get_width() - 1, inDepth.get_height() - 1)));
            float d = inDepth.read(coord).r;
            if (d > maxDepth) {
                maxDepth = d;
                bestMotion = inMotion.read(coord).rg;
            }
        }
    }

    outMotion.write(float4(bestMotion, 0.0, 1.0), gid);
}
```

### 2. Motion Vector Scaling & Tuning Parameters
```cpp
// If motion vectors are in normalized NDC [-1, 1] rather than pixel space,
// configure the scale factors on the compiled scaler instance:
pTemporalScaler->setMotionVectorScaleX(static_cast<float>(inputWidth));
pTemporalScaler->setMotionVectorScaleY(static_cast<float>(inputHeight));
```

## 4. Step-by-Step Implementation Checklist
- [ ] Inspect motion vector textures visually in Xcode GPU capture to verify edge alignment.
- [ ] Insert a lightweight velocity dilation compute dispatch before the temporal scaler pass.
- [ ] Test scene with fast-moving thin geometry (e.g. rotating wireframe cube).
- [ ] Confirm that thin lines remain continuous during movement without breaking or ghosting.
- [ ] Tune $\gamma$ bounding box clamping parameter to balance ghosting suppression against edge stability.

## 5. Common Pitfalls & Debugging Checklist
- **Ghosting Behind Fast Objects**:
  - *Cause*: Motion vector magnitudes are too small or inverted.
  - *Fix*: Multiply motion vectors by correct viewport dimension scale factors.
- **Flickering Thin Lines During Motion**:
  - *Cause*: Background velocity bleeding onto foreground geometry edges.
  - *Fix*: Apply velocity dilation using Reversed-Z maximum depth test.
