---
name: testing-standards
description: "Playwright testing standards for the Webtoffee Marketing Suite. Invoke before writing any tests. Covers test structure, selector priority, auth setup, API mocking, assertions, and coverage goals."
---

# Testing Standards — Webtoffee Marketing Suite

## Framework
**Playwright** for UI/integration tests. No unit test framework is currently configured.

## Test Location
```
tests/
└── features/
    └── [feature-name]/       # One folder per feature
        ├── [feature].spec.ts # Main test file
        └── helpers.ts        # Page objects / helpers (if needed)
```

Example: `tests/features/campaign-creation/campaign-creation.spec.ts`

## Running Tests
```bash
npx playwright test                              # All tests
npx playwright test tests/features/[feature]/   # Specific feature
npx playwright test --ui                         # Interactive UI mode
npx playwright test --headed                     # See browser
```

## Test Structure
```ts
import { test, expect } from '@playwright/test'

test.describe('Feature Name', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/feature-path')
  })

  test('should [expected behavior]', async ({ page }) => {
    // Arrange
    // Act
    await page.click('[data-testid="action-button"]')
    // Assert
    await expect(page.locator('[data-testid="result"]')).toBeVisible()
  })
})
```

## Selectors — Priority Order
1. `data-testid` attributes (preferred)
2. ARIA roles: `page.getByRole('button', { name: 'Submit' })`
3. Labels: `page.getByLabel('Email')`
4. Text: `page.getByText('Campaign created')`
5. CSS selectors (last resort)

When an element is missing a `data-testid`, use a fallback selector and add a comment:
```ts
await page.click('[data-testid="save-btn"]') // MISSING data-testid: save-btn
```

## Auth in Tests
Auth is handled automatically. `tests/auth.setup.ts` logs in once using `TEST_USER_EMAIL` / `TEST_USER_PASSWORD` from `.env` and saves storage state to `tests/.auth/user.json`. All tests in the `chromium` project load that state automatically — **do not log in inside individual tests**.

Required `.env` variables:
```
TEST_USER_EMAIL=your@email.com
TEST_USER_PASSWORD=yourpassword
```

To seed Zustand store state without API round-trips, use `page.addInitScript`:
```ts
test('my test', async ({ page }) => {
  await page.addInitScript((state) => {
    localStorage.setItem('website-storage', JSON.stringify({ state, version: 0 }))
  }, { websites: [...], currentWebsite: { ... } })
  await page.goto('/dashboard')
})
```

## Mocking API Calls
```ts
test('shows error on API failure', async ({ page }) => {
  await page.route('**/api/campaigns', route => {
    route.fulfill({ status: 500, body: 'Server error' })
  })
  await page.goto('/campaigns')
  await expect(page.getByText('Something went wrong')).toBeVisible()
})
```

## Assertions Best Practices
```ts
// ✅ Wait for elements properly
await expect(page.locator('[data-testid="result"]')).toBeVisible()
await expect(page.locator('[data-testid="list"]')).toHaveCount(3)
await expect(page.locator('[data-testid="title"]')).toHaveText('Campaign Name')

// ❌ Avoid arbitrary waits
await page.waitForTimeout(2000)
```

## Test Naming Convention
- File: `[feature-name].spec.ts`
- Describe block: Feature name in plain English
- Test name: `'should [action] when [condition]'`

## Coverage Goals (per feature) — All Mandatory
- All acceptance criteria
- Happy path (core user flow start to finish)
- At least 2 edge cases (empty states, loading states, form validation errors)
- At least 1 error state (mock with `page.route()`)
