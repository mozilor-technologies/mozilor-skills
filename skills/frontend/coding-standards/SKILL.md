---
name: coding-standards
description: "Mandatory coding standards for the Webtoffee Marketing Suite. Invoke before writing any code. Covers Tailwind sf- prefix rules, TypeScript conventions, component patterns, atomic design placement, form/state/routing/API patterns, and do-nots."
---

# Coding Standards — Webtoffee Marketing Suite

## Tailwind CSS — CRITICAL RULE
All Tailwind utility classes **must** use the `sf-` prefix.

```tsx
// ✅ Correct
<div className="sf-flex sf-items-center sf-gap-4 sf-text-sm sf-font-medium">

// ❌ Wrong — will not work
<div className="flex items-center gap-4 text-sm font-medium">
```

This applies to `!important` overrides on shadcn/ui components too: `!sf-h-10`, not `!h-10`.

Custom color tokens and design system are defined in `tailwind.config.ts`. Use those tokens — do not use arbitrary color values. For Figma color mapping, invoke the `figma-to-code` skill.

## TypeScript
- All new files must be `.ts` or `.tsx` — no `.js`
- Avoid `any` — use proper types or `unknown`
- Define prop types inline with the component using `interface` or `type`
- Export types/interfaces that are shared across files from a `types.ts` file in the same directory

## Component Patterns

### File naming
- Components: `PascalCase.tsx` (e.g., `CampaignCard.tsx`)
- Hooks: `camelCase.ts` prefixed with `use` (e.g., `useCampaignData.ts`)
- Services: `camelCase.ts` suffixed with `Service` (e.g., `campaignService.ts`)
- Stores: `camelCase.ts` suffixed with `Store` (e.g., `campaignStore.ts`)

### Component structure
```tsx
import { ... } from 'react'
import { ... } from '@/components/ui/...'
import { ... } from '@/atoms/...'
// ... other imports

interface ComponentNameProps {
  // props
}

export function ComponentName({ prop1, prop2 }: ComponentNameProps) {
  // hooks first
  // derived state
  // handlers
  // render
  return (...)
}
```

### Atomic Design placement
- **atoms/**: Single-purpose, no business logic, no API calls. Examples: Button variant, Badge, Avatar, Input wrapper.
- **molecules/**: Combines 2+ atoms, may have local state, no API calls. Examples: SearchInput with clear button, DropdownPanel.
- **organisms/**: Has business logic, may connect to stores/queries, composes molecules/atoms. Examples: CampaignList, ContactsTable.
- **templates/**: Full page layout, composes organisms, handles route-level data fetching.

## Forms
Always use React Hook Form + Zod:
```tsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const schema = z.object({
  email: z.string().email(),
  name: z.string().min(1, 'Required'),
})

type FormData = z.infer<typeof schema>

export function MyForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(schema),
  })
  // ...
}
```

Zod schemas for reusable forms go in `src/schemas/`.

## State Management
```ts
// Store pattern
import { create } from 'zustand'

interface FeatureStore {
  data: Item[]
  isLoading: boolean
  fetchData: () => Promise<void>
}

export const useFeatureStore = create<FeatureStore>((set) => ({
  data: [],
  isLoading: false,
  fetchData: async () => {
    set({ isLoading: true })
    const result = await featureService.getAll()
    set({ data: result, isLoading: false })
  },
}))
```

Use `persist` middleware only when state must survive page refresh.

### Zustand + useCallback/useEffect — Infinite Loop Prevention — CRITICAL RULE

**Never put Zustand store state in a `useCallback` dep array if that state is also reset or modified by the same effect or its siblings.** This causes infinite loops:

```
useEffect → resetX() → state changes → useCallback recreates → useEffect re-runs → loop
```

**Rules:**
1. **Zustand store actions are stable references** — never add them to `useCallback` or `useEffect` dep arrays (`setX`, `fetchX`, `resetX`, etc. never change identity).
2. **Zustand store state used only for reading inside a callback** — do NOT close over the subscribed value. Use `useXxxStore.getState().value` instead. This breaks the dep chain without creating a stale closure.
3. **`useEffect` that calls both a reset action AND a load action** — the load function must NOT depend on the state being reset.

```tsx
// ❌ WRONG — discounts in dep array causes infinite loop when resetDiscounts() clears it
const load = useCallback(async () => {
  const ids = new Set(discounts.map(d => d.id)) // closes over discounts
  ...
}, [discounts, filters]) // discounts here = infinite loop if reset fires

useEffect(() => {
  resetDiscounts()  // sets discounts: []  ← triggers load recreation ← triggers effect ← loop
  load()
}, [filters, load])

// ✅ CORRECT — read store state at call time, not via closure
const load = useCallback(async () => {
  const ids = new Set(useFeatureStore.getState().items.map(d => d.id)) // getState() = stable
  ...
}, [filters]) // no store state in deps

useEffect(() => {
  resetItems()
  load()
}, [filters, load]) // safe — load only recreates when filters changes
```

## Error Handling — CRITICAL RULE
**All catch blocks must show a destructive toast:**
```tsx
toast({ variant: "destructive", title: "...", description: "..." })
```
Never silently swallow errors or use `console.error` alone. Async store actions called from event handlers must be awaited or have `.catch()`.

## API / Service Layer
```ts
// src/services/featureService.ts
import { restClient } from '@/helpers/restClient'

export const featureService = {
  getAll: () => restClient.get<Item[]>('/api/feature'),
  create: (payload: CreatePayload) => restClient.post<Item>('/api/feature', payload),
  update: (id: string, payload: UpdatePayload) => restClient.put<Item>(`/api/feature/${id}`, payload),
  delete: (id: string) => restClient.delete(`/api/feature/${id}`),
}
```

Never call `restClient` directly from components or stores — always go through a service.

## Routing
```tsx
// src/routes/(dashboard)/feature/index.tsx
import { createFileRoute } from '@tanstack/react-router'
import { FeatureTemplate } from '@/templates/FeatureTemplate'

export const Route = createFileRoute('/(dashboard)/feature/')({
  component: FeatureTemplate,
})
```

## Imports
Always use the `@/` alias:
```tsx
// ✅
import { Button } from '@/components/ui/button'
import { useAuthStore } from '@/stores/authStore'

// ❌
import { Button } from '../../../components/ui/button'
```

## Test IDs
Add `data-testid` attributes to all interactive elements and key UI elements. Use kebab-case: `data-testid="campaign-create-button"`.

## Do Not
- Do not use inline styles — use Tailwind with `sf-` prefix
- Do not use `react-router-dom` directly — use TanStack Router
- Do not make API calls from components — use services + stores
- Do not use `console.log` in committed code
- Do not use `any` type
- Do not import from `src/` — always use `@/`
- Do not add UI elements not present in the Figma design
- **Do not put Zustand store state in `useCallback`/`useEffect` dep arrays if that state is mutated by the same effect** — use `useXxxStore.getState().value` instead to read without subscribing (prevents infinite loops)
- **Do not add Zustand store actions to dep arrays** — `set*`, `fetch*`, `reset*` actions are always stable references
