# TASK-21: Temporal Scaler Tuning & Artifact Mitigation

## 1. Concept & Mental Model

### Real-World MetalFX Tuning
MetalFX Temporal is a closed-source, highly optimized neural/heuristic reconstruction pipeline running directly on Apple Silicon GPU and Apple Neural Engine (ANE) hardware. 

You do not write internal AABB color clipping or YCoCg bounding box clamping kernels yourself—MetalFX performs these internally. Instead, tuning MetalFX involves configuring **hardware scaling factors, active content viewports, camera cut telemetry, and reactive masks**.

---

## 2. Essential MetalFX Tuning Parameters

### 1. Motion Vector Scaling (`setMotionVectorScaleX / Y`)
If your shader outputs motion vectors in normalized NDC coordinates $[-1, 1]$ rather than pixel units:
```cpp
// Inform MetalFX to scale normalized NDC velocities to input pixel units
pTemporalScaler->setMotionVectorScaleX(static_cast<float>(inputWidth));
pTemporalScaler->setMotionVectorScaleY(static_cast<float>(inputHeight));
```
If your shader already outputs pixel units (as done in TASK-18B), the default scale of `1.0` should be preserved.

### 2. Dynamic Content Resolution (`setInputContentWidth / Height`)
Dynamic Resolution Scaling (DRS) dynamically alters internal render resolution based on GPU workload without reallocating physical textures.
You allocate maximum bounds (e.g. $1920 \times 1080$), render into an active subregion (e.g. $1280 \times 720$), and instruct MetalFX:
```cpp
pTemporalScaler->setInputContentWidth(activeWidth);
pTemporalScaler->setInputContentHeight(activeHeight);
```

### 3. Reactive Masks (Preventing Ghosting on Particles & Alpha)
Translucent particles, fire, smoke, and glass do not write reliable motion vectors because they move independently of the geometry behind them. 
A **Reactive Mask** (`MTL::PixelFormatR8Unorm`) marks pixels where temporal history accumulation should be reduced or ignored:
- `0.0`: Full temporal accumulation (standard opaque geometry).
- `1.0`: Zero temporal accumulation (instant refresh, prevents alpha ghosting trails).

---

## 3. Diagnostic Matrix: Common Temporal Defects

| Visual Defect | Likely Root Cause | Verification & Fix |
|---|---|---|
| **Smearing / Ghosting behind moving objects** | Inverted motion vector sign ($\mathbf{p}_{\text{curr}} - \mathbf{p}_{\text{prev}}$) | Verify vector formula points **from current back to previous**: $\mathbf{p}_{\text{prev}} - \mathbf{p}_{\text{curr}}$. |
| **Violent screen-wide jitter** | Camera jitter applied to motion vector calculation | Ensure `previousMVP` and `currentMVP` used for velocities are strictly **unjittered**. |
| **Ghosting on camera cuts / teleports** | History not invalidated | Trigger `pTemporalScaler->setReset(true)` for the frame following a cut. |
| **Fine texture shimmering on resting objects** | Incorrect jitter scaling factor | Verify jitter is in $[-0.5, 0.5]$ range and matches input pixel dimensions. |
| **Black screen or static visual soup** | Missing `StoreActionStore` on depth or motion | Verify both depth and motion render pass attachments specify `MTL::StoreActionStore`. |

---

## 4. Step-by-Step Implementation Checklist
- [ ] Verify motion vector scaling factors match your shader output conventions.
- [ ] Test fast camera motion and verify that geometry edges remain sharp without long trailing artifacts.
- [ ] Add a rotating high-speed object to test disocclusion handling.
- [ ] Verify that triggering camera cuts (`setReset(true)`) instantly clears previous history without ghosting.
- [ ] Fine-tune camera motion thresholds to balance temporal stability against responsive updates.

---

## 5. Definition of Done & Commit Message
- **DoD**: MetalFX Temporal upscaling is tuned for motion stability, with verified motion vector scales and camera cut invalidation preventing ghosting artifacts.
- **Commit**:
  ```bash
  git add src/TemporalUpscaler.hpp src/Renderer.cpp
  git commit -m "feat(metalfx): tune motion vector scales and temporal stability parameters"
  ```
