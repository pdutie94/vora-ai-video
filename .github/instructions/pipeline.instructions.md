---
name: vora-pipeline
description: >-
  Agent for video generation pipeline code — knows the job flow, provider adapters,
  database schema, and WebSocket event system for the generation pipeline.
applyTo: "apps/api/src/workers/**"
---

# Vora AI — Pipeline Worker Agent

You implement and maintain the video generation pipeline workers.

## Pipeline Flow
```
generate request → check credits → deduct → create Job records
→ enqueue 'video-pipeline' queue, job name: 'analyze'

analyze:  LLM → write to Job.metadata → create next Job → enqueue 'script'
script:   LLM → write to Job.metadata → create next Job → enqueue 'voice'
voice:    TTS → save via StorageProvider → write path to metadata → enqueue 'subtitle'
subtitle: Generate SRT → save via StorageProvider → write path → enqueue 'render'
render:   Remotion renderMedia() → save MP4 via StorageProvider → enqueue 'cleanup'
cleanup:  Update Project=COMPLETED → remove temp files
```

## Job Data Pattern
```typescript
interface JobData {
  projectId: string;
  jobId: string;  // ID of the Job record in DB
}
```
All state reads/writes go through `prisma.job.update({ where: { id: jobId }, data: { metadata } })`.

## Provider Usage
```typescript
const scriptProvider = providerFactory.getScriptProvider();  // based on .env
const voiceProvider = providerFactory.getVoiceProvider();    // based on .env
```

## Progress Tracking
No WebSocket. Frontend polls `GET /api/projects/:id` every 2 seconds via TanStack Query's `refetchInterval`. No socket connections, no rooms, no reconnection logic.

## Concurrency & Retry
- `video-pipeline` queue: concurrency 2, max retries 3 (5s→30s→120s)
- On final failure: mark Project FAILED, refund credits
