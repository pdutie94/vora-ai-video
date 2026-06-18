# Plan: Vora AI — Complete Technical Specification & Architecture Document

## Over-Engineering Risk Assessment (Pre-Design)

### Risks Identified in Naive Architecture

| Risk | Description | Mitigation in This Design |
|------|-------------|--------------------------|
| **Too many packages** | Splitting into 7+ packages before first deploy adds complexity with zero proven benefit | Use 3 packages max — flat is better than nested |
| **Docker requirement** | Enforced Docker for local dev adds friction for solo developer on WSL | Native PM2 processes — exactly like production |
| **Cloud storage dependency** | R2/S3 ties you to cloud before you have a single customer | Local filesystem for MVP — swap to S3 later with adapter |
| **Too many queues** | 7 dedicated queues = 7 workers = 7 process entries to manage | Merge into 2 logical queues with switch-based handlers |
| **PostgreSQL** | User explicitly specified MySQL (already installed) | Use MySQL as specified |
| **Turbo build orchestration** | Adds complexity for a solo dev — caching, remote caching, pipeline deps | pnpm workspaces only — `--filter` is enough |
| **Clean Architecture layers** | Use-case → Repository → Controller layers before proving product-market fit | Flat NestJS modules — only decouple when repetition proves need |
| **Third-party auth SDK** | Better Auth adds an extra sync layer between Next.js and NestJS | NestJS-native JWT + refresh token — API stays self-contained |
| **Presigned URLs** | Client-upload-to-cloud pattern is over-engineering for local-only MVP | Server-side file handling — simple, direct, no cloud dependency |
| **Stripe integration sprint** | Payment integration before video generation works is premature | Removed entirely from MVP. Admin grants credits manually via DB. |

### Key Simplification Principle
> *"You aren't going to need it"* (YAGNI) — every abstraction, package, queue, and microservice has a concrete cost in maintenance time. Only introduce when the code literally forces you to.

---

## 1. Executive Summary

**Vora AI** converts product information into short-form 9:16 marketing videos. Users provide a product name, description, and images. The platform generates a marketing angle, video script, voice-over, subtitles, and a rendered MP4 video.

### Architecture at a Glance

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  Next.js 16 │────▶│  NestJS API  │────▶│   MySQL     │
│  (Frontend) │     │  (Backend)   │     │  (Prisma)   │
│             │     │  JWT Auth    │     │             │
└─────────────┘     └──────┬───────┘     └─────────────┘
                           │                    ▲
                           ▼                    │
                     ┌──────────┐         ┌──────────┐
                     │   Redis  │◀────────│  BullMQ  │
                     │  (Cache  │         │  Workers │
                     │   +Q)    │         └──────────┘
                     └──────────┘               │
                           │                    │
                           ▼                    ▼
                     ┌──────────┐      ┌──────────────┐
                     │WebSocket │      │   Remotion   │
                     │ Gateway  │      │   Renderer   │
                     └──────────┘      └──────┬───────┘
                                                ▼
                                          ┌──────────────┐
                                          │Local Storage │
                                          │  (MVP) → S3  │
                                          └──────────────┘
```

### Key Numbers (Scale Target)
- **100 paying customers** — initial design target
- **10,000 videos/month** — max throughput at target
- **Single VPS** — $10-40/mo Hetzner or similar

### MVP Constraints
- Local filesystem storage (no cloud storage dependency)
- PM2 process management (no Docker, no K8s)
- MySQL (already installed)
- Redis (already installed)
- 2 BullMQ queues (not 7)

### Package Versions (Latest as of June 2026)

| Package | Version | Notes |
|---------|---------|-------|
| **Frontend** | | |
| `next` | 16.2.9 | App Router, React Server Components |
| `react` / `react-dom` | 19.2.7 | Latest stable |
| `typescript` | 6.0.3 | Latest stable |
| `tailwindcss` | 4.3.1 | v4 with CSS-first config |
| `@tanstack/react-query` | 5.101.0 | Server state management |
| `zod` | 4.4.3 | Runtime validation |
| `socket.io-client` | 4.8.3 | WebSocket client |
| **Backend (NestJS)** | | |
| `@nestjs/core` / `@nestjs/common` | 11.1.27 | NestJS 11 |
| `@nestjs/jwt` | 11.0.2 | JWT auth |
| `@nestjs/passport` | 11.0.5 | Auth guards |
| `@nestjs/config` | 4.0.4 | Environment config |
| `@nestjs/platform-socket.io` | 11.1.27 | WebSocket server |
| `@nestjs/swagger` | 11.4.4 | API docs |
| `@nestjs/serve-static` | 5.0.5 | Static file serving |
| **Database** | | |
| `prisma` / `@prisma/client` | 7.8.0 | ORM |
| `mysql2` | 3.22.5 | MySQL driver |
| **Queue** | | |
| `bullmq` | 5.79.0 | Redis-backed job queues |
| `ioredis` | 5.11.1 | Redis client |
| `@bull-board/nestjs` | 8.0.1 | Queue admin dashboard |
| **Video** | | |
| `remotion` / `@remotion/renderer` | 4.0.481 | Server-side video rendering |
| **AI** | | |
| `openai` | 6.44.0 | OpenAI API client |
| **Security** | | |
| `bcryptjs` | 3.0.3 | Password hashing |
| `helmet` | 8.2.0 | HTTP security headers |
| `class-validator` / `class-transformer` | 0.15.1 / 0.5.1 | DTO validation |
| **Dev Tools** | | |
| `pnpm` | 11.8.0 | Package manager |
| `node` | 26.3.0 | Runtime |
| `pm2` | 7.0.1 | Process manager |
| `eslint` | 10.5.0 | Linting |
| `prettier` | 3.8.4 | Formatting |

---

## 2. System Architecture

### Monorepo Structure (Flat by Design)

```
vora-ai/
├── apps/
│   ├── web/                  # Next.js 16 + shadcn/ui
│   │   ├── src/
│   │   │   ├── app/          # App Router pages
│   │   │   ├── components/   # UI components
│   │   │   └── lib/          # API client, utils
│   │   ├── package.json
│   │   └── next.config.ts
│   │
│   └── api/                  # NestJS backend
│   │   ├── src/
│   │   │   ├── modules/      # Feature modules
│   │   │   ├── common/       # Shared decorators, guards, filters
│   │   │   ├── workers/      # BullMQ worker processors
│   │   │   ├── admin/        # Bull Board dashboard
│   │   │   └── main.ts       # Entry point (HTTP + Worker)
│   │   ├── prisma/
│   │   │   └── schema.prisma
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   └── renderer/             # Separate Remotion process
│       ├── src/
│       │   ├── index.ts      # Bootstrap BullMQ consumer
│       │   ├── templates/    # Remotion compositions
│       │   └── scenes/       # Reusable scene components
│       ├── package.json
│       └── tsconfig.json
│       ├── prisma/
│       │   └── schema.prisma
│       ├── package.json
│       └── tsconfig.json
│
└── packages/
    └── shared/                # Types, constants, validation schemas
        ├── src/
        │   ├── types/        # Domain types (Project, Job, Template)
        │   ├── constants/    # Enums, statuses, configuration
        │   └── validation/   # Zod schemas
        ├── package.json
        └── tsconfig.json
```

**Why NOT more packages?** Each npm workspace package requires its own `tsconfig.json`, build step, and dependency resolution. For a solo dev, the cost of managing 7+ packages exceeds the benefit before 10k lines of code. Consolidate into 3 packages. Split later if code volume justifies it.

### Authentication Architecture

Auth lives entirely inside NestJS — no external auth SDK:

```
NestJS
├── JWT (access token, 15min expiry)
├── Refresh Token (30 day expiry, hashed in DB)
└── HttpOnly cookie (refresh token) + Authorization header (JWT)

Next.js reads JWT from cookie → attaches to API calls
No auth sync needed between frontend and backend.
```

**Why NOT Better Auth?**
- Better Auth requires running alongside Next.js and syncing to NestJS
- Every auth check needs to cross two services → more latency, more bug surface
- NestJS has first-class JWT support via `@nestjs/jwt` and `@nestjs/passport`
- Refresh token rotation + HttpOnly cookies are standard security patterns
- Solo founder doesn't need multi-provider OAuth (just email/password for MVP)

### Process Architecture (PM2 Ecosystem)

Development (`pnpm dev` runs both):
```
pnpm dev
  ├── apps/web     — Next.js dev server  (port 3000)
  └── apps/api     — NestJS dev server   (port 4000)

pnpm worker
  └── apps/api     — NestJS worker mode  (BullMQ consumers)
```

Production (PM2 `ecosystem.config.js`):
```
PM2 Process Name │ Command                        │ Port  │ Restart
─────────────────┼────────────────────────────────┼───────┼───────
vora-web         │ pnpm --filter @vora/web start  │ 3000  │ on-fail
vora-api         │ node dist/apps/api/src/main    │ 4000  │ on-fail
vora-worker      │ node dist/apps/api/src/worker  │ —     │ on-fail
vora-renderer    │ node apps/renderer/dist/index   │ —     │ always
```

**Note**: The renderer is a **completely separate process** from the worker — it lives in its own `apps/renderer` directory with its own `package.json`. It only imports Remotion + the shared types package. When Remotion crashes (Chromium OOM, FFmpeg bug), only the renderer restarts — API and worker stay online. The renderer connects to the same Redis for BullMQ and the same DB via Prisma.

---

## 3. Database Design (MySQL + Prisma)

### Entity-Relationship Diagram

```
┌──────────────┐       ┌──────────────────┐       ┌──────────────────┐
│    User      │       │    Project       │       │   ProjectAsset   │
│──────────────│       │──────────────────│       │──────────────────│
│ id (PK)      │◄──────│ id (PK)          │       │ id (PK)          │
│ email        │  1:N  │ userId (FK)      │  1:N  │ projectId (FK)   │
│ name         │       │ name             │       │ type (ENUM)      │
│ credits      │       │ status (ENUM)    │       │ filePath         │
│ role         │       │ templateId (FK)  │       │ originalName     │
│ providerId   │       │ productName      │       │ mimeType         │
│ createdAt    │       │ productDesc      │       │ size             │
│ updatedAt    │       │ metadata (JSON)  │       │ order            │
└──────┬───────┘       │ createdAt        │       │ createdAt        │
       │               │ updatedAt        │       └──────────────────┘
       │               └────────┬─────────┘
       │                        │
       │               ┌────────▼─────────┐       ┌──────────────────┐
       │               │      Job         │       │    Template      │
       │               │──────────────────│       │──────────────────│
       │               │ id (PK)          │       │ id (PK)          │
       │               │ projectId (FK)   │  1:N  │ name             │
       │               │ type (ENUM)      │       │ slug             │
       │               │ status (ENUM)    │       │ description      │
       │               │ progress         │       │ thumbnailPath    │
       │               │ error            │       │ config (JSON)    │
       │               │ metadata (JSON)  │       │ isActive         │
       │               │ startedAt        │       │ createdAt        │
       │               │ completedAt      │       └──────────────────┘
       │               └──────────────────┘
       │
       │       ┌──────────────────┐       ┌──────────────────────┐
       │       │ CreditTransaction│       │   SubscriptionPlan   │
       │       │──────────────────│       │──────────────────────│
       │       │ id (PK)          │       │ id (PK)              │
       ├───────│ userId (FK)      │       │ name                 │
               │ amount           │       │ slug                 │
               │ type (ENUM)      │       │ price                │
               │ description      │       │ creditsPerMonth      │
               │ reference (JSON) │       │ features (JSON)      │
               │ createdAt        │       │ isActive             │
               └──────────────────┘       │ createdAt            │
                                           └──────────────────────┘
```

### Prisma Models

```prisma
// apps/api/prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
}

enum ProjectStatus {
  PENDING
  PROCESSING
  COMPLETED
  FAILED
}

enum JobType {
  ANALYZE_PRODUCT
  GENERATE_ANGLE
  GENERATE_SCRIPT
  GENERATE_VOICE
  GENERATE_SUBTITLE
  RENDER_VIDEO
  CLEANUP
}

enum JobStatus {
  PENDING
  ACTIVE
  COMPLETED
  FAILED
  CANCELLED
}

enum AssetType {
  IMAGE
  VIDEO
  AUDIO
  SUBTITLE
  SCRIPT
}

enum CreditTransactionType {
  PURCHASE
  USAGE
  REFUND
  BONUS
  ADMIN
}

model User {
  id         String   @id @default(cuid())
  email      String   @unique
  name       String?
  avatarUrl  String?
  credits    Int      @default(0)
  role       String   @default("user") // "user" | "admin"
  // Auth — NestJS-managed JWT
  passwordHash     String?  // bcrypt hash
  refreshTokenHash String?  // bcrypt hash of current refresh token (never store raw token)

  projects  Project[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([email])
}

model Project {
  id           String        @id @default(cuid())
  userId       String
  name         String
  status       ProjectStatus @default(DRAFT)
  templateId   String?
  productName  String?
  productDesc  String?
  metadata     Json?         // Flexible: selected scenes, custom settings
  deletedAt    DateTime?     // Soft delete

  user    User    @relation(fields: [userId], references: [id])
  assets  ProjectAsset[]
  jobs    Job[]
  template Template?  @relation(fields: [templateId], references: [id])

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([userId, status])
  @@index([userId, createdAt])
}

model ProjectAsset {
  id           String    @id @default(cuid())
  projectId    String
  type         AssetType
  filePath     String    // Local filesystem path
  originalName String
  mimeType     String
  size         Int       // In bytes
  order        Int       @default(0)
  metadata     Json?     // Image dimensions, audio duration, etc.

  project Project @relation(fields: [projectId], references: [id])

  createdAt DateTime @default(now())

  @@index([projectId, type])
}

model Job {
  id          String    @id @default(cuid())
  projectId   String
  type        JobType
  status      JobStatus @default(PENDING)
  progress    Int       @default(0)
  error       String?
  metadata    Json?
  startedAt   DateTime?
  completedAt DateTime?

  project Project @relation(fields: [projectId], references: [id])

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([projectId, type])
  @@index([projectId, status])
}

model Template {
  id             String   @id @default(cuid())
  name           String
  slug           String   @unique
  description    String?
  thumbnailPath  String?
  config         Json     // Scene definitions, durations, styling
  isActive       Boolean  @default(true)

  projects Project[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

// REMOVED from MVP:
// CreditTransaction — post-MVP with Stripe
// SubscriptionPlan — post-MVP with Stripe
// Admin grants credits manually via `UPDATE user SET credits = credits + N`
```

**Why these models and not more?**
- No `Session`/`Account` tables — auth is handled entirely by NestJS JWT + refresh token, stored directly on `User`
- No `Subscription` / `CreditTransaction` tables for MVP — billing is removed from MVP. Admin grants credits manually via DB.
- `metadata` JSON columns for flexibility — avoids schema migrations for template-specific or job-specific variations.
- `deletedAt` on Project for soft-delete — enables recovery and future analytics.

### Key Indexes
- `Project(userId, status)` — list user's projects filtered by status
- `Project(userId, createdAt)` — list user's projects sorted by date
- `Job(projectId, type)` — find specific job for a project
- `Job(projectId, status)` — check all jobs for a project
- `ProjectAsset(projectId, type)` — get assets of a specific type for a project
- `User(refreshTokenHash)` — quick refresh token lookup

---

## 4. Queue Design (BullMQ)

### Queue Architecture — Named Jobs, Switch-Free

BullMQ supports job names natively via `worker.process(name, handler)`. No `switch(step)` needed.

```
┌──────────────────────────────────────────────────┐
│              Queue: generation                   │
│  (Fast jobs — AI calls, file operations)         │
│                                                  │
│  Named jobs (each adds data for the next):       │
│  analyze  →  script  →  voice  →  subtitle      │
│                                                  │
│  Concurrency: 5                                  │
│  Retries: 3 (exponential backoff)                │
└──────────────────────┬───────────────────────────┘
                       │ on 'subtitle' success, enqueue to:
                       ▼
┌──────────────────────────────────────────────────┐
│              Queue: render                       │
│  (Heavy job — Remotion rendering)                │
│                                                  │
│  Named jobs:                                     │
│  render  →  cleanup                              │
│                                                  │
│  Concurrency: 1 (CPU-bound)                      │
│  Retries: 2 (long timeout)                       │
└──────────────────────────────────────────────────┘
```

### Why Named Jobs?

```typescript
// Clean, no switch statement.
// Job data carries only identifiers — state lives in DB:
interface JobData {
  projectId: string;
  jobId: string;
}

worker.process('analyze', async (job) => {
  const { projectId, jobId } = job.data;
  const dbJob = await prisma.job.findUnique({ where: { id: jobId } });
  // Read previous output from dbJob.metadata
  // Do work, write result to dbJob.metadata
  // Enqueue next job with projectId + next jobId
});

worker.process('script',    async (job) => { /* same pattern */ });
worker.process('voice',     async (job) => { /* same pattern */ });
worker.process('subtitle',  async (job) => { /* same pattern */ });
worker.process('render',    async (job) => { /* same pattern */ });
worker.process('cleanup',   async (job) => { /* same pattern */ });
```

**Critical**: BullMQ jobs carry only `{ projectId, jobId }`. All accumulated data is read/written to the `Job.metadata` JSON column in MySQL. This means:
- If Redis crashes mid-pipeline, no state is lost — the DB has everything
- If a worker is restarted, the job picks up where it left off
- BullMQ's Redis is used for scheduling/retries, not for persistent state storage

### Pipeline Flow (Detailed)

```
Generate Request
  │
  ▼
Check credits → Deduct credits → Create Job records in DB
  │
  ▼
Enqueue to 'generation' queue with job name: 'analyze'
  │
  ▼
Worker picks up job 'analyze':
  1. Read project + previous job output from DB (Job.metadata, Project)
  2. LLM call to analyze product
  3. Write analysis result to Job.metadata (JSON column)
  4. Create next Job record in DB (type: GENERATE_SCRIPT, status: PENDING)
  5. Enqueue 'script' job with { projectId, jobId: newJob.id }

Worker picks up job 'script':
  6. Read previous output from DB Job.metadata
  7. LLM call for marketing script
  8. Write script to Job.metadata
  9. Create next Job record, enqueue 'voice'

Worker picks up job 'voice':
  10. Read script from DB
  11. TTS (OpenAI/ElevenLabs) → save audio file via StorageProvider
  12. Write voice file path to Job.metadata
  13. Create next Job record, enqueue 'subtitle'

Worker picks up job 'subtitle':
  14. Read script + voice timing from DB
  15. Generate SRT → save via StorageProvider
  16. Write subtitle path to Job.metadata
  17. Create Render Job record, enqueue to 'render' queue
  │
  ▼
Renderer (separate PM2 process) picks up job 'render':
  18. Read all accumulated data from DB
  19. Remotion renderMedia() with all data
  20. Save MP4 via StorageProvider
  21. Write video path to Job.metadata
  22. Create next Job record, enqueue 'cleanup'

Renderer picks up job 'cleanup':
  23. Update Project status = COMPLETED
  24. Remove temp files
  25. Emit WebSocket event to frontend
```

### Progress Tracking

Each step updates:
1. **BullMQ job.progress(n)** — tracks within-queue progress (0-100)
2. **Job record in DB** — updates `progress` and `status` columns
3. **WebSocket event** — emitted to frontend in real-time

Frontend subscribes via WebSocket to `project:{projectId}:progress` channel.

### Retry & Error Strategy
| Queue | Max Retries | Backoff | On Final Failure |
|-------|-------------|---------|------------------|
| `generation` | 3 | 5s → 30s → 120s | Mark project FAILED, refund credits |
| `render` | 2 | 30s → 300s | Mark project FAILED, refund credits |

---

## 5. API Design (NestJS)

### Module Structure (Flat NestJS Modules)

```
apps/api/src/
├── main.ts                    # Bootstrap HTTP server
├── worker.ts                  # Bootstrap worker mode (no HTTP)
├── app.module.ts              # Root module
├── app.controller.ts          # Health check
│
├── modules/
│   ├── auth/
│   │   ├── auth.module.ts
│   │   ├── auth.controller.ts       # Login, register, refresh, logout
│   │   ├── auth.service.ts          # JWT sign/verify, refresh rotation
│   │   ├── jwt-auth.guard.ts        # JWT validation
│   │   └── current-user.decorator.ts
│   │
│   ├── projects/
│   │   ├── projects.module.ts
│   │   ├── projects.controller.ts
│   │   ├── projects.service.ts
│   │   └── dto/
│   │       ├── create-project.dto.ts
│   │       └── update-project.dto.ts
│   │
│   ├── videos/
│   │   ├── videos.module.ts
│   │   ├── videos.controller.ts      # Generate, progress, download
│   │   ├── videos.service.ts
│   │   └── generation.gateway.ts     # WebSocket gateway
│   │
│   ├── templates/
│   │   ├── templates.module.ts
│   │   ├── templates.controller.ts
│   │   └── templates.service.ts
│   │
│   ├── assets/
│   │   ├── assets.module.ts
│   │   ├── assets.controller.ts      # Upload, list, delete
│   │   └── assets.service.ts
│   │
│   └── credits/
│       ├── credits.module.ts
│       └── credits.service.ts         # Read balance, check, deduct, admin grant
│   # NOTE: No billing/stripe module — removed from MVP
│
├── workers/
│   ├── generation.worker.ts          # Queue consumer (analyze, script, voice, subtitle)
│   └── queue.service.ts             # Queue management
│
├── admin/
│   └── bull-board.controller.ts     # @bull-board dashboard at /admin/queues
│
└── renderer/                          # Completely separate app (apps/renderer/)
    ├── src/
    │   ├── index.ts                   # Bootstrap BullMQ consumer + Remotion
    │   ├── renderer.service.ts        # Remotion renderMedia() wrapper
    │   ├── templates/                 # Remotion composition templates
    │   │   ├── product-review.tsx
    │   │   ├── ugc-style.tsx
    │   │   ├── problem-solution.tsx
    │   │   ├── flash-sale.tsx
    │   │   └── features-showcase.tsx
    │   └── scenes/
    │       ├── title-scene.tsx
    │       ├── image-scene.tsx
    │       ├── text-scene.tsx
    │       ├── cta-scene.tsx
    │       └── outro-scene.tsx
    ├── package.json
    └── tsconfig.json
│
└── common/
    ├── filters/
    │   └── http-exception.filter.ts
    ├── interceptors/
    │   └── logging.interceptor.ts
    └── utils/
        └── file-storage.ts           # Local file operations
```

### REST Endpoints

```
Auth (NestJS-native JWT)
─────────────────────────────────────────────────────
POST   /api/auth/register          — Register with email/password
POST   /api/auth/login             — Login (returns JWT + sets refresh cookie)
POST   /api/auth/refresh           — Rotate refresh token, return new JWT
POST   /api/auth/logout            — Logout (clear refresh token)
GET    /api/auth/me                — Get current user profile

Projects
─────────────────────────────────────────────────────
GET    /api/projects               — List user's projects (paginated)
POST   /api/projects               — Create new project
GET    /api/projects/:id           — Get project with assets + jobs
PATCH  /api/projects/:id           — Update project info
DELETE /api/projects/:id           — Soft-delete project

Assets (Product Images)
─────────────────────────────────────────────────────
POST   /api/projects/:id/assets    — Upload product image (multipart)
GET    /api/projects/:id/assets    — List project assets
DELETE /api/projects/:id/assets/:assetId — Remove asset

Templates
─────────────────────────────────────────────────────
GET    /api/templates              — List available templates
GET    /api/templates/:id          — Get template details + scenes

Video Generation
─────────────────────────────────────────────────────
POST   /api/projects/:id/generate  — Start generation pipeline
WS     /ws                         — WebSocket namespace (subscribe to `project:{id}:progress`)
GET    /api/projects/:id/progress  — Poll current progress (fallback)
GET    /api/projects/:id/video     — Get rendered video metadata + stream URL
GET    /api/projects/:id/download  — Download rendered MP4

Billing (Post-MVP — stub for now)
─────────────────────────────────────────────────────
GET    /api/billing/credits        — Get credit balance
# Stripe, plans, subscriptions, webhooks removed from MVP
# Admin grants credits manually via DB query
```

### API Response Conventions

```typescript
// Success
{
  "data": { ... },
  "meta": { "page": 1, "pageSize": 20, "total": 42 }
}

// Error
{
  "error": {
    "code": "INSUFFICIENT_CREDITS",
    "message": "You need at least 5 credits to generate a video.",
    "details": { "available": 2, "required": 5 }
  }
}

// List
{
  "data": [ ... ],
  "meta": { "page": 1, "pageSize": 20, "total": 42 }
}
```

### WebSocket Gateway (Replaces SSE)

NestJS `@nestjs/websockets` provides first-class WebSocket support. A single gateway handles all real-time events:

```typescript
@WebSocketGateway({
  namespace: '/ws',
  cors: { origin: process.env.FRONTEND_URL }
})
export class GenerationGateway {
  @SubscribeMessage('subscribe')
  handleSubscribe(client: Socket, payload: { projectId: string }) {
    // Join room: client joins `project:{projectId}` room
    client.join(`project:${payload.projectId}`);
  }

  // Called from workers after each pipeline step
  emitProgress(projectId: string, data: ProgressEvent) {
    this.server.to(`project:${projectId}`).emit('progress', data);
  }

  emitCompleted(projectId: string, data: CompleteEvent) {
    this.server.to(`project:${projectId}`).emit('completed', data);
  }

  emitFailed(projectId: string, data: ErrorEvent) {
    this.server.to(`project:${projectId}`).emit('failed', data);
  }
}
```

**Why WebSocket over SSE?**
- BullMQ events → emit socket naturally (no polling wrapper)
- Supports multi-tab sync (same user, different tabs)
- Future: admin monitoring dashboard, real-time notification system
- NestJS `@nestjs/websockets` integrates with the same DI container — negligible complexity
- SSE requires custom `Observable` + manual reconnection logic; WebSocket handles reconnection client-side

### Error Codes
| Code | HTTP Status | Description |
|------|-------------|-------------|
| `VALIDATION_ERROR` | 422 | Invalid input |
| `UNAUTHORIZED` | 401 | Not authenticated |
| `FORBIDDEN` | 403 | Not authorized |
| `NOT_FOUND` | 404 | Resource not found |
| `INSUFFICIENT_CREDITS` | 402 | Payment required |
| `PROJECT_NOT_EDITABLE` | 409 | Project already being generated |
| `GENERATION_FAILED` | 500 | Pipeline step failed |
| `RATE_LIMITED` | 429 | Too many requests |

---

## 6. Remotion Design

### Architecture Overview

Remotion runs as a **library** within the NestJS renderer process (not a separate service). The `@remotion/renderer` package's `renderMedia()` function is called directly from the `render` queue processor.

```
'render' queue job received
  │
  ▼
renderer.service.ts
  ├── Build composition input props from job.data
  ├── Call renderMedia() with composition ID + input props
  │     │
  │     ▼
  │   Remotion renders each frame server-side
  │   (No browser — uses @remotion/renderer Node API)
  │     │
  │     ▼
  ├── MP4 written to temp directory
  ├── Move via StorageProvider.save() to /storage/{projectId}/video.mp4
  ├── Emit WebSocket 'completed' event via GenerationGateway
  └── Return path → stored in ProjectAsset record
```

### Template System

Templates are **React components** in `apps/api/src/renderer/templates/`. Each template is a Remotion `<Composition>` registered in a central registry.

```typescript
// Template Registry Concept
const templateRegistry = {
  'product-review': {
    component: ProductReview,
    compositionId: 'product-review',
    defaultDuration: 15,  // seconds
    fps: 30,
    inputProps: {
      productName: string,
      productDescription: string,
      marketingAngle: string,
      script: string[],
      images: string[],       // File paths
      voiceOverPath: string,  // Audio file path
      subtitleSrt: string,    // SRT file path
    }
  },
  'ugc-style': { /* ... */ },
  'problem-solution': { /* ... */ },
  'flash-sale': { /* ... */ },
  'features-showcase': { /* ... */ },
};
```

### Template Composition Structure

Each template is built from reusable scene components:
```typescript
// Building blocks shared across templates
├── scenes/
│   ├── TitleScene.tsx        // Animated title card
│   ├── ImageScene.tsx        // Product image + overlay
│   ├── TextScene.tsx         // Script line display
│   ├── CTAScene.tsx          // Call-to-action card
│   └── OutroScene.tsx        // Brand/closing card
```

### Rendering Options (per template)

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Resolution | 1080×1920 (9:16) | TikTok/Reels/Shorts format |
| FPS | 30 | Standard for social video |
| Codec | h264 | Best compatibility (h264 is universal) |
| Video bitrate | 8 Mbps | Good quality for 1080p |
| Audio codec | aac | Universal audio format |
| Duration | 15-60s | Short-form content |

### Template Catalog (MVP — 5 Templates)

| Template | Slug | Vibe | Scenes | Duration |
|----------|------|------|--------|----------|
| Product Review | `product-review` | Honest review, comparison | Hook → Showcase → Verdict → CTA | 30s |
| UGC Style | `ugc-style` | Authentic, raw, unboxing feel | Hook → Unboxing → Up close → CTA | 20s |
| Problem/Solution | `problem-solution` | Pain point → solution | Problem → Solution → Features → CTA | 25s |
| Flash Sale | `flash-sale` | Urgency, discounts, limited | Timer → Offer → Product → CTA | 15s |
| Feature Showcase | `features-showcase` | Clean feature walkthrough | Intro → Feature 1 → Feature 2 → Outro | 40s |

---

## 7. Folder Structure (Complete)

```
vora-ai/
├── .github/
│   └── workflows/
│       └── ci.yml                    # Lint, type-check, test
│
├── apps/
│   ├── web/                          # Next.js 16 Frontend
│   │   ├── public/
│   │   │   └── images/
│   │   │       ├── logo.svg
│   │   │       └── og-image.png
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── layout.tsx                    # Root layout + providers
│   │   │   │   ├── page.tsx                      # Landing page
│   │   │   │   ├── globals.css                   # Tailwind imports
│   │   │   │   ├── auth/
│   │   │   │   │   ├── login/page.tsx
│   │   │   │   │   └── register/page.tsx
│   │   │   │   └── (dashboard)/
│   │   │   │       ├── layout.tsx                # Dashboard shell
│   │   │   │       ├── page.tsx                  # Dashboard redirect
│   │   │   │       ├── projects/
│   │   │   │       │   ├── page.tsx              # Project grid
│   │   │   │       │   ├── new/
│   │   │   │       │   │   └── page.tsx          # Create wizard
│   │   │   │       │   └── [id]/
│   │   │   │       │       ├── page.tsx          # Project detail + video
│   │   │   │       │       └── edit/page.tsx     # Edit project
│   │   │   │       ├── templates/  # REMOVED from apps/api: moved to apps/renderer
│   │   │   │       │   └── page.tsx              # Template browser
│   │   │   │       └── credits/
│   │   │   │           └── page.tsx              # Credit balance display
│   │   │   │       └── settings/
│   │   │   │           └── page.tsx              # Account settings
│   │   │   ├── components/
│   │   │   │   ├── ui/                           # shadcn/ui primitives (latest)
│   │   │   │   │   ├── button.tsx
│   │   │   │   │   ├── card.tsx
│   │   │   │   │   ├── dialog.tsx
│   │   │   │   │   ├── input.tsx
│   │   │   │   │   ├── progress.tsx
│   │   │   │   │   ├── select.tsx
│   │   │   │   │   └── ...                       # Other shadcn components
│   │   │   │   ├── layout/
│   │   │   │   │   ├── app-shell.tsx             # Dashboard layout
│   │   │   │   │   ├── sidebar.tsx               # Nav sidebar
│   │   │   │   │   ├── topbar.tsx                # User menu + credits
│   │   │   │   │   └── mobile-nav.tsx
│   │   │   │   ├── projects/
│   │   │   │   │   ├── project-grid.tsx
│   │   │   │   │   ├── project-card.tsx
│   │   │   │   │   ├── create-project-wizard.tsx
│   │   │   │   │   ├── product-info-form.tsx
│   │   │   │   │   ├── image-uploader.tsx
│   │   │   │   │   ├── template-selector.tsx
│   │   │   │   │   ├── generation-progress.tsx
│   │   │   │   │   ├── video-player.tsx
│   │   │   │   │   └── download-button.tsx
│   │   │   │   └── credits/
│   │   │   │       └── credit-balance.tsx         # Display remaining credits
│   │   │   │   └── shared/
│   │   │   │       ├── status-badge.tsx
│   │   │   │       ├── empty-state.tsx
│   │   │   │       ├── confirm-dialog.tsx
│   │   │   │       └── file-dropzone.tsx
│   │   │   ├── lib/
│   │   │   │   ├── api-client.ts         # Axios/fetch wrapper
│   │   │   │   ├── auth-client.ts        # JWT auth client (login, register, refresh)
│   │   │   │   ├── query-keys.ts         # TanStack Query key constants
│   │   │   │   └── utils.ts             # cn(), formatters
│   │   │   └── hooks/
│   │   │       ├── use-projects.ts       # TanStack Query hooks
│   │   │       ├── use-generation.ts     # WebSocket hook for progress
│   │   │       └── use-credits.ts        # Credit balance hook
│   │   ├── .env.local                    # NEXT_PUBLIC_API_URL
│   │   ├── next.config.ts
│   │   ├── tailwind.config.ts
│   │   ├── tsconfig.json
│   │   ├── components.json               # shadcn/ui config
│   │   └── package.json
│   │
│   └── api/                              # NestJS Backend
│       ├── prisma/
│       │   ├── schema.prisma
│       │   └── seed.ts                   # Template seed data
│       ├── src/
│       │   ├── main.ts                   # Bootstrap HTTP
│       │   ├── worker.ts                 # Bootstrap worker only
│       │   ├── app.module.ts
│       │   ├── app.controller.ts         # /api/health
│       │   ├── modules/
│       │   │   ├── auth/
│       │   │   │   ├── auth.module.ts
│       │   │   │   ├── auth.controller.ts
│       │   │   │   ├── auth.service.ts
│       │   │   │   ├── jwt-auth.guard.ts
│       │   │   │   └── current-user.decorator.ts
│       │   │   ├── projects/
│       │   │   │   ├── projects.module.ts
│       │   │   │   ├── projects.controller.ts
│       │   │   │   ├── projects.service.ts
│       │   │   │   ├── dto/
│       │   │   │   │   ├── create-project.dto.ts
│       │   │   │   │   └── update-project.dto.ts
│       │   │   │   └── utils/
│       │   │   │       └── project-validator.ts
│       │   │   ├── videos/
│       │   │   │   ├── videos.module.ts
│       │   │   │   ├── videos.controller.ts
│       │   │   │   ├── videos.service.ts
│       │   │   │   ├── generation.gateway.ts   # WebSocket gateway
│       │   │   │   └── dto/
│       │   │   │       └── generate-video.dto.ts
│       │   │   ├── templates/
│       │   │   │   ├── templates.module.ts
│       │   │   │   ├── templates.controller.ts
│       │   │   │   └── templates.service.ts
│       │   │   ├── assets/
│       │   │   │   ├── assets.module.ts
│       │   │   │   ├── assets.controller.ts
│       │   │   │   ├── assets.service.ts
│       │   │   │   └── file-upload.interceptor.ts
│       │   │   └── credits/
│       │   │       ├── credits.module.ts
│       │   │       └── credits.service.ts
│       │   ├── admin/
│       │   │   └── bull-board.controller.ts   # @bull-board at /admin/queues
│       │   ├── workers/
│       │   │   ├── generation.worker.ts        # BullMQ consumer (analyze, script, voice, subtitle)
│       │   │   ├── queue.service.ts            # Queue management
│       │   │   └── queue.module.ts
│       │   # NOTE: Renderer moved to apps/renderer/ (separate process)
│       │   ├── common/
│       │   │   ├── filters/
│       │   │   │   └── http-exception.filter.ts
│       │   │   ├── interceptors/
│       │   │   │   └── logging.interceptor.ts
│       │   │   └── utils/
│       │   │       ├── file-storage.ts
│       │   │       └── logger.ts
│       │   └── config/
│       │       └── configuration.ts              # NestJS ConfigService
│       ├── uploads/                              # Gitignored — local storage root
│       │   └── .gitkeep
│       ├── storage/                              # Gitignored — rendered output
│       │   └── .gitkeep
│       ├── test/
│       │   ├── app.e2e-spec.ts
│       │   └── jest-e2e.json
│       ├── .env
│       ├── .env.example
│       ├── nest-cli.json
│       ├── tsconfig.json
│       ├── tsconfig.build.json
│       └── package.json
│
├── packages/
│   └── shared/
│       ├── src/
│       │   ├── types/
│       │   │   ├── project.types.ts
│       │   │   ├── job.types.ts
│       │   │   ├── template.types.ts
│       │   │   ├── asset.types.ts
│       │   │   └── billing.types.ts
│       │   ├── constants/
│       │   │   ├── project.constants.ts          # Enums, statuses
│       │   │   ├── limits.ts                     # File size, count limits
│       │   │   └── credits.ts                    # Credit costs per action
│       │   └── validation/
│       │       ├── project.schema.ts             # Zod schemas
│       │       ├── generate.schema.ts
│       │       └── common.schema.ts              # Pagination, etc.
│       ├── tsconfig.json
│       └── package.json
│
├── ecosystem.config.js           # PM2 config for all processes
├── .env.example                  # All env vars documented
├── .gitignore
├── .prettierrc
├── .eslintrc.js
├── pnpm-workspace.yaml
├── package.json                  # Root scripts
└── README.md
```

---

## 8. Provider Adapter System

### Why Provider Adapters?

An AI video platform will switch AI providers constantly:
- **Script**: OpenAI / Claude / Gemini (price, quality, feature differences)
- **Voice**: OpenAI TTS / ElevenLabs / Cartesia (voice quality vs cost)
- **Image**: DALL-E / Stable Diffusion / Midjourney (future expansion)
- **Video**: Kling / Veo / Runway / Hailuo (future expansion)

Each provider has different APIs, rate limits, and pricing. Wrapping them behind interfaces means swapping providers is a config change, not a code rewrite.

### Provider Interfaces

```typescript
// apps/api/src/providers/

interface ScriptProvider {
  analyzeProduct(name: string, description: string): Promise<ProductAnalysis>;
  generateMarketingAngle(analysis: ProductAnalysis): Promise<string>;
  generateScript(angle: string, product: ProductInfo): Promise<ScriptLine[]>;
}

interface VoiceProvider {
  generateVoice(script: string, options: VoiceOptions): Promise<Buffer>;
  getVoices(): Promise<VoiceOption[]>;
}

interface ImageProvider {
  generateImage(prompt: string, options?: ImageOptions): Promise<Buffer>;
}

interface VideoProvider {
  generateVideo(params: VideoParams): Promise<Buffer>;
}
```

### Provider Implementations (MVP)

```
providers/
├── interfaces/
│   ├── script.provider.ts
│   ├── voice.provider.ts
│   ├── image.provider.ts
│   └── video.provider.ts
├── script/
│   └── openai-script.provider.ts     // GPT-4o-mini
├── voice/
│   └── openai-voice.provider.ts      // OpenAI TTS (MVP)
├── image/
│   └── openai-image.provider.ts      // DALL-E 3 (future)
└── video/
    └── remotion-video.provider.ts    // Remotion (our renderer)
```

### Provider Factory

```typescript
@Injectable()
export class ProviderFactory {
  constructor(private config: ConfigService) {}

  getScriptProvider(): ScriptProvider {
    const provider = this.config.get('SCRIPT_PROVIDER');
    switch (provider) {
      case 'openai': return new OpenAIScriptProvider(this.config);
      case 'anthropic': return new AnthropicScriptProvider(this.config);
      default: return new OpenAIScriptProvider(this.config);
    }
  }

  getVoiceProvider(): VoiceProvider {
    const provider = this.config.get('VOICE_PROVIDER');
    switch (provider) {
      case 'openai': return new OpenAIVoiceProvider(this.config);
      case 'elevenlabs': return new ElevenLabsVoiceProvider(this.config);
      default: return new OpenAIVoiceProvider(this.config);
    }
  }
}
```

Swap provider via `.env`:
```
SCRIPT_PROVIDER=openai
VOICE_PROVIDER=openai
```

No code changes needed to switch between providers.

### Prompt Engineering Strategy

| Step | Model | Prompt Style |
|------|-------|-------------|
| Product Analysis | Gemini 2.5 Flash / Claude Sonnet | Analyze product category, target audience, selling points |
| Marketing Angle | Gemini 2.5 Flash / Claude Sonnet | Generate 3 angles, pick best based on product type |
| Script Generation | Gemini 2.5 Flash / Claude Sonnet | Generate timed script with scene markers |
| Subtitle Timing | Local algorithm | Map script text to voice timeline for SRT |

### Cost Optimization Notes
- MVP default: Gemini 2.5 Flash (best quality/price ratio as of 2026). Fallback: Claude Sonnet via `ScriptProvider`
- GPT-4o-mini also available as `SCRIPT_PROVIDER=gpt4o-mini` — set via `.env`, no code change
- Cache repeated analyses (same product name/desc → same result)
- OpenAI TTS at $0.015/1K chars vs ElevenLabs at $0.30/1K chars — 20x cheaper
- Total AI cost per video: ~$0.02-0.05 (LLM + TTS)

---

## 9. File Storage Design (Local → S3)

### Local Storage Structure (MVP)

```
/apps/api/
├── uploads/
│   └── {userId}/
│       └── {projectId}/
│           ├── images/              # Uploaded product images
│           │   ├── 01-original.jpg
│           │   ├── 02-original.png
│           │   └── ...
│           └── temp/                # Temp processing files
│               └── ...
│
└── storage/
    └── {projectId}/
        ├── voice.mp3               # Generated voice-over
        ├── subtitles.srt           # Generated subtitles
        ├── video.mp4               # Final rendered video
        └── thumbnail.jpg           # Video thumbnail
```

### File Size Limits
| Type | Max Size | Max Count | Allowed Formats |
|------|----------|-----------|-----------------|
| Product Images | 10 MB each | 5 per project | jpg, png, webp |
| Rendered Video | 200 MB | 1 per project | mp4 |
| Voice Audio | 50 MB | 1 per project | mp3, wav |
| Subtitles | 1 MB | 1 per project | srt |

### Storage Abstraction — Strict Interface from Day 1

```typescript
interface StorageProvider {
  save(file: Buffer, path: string): Promise<string>;   // Returns full path
  get(path: string): Promise<Buffer>;
  delete(path: string): Promise<void>;
  getPublicUrl(path: string): string;    // For streaming/download
}

// MVP Implementation — LOCAL ONLY
class LocalStorageProvider implements StorageProvider {
  // Base path: process.env.STORAGE_PATH || './storage'
  // save → write to disk
  // getPublicUrl → return /api/files/{encodedPath}
}

// Future Implementation
class S3StorageProvider implements StorageProvider {
  // Uses S3-compatible (R2, MinIO, AWS S3)
  // getPublicUrl → return signed URL
}
```

**Critical rule**: Business code NEVER calls `fs.writeFile`, `fs.readFile`, or any Node.js `fs` or `path` module directly. All file operations go through `StorageProvider`. This makes the S3 migration a matter of:

```
// production.ts — one-line swap
const storage = new S3StorageProvider({
  endpoint: process.env.S3_ENDPOINT,  // R2 / Backblaze / MinIO
  bucket: 'vora-ai',
});
```

Without this discipline, `fs.writeFile` calls will be scattered across 20+ files, and migration becomes a multi-week refactor.

---

## 10. Frontend Architecture (Next.js 16)

### WebSocket Client Hook

```typescript
// apps/web/src/hooks/use-generation.ts
const useGenerationProgress = (projectId: string) => {
  const [progress, setProgress] = useState<ProgressEvent | null>(null);

  useEffect(() => {
    const socket = io(`${process.env.NEXT_PUBLIC_WS_URL}/ws`, {
      withCredentials: true,
    });

    socket.emit('subscribe', { projectId });

    socket.on('progress', (data) => setProgress(data));
    socket.on('completed', (data) => {
      setProgress(data);
      // Invalidate TanStack Query cache
      queryClient.invalidateQueries({ queryKey: ['projects', projectId] });
    });
    socket.on('failed', (data) => {
      setProgress(data);
      queryClient.invalidateQueries({ queryKey: ['projects', projectId] });
    });

    return () => { socket.disconnect(); };
  }, [projectId]);

  return progress;
};
```

### Route Design

```
/                                → Landing page (marketing)
/auth/login                      → Login page
/auth/register                   → Registration page
/dashboard                       → Dashboard (project grid, default page)
/dashboard/projects/new          → Create project wizard (multi-step)
/dashboard/projects/[id]         → Project detail + video preview
/dashboard/projects/[id]/edit    → Edit project info
/dashboard/templates             → Template browser
/dashboard/credits               → Credit balance
/dashboard/settings              → Account settings
```

### Component Tree (Key Pages)

**Create Project Wizard** (`/dashboard/projects/new`)
```
CreateProjectWizard
├── StepIndicator (1: Info → 2: Images → 3: Template → 4: Generate)
├── Step 1: ProductInfoForm
│   ├── Input (Product Name)
│   └── Textarea (Product Description)
├── Step 2: ImageUploader
│   └── FileDropzone × up to 5
├── Step 3: TemplateSelector
│   └── TemplateCard × N (thumbnail, name, duration badge)
└── Step 4: GenerationProgress
    ├── ProgressBar (overall %)
    ├── StepList (analyzing ✓ → angle ✓ → script ✓ → voice ⟳ → subtitle → render)
    └── CancelButton
```

**Project Detail** (`/dashboard/projects/[id]`)
```
ProjectDetail
├── VideoPlayer (rendered MP4)
├── ProjectInfo (name, status, created date)
├── DownloadButton
├── RegenerateButton (opens wizard with pre-filled data)
└── JobHistory (list of pipeline steps with status badges)
```

**Dashboard** (`/dashboard`)
```
DashboardPage
├── Topbar (user menu, credit balance)
├── ProjectGrid
│   ├── ProjectCard × N (thumbnail, name, status badge, date)
│   └── EmptyState (when no projects)
└── CreateNewProjectButton → /dashboard/projects/new
```

### State Management (TanStack Query)

```typescript
// Query keys
const queryKeys = {
  projects: {
    all: ['projects'] as const,
    list: (filters: ProjectFilters) => ['projects', 'list', filters] as const,
    detail: (id: string) => ['projects', 'detail', id] as const,
  },
  templates: {
    all: ['templates'] as const,
    detail: (id: string) => ['templates', 'detail', id] as const,
  },
  credits: {
    balance: ['credits', 'balance'] as const,
  },
};

// Key mutations
const useCreateProject = () => useMutation({ mutationFn: ... });
const useUploadImage = () => useMutation({ mutationFn: ... });
const useGenerateVideo = () => useMutation({ mutationFn: ... });

// WebSocket hook for real-time progress
const useGenerationProgress = (projectId: string) => {
  // Uses WebSocket via socket.io-client
  // Subscribes to `project:{projectId}` room
  // Listens for: 'progress', 'completed', 'failed' events
  // Invalidates project query on completion/failure
};
```

---

## 11. Deployment Architecture

### Production Topology (Single VPS)

```
┌─────────────────────────────────────────────────────┐
│                    Ubuntu VPS                        │
│                    (Hetzner CX22 ~$11/mo)            │
│                                                      │
│  ┌─────────┐    ┌─────────┐    ┌──────────────────┐ │
│  │  Nginx  │───▶│  PM2    │───▶│  Next.js :3000   │ │
│  │  :443   │    │(Manager)│    └──────────────────┘ │
│  │  :80    │    │         │    ┌──────────────────┐ │
│  └─────────┘    │         │───▶│  NestJS :4000    │ │
│       ▲         │         │    └──────────────────┘ │
│       │         │         │    ┌──────────────────┐ │
│  ┌────┴────┐    │         │───▶│  Worker (int.)   │ │
│  │  Let's  │    │         │    └──────────────────┘ │
│  │ Encrypt │    │         │    ┌──────────────────┐ │
│  └─────────┘    │         │───▶│  Renderer (int.) │ │
│                 │         │    └──────────────────┘ │
│                 └─────────┘                         │
│                                                      │
│  ┌──────────┐        ┌──────────┐                   │
│  │  MySQL   │        │  Redis   │                   │
│  │  :3306   │        │  :6379   │                   │
│  └──────────┘        └──────────┘                   │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │  /var/www/vora-ai/                             │  │
│  │  ├── apps/web/.next/    (built Next.js)        │  │
│  │  ├── apps/api/dist/     (built NestJS)         │  │
│  │  ├── uploads/           (user images)          │  │
│  │  └── storage/           (rendered videos)      │  │
│  └────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

### Nginx Configuration

```
server {
    listen 443 ssl;
    server_name vora.ai;

    # SSL (Let's Encrypt via certbot)
    ssl_certificate /etc/letsencrypt/live/vora.ai/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/vora.ai/privkey.pem;

    # Static files (Next.js public assets)
    location /_next/static {
        alias /var/www/vora-ai/apps/web/public;
        expires 365d;
    }

    # API proxy
    location /api/ {
        proxy_pass http://127.0.0.1:4000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;

        # Longer timeout for SSE
        proxy_read_timeout 86400s;
    }

    # File serving (local storage)
    location /files/ {
        alias /var/www/vora-ai/storage/;
        internal;  # Only accessible through API
        add_header Content-Disposition 'attachment';
    }

    # Frontend
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### PM2 Ecosystem (`ecosystem.config.js`)

```javascript
module.exports = {
  apps: [
    {
      name: 'vora-web',
      cwd: './apps/web',
      script: 'node_modules/.bin/next',
      args: 'start',
      env: { PORT: 3000, NODE_ENV: 'production' },
      instances: 1,
      exec_mode: 'fork',
      max_memory_restart: '1G',
    },
    {
      name: 'vora-api',
      cwd: './apps/api',
      script: 'dist/src/main.js',
      env: { PORT: 4000, NODE_ENV: 'production' },
      instances: 1,
      exec_mode: 'fork',
      max_memory_restart: '1G',
    },
    {
      name: 'vora-worker',
      cwd: './apps/api',
      script: 'dist/src/worker.js',
      env: { NODE_ENV: 'production', WORKER_MODE: 'true' },
      instances: 1,
      exec_mode: 'fork',
      max_memory_restart: '1G',
      autorestart: true,  // PM2 restarts on OOM/uncaught exception/SIGSEGV
    },
    {
      name: 'vora-renderer',
      cwd: './apps/renderer',
      script: 'dist/index.js',
      env: { NODE_ENV: 'production' },
      instances: 1,
      exec_mode: 'fork',
      max_memory_restart: '4G',  // Remotion is memory-intensive
      autorestart: true,
    },
  ],
};
```

### Deployment Commands

```bash
# Initial server setup
sudo apt update && sudo apt upgrade -y
sudo apt install -y nginx mysql-server redis-server
curl -fsSL https://deb.nodesource.com/setup_26.x | sudo -E bash -
sudo apt install -y nodejs
sudo corepack enable && corepack prepare pnpm@latest --activate

# Clone & build
git clone https://github.com/your-org/vora-ai /var/www/vora-ai
cd /var/www/vora-ai
pnpm install
pnpm --filter @vora/api exec prisma generate
pnpm --filter @vora/api exec prisma db push   # or migrate
pnpm build

# Start
pm2 start ecosystem.config.js
pm2 save
pm2 startup   # Auto-start on reboot
```

---

## 12. Environment Variables

```
# .env (apps/api/.env)
DATABASE_URL=mysql://user:password@localhost:3306/vora_ai
REDIS_URL=redis://localhost:6379

# Auth (NestJS JWT)
JWT_SECRET=your-secret-here-rotate-in-production
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=30d

# AI Providers
OPENAI_API_KEY=sk-...

# Storage
STORAGE_PATH=./storage
UPLOAD_PATH=./uploads
MAX_FILE_SIZE=10485760

# Queue
GENERATION_QUEUE_CONCURRENCY=5
RENDER_QUEUE_CONCURRENCY=1

# Bull Board (admin dashboard)
BULL_BOARD_USERNAME=admin
BULL_BOARD_PASSWORD=your-password-here

# URLs (for CORS, redirects)
FRONTEND_URL=http://localhost:3000
API_URL=http://localhost:4000

# Next.js (.env.local for apps/web/)
NEXT_PUBLIC_API_URL=http://localhost:4000
NEXT_PUBLIC_WS_URL=http://localhost:4000
```

---

## 13. Development Roadmap

### Sprint 1: Foundation (Week 1-2)

| Step | Description | Dependencies |
|------|-------------|-------------|
| 1.1 | Initialize monorepo: root `package.json`, `pnpm-workspace.yaml`, `.gitignore`, `.prettierrc`, `.eslintrc` | None |
| 1.2 | Create `packages/shared`: TypeScript config, Zod validation schemas, type definitions, constants | None |
| 1.3 | Scaffold `apps/api` with NestJS CLI: AppModule, health endpoint, ConfigService | Step 1.1 |
| 1.4 | Scaffold `apps/web` with `create-next-app`: TypeScript, Tailwind, App Router, shadcn/ui init | Step 1.1 |
| 1.5 | Configure Prisma: `schema.prisma` with all models, `.env`, `prisma generate` | Step 1.3 |
| 1.6 | Implement JWT auth: `@nestjs/jwt`, `@nestjs/passport`, register/login/refresh/logout endpoints | Step 1.3 |
| 1.7 | Set up NestJS project structure: global exception filter, jwt-auth guard, `@CurrentUser()` decorator | Step 1.6 |
| 1.8 | Set up Next.js project structure: dashboard layout, sidebar, topbar, API client | Step 1.4 |
| 1.9 | Configure BullMQ: queue definitions, queue service, basic worker scaffold | Step 1.3 |
| 1.10 | Write PM2 `ecosystem.config.js`, project `README.md` | Step 1.1 |

### Sprint 2: Core Features (Week 3-4)

| Step | Description | Dependencies |
|------|-------------|-------------|
| 2.1 | **Projects CRUD**: NestJS module + controller + service, Prisma queries | Step 1.5 |
| 2.2 | **Projects UI**: Dashboard page with project grid, create project wizard (steps 1-2) | Step 1.8, 2.1 |
| 2.3 | **File Upload**: NestJS multer config, file validation (type, size), save to local `uploads/` | Step 1.3 |
| 2.4 | **Image Uploader UI**: Drag-and-drop component, preview, upload progress | Step 2.3 |
| 2.5 | **Template System**: Prisma seed with 5 templates, NestJS template module | Step 1.5 |
| 2.6 | **Template Selector UI**: Template cards with thumbnails, selection state | Step 2.5 |
| 2.7 | **Generate Endpoint**: Validate credits → create Job records → enqueue to BullMQ | Step 1.9, 2.1 |
| 2.8 | **AI Service**: OpenAI integration for product analysis, angle generation, script generation | Step 1.3 |
| 2.9 | **Pipeline Worker (AI)**: Video pipeline worker with ANALYZE → ANGLE → SCRIPT steps | Step 2.8, 1.9 |
| 2.10 | **WebSocket Gateway**: NestJS WebSocket gateway, event emission from workers | Step 2.7 |

### Sprint 3: Video Pipeline (Week 5-6)

| Step | Description | Dependencies |
|------|-------------|-------------|
| 3.1 | **Voice Generation**: TTS integration (OpenAI TTS), save audio file | Step 2.9 |
| 3.2 | **Subtitle Generation**: Generate SRT from script + voice duration timing | Step 3.1 |
| 3.3 | **Remotion Setup**: Install `@remotion/renderer`, create template compositions | Step 1.3 |
| 3.4 | **Template Compositions**: Build 5 template components with scene system | Step 3.3 |
| 3.5 | **Render Service**: `renderer.service.ts` — `renderMedia()` wrapper with input props | Step 3.4 |
| 3.6 | **Render App**: Create `apps/renderer` with BullMQ consumer + Remotion, connect to 'render' queue | Step 3.5, 1.9 |
| 3.7 | **CLEANUP Step**: Update project status, move files to storage, clean temp | Step 3.6 |
| 3.8 | **Video Player UI**: In-browser MP4 player, download button | Step 3.7 |
| 3.9 | **Generation Progress UI**: Animated progress bar, step status list, cancel | Step 2.10 |
| 3.10 | **End-to-End Test**: Full pipeline: create project → upload → generate → preview | All above |

### Sprint 4: Polish & Launch Prep (Week 7-8)

| Step | Description | Dependencies |
|------|-------------|-------------|
| 4.1 | **Credit System**: Admin panel to grant credits, credit checking in generate endpoint | Step 2.7 |
| 4.2 | **Error States**: Loading skeletons, empty states, error boundaries, toast notifications | None |
| 4.3 | **Form Validation**: Client-side (Zod) + server-side, inline error messages | None |
| 4.4 | **Responsive Design**: Mobile layout for dashboard, project wizard, billing | None |
| 4.5 | **Soft Delete**: Projects list filtered, deletion confirmation dialog | Step 2.1 |
| 4.6 | **WebSocket Polish**: Reconnection handling, multi-tab sync, loading states | Step 2.10 |
| 4.7 | **Provider Swap Test**: Verify changing SCRIPT_PROVIDER env var works | Step 2.8 |

# NOTE: Stripe / Billing / Subscriptions removed from MVP.
# Admin manually grants credits via DB: UPDATE user SET credits = credits + N WHERE email = '...'
# Add Stripe only after first paying customer.

### Sprint 5: Production Readiness (Week 9-10)

| Step | Description | Dependencies |
|------|-------------|-------------|
| 5.1 | **Nginx Config**: Reverse proxy config, SSL via Let's Encrypt, static file serving | None |
| 5.2 | **PM2 Setup**: Production ecosystem config, auto-start on boot, log rotation | None |
| 5.3 | **Error Tracking**: Sentry/Raygun integration for API + Frontend | None |
| 5.4 | **Bull Board**: Install `@bull-board`, add admin route `/admin/queues` with auth | None |
| 5.5 | **Rate Limiting**: Token bucket per user/IP on generate endpoint | None |
| 5.5 | **Security Audit**: CORS, Helmet headers, SQL injection (Prisma safe), upload validation | None |
| 5.6 | **Performance**: Redis caching for templates, image optimization, DB query optimization | None |
| 5.7 | **Load Testing**: k6/artillery script for generation pipeline, tune concurrency | None |
| 5.8 | **Deployment Script**: One-command deploy via bash script or GitHub Actions | None |
| 5.9 | **Documentation**: API docs, architecture overview, deployment guide | None |
| 5.10 | **Beta Launch**: Invite first 10 users, monitor, fix issues | All above |

---

## 14. Decisions & Rationale

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Storage | Local filesystem (MVP) → S3 (post-MVP) | Zero cloud dependency until revenue proves need. All code goes through `StorageProvider` — no raw `fs` calls |
| Database | MySQL (as specified) | Already installed, no extra setup |
| Queues | 2 BullMQ queues (`generation`, `render`) with named jobs | Job names (`analyze`, `script`, `voice`, etc.) eliminate `switch(step)` boilerplate |
| AI Model | GPT-4o-mini via `ScriptProvider` interface | 30x cheaper than GPT-4o. Swap to Claude/Gemini by changing `.env` |
| TTS | OpenAI TTS via `VoiceProvider` interface | 20x cheaper than ElevenLabs. Swap by changing `.env` |
| Real-time | WebSocket (not SSE) | NestJS native support. BullMQ events → emit naturally. Multi-tab. Future admin panel |
| Auth | NestJS JWT + refresh token | No external SDK. Self-contained. HttpOnly cookie + Bearer token |
| Monorepo | pnpm workspaces (no Turbo) | Turborepo adds config overhead without proven need |
| Remotion | Library mode in NestJS renderer process (not API process) | CPU-heavy rendering won't block API requests |
| Provider Adapters | Interface-based (Script, Voice, Image, Video) | Swap providers via `.env`. No code changes needed |
| Billing | Removed from MVP | Admin grants credits manually via DB. Add Stripe after first paying customer |
| Credit Deduction | Deduct on job START | Prevents race conditions, refund on failure |
| File Upload | Server-received (not presigned URLs) | Simpler for local storage, no cloud dependency |

---

## 15. Scope Boundaries

### Explicitly Included (MVP)
- User registration and login (NestJS JWT)
- Project CRUD with multi-step creation wizard
- Product image upload (up to 5, max 10MB each)
- 5 video templates (Product Review, UGC Style, Problem/Solution, Flash Sale, Feature Showcase)
- Full generation pipeline: Analyze → Angle → Script → Voice → Subtitle → Render
- Real-time progress tracking via WebSocket
- In-browser video preview
- MP4 download
- Credit system (admin grants credits manually via DB, 10 free credits on signup)
- Admin role (view all projects, retry failed jobs, add credits)
- Email/password authentication (NestJS JWT)
- Responsive dashboard UI

### Explicitly Excluded (Post-MVP)
- AI avatars / talking heads / digital humans
- Image-to-video generation (currently image-in-video)
- Billing / Stripe / subscriptions (admin grants credits manually)
- Team collaboration / shared projects / workspaces
- Template marketplace / community templates
- Multi-language support (English-only MVP)
- White-label / custom branding / custom domains
- Public API for third-party integrations
- Analytics dashboard (project-level only)
- A/B testing of templates
- Direct social media posting
- Thumbnail generation
- Caption/subtitle editing
- AI video editing (trim/cut within platform)

---

## 16. Verification Plan

| # | Check | Tool / Method | Success Criteria |
|---|-------|---------------|------------------|
| 1 | Prisma schema | `npx prisma validate` | Schema compiles without errors |
| 2 | DB migration | `npx prisma db push` | All tables created in MySQL |
| 3 | Auth flow | Register → Login → Session → Logout via API | Session cookie set/cleared correctly |
| 4 | API health | `GET /api/health` | Returns 200 `{ status: 'ok' }` |
| 5 | Project CRUD | Create → Read → Update → Delete project | Full lifecycle works, DB reflects changes |
| 6 | File upload | Upload 5 images via multipart | Files saved to `uploads/`, DB records created |
| 7 | Template list | `GET /api/templates` | Returns 5 templates with config JSON |
| 8 | Generate pipeline | Create project → Upload images → Select template → Generate | All Job records created, queue receives job |
| 9 | AI pipeline step | Check worker logs for LLM call | Analysis + angle + script generated successfully |
| 10 | Voice step | Check worker logs + audio file | Audio file exists and is playable |
| 11 | Subtitle step | Check SRT file | SRT file exists with proper timing |
| 12 | Remotion render | Check renderer logs + video file | MP4 file exists, playable, correct resolution |
| 13 | WebSocket progress | Subscribe to `project:{id}`, listen for events | Events received: analyzing → script → ... → completed |
| 14 | Video download | `GET /api/projects/:id/download` | Returns MP4 file |
| 15 | Credit deduction | Generate video with 0 credits | Returns `INSUFFICIENT_CREDITS` error |
| 16 | Credit refund | Fail a job mid-pipeline | Credits refunded to user |
| 17 | Credit grant | Admin updates credits via DB query | Credits reflected on next page load |
| 18 | Provider swap | Change `SCRIPT_PROVIDER` env, restart worker | Script generation still works with new provider |
| 19 | Nginx proxy | Access via HTTPS | Frontend loads, API responds, files download |
| 20 | PM2 startup | `pm2 status` after server reboot | All 4 processes running |
| 21 | E2E flow | Browser: Register → Login → Create project → Upload → Generate → Preview → Download | Complete user journey works |

---

## 17. Further Considerations (Post-MVP)

1. **S3-compatible storage migration**: `StorageProvider` interface is in place from day 1. Implement `S3StorageProvider` and swap via env var. No business code changes needed.

2. **Provider expansion**: Adding Claude for script or ElevenLabs for voice is a new file + factory entry. No pipeline code changes.

3. **Billing/Stripe integration**: Add `CreditTransaction` and `SubscriptionPlan` models back. Stripe webhook creates `CreditTransaction` on `checkout.session.completed`. Estimated 2-3 days of work once revenue is proven.

4. **Video processing offload**: If Remotion rendering blocks the VPS CPU, spin up a second cheap VPS dedicated to rendering. The render worker connects to the same Redis for BullMQ and same storage via NFS or Rsync.

5. **Template marketplace**: Add `Template.authorId`, `Template.price`, `Template.downloads` to schema. Build template browser with purchase flow. Post-MVP, this requires no schema migration thanks to existing `Template` model.
