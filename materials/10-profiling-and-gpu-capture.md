# TASK-10: GPU Profiling & Metal Capture Infrastructure

## 1. Concept & Mental Model
Accurate GPU performance profiling requires extracting true on-chip execution timestamps from the GPU command queue without introducing CPU synchronization stalls, complemented by programmatic capture triggers that export `.gputrace` archives for deep Xcode shader profiling. In OpenGL, timers rely on `glBeginQuery(GL_TIME_ELAPSED)`; in Metal, non-blocking GPU timestamps are retrieved directly via `MTL::CommandBuffer::gpuStartTime()` and `gpuEndTime()`, and programmatic GPU frame capture is orchestrated via `MTL::CaptureManager`.

## 2. Mathematical Foundations & Algorithms
### Hardware GPU Execution Duration
Metal command buffers measure execution in real-time hardware clock ticks converted to fractional seconds (`CFTimeInterval`):
$$\Delta t_{\text{gpu}} = t_{\text{gpuEndTime}} - t_{\text{gpuStartTime}} \quad (\text{seconds})$$
$$\Delta t_{\text{gpu, ms}} = \Delta t_{\text{gpu}} \times 1000.0 \text{ ms}$$

### Exponential Moving Average (EMA) Frame Profiler
To display stable timing metrics without single-frame spikes:
$$\overline{t}_k = \alpha \cdot t_k + (1 - \alpha) \cdot \overline{t}_{k-1}$$
$$\text{FPS}_{\text{smoothed}} = \frac{1000.0}{\overline{t}_k \text{ (ms)}}$$
where $\alpha \approx 0.05$ (equivalent to a $\sim 20$-frame rolling average).

## 3. Metal & C++ Blueprint

### 1. Asynchronous GPU Timestamp Collector
```cpp
#include <Metal/Metal.hpp>
#include <iostream>
#include <atomic>

class GpuProfiler {
public:
    void registerCommandBuffer(MTL::CommandBuffer* pCmd) {
        pCmd->addCompletedHandler(^(MTL::CommandBuffer* cmd) {
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

class MetalCaptureSession {
public:
    static bool captureFrame(MTL::CommandQueue* pQueue, const char* pOutputFilePath) {
        MTL::CaptureManager* pMgr = MTL::CaptureManager::sharedCaptureManager();
        if (!pMgr->supportsDestination(MTL::CaptureDestinationGPUTraceDocument)) {
            std::cerr << "GPU Trace capture unsupported on this device.\n";
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

### 3. Usage in Render Loop
```cpp
// Check if user pressed key (e.g., 'C' for Capture)
if (g_TriggerCaptureNextFrame) {
    MetalCaptureSession::captureFrame(pQueue, "/tmp/frame_capture.gputrace");
}

MTL::CommandBuffer* cmd = pQueue->commandBuffer();
// [Encode render/compute passes...]

profiler.registerCommandBuffer(cmd);

cmd->presentDrawable(drawable);
cmd->commit();

if (g_TriggerCaptureNextFrame) {
    MetalCaptureSession::endCapture();
    g_TriggerCaptureNextFrame = false;
}
```

## 4. Step-by-Step Implementation Checklist
- [ ] Implement `GpuProfiler` class using `cmd->addCompletedHandler()`.
- [ ] Compute `(gpuEndTime - gpuStartTime) * 1000.0` inside completion handler.
- [ ] Implement Exponential Moving Average filter ($\alpha = 0.05$) to smooth millisecond readouts.
- [ ] Implement `MetalCaptureSession` using `MTL::CaptureManager`.
- [ ] Set `MTL::CaptureDestinationGPUTraceDocument` and define output URL path.
- [ ] Wire keyboard trigger (e.g. Space or 'C') to start capture before command encoding and stop capture immediately after `commit()`.
- [ ] Open the resulting `.gputrace` file in Xcode to inspect draw call timings and shader pipeline execution.

## 5. Common Pitfalls & Debugging Checklist
- **`gpuStartTime` and `gpuEndTime` Return 0.0**:
  - *Cause*: Attempting to read timestamps before the command buffer completes or inspecting a dropped buffer.
  - *Fix*: Only access timestamps inside the `addCompletedHandler` callback after checking `start > 0.0 && end > 0.0`.
- **Capture Fails with "Destination Not Supported"**:
  - *Cause*: Info.plist lacks Metal capture permissions or `MetalCaptureEnabled` flag is missing in scheme settings.
  - *Fix*: In Xcode or CLI environment, set `export METAL_CAPTURE_ENABLED=1`.
- **File Overwrite Error during Capture**:
  - *Error*: `startCapture` fails if the target `.gputrace` folder already exists on disk.
  - *Fix*: Delete existing trace files before initiating capture, or generate unique timestamped filenames.
