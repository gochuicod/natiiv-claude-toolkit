---
name: infigroup-performance
description: InfiGroup Next.js performance, Core Web Vitals, and accessibility skill. Covers SSG/ISR rendering strategy, image optimization, font loading, CSP vs static-rendering conflicts, WCAG AA contrast/touch-targets/heading order, caching headers, and code-splitting. Use for any Lighthouse audit, Core Web Vitals fix, or new project performance setup.
argument-hint: "[audit | ssg | images | csp | a11y | caching]"
---

# InfiGroup Performance + Accessibility System

## Role

Performance engineer who:
- Converts dynamic-by-accident pages to SSG/ISR without breaking functionality
- Eliminates duplicate image loading and font-loading layout shift
- Resolves CSP configurations that silently fight static rendering
- Runs a scored Lighthouse/WCAG checklist and fixes what's found
- Templates proven patterns so the same class of bug isn't re-discovered per project

**Sources this skill is built from:** a Powershop session that took Performance 52→100 and LCP 9.1s→1.8s via SSG conversion, and a ShipX-next audit (2026-07-06) of a site that already scores well in production — used to separate "theory" from "what a real, shipped InfiGroup site actually does." Where the two disagreed, both are documented so you can judge the trade-off instead of guessing.

---

## Phase 0: Before Any Performance Work

Audit existing state first:
```
src/app/[locale]/layout.tsx   ← calls headers()/cookies()? generateStaticParams present?
src/app/[locale]/page.tsx     ← generateStaticParams + revalidate set?
next.config.ts                ← headers(), images config, CSP
src/middleware.ts             ← nonce generation? locale-only?
src/lib/fonts.ts (or equiv)   ← next/font usage, display strategy
```

Run Lighthouse before touching anything, so you have a baseline to compare against:
```bash
CHROME_PATH="/Applications/Brave Browser.app/Contents/MacOS/Brave Browser" \
  lighthouse http://localhost:3000 \
  --output=html --output-path=./lighthouse-report.html \
  --preset=desktop --no-enable-error-reporting
```
Note: use `CHROME_PATH` env var — `--chrome-path` CLI flag was removed in Lighthouse 13.

---

## Phase 1: Rendering Strategy — SSG/ISR

### The biggest single win: kill accidental dynamic rendering

**Rule:** `await headers()` or `await cookies()` anywhere in a shared layout makes **every route under that segment dynamic**, even if the result goes unused.

**Why it matters:** Powershop went from LCP 9.1s → 1.8s, TTFB 660ms → 10ms from this one fix (dynamic → static). The layout was extracting a CSP nonce that nothing downstream actually consumed.

**How to apply — minimum viable static layout:**
```tsx
// src/app/[locale]/layout.tsx
export function generateStaticParams() {
  return siteConfig.locales.map((locale) => ({ locale }));
}
export default async function LocaleLayout({ children, params }) {
  const { locale } = await params;
  setRequestLocale(locale); // fine — doesn't force dynamic
  // NOT fine unless you truly need it: const nonce = (await headers()).get('x-nonce')
  return <html lang={locale}>...</html>;
}
```
Audit: if the only use of `headers()` is a nonce that's unused downstream, delete it entirely (see Phase 3, CSP).

### Pair generateStaticParams with revalidate — the ShipX pattern

Static isn't binary. ShipX-next runs every locale page through `generateStaticParams`, then tunes `revalidate` per content type:
```ts
// Static marketing content, changes rarely
export const revalidate = 86400; // daily

// CMS/editor-updated content
export const revalidate = 3600; // hourly
```
For CMS-backed dynamic params (e.g. blog slugs from a headless CMS), wrap the build-time fetch in try/catch so a CMS outage at build time can't fail the whole build — fall back to `return []` and let ISR generate pages on first request:
```ts
export async function generateStaticParams() {
  try {
    const posts = await cms.getAllSlugs();
    return posts.map((p) => ({ slug: p.slug }));
  } catch {
    return []; // build-safe: ISR fills these in on-demand
  }
}
```
Wrap the actual per-request CMS fetch in `unstable_cache` for request-level dedupe on top of ISR.

**Verify it worked:** `npm run build` output should show `○` (static) or `●` (SSG w/ generateStaticParams) for your routes, not `ƒ` (dynamic), for anything that doesn't genuinely need per-request data.

---

## Phase 2: Images

### Do
- `<Image priority>` on the LCP element (usually the hero image) — Next.js auto-generates `fetchpriority="high"` + its own `<link rel="preload">`. Don't add `fetchPriority` manually on top; it's redundant.
- Set `sizes` on every `fill` image. **Real gap found in production (ShipX):** several `fill`-mode hero/photo components had no `sizes` prop, silently falling back to `sizes="100vw"` — over-fetching a full-viewport-width image even where it renders at a fraction of that on desktop.
  ```tsx
  <Image src={...} fill sizes="(max-width: 768px) 100vw, 50vw" />
  ```
- Configure modern formats in `next.config.ts`:
  ```ts
  images: { formats: ['image/avif', 'image/webp'], remotePatterns: [...] }
  ```

### Don't
**Anti-pattern — duplicate preload:** never add a manual `<link rel="preload" href="/raw-image.jpg">` for an image already rendered with `<Image priority>`. This downloads the raw original AND the Next-optimized version, wasting bandwidth and hurting LCP instead of helping it. (Powershop root cause of a real regression.)

**Gap to close, not yet solved anywhere audited:** `placeholder="blur"` (LQIP blur-up) — neither ShipX nor Powershop use it. Worth adding for any image-heavy page (product galleries, hero photography) where a visible pop-in is noticeable.

---

## Phase 3: Fonts

**Rule:** Self-host via `next/font/google` (or `next/font/local`), always set `display: 'swap'`, expose as a CSS variable, wire once in the root layout.

```ts
// src/lib/fonts.ts
export const bodyFont = Inter({ subsets: ['latin'], variable: '--font-sans', display: 'swap' });
```
```tsx
<html className={`${bodyFont.variable} ${headingFont.variable}`}>
```

**Why self-host, not a `<link>` to Google's CDN:** avoids a runtime network request to `fonts.googleapis.com`/`fonts.gstatic.com` entirely — both the request itself and, for InfiGroup's SEA/regional client base, the real risk that Google Fonts is throttled or blocked in-region, silently falling back to system fonts. Document this explicitly in the project's CLAUDE.md as a banned pattern once fixed, so nobody reintroduces a CDN `@import`.

**Check for drift:** if a static/permissive CSP still whitelists `fonts.googleapis.com` in `style-src`/`font-src` after switching to self-hosted fonts, that's dead config — harmless but worth cleaning up so the CSP reflects what's actually loaded.

---

## Phase 4: CSP vs Static Rendering

Two valid approaches — pick one, don't half-do a nonce:

### Option A — static/permissive CSP (no nonce)
Defined once in `next.config.ts` `headers()`, applied by path-matching, not baked per-request:
```ts
"script-src 'self' 'unsafe-inline' 'unsafe-eval' https://your-analytics-domain.com",
```
**Trade-off:** `'unsafe-inline'`/`'unsafe-eval'` is a weaker CSP posture (common when you need inline scripts for GTM/analytics), but it has zero nonce/SSG conflict because there's no nonce to go stale. This is what ShipX-next actually ships in production.

### Option B — nonce-based CSP
**Anti-pattern:** nonce-based CSP is fundamentally incompatible with SSG. SSG bakes the build-time nonce into static HTML; middleware generates a *new* per-request nonce for the response header. They never match → every inline script trips a CSP violation → Lighthouse Best Practices penalty. (This was the exact bug Powershop's 52-score baseline had.)

**Fix:** either drop nonces for hash-based CSP (compute a SHA-256 hash of each inline script/style at build time, allowlist the hash — works fine with SSG since hashes don't rotate per-request), or drop nonce extraction from the layout entirely and accept Option A's trade-off.

**Never do:** keep a nonce AND SSG at the same time and hope Lighthouse doesn't notice.

---

## Phase 5: Accessibility (WCAG AA)

### Color contrast
**Rule:** small/body text needs ≥4.5:1 contrast against its background. Tokens don't guarantee this by themselves — audit every hardcoded hex too.

**Known-good muted text color:** `#6b7280` on white = 4.63:1 (passes). `#787983` = 4.36:1 (fails) — a one-character-looking difference that fails the audit.

**Real gap found in production (ShipX):** primary body/heading tokens were fine (7.8:1, 14.6:1), but caption text and form placeholders used `#9ca3af`/`#99a1af` directly in CSS outside the token system — both land at ~2.5:1, a hard fail. **Placeholder and caption text are the most common place low-contrast hex sneaks in** because they feel "secondary" and get hand-tuned outside the design token system. Grep for hardcoded hex in CSS files even when the main token palette is compliant.

### Touch targets (WCAG 2.5.5 / 2.5.8)
**Rule:** every interactive element needs a ≥24×24px hit area, even if it's visually smaller.

**Real gap found in production (ShipX):** carousel-dot pagination indicators across multiple components were implemented at 8×8px or 8-10px with `padding: 0` — visually correct, but the actual clickable/tappable area was under the minimum, and this was on mobile-only UI (exactly where it matters most).

**Fix — wrapper pattern:**
```tsx
<button className="w-6 h-6 flex items-center justify-center" aria-label={`Go to slide ${i + 1}`}>
  <span className="w-2 h-2 rounded-full block" />
</button>
```

**Compounding gotcha to check for:** don't put `aria-hidden="true"` on a wrapper `<div>` that contains real interactive `<button>` elements with their own `aria-label`s. The `aria-hidden` ancestor hides the buttons from assistive tech entirely, making the labels unreachable — found live in one ShipX carousel where the dots' `aria-label`s were completely inaccessible because of this. If the wrapper is decorative, don't mark it `aria-hidden` when its children are actionable; mark truly decorative-only elements (icons, dividers) instead.

### Heading order
**Rule:** one `<h1>` per page, section headings `<h2>`, card/subsection headings step down one level at a time — never skip (`h2` → `h4`).

**Verification approach that scales:** rather than manually reading every component, sample 2-3 representative pages and trace the heading tree top to bottom. A page with a single hero `<h1>`, section `<h2>`s, and nested-card `<h3>`s (never jumping straight to `<h4>`) is the target shape.

---

## Phase 6: Caching Headers

A tiered `Cache-Control` strategy in `next.config.ts` `headers()`, matched by path — this is the highest-leverage caching pattern found in production (ShipX):

```ts
async headers() {
  return [
    { source: '/api/:path*', headers: [{ key: 'Cache-Control', value: 'no-store' }] },
    { source: '/admin/:path*', headers: [{ key: 'Cache-Control', value: 'no-store' }] },
    {
      // HTML pages — short s-maxage relies on CDN purge-on-deploy to stay fresh
      source: '/((?!_next/static|_next/image|favicon.ico|api|admin).*)',
      headers: [{ key: 'Cache-Control', value: 'public, s-maxage=60, stale-while-revalidate=86400' }],
    },
    {
      // Versioned/immutable static data (e.g. a bundled JSON dataset)
      source: '/data/:path*',
      headers: [{ key: 'Cache-Control', value: 'public, max-age=31536000, immutable' }],
    },
  ];
}
```
`_next/static` and `_next/image` need no explicit rule — Next.js already serves them with content-hashed immutable caching.

Pair with the ISR `revalidate` values from Phase 1 — caching headers control the CDN/browser layer, `revalidate` controls the origin regeneration layer. Both matter.

---

## Phase 7: Code-Splitting

**Rule:** reach for `next/dynamic({ ssr: false })` only for genuinely non-critical, non-SSR-safe visual effects — canvas animations, particle effects, map libraries. Don't split content that's needed for the initial paint or SEO.

```tsx
const RouteMapCanvas = dynamic(() => import('./RouteMapCanvas'), {
  ssr: false,
  loading: () => <div aria-hidden className="bg-gradient-to-b from-sky-50 to-white" />,
});
```
Give the `loading` fallback the same approximate background/dimensions as the real component to avoid layout shift while it streams in.

**On `loading.tsx` / route-level Suspense:** if the page is SSG/ISR (no per-request data fetch on the request path), you usually don't need a route `loading.tsx` skeleton at all — there's no server waterfall to mask. Reserve `<Suspense>` boundaries for client components that need one structurally (e.g. a component calling `useSearchParams()`), not as a general loading-state pattern.

---

## Technical Checklist

Run before marking performance work done:

### Rendering
- [ ] No `headers()`/`cookies()` in any shared layout unless the value is actually consumed downstream
- [ ] `generateStaticParams` present on every page that can be static
- [ ] `revalidate` set appropriately per content type (static marketing vs CMS-backed)
- [ ] CMS-backed `generateStaticParams` has a build-safe try/catch fallback
- [ ] `npm run build` output shows `○`/`●`, not `ƒ`, for pages with no genuine per-request need

### Images
- [ ] Hero/LCP image uses `<Image priority>`, no duplicate manual `<link rel="preload">`
- [ ] Every `fill` image has an explicit `sizes` prop matching its actual rendered width
- [ ] `next.config.ts` images config includes avif/webp formats
- [ ] Consider `placeholder="blur"` for image-heavy pages

### Fonts
- [ ] Fonts loaded via `next/font`, not a CDN `<link>`
- [ ] `display: 'swap'` set
- [ ] CSP `font-src`/`style-src` doesn't still whitelist a font CDN you no longer use

### CSP
- [ ] Either fully static/hash-based CSP, or fully nonce-based with per-page dynamic rendering — never nonce + SSG together
- [ ] Admin/auth-gated routes exempted from the marketing site's CSP if they need different rules

### Accessibility
- [ ] Body/heading text ≥4.5:1 contrast (check tokens AND hardcoded hex)
- [ ] Placeholder and caption text specifically checked — most common place low-contrast hex hides
- [ ] All interactive elements ≥24×24px hit area, including "decorative-looking" carousel dots
- [ ] No `aria-hidden` on a wrapper containing real interactive children
- [ ] No heading-level skips; one `<h1>` per page

### Caching
- [ ] Tiered `Cache-Control` in `next.config.ts` (no-store for API/admin, short s-maxage+SWR for HTML, long immutable for versioned assets)
- [ ] `_next/static`/`_next/image` left to Next's default immutable caching

### Bundle
- [ ] `next/dynamic({ ssr: false })` used only for genuinely non-critical/non-SSR-safe widgets
- [ ] No route `loading.tsx` masking a waterfall that SSG/ISR already eliminated

---

## Performance Commands (quick invocations)

| Command | Action |
|---------|--------|
| `audit` | Run Lighthouse + the full checklist above, score current state |
| `ssg` | Convert a dynamic page/layout to static — find and remove the `headers()`/`cookies()` culprit |
| `images` | Audit `<Image>` usage for missing `sizes`, duplicate preloads, missing `priority` |
| `csp` | Diagnose CSP vs SSG conflicts, choose static/hash vs nonce |
| `a11y` | Contrast + touch-target + heading-order pass |
| `caching` | Design/audit the tiered `Cache-Control` strategy in `next.config.ts` |
