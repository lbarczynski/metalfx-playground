---
name: start-task
description: >-
  Use this skill when the user wants to start, resume, or kick off a learning task in the MetalFX curriculum
  (e.g., "start next task", "let's do task 3", "what's next?", "start task 01").
---

# Start Task Skill: Daily Session Kickoff

This skill orchestrates a friction-free start for a daily 30–60 minute learning session.

## Workflow Instructions

### 1. Identify the Target Task
- Read `WORK_PLAN.md` in the project root.
- If the user specified a task (e.g. "Task 4"), locate that task.
- Otherwise, locate the **first unchecked task** (`- [ ] TASK-XX`).
- Extract the task ID, title, estimated time, and the relative path to its guide in `materials/`.

### 2. Ingest the Task Guide
- Read the corresponding guide file (e.g. `materials/03-hello-triangle-pipeline.md`).
- Extract the key elements:
  1. **Mental Model & Context**: Why this step matters and how it bridges from Kotlin/Compose or OpenGL/CUDA.
  2. **Core Math / Layout Primitives**: The exact formula, matrix, or struct alignment needed.
  3. **Implementation Step 1**: The first concrete action to take.

### 3. Deliver the Kickoff Briefing
Present a concise, structured briefing formatted as follows:

```markdown
### 🚀 Today's Task: TASK-XX — [Title]
- **Estimated Duration**: [XX] min
- **Primary Goal**: [1-sentence goal / Definition of Done]
- **Guide**: [materials/XX-*.md](materials/XX-*.md)

#### 🧠 Key Mental Model & Math
[1-2 sentences on the core concept and the exact mathematical formula or layout rule to remember]

#### 🛠️ Immediate Next Step
[Concrete instruction for Step 1, e.g.: "Open `src/Renderer.cpp` and define the `Vertex` struct with `alignas(16)` as outlined in Section 3.1."]
```

### 4. Socratic Check-In
- Ask the user if they want a quick clarification on any formula/concept before they write code, or if they are ready to dive in.
- Remind them: *"Whenever you're ready for feedback, just say `Review my change`!"*
- Remember: **Do not generate or write the task solution code for the user.**
