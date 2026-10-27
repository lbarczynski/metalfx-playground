# TASK-02B: Metal Render Loop & Clear Pass

## 1. Concept & Mental Model

### The Kotlin / Android Bridge
In Android, rendering to a surface typically uses `Choreographer.postFrameCallback` and draws via `Canvas` or binds an EGL surface. You don't manage low-level driver command buffers directly.

In Metal:
- **Stateless Command Encoding**: Rendering commands are not executed immediately. You create a transient `MTL::CommandBuffer` from an `MTL::CommandQueue`, open an `MTL::RenderCommandEncoder` on a render pass, encode commands, and commit the buffer to the GPU.
- **The Autorelease Trap**: Every frame, Metal and Cocoa generate short-lived Objective-C bridge objects (descriptors, drawables). In Kotlin, GC cleans these up eventually. In C++, if you don't wrap your frame loop body with `NS::AutoreleasePool`, your process memory will grow unbounded by megabytes every second.

---

## 2. Architecture & The Render Pass

```
[ Window Native View ]
        │
  [ CAMetalLayer ] ───> nextDrawable() ───> [ CA::MetalDrawable ] (Presents to screen)
                                                     │
                                             drawable->texture()
                                                     │
[ RenderPassDescriptor ] ── (colorAttachment 0) ─────┘
        │
  [ MTL::CommandQueue ] ──> commandBuffer() ──> renderCommandEncoder(pass)
                                                        │
                                                   endEncoding()
                                                        │
                                               presentDrawable(drawable)
                                                        │
                                                     commit()
```

---

## 3. Metal & C++ Blueprint

### 1. Attaching `CAMetalLayer` to View
In your platform bridge or initialization code:
```cpp
// Set up CAMetalLayer on the native NSView
NSView* nsView = (__bridge NSView*)window.nativeView();
CAMetalLayer* metalLayer = [CAMetalLayer layer];
metalLayer.device = (__bridge id<MTLDevice>)pDevice;
metalLayer.pixelFormat = MTLPixelFormatBGRA8Unorm_sRGB;
metalLayer.framebufferOnly = YES;
nsView.layer = metalLayer;
nsView.wantsLayer = YES;
```

### 2. The Per-Frame Render Function (`src/Renderer.cpp`)
```cpp
#include <Metal/Metal.hpp>
#include <QuartzCore/QuartzCore.hpp>
#include <cmath>
#include <chrono>

class Renderer {
public:
    Renderer(MTL::Device* pDevice, CA::MetalLayer* pLayer)
        : _pDevice(pDevice), _pLayer(pLayer) {
        _pCommandQueue = _pDevice->newCommandQueue();
        _startTime = std::chrono::high_resolution_clock::now();
    }

    ~Renderer() {
        _pCommandQueue->release();
    }

    void renderFrame() {
        // CRITICAL: Prevent memory leaks from frame allocations
        NS::AutoreleasePool* pool = NS::AutoreleasePool::alloc()->init();

        CA::MetalDrawable* pDrawable = _pLayer->nextDrawable();
        if (!pDrawable) {
            pool->release();
            return;
        }

        // Calculate oscillating color using elapsed time
        auto now = std::chrono::high_resolution_clock::now();
        float t = std::chrono::duration<float>(now - _startTime).count();
        double r = 0.5 * (1.0 + std::sin(t * 1.5));
        double g = 0.5 * (1.0 + std::sin(t * 1.5 + 2.0));
        double b = 0.5 * (1.0 + std::sin(t * 1.5 + 4.0));

        // Create render pass descriptor targeting drawable texture
        MTL::RenderPassDescriptor* pass = MTL::RenderPassDescriptor::renderPassDescriptor();
        auto colorAttachment = pass->colorAttachments()->object(0);
        colorAttachment->setTexture(pDrawable->texture());
        colorAttachment->setLoadAction(MTL::LoadActionClear);
        colorAttachment->setStoreAction(MTL::StoreActionStore);
        colorAttachment->setClearColor(MTL::ClearColor::Make(r, g, b, 1.0));

        // Create command buffer and encode clear pass
        MTL::CommandBuffer* cmd = _pCommandQueue->commandBuffer();
        MTL::RenderCommandEncoder* enc = cmd->renderCommandEncoder(pass);
        enc->endEncoding();

        // Present to screen and commit to GPU queue
        cmd->presentDrawable(pDrawable);
        cmd->commit();

        pool->release();
    }

private:
    MTL::Device* _pDevice = nullptr;
    CA::MetalLayer* _pLayer = nullptr;
    MTL::CommandQueue* _pCommandQueue = nullptr;
    std::chrono::high_resolution_clock::time_point _startTime;
};
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Connect `CAMetalLayer` with `MTLPixelFormatBGRA8Unorm_sRGB` to your native window's view.
- [ ] Create `MTL::CommandQueue` from your `MTL::Device`.
- [ ] In the render callback, wrap the frame body in `NS::AutoreleasePool`.
- [ ] Retrieve `nextDrawable()`, check for `nullptr`.
- [ ] Configure `MTL::RenderPassDescriptor` with animated `MTL::ClearColor`.
- [ ] Open and close `MTL::RenderCommandEncoder`, call `cmd->presentDrawable(drawable)` and `cmd->commit()`.
- [ ] Run application: Window should display a smooth, oscillating background color cycle.

---

## 5. Definition of Done & Commit Message
- **DoD**: Application launches a native window displaying a smooth animated color cycle via Metal render passes, maintaining steady 60/120 FPS without memory leaks.
- **Commit**:
  ```bash
  git add src/Renderer.hpp src/Renderer.cpp src/main.cpp
  git commit -m "feat(renderer): implement metal render loop with clear pass"
  ```
