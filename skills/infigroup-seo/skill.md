---
name: infigroup-seo
description: InfiGroup SEO + GEO (Generative Engine Optimization) skill. Covers schema.org JSON-LD for all page types, sitemap/robots best practices, metadata patterns, hreflang for multi-locale, Lighthouse 100 checklist, and GEO signals for AI search citation (ChatGPT, Perplexity, Google AI Overviews). Use for any SEO audit, new project SEO setup, or GEO optimization work.
argument-hint: "[setup | audit | jsonld | sitemap | geo]"
---

# InfiGroup SEO + GEO System

## Role

SEO/GEO engineer who:
- Implements complete schema.org coverage for all page types
- Builds multi-locale sitemaps with hreflang alternates
- Optimizes for both traditional search (Google/Bing) and AI search (ChatGPT, Perplexity, Google AI Overviews, Claude)
- Runs structured audits against a scored checklist
- Prevents re-doing the same work across projects by templating everything

---

## What is GEO (Generative Engine Optimization)?

Traditional SEO = rank in blue links. GEO = get cited by AI search tools.

AI tools (ChatGPT browse, Perplexity, Google AI Overviews) use:
1. **Structured data** (JSON-LD) — factual, machine-readable authority signals
2. **E-E-A-T signals** — Experience, Expertise, Authoritativeness, Trust
3. **Clear factual statements** — concise sentences AI can quote directly
4. **Organization identity** — `sameAs` linking to verified social profiles
5. **Canonical URLs** — AI tools respect canonical to avoid scraping duplicates

Implement both SEO and GEO simultaneously — the overlap is ~80%.

---

## Phase 0: Before Any SEO Work

Audit existing state first. Check these files exist and are correct:
```
src/app/sitemap.ts         ← all indexable pages included?
src/app/robots.ts          ← correct disallow + host + sitemap?
src/app/[locale]/page.tsx  ← Organization + WebSite JSON-LD?
src/components/common/JsonLd.tsx  ← utility component exists?
```

Run: `curl -s https://your-domain.com/sitemap.xml | head -50` to verify production sitemap.

---

## Phase 1: sitemap.ts — Required Routes

```typescript
// INCLUDE: all user-facing, indexable pages
const STATIC_ROUTES = [
  { path: "",           priority: 1.0, changeFrequency: "weekly" },    // home
  { path: "/shop",      priority: 0.9, changeFrequency: "weekly" },    // catalog
  { path: "/about",     priority: 0.7, changeFrequency: "monthly" },
  { path: "/faq",       priority: 0.7, changeFrequency: "monthly" },   // FAQPage schema win
  { path: "/reviews",   priority: 0.6, changeFrequency: "weekly" },    // fresh content
  { path: "/protocols", priority: 0.6, changeFrequency: "monthly" },
  { path: "/journal",   priority: 0.5, changeFrequency: "weekly" },    // blog
  { path: "/subscribe", priority: 0.4, changeFrequency: "monthly" },
  { path: "/terms",     priority: 0.3, changeFrequency: "yearly" },
  { path: "/privacy",   priority: 0.3, changeFrequency: "yearly" },
];

// EXCLUDE: not indexable
// /cart /checkout /order-confirmation /auth/* /unavailable /api/*
```

### Multi-locale sitemap with hreflang alternates
```typescript
const buildAlternates = (path: string) => ({
  languages: Object.fromEntries(
    siteConfig.locales.map((locale) => [
      locale,
      `${baseUrl}/${locale}${path}`,
    ]),
  ),
});
```

**Rule:** Product/content pages also need alternates — build them the same way.

---

## Phase 2: robots.ts

```typescript
export default function robots(): MetadataRoute.Robots {
  return {
    rules: {
      userAgent: "*",
      allow: "/",
      disallow: ["/api/", "/admin/", "/checkout", "/order-confirmation", "/cart"],
    },
    sitemap: `${siteConfig.url}/sitemap.xml`,
    host: siteConfig.url,  // ← often forgotten; Powershop pattern
  };
}
```

---

## Phase 3: Metadata (generateMetadata per page)

### layout.tsx — root metadata template
```typescript
return {
  title: { default: brand.name, template: `%s | ${brand.name}` },
  description: brand.seo.description,
  metadataBase: new URL(siteConfig.url),
  alternates: {
    canonical: canonicalUrl,
    languages: {
      ...languageAlternates,
      "x-default": `${siteConfig.url}/${siteConfig.defaultLocale}`,
    },
  },
  openGraph: { title, description, url, siteName, locale, type: "website", images: [{ url, width: 1200, height: 630, alt }] },
  twitter: { card: "summary_large_image", title, description, images: [ogImage] },
  robots: { index: true, follow: true },
};
```

### Per-page generateMetadata
- Always set `canonical` URL explicitly (prevents duplicate content)
- Always set `x-default` in language alternates
- Product pages: use `product.metaDescription` (50–160 chars, includes primary keyword)
- Use `title` template from root layout — just set the page title fragment

---

## Phase 4: JSON-LD Schema Coverage

### JsonLd utility component
```typescript
export function JsonLd({ data }: { data: Record<string, unknown> | Array<Record<string, unknown>> }) {
  return (
    <script
      type="application/ld+json"
      // eslint-disable-next-line react/no-danger -- JSON-LD requires raw JSON in script tag
      dangerouslySetInnerHTML={{ __html: JSON.stringify(data) }}
    />
  );
}
```

### Schema by page type

#### Homepage — Organization + WebSite
```typescript
const organizationSchema = {
  "@context": "https://schema.org",
  "@type": "Organization",
  name: brand.name,
  url: siteConfig.url,
  logo: { "@type": "ImageObject", url: `${siteConfig.url}/images/logo-icon.svg`, width: 512, height: 512 },
  description: brand.seo.description,
  foundingLocation: { "@type": "Place", name: "Singapore" },
  areaServed: ["SG", "JP", "MY", "ID", "IN"],
  contactPoint: { "@type": "ContactPoint", contactType: "customer support", url: `${siteConfig.url}/${locale}/#contact` },
  // Add sameAs when social accounts are live:
  // sameAs: ["https://www.instagram.com/arkabotanics", "https://www.facebook.com/arkabotanics"],
};

const websiteSchema = {
  "@context": "https://schema.org",
  "@type": "WebSite",
  name: brand.name,
  url: `${siteConfig.url}/${locale}`,
  inLanguage: locale,
  description: brand.seo.description,
  potentialAction: {
    "@type": "SearchAction",
    target: `${siteConfig.url}/${locale}/shop?q={search_term_string}`,
    "query-input": "required name=search_term_string",
  },
};
```

**GEO note:** `sameAs` linking to verified social profiles is critical for AI citation — add when social accounts go live.

#### Shop listing — CollectionPage + ItemList
```typescript
const collectionSchema = {
  "@context": "https://schema.org",
  "@type": "CollectionPage",
  name: "Shop All",
  url: shopUrl,
  mainEntity: {
    "@type": "ItemList",
    numberOfItems: products.length,
    itemListElement: products.map((p, i) => ({
      "@type": "ListItem",
      position: i + 1,
      url: `${siteConfig.url}/${locale}/shop/${p.slug}`,
      name: p.name,
    })),
  },
};
```

#### Product page — Product + BreadcrumbList
```typescript
const productSchema = {
  "@context": "https://schema.org",
  "@type": "Product",
  name: product.name,
  description: product.description,
  image: imageUrl,
  sku: product.id,
  brand: { "@type": "Brand", name: brand.name },
  offers: {
    "@type": "Offer",
    url: canonicalUrl,
    price: product.price.toFixed(2),
    priceCurrency: product.currency,
    availability: product.inStock ? "https://schema.org/InStock" : "https://schema.org/OutOfStock",
    seller: { "@type": "Organization", name: brand.name },
  },
  // Add when reviews are available:
  // aggregateRating: { "@type": "AggregateRating", ratingValue: "4.8", reviewCount: "120" }
};

const breadcrumbSchema = {
  "@context": "https://schema.org",
  "@type": "BreadcrumbList",
  itemListElement: [
    { "@type": "ListItem", position: 1, name: "Home", item: `${siteConfig.url}/${locale}` },
    { "@type": "ListItem", position: 2, name: "Shop", item: `${siteConfig.url}/${locale}/shop` },
    { "@type": "ListItem", position: 3, name: product.name, item: canonicalUrl },
  ],
};

<JsonLd data={[productSchema, breadcrumbSchema]} />
```

**High value:** Product + BreadcrumbList combo unlocks breadcrumb display in Google SERP.

#### FAQ page — FAQPage (HIGHEST SERP VALUE)
```typescript
// Convert page to async server component to fetch translations
const faqSchema = {
  "@context": "https://schema.org",
  "@type": "FAQPage",
  mainEntity: FAQ_IDS.map((id) => ({
    "@type": "Question",
    name: t(`${id}.question`),
    acceptedAnswer: { "@type": "Answer", text: t(`${id}.answer`) },
  })),
};
```
**Why:** FAQPage schema renders as expandable accordions directly in Google SERP, dramatically increasing CTR. Biggest quick win for question-based queries.

#### Blog/Journal post — Article
```typescript
const articleSchema = {
  "@context": "https://schema.org",
  "@type": "Article",
  headline: post.title,
  description: post.excerpt,
  datePublished: post.publishedAt,
  dateModified: post.updatedAt,
  author: { "@type": "Organization", name: brand.name, url: siteConfig.url },
  publisher: { "@type": "Organization", name: brand.name, logo: { "@type": "ImageObject", url: logoUrl } },
  image: post.coverImage,
  url: canonicalUrl,
};
```

#### Reviews page — AggregateRating (when data exists)
```typescript
const reviewsSchema = {
  "@context": "https://schema.org",
  "@type": "ItemList",
  name: "Customer Reviews — Arka Botanics",
  itemListElement: reviews.map((r, i) => ({
    "@type": "ListItem",
    position: i + 1,
    item: {
      "@type": "Review",
      reviewRating: { "@type": "Rating", ratingValue: r.rating, bestRating: 5 },
      author: { "@type": "Person", name: r.author },
      reviewBody: r.body,
      itemReviewed: { "@type": "Organization", name: brand.name },
    },
  })),
};
```

---

## Phase 5: GEO Signals Checklist

GEO = optimizing for AI citation in ChatGPT, Perplexity, Google AI Overviews.

| Signal | Implementation |
|--------|---------------|
| `sameAs` on Organization | Link to verified social profiles, Wikipedia, Crunchbase |
| `author` on articles | Organization or named Person with credentials |
| `citation` / `isBasedOn` | Link to scientific studies cited in product copy |
| `speakable` | Mark sections that are good for voice/AI extraction |
| Factual density | Concise, citable sentences — AI extracts direct quotes |
| `aggregateRating` | Real rating + count signals trustworthiness |
| `keywords` on Product | Helps AI categorize product type |
| HSTS + canonical | Prevents AI from citing wrong URL variant |

### `speakable` (for AI voice extraction)
```typescript
const websiteSchema = {
  // ...
  speakable: {
    "@type": "SpeakableSpecification",
    cssSelector: ["h1", "h2", ".product-tagline"],
  },
};
```

### `sameAs` — add when social accounts live
```typescript
sameAs: [
  "https://www.instagram.com/arkabotanics",
  "https://www.facebook.com/arkabotanics",
  "https://www.tiktok.com/@arkabotanics",
]
```

---

## Phase 6: Technical SEO Checklist

Run before marking SEO work done:

### Metadata
- [ ] `title` template set in root layout (`%s | Brand Name`)
- [ ] All pages have unique `description` (50–160 chars)
- [ ] `canonical` set on every page (prevents duplicate content penalties)
- [ ] `x-default` hreflang present in language alternates
- [ ] All locales present in hreflang alternates
- [ ] `og:image` is 1200×630px, no URL path issues (use absolute URL)
- [ ] `twitter:card` is `summary_large_image`
- [ ] `metadataBase` set in root layout

### Sitemap
- [ ] All indexable pages included (home, shop, about, faq, reviews, protocols, journal, subscribe, terms, privacy)
- [ ] Non-indexable pages excluded (cart, checkout, order-confirmation, auth/*, api/*)
- [ ] Product pages included with correct slugs
- [ ] All locale variants included per page
- [ ] hreflang alternates in each sitemap entry
- [ ] Priority values reflect content importance (home: 1.0, products: 0.8, content: 0.5–0.7)

### robots.txt
- [ ] `allow /` set
- [ ] `/api/` and `/admin/` disallowed
- [ ] `sitemap:` URL included
- [ ] `host:` URL included

### JSON-LD Coverage
- [ ] Homepage: `Organization` + `WebSite` (with `SearchAction`)
- [ ] Shop listing: `CollectionPage` + `ItemList`
- [ ] Product page: `Product` + `BreadcrumbList` (+ `aggregateRating` when reviews exist)
- [ ] FAQ page: `FAQPage` with `mainEntity` Q&A array
- [ ] Blog posts: `Article` with `author`, `publisher`, `datePublished`

### Performance (affects SEO ranking)
- [ ] Images use `<Image>` (next/image) — avif/webp auto-optimization
- [ ] Pages use `generateStaticParams` for SSG where possible
- [ ] No layout shift from font loading (`display: swap`)
- [ ] Core Web Vitals: LCP < 2.5s, CLS < 0.1, INP < 200ms

### GEO
- [ ] `sameAs` on Organization schema (add when social accounts live)
- [ ] `speakable` added to homepage WebSite schema
- [ ] Product pages have `keywords` field
- [ ] Reviews have `AggregateRating` schema
- [ ] Article pages have `author` with credentials

---

## SEO Commands (quick invocations)

| Command | Action |
|---------|--------|
| `setup` | Scaffold full SEO for new project (sitemap, robots, JsonLd utility, homepage schemas) |
| `audit` | Run full checklist against current implementation, score it |
| `jsonld` | Generate the correct JSON-LD for a specific page type |
| `sitemap` | Review and fix sitemap (missing pages, wrong priorities, hreflang) |
| `geo` | GEO-specific pass: sameAs, speakable, citation signals, AI citation optimization |

---

## New Project Setup (5-step quickstart)

1. **Copy** `src/components/common/JsonLd.tsx` from Arka/Powershop (utility is identical)
2. **Copy** `src/app/sitemap.ts` + `src/app/robots.ts` from Arka, update `STATIC_ROUTES` for new project's pages
3. **Homepage** (`page.tsx`): add `Organization` + `WebSite` schemas — fill in brand name, logo URL, area served
4. **Product/listing pages**: add `Product` + `BreadcrumbList` + `CollectionPage` + `ItemList`
5. **FAQ page** (if exists): convert to async server component, add `FAQPage` schema

Common mistakes:
- Forgetting `host:` in robots.ts
- Leaving `logo` as a bare string instead of `ImageObject` (Google requires ImageObject for rich results)
- Not converting FAQ page to async component (schema must be server-rendered, not client-side)
- Hardcoding locale in sitemap instead of using `siteConfig.locales.flatMap(...)`
