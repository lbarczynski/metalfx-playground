# TASK-10: GPU Profiling & Metal Capture Infrastructure

## 1. Concept & Mental Model

### The Kotlin / Android Bridge
In Android, profiling graphics usually involves Android Studio's GPU Inspector, Perfetto system traces, or FrameMetrics API. 

In Modern C++ and Metal:
- **Non-blocking GPU Timers**: Instead of synchronizing the CPU with the GPU (which ruins performance), Metal command buffers record true hardware execution timestamps asynchronously (`gpuStartTime()` and `gpuEndTime()`).
- **Programmatic Xcode Capture**: You can programmatically trigger a frame capture using `MTL::CaptureManager`. This saves a `.gputrace` package that you can double-click to open in Xcode's GPU Frame Debugger, displaying every pipeline state, texture attachment, and shader execution timeline down to the microsecond.

---

## 2. Mathematical Foundations & Algorithms

### Hardware GPU Execution Duration
Metal command buffers measure execution in hardware clock ticks converted to fractional seconds (`CFTimeInterval`):
$$\Delta t_{\text{gpu}} = t_{\text{gpuEndTime}} - t_{\text{gpuStartTime}} \quad (\text{seconds})$$
$$\Delta t_{\text{gpu, ms}} = \Delta t_{\text{gpu}} \times 1000.0 \text{ ms}$$

### Exponential Moving Average (EMA) Frame Profiler
To display stable timing metrics without single-frame fluctuations:
$$\overline{t}_k = \alpha \cdot t_k + (1 - \alpha) \cdot \overline{t}_{k-1}$$
$$\text{FPS}_{\text{smoothed}} = \frac{1000.0}{\overline{t}_k \text{ (ms)}}$$
where $\alpha \approx 0.05$ (equivalent to a $\sim 20$-frame rolling average).

---

## 3. Metal & C++ Blueprint

### 1. Asynchronous GPU Timestamp Collector (`src/GpuProfiler.hpp`)
```cpp
#pragma once
#include <Metal/Metal.hpp>
#include <iostream>
#include <atomic>
#include <algorithm>

class GpuProfiler {
public:
    void registerCommandBuffer(MTL::CommandBuffer* pCmd) {
        // C++ lambda automatically bridges to block under AppleClang -fblocks
        pCmd->addCompletedHandler([this](MTL::CommandBuffer* cmd) {
            CFTimeInterval start = cmd->GPUStartTime();
            CFTimeInterval end = cmd->GPUEndTime();
            
            // Non-zero values guarantee the buffer executed on GPU
            if (start > 0.0 && end > 0.0) {
                double durationMs = (end - start) * 1000.0;
                double current = _smoothGpuTimeMs.load(std::memory_order_relaxed);
                double updated = _alpha * durationMs + (1.0 - _alpha) * current;
                _smoothGpuTimeMs.store(updated, std::memory_order_relaxed);
            }
        });
    }

    double getSmoothedGpuTimeMs() const { return _smoothGpuTimeMs.load(std::memory_order_relaxed); }
    double getFps() const { return 1000.0 / std::max(0.001, getSmoothedGpuTimeMs()); }

private:
    std::atomic<double> _smoothGpuTimeMs{16.6};
    double _alpha = 0.05;
};
```

### 2. Programmatic Xcode GPU Trace Capture (`MTL::CaptureManager`)
```cpp
#include <Metal/Metal.hpp>
#include <Foundation/Foundation.hpp>
#include <filesystem>

class MetalCaptureSession {
public:
    static bool captureFrame(MTL::CommandQueue* pQueue, const char* pOutputFilePath) {
        // startCapture fails if the output directory already exists
        std::error_code ec;
        std::filesystem::remove_all(pOutputFilePath, ec);

        MTL::CaptureManager* pMgr = MTL::CaptureManager::sharedCaptureManager();
        if (!pMgr->supportsDestination(MTL::CaptureDestinationGPUTraceDocument)) {
            std::cerr << "GPU Trace capture unsupported or METAL_CAPTURE_ENABLED not set.\n"
                      << "Tip: Run 'export METAL_CAPTURE_ENABLED=1' in your shell.\n";
            return false;
        }

        MTL::CaptureDescriptor* pDesc = MTL::CaptureDescriptor::alloc()->init();
        pDesc->setCaptureObject(pQueue);
        pDesc->setDestination(MTL::CaptureDestinationGPUTraceDocument);

        NS::String* pathStr = NS::String::string(pOutputFilePath, NS::UTF8StringEncoding);
        NS::URL* outputURL = NS::URL::fileURLWithPath(pathStr);
        pDesc->setOutputURL(outputURL);

        NS::Error* pError = nullptr;
        bool success = pMgr->startCapture(pDesc, &pError);
        pDesc->release();

        if (!success) {
            std::cerr << "Failed to start capture: " 
                      << pError->localizedDescription()->utf8String() << "\n";
            return false;
        }

        std::cout << ">>> GPU CAPTURE STARTED: Saving to " << pOutputFilePath << "\n";
        return true;
    }

    static void endCapture() {
        MTL::CaptureManager* pMgr = MTL::CaptureManager::sharedCaptureManager();
        if (pMgr->isCapturing()) {
            pMgr->stopCapture();
            std::cout << ">>> GPU CAPTURE COMPLETE.\n";
        }
    }
};
```

---

## 4. Step-by-Step Implementation Checklist
- [ ] Implement `GpuProfiler` class registering `addCompletedHandler` on command buffers.
- [ ] Compute `(gpuEndTime - gpuStartTime) * 1000.0` inside completion handler.
- [ ] Implement Exponential Moving Average filter ($\alpha = 0.05$) to smooth millisecond readouts.
- [ ] Implement `MetalCaptureSession` using `MTL::CaptureManager`.
- [ ] Wire a keyboard trigger (e.g. 'C' key) to trigger `captureFrame()` before encoding and `endCapture()` after `commit()`.
- [ ] Open the resulting `.gputrace` file in Xcode to inspect draw call timings and shader pipeline execution.

---

## 5. Definition of Done & Commit Message
- **DoD**: Application logs live microsecond GPU frame time; pressing 'C' generates a valid `.gputrace` archive inspectable in Xcode Frame Debugger.
- **Commit**:
  ```bash
  git add src/GpuProfiler.hpp src/Renderer.cpp
  git commit -m "feat(profile): integrate metal gpu profiling and programmatic capture"
  ```
