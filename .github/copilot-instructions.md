# Vora AI — Project Instructions

## Tech Stack

- **Frontend**: Next.js 16, React 19, TypeScript 6, TailwindCSS 4, shadcn/ui, TanStack Query 5
- **Backend**: NestJS 11, TypeScript, Prisma 7, MySQL 8, Redis, BullMQ 5
- **Video**: Remotion 4 (server-side rendering via `@remotion/renderer`, runs inline in worker)
- **Auth**: NestJS-native JWT (7-day expiry, no refresh token)
- **Real-time**: REST polling (`GET /api/projects/:id` every 2s)
- **Storage**: Local filesystem via `StorageProvider` interface (MVP), S3-compatible later
- **AI**: Provider Adapter pattern (`ScriptProvider`, `VoiceProvider`)
- **Process Manager**: PM2 (no Docker, no K8s)
- **Package Manager**: pnpm workspaces (no Turbo)

## Architecture Rules

### Monorepo Structure
```
vora-ai/
├── apps/
│   ├── web/          # Next.js 16
│   └── api/          # NestJS 11 (HTTP + BullMQ workers + Remotion)
└── packages/
    └── shared/       # Types, constants
```

### PM2 Processes
- `vora-api` — NestJS (HTTP + workers + Remotion rendering) — port 4000
- `vora-web` — Next.js (port 3000)

### Authentication
- **NO** Better Auth, **NO** third-party auth SDK
- NestJS-native JWT: `@nestjs/jwt` + `@nestjs/passport`
- JWT: 7-day expiry, no refresh token
- Authorization header: `Bearer <token>`
- Endpoints: `POST /api/auth/register`, `POST /api/auth/login`, `GET /api/auth/me`

### Database (MySQL + Prisma 7)
- Models: `User`, `Project`, `ProjectAsset`, `Job`
- **NO** `Session`/`Account` tables (auth is JWT-based)
- **NO** `Template` table (1 template hardcoded in code)
- **NO** `CreditTransaction`/`SubscriptionPlan` (billing removed from MVP)
- `ProjectStatus` enum: `PENDING`, `PROCESSING`, `COMPLETED`, `FAILED`
- `Job.metadata` JSON column stores accumulated pipeline data
- Hard delete on Project (no `deletedAt`)

### Queue Design (BullMQ)
- **1 queue only**: `video-pipeline`
- Named jobs (no `switch(step)`): `analyze`, `script`, `voice`, `subtitle`, `render`, `cleanup`
- Job data carries **only** `{ projectId, jobId }` — all state in DB (`Job.metadata`)
- If Redis crashes mid-pipeline, no state is lost
- Retry: all jobs 3x (5s→30s→120s)

### Progress Tracking (REST Polling)
- No WebSocket, no SSE
- Frontend polls `GET /api/projects/:id` every 2 seconds during generation
- TanStack Query `refetchInterval` handles auto-polling
- Simpler: no socket connections, no reconnection logic, no rooms

### StorageProvider — Strict Interface
- Business code **NEVER** calls `fs.writeFile`, `fs.readFile` directly
- All file operations through `StorageProvider` interface
- MVP: `LocalStorageProvider`, Future: `S3StorageProvider`
- One-line swap in production via env var

### Provider Adapters
- `ScriptProvider` — analyze, angle, script generation (MVP: OpenAI/Gemini/Claude)
- `VoiceProvider` — TTS (MVP: OpenAI TTS)
- `ImageProvider`/`VideoProvider` removed — add only when needed
- Swap provider via `.env`: `SCRIPT_PROVIDER=openai`, `VOICE_PROVIDER=openai`

### Remotion (Inline in Worker)
- Runs inside the same NestJS process (`apps/api`)
- `renderMedia()` called directly from job handler
- When Remotion crashes → PM2 restarts the API process
- For 14 renders/hr (10k videos/month), single process handles it easily

## Development Commands
```bash
pnpm dev              # Run Next.js + NestJS in dev mode
pnpm worker           # Run BullMQ worker only
pnpm build            # Build all packages
pnpm --filter @vora/api exec prisma generate
pnpm --filter @vora/api exec prisma db push
```

## Key Design Decisions
- No Docker, no K8s — native PM2 on Ubuntu VPS
- Local storage before S3
- No billing/Stripe in MVP (admin grants credits manually via DB)
- REST polling over WebSocket/SSE — simpler, no socket infrastructure
- No Turbo — pnpm `--filter` is sufficient for solo dev
- Flat NestJS modules (no Clean Architecture layers until needed)
- Remotion runs inline — no separate renderer process needed for MVP
- 1 BullMQ queue — not 2, not 7. Named jobs handle routing
