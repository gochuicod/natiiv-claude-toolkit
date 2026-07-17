---
name: infigroup-i18n
description: InfiGroup i18n implementation skill. Covers next-intl v4 setup (Powershell pattern), multi-brand translation file structure, static data translation by product ID, persistent locale selection (localStorage + cookie dual-persistence), browser locale detection modal, and common pitfalls. Use for any localisation, translation, or locale-switching work.
argument-hint: "[setup | translate | detect | audit]"
---

# InfiGroup i18n — Translation & Localisation System

## Role

i18n engineer who:
- Implements next-intl v4 with the Powershop explicit-locale-threading pattern
- Structures message files per brand and locale
- Translates static data (products, content) by ID — never inline in component props
- Wires persistent locale selection with localStorage + cookie dual-persistence
- Adds browser-language auto-detection with a non-intrusive confirmation modal
- Prevents all known pitfalls before they happen

---

## Phase 0: Before Any i18n Work

Answer these first:

1. **Which brand(s)?** Adam (en/ja/ms) · Nirmal (id/hi) · or both?
2. **What needs translating?** UI strings only? Static product/content data too?
3. **Locale detection?** First-visit modal active? LocaleRestorer (hard-refresh fix) active?
4. **Persistent selection?** Always yes — localStorage + cookie dual-persistence is mandatory.

---

## Phase 1: File Structure

### Message file layout (base_system standard)
```
messages/
  adam/
    en.json   ← primary source of truth (write English first)
    ja.json
    ms.json
  nirmal/
    id.json
    hi.json
```

### Brand resolution in `src/i18n/request.ts`
```typescript
import { getRequestConfig } from "next-intl/server";
import { routing } from "./routing";
import { LOCALE_TO_BRAND } from "@/config/brands";

export default getRequestConfig(async ({ requestLocale }) => {
  let locale = await requestLocale;
  if (!locale || !routing.locales.includes(locale as never)) {
    locale = routing.defaultLocale;
  }
  const brand = LOCALE_TO_BRAND[locale] ?? "adam";
  return {
    locale,
    messages: (await import(`../../messages/${brand}/${locale}.json`)).default,
  };
});
```

### Locale codes (CRITICAL — common mistake)
| Language | Correct code | Wrong code |
|----------|-------------|------------|
| Japanese | `ja` | ~~`jp`~~ |
| Malay | `ms` | ~~`my`~~ |
| Indonesian | `id` | ~~`in`~~ |
| Hindi | `hi` | ~~`in`~~ |

---

## Phase 2: next-intl v4 Setup (Powershop Pattern)

The Powershop pattern fixes three failure modes: Turbopack middleware bypass, RSC async context loss through Promise.all, and locale not registered with Next.js router.

### `src/app/[locale]/layout.tsx` — mandatory entries

```typescript
import { setRequestLocale } from "next-intl/server";
import { getMessages } from "next-intl/server";
import { siteConfig } from "@/config/site";

// ① Register all valid [locale] segments with Next.js router
export function generateStaticParams() {
  return siteConfig.locales.map((locale) => ({ locale }));
}

// ② Call setRequestLocale in generateMetadata too
export async function generateMetadata({ params }) {
  const { locale } = await params;
  setRequestLocale(locale);           // ← required here too
  // ...
}

export default async function LocaleLayout({ children, params }) {
  const { locale } = await params;
  setRequestLocale(locale);           // ← write locale to React cache

  // ③ Always pass { locale } explicitly — never rely on async context
  const messages = await getMessages({ locale });
  // ...
}
```

### Nested pages (every `page.tsx` under `[locale]/`)

```typescript
export default async function SomePage({ params }) {
  const { locale } = await params;
  setRequestLocale(locale);           // ← required in every page
  // ...
}
```

### `generateStaticParams` on nested pages — use parent locales

```typescript
// src/app/[locale]/shop/[slug]/page.tsx
export function generateStaticParams() {
  return siteConfig.locales.flatMap((locale) =>
    products.map((p) => ({ locale, slug: p.slug }))
  );
}
```

**NEVER** hardcode locale arrays in `generateStaticParams` — always use `siteConfig.locales`.

---

## Phase 3: Translating Static Data (Product-by-ID Pattern)

When product/content data lives in TypeScript files (not a CMS), translate it by ID in message files. Do NOT duplicate into component props.

### Message file structure for products
```json
{
  "shop": {
    "products": {
      "supp-001": {
        "tagline": "Sharp Focus & Clear Thinking",
        "heroHeadline": "Your Mind, Sharpened.",
        "description": "...",
        "benefits": ["Benefit 1", "Benefit 2"],
        "ingredients": [
          { "name": "Lion's Mane", "benefit": "Cognitive enhancement" }
        ],
        "keyMechanisms": [
          { "name": "Neuroplasticity", "description": "..." }
        ],
        "faqs": [
          { "q": "Question?", "a": "Answer." }
        ]
      }
    }
  }
}
```

### Server component usage
```typescript
// Convert to async server component
import { getTranslations } from "next-intl/server";

export default async function ProductDetail({ product }: Props) {
  const t = await getTranslations("shop");

  // Strings: t(`products.${id}.field`)
  const tagline = t(`products.${product.id}.tagline`);

  // Arrays: MUST use t.raw() — t() only works for string leaves
  const benefits = t.raw(`products.${product.id}.benefits`) as string[];
  const faqs = t.raw(`products.${product.id}.faqs`) as Array<{ q: string; a: string }>;
}
```

**Rule:** `t()` for string leaves. `t.raw()` for arrays and nested objects. Never `t()` on an array key — it returns `[object Object]`.

---

## Phase 4: Persistent Locale Selection

Two components work together. Both are placed in `[locale]/layout.tsx` content block.

### LocaleRestorer — fixes hard refresh / Turbopack middleware bypass
```typescript
// src/components/layout/LocaleRestorer.tsx
"use client";

export default function LocaleRestorer() {
  const router = useRouter();           // from @/i18n/routing
  const pathname = usePathname();       // from @/i18n/routing
  const currentLocale = useLocale() as LocaleCode;

  useEffect(() => {
    try {
      const stored = localStorage.getItem("NEXT_LOCALE") as LocaleCode | null;
      const isValid = stored && siteConfig.locales.includes(stored);
      if (isValid && stored !== currentLocale) {
        document.cookie = `NEXT_LOCALE=${stored};path=/;max-age=31536000;SameSite=Lax`;
        router.replace(pathname, { locale: stored });
      }
    } catch {}
  }, []);

  return null;
}
```

### Locale persistence on user switch (language picker)
```typescript
function switchLocale(newLocale: LocaleCode) {
  try {
    localStorage.setItem("NEXT_LOCALE", newLocale);
    document.cookie = `NEXT_LOCALE=${newLocale};path=/;max-age=31536000;SameSite=Lax`;
  } catch {}
  router.replace(pathname, { locale: newLocale });
}
```

Always write **both** localStorage and cookie. Cookie is read by middleware (production). localStorage covers Turbopack dev (middleware bypass).

---

## Phase 5: Browser Locale Detection Modal

Show once on first visit. Skip if user already has a preference stored.

### Detection logic
```typescript
function detectBrowserLocale(supported: readonly string[]): LocaleCode | null {
  if (typeof navigator === "undefined") return null;
  const langs = navigator.languages?.length
    ? navigator.languages
    : [navigator.language];
  for (const lang of langs) {
    const prefix = lang.split("-")[0].toLowerCase();
    if ((supported as readonly string[]).includes(prefix)) {
      return prefix as LocaleCode;
    }
  }
  return null;
}
```

### Guard conditions (skip modal if any true)
```typescript
useEffect(() => {
  if (localStorage.getItem("NEXT_LOCALE")) return;      // already chosen
  if (localStorage.getItem("ARKA_LOCALE_PROMPTED")) return; // already dismissed
  const detected = detectBrowserLocale(siteConfig.locales);
  if (!detected) return;                                // no match
  if (detected === currentLocale) return;               // already on right locale
  if (detected === "en") return;                        // English is default, no need
  setSuggested(detected);
}, [currentLocale]);
```

### On accept
```typescript
localStorage.setItem("NEXT_LOCALE", locale);
localStorage.setItem("ARKA_LOCALE_PROMPTED", "1");
document.cookie = `NEXT_LOCALE=${locale};path=/;max-age=31536000;SameSite=Lax`;
router.replace(pathname, { locale });
```

### On dismiss
```typescript
localStorage.setItem("ARKA_LOCALE_PROMPTED", "1"); // never show again
```

### UX pattern
- Bottom sheet on mobile (`items-end`), centered on desktop (`sm:items-center`)
- Backdrop click = dismiss
- Show language in both native script AND English: "Switch to 日本語" + subtitle "We detected your browser prefers 日本語 (Japanese)"
- Two buttons: primary = switch, secondary = stay in current

### localStorage keys
| Key | Purpose |
|-----|---------|
| `NEXT_LOCALE` | User's chosen locale (read by LocaleRestorer + middleware) |
| `ARKA_LOCALE_PROMPTED` | Set after modal shown — prevents re-display |

---

## Phase 6: Routing Config

```typescript
// src/i18n/routing.ts
export const routing = defineRouting({
  locales: siteConfig.locales,         // always from siteConfig, never hardcoded
  defaultLocale: siteConfig.defaultLocale,
  localeCookie: { name: "NEXT_LOCALE", maxAge: 31536000, sameSite: "lax" },
});

export const { Link, redirect, usePathname, useRouter } = createNavigation(routing);
```

Always import `useRouter` / `usePathname` from `@/i18n/routing`, NOT from `next/navigation` — the next-intl wrappers carry locale context.

---

## Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Wrong locale code | `jp` instead of `ja` | Use BCP 47: `ja`, `ms`, `id`, `hi` |
| No `generateStaticParams` in locale layout | Locale reverts on navigation | Add it — registers segments with Next.js router |
| `getMessages()` without `{ locale }` | Messages use wrong locale under Promise.all | Always `getMessages({ locale })` |
| `setRequestLocale` missing in page | DYNAMIC_SERVER_USAGE in prod | Every `page.tsx` under `[locale]/` needs it |
| `t()` on array key | `[object Object]` rendered | Use `t.raw()` for arrays/objects, cast to correct type |
| Import router from `next/navigation` | Locale lost on navigate | Import from `@/i18n/routing` |
| Curly/smart quotes in JSON | Turbopack build crash | ASCII straight quotes only — common from Notion/WhatsApp copy-paste |
| `generateStaticParams` with hardcoded locales | New locale added to config but breaks build | Always `siteConfig.locales.map(...)` |

---

## Audit Checklist (run before marking done)

- [ ] `generateStaticParams` exists in `[locale]/layout.tsx`
- [ ] Every `page.tsx` calls `setRequestLocale(locale)`
- [ ] `getMessages({ locale })` — explicit locale arg
- [ ] `getTranslations({ locale, namespace })` — explicit locale in server components
- [ ] Arrays translated with `t.raw()` + type cast
- [ ] LocaleRestorer wired in layout
- [ ] LocaleDetectModal wired in layout (first-visit detection)
- [ ] Locale switch writes both localStorage + cookie
- [ ] `useRouter` / `usePathname` imported from `@/i18n/routing`
- [ ] No hardcoded locale arrays (always `siteConfig.locales`)
- [ ] No smart/curly quotes in JSON message files
- [ ] Locale codes are BCP 47 (`ja` not `jp`)

---

## i18n Commands (quick invocations)

| Command | Action |
|---------|--------|
| `setup` | Scaffold full i18n for a new project (routing, request config, layout wiring) |
| `translate` | Add translations for a namespace or static data type |
| `detect` | Add/review the browser locale detection modal |
| `audit` | Run the full checklist against current implementation |
