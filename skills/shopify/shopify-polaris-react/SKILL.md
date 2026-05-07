---
name: shopify-polaris-react
description: "Build the React UI for Shopify embedded admin apps that use the Remix template (`@shopify/shopify-app-remix`). Polaris React components from `@shopify/polaris` v12 plus App Bridge React (`@shopify/app-bridge-react`). Use when writing or modifying anything under `app/` of a Shopify Remix repo: routes, loaders, actions, components, AppProvider, NavMenu, useAppBridge, save bar, toast, modal, Polaris layout, GraphQL via admin.graphql, webhook handlers. Do NOT use for admin UI extensions in `extensions/` (those use web components — see `shopify-polaris-admin-extensions`)."
metadata:
  author: Mozilor Technologies
  version: "1.0.0"
---

# Shopify Polaris React (Remix template)

For Shopify embedded admin apps built on `@shopify/shopify-app-remix`. UI is **Polaris React components** from `@shopify/polaris` — NOT the new Polaris web components (`s-*` tags). The web components target Shopify's new Vite scaffold; this skill is for the Remix template that Shopify shipped through 2025.

> **Maintenance status (2026-05):** The `Shopify/polaris` GitHub repo was archived 2026-01-06. v12 is effectively the terminal version of Polaris React — new components and bug fixes will not land. Existing apps continue to ship; Shopify's forward investment is in web components for the new scaffold. When suggesting fixes, work within v12 APIs. Do not propose "upgrade to a newer Polaris version" — there isn't one.

## When this skill applies

- Files under `app/` of a `@shopify/shopify-app-remix` project.
- Writing or modifying Remix routes, loaders, actions, embedded admin components.
- Questions about Polaris layout, forms, navigation, save bar, toast, modal in a Remix admin app.
- Wiring `authenticate.admin`, `admin.graphql()`, `authenticate.webhook`.

## When this skill does NOT apply

- **Admin UI extensions** under `extensions/<name>/` using `@shopify/ui-extensions-react` — use `shopify-polaris-admin-extensions`.
- **Liquid theme code** — use `shopify-liquid`.
- **Authoring Admin GraphQL queries by themselves** before deciding how to run them — use `shopify-admin`.
- **Shopify CLI tasks** (`shopify app config validate`, `shopify store execute`) — use `shopify-use-shopify-cli`.

---

## Canonical setup

### `app/shopify.server.ts`

The single source of truth for the Shopify app client. Export `authenticate`, `unauthenticated`, `login`, `registerWebhooks` for routes to consume.

```ts
import "@shopify/shopify-app-remix/adapters/node";
import {
  ApiVersion,
  AppDistribution,
  shopifyApp,
} from "@shopify/shopify-app-remix/server";
import { PrismaSessionStorage } from "@shopify/shopify-app-session-storage-prisma";
import prisma from "./db.server";

const shopify = shopifyApp({
  apiKey: process.env.SHOPIFY_API_KEY,
  apiSecretKey: process.env.SHOPIFY_API_SECRET || "",
  apiVersion: ApiVersion.October24,
  scopes: process.env.SCOPES?.split(","),
  appUrl: process.env.SHOPIFY_APP_URL || "",
  authPathPrefix: "/auth",
  sessionStorage: new PrismaSessionStorage(prisma),
  distribution: AppDistribution.AppStore,
  future: {
    unstable_newEmbeddedAuthStrategy: true,
    removeRest: true,
  },
});

export default shopify;
export const authenticate = shopify.authenticate;
export const unauthenticated = shopify.unauthenticated;
export const login = shopify.login;
export const registerWebhooks = shopify.registerWebhooks;
```

### `app/routes/app.tsx` — embedded shell

`<AppProvider>` here comes from **`@shopify/shopify-app-remix/react`**, NOT from `@shopify/polaris`. The framework wrapper sets up Polaris and App Bridge in one shot. Do not use `@shopify/polaris`'s `<AppProvider i18n={...}>` for embedded routes — that's for non-embedded apps.

```tsx
import type { HeadersFunction, LoaderFunctionArgs } from "@remix-run/node";
import { json } from "@remix-run/node";
import { Link, Outlet, useLoaderData, useRouteError } from "@remix-run/react";
import { boundary } from "@shopify/shopify-app-remix/server";
import { AppProvider } from "@shopify/shopify-app-remix/react";
import { NavMenu } from "@shopify/app-bridge-react";
import polarisStyles from "@shopify/polaris/build/esm/styles.css?url";

import { authenticate } from "../shopify.server";

export const links = () => [{ rel: "stylesheet", href: polarisStyles }];

export const loader = async ({ request }: LoaderFunctionArgs) => {
  await authenticate.admin(request);
  return json({ apiKey: process.env.SHOPIFY_API_KEY || "" });
};

export default function App() {
  const { apiKey } = useLoaderData<typeof loader>();
  return (
    <AppProvider isEmbeddedApp apiKey={apiKey}>
      <NavMenu>
        <Link to="/app" rel="home">Home</Link>
        <Link to="/app/settings">Settings</Link>
      </NavMenu>
      <Outlet />
    </AppProvider>
  );
}

export function ErrorBoundary() {
  return boundary.error(useRouteError());
}

export const headers: HeadersFunction = (h) => boundary.headers(h);
```

Polaris CSS uses Vite's `?url` suffix and is emitted via Remix `links()` — NOT the bundler-side `import "@shopify/polaris/build/esm/styles.css"` shown in Polaris's standalone README. Remix needs the URL form to ship the stylesheet over the document.

### `app/root.tsx`

Keep it minimal — no `AppProvider`. The Polaris/App Bridge providers belong in `app.tsx` so non-embedded routes (`auth/`, public pages) don't get them.

```tsx
import { Links, Meta, Outlet, Scripts, ScrollRestoration } from "@remix-run/react";

export default function App() {
  return (
    <html>
      <head>
        <meta charSet="utf-8" />
        <meta name="viewport" content="width=device-width,initial-scale=1" />
        <link rel="preconnect" href="https://cdn.shopify.com/" />
        <link rel="stylesheet" href="https://cdn.shopify.com/static/fonts/inter/v4/styles.css" />
        <Meta />
        <Links />
      </head>
      <body>
        <Outlet />
        <ScrollRestoration />
        <Scripts />
      </body>
    </html>
  );
}
```

---

## Auth and data loading

Every route under `app/` must authenticate the request. The framework handles session token verification, scope checks, and reauth redirects — never roll your own.

```tsx
import type { LoaderFunctionArgs, ActionFunctionArgs } from "@remix-run/node";
import { authenticate } from "../shopify.server";

export const loader = async ({ request }: LoaderFunctionArgs) => {
  const { admin, session } = await authenticate.admin(request);
  // admin: Admin API client; session has shop + accessToken
  return json({ shop: session.shop });
};

export const action = async ({ request }: ActionFunctionArgs) => {
  const { admin } = await authenticate.admin(request);
  // ...
};
```

### Admin GraphQL

Use `admin.graphql()`. Do NOT call `fetch()` against `*.myshopify.com/admin/api/...` directly — bypasses session/version handling.

```tsx
const response = await admin.graphql(`
  query {
    shop { id email name }
  }
`);
const { data } = await response.json();
```

For variables:

```tsx
const response = await admin.graphql(
  `mutation productCreate($input: ProductInput!) {
    productCreate(input: $input) {
      product { id }
      userErrors { field message }
    }
  }`,
  { variables: { input: { title: "New product" } } }
);
const { data } = await response.json();

if (data?.productCreate?.userErrors?.length) {
  // GraphQL doesn't surface userErrors as HTTP errors — always check
}
```

---

## Webhooks

```tsx
// app/routes/webhooks.app.uninstalled.tsx
import type { ActionFunctionArgs } from "@remix-run/node";
import { authenticate } from "../shopify.server";

export const action = async ({ request }: ActionFunctionArgs) => {
  const { shop, session, topic, payload } = await authenticate.webhook(request);
  // HMAC verification is already done by authenticate.webhook.
  // session may be null if the app is being uninstalled — handle that.
  return new Response();
};
```

HMAC signature verification is handled by `authenticate.webhook(request)`. Do not reimplement it.

Register webhooks declaratively in `shopify.app.toml` or via `shopifyApp({ webhooks: { ... } })`.

---

## App Bridge React — runtime UI helpers

Get the App Bridge instance via `useAppBridge()`:

```tsx
import { useAppBridge } from "@shopify/app-bridge-react";

const shopify = useAppBridge();

// Toast
shopify.toast.show("Saved");
shopify.toast.show("Update failed", { isError: true, duration: 5000 });

// Save bar (contextual save bar at the top of the admin)
shopify.saveBar.show("my-save-bar-id");
shopify.saveBar.hide("my-save-bar-id");

// Modal — controlled by id
shopify.modal.show("my-modal-id");
shopify.modal.hide("my-modal-id");
```

**SaveBar component** (place once at route level, then drive via `shopify.saveBar.show/hide(id)`):

```tsx
import { SaveBar } from "@shopify/app-bridge-react";

<SaveBar id="my-save-bar-id">
  <button variant="primary" onClick={handleSave}>Save</button>
  <button onClick={handleDiscard}>Discard</button>
</SaveBar>
```

For form submission inside Remix routes, pair `useFetcher()` with the save bar — show on dirty state, hide + toast on `fetcher.state === "idle" && fetcher.data?.success`.

---

## Polaris React v12 — what to use

**Layout primitives:**
- `Page` — top-level wrapper (`title`, `backAction`, `primaryAction`, `secondaryActions`).
- `Layout` / `Layout.Section` — column/sidebar grids.
- `BlockStack` — vertical stack (`gap`). Replaces v11's `VerticalStack`.
- `InlineStack` — horizontal stack (`gap`, `align`, `blockAlign`). Replaces `HorizontalStack`.
- `Box` — generic spacing/background container.
- `InlineGrid` — grid columns.
- `Divider`.

**Surfaces:**
- `Card` — content card. v12 dropped `Card.Section`; nest a `BlockStack` inside instead.
- `Banner` — inline notice (`tone`: `info` / `success` / `warning` / `critical`).
- `Modal` — overlay dialog.
- `Popover`.

**Forms:**
- `TextField`, `Select`, `Checkbox`, `RadioButton`, `ColorPicker`, `FormLayout`.
- For forms tied to Remix actions, use `useFetcher()` from `@remix-run/react`.

**Actions:**
- `Button`, `ButtonGroup`. Use `variant="primary"`, `tone="critical"` for destructive.
- Polaris `Link` for in-flow styled links. For Remix navigation, use `@remix-run/react`'s `Link` and let the wrapping Polaris component (`Page`, `Card`) provide the styling context.

**Feedback:**
- `Spinner`, `SkeletonBodyText`, `SkeletonDisplayText`, `ProgressBar`.

**Icons:** `@shopify/polaris-icons` — import named icons (`ChatIcon`, `EditIcon`, `DeleteIcon`, etc.) and pass them to Polaris components' `icon` prop.

### Anti-patterns

- ❌ `LegacyCard` / `LegacyStack` / `VerticalStack` / `HorizontalStack` — v11 names. Use `Card` / `BlockStack` / `InlineStack`.
- ❌ Polaris web components (`<s-page>`, `<s-button>`, `<s-card>`) in the app shell — wrong runtime here.
- ❌ Importing `@shopify/polaris/styles.css` (v11 path). v12 path is `@shopify/polaris/build/esm/styles.css`.
- ❌ Wrapping the embedded app in `<AppProvider i18n={...}>` from `@shopify/polaris`. Use `<AppProvider>` from `@shopify/shopify-app-remix/react`.
- ❌ Calling `fetch("https://<shop>.myshopify.com/admin/api/...")` directly. Use `admin.graphql()`.
- ❌ Verifying webhook HMAC manually. Use `authenticate.webhook(request)`.
- ❌ Putting `AppProvider` in `root.tsx`. It belongs in `app/routes/app.tsx` so non-embedded routes don't inherit it.

---

## Mixed-pattern files

If a file already uses a non-canonical pattern (older `LegacyCard`, custom auth wrapper, manual HMAC check), match the surrounding file. Do not introduce new patterns mid-file. Refactors are separate work — call them out explicitly, don't sneak them in.

---

## Quick reference

| Need | Use |
|---|---|
| Authenticate an admin route | `authenticate.admin(request)` |
| Authenticate a webhook | `authenticate.webhook(request)` |
| Run an Admin GraphQL query | `admin.graphql(query, { variables })` |
| Show a toast | `useAppBridge().toast.show(message, options)` |
| Show/hide save bar | `useAppBridge().saveBar.show/hide(id)` |
| App nav menu | `<NavMenu>` from `@shopify/app-bridge-react` |
| Vertical layout | `<BlockStack gap="...">` |
| Horizontal layout | `<InlineStack gap="..." align="...">` |
| Page shell | `<Page title="..." backAction={...} primaryAction={...}>` |
| Polaris CSS in Remix | `import x from "@shopify/polaris/build/esm/styles.css?url"` + `links()` |
| Catch route errors | `boundary.error(useRouteError())` |
| Embedded route headers | `boundary.headers(headersArgs)` |
