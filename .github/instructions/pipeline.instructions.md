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
→ enqueue 'generation' queue, job name: 'analyze'

analyze:  LLM → write to Job.metadata → create next Job → enqueue 'script'
script:   LLM → write to Job.metadata → create next Job → enqueue 'voice'
voice:    TTS → save via StorageProvider → write path to metadata → enqueue 'subtitle'
subtitle: Generate SRT → save via StorageProvider → write path → enqueue 'render'

'render' queue → (separate PM2 process: apps/renderer)
render:   Remotion renderMedia() → save MP4 via StorageProvider → enqueue 'cleanup'
cleanup:  Update Project=COMPLETED → remove temp files → emit WebSocket event
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

## WebSocket Emit
```typescript
@Inject() private generationGateway: GenerationGateway;
// this.generationGateway.emitProgress(projectId, { step: 'analyze', progress: 50 });
```

## Concurrency & Retry
- `generation` queue: concurrency 5, max retries 3 (5s→30s→120s)
- `render` queue: concurrency 1, max retries 2 (30s→300s)
- On final failure: mark Project FAILED, refund credits
