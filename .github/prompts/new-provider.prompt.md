---
name: Add a new provider implementation
description: Add a new AI provider (e.g., Anthropic Claude for script, ElevenLabs for voice) following the ProviderAdapter pattern
---

# Add Provider Implementation

## Steps

1. Create a new file in `apps/api/src/providers/{category}/{name}.provider.ts`
2. Implement the appropriate interface (ScriptProvider / VoiceProvider)
3. Register it in `ProviderFactory` with a new `.env` option
4. Add env var documentation to `.env.example`

**Note**: Only `ScriptProvider` and `VoiceProvider` exist in MVP. Do not create `ImageProvider` or `VideoProvider` until needed.

## Provider Interface Example
```typescript
import { ScriptProvider } from '../interfaces/script.provider';

export class AnthropicScriptProvider implements ScriptProvider {
  constructor(private config: ConfigService) {}

  async analyzeProduct(name: string, description: string): Promise<ProductAnalysis> {
    // Anthropic API call
  }

  async generateMarketingAngle(analysis: ProductAnalysis): Promise<string> {
    // Anthropic API call
  }

  async generateScript(angle: string, product: ProductInfo): Promise<ScriptLine[]> {
    // Anthropic API call
  }
}
```

## Factory Registration
In `ProviderFactory.getScriptProvider()`:
```typescript
case 'anthropic': return new AnthropicScriptProvider(this.config);
```

## .env
```
SCRIPT_PROVIDER=anthropic
```
