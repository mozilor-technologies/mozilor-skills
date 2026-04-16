---
name: nodejs-backend
description: Node.js scalable backend best practices, architecture patterns, and production standards for 2026.
triggers:
  - "node.js"
  - "nodejs backend"
  - "express backend"
  - "node api"
  - "backend patterns"
  - "scalable backend"
  - "node architecture"
---

# Node.js Backend — Best Practices & Scalable Architecture

## 1) Architecture Selection

Choose your architecture based on scale:

| Scale | Architecture | When |
|---|---|---|
| MVP / Small | Layered Monolith | Single team, predictable load, simple domain |
| Growing | Modular Monolith (feature-based) | Multiple domain areas, growing team |
| Large | Microservices | Independent scaling, separate deployment needs |
| Spiky traffic | Serverless | Unpredictable, bursty load (AWS Lambda) |
| Real-time | Event-Driven | Chat, IoT, trading, notification systems |

## 2) Project Structure

Prefer **feature-based** over layer-based for anything beyond simple CRUD.

```
src/
├── modules/
│   ├── auth/
│   │   ├── auth.router.ts
│   │   ├── auth.service.ts
│   │   ├── auth.repository.ts
│   │   ├── auth.schema.ts
│   │   └── auth.test.ts
│   ├── users/
│   └── orders/
├── shared/
│   ├── middleware/
│   ├── errors/
│   ├── config/
│   └── utils/
├── infrastructure/
│   ├── database/
│   ├── cache/
│   └── queue/
└── main.ts
```

Why feature-based? Reduces merge conflicts, keeps related code together, scales naturally with teams.

## 3) Core Architecture Patterns

### Layered Separation of Concerns

```
Router → Controller → Service → Repository → Database
```

- **Router**: Route definitions only, no logic
- **Controller**: HTTP in/out, validation, response shaping
- **Service**: Business logic — must be framework-agnostic (no Express imports)
- **Repository**: All database access, no business logic

Framework-agnostic services enable unit testing without HTTP mocks and allow framework swapping.

### Error Handling

Use custom error classes to catch all errors predictably:

```typescript
class AppError extends Error {
  constructor(
    public message: string,
    public statusCode: number,
    public code: string
  ) {
    super(message);
  }
}

class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 404, 'NOT_FOUND');
  }
}

// Global error handler middleware
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      error: { code: err.code, message: err.message }
    });
  }
  // Never leak internal details
  return res.status(500).json({ error: { code: 'INTERNAL', message: 'Internal server error' } });
});
```

### Circuit Breaker Pattern

Protect against cascading failures when calling external services:

```typescript
import CircuitBreaker from 'opossum';

const breaker = new CircuitBreaker(callExternalService, {
  timeout: 3000,
  errorThresholdPercentage: 50,
  resetTimeout: 30000,
});

breaker.fallback(() => cachedResponse);
breaker.on('open', () => logger.warn('Circuit breaker opened'));
```

## 4) Asynchronous Processing

Never block the request-response cycle with heavy operations:

```typescript
// BAD: blocks request
app.post('/send-email', async (req, res) => {
  await sendEmailDirectly(req.body);
  res.json({ success: true });
});

// GOOD: queue the work
app.post('/send-email', async (req, res) => {
  await emailQueue.add('send', req.body);
  res.json({ queued: true });
});
```

Use Bull/BullMQ with Redis for job queues. Use event emitters for lightweight in-process events.

## 5) Multi-Layer Caching

```typescript
async function getProduct(id: string): Promise<Product> {
  // L1: in-memory
  const memCached = memoryCache.get(id);
  if (memCached) return memCached;

  // L2: Redis
  const redisCached = await redis.get(`product:${id}`);
  if (redisCached) {
    const parsed = JSON.parse(redisCached);
    memoryCache.set(id, parsed, 60);
    return parsed;
  }

  // L3: database
  const product = await db.products.findById(id);
  await redis.setex(`product:${id}`, 300, JSON.stringify(product));
  memoryCache.set(id, product, 60);
  return product;
}
```

## 6) API Gateway Pattern

Centralize cross-cutting concerns:

```typescript
// Rate limiting, auth, logging at the gateway level
app.use(rateLimit({ windowMs: 60_000, max: 100 }));
app.use(authenticate);
app.use(requestLogger);
app.use('/api/v1', apiRouter);
```

## 7) Configuration Management

```typescript
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
});

export const config = envSchema.parse(process.env); // Fails fast on invalid config
```

## 8) Logging

Use structured JSON logging — never `console.log` in production:

```typescript
import pino from 'pino';

export const logger = pino({
  level: config.LOG_LEVEL,
  serializers: { err: pino.stdSerializers.err },
  redact: ['req.headers.authorization', 'body.password'],
});
```

Always include: `requestId`, `userId`, `service`, `duration`.

## 9) Security Checklist

- [ ] Validate all inputs with Zod or Joi at controller boundaries
- [ ] Sanitize all outputs — never return raw DB objects
- [ ] Use Helmet for HTTP security headers
- [ ] Rate limit all public endpoints
- [ ] Implement CORS with explicit origins
- [ ] Use parameterized queries — never string-concatenate SQL
- [ ] Hash passwords with bcrypt (cost factor ≥ 12)
- [ ] Store secrets in environment variables or vault, never in code
- [ ] JWT: short-lived access tokens + refresh token rotation
- [ ] Audit all authentication and authorization events

## 10) Performance Checklist

- [ ] Use async/await consistently — no callback patterns
- [ ] Implement connection pooling for DB (pg-pool, mongoose poolSize)
- [ ] Add pagination to all list endpoints
- [ ] Use streaming for large data responses
- [ ] Cluster mode or PM2 to use all CPU cores
- [ ] Health check endpoints at `/health` and `/ready`
- [ ] Graceful shutdown: drain connections before exit

## 11) Testing Standards

```
tests/
├── unit/          # Pure function and service tests, no I/O
├── integration/   # DB + service layer with real DB (test container)
└── e2e/           # Full HTTP request cycle
```

Coverage targets: Unit ≥ 80%, Integration ≥ 60%, E2E critical paths only.

```bash
# Test commands
npm test              # unit tests
npm run test:int      # integration tests
npm run test:e2e      # end-to-end tests
npm run test:cov      # coverage report
```

## 12) Quality Gates

Before every PR:
1. `npm run lint` — ESLint with `@typescript-eslint` rules
2. `npm test` — full test suite green
3. `npm run build` — TypeScript compilation succeeds
4. No secrets in diff (`git diff --staged | grep -i 'secret\|key\|password'`)
5. Dependencies audited (`npm audit --audit-level=high`)

## References

- [Node.js Best Practices (goldbergyoni)](https://github.com/goldbergyoni/nodebestpractices)
- [Node.js Architecture Patterns 2026](https://dev.to/kafeel-ahmad/nodejs-architecture-patterns-for-scalable-apps-2026-guide-f3h)
- [Production-Ready Backend Folder Structure 2026](https://dev.to/akshaykurve/designing-a-production-ready-backend-folder-structure-using-nodejs-2026-edition-3a8k)
