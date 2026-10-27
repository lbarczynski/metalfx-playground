# TASK-02: Native Windowing & The Metal Render Loop

## 1. Concept & Mental Model
The Metal render loop on macOS pairs Cocoa's application lifecycle (`NS::Application`, `NS::Window`) with a MetalKit view (`MTK::View`) and view delegate (`MTK::ViewDelegate`), presenting frame drawables synchronized to display refresh (CVDisplayLink / ProMotion). In Android Compose/SurfaceView, you implement `SurfaceHolder.Callback`; in GLFW/OpenGL, you run an explicit `while(!glfwWindowShouldClose()) { glfwSwapBuffers(); }` loop; in MetalKit, the OS drives `drawInMTKView()` callbacks at the display's native refresh rate.

## 2. Mathematical Foundations & Algorithms
### Periodic Clear Color Cycle
To verify the render loop without static artifacts, modulate the clear color across normalized RGB space $[0.0, 1.0]^3$ using continuous time $t$ with $120^\circ$ ($\frac{2\pi}{3}$ rad) phase separation:
$$R(t) = \frac{1}{2}\left(1.0 + \sin(\omega t)\right)$$
$$G(t) = \frac{1}{2}\left(1.0 + \sin\left(\omega t + \frac{2\pi}{3}\right)\right)$$
$$B(t) = \frac{1}{2}\left(1.0 + \sin\left(\omega t + \frac{4\pi}{3}\right)\right)$$
where $\omega = 2\pi f$ (e.g., $f = 0.5 \text{ Hz}$ for a 2-second full color cycle).

### Frame Delta Time
For frame $k$ at wall-clock time $t_k$:
$$\Delta t_k = t_k - t_{k-1}$$

## 3. Metal & C++ Blueprint

### 1. View Delegate & Renderer Architecture
```cpp
#include <Metal/Metal.hpp>
#include <MetalKit/MetalKit.hpp>
#include <chrono>
#include <cmath>

class Renderer : public MTK::ViewDelegate {
public:
    explicit Renderer(MTL::Device* pDevice) 
        : _pDevice(pDevice->retain())
        , _pCommandQueue(pDevice->newCommandQueue())
        , _startTime(std::chrono::high_resolution_clock::now()) {}

    virtual ~Renderer() override {
        _pCommandQueue->release();
        _pDevice->release();
    }

    virtual void drawInMTKView(MTK::View* pView) override {
        // Autorelease pool per frame is strictly required to prevent autorelease accumulation
        NS::AutoreleasePool* pool = NS::AutoreleasePool::alloc()->init();

        auto now = std::chrono::high_resolution_clock::now();
        float t = std::chrono::duration<float>(now - _startTime).count();

        // Calculate phase-shifted sinusoidal RGB
        double r = 0.5 * (1.0 + std::sin(t * 2.0));
        double g = 0.5 * (1.0 + std::sin(t * 2.0 + 2.0943951)); // + 2pi/3
        double b = 0.5 * (1.0 + std::sin(t * 2.0 + 4.1887902)); // + 4pi/3

        MTL::RenderPassDescriptor* passDesc = pView->currentRenderPassDescriptor();
        CA::MetalDrawable* drawable = pView->currentDrawable();

        // Guard against minimized or zero-size window draws
        if (passDesc && drawable) {
            MTL::RenderPassColorAttachmentDescriptor* colorAttachment = 
                passDesc->colorAttachments()->object(0);
            
            colorAttachment->setClearColor(MTL::ClearColor::Make(r, g, b, 1.0));
            colorAttachment->setLoadAction(MTL::LoadActionClear);
            colorAttachment->setStoreAction(MTL::StoreActionStore);

            MTL::CommandBuffer* cmd = _pCommandQueue->commandBuffer();
            MTL::RenderCommandEncoder* enc = cmd->renderCommandEncoder(passDesc);
            
            // Render commands would be encoded here
            enc->endEncoding();

            cmd->presentDrawable(drawable);
            cmd->commit();
        }

        pool->release();
    }

    virtual void drawableSizeWillChange(MTK::View* pView, CGSize size) override {
        (void)pView;
        (void)size;
    }

private:
    MTL::Device* _pDevice;
    MTL::CommandQueue* _pCommandQueue;
    std::chrono::high_resolution_clock::time_point _startTime;
};
```

### 2. Native Cocoa Application Delegate (Pure C++)
```cpp
#include <AppKit/AppKit.hpp>

class AppDelegate : public NS::ApplicationDelegate {
public:
    virtual ~AppDelegate() override {
        delete _pRenderer;
        _pView->release();
        _pWindow->release();
        _pDevice->release();
    }

    virtual void applicationWillFinishLaunching(NS::Notification* pNotification) override {
        (void)pNotification;
        NS::Application* app = reinterpret_cast<NS::Application*>(pNotification->object());
        app->setActivationPolicy(NS::ActivationPolicyRegular);
    }

    virtual void applicationDidFinishLaunching(NS::Notification* pNotification) override {
        (void)pNotification;
        _pDevice = MTL::CreateSystemDefaultDevice();

        const CGRect frame = CGRectMake(100.0, 100.0, 800.0, 600.0);
        _pWindow = NS::Window::alloc()->init(
            frame,
            NS::WindowStyleMaskTitled | NS::WindowStyleMaskClosable | NS::WindowStyleMaskResizable,
            NS::BackingStoreBuffered,
            false
        );

        _pView = MTK::View::alloc()->init(frame, _pDevice);
        _pView->setColorPixelFormat(MTL::PixelFormatBGRA8Unorm_sRGB);
        _pView->setClearColor(MTL::ClearColor::Make(0.1, 0.1, 0.1, 1.0));

        _pRenderer = new Renderer(_pDevice);
        _pView->setDelegate(_pRenderer);

        _pWindow->setContentView(_pView);
        _pWindow->setTitle(NS::String::string("Metal Render Loop", NS::UTF8StringEncoding));
        _pWindow->makeKeyAndOrderFront(nullptr);

        NS::Application* app = NS::Application::sharedApplication();
        app->activateIgnoringOtherApps(true);
    }

    virtual bool applicationShouldTerminateAfterLastWindowClosed(NS::Application* pApp) override {
        (void)pApp;
        return true;
    }

private:
    NS::Window* _pWindow = nullptr;
    MTK::View* _pView = nullptr;
    MTL::Device* _pDevice = nullptr;
    Renderer* _pRenderer = nullptr;
};
```

### 3. Entry Point (`main.cpp`)
```cpp
int main() {
    NS::AutoreleasePool* pool = NS::AutoreleasePool::alloc()->init();

    {
        NS::Application* app = NS::Application::sharedApplication();
        AppDelegate delegate;
        app->setDelegate(&delegate);
        app->run();
    } // delegate destroyed here while autorelease pool is still valid

    pool->release();
    return 0;
}
```

> [!NOTE]
> **Headless & CI/CD Testing**: `app->run()` launches a blocking Cocoa GUI event loop. For automated testing and CI runners without a display, execute offscreen render passes using a standalone `MTL::CommandQueue` and headless render targets without `NS::Application`.


## 4. Step-by-Step Implementation Checklist
- [ ] Ensure `AppKit/AppKit.hpp` and `MetalKit/MetalKit.hpp` are included and CMake links `-framework AppKit` and `-framework MetalKit`.
- [ ] Implement `Renderer` conforming to `MTK::ViewDelegate`.
- [ ] In `Renderer::drawInMTKView()`, wrap all rendering calls in `NS::AutoreleasePool`.
- [ ] Check `pView->currentRenderPassDescriptor()` and `pView->currentDrawable()` for `nullptr` prior to encoding.
- [ ] Implement `AppDelegate` conforming to `NS::ApplicationDelegate`.
- [ ] Create `NS::Window` and embed an `MTK::View` configured with `MTL::PixelFormatBGRA8Unorm_sRGB`.
- [ ] Call `app->run()` in `main()`.
- [ ] Build and launch: verify a native window opens showing a smooth animated color cycle.

## 5. Common Pitfalls & Debugging Checklist
- **Uncontrolled Autorelease Memory Leak**:
  - *Symptom*: Process memory grows by tens of megabytes every second until macOS terminates the process.
  - *Fix*: Allocate and release `NS::AutoreleasePool` inside `drawInMTKView()` every single frame.
- **Null Pointer Dereference Crash on Minimized Window**:
  - *Symptom*: Crash inside `drawInMTKView` when window is minimized or hidden.
  - *Fix*: Always verify `if (passDesc && drawable)` before calling `commandBuffer()` or encoding passes.
- **Window Opens Behind Other Windows**:
  - *Fix*: Call `app->setActivationPolicy(NS::ActivationPolicyRegular)` and `app->activateIgnoringOtherApps(true)`.
- **View Delegate Never Fires**:
  - *Fix*: Ensure `_pView->setDelegate(_pRenderer)` is called and `_pRenderer` is retained/kept alive on the heap.
