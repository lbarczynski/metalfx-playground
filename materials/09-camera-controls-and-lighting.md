# TASK-09: Interactive Camera & Simple Lighting

## 1. Concept & Mental Model
Dynamic 3D illumination computes light reflectance across surface geometry based on surface normals, light direction, and view orientation, while interactive camera controllers transform spherical/polar inputs into view matrices. In OpenGL/GLSL, Blinn-Phong shading is evaluated per-fragment using uniform structs; in Metal, normal vectors are transformed into world space using an inverse-transpose normal matrix $\mathbf{N} = (\mathbf{M}^{-1})^T$ in unified constant buffers, and specular highlights evaluate the halfway vector $\mathbf{H}$.

## 2. Mathematical Foundations & Algorithms
### Blinn-Phong Illumination Model
$$\mathbf{I}_{\text{total}} = \mathbf{I}_{\text{ambient}} + \mathbf{I}_{\text{diffuse}} + \mathbf{I}_{\text{specular}}$$

1. **Ambient Term**:
   $$\mathbf{I}_{\text{ambient}} = k_a \cdot \mathbf{C}_{\text{light}}$$
2. **Diffuse (Lambertian) Term**:
   $$\mathbf{I}_{\text{diffuse}} = k_d \cdot \max(\mathbf{N} \cdot \mathbf{L}, 0.0) \cdot \mathbf{C}_{\text{light}}$$
3. **Specular (Blinn Halfway) Term**:
   $$\mathbf{H} = \frac{\mathbf{L} + \mathbf{V}}{\|\mathbf{L} + \mathbf{V}\|}$$
   $$\mathbf{I}_{\text{specular}} = k_s \cdot \left(\max(\mathbf{N} \cdot \mathbf{H}, 0.0)\right)^\alpha \cdot \mathbf{C}_{\text{light}}$$

where:
- $\mathbf{N}$ = normalized surface normal.
- $\mathbf{L} = \frac{\mathbf{P}_{\text{light}} - \mathbf{P}_{\text{frag}}}{\|\mathbf{P}_{\text{light}} - \mathbf{P}_{\text{frag}}\|}$ (direction to light).
- $\mathbf{V} = \frac{\mathbf{P}_{\text{cam}} - \mathbf{P}_{\text{frag}}}{\|\mathbf{P}_{\text{cam}} - \mathbf{P}_{\text{frag}}\|}$ (direction to eye/camera).
- $\mathbf{H}$ = halfway vector (cheaper and smoother than Phong's reflection vector $\mathbf{R}$).
- $\alpha$ = specular shininess exponent (e.g. 32.0).

### Normal Matrix Transformation
To preserve perpendicularity under non-uniform scale $\mathbf{S}$:
$$\mathbf{N}_{\text{world}} = \text{normalize}\left( \left(\mathbf{M}_{3 \times 3}^{-1}\right)^T \cdot \mathbf{N}_{\text{model}} \right)$$

### Orbit Camera Spherical Coordinates
Given distance $r$, azimuth/yaw $\phi$, and elevation/pitch $\theta$:
$$\mathbf{E}_x = r \cos\theta \sin\phi, \quad \mathbf{E}_y = r \sin\theta, \quad \mathbf{E}_z = r \cos\theta \cos\phi$$

## 3. Metal & C++ Blueprint

### 1. Data Structures & Memory Alignment
```cpp
#include <simd/simd.h>

struct VertexLit {
    simd::float3 position;
    simd::float3 normal;
};

// Strict 16-byte vector alignment packing scalar parameters into vector .w
struct alignas(16) LightUniforms {
    simd::float4x4 modelMatrix;          // 64 bytes (offset 0..63)
    simd::float4x4 viewProjectionMatrix; // 64 bytes (offset 64..127)
    simd::float3x3 normalMatrix;         // 48 bytes (offset 128..175)
    simd::float4   cameraPosAndAmbient;  // xyz = cameraPos, w = ambientIntensity (offset 176..191)
    simd::float4   lightPosAndShininess; // xyz = lightPos,  w = specularShininess (offset 192..207)
    simd::float4   lightColorAndSpec;    // xyz = lightColor,w = specularIntensity (offset 208..223)
};
```

### 2. MSL Blinn-Phong Shader
```metal
#include <metal_stdlib>
using namespace metal;

struct VertexInput {
    float3 position;
    float3 normal;
};

struct LightUniforms {
    float4x4 modelMatrix;
    float4x4 viewProjectionMatrix;
    float3x3 normalMatrix;
    float4   cameraPosAndAmbient;
    float4   lightPosAndShininess;
    float4   lightColorAndSpec;
};

struct RasterizerData {
    float4 clipPosition [[position]];
    float3 worldPosition;
    float3 worldNormal;
};

vertex RasterizerData litVertexShader(uint vertexID [[vertex_id]],
                                      constant VertexInput* vertices [[buffer(0)]],
                                      constant LightUniforms& uniforms [[buffer(1)]]) {
    RasterizerData out;
    float4 worldPos = uniforms.modelMatrix * float4(vertices[vertexID].position, 1.0);
    out.clipPosition = uniforms.viewProjectionMatrix * worldPos;
    out.worldPosition = worldPos.xyz;
    out.worldNormal = normalize(uniforms.normalMatrix * vertices[vertexID].normal);
    return out;
}

fragment float4 litFragmentShader(RasterizerData in [[stage_in]],
                                  constant LightUniforms& uniforms [[buffer(1)]]) {
    float3 N = normalize(in.worldNormal);
    float3 lightPos = uniforms.lightPosAndShininess.xyz;
    float  shininess = uniforms.lightPosAndShininess.w;
    float3 cameraPos = uniforms.cameraPosAndAmbient.xyz;
    float  ambientIntensity = uniforms.cameraPosAndAmbient.w;
    float3 lightColor = uniforms.lightColorAndSpec.xyz;
    float  specIntensity = uniforms.lightColorAndSpec.w;

    float3 L = normalize(lightPos - in.worldPosition);
    float3 V = normalize(cameraPos - in.worldPosition);
    float3 H = normalize(L + V);

    // 1. Ambient
    float3 ambient = ambientIntensity * lightColor;

    // 2. Diffuse
    float diff = max(dot(N, L), 0.0);
    float3 diffuse = diff * lightColor;

    // 3. Specular (Blinn-Phong)
    float spec = pow(max(dot(N, H), 0.0), shininess);
    float3 specular = specIntensity * spec * lightColor;

    float3 finalColor = ambient + diffuse + specular;
    return float4(finalColor, 1.0);
}
```

### 3. Orbit Camera Controller (C++)
```cpp
class OrbitCamera {
public:
    void update(float deltaYaw, float deltaPitch, float deltaZoom) {
        _yaw += deltaYaw;
        _pitch = std::clamp(_pitch + deltaPitch, -1.5f, 1.5f); // Prevent gimbal lock
        _radius = std::max(1.0f, _radius - deltaZoom);

        _eye.x = _radius * std::cos(_pitch) * std::sin(_yaw);
        _eye.y = _radius * std::sin(_pitch);
        _eye.z = _radius * std::cos(_pitch) * std::cos(_yaw);
    }

    simd::float3 getEye() const { return _eye; }
    simd::float4x4 getViewMatrix() const {
        return makeLookAt(_eye, { 0.0f, 0.0f, 0.0f }, { 0.0f, 1.0f, 0.0f });
    }

private:
    float _yaw = 0.0f;
    float _pitch = 0.3f;
    float _radius = 5.0f;
    simd::float3 _eye;
};
```

## 4. Step-by-Step Implementation Checklist
- [ ] Define `VertexLit` including normal vectors for each vertex.
- [ ] Construct vertex buffer for a 3D cube with outward-facing normals (24 vertices: 4 per face).
- [ ] Implement `simd::float3x3` normal matrix calculation: `transpose(inverse(model3x3))`.
- [ ] Implement `OrbitCamera` class calculating eye position from spherical coordinates.
- [ ] Write MSL vertex and fragment shaders computing ambient, diffuse, and Blinn specular terms.
- [ ] Hook AppKit mouse events (`mouseDragged`, `scrollWheel`) into `OrbitCamera`.
- [ ] Pass `LightUniforms` to both vertex and fragment stages at buffer index 1.
- [ ] Run application; rotate camera around illuminated 3D cube with interactive specular highlights.

## 5. Common Pitfalls & Debugging Checklist
- **Inverted or Distorted Specular Highlights**:
  - *Cause*: Normal vector not normalized in fragment shader.
  - *Fix*: Rasterization interpolation reduces normal vector lengths; always re-normalize: `float3 N = normalize(in.worldNormal)`.
- **`simd::float3x3` Memory Alignment Crash**:
  - *Cause*: In MSL, each column of a `float3x3` is padded to 16 bytes (size of `float4`).
  - *Fix*: Ensure C++ matrix struct pads column vectors to 16 bytes, or pass `float4x4` for the normal transform.
- **Negative Dot Products Producing Dark Artifacts**:
  - *Fix*: Always clamp light products to positive values using `max(dot(N, L), 0.0)`.
