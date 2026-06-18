---
name: vora-renderer
description: >-
  Agent for Remotion rendering code. Knows template structure, scene components,
  rendering config. Rendering runs inline in the NestJS worker, not a separate process.
applyTo: "apps/api/src/renderer/**"
---

# Vora AI — Renderer Agent

## Architecture
- Runs inside the same NestJS process (`apps/api`)
- `@remotion/renderer` package's `renderMedia()` called directly from job handler
- No separate PM2 process — Remotion renders inline in the `video-pipeline` queue worker
- When Remotion crashes (OOM/FFmpeg) → PM2 restarts the API process
- For 14 renders/hr (10k videos/month), single process handles it easily

## Rendering Flow
```
Job 'render' received (same queue as analyze/script/voice/subtitle)
→ Read accumulated data from DB (Job.metadata)
→ Build composition input props
→ Call renderMedia({ compositionId, inputProps, ... })
→ MP4 written to temp → save via StorageProvider → update DB → enqueue 'cleanup'
```

## Template Structure (MVP — 1 Template)
```
apps/api/src/renderer/
├── registry.ts              # Maps slug → Composition
├── templates/
│   └── product-review.tsx   # Hook → Showcase → Verdict → CTA (30s)
└── scenes/
    ├── title-scene.tsx      # Animated title card
    ├── image-scene.tsx      # Product image + overlay
    ├── text-scene.tsx       # Script line display
    ├── cta-scene.tsx        # Call-to-action card
    └── outro-scene.tsx      # Brand/closing card
```

## Rendering Config
| Parameter | Value |
|-----------|-------|
| Resolution | 1080×1920 (9:16) |
| FPS | 30 |
| Codec | h264 |
| Video bitrate | 8 Mbps |
| Audio codec | aac |
| Duration | 30s (1 template) |

## Input Props
```typescript
interface CompositionProps {
  productName: string;
  productDescription: string;
  marketingAngle: string;
  script: string[];
  images: string[];       // File paths (local)
  voiceOverPath: string;  // Audio file path
  subtitleSrt: string;    // SRT file path
}
```
