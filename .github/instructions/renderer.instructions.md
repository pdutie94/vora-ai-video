---
name: vora-renderer
description: >-
  Agent for the Remotion renderer process. Knows template structure, scene components,
  rendering config, and how to connect to BullMQ from the separate renderer app.
applyTo: "apps/renderer/**"
---

# Vora AI — Renderer Agent

## Architecture
- Lives in `apps/renderer/` with its own `package.json`
- Not part of NestJS worker — completely independent PM2 process
- Imports: `@remotion/renderer`, `@prisma/client`, `bullmq`, shared types
- Connects to same Redis (BullMQ) + MySQL (Prisma)

## Rendering Flow
```
'render' queue job received
→ Read accumulated data from DB (Job.metadata)
→ Build composition input props
→ Call renderMedia({ compositionId, inputProps, ... })
→ MP4 written to temp → save via StorageProvider → update DB → enqueue 'cleanup'
```

## Template Structure
```
apps/renderer/src/templates/
├── registry.ts              # Maps slug → Composition
├── product-review.tsx       # Hook → Showcase → Verdict → CTA (30s)
├── ugc-style.tsx            # Hook → Unboxing → Up close → CTA (20s)
├── problem-solution.tsx     # Problem → Solution → Features → CTA (25s)
├── flash-sale.tsx           # Timer → Offer → Product → CTA (15s)
└── features-showcase.tsx    # Intro → Feature 1 → Feature 2 → Outro (40s)

scenes/
├── title-scene.tsx          # Animated title card
├── image-scene.tsx          # Product image + overlay
├── text-scene.tsx           # Script line display
├── cta-scene.tsx            # Call-to-action card
└── outro-scene.tsx          # Brand/closing card
```

## Rendering Config
| Parameter | Value |
|-----------|-------|
| Resolution | 1080×1920 (9:16) |
| FPS | 30 |
| Codec | h264 |
| Video bitrate | 8 Mbps |
| Audio codec | aac |
| Duration | 15-60s |

## Input Props (all templates)
```typescript
interface CompositionProps {
  productName: string;
  productDescription: string;
  marketingAngle: string;
  script: string[];
  images: string[];       // File paths (local or S3)
  voiceOverPath: string;  // Audio file path
  subtitleSrt: string;    // SRT file path
}
```
