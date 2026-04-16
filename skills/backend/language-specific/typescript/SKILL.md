---
name: typescript
description: TypeScript best practices, strict type safety, utility types, and production-grade patterns for 2026.
triggers:
  - "typescript"
  - "ts best practices"
  - "type safety"
  - "typescript patterns"
  - "strict typescript"
  - "ts config"
---

# TypeScript — Best Practices & Production Patterns

## 1) Strict Configuration (tsconfig.json)

Start every project with maximum strictness:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noPropertyAccessFromIndexSignature": true,
    "exactOptionalPropertyTypes": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": false,
    "esModuleInterop": true,
    "resolveJsonModule": true
  }
}
```

Enable strict from day one — retrofitting it into an existing codebase is painful.

## 2) Type Safety Rules

### Never Use `any` — Use `unknown` Instead

```typescript
// BAD: disables all type checking
function process(data: any) {
  data.name.toUpperCase(); // no error, crashes at runtime
}

// GOOD: forces narrowing before use
function process(data: unknown) {
  if (typeof data === 'object' && data !== null && 'name' in data) {
    const name = (data as { name: string }).name;
    name.toUpperCase(); // safe
  }
}
```

Reserve `any` only for genuine escape hatches, always with a comment explaining why.

### Discriminated Unions for State Modeling

```typescript
// BAD: optional fields allow impossible states
type ApiResponse = {
  data?: User;
  error?: string;
  loading?: boolean;
};

// GOOD: discriminated union — impossible states are unrepresentable
type ApiResponse =
  | { status: 'loading' }
  | { status: 'success'; data: User }
  | { status: 'error'; error: string };

function render(response: ApiResponse) {
  switch (response.status) {
    case 'loading': return 'Loading...';
    case 'success': return response.data.name; // TypeScript knows data exists
    case 'error': return response.error; // TypeScript knows error exists
  }
}
```

### Template Literal Types for Strict Strings

```typescript
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
type ApiPath = `/api/v${number}/${string}`;
type EventName = `on${Capitalize<string>}`;
```

## 3) Interface vs Type Alias

Use **interfaces** for object shapes — they give clearer error messages and support declaration merging:

```typescript
interface User {
  id: string;
  email: string;
  createdAt: Date;
}

interface AdminUser extends User {
  permissions: Permission[];
}
```

Use **type aliases** for unions, intersections, mapped types, and utility compositions:

```typescript
type UserId = string;
type Result<T> = { data: T; meta: PaginationMeta };
type CreateUserDto = Omit<User, 'id' | 'createdAt'>;
type UpdateUserDto = Partial<Pick<User, 'email' | 'name'>>;
```

## 4) Utility Types — Production Patterns

```typescript
// Request/response shaping
type CreateProductRequest = Omit<Product, 'id' | 'createdAt' | 'updatedAt'>;
type UpdateProductRequest = Partial<Pick<Product, 'name' | 'price' | 'stock'>>;
type ProductSummary = Pick<Product, 'id' | 'name' | 'price'>;

// Nullable handling
type Nullable<T> = T | null;
type Optional<T> = T | undefined;

// Deep readonly for immutable configs
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

// Require specific keys
type RequireKeys<T, K extends keyof T> = T & Required<Pick<T, K>>;

// Extract keys of a specific value type
type KeysOfType<T, V> = { [K in keyof T]: T[K] extends V ? K : never }[keyof T];
```

## 5) Runtime Validation with Zod

TypeScript types disappear at runtime. Always validate external data (API bodies, env vars, external responses):

```typescript
import { z } from 'zod';

const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(2).max(100),
  role: z.enum(['admin', 'user', 'guest']).default('user'),
});

type CreateUserDto = z.infer<typeof CreateUserSchema>;

// In controller
const result = CreateUserSchema.safeParse(req.body);
if (!result.success) {
  return res.status(400).json({ errors: result.error.flatten() });
}
const dto: CreateUserDto = result.data; // fully typed and validated
```

## 6) Generic Patterns

```typescript
// Repository pattern with generics
interface Repository<T extends { id: string }> {
  findById(id: string): Promise<T | null>;
  findAll(filter?: Partial<T>): Promise<T[]>;
  create(data: Omit<T, 'id'>): Promise<T>;
  update(id: string, data: Partial<T>): Promise<T | null>;
  delete(id: string): Promise<boolean>;
}

// Service result type
type ServiceResult<T> =
  | { success: true; data: T }
  | { success: false; error: string; code: string };

async function createUser(dto: CreateUserDto): Promise<ServiceResult<User>> {
  try {
    const user = await userRepo.create(dto);
    return { success: true, data: user };
  } catch (e) {
    return { success: false, error: 'Creation failed', code: 'USER_CREATE_FAIL' };
  }
}
```

## 7) Async Patterns

```typescript
// Always type Promise returns explicitly in public APIs
async function fetchUser(id: string): Promise<User> { ... }

// Use Promise.all for parallel independent operations
const [user, orders, permissions] = await Promise.all([
  userRepo.findById(id),
  orderRepo.findByUser(id),
  permissionRepo.findByUser(id),
]);

// Use Promise.allSettled when partial failure is acceptable
const results = await Promise.allSettled(items.map(processItem));
const successes = results.filter(r => r.status === 'fulfilled');
```

## 8) Declaration Files & Module Augmentation

```typescript
// Extending Express Request with user context
declare global {
  namespace Express {
    interface Request {
      user?: AuthenticatedUser;
      requestId: string;
    }
  }
}

// Environment variable typing
declare global {
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV: 'development' | 'test' | 'production';
      DATABASE_URL: string;
      JWT_SECRET: string;
    }
  }
}
```

## 9) Common Anti-Patterns to Avoid

| Anti-Pattern | Why | Fix |
|---|---|---|
| `as SomeType` (type assertion) | Hides bugs, circumvents type checker | Use type guards or narrowing |
| `!` non-null assertion everywhere | Crashes at runtime | Explicit null checks |
| `any` on function parameters | Defeats the purpose | Use generics or `unknown` |
| Storing secrets in typed constants | Security risk | Use environment variables |
| Not typing async function returns | Inference errors cascade | Explicit return types |
| Giant union types (10+ members) | Hard to maintain | Consider discriminated union with factory |

## 10) ESLint Configuration

```json
{
  "extends": [
    "@typescript-eslint/recommended",
    "@typescript-eslint/recommended-requiring-type-checking"
  ],
  "rules": {
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/no-non-null-assertion": "error",
    "@typescript-eslint/explicit-function-return-type": "warn",
    "@typescript-eslint/no-floating-promises": "error",
    "@typescript-eslint/await-thenable": "error",
    "@typescript-eslint/no-misused-promises": "error"
  }
}
```

## 11) Quality Checklist

- [ ] `strict: true` enabled in tsconfig
- [ ] `noUncheckedIndexedAccess` enabled
- [ ] Zero `any` types (enforce with ESLint rule)
- [ ] All external data validated with Zod at boundaries
- [ ] Public API functions have explicit return types
- [ ] No type assertions without explanation comments
- [ ] Discriminated unions used for state modeling
- [ ] `Promise` returns are always awaited or returned

## References

- [TypeScript Best Practices 2026](https://dev.to/_d7eb1c1703182e3ce1782/typescript-best-practices-for-production-code-in-2026-lb0)
- [Type-Driven Design 2026](https://typescript.page/type-driven-design-2026)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/)
