---
name: figma-to-code
description: "Authoritative guide for translating Figma designs to project-compliant code. Invoke before writing any UI component derived from Figma. Covers color token mapping, typography, spacing, radius, icons, layout, shadows, gradients, existing component reuse, and a pre-code checklist."
---

# Figma → Code Integration Guide

Read this before writing any component from a Figma design. This is the single source of truth for all Figma-to-code translations.

---

## Pre-Code Checklist (run before writing any Figma-derived component)

- [ ] Colors translated to tokens (see Color Token Map below) — no raw hex/rgb/hsl
- [ ] Font sizes and weights mapped to Tailwind classes
- [ ] Spacing values divided by 4 and expressed as Tailwind units
- [ ] Border radius mapped to `sf-rounded-*`
- [ ] Icons sourced from `lucide-react` — no inline SVGs
- [ ] Absolute Figma layout converted to flex/grid
- [ ] Existing atoms/molecules/ui components checked for reuse
- [ ] Images use `getMediaUrl()` or prop-passed strings
- [ ] Shadows mapped to `sf-shadow-*` tokens
- [ ] Gradient uses `sf-bg-primary-gradient` utility
- [ ] All Tailwind classes have `sf-` prefix
- [ ] No inline styles anywhere

---

## Color Token Map

When reading Figma designs, **never use raw color values**. Translate every color to the nearest project token regardless of format (hex, RGB, HSL, or Figma's 0–1 float RGBA).

All tokens via Tailwind with `sf-` prefix:
- background → `sf-bg-{token}`
- text → `sf-text-{token}`
- border → `sf-border-{token}`
- ring → `sf-ring-{token}`

### Primary Blues (dark navy — branding / hero surfaces)

| Token | Hex | RGB | HSL | Usage |
|---|---|---|---|---|
| `primary` | `#132e5a` | `rgb(19, 46, 90)` | `hsl(217, 65%, 21%)` | Dark navy — primary brand, hero backgrounds |
| `primary-hover` | `#1c3567` | `rgb(28, 53, 103)` | `hsl(211, 53%, 28%)` | Hover on primary surfaces |
| `primary-foreground` | `#16295a` | `rgb(22, 41, 90)` | `hsl(217, 66%, 15%)` | Text/icons on primary dark backgrounds |
| `webtoffee-tertiary` | `#344a6e` | `rgb(52, 74, 110)` | `hsl(221, 54%, 42%)` | Mid-tone blue accents |

### Action Blues (interactive — buttons, links, borders)

| Token | Hex | RGB | HSL | Usage |
|---|---|---|---|---|
| `webtoffee` | `#1763dc` | `rgb(23, 99, 220)` | `hsl(217, 81%, 48%)` | **Primary action** — CTA buttons, active states |
| `webtoffee-hover` | `#0050b3` | `rgb(0, 80, 179)` | `hsl(216, 100%, 35%)` | Hover on webtoffee elements |
| `webtoffee-link` | `#1763dc` | same as `webtoffee` | — | Link colour |
| `webtoffee-link-hover` | `#0e408b` | `rgb(14, 64, 139)` | `hsl(216, 82%, 30%)` | Hovered link |

### Text Blues

| Token | Hex | RGB | Usage |
|---|---|---|---|
| `webtoffee-text` | `#003f9e` | `rgb(0, 63, 158)` | Primary body text on light backgrounds |
| `webtoffee-text-secondary` | `#2b4f7c` | `rgb(43, 79, 124)` | Secondary / muted text |

### Backgrounds & Surfaces

| Token | Hex | Usage |
|---|---|---|
| `webtoffee-secondary` | `#ebf2ff` | Light blue chip / tag backgrounds |
| `background` / `card` | `#ffffff` | Page / card backgrounds |
| `muted` | `#f5f5f5` | Muted / disabled fills |
| `accent` | `#eff1f5` | Subtle section backgrounds, icon wrappers |

Note: Figma may output `#eceef2` or `#eff1f5` for the same accent surface — both map to `sf-bg-accent`.

### Borders

| Token | Hex | Usage |
|---|---|---|
| `webtoffee-border` | `#c8d4e9` | Default component borders |
| `webtoffee-border-secondary` | `#d3dced` | Dividers, secondary borders |
| `border` | `#e5e7eb` | Generic border (Radix default) |

### Status / Semantic

| Token | Hex | Usage |
|---|---|---|
| `destructive` | `#ffebeb` | Error background fills |
| `destructive-foreground` | `#c0392b` | Error text / icons |
| `success` | `#eafaf1` | Success background fills |
| `success-foreground` | `#1a7340` | Success text / icons |

### Badge / Pill Colours (`sf-bg-badge-{name}`)

| Token | Hex | Label |
|---|---|---|
| `badge-sage` | `#b1e8cb` | Live / active |
| `badge-blue` | `#bce4f1` | Scheduled |
| `badge-pink` | `#fedce2` | Expired / error |
| `badge-gray` | `#e0e4ea` | Disabled / inactive |
| `badge-lavender` | `#ede9fe` | Other |
| `badge-sand` | `#f5f0dc` | Other |

### Quick-Reference (most common Figma colors)

```
#1763dc  /  rgb(23, 99, 220)   /  hsl(217, 81%, 48%)  →  sf-text/bg/border-webtoffee
#132e5a  /  rgb(19, 46, 90)    /  hsl(217, 65%, 21%)  →  sf-bg/text-primary
#003f9e  /  rgb(0, 63, 158)    /  hsl(216, 100%, 31%) →  sf-text-webtoffee-text
#ebf2ff  /  rgb(235, 242, 255) /  hsl(218, 100%, 95%) →  sf-bg-webtoffee-secondary
#c8d4e9  /  rgb(200, 212, 233) /  hsl(219, 40%, 86%)  →  sf-border-webtoffee
#eff1f5  /  rgb(239, 241, 245) /  hsl(220, 19%, 94%)  →  sf-bg-accent
```

### Handling Color Opacity

Figma may output any of: `#1763dc`, `#1763dc80`, `rgb(23, 99, 220)`, `rgba(23, 99, 220, 0.5)`, `hsl(...)`, float RGBA `rgba(0.09, 0.39, 0.86, 0.5)`.

Steps:
1. Strip alpha, match base color to a token.
2. Apply opacity via Tailwind's `/` modifier — never raw RGBA.

```tsx
// rgba(23, 99, 220, 0.5) → webtoffee at 50%
<div className="sf-bg-webtoffee/50" />

// rgba(19, 46, 90, 0.1) → primary at 10%
<div className="sf-bg-primary/10" />
```

Tailwind opacity steps are multiples of 5. Round Figma alpha to nearest: `0.5 → /50`, `0.8 → /80`, etc.

---

## 1. Typography

Never use `style={{ fontSize }}` or `style={{ fontWeight }}`.

### Font size (px → Tailwind)

| Figma px | Tailwind class |
|---|---|
| 10–12px | `sf-text-xs` |
| 13–14px | `sf-text-sm` |
| 15–16px | `sf-text-base` |
| 18px | `sf-text-lg` |
| 20px | `sf-text-xl` |
| 24px | `sf-text-2xl` |
| 30px | `sf-text-3xl` |
| 36px | `sf-text-4xl` |

### Font weight

| Figma | Tailwind |
|---|---|
| 400 | `sf-font-normal` |
| 500 | `sf-font-medium` |
| 600 | `sf-font-semibold` |
| 700 | `sf-font-bold` |

### Typography atoms — use before raw elements

| Atom | When to use |
|---|---|
| `<Heading>` | Page-level h1 |
| `<SubHeading>` | Section headings (h2–h6) |
| `<Text>` | Body copy, descriptions, labels |
| `<TextLink>` | Anchor/link text |

> **`<Text>` already applies `sf-text-sm sf-text-primary-foreground`** — do NOT add a color class to it unless explicitly overriding. The CSS cascade silently produces the wrong color.

---

## 2. Spacing

Tailwind spacing: **1 unit = 4px**. Divide Figma px by 4.

| Figma px | Tailwind |
|---|---|
| 4px | `sf-p-1` / `sf-gap-1` |
| 8px | `sf-p-2` / `sf-gap-2` |
| 12px | `sf-p-3` / `sf-gap-3` |
| 16px | `sf-p-4` / `sf-gap-4` |
| 24px | `sf-p-6` / `sf-gap-6` |
| 32px | `sf-p-8` |

For values off the 4px grid, use bracket notation: `sf-p-[30px]`. Use sparingly.

---

## 3. Border Radius

| Figma px | Tailwind |
|---|---|
| 4px | `sf-rounded` |
| 6px | `sf-rounded-md` |
| 8px | `sf-rounded-lg` |
| 10px | `sf-rounded-[10px]` |
| 12px | `sf-rounded-xl` |
| 9999px / pill | `sf-rounded-full` |

> **Card containers must use `sf-rounded-[10px]`** — do not use `sf-rounded-lg` (8px) or `sf-rounded-xl` (12px).

---

## 4. Shadows

| Figma shadow | Tailwind |
|---|---|
| Subtle / small elevation | `sf-shadow-sm` |
| Standard card | `sf-shadow-md` |
| Floating / modal | `sf-shadow-lg` |
| Template / email card | `sf-shadow-template-card` |

---

## 5. Icons

**Library: `lucide-react`** — the only icon library. Never inline SVG paths.

```tsx
import { ChevronRight, Settings } from "lucide-react"

<ChevronRight size={16} className="sf-text-webtoffee" />
<Settings size={20} strokeWidth={1.5} className="sf-text-muted-foreground" />
```

- `size` prop for dimensions (not `sf-w-` / `sf-h-` on the icon).
- `className` with token for color — never `stroke="#hex"`.
- Icon buttons: `sf-text-primary-foreground` on icon, `sf-border-webtoffee-border` on button container.

---

## 6. Layout

Never use absolute positioning to recreate a Figma flex/grid layout.

| Figma | Code |
|---|---|
| Auto layout — horizontal | `sf-flex sf-flex-row` |
| Auto layout — vertical | `sf-flex sf-flex-col` |
| Alignment: center | `sf-items-center sf-justify-center` |
| Space between | `sf-justify-between` |
| Gap | `sf-gap-{n}` |
| Fill width | `sf-w-full` |

---

## 7. Images

Always use `getMediaUrl()` for project-hosted assets:

```tsx
import { getMediaUrl } from "@/utils/mediaUrl"
<img src={getMediaUrl("illustrations/empty-state.svg")} alt="No data" />
```

---

## 8. Existing Components — Reuse Before Creating

### Atoms (`src/atoms/`)

| Need | Use |
|---|---|
| Button variants | `Button` |
| Text input | `Input` |
| Icon-only button | `IconButton` |
| Page heading | `Heading` |
| Section heading | `SubHeading` |
| Body text | `Text` |
| Card container | `Card`, `SettingsCard` |
| Loading | `LoadingStrip`, `WavyDotsLoader` |

### Molecules (`src/molecules/`)

| Need | Use |
|---|---|
| Collapsible search / filter bar | `src/molecules/Automations/TemplateFilterBar.tsx` |
| Date range picker | `DateRangePicker` |
| Stats card | `StatsCard` |
| Form field + label + error | `FormField` |

### Critical patterns

- **Dropdown filters** → `Select` + `SelectTrigger` (shadcn/ui) — NOT `DropdownMenu` + `Button` (causes blue-border side effect)
- **Table pagination** → `src/organisms/PaginationTemplate.tsx`
- **Row action menus** → search for `MoreVertical` patterns in `src/molecules/`
- **Radio buttons** → `RadioGroup` + `RadioGroupItem` from `src/components/ui/radio-group.tsx`

---

## 9. Gradients

```tsx
// Only defined gradient — navy-to-blue diagonal
<div className="sf-bg-primary-gradient" />
```

Do not reproduce as inline style.

---

## 10. Responsive Design

The app is desktop-first (1280px+). Apply responsive classes only when Figma shows a mobile/tablet variant. Do not add responsive variants speculatively.

---

## 11. Fidelity Rules (zero-tolerance — these cause visible regressions)

- **Never add UI elements not in Figma.** No toggles, switches, radio buttons, or badges unless explicitly shown.
- **Verify left-to-right element order.** Match button/icon order in every header and toolbar row exactly.
- **Verify save button placement.** Inside or outside card/panel? Left- or right-aligned? Do not default to `sf-self-end` without confirming.
- **Copy text verbatim.** Never paraphrase titles, subtitles, or descriptions.
- **Handle nested routes.** If a new route is a child of an existing parent, the parent must render `<Outlet />`. Add a `useLocation()` guard: if `pathname !== "/parent-path"` render `<Outlet />`, else render the parent template.
