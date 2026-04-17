---
name: security-review
description: Run a comprehensive security review on changed code files
---

# Security Review

Conduct a security audit on the files listed in `implementation.json`. This skill is loaded and executed inline by the Code Review agent — no delegation or external tools required.

## Scope

Review only the files listed in `files_changed` across all subtasks in `implementation.json`. Do not scan the entire codebase.

## Checklist

Work through each section. For each finding, record:
- File path and line number
- Severity: `CRITICAL` | `HIGH` | `MEDIUM` | `LOW`
- Description of the issue
- Remediation

### 1. Injection (OWASP A03)
- [ ] SQL queries use parameterized statements — no string concatenation with user input
- [ ] NoSQL queries are not constructed from unsanitized input
- [ ] Shell commands do not include unsanitized user input (`subprocess`, `exec`, `os.system`)
- [ ] Template rendering does not pass raw user input (XSS via server-side templates)

### 2. Broken Access Control (OWASP A01)
- [ ] All new endpoints have authentication checks where required
- [ ] Authorization is verified before returning or modifying data (no IDOR)
- [ ] No user-controlled input used directly as file path or resource identifier without validation

### 3. Cryptographic Failures (OWASP A02)
- [ ] No hardcoded secrets, API keys, passwords, or tokens
- [ ] Passwords are hashed with a strong algorithm (bcrypt, argon2) — not MD5, SHA1, or plain text
- [ ] Sensitive data is not logged or included in error responses
- [ ] TLS is enforced for any outbound HTTP calls to external services

### 4. Insecure Design (OWASP A04)
- [ ] Business logic cannot be bypassed by manipulating request parameters
- [ ] Rate limiting or abuse controls exist for new public-facing endpoints

### 5. Security Misconfiguration (OWASP A05)
- [ ] Debug mode, verbose errors, or stack traces are not exposed to end users
- [ ] CORS settings are not overly permissive (`*`) for authenticated endpoints
- [ ] New dependencies do not introduce known high/critical CVEs (check with `npm audit`, `pip-audit`, or `composer audit` as appropriate)

### 6. Authentication Failures (OWASP A07)
- [ ] Session tokens are not exposed in URLs or logs
- [ ] JWT tokens (if used) are validated — signature, expiry, audience
- [ ] No new unauthenticated paths that should be protected

### 7. Software Integrity (OWASP A08)
- [ ] New dependencies are from trusted sources with pinned versions
- [ ] No dynamic `eval`, `exec`, or `require` with user-controlled input

### 8. Logging & Monitoring (OWASP A09)
- [ ] Authentication failures are logged
- [ ] No sensitive data (PII, tokens, passwords) appears in log statements

### 9. SSRF (OWASP A10)
- [ ] Any code that fetches a URL uses an allowlist — user input must not control the full URL

## Severity Definitions

| Severity | Meaning |
|---|---|
| CRITICAL | Exploitable now with no preconditions — data breach, RCE, credential theft |
| HIGH | Serious impact, requires specific conditions |
| MEDIUM | Limited impact or difficult to exploit |
| LOW | Best-practice violation or minor concern |

## Severity → Review Schema Mapping

| Security severity | Code Review blocking level |
|---|---|
| CRITICAL | `blocking` |
| HIGH | `blocking` |
| MEDIUM | `important` |
| LOW | `minor` |

## Output

Return findings in this format for each issue:

```
[SEVERITY] file/path.py:42 — ISSUE_CODE
Description: <what the issue is>
Remediation: <how to fix it>
```

If no issues are found, return: `Security review passed — no findings.`
