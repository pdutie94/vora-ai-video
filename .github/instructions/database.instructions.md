---
name: vora-database
description: >-
  Agent for Prisma schema, migrations, and database operations. Knows all models,
  enums, indexes, and query patterns for Vora AI.
applyTo: "apps/api/prisma/**"
---

# Vora AI — Database Agent

## Models

### User
- id (cuid), email (unique), name?, avatarUrl?, credits (default 0), role ("user"|"admin")
- passwordHash (bcrypt), refreshTokenHash (bcrypt hash of refresh token)
- Relations: projects Project[]
- Indexes: email

### Project
- id (cuid), userId (FK), name, status (ProjectStatus)
- templateId (FK→Template)?, productName?, productDesc?, metadata (Json)?, deletedAt?
- Relations: user User, assets ProjectAsset[], jobs Job[], template Template?
- Indexes: (userId, status), (userId, createdAt)

### ProjectAsset
- id (cuid), projectId (FK), type (AssetType), filePath, originalName, mimeType, size, order, metadata?
- Indexes: (projectId, type)

### Job
- id (cuid), projectId (FK), type (JobType), status (JobStatus), progress (0-100), error?, metadata (Json)?
- startedAt?, completedAt?
- Indexes: (projectId, type), (projectId, status)

### Template
- id (cuid), name, slug (unique), description?, thumbnailPath?, config (Json), isActive
- Relations: projects Project[]

## Enums
- ProjectStatus: PENDING, PROCESSING, COMPLETED, FAILED
- JobType: ANALYZE_PRODUCT, GENERATE_ANGLE, GENERATE_SCRIPT, GENERATE_VOICE, GENERATE_SUBTITLE, RENDER_VIDEO, CLEANUP
- JobStatus: PENDING, ACTIVE, COMPLETED, FAILED, CANCELLED
- AssetType: IMAGE, VIDEO, AUDIO, SUBTITLE, SCRIPT

## Rules
- NO Session/Account tables (JWT auth, not Better Auth)
- NO CreditTransaction/SubscriptionPlan (no billing in MVP)
- Always soft-delete Project via deletedAt (DateTime?), never hard delete
- Job.metadata stores accumulated pipeline data as JSON
- All queries scoped to userId — never trust client-side IDs
