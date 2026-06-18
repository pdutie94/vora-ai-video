---
name: Create a new API module
description: Scaffold a new NestJS module following Vora AI conventions
---

# Create NestJS Module

Create a new NestJS module at `apps/api/src/modules/{name}/` with:

1. `{name}.module.ts` — @Module with imports, controllers, providers
2. `{name}.controller.ts` — REST controller with @Controller('api/{name}')
3. `{name}.service.ts` — Business logic with constructor DI

## Conventions
- Flat module (no UseCase/Repository layers)
- Service injected via constructor
- Auth guard: `@UseGuards(JwtAuthGuard)` on controller class
- Current user: `@CurrentUser() user: User` decorator on handler
- Error response: throw `new HttpException({ code, message, details }, status)`
- All queries scoped to authenticated user's userId
- Validation via class-validator DTOs in the same module folder
