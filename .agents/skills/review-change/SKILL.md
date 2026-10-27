---
name: review-change
description: >-
  Use this skill when the user requests a code review, feedback, or verification of their implementation
  or git changes (e.g., "review my change", "check my code", "did I do this right?", "task review").
---

# Review Change Skill: Systems Programming Code Reviewer

This skill conducts a rigorous, mentor-style code review on the user's latest code changes, evaluating C++20 and Metal/MetalFX best practices.

## Workflow Instructions

### 1. Inspect Current Workspace State
- Run `git status` and `git diff` (or inspect modified files) to understand the exact changes made by the user.
- Identify which task in `WORK_PLAN.md` the user is currently tackling.
- Read the corresponding guide in `materials/` to check against its **Definition of Done** and **Common Pitfalls**.

### 2. Run the Systems Programming Audit Checklist
Evaluate the code against these 4 critical pillars:

#### A. Memory Layout & Struct Alignment
- [ ] Are GPU-shared structs aligned to 16 bytes using `alignas(16)`?
- [ ] Are constant buffer dynamic offsets aligned to 256 bytes (`(size + 255) & ~size_t(255)`)?
- [ ] Did the user avoid accidental padding gaps (e.g., `simd::float3` occupying 16 bytes)?

#### B. Metal Pipeline & State Invariants
- [ ] Do pixel formats match across texture descriptors, pipeline descriptors, and view formats?
- [ ] Are texture usage flags correct (e.g., `RenderTarget | ShaderRead` for offscreen textures/scalers)?
- [ ] Is `setStoreAction(MTL::StoreActionStore)` configured when intermediate textures need to be sampled later?
- [ ] Is `enc->endEncoding()` called before starting another encoder or encoding MetalFX passes?
- [ ] Are winding orders and culling modes aligned (CCW default, or `CullModeNone` for fullscreen procedural blits)?

#### C. Concurrency & Resource Lifetimes
- [ ] Is an `NS::AutoreleasePool` created and released inside every render frame iteration?
- [ ] If using triple-buffering, is `dispatch_semaphore_wait` guarded inside `if (passDesc && drawable)` to prevent deadlocks?
- [ ] Are transient descriptors (`MTL::RenderPipelineDescriptor`, `MTLFX::SpatialScalerDescriptor`) released after creating the pipeline/scaler object?

#### D. Modern C++20 Idioms
- [ ] Are non-owning memory slices passed as `std::span`?
- [ ] Are RAII and value semantics preferred over raw `new`/`delete`?
- [ ] Are headers and includes cleanly separated?

### 3. Deliver Mentor-Style Review Feedback
Provide structured feedback following this format:

```markdown
### 🔍 Code Review: TASK-XX — [Title]

#### ✅ What's Working Well
- [Highlight 1-2 correct architectural decisions or clean C++20 idioms]

#### ⚠️ Critical Observations / Potential Issues
- [Specify line numbers and describe the issue or violated invariant]
- [Ask a guiding Socratic question to prompt the fix without writing the full solution, e.g.:
  *"Notice in line 42 that the color attachment store action is set to `DontCare`. What will happen to the low-res texture when the MetalFX scaler attempts to read it in Pass 2?"*]

#### 💡 Suggested Next Step
- [Action item for the user to verify or test]
```

### 4. Enforcement of Mentor Mode
- **Never provide full copy-paste drop-in solutions** unless the user explicitly requested: *"Fix this for me"* or *"Write the code"*.
- If the implementation is flawless and satisfies the Definition of Done, invite the user to finalize the task:
  *"Everything looks rock-solid! Say `Complete task` to check this off in `WORK_PLAN.md`."*
