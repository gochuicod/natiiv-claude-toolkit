---
name: infigroup-design
description: InfiGroup design-to-code skill. Combines Anthropic frontend-design aesthetics + Impeccable anti-pattern enforcement + InfiGroup token system + Claude Design static-export handoff (primary) + Figma MCP workflow (fallback). Use for all UI work, Josh/Claude Design handoffs, component creation, landing pages, and any visual/layout task.
argument-hint: "[handoff | figma | component | audit | polish | bolder | quieter | animate | critique]"
---

# InfiGroup Design-to-Code — Full System

## Role

Senior design engineer who:
- Translates Josh's Figma designs to pixel-perfect, production-grade code
- Enforces anti-pattern rules before, during, and after coding
- Thinks in design tokens, not raw values
- Commits to bold aesthetic directions — never generic AI output
- Audits every implementation before marking done

---

## Phase 0: Design Thinking (before any code)

Answer these before touching a file:

1. **Purpose** — what problem does this interface solve? who uses it?
2. **Tone** — pick one extreme: brutally minimal / maximalist / retro-futuristic / organic / luxury / playful / editorial / brutalist / art deco / industrial / soft pastel / utilitarian
3. **Differentiation** — what's the ONE thing someone will remember?
4. **AI Slop Test** — first-order: can someone guess the theme+palette from the category alone? If yes, wrong direction. Second-order: can they guess the aesthetic family from category + anti-references? If yes, wrong direction.

**CRITICAL**: Commit to one direction. Execute with precision. Both maximalism and minimalism work — intentionality is everything.

---

## Design Source Routing

InfiGroup is moving off Figma. Before starting any handoff:

1. **Check for a downloaded Claude Design static export first** — typically `~/Downloads/<Project Name>/` containing `.html` + `.css` files (e.g. `about.html`, `journal.css`, `tokens.css`). If it exists → use **Phase 1a** below. This is now the **default, preferred path**.
2. **Only fall back to Figma (Phase 1b)** if no static export exists and a Figma URL is explicitly provided.

---

## Phase 1a: Claude Design Static Export Handoff (primary — replaces Figma for new work)

This is the workflow proven across the Arka Nirmal rebuild (journal listing, individual article pages, about page) — a full 1:1 port from a downloaded static HTML/CSS export into brand-branched Next.js components, verified pixel-for-pixel against the real design source.

### 1. Locate and triage the source
- Find the export folder (`~/Downloads/<Project Name>/`). It contains one `.html` + often a matching `.css` per page (`about.html`/`about.css`, `journal.html`/`journal.css`), plus shared `tokens.css`, `colors_and_type_<brand>.css`, `arka-theme.css`, and a `screenshots/` folder (usually just component/asset previews, not full-page mockups — don't rely on it).
- **Multi-variant pages** (e.g. a blog template with every post embedded as hidden `<div data-post="N" hidden>` blocks, toggled by a `?post=` query param): the WHOLE dataset is in one file. Check for this pattern before assuming you need per-post screenshots.
- **Small single-page files** (roughly under ~300 lines): read directly.
- **Large files** (roughly 500+ lines, or multi-variant): dispatch a research agent to extract into structured JSON — exact copy per content block (paragraph/heading/list/quote/etc., preserving inline tags), plus every CSS class's resolved property values (cross-reference `var(--x)` back to `tokens.css`/`colors_and_type_*.css` for literal hex/px numbers — don't leave it as "uses a variable"). This keeps the huge raw HTML out of your own context while still getting exact, verbatim values.

### 2. Map resolved values to existing tokens — don't invent new ones
- Resolved colors/shadows/radii from the export should already match your project's existing CSS custom properties (e.g. `--nirmal-forest-600: rgb(44,82,53)`) if the token system was built from the same design source. Verify a handful against `globals.css` before assuming a match; if they line up, use the existing var, never a raw hex.
- If a class only exists in the export CSS (no matching project token yet), that's a signal the previous page's port missed something — check sibling pages already ported for the same class before adding a new one.

### 3. Build brand-branched, following the existing split pattern
- Route file (`page.tsx`) branches on `resolveBrandFromLocale(locale).key` and renders `<NirmalXPage />` vs `<AdamXPage />` — mirrors how `/journal` and `/about` are structured.
- If the route previously had a single hardcoded (Adam-only) implementation, move it **verbatim** into its own `AdamXPage.tsx` first (zero content/behavior change) before adding the new Nirmal build — this guarantees no regression, and lets you screenshot-diff Adam before/after to prove it.
- Extract genuinely shared pieces (a card component, an icon set) into their own files rather than duplicating per page — but don't over-abstract content that's only used once.
- **Route-aware shared chrome**: if a shared layout piece (footer CTA, promo banner) needs different content per page, don't hardcode the first page's copy into the shared component and gate visibility with a boolean — use a small client component with `usePathname()` + a route→content lookup table (namespace, icon set, whatever varies). Adding a second page becomes a one-line table entry instead of a refactor.
- **Check shared nav/layout allowlists before rebuilding**: if a page seems to be missing chrome entirely (a promo banner, correct nav styling, expected spacing), check whether it's registered in whatever lookup array controls that behavior (e.g. a `SECONDARY_PAGES` list) before assuming the content itself needs rebuilding — an unregistered route can silently disable several unrelated-looking things at once (nav variant + banner + spacing all traced back to one missing array entry in this project).

### 4. Content and translation handling
- Inject the verbatim source-locale content first (matching the export exactly — don't paraphrase, this is production copy).
- Translate to other locales **yourself, directly** — do not trust a background agent's self-reported "done" without verifying via `git diff` that it actually wrote the files. (This bit us once: an agent reported 30 successful tool calls and full completion while writing nothing at all.)
- Validate every locale file is valid JSON and structurally complete (same keys present) before moving on.

### 5. QA: side-by-side against the REAL design, not memory
This is the part worth doing carefully — it catches real bugs, not just polish nits.

- **Serve the export over a local HTTP server, never `file://`**: `cd ~/Downloads/<Project>/ && python3 -m http.server <port>`. The `file://` protocol can silently break JS-dependent features (scroll-reveal `IntersectionObserver`s, custom elements) — a real, present design element can screenshot as if it's completely missing purely because of the protocol, not because it's actually absent. Always verify over HTTP before concluding something isn't in the design.
- Screenshot both the live Next.js page and the design source at the same viewport width with Playwright. For the design source, script a scroll-through first (`window.scrollTo` in a loop with short waits) to trigger any scroll-reveal animations before capturing.
- Compare full-page screenshots section by section.
- **When something looks like a color mismatch or a "missing" gap, pixel-sample instead of eyeballing.** Read the screenshot back in and sample actual RGB values at specific `(x, y)` coordinates (`PIL`/`Pillow` via a quick Python one-liner is enough). A "looks a bit off" impression is often a precise, fixable bug once you have the numbers.
- **Full-column scan for white gaps**: after a fix, sweep a vertical strip of pixels down the entire page height at left-edge, right-edge, and center x-positions, flagging any run of near-white pixels. This catches every remaining gap in one pass instead of fixing them one visual glance at a time.

### 6. Common real bugs this process catches (check these first)
- **Margin vs. padding on a colored section**: a child element's outer `margin` creates a gap that shows whatever's *behind* the section (usually white), even though the section itself has the right `background`. Fix by converting that spacing to `padding` on a wrapping element — margins don't get "filled" by a sibling's background, padding does.
- **A wrapper element with no background color at all** — easy to miss when the content inside (an image, a card) looks fine; the bug is in the empty margin/padding area around it.
- **`bg-[var(--custom-property)]/NN`** (a CSS custom property combined with a Tailwind opacity modifier) can silently resolve to a fully transparent background in some Tailwind setups — computed styles show `rgba(0,0,0,0)` with no error. If a translucent/glass effect isn't showing, check this before assuming a logic bug. Use an explicit `bg-[rgba(r,g,b,a)]` instead.
- **A decorative element meant to visually span two adjacent sections that each have `overflow-hidden` cannot actually cross that boundary.** Two independent instances (one per section) reads as visual duplication, not one continuous shape, since neither can bleed into the other's box. Either own it in one section only, or size/position two instances carefully to *look* continuous without literally being the same element.
- **IntersectionObserver "is this sticky element currently stuck" detection**: compare `entry.boundingClientRect.top` against the actual sticky `top` offset value (e.g. `73`), not `0`. Comparing against `0` can't distinguish "hasn't scrolled into view yet" from "has scrolled past the stick point" — both read as `isIntersecting: false`.

### 7. Regression check on the other brand
After any shared-component refactor (footer, nav, icons), screenshot the **other** brand's equivalent page too (e.g. an Adam locale after touching `NirmalFooter`-adjacent shared code) to confirm zero visual change there.

### 8. Verification gate before commit
1. Kill any stray dev server / `python3 -m http.server` processes started for comparison.
2. `tsc --noEmit` — clean (ignore known pre-existing unrelated failures, don't let them block).
3. `npm run lint` — clean.
4. `npm run build` — must succeed and show all expected static params generated (e.g. N posts × M locales) in the route output.
5. Commit with a descriptive message. **Never push without explicit user instruction** — this is a hard rule independent of this skill.

---

## Phase 1b: Figma Handoff Workflow (fallback — Josh's designs, pre-Figma-sunset projects)

When a Figma URL is provided:

1. Call `get_design_context` with fileKey + nodeId
2. Extract: spacing values, type styles, color tokens, auto-layout constraints
3. Map to InfiGroup token system (see Token System below) — NEVER hardcode raw values
4. Check all responsive variants (mobile / tablet / desktop frames exist in Josh's files)
5. Note fill/hug/fixed sizing intent per element

### Auto-Layout → Tailwind Mapping
| Figma | Tailwind |
|-------|----------|
| Vertical direction | `flex-col` |
| Horizontal direction | `flex-row` |
| Gap | `gap-{n}` |
| Fill container | `flex-1` / `w-full` |
| Hug contents | `w-fit` |
| Fixed size | explicit `w-{n} h-{n}` |
| Space between | `justify-between` |
| Wrap | `flex-wrap` |

### Reading Josh's Designs Accurately
- Spacing follows 4pt grid (4, 8, 12, 16, 24, 32, 48, 64, 96px) — convert px to rem (16px base)
- Don't copy line-height percentages blindly — inspect each text style
- Extract colors as semantic tokens, not raw hex
- Figma "auto" line-height has no CSS equivalent — derive from context
- Check mobile frame first (mobile-first implementation)

---

## Phase 2: InfiGroup Token System

All values from `globals.css` `:root`, mapped via Tailwind `extend`. Never use raw values.

```
Colors:    --primary-50 to --primary-950, --accent-*, --color-header/body/tag/nav
Type:      --text-h1-size/height through h6, --body-lg/md/sm, --caption, --tag
Spacing:   --padding-x-relaxed (40→56→80px), --padding-x-tight, --padding-y-hero/navbar/footer
Layout:    --grid-columns, --grid-gutter, --grid-margin (responsive per breakpoint)
Radius:    calc hierarchy from --radius base (2.5rem)
Fonts:     --font-heading (Oswald), --font-body (Poppins), --font-gilroy
```

Mobile-first: defaults in `:root`, overrides at `768px`, `1024px`, `1920px`.

### Three-Layer Token Architecture
```
Primitives:  --color-blue-500, --spacing-4
Semantic:    --color-primary, --color-surface, --spacing-component-gap
Component:   --button-bg, --card-padding
```
Components reference semantic. Semantic references primitives. Dark mode swaps semantic values.

---

## Phase 3: Component Implementation

### Code Rules
- One component per file, PascalCase, props interface above component
- Max 200 lines per component, 800 max per file
- `cn()` from `@/lib/utils` for class merging (clsx + tailwind-merge)
- NEVER inline styles unless Tailwind (including arbitrary `[]` syntax) cannot achieve the result
- NEVER `any`, NEVER manual class concatenation
- Server components by default — `'use client'` only when interactivity needed
- Shadcn UI as foundation — extend via wrapper pattern, never modify base files

### Section Layout Pattern (Standard)
```tsx
<section className="w-full py-16 px-6 lg:px-margin">
  <div className="max-w-[1248px] mx-auto flex flex-col gap-8 items-center">
    {/* Badge + Title */}
    {/* Content grid/flex */}
  </div>
</section>
```

### CVA Variant Pattern
```tsx
const buttonVariants = cva("base-classes", {
  variants: { variant: {...}, size: {...} },
  compoundVariants: [...],
  defaultVariants: {...}
});
```

---

## Phase 4: Aesthetic Execution

### Typography
- NEVER use reflex fonts: Inter, Roboto, Arial, DM Sans, Plus Jakarta Sans, Outfit, Space Grotesk, Space Mono, Fraunces, Lora, Playfair Display, Cormorant, IBM Plex Sans/Mono
- Pair a distinctive display font with a refined body font
- Unexpected characterful choices elevate everything
- Technical briefs do NOT need a serif "for warmth"
- "Modern" briefs do NOT need a geometric sans

### Color & Contrast
- Use OKLCH, not HSL
- NEVER pure `#000` or `#fff` — always tint neutrals (chroma 0.005–0.01 minimum)
- Gray text on colored background = dangerous — use darker shade of bg or transparency
- Dark mode: dark gray (12–18% lightness), depth from surface lightness not shadows
- Dominant colors with sharp accents beat timid evenly-distributed palettes

### Spatial Composition
- Unexpected layouts: asymmetry, overlap, diagonal flow, grid-breaking elements
- Generous negative space OR controlled density — never neither
- Cards are overused — only use when content is truly distinct and actionable
- NEVER nest cards inside cards
- 4pt base grid: 4, 8, 12, 16, 24, 32, 48, 64, 96px

### Motion & Animation
Decision framework:
- **CSS**: hover/focus states, simple fades, transform/opacity transitions
- **Intersection Observer**: scroll-triggered reveals (preferred for Fyn/ShipX — lightweight)
- **Framer Motion**: state-driven, exit animations, drag/gesture, scroll-linked parallax, staggered children
- **Canvas**: organic effects (noise, particles)
- **Tailwind keyframes**: simple named animations with stagger variants

Rules:
- Always include `prefers-reduced-motion` support
- One well-orchestrated page load > scattered micro-interactions
- NEVER bounce/elastic easing (trendy 2015, now amateurish)
- NEVER animate layout-driving properties (`width`, `height`, `top`, `left`, margins)

### Backgrounds & Depth
Create atmosphere instead of defaulting to solid colors:
- Gradient meshes, noise textures, geometric patterns
- Layered transparencies, dramatic shadows
- Decorative borders, custom cursors, grain overlays

---

## Absolute Design Bans (Impeccable Laws)

These are non-negotiable. Violation = rewrite.

| Ban | Why |
|-----|-----|
| Side-stripe accent borders (`border-left/right` >1px as colored accent) | Cliché SaaS pattern |
| Gradient text (`background-clip: text` + gradient) | Overused to death |
| Glassmorphism as default (decorative blurs) | 2021 trend, now generic |
| Hero-metric template (big number + small label + gradient accent) | Saturated |
| Identical card grids (same-sized icon+heading+text repeated endlessly) | Lazy |
| Modal as first thought | Better options almost always exist |
| Em dashes | Use commas, colons, semicolons, periods, or parens |
| Pure `#000` or `#fff` | Always tint |
| Bounce/elastic easing | Tacky |
| Animating CSS layout properties casually | Performance killer |
| Monospace as lazy "technical" shorthand | Reflex choice |
| Large rounded-corner icons above every heading | SaaS template tell |
| All-caps body copy | Unreadable at scale |
| Timid palettes / average layouts | Forgettable |
| Zero imagery on briefs implying imagery (food, fashion, travel, hotel) | Missing intent |
| Editorial-magazine aesthetic on non-magazine briefs | Wrong register |

---

## Phase 5: Audit Checklist (mandatory before marking done)

Run this after every implementation. If working from a Claude Design static export, follow Phase 1a Section 5 (serve over HTTP, screenshot both, pixel-sample, full-column scan). If Josh gave a Figma file, screenshot both and compare.

### Design Quality
- [ ] AI Slop Test passes (first-order + second-order)
- [ ] No bans from Absolute Design Bans list
- [ ] Typography: distinctive font choice, not from reflex-reject list
- [ ] Color: no pure black/white, OKLCH-aware tinting
- [ ] Spacing: 4pt grid used, token-based not hardcoded
- [ ] Motion: prefers-reduced-motion handled, no bounce easing

### Design Fidelity (when handoff exists — static export or Figma)
- [ ] Mobile viewport matches
- [ ] Tablet viewport matches (if applicable)
- [ ] Desktop viewport matches
- [ ] Spacing values correctly mapped to existing tokens (no new ones invented)
- [ ] Color tokens match — no raw hex values
- [ ] Typography styles correct (size, weight, line-height, tracking)
- [ ] Static export only: served over HTTP (not `file://`) before concluding anything is "missing"
- [ ] Static export only: full-column pixel scan run, zero unexplained white/gap bands
- [ ] Static export only: other brand's equivalent page screenshot-checked for zero regression

### Code Quality
- [ ] No inline styles (unless Tailwind physically cannot do it)
- [ ] No hardcoded raw values (use CSS vars / tokens)
- [ ] Server component by default
- [ ] `cn()` used for class merging
- [ ] Component under 200 lines

### Accessibility
- [ ] Focus states visible (NEVER `outline: none` without replacement)
- [ ] Color contrast WCAG AA minimum
- [ ] Touch targets ≥ 44px
- [ ] Heading order logical (h1 → h2 → h3)
- [ ] Images have alt text

---

## Design Commands (quick invocations)

| Command | Action |
|---------|--------|
| `audit` | Full checklist audit of current implementation |
| `polish` | Refine spacing, typography, micro-details |
| `critique` | Honest assessment of what's weak and why |
| `bolder` | Push aesthetic further — more contrast, scale, personality |
| `quieter` | Pull back — more restraint, refinement, breathing room |
| `animate` | Add motion layer respecting reduced-motion |
| `colorize` | Rethink color strategy from scratch |
| `typeset` | Typography audit and upgrade |
| `layout` | Rethink spatial composition |
| `distill` | Reduce to essence — remove everything non-essential |
| `harden` | Accessibility + focus + contrast pass |

---

## Responsive Design Approaches (InfiGroup)

1. **CSS Variable Breakpoints** (ShipX/Fyn) — tokens change at breakpoints
2. **Tailwind Breakpoint Classes** (Astro) — `grid-cols-1 md:grid-cols-2 lg:grid-cols-3`
3. **Viewport-Relative Units** (Portfolio) — `lg:text-[2vw] md:text-[4vw]`
4. **Container Queries** — `@container` + `@sm:`, `@md:`, `@lg:` in Tailwind v4
5. **Fluid Typography** — `clamp(1rem, 0.5rem + 1.5vw, 2rem)`
