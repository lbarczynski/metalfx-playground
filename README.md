# MetalFX & Modern C++ Learning Playground

A structured, 22-day engineering curriculum designed to build deep mastery in **Modern C++ (C++20)**, **Apple Metal**, and **MetalFX** upscaling on Apple Silicon.

---

## 🛠️ Infrastructure & Tooling

- **Language Standard**: C++20 (`-std=c++20`, strict flags `-Wall -Wextra -Wpedantic -Wconversion`)
- **Build System**: CMake 3.25+ with `CMakePresets.json` & **Ninja** generator
- **Formatting & Linting**: `.clang-format` (Google/LLVM style, 4-space indent) & `.clang-tidy`
- **Testing**: Catch2 v3 test runner via CTest
- **Package Management**: Microsoft `vcpkg.json` (manifest mode) + automated Apple `metal-cpp` CDN ingestion
- **CI/CD**:
  - GitHub Actions (`.github/workflows/ci.yml`) on `macos-14` (Apple Silicon M1 runners)
  - GitLab CI (`.gitlab-ci.yml`) for self-hosted macOS runners

---

## 🚀 Quick Start

### 1. Enable Git Pre-Commit Hook
Enforce automated code formatting before commits:
```bash
git config core.hooksPath .githooks
```

### 2. Configure & Build with Presets
```bash
# Debug build (with ASan & UBSan enabled)
cmake --preset macos-debug
cmake --build --preset macos-debug-build

# Release build (optimized)
cmake --preset macos-release
cmake --build --preset macos-release-build

# Run tests
ctest --preset macos-debug-test --output-on-failure
```

---

## 🤖 AI Mentor Mode & Interactive Workflow

The repository is pre-configured with a dedicated Socratic mentor agent and 4 workflow skills:

| Command / Prompt | Action |
|---|---|
| `Start next task` / `Start task 1` | Kicks off the next task, sets up the mental model, and lists step-by-step guidance. |
| `Review my change` / `Check my code` | Mentorship review of your changes against DoD, alignment rules, and Metal invariants. |
| `Debug Metal` / `Validation error` | Analyzes Metal validation errors, visual glitches, or pipeline crashes. |
| `Complete task` | Marks task complete in `WORK_PLAN.md` and generates commit summary. |

---

## 📚 Curriculum Plan

See [WORK_PLAN.md](./WORK_PLAN.md) for the complete 22-day curriculum and links to all technical guides in [materials/](./materials/).
