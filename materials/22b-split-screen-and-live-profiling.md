# TASK-22B: Split-Screen Comparison Playground & GPU Metrics

## 1. Concept & Mental Model

### The Capstone: Interactive Split-Screen Playground
The ultimate evaluation tool for real-time graphics engineers is a **Split-Screen Wipe View**:
- An interactive vertical divider splits the window.
- The left half displays the baseline image (e.g. Native Low-Res or Bilinear).
- The right half displays the reconstructed image (MetalFX Spatial or MetalFX Temporal).
- Moving the mouse drags the divider across the screen, allowing pixel-by-pixel inspection of geometric edges, texture clarity, and temporal stability.

---

## 2. Metal Implementation: Scissor Rectangles

In Metal, restricting rendering or blitting to a specific screen subregion is achieved with **Scissor Rectangles** (`MTL::ScissorRect`):

```
(0, 0) ─────────────────── [ Divider X ] ────────────────── (Width, 0)
  │                              │                              │
  │      Left Viewport           │       Right Viewport         │
  │  (Mode A: Bilinear 540p)     │  (Mode B: MetalFX Temporal)  │
  │                              │                              │
(0, Height) ─────────────────────┴───────────────────────── (Width, Height)
```

```cpp
// Restrict rendering to left half
MTL::ScissorRect leftScissor = { 0, 0, dividerX, windowHeight };
enc->setScissorRect(leftScissor);

// Restrict rendering to right half
MTL::ScissorRect rightScissor = { dividerX, 0, windowWidth - dividerX, windowHeight };
enc->setScissorRect(rightScissor);
```

---

## 3. Performance Metrics: Measuring Upscaling Speedup

Using the GPU profiler implemented in TASK-10, we measure the hardware execution duration for each pipeline:

$$\text{Speedup} = \frac{\Delta t_{\text{gpu, Native 1080p}}}{\Delta t_{\text{gpu, MetalFX Temporal 540p}}}$$

### Typical Measurements on Apple Silicon:
- **Native 1080p Shading Pass**: $\sim 3.8 \text{ ms}$
- **540p Shading Pass**: $\sim 1.0 \text{ ms}$
- **MetalFX Temporal Pass**: $\sim 0.6 \text{ ms}$
- **Total MetalFX Frame**: $\sim 1.6 \text{ ms}$
- **Effective GPU Speedup**: **$2.37\times$ faster!**

---

## 4. Final Codebase Polish Checklist
- [ ] Implement interactive split-screen divider draggable with the mouse.
- [ ] Connect `GpuProfiler` to measure GPU execution time for each mode in microseconds.
- [ ] Display live FPS, GPU frame time (ms), and active scaling preset in the window title bar or console.
- [ ] Verify that all code compiles with zero compiler warnings under:
  `-std=c++20 -Wall -Wextra -Wpedantic -Wconversion`.
- [ ] Verify memory stability: Run the playground for 5 minutes under ASan (`macos-debug`) and verify zero memory leaks.

---

## 5. Definition of Done & Commit Message
- **DoD**: A polished interactive split-screen playground allowing live side-by-side visual and performance comparisons between all 5 upscaling modes with live GPU timing metrics.
- **Commit**:
  ```bash
  git add src/Renderer.cpp src/platform/WindowBridge.mm src/main.cpp
  git commit -m "feat(capstone): complete split-screen comparison playground with gpu metrics"
  ```
