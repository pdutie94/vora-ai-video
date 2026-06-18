# Vora AI — Project Instructions

## Tech Stack

- **Frontend**: Next.js 16, React 19, TypeScript 6, TailwindCSS 4, shadcn/ui, TanStack Query 5
- **Backend**: NestJS 11, TypeScript, Prisma 7, MySQL 8, Redis, BullMQ 5
- **Video**: Remotion 4 (server-side rendering via `@remotion/renderer`)
- **Auth**: NestJS-native JWT + bcrypt-hashed refresh tokens
- **Real-time**: WebSocket (`@nestjs/platform-socket.io`, `socket.io-client`)
- **Storage**: Local filesystem via `StorageProvider` interface (MVP), S3-compatible later
- **AI**: Provider Adapter pattern (`ScriptProvider`, `VoiceProvider`, `ImageProvider`, `VideoProvider`)
- **Process Manager**: PM2 (no Docker, no K8s)
- **Package Manager**: pnpm workspaces (no Turbo)

## Architecture Rules

### Monorepo Structure
```
vora-ai/
├── apps/
│   ├── web/          # Next.js 16
│   ├── api/          # NestJS 11 (HTTP + Worker)
│   └── renderer/     # Remotion 4 (separate PM2 process)
└── packages/
    └── shared/       # Types, Zod schemas, constants
```

### PM2 Processes
- `vora-api` — NestJS HTTP server (port 4000)
- `vora-worker` — BullMQ consumer (AI, TTS, subtitle)
- `vora-renderer` — Remotion renderer (separate process, CPU-heavy)
- `vora-web` — Next.js (port 3000)

### Authentication
- **NO** Better Auth, **NO** third-party auth SDK
- NestJS-native JWT: `@nestjs/jwt` + `@nestjs/passport`
- Access token: 15min expiry
- Refresh token: 30 day expiry, stored as **bcrypt hash** in `User.refreshTokenHash`
- HttpOnly cookie for refresh token, Authorization header for JWT
- Endpoints: `POST /api/auth/register`, `POST /api/auth/login`, `POST /api/auth/refresh`, `POST /api/auth/logout`, `GET /api/auth/me`

### Database (MySQL + Prisma 7)
- Models: `User`, `Project`, `ProjectAsset`, `Job`, `Template`
- **NO** `Session`/`Account` tables (auth is JWT-based)
- **NO** `CreditTransaction`/`SubscriptionPlan` (billing removed from MVP)
- `ProjectStatus` enum: `PENDING`, `PROCESSING`, `COMPLETED`, `FAILED`
- `Job.metadata` JSON column stores accumulated pipeline data
- Soft delete on `Project.deletedAt`

### Queue Design (BullMQ)
- **2 queues only**: `generation` + `render`
- Named jobs (no `switch(step)`): `analyze`, `script`, `voice`, `subtitle`, `render`, `cleanup`
- Job data carries **only** `{ projectId, jobId }` — all state in DB (`Job.metadata`)
- If Redis crashes mid-pipeline, no state is lost
- Retry: `generation` 3x (5s→30s→120s), `render` 2x (30s→300s)

### WebSocket (NOT SSE)
- Use `@nestjs/platform-socket.io` on both server and client
- Single namespace: `/ws`
- Client subscribes to room `project:{projectId}`
- Events: `progress`, `completed`, `failed`
- BullMQ workers emit events via `GenerationGateway`

### StorageProvider — Strict Interface
- Business code **NEVER** calls `fs.writeFile`, `fs.readFile` directly
- All file operations through `StorageProvider` interface
- MVP: `LocalStorageProvider`, Future: `S3StorageProvider`
- One-line swap in production via env var

### Provider Adapters
- `ScriptProvider` — analyze, angle, script generation (MVP: OpenAI/Gemini)
- `VoiceProvider` — TTS (MVP: OpenAI TTS)
- `ImageProvider` — image generation (future)
- `VideoProvider` — video generation (Remotion)
- Swap provider via `.env`: `SCRIPT_PROVIDER=openai`, `VOICE_PROVIDER=openai`

### Remotion (Separate Process)
- Lives in its own `apps/renderer/` with its own `package.json`
- Only imports Remotion + shared types — **not** part of worker
- When Remotion crashes (OOM, FFmpeg), only renderer restarts
- Connects to same Redis (BullMQ) + MySQL (Prisma)

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
- SSE rejected in favor of WebSocket (multi-tab, future admin panel)
- No Turbo — pnpm `--filter` is sufficient for solo dev
- Flat NestJS modules (no Clean Architecture layers until needed)
