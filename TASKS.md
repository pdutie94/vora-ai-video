# Vora AI — Task Tracker

> **Mục tiêu**: MVP trong 6 tuần. 100 paying customers. 10,000 videos/month.
> **Kiến trúc**: 2 PM2 processes, 1 BullMQ queue, REST polling, JWT 7d, 1 Remotion template.

**Legend**: ⬜ Not started | 🔄 In progress | ✅ Done | ❌ Blocked

---

## Sprint 1: Foundation (Week 1-2)

### 1.1 Monorepo Setup
- [ ] ⬜ Initialize root `package.json` with pnpm workspaces
- [ ] ⬜ Create `pnpm-workspace.yaml` (apps/web, apps/api, packages/shared)
- [ ] ⬜ Add `.gitignore` (node_modules, .next, dist, uploads, storage, .env)
- [ ] ⬜ Add `.prettierrc` + `.eslintrc.js`
- [ ] ⬜ Write `ecosystem.config.js` (PM2 — 2 processes: vora-api, vora-web)
- [ ] ⬜ Write `README.md`

### 1.2 Shared Package
- [ ] ⬜ Create `packages/shared/` with TypeScript config
- [ ] ⬜ Define domain types: `Project`, `Job`, `ProjectAsset`
- [ ] ⬜ Define constants: enums (`ProjectStatus`, `JobType`, `JobStatus`, `AssetType`)

### 1.3 NestJS Scaffold
- [ ] ⬜ Run `nest new` or manually scaffold `apps/api`
- [ ] ⬜ Create `AppModule` + health endpoint `GET /api/health`
- [ ] ⬜ Configure `@nestjs/config` with `.env`
- [ ] ⬜ Add global exception filter
- [ ] ⬜ Add logging interceptor

### 1.4 Next.js Scaffold
- [ ] ⬜ Run `create-next-app` in `apps/web` (TypeScript, App Router, Tailwind)
- [ ] ⬜ Initialize shadcn/ui (`npx shadcn@latest init`)
- [ ] ⬜ Add base layout + globals.css
- [ ] ⬜ Create API client wrapper (`lib/api-client.ts`)

### 1.5 Prisma Schema
- [ ] ⬜ Write `schema.prisma` with 4 models:
  - `User` (id, email, name, credits, role, passwordHash)
  - `Project` (id, userId, name, status, productName, productDesc, metadata?)
  - `ProjectAsset` (id, projectId, type, filePath, originalName, mimeType, size, order)
  - `Job` (id, projectId, type, status, progress, error?, metadata?, startedAt?, completedAt?)
- [ ] ⬜ Run `prisma generate` + `prisma db push`

### 1.6 JWT Auth
- [ ] ⬜ Install `@nestjs/jwt` + `@nestjs/passport` + `passport` + `passport-jwt`
- [ ] ⬜ Create `auth.module.ts` + `auth.controller.ts`
- [ ] ⬜ Implement `POST /api/auth/register` (email + password, hash with bcryptjs)
- [ ] ⬜ Implement `POST /api/auth/login` (returns JWT, 7-day expiry)
- [ ] ⬜ Implement `GET /api/auth/me` (returns current user)
- [ ] ⬜ Create `JwtAuthGuard` + `@CurrentUser()` decorator

### 1.7 NestJS Structure
- [ ] ⬜ Set up global validation pipe
- [ ] ⬜ Add `helmet` middleware
- [ ] ⬜ Configure CORS for frontend URL

### 1.8 Next.js Dashboard
- [ ] ⬜ Create dashboard layout (sidebar + topbar)
- [ ] ⬜ Create auth pages: `/auth/login`, `/auth/register`
- [ ] ⬜ Create auth client (`lib/auth-client.ts`)
- [ ] ⬜ Add TanStack Query provider

### 1.9 BullMQ Setup
- [ ] ⬜ Install `bullmq` + `ioredis`
- [ ] ⬜ Create queue `video-pipeline`
- [ ] ⬜ Create `queue.service.ts` (enqueue, get job status)
- [ ] ⬜ Scaffold named job handlers: `analyze`, `script`, `voice`, `subtitle`, `render`, `cleanup`

### 1.10 Project Foundation
- [ ] ⬜ Write initial `ecosystem.config.js`
- [ ] ⬜ Write `README.md` with dev commands
- [ ] ⬜ Test: `pnpm dev` starts both web + api

---

## Sprint 2: Core Features (Week 3-4)

### 2.1 Projects CRUD (API)
- [ ] ⬜ Create `projects.module.ts` + controller + service
- [ ] ⬜ `GET /api/projects` — list user's projects (paginated)
- [ ] ⬜ `POST /api/projects` — create new project
- [ ] ⬜ `GET /api/projects/:id` — get project with assets + jobs
- [ ] ⬜ `PATCH /api/projects/:id` — update project info
- [ ] ⬜ `DELETE /api/projects/:id` — hard delete

### 2.2 Projects UI
- [ ] ⬜ Dashboard page with project grid (`/dashboard`)
- [ ] ⬜ Project card component (thumbnail, name, status badge, date)
- [ ] ⬜ Create project wizard — Step 1: Product info form
- [ ] ⬜ Empty state when no projects

### 2.3 File Upload (API)
- [ ] ⬜ Configure multer for multipart uploads
- [ ] ⬜ Validate file type (jpg, png, webp) + size (10MB max)
- [ ] ⬜ Save to `uploads/{userId}/{projectId}/images/`
- [ ] ⬜ `POST /api/projects/:id/assets` — upload product image
- [ ] ⬜ `GET /api/projects/:id/assets` — list assets
- [ ] ⬜ `DELETE /api/projects/:id/assets/:assetId` — remove asset

### 2.4 Image Uploader UI
- [ ] ⬜ Drag-and-drop file dropzone
- [ ] ⬜ Image preview after upload
- [ ] ⬜ Upload progress indicator
- [ ] ⬜ Max 5 images per project

### 2.5 Template System
- [ ] ⬜ Create `templates.module.ts` + controller + service
- [ ] ⬜ Hardcode 1 template (product-review) — no DB table
- [ ] ⬜ `GET /api/templates` — list available templates
- [ ] ⬜ `GET /api/templates/:slug` — get template details

### 2.6 Template Selector UI
- [ ] ⬜ Template card with thumbnail + name + duration
- [ ] ⬜ Step 3 of wizard: template selection

### 2.7 Generate Endpoint
- [ ] ⬜ `POST /api/projects/:id/generate`
- [ ] ⬜ Validate credits (min 5 credits)
- [ ] ⬜ Deduct credits on job start
- [ ] ⬜ Create Job records in DB
- [ ] ⬜ Enqueue first job ('analyze') to BullMQ

### 2.8 AI Service (ScriptProvider)
- [ ] ⬜ Install `openai` package
- [ ] ⬜ Create `ScriptProvider` interface
- [ ] ⬜ Implement `OpenAIScriptProvider` (GPT-4o-mini)
- [ ] ⬜ Implement `analyzeProduct()` — LLM prompt
- [ ] ⬜ Implement `generateMarketingAngle()` — LLM prompt
- [ ] ⬜ Implement `generateScript()` — LLM prompt
- [ ] ⬜ Create `ProviderFactory` with `.env` switch

### 2.9 Pipeline Worker (analyze → script → voice)
- [ ] ⬜ Implement `worker.process('analyze')` — calls ScriptProvider
- [ ] ⬜ Implement `worker.process('script')` — calls ScriptProvider
- [ ] ⬜ Implement `worker.process('voice')` — calls VoiceProvider (OpenAI TTS)
- [ ] ⬜ Wire up StorageProvider for audio file saving
- [ ] ⬜ Write pipeline state to `Job.metadata` after each step
- [ ] ⬜ Enqueue next job with `{ projectId, nextJobId }`

### 2.10 Progress Polling
- [ ] ⬜ TanStack Query `refetchInterval: 2000` on project detail
- [ ] ⬜ Handle PROCESSING / COMPLETED / FAILED states in UI
- [ ] ⬜ Show status badge changes in real-time

---

## Sprint 3: Video Pipeline (Week 5)

### 3.1 Voice Generation
- [ ] ⬜ Implement `VoiceProvider` interface
- [ ] ⬜ Implement `OpenAIVoiceProvider` (tts-1 model, alloy voice)
- [ ] ⬜ Save audio file via StorageProvider
- [ ] ⬜ Write voice file path to `Job.metadata`

### 3.2 Subtitle Generation
- [ ] ⬜ Generate SRT from script lines + voice duration
- [ ] ⬜ Save SRT file via StorageProvider
- [ ] ⬜ Write subtitle path to `Job.metadata`

### 3.3 Remotion Setup
- [ ] ⬜ Install `remotion` + `@remotion/renderer`
- [ ] ⬜ Create `renderer.service.ts` — wraps `renderMedia()`
- [ ] ⬜ Create template registry (`registry.ts`)

### 3.4 Template Composition
- [ ] ⬜ Build `ProductReview.tsx` — Hook → Showcase → Verdict → CTA
- [ ] ⬜ Build scene components:
  - [ ] `TitleScene.tsx` — animated title card
  - [ ] `ImageScene.tsx` — product image + overlay
  - [ ] `TextScene.tsx` — script line display
  - [ ] `CTAScene.tsx` — call-to-action card
  - [ ] `OutroScene.tsx` — brand/closing card
- [ ] ⬜ Wire up input props from `Job.metadata`

### 3.5 Render Handler
- [ ] ⬜ Implement `worker.process('render')`:
  - Read all accumulated data from DB
  - Call `renderMedia()` with composition + props
  - Save MP4 via StorageProvider
  - Update DB → enqueue 'cleanup'
- [ ] ⬜ Set rendering config: 1080×1920, 30fps, h264, 8Mbps

### 3.6 Cleanup Handler
- [ ] ⬜ Implement `worker.process('cleanup')`:
  - `UPDATE Project.status = 'COMPLETED'`
  - Remove temp files
  - Refund credits on failure

### 3.7 Video Player UI
- [ ] ⬜ In-browser MP4 player component
- [ ] ⬜ Download button for rendered MP4
- [ ] ⬜ `GET /api/projects/:id/download` endpoint
- [ ] ⬜ Show video only when `Project.status === 'COMPLETED'`

### 3.8 Generation Progress UI
- [ ] ⬜ Animated progress bar (polls project status every 2s)
- [ ] ⬜ Step-by-step status list (analyzing ✓ → script ✓ → voice ⟳ → ...)
- [ ] ⬜ Cancel button (optional for MVP)

### 3.9 End-to-End Test
- [ ] ⬜ Full flow: Register → Create project → Upload images → Generate → Preview → Download
- [ ] ⬜ Debug: verify all 6 named jobs execute in order
- [ ] ⬜ Debug: verify MP4 is playable with correct resolution

---

## Sprint 4: Polish & Launch Prep (Week 6)

### 4.1 Credit System
- [ ] ⬜ Credit checking in generate endpoint (min 5 credits)
- [ ] ⬜ Deduct credits on job start, refund on failure
- [ ] ⬜ Admin: `UPDATE user SET credits = credits + N` (manual via DB)
- [ ] ⬜ `GET /api/users/me` returns `credits` balance

### 4.2 Error States
- [ ] ⬜ Loading skeletons for project list, detail, wizard
- [ ] ⬜ Empty state illustrations (no projects, no assets)
- [ ] ⬜ Error boundaries on dashboard + project pages
- [ ] ⬜ Toast notifications for success/error actions

### 4.3 Form Validation
- [ ] ⬜ Client-side validation with Zod schemas
- [ ] ⬜ Server-side validation with class-validator
- [ ] ⬜ Inline error messages on forms
- [ ] ⬜ Disable generate button when credits insufficient

### 4.4 Responsive Design
- [ ] ⬜ Mobile layout for dashboard (sidebar → hamburger menu)
- [ ] ⬜ Mobile layout for project wizard (full-width steps)
- [ ] ⬜ Mobile layout for project detail (video full-width)

### 4.5 Provider Swap Test
- [ ] ⬜ Verify `SCRIPT_PROVIDER=gemini` works (if API key available)
- [ ] ⬜ Verify `SCRIPT_PROVIDER=openai` works
- [ ] ⬜ Verify `VOICE_PROVIDER=openai` works
- [ ] ⬜ Document env vars in `.env.example`

---

## Sprint 5: Launch (Week 6+)

### 5.1 Nginx Config
- [ ] ⬜ Reverse proxy config: `/api/` → `localhost:4000`
- [ ] ⬜ SSL via Let's Encrypt (certbot)
- [ ] ⬜ Static file serving for Next.js public assets
- [ ] ⬜ Internal file serving for `/files/` (local storage)

### 5.2 PM2 Setup
- [ ] ⬜ `ecosystem.config.js` with 2 processes
- [ ] ⬜ Auto-start on boot (`pm2 startup`)
- [ ] ⬜ Log rotation (`pm2-logrotate`)
- [ ] ⬜ Max memory restart limits

### 5.3 Error Tracking
- [ ] ⬜ Install Sentry (or alternative) for API + Frontend
- [ ] ⬜ Source map upload for stack traces

### 5.4 Rate Limiting
- [ ] ⬜ Token bucket per user/IP on `/generate` endpoint
- [ ] ⬜ Configurable limit via `.env`

### 5.5 Security Audit
- [ ] ⬜ Helmet headers configured
- [ ] ⬜ CORS whitelist (only frontend URL)
- [ ] ⬜ Request size limiting (body-parser)
- [ ] ⬜ File upload validation (type, size, scan prep)

### 5.6 Performance
- [ ] ⬜ Redis caching for template registry
- [ ] ⬜ Image optimization (sharp.js for thumbnails)
- [ ] ⬜ DB query optimization (check EXPLAIN for slow queries)

### 5.7 Load Testing
- [ ] ⬜ k6 or artillery script for generation pipeline
- [ ] ⬜ Test with concurrent generation requests
- [ ] ⬜ Tune BullMQ concurrency (start at 2, adjust)

### 5.8 Deployment Script
- [ ] ⬜ One-command deploy: `bash deploy.sh`
  - `git pull`
  - `pnpm install`
  - `pnpm --filter @vora/api exec prisma generate`
  - `pnpm --filter @vora/api exec prisma db push`
  - `pnpm build`
  - `pm2 restart ecosystem.config.js`

### 5.9 Documentation
- [ ] ⬜ API endpoints doc (Postman collection or README)
- [ ] ⬜ Architecture overview (refer to `plan.md`)
- [ ] ⬜ Deployment guide (server setup, env vars, PM2)

### 5.10 Beta Launch
- [ ] ⬜ Invite first 10 users
- [ ] ⬜ Monitor error logs daily
- [ ] ⬜ Collect feedback on video quality + UX
- [ ] ⬜ Fix critical bugs within 24h

---

## Post-MVP Backlog

These are explicitly removed from MVP scope. Add only when customers request or revenue justifies.

- [ ] 📋 **WebSocket upgrade**: Swap REST polling for socket.io when latency becomes an issue
- [ ] 📋 **More templates**: Add UGC Style, Problem/Solution, Flash Sale, Feature Showcase on demand
- [ ] 📋 **Stripe billing**: Add `CreditTransaction` + `SubscriptionPlan` models, webhooks, checkout flow
- [ ] 📋 **S3 storage**: Implement `S3StorageProvider` for R2/Backblaze/MinIO
- [ ] 📋 **Refresh tokens**: Add refresh token rotation for better security at scale
- [ ] 📋 **Separate renderer**: Extract Remotion to separate process if CPU becomes bottleneck
- [ ] 📋 **ImageProvider + VideoProvider**: Add interfaces when image/video generation features are built
- [ ] 📋 **Template marketplace**: Add Template model, author system, marketplace UI
- [ ] 📋 **Bull Board**: Queue dashboard for debugging production issues
- [ ] 📋 **@nestjs/swagger**: API docs when public API is launched
- [ ] 📋 **Multi-language**: i18n support for non-English markets
- [ ] 📋 **Team collaboration**: Shared workspaces, multi-user projects

---

## Notes

- **Commit convention**: `type: description` (e.g., `feat: add JWT auth`, `fix: subtitle timing off by 1s`)
- **Branch strategy**: Work on `main` until first beta, then switch to `main` + feature branches
- **Testing**: Manual E2E testing for MVP. Add unit tests only for critical business logic
- **DB changes**: Always run `prisma generate` + `prisma db push` after schema changes
