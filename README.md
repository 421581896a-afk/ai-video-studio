# AI Video Studio

Production-grade AI video editing SaaS.

## Product principle

**AI edits the Timeline; the renderer produces the video.**

Core pipeline:

`Asset → Script → Storyboard → Timeline → Render`

Target:

`real video → real AI analysis → real Timeline → real editing → real render → real MP4`

## Technology direction

- Web: Next.js + React + TypeScript + Tailwind + shadcn/ui + Zustand + TanStack Query
- API: Python + FastAPI + Pydantic + SQLAlchemy 2 + Alembic
- Video: Remotion + FFmpeg + FFprobe
- Infrastructure: PostgreSQL + Redis + Celery + S3-compatible object storage
- Production: Docker, GPU/NVENC workers and CDN where appropriate

## Repository structure

```text
apps/web       Web application
apps/api       FastAPI backend
packages/      Shared Timeline and types
renderer/      Render compiler and Remotion compositions
workers/       Media, AI and render workers
docs/          Engineering specifications
migrations/    Database migrations
tests/         Unit/integration/E2E tests
docker/        Container configuration
```

## Coding Agent workflow

Read `AGENTS.md` first. Implement `TASK-001` through `TASK-030` strictly in order, one task at a time. Do not implement future tasks early. Every task must include tests and a completion report.

## Specification

The canonical implementation specification is:

`docs/AI_VIDEO_STUDIO_CODING_SPEC_V1.md`

## First task

Start with TASK-001: initialize the monorepo and local development infrastructure, including Next.js, FastAPI, PostgreSQL, Redis and Docker. The API health endpoint must return `{"status":"ok"}`. Verify the stack and tests before proceeding to TASK-002.
