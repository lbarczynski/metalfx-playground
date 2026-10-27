---
name: debug-metal
description: >-
  Use this skill when the user encounters a Metal validation layer error, compiler error, shader compilation crash,
  visual glitch (e.g., black screen, flickering, inverted motion blur, z-fighting), or memory leak in the MetalFX playground.
---

# Debug Metal Skill: GPU Diagnostic Assistant

This skill guides the developer in identifying, understanding, and resolving low-level Metal validation crashes, shader mismatches, and rendering bugs.

## Diagnostic Protocol

### 1. Identify the Bug Category

| Category | Typical Symptoms | Initial Investigation Focus |
|---|---|---|
| **Validation Layer Crash** | Console assertion with `[MTLDebugRenderCommandEncoder ...]` | Pixel format mismatch, missing texture usage flags, unbound buffer index, missing `endEncoding()`. |
| **Silent Black Screen** | Application runs, no crash, but window is black | Culling mode vs winding order, depth clear value mismatch (Reversed-Z), storeAction set to `DontCare`. |
| **Visual Distortion / Skew** | Diagonal tearing, upside-down texture, wrong colors | `bytesPerRow` row stride mismatch, top-left vs bottom-left UV convention, Little-Endian ARM64 byte packing. |
| **Jitter / Shimmering / Ghosting** | Smearing trails on moving objects, static jitter | Motion vector direction inverted ($\mathbf{p}_{\text{prev}} - \mathbf{p}_{\text{curr}}$), jitter applied to velocity projection, missing history reset. |
| **Freezes / Deadlocks** | App hangs after a few seconds or when minimized | Semaphore deadlock in triple buffering (`beginFrame` executed outside drawable check). |
| **Unbounded Memory Growth** | RSS increases continuously by megabytes/sec | Missing `NS::AutoreleasePool` in frame callback, reallocating textures every frame. |

### 2. Root Cause Analysis Checklist

#### A. Decoding Metal Validation Errors
Translate the validation error into plain terms:
1. **Quote the exact assertion**: Identify which API parameter triggered it.
2. **Explain the broken invariant**: Why does Metal mandate this rule? (e.g. tile memory hardware constraints, texture cache consistency).
3. **Point to the file & struct**: Direct the user to the exact setup code where the invariant was misconfigured.

#### B. Diagnosing Visual Bugs (The "Big 5" Traps)
1. **Black Screen**:
   - Check if `passDesc->colorAttachments()->object(0)->setStoreAction(MTL::StoreActionStore)` is missing.
   - In Reversed-Z: Verify `clearDepth` is set to `0.0` (not `1.0`) and compare function is `GreaterEqual`.
   - Procedural triangles: Check if `setCullMode(MTL::CullModeNone)` is set.
2. **Z-Fighting**:
   - Near plane $n \le 0.0$ generates `NaN` or division by zero in perspective matrices.
   - Reversed-Z requires 32-bit floating-point depth (`MTL::PixelFormatDepth32Float`).
3. **Motion Vector Streaks**:
   - MetalFX strictly requires motion vectors to point **from current BACK TO previous**: $\mathbf{p}_{\text{prev}} - \mathbf{p}_{\text{curr}}$.
   - Ensure the velocity projection matrix is strictly **unjittered**.
4. **Distorted Colors & Struct Shifts**:
   - Check if scalar members broke 16-byte alignment in MSL structs (e.g. `simd::float3` has a 16-byte stride).
5. **Memory Leak**:
   - Ensure `NS::AutoreleasePool* pool = NS::AutoreleasePool::alloc()->init();` and `pool->release();` wrap the entire per-frame logic.

### 3. Response Structure (Mentor Mode)
Deliver the diagnosis without writing the solution code for the user:

```markdown
### 🛠️ Metal Diagnostic: [Bug Summary]

#### 🔍 Root Cause Explanation
[Explain *why* Metal or the GPU behaved this way. Break down the invariant.]

#### 📍 Suspect Location
- **File / Function**: `[path/to/file.cpp:LineNumber]`
- **Involved Struct / State**: `[Name of descriptor, state, or shader]`

#### 💡 Guiding Question / Clue
[Ask a question or offer an isolated syntax clue that leads the user to fix the bug themselves]
```
