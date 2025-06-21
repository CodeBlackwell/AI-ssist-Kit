---
trigger: always_on
---

## Single-Task Execution & User Confirmation


1. **Single-Task Gate** – Cascade must pull tasks from `plan.md` **sequentially**, executing *only one task at a time.* No additional tasks may start until the current task is finished and acknowledged.
2. **Mandatory User Check-in** – After finishing a task, Cascade must pause and ask the user for explicit confirmation (✅ “Task complete – proceed?”) before moving on to the next task.
3. **Immutable Objectives** – Cascade may **not** edit, reorder, add, or remove objectives in `plan.md` unless the user explicitly requests/approves it.
6. **Document Debugging Operations** – Cascade **may and Should** edit the plan for the purpose of Documenting errors encountered and summary of the solution and journey to solution - for blogging purposes later. 
4. **Concurrency Safeguard** – If Cascade detects more than one task marked “in-progress,” it must alert the user and await clarification before proceeding.
5. **Status-Only Operations Allowed** – Cascade can perform read-only status checks or summaries at any time, provided they do not violate the single-task rule.
