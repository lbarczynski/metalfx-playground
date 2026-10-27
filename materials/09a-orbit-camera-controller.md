# TASK-09A: Orbit Camera Controller & Frame Delta Time

## 1. Concept & Mental Model

### Frame Rate Independence ($\Delta t$)
If your simulation advances by a fixed amount per frame without measuring time, the camera will rotate twice as fast on a 120 Hz ProMotion display as on a 60 Hz screen. 

Using `std::chrono::high_resolution_clock`, we calculate the elapsed time between consecutive frames:
$$\Delta t = t_{\text{current}} - t_{\text{previous}}$$
All velocities and rotational rates are multiplied by $\Delta t$ to ensure perfectly consistent physical motion regardless of hardware frame rate.

---

## 2. Mathematical Foundations: Orbit Camera

An **Orbit Camera** rotates around a target focal point $\mathbf{T}$ at distance (radius) $r$ using spherical angles:
- **Yaw ($\phi$)**: Horizontal angle around the Y-axis.
- **Pitch ($\theta$)**: Vertical elevation angle above the XZ plane (clamped to $[-\frac{\pi}{2} + \epsilon, \frac{\pi}{2} - \epsilon]$).

### Eye Position:
$$\mathbf{E}_x = \mathbf{T}_x + r \cdot \cos\theta \cdot \sin\phi$$
$$\mathbf{E}_y = \mathbf{T}_y + r \cdot \sin\theta$$
$$\mathbf{E}_z = \mathbf{T}_z + r \cdot \cos\theta \cdot \cos\phi$$

### Look-At View Matrix:
Given forward vector $\mathbf{F} = \frac{\mathbf{T} - \mathbf{E}}{\|\mathbf{T} - \mathbf{E}\|}$ and world up $\mathbf{U} = (0, 1, 0)$:
$$\mathbf{R} = \frac{\mathbf{F} \times \mathbf{U}}{\|\mathbf{F} \times \mathbf{U}\|}, \quad \mathbf{U}' = \mathbf{R} \times \mathbf{F}$$

$$\mathbf{V}_{\text{lookAt}} = \begin{bmatrix}
R_x & R_y & R_z & -\mathbf{R} \cdot \mathbf{E} \\
U'_x & U'_y & U'_z & -\mathbf{U}' \cdot \mathbf{E} \\
F_x & F_y & F_z & -\mathbf{F} \cdot \mathbf{E} \\
0 & 0 & 0 & 1
\end{bmatrix}$$

---

## 3. Metal & C++ Blueprint

### Camera Controller Class (`src/OrbitCamera.hpp`)
```cpp
#pragma once
#include <simd/simd.h>
#include <cmath>
#include <algorithm>

class OrbitCamera {
public:
    simd::float3 target = { 0.0f, 0.0f, 0.0f };
    float radius = 3.5f;
    float yaw = 0.0f;
    float pitch = 0.3f;

    void update(float dt, float deltaYaw, float deltaPitch, float deltaZoom) {
        yaw += deltaYaw * dt;
        pitch = std::clamp(pitch + deltaPitch * dt, -1.5f, 1.5f);
        radius = std::clamp(radius - deltaZoom, 1.0f, 20.0f);
    }

    simd::float3 eyePosition() const {
        return simd::make_float3(
            target.x + radius * std::cos(pitch) * std::sin(yaw),
            target.y + radius * std::sin(pitch),
            target.z + radius * std::cos(pitch) * std::cos(yaw)
        );
    }

    simd::float4x4 viewMatrix() const {
        simd::float3 eye = eyePosition();
        simd::float3 forward = simd_normalize(target - eye);
        simd::float3 right = simd_normalize(simd_cross(forward, simd::make_float3(0.0f, 1.0f, 0.0f)));
        simd::float3 up = simd_cross(right, forward);

        simd::float4x4 m = matrix_identity_float4x4;
        m.columns[0] = simd::make_float4(right.x, up.x, forward.x, 0.0f);
        m.columns[1] = simd::make_float4(right.y, up.y, forward.y, 0.0f);
        m.columns[2] = simd::make_float4(right.z, up.z, forward.z, 0.0f);
        m.columns[3] = simd::make_float4(-simd_dot(right, eye), -simd_dot(up, eye), -simd_dot(forward, eye), 1.0f);
        return m;
    }
};
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Implement `OrbitCamera` calculating spherical eye coordinates and `viewMatrix()`.
- [ ] In `Renderer`, measure per-frame $\Delta t$ with `std::chrono::high_resolution_clock`.
- [ ] Hook mouse dragging and scroll events from the window bridge to camera yaw, pitch, and zoom.
- [ ] Multiply `projectionMatrix * viewMatrix * modelMatrix` in your per-frame uniforms.
- [ ] Verify: Camera orbits smoothly around the 3D cube with interactive mouse controls.

---

## 5. Definition of Done & Commit Message
- **DoD**: 3D scene can be navigated interactively using an orbit camera driven by mouse drag and scroll wheel with frame-rate-independent delta time.
- **Commit**:
  ```bash
  git add src/OrbitCamera.hpp src/Renderer.cpp src/platform/WindowBridge.mm
  git commit -m "feat(camera): implement interactive orbit camera controller"
  ```
