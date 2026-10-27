# TASK-02A: Platform Window Bridge

## 1. Concept & Mental Model

### The Kotlin / Android Bridge
In Android, the system manages windowing and lifecycle through `Activity`, `Window`, and the main thread `Looper`. Your rendering surface is typically an `ANativeWindow` provided by a `SurfaceView`.

On macOS:
- The OS UI framework is **Cocoa / AppKit** (`NSApplication`, `NSWindow`, `NSView`).
- `metal-cpp` deliberately does not include C++ wrappers for AppKit because AppKit is a massive, dynamic Objective-C API.
- In professional game engines (Unreal, Frostbite, custom native engines), developers write a thin **Objective-C++ (`.mm`) bridge**. This bridge creates the window and hooks the event loop, exposing a clean, pure C++ interface to your engine.

---

## 2. Architecture & Design

```
[ Cocoa / AppKit (macOS) ]
        │  (Objective-C++)
[ WindowBridge.mm ] ─── exposes pure C++ API ───> [ WindowBridge.hpp ]
                                                          │
                                                    [ main.cpp (Pure C++) ]
```

### Pure C++ Interface (`src/platform/WindowBridge.hpp`)
```cpp
#pragma once
#include <functional>
#include <string>

namespace platform {

struct WindowConfig {
    int width = 1280;
    int height = 720;
    std::string title = "MetalFX Playground";
};

class Window {
public:
    using RenderCallback = std::function<void()>;
    using ResizeCallback = std::function<void(int width, int height)>;

    static Window& instance();

    bool init(const WindowConfig& config);
    void run(RenderCallback onRender, ResizeCallback onResize = nullptr);
    void close();

    void* nativeWindow() const;
    void* nativeView() const;

private:
    Window() = default;
    ~Window();
    void* _opaqueApp = nullptr;
    void* _opaqueWindow = nullptr;
    void* _opaqueView = nullptr;
};

} // namespace platform
```

### Objective-C++ Implementation (`src/platform/WindowBridge.mm`)
In an `.mm` file, you can freely mix C++ with native Objective-C Cocoa calls:
```objc
#import <Cocoa/Cocoa.h>
#import <QuartzCore/CAMetalLayer.h>
#include "WindowBridge.hpp"

@interface AppDelegate : NSObject <NSApplicationDelegate, NSWindowDelegate>
@property (nonatomic, assign) platform::Window::RenderCallback renderCallback;
@property (nonatomic, assign) platform::Window::ResizeCallback resizeCallback;
@property (nonatomic, assign) BOOL shouldKeepRunning;
@end

@implementation AppDelegate
- (void)windowWillClose:(NSNotification *)notification {
    _shouldKeepRunning = NO;
}

- (void)windowDidResize:(NSNotification *)notification {
    if (_resizeCallback) {
        NSWindow* win = (NSWindow*)notification.object;
        NSSize size = win.contentView.bounds.size;
        _resizeCallback(static_cast<int>(size.width), static_cast<int>(size.height));
    }
}
@end

namespace platform {

Window& Window::instance() {
    static Window s_window;
    return s_window;
}

Window::~Window() = default;

bool Window::init(const WindowConfig& config) {
    [NSApplication sharedApplication];
    [NSApp setActivationPolicy:NSApplicationActivationPolicyRegular];

    NSRect frame = NSMakeRect(0, 0, config.width, config.height);
    NSWindowStyleMask mask = NSWindowStyleMaskTitled | 
                             NSWindowStyleMaskClosable | 
                             NSWindowStyleMaskResizable;

    NSWindow* window = [[NSWindow alloc] initWithContentRect:frame
                                                   styleMask:mask
                                                     backing:NSBackingStoreBuffered
                                                       defer:NO];
    [window center];
    [window setTitle:[NSString stringWithUTF8String:config.title.c_str()]];

    NSView* view = [[NSView alloc] initWithFrame:frame];
    [view setWantsLayer:YES];
    [window setContentView:view];
    [window makeKeyAndOrderFront:nil];

    _opaqueApp = (__bridge void*)NSApp;
    _opaqueWindow = (__bridge void*)window;
    _opaqueView = (__bridge void*)view;

    return true;
}

void Window::run(RenderCallback onRender, ResizeCallback onResize) {
    AppDelegate* delegate = [[AppDelegate alloc] init];
    delegate.renderCallback = onRender;
    delegate.resizeCallback = onResize;
    delegate.shouldKeepRunning = YES;

    NSWindow* window = (__bridge NSWindow*)_opaqueWindow;
    [window setDelegate:delegate];

    [NSApp finishLaunching];
    [NSApp activateIgnoringOtherApps:YES];

    // Main interactive frame loop
    while (delegate.shouldKeepRunning) {
        @autoreleasepool {
            NSEvent* event;
            while ((event = [NSApp nextEventMatchingMask:NSEventMaskAny
                                               untilDate:nil
                                                  inMode:NSDefaultRunLoopMode
                                                 dequeue:YES])) {
                [NSApp sendEvent:event];
                [NSApp updateWindows];
            }

            if (onRender) {
                onRender();
            }
        }
    }
}

void Window::close() {
    if (_opaqueWindow) {
        NSWindow* window = (__bridge NSWindow*)_opaqueWindow;
        [window close];
    }
}

void* Window::nativeWindow() const { return _opaqueWindow; }
void* Window::nativeView() const { return _opaqueView; }

} // namespace platform
```

---

## 3. Step-by-Step Implementation Checklist
- [ ] Create `src/platform/WindowBridge.hpp` declaring `platform::Window`.
- [ ] Create `src/platform/WindowBridge.mm` implementing window creation and Cocoa event pump.
- [ ] In `src/main.cpp`, initialize `platform::Window`, specify dimensions $1280 \times 720$, and call `run()`.
- [ ] Build with CMake: `cmake --build --preset macos-debug-build`.
- [ ] Launch `./build/macos-debug/src/metalfx_app`: A native window opens, can be moved/resized, and cleanly exits when clicking the close button.

---

## 4. Definition of Done & Commit Message
- **DoD**: A native macOS window titled "MetalFX Playground" opens on screen, processes window events (resize, move), and exits cleanly upon closing without hanging.
- **Commit**:
  ```bash
  git add src/platform/WindowBridge.hpp src/platform/WindowBridge.mm src/main.cpp
  git commit -m "feat(platform): implement native cocoa window bridge"
  ```
