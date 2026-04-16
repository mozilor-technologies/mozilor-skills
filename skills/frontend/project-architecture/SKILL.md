---
name: project-architecture
description: "Reference for the Webtoffee Marketing Suite app architecture. Use before designing or implementing any new feature to understand tech stack, component hierarchy, routing, state management, and API patterns."
---

# Architecture Overview — Webtoffee Marketing Suite

## What This App Is
An email marketing and campaign automation platform for e-commerce stores (Shopify and WooCommerce). Core capabilities: campaign creation, email automation workflows, product recommendations, analytics, and contact management.

## Tech Stack
| Layer | Technology |
|-------|-----------|
| Framework | React 18 + TypeScript |
| Bundler | Vite 5 |
| Routing | TanStack Router v1 (file-based) |
| State | Zustand 5 |
| Server State | TanStack React Query v5 |
| Forms | React Hook Form + Zod |
| UI Primitives | Radix UI |
| Styling | Tailwind CSS with `sf-` prefix |
| Tables | TanStack React Table |
| Email Editor | GrapesJS + MJML |
| Charts | Recharts |
| File Storage | AWS SDK S3 |
| Analytics | Mixpanel, Google Analytics, Hotjar |

## Component Hierarchy (Atomic Design)
```
src/
├── atoms/          # Smallest reusable UI primitives: buttons, inputs, badges
├── molecules/      # Composite components combining atoms: panels, popups
├── organisms/      # Complex sections: AnalyticsChart, campaign sections
├── templates/      # Full page layouts: auth pages, dashboard pages
└── components/ui/  # Radix UI-based design system components
```

**Rule:** Never skip levels. A molecule uses atoms. An organism uses molecules/atoms. A template composes organisms.

## Routing
File-based routing in `src/routes/`. Structure:
```
src/routes/
├── __root.tsx              # Auth guard: unauthenticated → /login, authenticated → away from auth routes
├── (auth)/                 # Login, signup, OTP verification, store connection
├── (dashboard)/            # Main app (campaigns, automations, contacts, insights, settings)
└── oauth/                  # Google OAuth callback
```
New routes go in the appropriate group. Use `createFileRoute` from TanStack Router.

## State Management
30+ Zustand stores in `src/stores/`. Key stores:
- `useAuthStore` — User session, JWT tokens, Google OAuth
- `useCampaignStore` / `useCampaignEditorStore` — Campaign CRUD and editor state
- `automationStore` — Automation workflow state
- `useAnalyticsStore` — Analytics data
- `useContactsStore` — Audience/contacts
- `useWebsiteStore` / `useBrandAssetsStore` / `useDomainStore` — Store/brand config

Several stores use Zustand's `persist` middleware for localStorage.

**When adding a store:** Create in `src/stores/`, export a named hook (e.g. `useFeatureStore`), use `persist` only if state must survive page refresh.

## API Communication
Custom REST client in `src/helpers/restClient.ts`:
- Queued single-at-a-time request processing
- Bearer token auth with automatic refresh token rotation (tokens come back via response headers)
- Auto-logout on 401
- Supports GET/POST/PUT/PATCH/DELETE

**Service layer** in `src/services/` wraps the REST client per domain (auth, analytics, domain, WooCommerce, etc.). Always add new API calls to a service file — never call restClient directly from a component.

Auth tokens stored in localStorage under `ema-auth` as `{ access_token, refresh_token, expires_at }`.

## Environment Variables
Via `.env` using `import.meta.env`:
- `VITE_BACKEND_URL` / `VITE_BACKEND_API_URL` — Backend API base URL
- `VITE_APP_STAGE` — Environment stage (`development` | `production`)
- `VITE_APP_MEDIA_URL` — Media assets CDN URL

## Path Aliases
`@/*` maps to `./src/*` — always use this for imports, never relative `../` chains beyond one level.

## Analytics Events
New features that involve user interaction must fire Mixpanel tracking events. Follow the existing event naming pattern found in `src/helpers/` or existing store files.
