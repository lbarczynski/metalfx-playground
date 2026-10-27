---
name: complete-task
description: >-
  Use this skill when the user finishes a task, verifies that its Definition of Done is met,
  and wants to mark it complete, commit changes, or wrap up the session (e.g., "complete task", "finish task 3", "mark task done", "ready to commit").
---

# Complete Task Skill: Milestone Wrap-Up & Progress Tracker

This skill formalizes task completion, updates the master curriculum tracker in `WORK_PLAN.md`, generates git commit metadata, and prepares the developer for the next day's task.

## Workflow Instructions

### 1. Verify Definition of Done
- Determine which task was completed (e.g. `TASK-03`).
- Re-read its **Definition of Done** in `WORK_PLAN.md`.
- Confirm with the user (or via build/run checks) that the executable compiles and produces the expected output.

### 2. Update Progress in `WORK_PLAN.md`
- Edit `WORK_PLAN.md`: change the corresponding checkbox from `- [ ]` to `- [x]`.
- Example:
  ```markdown
  - [x] **[TASK-03: The Hello Triangle & Metal Shading Language](./materials/03-hello-triangle-pipeline.md)** *(60 min)*
  ```

### 3. Generate Conventional Commit Metadata
Generate a clean, standardized git commit command for the user:
```bash
git add .
git commit -m "feat(task-XX): <short summary of what was implemented>"
```

### 4. Provide Session Summary & Next Day Preview
Deliver a congratulatory wrap-up formatted as follows:

```markdown
### 🎉 TASK-XX Complete!

#### 🏆 What You Mastered Today
- **C++20**: [1 bullet summarizing C++ concept practiced, e.g. "Configured type-safe vertex descriptors using `offsetof` and memory alignment."]
- **Metal / GPU**: [1 bullet summarizing Metal concept mastered, e.g. "Compiled immutable `MTL::RenderPipelineState` and authored custom MSL shaders with `[[stage_in]]`."]

#### 📝 Suggested Git Commit
```bash
git add .
git commit -m "feat(task-XX): implement <feature summary>"
```

---

#### ⏭️ Tomorrow's Preview: TASK-YY — [Title]
- **Focus**: [1 sentence on tomorrow's topic, e.g. "Moving from static geometry to rotating dynamic uniforms using triple buffering."]
- **Guide**: [materials/YY-*.md](materials/YY-*.md)

*Great job today! When you are ready for the next session, just say `Start next task`!*
```
