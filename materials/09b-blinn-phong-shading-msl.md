# TASK-09B: Blinn-Phong Shading in MSL

## 1. Concept & Mental Model

### The Blinn-Phong Reflection Model
Flat colored geometry looks like plastic or paper. Real surfaces interact with light in three distinct ways:

$$\mathbf{I}_{\text{total}} = \mathbf{I}_{\text{ambient}} + \mathbf{I}_{\text{diffuse}} + \mathbf{I}_{\text{specular}}$$

1. **Ambient**: Constant omnidirectional baseline light scattered by the environment.
2. **Diffuse (Lambertian)**: Light scattered equally in all directions, proportional to the cosine of the angle between surface normal $\mathbf{N}$ and light direction $\mathbf{L}$:
$$\mathbf{I}_{\text{diffuse}} = k_d \cdot \max(0, \mathbf{N} \cdot \mathbf{L}) \cdot \mathbf{C}_{\text{surface}}$$
3. **Specular (Blinn-Phong)**: Bright glare or highlight reflecting off glossy surfaces. Evaluated using the **halfway vector** $\mathbf{H} = \frac{\mathbf{L} + \mathbf{V}}{\|\mathbf{L} + \mathbf{V}\|}$:
$$\mathbf{I}_{\text{specular}} = k_s \cdot (\max(0, \mathbf{N} \cdot \mathbf{H}))^{\alpha} \cdot \mathbf{C}_{\text{light}}$$
Where $\alpha$ is the specular shininess exponent (e.g. 32 to 128).

---

## 2. Vectors and Geometric Relations

```
       Light (L)              Normal (N)          Camera View (V)
           ▲                       ▲                    ▲
            \                      │                   /
             \                     │  Halfway (H)     /
              \                    │     ▲           /
               \                   │    /           /
                \                  │   /           /
                 \                 │  /           /
                  \                │ /           /
                   \               │/           /
       ─────────────┴──────────────┴───────────┴───────────── Surface
```

---

## 3. Metal & C++ Blueprint

### 1. Light Uniforms Struct (`src/SimdMath.hpp`)
```cpp
struct alignas(16) LightUniforms {
    simd::float4 lightDirection; // Normalized, w = 0
    simd::float4 lightColor;     // RGB, w = ambient intensity
    simd::float4 cameraPosition; // World space eye position
    float        shininess;
    float        _pad[3];
};

static_assert(sizeof(LightUniforms) % 16 == 0, "LightUniforms alignment error!");
```

### 2. MSL Lighting Shaders
```metal
#include <metal_stdlib>
using namespace metal;

struct LitVertexInput {
    float3 position;
    float3 normal;
    float2 uv;
};

struct LitVertexOut {
    float4 position [[position]];
    float3 worldPosition;
    float3 worldNormal;
    float2 uv;
};

struct FrameUniforms {
    float4x4 modelMatrix;
    float4x4 viewProjectionMatrix;
};

struct LightUniforms {
    float4 lightDirection;
    float4 lightColor;
    float4 cameraPosition;
    float  shininess;
};

vertex LitVertexOut litVertexMain(
    uint vertexID [[vertex_id]],
    constant LitVertexInput* vertices [[buffer(0)]],
    constant FrameUniforms& frame     [[buffer(1)]]
) {
    LitVertexOut out;
    float4 worldPos = frame.modelMatrix * float4(vertices[vertexID].position, 1.0);
    out.worldPosition = worldPos.xyz;
    out.position = frame.viewProjectionMatrix * worldPos;

    // Transform normal with model matrix (assuming uniform scale)
    out.worldNormal = (frame.modelMatrix * float4(vertices[vertexID].normal, 0.0)).xyz;
    out.uv = vertices[vertexID].uv;
    return out;
}

fragment float4 litFragmentMain(
    LitVertexOut in [[stage_in]],
    constant LightUniforms& light [[buffer(0)]],
    texture2d<float> colorTexture [[texture(0)]],
    sampler colorSampler          [[sampler(0)]]
) {
    float3 N = normalize(in.worldNormal);
    float3 L = normalize(-light.lightDirection.xyz);
    float3 V = normalize(light.cameraPosition.xyz - in.worldPosition);
    float3 H = normalize(L + V);

    float4 surfaceColor = colorTexture.sample(colorSampler, in.uv);

    // 1. Ambient
    float ambientIntensity = light.lightColor.w;
    float3 ambient = ambientIntensity * surfaceColor.rgb;

    // 2. Diffuse
    float NdotL = max(0.0f, dot(N, L));
    float3 diffuse = NdotL * light.lightColor.rgb * surfaceColor.rgb;

    // 3. Specular
    float NdotH = max(0.0f, dot(N, H));
    float specularFactor = pow(NdotH, light.shininess);
    float3 specular = specularFactor * light.lightColor.rgb;

    float3 finalColor = ambient + diffuse + specular;
    return float4(finalColor, surfaceColor.a);
}
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Update cube geometry to 24 vertices so each face has independent perpendicular normals.
- [ ] Define `LightUniforms` struct in C++ matching MSL layout.
- [ ] Add directional light (e.g. pointing $[-0.5, -1.0, -0.3]$ with white light).
- [ ] Bind `LightUniforms` to fragment buffer 0 in `renderFrame()`.
- [ ] Orbit camera around cube: Verify that faces facing the light shine brightly, and a sharp specular highlight glints off the surface as the camera moves.

---

## 5. Definition of Done & Commit Message
- **DoD**: 3D cube geometry renders with realistic Blinn-Phong directional shading, diffuse falloff, and specular highlights reacting to camera movement.
- **Commit**:
  ```bash
  git add src/Renderer.cpp src/SimdMath.hpp src/Shaders.metal
  git commit -m "feat(shader): implement blinn-phong lighting model in msl"
  ```
