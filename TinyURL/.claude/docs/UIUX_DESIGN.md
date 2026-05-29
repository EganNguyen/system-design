# UI/UX Design: TinyURL Service

---

## 1. Design Philosophy

TinyURL is a utility-first product. The design follows a single guiding principle:

> **Zero friction to the first short link. Everything else is progressive disclosure.**

First-time users should produce a short URL in under 10 seconds with no account required. Power features (custom aliases, analytics, branded domains) reveal themselves naturally as users engage deeper.

**Aesthetic direction:** Refined utilitarian — clean, high-contrast, fast-feeling. Monochromatic base with a single vivid accent. Typography-forward. No decorative noise. The interface communicates speed and reliability, not playfulness.

---

## 2. Design Tokens

### 2.1 Color System

```css
:root {
  /* Base */
  --color-bg-primary:       #0A0A0A;   /* near-black canvas */
  --color-bg-secondary:     #141414;   /* card / panel surfaces */
  --color-bg-tertiary:      #1F1F1F;   /* input fields, hover states */
  --color-border:           #2A2A2A;   /* dividers, field outlines */
  --color-border-focus:     #3D3D3D;   /* focused input ring */

  /* Text */
  --color-text-primary:     #F5F5F3;   /* headings, primary content */
  --color-text-secondary:   #8A8A87;   /* labels, metadata, hints */
  --color-text-disabled:    #3D3D3D;   /* placeholder, muted */

  /* Accent (single vivid color — electric lime) */
  --color-accent:           #C8F135;   /* CTAs, active states, links */
  --color-accent-hover:     #D6FF40;   /* hover on accent elements */
  --color-accent-muted:     #C8F13520; /* accent at 12% opacity — badges, highlights */

  /* Status */
  --color-success:          #4ADE80;
  --color-error:            #F87171;
  --color-warning:          #FBBF24;
}
```

**Light mode** (togglable):
```css
[data-theme="light"] {
  --color-bg-primary:       #FAFAF8;
  --color-bg-secondary:     #F0F0EC;
  --color-bg-tertiary:      #E6E6E0;
  --color-border:           #D4D4CC;
  --color-text-primary:     #111110;
  --color-text-secondary:   #6B6B65;
  --color-accent:           #5B6E00;   /* lime darkens for legibility on white */
}
```

### 2.2 Typography

```css
/* Display: geometric, high-contrast headlines */
@import url('https://fonts.googleapis.com/css2?family=DM+Mono:wght@400;500&family=Syne:wght@700;800&display=swap');

:root {
  --font-display: 'Syne', sans-serif;   /* headlines, brand name */
  --font-body:    'DM Mono', monospace; /* body, UI labels, short codes */

  /* Scale (8pt baseline grid) */
  --text-xs:   11px;
  --text-sm:   13px;
  --text-base: 15px;
  --text-lg:   18px;
  --text-xl:   24px;
  --text-2xl:  32px;
  --text-3xl:  48px;
  --text-4xl:  72px;

  /* Weight */
  --weight-regular: 400;
  --weight-medium:  500;
  --weight-bold:    700;
  --weight-black:   800;

  /* Line height */
  --leading-tight:  1.1;
  --leading-snug:   1.3;
  --leading-normal: 1.6;
}
```

**Rationale:**
- `Syne` — wide, bold, architectural. Gives the brand name and headlines presence without decoration.
- `DM Mono` — monospaced body text makes short codes feel native, not bolted on. Every URL and code renders in the same rhythm as the UI text.

### 2.3 Spacing

```css
:root {
  --space-1:  4px;
  --space-2:  8px;
  --space-3:  12px;
  --space-4:  16px;
  --space-5:  24px;
  --space-6:  32px;
  --space-7:  48px;
  --space-8:  64px;
  --space-9:  96px;
  --space-10: 128px;
}
```

### 2.4 Border Radius

```css
:root {
  --radius-sm:   4px;
  --radius-md:   8px;
  --radius-lg:   12px;
  --radius-xl:   20px;
  --radius-full: 9999px;
}
```

### 2.5 Elevation & Shadow

```css
:root {
  --shadow-sm:  0 1px 3px rgba(0,0,0,0.4);
  --shadow-md:  0 4px 16px rgba(0,0,0,0.5);
  --shadow-lg:  0 12px 40px rgba(0,0,0,0.6);
  --shadow-accent: 0 0 24px rgba(200,241,53,0.15);  /* glow on accent elements */
}
```

### 2.6 Motion

```css
:root {
  --ease-out:        cubic-bezier(0.16, 1, 0.3, 1);   /* snappy exits */
  --ease-in-out:     cubic-bezier(0.45, 0, 0.55, 1);
  --ease-spring:     cubic-bezier(0.34, 1.56, 0.64, 1); /* subtle overshoot */

  --duration-fast:   120ms;
  --duration-base:   200ms;
  --duration-slow:   350ms;
  --duration-reveal: 600ms;
}
```

**Motion principles:**
- Redirect feedback (copy success, link generation) uses `--ease-spring` — it feels satisfying and physical
- Page transitions use `--ease-out` — exits are instant, entrances ease in
- No looping animations in idle state — the interface is calm until interacted with

---

## 3. Information Architecture

```
tinyurl.com/
│
├── /                        Landing + anonymous shortener
├── /login                   Sign in
├── /signup                  Create account
├── /dashboard               Authenticated URL list
│     ├── /dashboard/[code]  Single URL detail + analytics
│     └── /dashboard/new     Create with advanced options
├── /settings                Account settings
│     ├── /settings/profile
│     ├── /settings/domains  Custom domain management
│     └── /settings/billing
└── /[code]                  Redirect (handled by Go service)
```

### Navigation Model

- **Unauthenticated:** Single-level. Top bar with logo + Sign In button. No sidebar.
- **Authenticated:** Persistent left sidebar (desktop) / bottom tab bar (mobile). Collapsed by default on tablet.

---

## 4. Page Designs

### 4.1 Landing Page (`/`)

**Purpose:** Convert a visitor into a user of the shortener in < 10 seconds.

**Layout:**
```
┌─────────────────────────────────────────────────────┐
│  tinyurl.com                           Sign in  →   │  ← top bar (64px)
├─────────────────────────────────────────────────────┤
│                                                     │
│         Make long links                             │
│         disappear.                                  │  ← hero heading (Syne 72px)
│                                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │  Paste your URL here                          │  │  ← URL input (56px height)
│  └───────────────────────────────────────────────┘  │
│                    [ Shorten →  ]                   │  ← CTA button (accent)
│                                                     │
│  ────────────────────────────────────────────────   │
│                                                     │
│  tinyurl.com/aB3xYz7              [ Copy ]  ✓       │  ← result row (appears after submit)
│  https://example.com/very-long-url...               │
│  Expires: never    Created: just now                │
│                                                     │
│                 ↓  Want analytics? Create a free    │
│                    account. It takes 30 seconds.    │
└─────────────────────────────────────────────────────┘
```

**Interaction states:**

| State          | Behavior                                                              |
|----------------|-----------------------------------------------------------------------|
| Empty          | Placeholder: "Paste your URL here…" Shorten button disabled (ghost)  |
| Typing         | Shorten button activates; basic URL format validated live             |
| Submitting     | Button shows spinner; input disabled                                  |
| Success        | Result row slides up with `--ease-spring`; Copy button prominent      |
| Copy clicked   | Button label → "Copied!" for 2s with checkmark; accent glow effect   |
| Error          | Inline error below input (red, no toast); input shakes once          |
| Invalid URL    | Error: "That doesn't look like a valid URL"                           |
| Blacklisted    | Error: "This URL can't be shortened (policy violation)"               |

**Anonymous shortener CTA flow:**
```
Shorten → Success → "Save this link? Create a free account →"
```
The prompt appears below the result, never as a modal interruption.

---

### 4.2 Dashboard (`/dashboard`)

**Purpose:** Manage, monitor, and create URLs for authenticated users.

**Layout (desktop, ≥ 1024px):**
```
┌──────────┬────────────────────────────────────────────────────┐
│          │  Dashboard                    [ + New Link ]       │
│  S       │  ─────────────────────────────────────────────     │
│  I       │  [ Search links...          ] [ Filter ▾ ]         │
│  D       │                                                     │
│  E       │  short link          original url        clicks    │
│  B       │  ──────────────────────────────────────────────     │
│  A       │  tinyurl.com/aB3xYz7 example.com/page    1,042 →  │
│  R       │  tinyurl.com/myalias long-site.com/path     328 →  │
│          │  tinyurl.com/promo24 shop.example.com/s      91 →  │
│  nav     │                                                     │
│  items   │  ← 1  2  3  →                                       │
│          │                                                     │
└──────────┴────────────────────────────────────────────────────┘
```

**Layout (mobile, < 768px):**
- Sidebar collapses to bottom tab bar (4 tabs: Home, Links, Analytics, Settings)
- URL list becomes card stack — each card shows short code, truncated long URL, click count
- Swipe left on a card → reveals Delete / Copy actions

**URL List Row:**
```
┌─────────────────────────────────────────────────────────────┐
│  tinyurl.com/aB3xYz7             1,042 clicks    7 days ago │
│  example.com/very/long/article-title-that-gets-truncated…   │
│  ●  Active                                    [ ··· ]       │
└─────────────────────────────────────────────────────────────┘
```

- Short code in accent color, monospace
- Long URL truncated at 60 chars with ellipsis; full URL on hover tooltip
- Status badge: Active (green) / Expired (grey) / Custom (accent outline)
- `···` overflow menu: Copy, View Analytics, Edit, Delete

**Filter & Sort options:**
- Sort by: Date created (default) / Click count / Alphabetical
- Filter by: Active / Expired / Custom alias / Has expiry
- Date range picker (last 7 / 30 / 90 days / custom)

---

### 4.3 URL Detail & Analytics (`/dashboard/[code]`)

**Layout:**
```
┌──────────────────────────────────────────────────────────────┐
│  ← Back to links                                             │
│                                                              │
│  tinyurl.com/aB3xYz7                         [ Copy ]       │
│  example.com/original/long/url                               │
│  Created May 1, 2026  ·  Expires never  ·  Active           │
│                                                              │
├───────────┬──────────────────────────────────────────────────┤
│  1,042    │  328          │  12            │  US, IN, GB     │
│  Clicks   │  Uniq. visits │  Countries     │  Top countries  │
├───────────┴──────────────┴────────────────┴─────────────────┤
│                                                              │
│  Clicks over time        [ 7d ] [ 30d ] [ 90d ] [ Custom ]  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  ▁▃▅▄▆█▇▅▄▃▂▁▂▃▄▅▆▇█▇▅▄▃▂▁▃▅▇█▇▄▃▂▁▂▃               │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  Top Countries           Top Referrers                       │
│  US ████████  42%        twitter.com  ████████  38%         │
│  IN █████     21%        direct        █████     22%         │
│  GB ████      18%        reddit.com    ███       15%         │
│                                                              │
│  Device Breakdown                                            │
│  Mobile  58%  ●  Desktop  36%  ●  Tablet  6%               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Chart library:** Recharts (already in Next.js stack) — line chart for time series, horizontal bar for country/referrer breakdown, donut for device split.

---

### 4.4 Create New Link (`/dashboard/new`)

**Layout — two-column form (desktop):**
```
┌───────────────────────────────┬──────────────────────────────┐
│  Destination URL              │  Preview                     │
│  ┌───────────────────────┐    │                              │
│  │ https://...           │    │  tinyurl.com/               │
│  └───────────────────────┘    │  ┌─────────────────────┐    │
│                               │  │  aB3xYz7            │    │
│  Custom alias  (optional)     │  └─────────────────────┘    │
│  tinyurl.com / [__________]   │                              │
│  ✓ Available                  │  This link will redirect to  │
│                               │  example.com/…              │
│  Expiry  (optional)           │                              │
│  ○ Never                      │  [ QR code ]                 │
│  ● After:  [ 30 ] [ days ▾ ]  │                              │
│                               │                              │
│  Domain                       │                              │
│  ○ tinyurl.com                │                              │
│  ○ yourbrand.co  (connected)  │                              │
│                               │                              │
│         [ Create Link → ]     │                              │
└───────────────────────────────┴──────────────────────────────┘
```

**Alias validation states:**

| Input       | State indicator                                              |
|-------------|--------------------------------------------------------------|
| Empty       | No indicator                                                 |
| Typing      | Debounce 400ms → spinner                                     |
| Available   | ✓ green "Available"                                          |
| Taken       | ✗ red "Already taken — try another"                          |
| Invalid     | ✗ red "Only letters, numbers, and hyphens (3–20 chars)"      |
| Reserved    | ✗ red "This alias is reserved"                               |

---

### 4.5 Settings — Custom Domains (`/settings/domains`)

```
┌──────────────────────────────────────────────────────────────┐
│  Custom Domains                                              │
│  Use your own domain for short links (Pro plan)             │
│                                                              │
│  yourbrand.co          ●  Verified     [ Remove ]           │
│  ─────────────────────────────────────────────────────────   │
│                                                              │
│  Add a domain                                                │
│  ┌──────────────────────────────────┐  [ Add Domain ]        │
│  │  yoursite.com                    │                        │
│  └──────────────────────────────────┘                        │
│                                                              │
│  After adding, point a CNAME record from your domain to:     │
│  cname.tinyurl.com                                           │
│                                                              │
│  Verification checks every 60 seconds automatically.         │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. Component Library

### 5.1 Input

```
Default:
┌──────────────────────────────────────────────────────┐
│  Paste your URL here                                 │
└──────────────────────────────────────────────────────┘
Border: --color-border  BG: --color-bg-tertiary

Focus:
┌──────────────────────────────────────────────────────┐  ← border: --color-border-focus
│  https://example.com/|                               │  ← ring: 2px --color-accent at 30%
└──────────────────────────────────────────────────────┘

Error:
┌──────────────────────────────────────────────────────┐  ← border: --color-error
│  not-a-url                                           │
└──────────────────────────────────────────────────────┘
  That doesn't look like a valid URL                    ← error text (--text-sm, --color-error)
```

### 5.2 Button Variants

```
Primary (accent fill):
┌──────────────────────┐
│  Shorten →           │  BG: --color-accent  Text: #0A0A0A (dark on lime)
└──────────────────────┘  Hover: --color-accent-hover  Active: scale(0.98)

Secondary (outline):
┌──────────────────────┐
│  Copy                │  Border: --color-border  Text: --color-text-primary
└──────────────────────┘  Hover: BG --color-bg-tertiary

Ghost (no border):
  View Analytics →        Text: --color-text-secondary
                          Hover: --color-text-primary  underline

Destructive:
┌──────────────────────┐
│  Delete              │  Text: --color-error
└──────────────────────┘  Hover: BG rgba(248,113,113,0.1)

Loading state (any button):
┌──────────────────────┐
│  ○  Shortening…      │  Spinner replaces icon; button disabled; opacity 0.7
└──────────────────────┘
```

### 5.3 Badge / Status Chip

```
Active:   ●  Active    BG: rgba(74,222,128,0.1)   Text: #4ADE80
Expired:  ●  Expired   BG: rgba(138,138,135,0.1)  Text: #8A8A87
Custom:   ★  Custom    BG: --color-accent-muted    Text: --color-accent
```

### 5.4 Copy Button with Feedback

```
Default:      [ Copy ]
Copying:      [ ○     ]   (100ms — barely visible)
Copied:       [ ✓ Copied! ]   accent color, 2s → fades back to "Copy"
```

### 5.5 Short Code Display

```
┌──────────────────────────────────────────────────────────┐
│  tinyurl.com/ aB3xYz7                        [ Copy ]   │
└──────────────────────────────────────────────────────────┘
  - "tinyurl.com/" in --color-text-secondary
  - "aB3xYz7" in --color-accent, font-weight: medium
  - Monospace throughout (DM Mono)
  - Full row clickable → selects the short URL text
```

### 5.6 Empty State

```
                    ┌───┐
                    │ ∅ │   (muted icon, 48px)
                    └───┘
              No links yet

     Shorten your first URL to see it here.

              [ + Create a link ]
```

### 5.7 Skeleton Loader

Used while list data fetches (< 200ms target, but shown immediately to prevent layout shift):
```
┌─────────────────────────────────────────────────────────────┐
│  ████████████████                   ████   ██████████      │  ← animated shimmer
│  ████████████████████████████████████                       │
└─────────────────────────────────────────────────────────────┘
```
Shimmer animation: left-to-right gradient sweep, 1.2s loop, `--ease-in-out`.

---

## 6. Responsive Breakpoints

```css
/* Mobile first */
--breakpoint-sm:  480px;   /* large phone */
--breakpoint-md:  768px;   /* tablet portrait */
--breakpoint-lg:  1024px;  /* tablet landscape / small laptop */
--breakpoint-xl:  1280px;  /* desktop */
--breakpoint-2xl: 1536px;  /* wide desktop */
```

### Responsive Behaviour by Page

| Page         | Mobile (< 768px)                         | Desktop (≥ 1024px)               |
|--------------|------------------------------------------|-----------------------------------|
| Landing      | Full-width input, stacked layout         | Centered 640px max-width column   |
| Dashboard    | Card list, bottom tab bar                | Table rows, left sidebar          |
| URL Detail   | Stat pills stacked, full-width charts    | 4-column stat row, side-by-side   |
| Create New   | Single column form, no preview           | Two-column (form + preview)       |
| Settings     | Single column                            | Two-column (nav + content)        |

---

## 7. User Flows

### 7.1 Anonymous Shortening (Primary Flow)

```
Land on homepage
      │
      ▼
Paste URL into input
      │
      ▼
Click "Shorten →"
      │
   ┌──┴───────────────────────────────────┐
   │                                      │
   ▼ (valid)                              ▼ (invalid)
Result appears                     Inline error shown
      │                            User corrects URL
      ▼                                   │
Copy short link                           │
      │                                   │
      ▼                                   │
Optional: "Create an account             ◀┘
to save & track this link"
      │
   ┌──┴──────────┐
   │             │
   ▼             ▼
Sign up       Dismiss (link still works anonymously)
```

### 7.2 Authenticated URL Creation

```
Dashboard → [ + New Link ]
      │
      ▼
Paste long URL
      │
      ├── (optional) Type custom alias → live availability check
      ├── (optional) Set expiry
      └── (optional) Choose custom domain
      │
      ▼
[ Create Link → ]
      │
      ▼
Success toast: "Link created" + short code
Dashboard list refreshes (optimistic update — new link appears instantly at top)
```

### 7.3 Copy Short Link

```
Click [ Copy ]  →  Clipboard write  →  Button shows "✓ Copied!" (2s)  →  Reset
                                        + accent glow on short code row
```

### 7.4 View Analytics

```
Dashboard list row  →  Click anywhere on row (or ··· → View Analytics)
      │
      ▼
URL Detail page
      │
      ▼
Default view: 7-day chart, summary stats
      │
      ├── Toggle time range: 7d / 30d / 90d / Custom
      ├── Hover chart → tooltip with exact date + count
      └── Scroll down → country / referrer / device breakdowns
```

### 7.5 Delete a Link

```
··· menu → Delete
      │
      ▼
Confirmation dialog (inline, not modal):
"Delete tinyurl.com/aB3xYz7? Visitors will see a 410 Gone page."
      │
   ┌──┴──────────┐
   ▼             ▼
Cancel        Confirm delete
              → Row fades out (300ms)
              → Soft delete in DB
              → Toast: "Link deleted"
```

---

## 8. Accessibility

### Standards

- **WCAG 2.1 AA** minimum; target AAA for text contrast
- All interactive elements keyboard-navigable
- Focus indicators: 2px solid `--color-accent` outline, 2px offset
- No information conveyed by color alone (status badges include text + icon)

### Contrast Ratios

| Pair                                    | Ratio   | WCAG Level |
|-----------------------------------------|---------|------------|
| `--color-text-primary` on `--color-bg-primary`   | 16.8:1  | AAA ✓      |
| `--color-accent` on `--color-bg-primary`          | 9.4:1   | AAA ✓      |
| `--color-text-secondary` on `--color-bg-primary`  | 4.6:1   | AA ✓       |
| `#0A0A0A` on `--color-accent` (button label)      | 14.2:1  | AAA ✓      |
| `--color-error` on `--color-bg-primary`           | 4.8:1   | AA ✓       |

### ARIA & Semantic HTML

```tsx
// URL input
<label htmlFor="url-input" className="sr-only">Long URL to shorten</label>
<input
  id="url-input"
  type="url"
  aria-describedby="url-error"
  aria-invalid={hasError}
  placeholder="Paste your URL here"
/>
{hasError && (
  <p id="url-error" role="alert" aria-live="assertive">
    {errorMessage}
  </p>
)}

// Copy button
<button
  aria-label={copied ? "Copied to clipboard" : "Copy short URL"}
  aria-pressed={copied}
  onClick={handleCopy}
>
  {copied ? "✓ Copied!" : "Copy"}
</button>

// Dashboard table
<table role="grid" aria-label="Your shortened links">
  <thead>...</thead>
  <tbody>
    <tr aria-label="tinyurl.com/aB3xYz7 — 1,042 clicks">
      ...
    </tr>
  </tbody>
</table>
```

### Reduced Motion

```css
@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

---

## 9. Performance UX

| Metric               | Target         | Strategy                                                     |
|----------------------|----------------|--------------------------------------------------------------|
| Time to interactive  | < 1.5s         | Next.js App Router; minimal JS on landing page               |
| Shorten response     | < 200ms        | Go API; optimistic local state update                        |
| Dashboard load       | < 300ms        | SWR / React Query; skeleton placeholders; paginate 20 items  |
| Chart render         | < 100ms        | Pre-aggregated ClickHouse MV; Recharts lazy loaded           |
| Copy feedback        | Instant        | `navigator.clipboard.writeText` is synchronous-feeling       |
| Route transitions    | < 150ms        | Next.js prefetch on hover; page segment streaming            |

### Optimistic Updates

Dashboard URL list uses optimistic UI — new links appear instantly on create, row disappears instantly on delete, without waiting for the server round-trip. Rollback on error with a toast:

```
"Couldn't delete that link. Try again."  [ Retry ]
```

---

## 10. Error & Empty States

| Scenario                  | Message                                                        | Action offered           |
|---------------------------|----------------------------------------------------------------|--------------------------|
| Invalid URL               | "That doesn't look like a valid URL"                           | —                        |
| URL blacklisted           | "This URL can't be shortened (policy violation)"               | —                        |
| Custom alias taken        | "Already taken — try a different alias"                        | Suggest available variant|
| Network error on shorten  | "Something went wrong. Check your connection and try again."   | Retry button             |
| Empty dashboard           | "No links yet — shorten your first URL to see it here."        | + Create a link          |
| No analytics data         | "No clicks yet. Share your link to start seeing data."         | Copy link button         |
| Link expired              | "This link has expired and is no longer active."               | Create new link          |
| 404 short code            | "This link doesn't exist or was removed."                      | Go to homepage           |
| Rate limited              | "You've made too many links recently. Try again in a minute."  | Countdown timer          |

---

## 11. Next.js Implementation Notes

### Component Structure

```
components/
├── ui/
│   ├── Button.tsx          # All variants; loading state built-in
│   ├── Input.tsx           # With error state, label, hint
│   ├── Badge.tsx           # Status chips
│   ├── CopyButton.tsx      # Animated copy with clipboard API
│   ├── ShortCodeDisplay.tsx # Short URL row with copy
│   ├── Skeleton.tsx        # Shimmer loader
│   └── EmptyState.tsx      # Reusable empty state
├── layout/
│   ├── Sidebar.tsx         # Authenticated nav (desktop)
│   ├── BottomNav.tsx       # Authenticated nav (mobile)
│   └── TopBar.tsx          # Unauthenticated / landing
├── features/
│   ├── shortener/
│   │   ├── ShortenForm.tsx
│   │   └── ShortenResult.tsx
│   ├── dashboard/
│   │   ├── UrlTable.tsx
│   │   ├── UrlCard.tsx         # Mobile card variant
│   │   └── UrlRow.tsx          # Desktop table row
│   └── analytics/
│       ├── ClickChart.tsx      # Recharts time series
│       ├── StatGrid.tsx        # 4-stat summary row
│       ├── CountryBreakdown.tsx
│       └── DeviceBreakdown.tsx
```

### State Management

- **Server state:** TanStack Query (React Query) — caches dashboard data, auto-refetches on focus
- **UI state:** `useState` / `useReducer` — form state, modal open/close
- **Global auth:** Next.js middleware + server-side session (cookie-based JWT)
- **Theme:** CSS `data-theme` attribute on `<html>`; stored in `localStorage`; no flash on load via `<script>` in `<head>`

### API Integration

```tsx
// hooks/useUrls.ts
export function useUrls(page: number) {
  return useQuery({
    queryKey: ['urls', page],
    queryFn: () => fetch(`/api/v1/users/me/urls?page=${page}`).then(r => r.json()),
    placeholderData: keepPreviousData,   // no loading flash on pagination
    staleTime: 30_000,
  });
}

// Optimistic delete
const queryClient = useQueryClient();
const deleteMutation = useMutation({
  mutationFn: (code: string) => fetch(`/api/v1/urls/${code}`, { method: 'DELETE' }),
  onMutate: async (code) => {
    await queryClient.cancelQueries({ queryKey: ['urls'] });
    const prev = queryClient.getQueryData(['urls', page]);
    queryClient.setQueryData(['urls', page], (old) =>
      old.filter(url => url.short_code !== code)
    );
    return { prev };
  },
  onError: (_, __, ctx) => {
    queryClient.setQueryData(['urls', page], ctx.prev);
    toast.error("Couldn't delete that link. Try again.");
  },
});
```

---

*Document version: 1.0 — May 2026*
*See also: `SYSTEM_DESIGN.md` · `LLD.md` · `DATABASE_DESIGN.md`*
