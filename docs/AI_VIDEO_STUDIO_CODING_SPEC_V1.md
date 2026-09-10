# AI Video Studio — Coding Specification v1.0

## 0. Mission

Build a commercial-grade AI video editing SaaS whose primary loop is:

**real video → real AI analysis → real Timeline → real editing → real render → real MP4**

The product is an AI-first editor. AI does not directly modify media files. AI produces validated Timeline patches; the renderer deterministically turns Timeline state into video.

## 1. Product goal

Core object flow:

`Asset → Script → Storyboard → Timeline → Render`

Production modes:

1. Assets → automatic video.
2. Script + assets → automatic video (primary MVP mode).
3. Script + voice + assets → automatic video.
4. Topic → AI script → AI asset retrieval → AI voice → complete video.

## 2. MVP scope

Users, projects, upload, asset library, FFprobe, proxy/thumbnail generation, ASR, silence removal, shot/scene detection, clip selection, automatic sequence, subtitles, vertical video, zoom, BGM, basic color, transitions, Timeline editing, preview rendering, final rendering, usage tracking and Dockerized local deployment.

## 3. V1 scope

Script-to-video, AI copywriting, TTS, B-roll, SFX, transitions, multiple styles/aspect ratios, natural-language Timeline edits, patch-based Undo/Redo/versioning, storyboard and richer AI Director behavior.

## 4. V2 direction

Multi-agent orchestration, AI B-roll/video generation, multilingual workflows, multi-character voices, translation, lip sync, ad variants, Brand Kit, collaboration, API and batch production.

## 5. Technology stack

- Frontend: Next.js, React, TypeScript, Tailwind CSS, shadcn/ui, Zustand, TanStack Query.
- Backend: Python, FastAPI, Pydantic, SQLAlchemy 2, Alembic.
- Video: Remotion, FFmpeg, FFprobe.
- Infra: PostgreSQL, Redis, Celery, S3-compatible object storage, Docker.
- Production extensions: NVIDIA CUDA/NVENC GPU workers and CDN.

## 6. Monorepo

```text
ai-video-studio/
├── apps/
│   ├── web/
│   └── api/
├── packages/
│   ├── timeline/
│   └── shared/
├── renderer/
├── workers/
│   ├── media/
│   ├── ai/
│   └── render/
├── migrations/
├── tests/
├── docs/
├── docker/
├── docker-compose.yml
├── package.json
├── pnpm-workspace.yaml
├── AGENTS.md
├── README.md
└── .env.example
```

## 7. Frontend routes

Implement `/dashboard`, `/create`, `/editor/[projectId]`, `/projects`, `/assets`, `/exports`, `/settings`.

Editor layout: left tool rail, central Preview, right AI Assistant, bottom multi-track Timeline. Required tracks: Video, B-roll, Voice, Music, SFX, Subtitle.

## 8. Timeline source of truth

```typescript
export interface Timeline {
  schemaVersion: string;
  project: { width: number; height: number; fps: number; duration: number };
  tracks: Track[];
  metadata?: {
    createdBy?: string;
    aiGenerated?: boolean;
    sourceScriptId?: string;
  };
}

export interface Track {
  id: string;
  type: "video" | "broll" | "voice" | "music" | "sfx" | "subtitle";
  locked?: boolean;
  muted?: boolean;
  clips: Clip[];
}

export interface Clip {
  id: string;
  assetId?: string;
  sourceStart?: number;
  sourceEnd?: number;
  start: number;
  duration: number;
  speed?: number;
  volume?: number;
  transform?: { x: number; y: number; scale: number; rotation: number };
  crop?: { type: "none" | "center" | "smart"; x?: number; y?: number; width?: number; height?: number };
  color?: { exposure?: number; contrast?: number; saturation?: number; temperature?: number; highlights?: number; shadows?: number; sharpness?: number };
  transitionIn?: Transition;
  transitionOut?: Transition;
  subtitle?: SubtitleData;
  metadata?: Record<string, unknown>;
}
```

The schema must be versioned and validated at API boundaries and before render.

## 9. Timeline constraints

Validate non-negative start/duration, valid source ranges, supported track types, clip IDs, asset references, transform ranges, transition compatibility and project duration. Reject malformed or unsafe Timeline state with structured errors.

## 10. Timeline Patch

```typescript
export interface TimelinePatch {
  id: string;
  projectId: string;
  baseVersion: number;
  operations: PatchOperation[];
  createdBy: "user" | "ai";
  reason?: string;
  createdAt: string;
}

export interface PatchOperation {
  op: "add" | "remove" | "update" | "move" | "split";
  target: string;
  field?: string;
  value?: unknown;
}
```

Required engine methods: `applyPatch()`, `validatePatch()`, `reversePatch()`, `mergePatches()`.

Undo/Redo must be patch based. Optimistic concurrency must reject a patch whose `baseVersion` is stale unless an explicit safe merge is possible.

## 11. Database

Core tables: `users`, `projects`, `assets`, `asset_analysis`, `scripts`, `timelines`, `timeline_versions`, `ai_jobs`, `render_jobs`, `usage_records`.

Use UUIDs, timestamps, indexes for project ownership and job state, JSONB for extensible analysis/settings, foreign keys and transactional version increments.

## 12. API

Project CRUD; asset upload/init, upload/complete, get and delete; analysis status; script generate/import/rewrite; AI create/revise/edit; Timeline get/patch/undo/redo/versions; render create/status/cancel.

WebSocket: `/ws/projects/{project_id}`.

Events: analysis progress, AI job progress, Timeline changes, render progress and errors.

## 13. Upload pipeline

Browser → FastAPI presigned URL → S3-compatible object storage → complete → async processing.

On completion enqueue probe, proxy, thumbnail and analysis work. Do not proxy media through the API process.

## 14. Media intelligence pipeline

Original → FFprobe → Proxy → shot/scene detection → frame sampling → ASR → OCR → object/face detection → quality analysis → audio analysis → embeddings.

Persist analysis artifacts so repeat edits do not repeat expensive analysis.

## 15. Clip ranking

Use a deterministic configurable score:

`0.30 SemanticRelevance + 0.20 VisualQuality + 0.15 NarrativeRelevance + 0.10 FaceQuality + 0.10 MotionQuality + 0.05 AudioQuality + 0.05 Novelty + 0.05 BrandMatch - penalties + rhythm bonus`

Store component scores and reason codes for explainability.

## 16. Automatic editing

Given script/storyboard and analyzed assets, select clips, trim source ranges, arrange sequence, preserve narrative continuity, avoid excessive repetition, and fit the requested aspect ratio.

Silence removal default gap threshold: `0.45s`, configurable from `0.2–1.0s`.

## 17. Subtitles

Generate from ASR timestamps. Support line breaking, safe areas, font/size/weight, alignment, highlighted words where provider data supports it, and style presets. Subtitle edits must remain Timeline data.

## 18. Music

BGM ranking:

`0.30 Mood + 0.25 Energy + 0.20 BPM + 0.15 Genre + 0.10 Duration`

Support beat-aware cuts, fade in/out and ducking. Default voice-over ducking target is approximately `-12 to -18 dB`, configurable.

## 19. Color

Per-clip controls: exposure, white balance/temperature, contrast, highlights, shadows, saturation and sharpness.

Presets: Clean, Cinematic, Warm, Cold, Commercial, Luxury, Film, Vintage, Bright, Dark.

## 20. Transitions

Support CUT, FADE, DISSOLVE, ZOOM, WHIP and SLIDE. Choose transitions using shot change, motion, beat and emotion. Avoid transition overuse.

## 21. AI architecture

Director Agent orchestrates specialist capabilities:

- Script Agent
- Storyboard Agent
- Asset Agent
- Editing Agent
- Subtitle Agent
- Music Agent
- Voice Agent
- Color Agent
- QC Agent

The Director should produce structured intermediate artifacts and a final Timeline/Patch plan. Validate every model response against strict schemas.

## 22. Provider abstraction

Create internal interfaces for LLM, ASR, TTS, Vision and Embedding. Provider implementations must be swappable. Business code cannot import vendor SDKs directly. Configuration selects providers through environment variables.

Required concepts:

```text
LLMProvider.generate()
ASRProvider.transcribe()
TTSProvider.synthesize()
VisionProvider.analyze()
EmbeddingProvider.embed()
```

Include retry policy, timeout, structured error mapping, usage accounting and result caching.

## 23. Natural-language editing

User commands such as “字幕大一点”, “把开头节奏快一点”, “换一个更高级的 BGM” must be translated into a constrained edit plan and Timeline Patch. The agent must inspect current Timeline state before proposing changes.

Never allow natural-language output to bypass patch validation.

## 24. Preview

Preview render is optimized for latency and editor feedback. It may use lower resolution/bitrate and proxy media. It must represent the same Timeline semantics as final rendering.

## 25. Final renderer

Pipeline:

`Timeline JSON → validate → resolve assets/audio/subtitles → compile → Remotion → FFmpeg → MP4`

Default final target: 1080p H.264 MP4. Support 16:9, 9:16 and 1:1 project configurations.

Renderer must be deterministic, report progress, preserve audio sync, and fail with actionable diagnostics.

## 26. Render compiler

Build a compiler that converts Timeline clips into Remotion composition inputs. Resolve source ranges, speed, transforms, crops, color, transitions, subtitle overlays and audio mixing in a stable order.

Do not duplicate editing rules independently in frontend and renderer.

## 27. Workers

Task types:

`UPLOAD_PROCESS, VIDEO_PROBE, PROXY_GENERATE, THUMBNAIL_GENERATE, ASR, SCENE_DETECT, SHOT_DETECT, VISION_ANALYSIS, EMBEDDING, SCRIPT_GENERATE, STORYBOARD_GENERATE, CLIP_SELECTION, TIMELINE_GENERATE, SUBTITLE_GENERATE, MUSIC_MATCH, COLOR_ANALYSIS, QC, PREVIEW_RENDER, FINAL_RENDER`

Use Celery + Redis for asynchronous execution. Workers must be retryable, idempotent and observable.

DAG:

`Upload → Probe → (ASR/Shot/Vision) → Asset Intelligence → Script → Storyboard → Clip Selection → Timeline → (Subtitle/Music/Color) → QC → Render`

## 28. Idempotency and caching

Render idempotency key: project + Timeline version + render settings hash.

Cache AI results, analysis, embeddings, proxies and render outputs whenever inputs are unchanged. Store provider usage for each expensive call.

## 29. Quality control

QC must check missing assets, invalid source ranges, duration mismatches, subtitle overflow, audio clipping, silent output, unexpected black frames, invalid dimensions/fps and render output existence.

A failed QC must block final export unless explicitly overridden by a privileged workflow.

## 30. Usage and cost tracking

Record LLM tokens, ASR seconds, TTS characters/seconds, Vision calls, Embedding calls, GPU seconds, render seconds and storage GB. Associate usage with user/project/job and provider/model where applicable.

## 31. Security

Validate uploads, MIME/type and size; never execute user-provided filenames as commands; sanitize paths; use signed storage URLs; isolate FFmpeg arguments from shell injection; enforce project authorization; protect WebSocket subscriptions; redact secrets and sensitive provider payloads from logs.

## 32. Configuration

Provide `.env.example` for database, Redis, object storage, AI provider selection/keys, render configuration and application URLs. Secrets must never be committed.

## 33. Docker

Local stack must provide Web, API, PostgreSQL and Redis. Production design should permit separate worker/media/render services and optional GPU worker images.

## 34. Health/observability

API `/health` returns `{"status":"ok"}` when dependencies are healthy enough for the configured readiness semantics. Add structured logs, correlation/job IDs and clear worker failure states.

## 35. Testing

Unit tests for Timeline validation, patch engine, clip ranking, silence removal, music ranking and render compiler. Integration tests for API/database/jobs. E2E test for the core upload-to-render journey.

The acceptance test must cover:

`30s MP4 upload → proxy/thumbnail → ASR → ≥5 shots → AI selection → Timeline → frontend edit → subtitles/BGM/color/transitions → preview → final 1080x1920 H.264 MP4 → natural-language “字幕大一点” Patch → Undo/Redo/versioning/QC/usage tracking → Docker/E2E`.

## 36. TASK-001 → TASK-030

### TASK-001 — Monorepo bootstrap
Initialize pnpm workspace, Next.js web, FastAPI API, shared packages, Docker Compose, PostgreSQL, Redis, environment templates and basic tests. Implement `/health` returning `{"status":"ok"}`.

### TASK-002 — Shared contracts
Create shared TypeScript/Python contract conventions, API error format, IDs, timestamps and initial project DTOs.

### TASK-003 — Timeline package
Implement Timeline types, schema validation, serialization and compatibility tests.

### TASK-004 — Patch engine
Implement patch operations, validation, reverse, merge, version increments and tests.

### TASK-005 — Project backend
Implement project CRUD, persistence, authorization and API tests.

### TASK-006 — Editor shell
Build editor layout, routing, project loading and state management.

### TASK-007 — Timeline UI
Implement tracks, clips, selection, move/trim/split, snapping, zoom and timeline state synchronization.

### TASK-008 — Preview player
Build Timeline-aware preview player with playhead, seek and selection synchronization.

### TASK-009 — Asset system
Implement upload initialization/completion, object storage, asset metadata, library UI and deletion.

### TASK-010 — Media processing
Implement FFprobe, proxy and thumbnail workers with persistent job states.

### TASK-011 — Shot/scene detection
Implement shot boundaries, scene grouping and visual samples.

### TASK-012 — ASR
Implement provider interface and production ASR pipeline with timestamp persistence.

### TASK-013 — Asset intelligence
Add OCR/vision/quality/audio analysis and embeddings behind provider interfaces.

### TASK-014 — Script model
Implement script import, generation and rewrite APIs plus persisted structured script sections.

### TASK-015 — Storyboard
Convert scripts into scenes/shots with semantic requirements and target durations.

### TASK-016 — Clip ranking
Implement explainable ranking and candidate selection from asset intelligence.

### TASK-017 — Automatic Timeline generation
Generate a valid Timeline from script/storyboard/candidates and validate it before persistence.

### TASK-018 — Silence removal
Use ASR/audio timing to create safe silence-removal patches with configurable thresholds.

### TASK-019 — Subtitle engine
Generate styled subtitle Timeline clips and renderer support.

### TASK-020 — Music matching
Implement music metadata, ranking, ducking and beat-aware placement.

### TASK-021 — Color and transitions
Implement automatic color recommendations, presets and transition selection.

### TASK-022 — Render compiler
Compile Timeline to Remotion composition data with deterministic asset resolution.

### TASK-023 — Preview render
Implement asynchronous preview rendering, progress events and cached outputs.

### TASK-024 — Final render
Implement production final rendering to 1080p H.264 MP4 and render job lifecycle.

### TASK-025 — QC
Implement pre-render and post-render quality checks and export gating.

### TASK-026 — AI provider layer
Finalize LLM/ASR/TTS/Vision/Embedding provider adapters, retries, caching and usage accounting.

### TASK-027 — Director + specialist agents
Implement orchestration, structured intermediate artifacts and agent execution state.

### TASK-028 — Natural-language edit
Translate user commands into constrained edit plans and Timeline patches with explainable results.

### TASK-029 — Undo/Redo/versioning
Complete durable Timeline versions, patch history, undo/redo APIs and UI history.

### TASK-030 — Production hardening
Complete security review, idempotency, observability, cost tracking, Docker/E2E, failure recovery and MVP acceptance test.

## 37. Coding Agent launch prompt

Paste the following into a Coding Agent:

> Read `AGENTS.md` and `docs/AI_VIDEO_STUDIO_CODING_SPEC_V1.md` completely. Inspect the repository. Execute **TASK-001 only**. Do not implement TASK-002 or any later task. Initialize the monorepo with Next.js/React/TypeScript web, FastAPI/Python API, PostgreSQL, Redis, pnpm workspace, Docker Compose, shared package scaffolding, `.env.example`, tests and documentation. Implement `/health` returning `{"status":"ok"}`. Run the relevant tests, type/lint/build checks and Docker smoke checks. Fix failures caused by TASK-001. Then output a completion report listing changed files, commands run, test results, known limitations and the exact next task: TASK-002.

For every subsequent invocation:

> Read `AGENTS.md` and the canonical specification. Execute **TASK-NNN only**. Do not implement future tasks. Inspect existing code first. Reuse existing abstractions. Implement production behavior, not placeholders. Run relevant tests and quality checks, fix failures, and provide the standard completion report.

## 38. Definition of Done

A task is done only when:

- Acceptance behavior is implemented.
- Relevant tests exist and pass.
- Types/lint/build checks pass where applicable.
- Errors are handled explicitly.
- No mock is presented as production functionality.
- Security-sensitive paths are validated.
- Documentation is updated where behavior or architecture changed.
- The completion report identifies limitations honestly.

## 39. Final architecture

```text
                        ┌─────────────────────┐
                        │      Next.js UI     │
                        │ Editor / Timeline   │
                        └──────────┬──────────┘
                                   │ API / WS
                        ┌──────────▼──────────┐
                        │     FastAPI API     │
                        │ Auth / Projects     │
                        │ Timeline / AI / Job │
                        └───────┬─────┬───────┘
                                │     │
                   ┌────────────▼┐   ┌▼─────────────┐
                   │ PostgreSQL   │   │ Redis/Celery │
                   │ source state │   │ async jobs   │
                   └─────────────┘   └──────┬───────┘
                                            │
                  ┌─────────────────────────┼─────────────────────────┐
                  │                         │                         │
           ┌──────▼──────┐          ┌───────▼──────┐          ┌──────▼──────┐
           │ Media Worker │          │  AI Workers  │          │Render Worker│
           │ FFmpeg/proxy │          │ providers    │          │Remotion/FF │
           └──────┬──────┘          └───────┬──────┘          └──────┬──────┘
                  │                         │                         │
                  └─────────────────────────┼─────────────────────────┘
                                            │
                                   ┌────────▼────────┐
                                   │ S3 Object Store │
                                   │ originals/proxy │
                                   │ outputs         │
                                   └─────────────────┘
```

The single most important invariant is: **Timeline is the editing truth; AI proposes validated Timeline changes; Renderer turns that truth into the actual video.**
