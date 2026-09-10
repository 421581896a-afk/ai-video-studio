# AI Video Studio — Coding Agent Rules

## Mission
Build a production-grade AI video editing SaaS. The target is a real end-to-end pipeline:

**real video → real AI analysis → real Timeline → real editing → real render → real MP4**

## Non-negotiable architecture rules
1. Timeline JSON is the single source of truth for editing.
2. AI must modify Timeline through validated patches; AI must not directly mutate media files.
3. Rendering is deterministic from Timeline + render settings + resolved assets.
4. All external AI providers must be accessed through internal provider interfaces. Business logic must never be coupled to a vendor SDK.
5. Every asynchronous job must be idempotent and observable.
6. Never mark a task complete with a mock/stub when the task requires a real implementation.
7. Preserve backward compatibility of the Timeline schema through explicit `schemaVersion`.
8. Undo/Redo is patch-based, not full-Timeline snapshot copying.
9. Preview rendering and final rendering are separate pipelines.
10. Media originals, proxies, thumbnails and generated outputs must be safely reusable.

## Execution protocol
- Implement tasks strictly in order: TASK-001 → TASK-030.
- A Coding Agent must execute **one TASK per invocation** unless the task explicitly contains substeps.
- Do not pre-implement later tasks.
- Before coding, inspect the repository and existing implementation.
- After coding, run relevant tests, lint/type checks and build checks.
- Fix failures caused by the current task before reporting completion.
- End each task with: files changed, behavior implemented, commands run, test results, known limitations, and next TASK.

## Product priorities
MVP must prioritize the complete media-to-render loop over breadth of UI features.

## Forbidden shortcuts
- No fake render-success responses.
- No fake AI analysis presented as real analysis.
- No hard-coded provider credentials.
- No direct database access from UI components.
- No business logic hidden inside page components.
- No silently destructive Timeline mutations.
- No unvalidated AI-generated JSON entering the Timeline engine.

## Development principle
Prefer small, composable modules, explicit types, deterministic transformations, structured logging, and testable pure functions.
