---
name: security
description: Security best practices covering OWASP patterns, secrets management, secure development, security testing, and security review process.
triggers:
  - "security"
  - "owasp"
  - "security review"
  - "secrets management"
  - "security testing"
  - "secure coding"
  - "vulnerability"
  - "penetration testing"
  - "sast"
  - "authentication security"
  - "authorization"
---

# Security — Best Practices, OWASP Patterns & Security Review

## 1) OWASP Top 10 (2025) — Developer Checklist

### A01: Broken Access Control

```typescript
// WRONG: trust client-supplied IDs
app.get('/orders/:id', async (req, res) => {
  const order = await Order.findById(req.params.id);
  res.json(order);
});

// CORRECT: always scope by authenticated user
app.get('/orders/:id', authenticate, async (req, res) => {
  const order = await Order.findOne({
    id: req.params.id,
    userId: req.user.id,  // Ownership check — never skip
  });
  if (!order) return res.status(404).json({ error: 'Not found' });
  res.json(order);
});
```

Rules:
- Deny by default — require explicit permission grants
- Enforce ownership on every resource operation
- Validate permissions server-side, never trust client role claims
- Log all access control failures

### A02: Cryptographic Failures

```python
# WRONG: MD5/SHA1 for passwords, or plaintext storage
import hashlib
hashed = hashlib.md5(password.encode()).hexdigest()

# CORRECT: bcrypt/argon2 with cost factor
from passlib.hash import argon2
hashed = argon2.hash(password)        # Argon2id recommended
valid  = argon2.verify(password, hashed)

# CORRECT for Node.js
import bcrypt
const hash = await bcrypt.hash(password, 12);  # cost ≥ 12
```

- Never store passwords in plaintext or with reversible encryption
- Use TLS 1.2+ for all connections (prefer 1.3)
- Encrypt sensitive data at rest (PII, financial data)
- Use AES-256-GCM for symmetric encryption
- Use RSA-4096 or Ed25519 for asymmetric

### A03: Injection

```python
# WRONG: string concatenation in SQL
query = f"SELECT * FROM users WHERE email = '{email}'"

# CORRECT: parameterized queries always
query = "SELECT * FROM users WHERE email = $1"
result = await conn.fetch(query, email)

# CORRECT: ORM with parameterized queries
user = await User.objects.get(email=email)
```

```javascript
// WRONG: eval() or dynamic code execution
eval(userInput);
new Function(userInput)();

// WRONG: template injection
const template = `Hello ${userInput}`;  // If userInput contains JS expressions

// CORRECT: sanitize all external input
import DOMPurify from 'dompurify';
const safe = DOMPurify.sanitize(userInput);
```

### A04: Insecure Design

- Threat model every feature before implementation
- Apply defense in depth — multiple layers of controls
- Fail securely: default to most restrictive state on error
- Limit resource consumption (rate limiting, quotas, size limits)

### A05: Security Misconfiguration

```typescript
// Production Helmet configuration
import helmet from 'helmet';

app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc:  ["'self'"],
      styleSrc:   ["'self'"],
      imgSrc:     ["'self'", 'data:', 'https:'],
    },
  },
  hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
  noSniff: true,
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
}));

// Disable stack traces in production
app.use((err, req, res, next) => {
  res.status(err.status || 500).json({
    error: { message: 'Internal server error' }
    // Never: error: { message: err.message, stack: err.stack }
  });
});
```

### A07: Authentication Failures

```typescript
// JWT best practices
const token = jwt.sign(
  { userId: user.id, role: user.role },
  process.env.JWT_SECRET,
  {
    expiresIn: '15m',    // Short-lived access tokens
    algorithm: 'RS256',  // Asymmetric — RS256 > HS256 for distributed systems
    issuer: 'myapp',
    audience: 'myapp-api',
  }
);

// Refresh token rotation
// - Store refresh tokens in DB (allows revocation)
// - Rotate on every use (detect theft)
// - Invalidate on logout or password change

// Brute force protection
const loginAttempts = new Map();

async function login(email: string, password: string) {
  const key = `login:${email}`;
  const attempts = await redis.incr(key);
  if (attempts === 1) await redis.expire(key, 900); // 15 min window

  if (attempts > 5) {
    throw new TooManyRequestsError('Account locked. Try again in 15 minutes.');
  }
  // ...
}
```

### A08: Software & Data Integrity Failures

```bash
# Verify package integrity with lockfiles — never skip
npm ci  # uses package-lock.json, never modifies it

# Audit dependencies regularly
npm audit --audit-level=high
pip-audit
govulncheck ./...

# Sign artifacts and verify before deployment
cosign verify-image myimage:latest
```

### A09: Security Logging & Monitoring

```typescript
// Log security events — structured and immutable
securityLogger.warn('auth.login.failed', {
  email,                   // Do NOT log password
  ip: req.ip,
  userAgent: req.headers['user-agent'],
  reason: 'invalid_credentials',
  timestamp: new Date().toISOString(),
});

// Always log:
// - Authentication successes and failures
// - Authorization failures (access denied)
// - Input validation failures on sensitive endpoints
// - Admin actions
// - Data exports
// - Privilege escalations
```

## 2) Secrets Management

### Storage Hierarchy (prefer higher levels)

```
Level 1 (Best):  Hardware Security Module (HSM)
Level 2:         Managed Identity / IAM roles (AWS, GCP, Azure)
Level 3:         Secrets Manager (AWS SM, HashiCorp Vault, Azure Key Vault)
Level 4:         Environment variables (via orchestrator like K8s Secrets)
Level 5 (Worst): .env files, config files, source code
```

### Vault Integration Pattern

```typescript
// Load secrets at startup from vault — never hardcode
import { SecretsManager } from '@aws-sdk/client-secrets-manager';

async function loadSecrets(): Promise<AppSecrets> {
  const client = new SecretsManager({ region: 'us-east-1' });
  const response = await client.getSecretValue({ SecretId: 'myapp/production' });
  return JSON.parse(response.SecretString!);
}

const secrets = await loadSecrets();

// Inject into app config — never into environment variables of child processes
const config = {
  db: { url: secrets.databaseUrl },
  jwt: { secret: secrets.jwtSecret },
};
```

### Secret Rotation

```bash
# Rotate secrets without downtime:
# 1. Generate new secret
# 2. Add new secret alongside old (dual-write)
# 3. Update all consumers to accept both
# 4. Update all producers to use new secret
# 5. Remove old secret from consumers
# 6. Remove old secret from vault
```

### Pre-commit Secret Scanning

```bash
# Install gitleaks
brew install gitleaks

# .gitleaks.toml
[allowlist]
regexes = [
  "EXAMPLE_KEY",     # Known false positives
]

# Add to pre-commit hooks
pre-commit install

# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

Never commit: API keys, JWT secrets, DB passwords, private keys, `.env` files.

## 3) Input Validation & Sanitization

```typescript
// Validate at system boundaries — every external input
import { z } from 'zod';

const UserInputSchema = z.object({
  email: z.string().email().max(255),
  name: z.string().min(1).max(100).regex(/^[a-zA-Z\s'-]+$/),
  age: z.number().int().min(0).max(150).optional(),
});

// Sanitize before storing user-generated content
import createDOMPurify from 'dompurify';
import { JSDOM } from 'jsdom';

const window = new JSDOM('').window;
const DOMPurify = createDOMPurify(window as unknown as Window);

function sanitizeHtml(input: string): string {
  return DOMPurify.sanitize(input, { ALLOWED_TAGS: ['b', 'i', 'em', 'strong'] });
}

// Path traversal prevention
import path from 'path';

function safeFilePath(base: string, userInput: string): string {
  const resolved = path.resolve(base, userInput);
  if (!resolved.startsWith(path.resolve(base))) {
    throw new SecurityError('Path traversal detected');
  }
  return resolved;
}
```

## 4) Security Testing Patterns

### Static Analysis (SAST)

```bash
# JavaScript/TypeScript
npm run audit                  # npm audit
npx semgrep --config auto src/ # Semgrep SAST

# Python
bandit -r src/ -f json         # Bandit security scanner
safety check                   # known CVEs in dependencies

# Go
govulncheck ./...              # official Go vulnerability scanner
gosec ./...                    # Go security checker

# PHP
composer audit                 # dependency audit
psalm --taint-analysis         # taint analysis for injection
```

### OWASP ZAP (DAST)

```bash
# Baseline scan — safe for CI
docker run --rm owasp/zap2docker-stable \
  zap-baseline.py -t https://staging.myapp.com -r zap-report.html

# Active scan (staging only — never production)
docker run --rm owasp/zap2docker-stable \
  zap-full-scan.py -t https://staging.myapp.com
```

### Security Test Cases to Write

```typescript
// Test authorization — always verify ownership
describe('GET /orders/:id', () => {
  it('returns 404 when requesting another user\'s order', async () => {
    const otherUsersOrder = await createOrder({ userId: 'user-2' });
    const response = await api.get(`/orders/${otherUsersOrder.id}`)
      .set('Authorization', `Bearer ${user1Token}`);
    expect(response.status).toBe(404);  // Not 403 — don't leak existence
  });

  it('requires authentication', async () => {
    const response = await api.get('/orders/123');
    expect(response.status).toBe(401);
  });
});

// Test injection prevention
it('rejects SQL injection in search query', async () => {
  const response = await api.get("/products?search=' OR '1'='1")
    .set('Authorization', `Bearer ${token}`);
  expect(response.status).toBe(400);
});
```

## 5) Security Review Agent Checklist

Use before every PR merge involving auth, data access, external APIs, or user input:

### Authentication & Authorization
- [ ] All endpoints require authentication (explicit allowlist for public routes)
- [ ] Ownership/tenancy checked on every resource access
- [ ] JWT tokens are short-lived (≤15 min access, ≤7 day refresh)
- [ ] Refresh token rotation implemented
- [ ] Rate limiting on auth endpoints (login, password reset, OTP)
- [ ] Account lockout after N failed attempts

### Input & Output
- [ ] All external inputs validated at boundaries (Zod, Pydantic, etc.)
- [ ] SQL queries use parameterized statements
- [ ] HTML output escaped or sanitized
- [ ] File paths validated against traversal
- [ ] File uploads: type checked, size limited, stored outside webroot

### Secrets
- [ ] No secrets in source code or config files
- [ ] No secrets in logs, error messages, or API responses
- [ ] Secrets loaded from vault or environment at runtime
- [ ] `.env` in `.gitignore`
- [ ] Pre-commit secret scanning configured

### Data Protection
- [ ] Passwords hashed with bcrypt/argon2 (cost ≥ 12)
- [ ] PII encrypted at rest
- [ ] Sensitive data redacted from logs
- [ ] DB connections encrypted (TLS)
- [ ] Sensitive columns not returned in API responses

### HTTP Security
- [ ] HTTPS enforced (HSTS enabled)
- [ ] Security headers via Helmet or equivalent
- [ ] CORS configured with explicit origins
- [ ] Rate limiting on all public endpoints
- [ ] Request size limits configured

### Dependencies
- [ ] `npm audit` / `pip-audit` / `govulncheck` — no HIGH/CRITICAL
- [ ] Dependency lockfiles committed
- [ ] No known-vulnerable packages

## 6) Compliance Notes

| Regulation | Key Requirement | Action |
|---|---|---|
| GDPR | Data minimization, right to erasure | Only collect needed data; implement delete |
| PCI-DSS | Never store raw card data | Use Stripe/Braintree tokenization |
| SOC 2 | Audit logging, access controls | Immutable audit logs; least-privilege roles |
| HIPAA | PHI encryption, audit trails | Encrypt PII; log all PHI access |

## References

- [OWASP Top 10 2025](https://owasp.org/Top10/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [Node.js Security Best Practices](https://nodejs.org/en/learn/getting-started/security-best-practices)
