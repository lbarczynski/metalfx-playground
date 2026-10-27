# TASK-05A: Procedural Texture Generation & Memory Stride

## 1. Concept & Mental Model

### The Kotlin / Android Bridge
In Android, generating procedural images involves creating an in-memory `android.graphics.Bitmap` or filling a direct `java.nio.ByteBuffer` before uploading it to an OpenGL texture via `glTexImage2D`.

In Modern C++ and Metal:
- **Raw Byte Buffers**: An RGBA8 texture is simply a contiguous array of 32-bit unsigned integers (`uint32_t` or 4 bytes: R, G, B, A).
- **Row Pitch (`bytesPerRow`)**: GPUs fetch texture rows in cache-line chunks. The memory stride (pitch) across rows must be correctly accounted for:
$$\text{bytesPerRow} = \text{width} \times \text{bytesPerPixel} = W \times 4$$
- **Direct Subresource Upload**: Metal does not use staging buffers for shared resources. We populate a `std::vector<uint32_t>` on the CPU and transfer it directly to GPU memory using `MTL::Texture::replaceRegion()`.

---

## 2. Mathematical Foundations: Procedural Checkerboard

For any pixel coordinate $(x, y)$ in a $W \times H$ texture with tile frequency $F$:
$$\text{tile}_x = \lfloor \frac{x}{F} \rfloor, \quad \text{tile}_y = \lfloor \frac{y}{F} \rfloor$$
$$\text{isWhite} = (\text{tile}_x + \text{tile}_y) \pmod 2 == 0$$

---

## 3. Metal & C++ Blueprint

### Texture Generator Helper (`src/TextureUtils.hpp`)
```cpp
#pragma once
#include <Metal/Metal.hpp>
#include <vector>
#include <cstdint>

inline MTL::Texture* createCheckerboardTexture(MTL::Device* pDevice, uint32_t width, uint32_t height) {
    constexpr uint32_t kTileSize = 32;
    std::vector<uint32_t> pixels(width * height);

    for (uint32_t y = 0; y < height; ++y) {
        for (uint32_t x = 0; x < width; ++x) {
            bool isWhite = ((x / kTileSize) + (y / kTileSize)) % 2 == 0;
            pixels[y * width + x] = isWhite ? 0xFFFFFFFF : 0xFF202020; // 0xAABBGGRR in Little Endian
        }
    }

    // Configure texture descriptor
    MTL::TextureDescriptor* desc = MTL::TextureDescriptor::texture2DDescriptor(
        MTL::PixelFormatRGBA8Unorm,
        width,
        height,
        false // no mipmaps for now
    );
    desc->setUsage(MTL::TextureUsageShaderRead);
    desc->setStorageMode(MTL::StorageModeShared);

    MTL::Texture* pTexture = pDevice->newTexture(desc);
    desc->release();

    // Upload host pixels to GPU texture
    MTL::Region region = MTL::Region::Make2D(0, 0, width, height);
    pTexture->replaceRegion(region, 0, pixels.data(), width * sizeof(uint32_t));

    return pTexture;
}
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Implement `createCheckerboardTexture` in C++ with width $256$ and height $256$.
- [ ] Calculate `bytesPerRow = width * 4`.
- [ ] Configure `MTL::TextureDescriptor` with `MTL::PixelFormatRGBA8Unorm` and `MTL::TextureUsageShaderRead`.
- [ ] Upload the pixel data using `pTexture->replaceRegion(...)`.
- [ ] Inspect texture object validity and dimensions in console logs.

---

## 5. Definition of Done & Commit Message
- **DoD**: Procedural $256 \times 256$ checkerboard pattern is generated in CPU memory and successfully uploaded to an `MTL::Texture` with valid dimensions and format.
- **Commit**:
  ```bash
  git add src/TextureUtils.hpp src/Renderer.cpp
  git commit -m "feat(texture): generate procedural texture and upload to gpu"
  ```
