# Guide India Tours — AI Readability Audit Report

**Site audited:** https://guideindiatours.com (live, served from `www.guideindiatours.com`, hosted on Vercel, Next.js App Router, SSG-prerendered)
**Audit date:** 2026-05-20
**Method:** Live `curl` + raw-HTML inspection of homepage, a plan page (`/plans/1002`), `/faq`, `/about`, `/contact`, `/reviews`, and a blog post (`/blog/taj-mahal-visiting-guide`), plus `robots.txt`, `sitemap.xml`, `llms.txt`, and HTTP headers.

> **Important context:** The GitHub repo this audit branch lives in (`mohdbilal2000/guideindiatours`) is **empty** — it is not the deployment source for the live site. So nothing here was applied to the live site automatically. Every fix below is provided as a **ready-to-paste code block** to drop into the real Next.js source. Move this file to the real repo, or hand the code blocks to whoever owns the source.

---

## Summary

| Status | Count | Meaning |
|---|---|---|
| ✅ Passing | 22 | Present and correct |
| ⚠️ Flawed | 14 | Present but needs a fix |
| ❌ Missing | 11 | Not present |

**The good news:** The site is in far better shape than a typical small operator. It is **server-rendered (SSG)**, fully crawlable without JS, ships a **valid `llms.txt`**, a **136-URL sitemap**, a strong **global Organization/TravelAgency JSON-LD block on every page**, `TouristTrip`/`Offer` schema on tour pages, **18 `Review` entries** on `/reviews`, `BlogPosting` schema on articles, and OG/Twitter cards. Tarik bhai's crawl-layer work is solid — this audit **extends** it, it does not rewrite it.

**Highest-leverage gaps (fix these first):**
1. **Multiple `<h1>` per page** (homepage has 3, every other page has 2) — breaks document hierarchy.
2. **Security headers missing** (`X-Content-Type-Options`, `Referrer-Policy`, `X-Frame-Options`, `Permissions-Policy`, CSP; HSTS lacks `includeSubDomains; preload`).
3. **Host canonicalization mismatch** — pages serve on `www`, but `<link rel=canonical>` and the sitemap point to non-`www`, and the apex→www redirect is a **307 (temporary)** not 308/301.
4. **robots.txt ↔ sitemap conflict** — `/digital-card` is `Disallow`'d in robots.txt but listed in the sitemap (and returns 200).
5. **No per-tour `Review` or `FAQPage` schema** on plan pages.
6. **No analytics detected** (no GA4 / GTM / Microsoft Clarity / Vercel Analytics in served HTML).
7. **No `AboutPage`/`Person` schema** for the team, and **no founder/guide authority page** for Avneesh Dixit.

---

## Phase 1 — Crawl & Access

| # | Check | Status | Finding |
|---|---|---|---|
| 1.1 | robots.txt exists, 200 | ✅ | Present, returns 200. |
| 1.1 | Major bots allowed | ✅ | `User-Agent: *` → `Allow: /` covers Googlebot/Bingbot/DuckDuckBot/YandexBot (none blocked). |
| 1.1 | AI crawlers allowed | ⚠️ | Present: GPTBot, ChatGPT-User, Claude-Web, anthropic-ai, PerplexityBot, Google-Extended, cohere-ai, Amazonbot, Bytespider. **Missing:** `ClaudeBot` (current Anthropic crawler — `Claude-Web` is legacy), `OAI-SearchBot`, `Perplexity-User`, `Applebot`, `Applebot-Extended`, `CCBot`, `Diffbot`, `FacebookBot`, `Meta-ExternalAgent`. |
| 1.1 | Sitemap declared | ✅ | `Sitemap: https://guideindiatours.com/sitemap.xml`. |
| 1.1 | No accidental `Disallow: /` | ⚠️ | `Disallow: /digital-card` conflicts with the sitemap (which lists `/digital-card`, HTTP 200). Pick one. |
| 1.1 | `Host:` directive | ⚠️ | Non-standard/deprecated directive, and points to non-`www` while the site serves on `www`. Remove it. |
| 1.2 | sitemap exists, valid XML, 200 | ✅ | 136 URLs, valid, single sitemap (under 50K). |
| 1.2 | All public URLs present | ✅ | Home, /plans + 105 plan details, /blog + 8 posts, /about, /contact, /faq, /reviews, location pages all listed. |
| 1.2 | `lastmod` recent | ⚠️ | Present, but **every URL shares one identical timestamp** (`2026-03-10T15:27:19Z`, ~71 days old). Make `lastmod` per-page and current. |
| 1.2 | `changefreq` + `priority` | ✅ | Present and sensibly distributed (home 1.0, tours 0.8–0.9, policy 0.2–0.3). |
| 1.2 | No disallowed URLs in sitemap | ⚠️ | `/digital-card` is in sitemap but `Disallow`'d in robots. |
| 1.2 | Host consistency | ⚠️ | Sitemap uses non-`www` URLs; site serves `www`. |
| 1.3 | Verification meta in `<head>` | ⚠️ | **No** `google-site-verification` / `msvalidate.01` / `yandex-verification` meta found in served HTML. GSC/Bing are likely DNS- or file-verified (Tarik bhai's work) — that's valid — but adding meta tags is a cheap belt-and-suspenders. |
| 1.4 | IndexNow | ❌ | No IndexNow key file detected and no submission route. Implement for instant Bing/Yandex/ChatGPT-index updates. |
| 1.5 | HSTS | ⚠️ | `strict-transport-security: max-age=63072000` present but **missing** `includeSubDomains; preload`. |
| 1.5 | CSP | ❌ | No `Content-Security-Policy` header. |
| 1.5 | `X-Content-Type-Options` | ❌ | Missing (`nosniff`). |
| 1.5 | `Referrer-Policy` | ❌ | Missing. |
| 1.5 | `Permissions-Policy` | ❌ | Missing. |
| 1.5 | `X-Frame-Options` | ❌ | Missing (`SAMEORIGIN`). |
| 1.5 | HTTPS enforced + valid SSL | ✅ | HTTP→HTTPS; valid Vercel cert; HTTP/2. |
| 1.6 | Renderable without JS | ✅ | `x-nextjs-prerender: 1`, full 176 KB HTML returned; content present without JS. Excellent for AI crawlers. |
| 1.7 | Cloudflare/CDN bot rules | ✅ (N/A) | Vercel, not Cloudflare. No bot-blocking observed; AI crawlers get full HTML. |

### Fix 1.1 — Corrected `robots.txt`

Drop-in replacement (extends the current file; adds missing AI crawlers, removes the `/digital-card` conflict by `noindex`-ing it instead — see note — removes the deprecated `Host:` line, standardizes on `www`):

```
# robots.txt — guideindiatours.com
User-agent: *
Allow: /
Disallow: /api/

# --- Search engines (explicit, optional since * already allows) ---
User-agent: Googlebot
Allow: /
User-agent: Bingbot
Allow: /
User-agent: DuckDuckBot
Allow: /
User-agent: YandexBot
Allow: /

# --- AI / LLM crawlers ---
User-agent: GPTBot
Allow: /
User-agent: ChatGPT-User
Allow: /
User-agent: OAI-SearchBot
Allow: /
User-agent: ClaudeBot
Allow: /
User-agent: anthropic-ai
Allow: /
User-agent: Claude-Web
Allow: /
User-agent: PerplexityBot
Allow: /
User-agent: Perplexity-User
Allow: /
User-agent: Google-Extended
Allow: /
User-agent: Applebot
Allow: /
User-agent: Applebot-Extended
Allow: /
User-agent: CCBot
Allow: /
User-agent: cohere-ai
Allow: /
User-agent: Diffbot
Allow: /
User-agent: FacebookBot
Allow: /
User-agent: Meta-ExternalAgent
Allow: /
User-agent: Amazonbot
Allow: /

Sitemap: https://www.guideindiatours.com/sitemap.xml
```

> **Note on `/digital-card`:** Decide what it is. If it's a private vCard you don't want indexed, **remove it from the sitemap** (don't list disallowed URLs) and keep it out of robots or use a `noindex` meta on the page instead. If it's a normal public page, **drop the `Disallow: /digital-card` line**. Right now robots and sitemap disagree, which sends crawlers a mixed signal.

> **Host decision:** I've standardized everything in this report on `www` because that's where the apex currently redirects. If you'd rather use the bare apex, flip the redirect (below) and use apex everywhere instead. The one rule that matters: **robots `Sitemap:`, sitemap `<loc>`s, and every `<link rel=canonical>` must all use the same host.**

### Fix 1.4 — IndexNow (Next.js App Router)

1. Generate a key (32+ hex chars), e.g. `a1b2c3d4e5f6...`. Create the key file at `app/<key>.txt/route.ts`:

```ts
// app/a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4.txt/route.ts
export const dynamic = "force-static";
export function GET() {
  return new Response("a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4", {
    headers: { "content-type": "text/plain" },
  });
}
```

2. Ping IndexNow whenever you publish/update a URL (call this from your publish flow or a webhook):

```ts
// lib/indexnow.ts
const KEY = "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4";
const HOST = "www.guideindiatours.com";

export async function pingIndexNow(urls: string[]) {
  await fetch("https://api.indexnow.org/indexnow", {
    method: "POST",
    headers: { "Content-Type": "application/json; charset=utf-8" },
    body: JSON.stringify({
      host: HOST,
      key: KEY,
      keyLocation: `https://${HOST}/${KEY}.txt`,
      urlList: urls,
    }),
  });
}
```

Then register the key once in Bing Webmaster Tools → IndexNow.

### Fix 1.5 — Security headers (Next.js `next.config.ts`)

```ts
// next.config.ts
const securityHeaders = [
  { key: "Strict-Transport-Security", value: "max-age=63072000; includeSubDomains; preload" },
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
  { key: "X-Frame-Options", value: "SAMEORIGIN" },
  { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=(self), browsing-topics=()" },
  // Start CSP in Report-Only, confirm nothing breaks, then switch to Content-Security-Policy.
  {
    key: "Content-Security-Policy-Report-Only",
    value:
      "default-src 'self'; img-src 'self' data: https:; script-src 'self' 'unsafe-inline' https://www.googletagmanager.com https://www.clarity.ms; style-src 'self' 'unsafe-inline'; font-src 'self' data:; connect-src 'self' https://www.google-analytics.com https://api.indexnow.org; frame-ancestors 'self';",
  },
];

const nextConfig = {
  async headers() {
    return [{ source: "/:path*", headers: securityHeaders }];
  },
};
export default nextConfig;
```

> CSP is the one to roll out carefully — keep it `-Report-Only` until you confirm GA4/Clarity/maps/embeds aren't blocked, then rename the header to `Content-Security-Policy`.

### Fix — apex→www redirect should be permanent (308)

The apex currently 307-redirects (temporary) to `www`. Make it permanent so link equity consolidates. In Vercel: **Project → Domains → set `www` as primary** (Vercel issues a 308). Or in `next.config.ts`:

```ts
async redirects() {
  return [{
    source: "/:path*",
    has: [{ type: "host", value: "guideindiatours.com" }],
    destination: "https://www.guideindiatours.com/:path*",
    permanent: true, // 308
  }];
}
```

---

## Phase 2 — Machine-Readable Content

| # | Check | Status | Finding |
|---|---|---|---|
| 2.1 | `llms.txt` | ✅ | Present, `200`, `text/plain`, well-structured (business overview, contact, offerings, features). |
| 2.2 | Homepage: Organization/TravelAgency/LocalBusiness | ✅ | `TravelAgency` (a `LocalBusiness` subtype) with address, geo, opening hours, contact point. |
| 2.2 | Homepage: WebSite + SearchAction | ✅ | Present. |
| 2.2 | Homepage: AggregateRating | ✅ | Present (4.9 / 366+). |
| 2.2 | Homepage: BreadcrumbList | ⚠️ | Not on homepage (acceptable for home; present on deeper pages). |
| 2.2 | Homepage: FAQPage | ❌ | No `FAQPage` schema on homepage (only the dedicated `/faq` page carries it). |
| 2.2 | Plan page: TouristTrip + Offer | ✅ | Present, with itinerary cities, offers, image. |
| 2.2 | Plan page: BreadcrumbList | ✅ | Present. |
| 2.2 | Plan page: AggregateRating | ✅ | Present. |
| 2.2 | Plan page: per-tour Review | ❌ | No individual `Review` entries on plan pages. |
| 2.2 | Plan page: FAQPage | ❌ | No tour-specific FAQ schema on plan pages. |
| 2.2 | Plan page: ImageObject | ✅ | Present. |
| 2.2 | Blog: BlogPosting + Breadcrumb + ImageObject | ✅ | Present (`og:type=article` too). |
| 2.2 | Blog: named author `Person` | ⚠️ | Author is `Organization`, not a credentialed `Person`. Add a real byline. |
| 2.2 | About: AboutPage + Person (team) | ❌ | Neither present. Only the global TravelAgency block. |
| 2.2 | Contact: ContactPage | ❌ | `ContactPoint` present (global block) but no `ContactPage` type. |
| 2.2 | FAQ page: FAQPage | ⚠️ | Present but only **5** Q&A pairs (standard wants 12+ on home/FAQ). |
| 2.2 | Reviews page: Review/Rating/Person | ✅ | 18 `Review` + `Person` + `Rating` + `AggregateRating` + breadcrumb. Excellent. |
| 2.3 | Open Graph + Twitter cards | ✅ | 9 OG tags + 4 Twitter tags present. |
| 2.4 | hreflang | ⚠️ | Has `en-US`, `en-GB`, `en-AU`, `x-default`. **Missing** generic `en` and `en-IN`. |
| 2.5 | Exactly one `<h1>` | ⚠️ | **Homepage: 3 `<h1>`; every other page: 2 `<h1>`.** Should be exactly one per page. |
| 2.5 | Semantic landmarks | ✅ | Site uses semantic structure and breadcrumbs (schema-backed). |
| 2.6 | Image alt text | ✅ | All 7 homepage `<img>` have non-empty `alt`. |
| 2.6 | Responsive `srcset` | ⚠️ | No `srcset` on homepage images — confirm `next/image` is used for hero/below-fold images. |
| 2.7 | Canonical present | ✅ | Every page has `<link rel=canonical>`. |
| 2.7 | Host canonicalization | ⚠️ | Canonical points to **non-`www`** while pages serve on **`www`**; apex→www is 307. Mismatch. |
| 2.8 | Internal linking / breadcrumbs | ✅ | Breadcrumbs present and schema-marked; footer + nav link key pages. |

### Fix 2.2 — Single h1 / canonical / hreflang via Next.js metadata

Set the canonical host once and let every page inherit:

```ts
// app/layout.tsx
export const metadata = {
  metadataBase: new URL("https://www.guideindiatours.com"),
  alternates: {
    canonical: "/",
    languages: {
      "en": "/",
      "en-US": "/",
      "en-GB": "/",
      "en-IN": "/",
      "en-AU": "/",
      "x-default": "/",
    },
  },
};
```

> Then **change the sitemap and robots `Sitemap:` to `www`** so all three agree. Per-page `metadata` should set `alternates.canonical: "/plans/1002"` etc. (relative paths resolve against `metadataBase`).

**Single `<h1>` fix:** Pick the one true page heading (the hero `<h1>` like "Premium Golden Triangle Tours in India"). Demote the others:
- The logo wordmark "GuideIndia Tours" in the header → it's a link, not a heading: use `<span>`/`<div>` (or `aria-label` on the logo link), **not** `<h1>`.
- Any secondary "Guide India Tours – Premium Golden Triangle Specialist" banner → `<p>` or `<h2>`.
- On `/plans`, `/faq`, `/about`, `/contact`, `/blog`: ensure the page title is the only `<h1>` and the logo is not an `<h1>`.

### Fix 2.2 — Per-tour `Review` + `FAQPage` JSON-LD (plan pages)

Add to `app/plans/[id]/page.tsx`. Pull the reviews/FAQs that are specific to that tour (or reuse a curated subset):

```tsx
function TourSchema({ tour }: { tour: Tour }) {
  const json = {
    "@context": "https://schema.org",
    "@type": "TouristTrip",
    name: tour.title,
    url: `https://www.guideindiatours.com/plans/${tour.id}`,
    aggregateRating: {
      "@type": "AggregateRating",
      ratingValue: "4.9",
      reviewCount: String(tour.reviewCount ?? 366),
      bestRating: "5",
    },
    review: tour.reviews.map((r) => ({
      "@type": "Review",
      author: { "@type": "Person", name: r.name },
      reviewRating: { "@type": "Rating", ratingValue: r.stars, bestRating: "5" },
      reviewBody: r.text,
      datePublished: r.date, // ISO 8601
    })),
  };
  const faq = {
    "@context": "https://schema.org",
    "@type": "FAQPage",
    mainEntity: tour.faqs.map((f) => ({
      "@type": "Question",
      name: f.q,
      acceptedAnswer: { "@type": "Answer", text: f.a },
    })),
  };
  return (
    <>
      <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: JSON.stringify(json) }} />
      <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: JSON.stringify(faq) }} />
    </>
  );
}
```

### Fix 2.2 — `AboutPage` + team `Person` schema (`/about`)

```json
{
  "@context": "https://schema.org",
  "@type": "AboutPage",
  "url": "https://www.guideindiatours.com/about",
  "mainEntity": {
    "@type": "Organization",
    "name": "Guide India Tours",
    "url": "https://www.guideindiatours.com",
    "founder": { "@type": "Person", "name": "Bilal", "jobTitle": "Founder" },
    "employee": [
      {
        "@type": "Person",
        "name": "Avneesh Dixit",
        "jobTitle": "Lead Guide (Government-Approved)",
        "knowsLanguage": ["English", "Hindi", "German", "French", "Italian", "Spanish"],
        "worksFor": { "@type": "Organization", "name": "Guide India Tours" }
      },
      { "@type": "Person", "name": "Danish", "jobTitle": "Ground Operations" }
    ]
  }
}
```

### Fix 2.2 — `ContactPage` schema (`/contact`)

```json
{
  "@context": "https://schema.org",
  "@type": "ContactPage",
  "url": "https://www.guideindiatours.com/contact",
  "mainEntity": {
    "@type": "TravelAgency",
    "name": "Guide India Tours",
    "url": "https://www.guideindiatours.com",
    "contactPoint": [
      { "@type": "ContactPoint", "telephone": "+91-9410000991", "contactType": "customer service", "availableLanguage": ["English","Hindi"], "areaServed": "IN" },
      { "@type": "ContactPoint", "telephone": "+91-8979810991", "contactType": "reservations", "contactOption": "TollFree" }
    ],
    "email": "info@guideindiatours.com"
  }
}
```

---

## Phase 3 — Content & AEO

| # | Check | Status | Finding |
|---|---|---|---|
| 3.1 | `<title>` length | ⚠️ | Homepage title is **78 chars** (`Guide India Tours \| #1 Specialist for Golden Triangle Tours Delhi, Agra, Jaipur`); target 50–60. Front-load the keyword, trim. |
| 3.1 | Meta description | ✅ | Present across pages. |
| 3.2 | FAQ + schema | ⚠️ | `/faq` exists but only 5 Q&A; no homepage FAQ schema. |
| 3.2 | TL;DR / question-format H2s / comparison tables | ⚠️ | Not consistently used. Add a 40–80 word summary box + question-style H2s on key pages. |
| 3.3 | 12+ homepage FAQs | ❌ | Only 5 in schema. See block below for 12. |
| 3.3 | 6–10 tour FAQs each | ❌ | None on plan pages (covered by Fix 2.2). |
| 3.4 | Comparison page(s) | ❌ | None (`/compare/...`). High AI-citation value. |
| 3.5 | Location landing pages | ⚠️ | Has `/delhi-tours`, `/agra-tours`, `/jaipur-tours`, `/golden-triangle-tours`. **Missing** `/taj-mahal-tour-guide`, `/delhi-to-agra-day-trip`, `/private-taj-mahal-sunrise-tour`, `/agra-photography-tour`, `/accessible-india-tours`. |
| 3.6 | Pricing transparency | ⚠️ | Prices shown (USD in `llms.txt`). Add multi-currency (EUR/GBP/INR) and explicit included/excluded lists per tour. |

### Fix 3.3 — Homepage FAQ (12 Q&A) + `FAQPage` schema

Render these as visible `<section>` content **and** emit the matching `FAQPage` JSON-LD (questions verbatim from the standard; fill answers from real policy):

```json
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {"@type":"Question","name":"How do I book a tour with Guide India Tours?","acceptedAnswer":{"@type":"Answer","text":"[Book direct via the website form or WhatsApp +91 8979810991; confirm dates, pay deposit, receive itinerary.]"}},
    {"@type":"Question","name":"What does a Golden Triangle tour cost?","acceptedAnswer":{"@type":"Answer","text":"[From $X / €X for N days, private, all-inclusive. Link to pricing.]"}},
    {"@type":"Question","name":"How long is the typical Golden Triangle tour?","acceptedAnswer":{"@type":"Answer","text":"[3–7 days; 4 days is most popular.]"}},
    {"@type":"Question","name":"Do you provide pickup from Delhi airport?","acceptedAnswer":{"@type":"Answer","text":"[Yes — private AC car, 24/7.]"}},
    {"@type":"Question","name":"Are your guides government-approved?","acceptedAnswer":{"@type":"Answer","text":"Yes. Lead guide Avneesh Dixit is a government-approved, licensed guide."}},
    {"@type":"Question","name":"What languages do your guides speak?","acceptedAnswer":{"@type":"Answer","text":"English and Hindi, with German, French, Italian, and Spanish guides on request."}},
    {"@type":"Question","name":"Can you accommodate travelers with mobility needs?","acceptedAnswer":{"@type":"Answer","text":"[Yes — describe accessible vehicles/itineraries.]"}},
    {"@type":"Question","name":"What's included in a private Taj Mahal tour?","acceptedAnswer":{"@type":"Answer","text":"[Private AC car, licensed guide, entry tickets, etc.]"}},
    {"@type":"Question","name":"When is the best time to visit Agra?","acceptedAnswer":{"@type":"Answer","text":"[October–March; sunrise for the Taj.]"}},
    {"@type":"Question","name":"Do you accept payment in EUR / USD / GBP?","acceptedAnswer":{"@type":"Answer","text":"[List accepted currencies and methods.]"}},
    {"@type":"Question","name":"What's your cancellation policy?","acceptedAnswer":{"@type":"Answer","text":"[State refund windows — link /refund-policy.]"}},
    {"@type":"Question","name":"How is Guide India Tours different from booking through Viator or GetYourGuide?","acceptedAnswer":{"@type":"Answer","text":"[Book direct with the operator: no markup, direct WhatsApp contact with the guide, fully customizable.]"}}
  ]
}
```

---

## Phase 4 — Trust & Credibility

| # | Check | Status | Finding |
|---|---|---|---|
| 4.1 | Testimonials with schema | ✅ | `/reviews` has 18 `Review` entries with `Person` + `Rating`. Strong. |
| 4.1 | Reviews: country / date / source link / photo | ⚠️ | Verify each review shows country, tour date, and a link to the Google/TripAdvisor source for verifiability. |
| 4.2 | Third-party listings (TripAdvisor/GYG/Viator/Klook) | ⚠️ | Off-site — not verifiable from code. Pursue listings (each is an inbound citation surface). |
| 4.3 | Founder/guide authority page | ❌ | No `/about/avneesh-dixit` page; no `Person` schema tying reviews to the named guide. |
| 4.4 | Press / featured-in | ❌ | None found. |
| 4.5 | Awards / certifications display | ⚠️ | "Government-approved" is mentioned; surface the certification + years-in-business + guests-served counters above the fold and in the footer. |

### Fix 4.3 — Founder/guide authority page (`/about/avneesh-dixit`)

Create a dedicated page with bio, years of experience, languages, government certification number, and photos at the Taj with guests, wrapped in `Person` schema and linked from every tour page:

```json
{
  "@context": "https://schema.org",
  "@type": "ProfilePage",
  "mainEntity": {
    "@type": "Person",
    "name": "Avneesh Dixit",
    "jobTitle": "Government-Approved Tour Guide",
    "worksFor": { "@type": "TravelAgency", "name": "Guide India Tours", "url": "https://www.guideindiatours.com" },
    "knowsLanguage": ["English","Hindi","German","French","Italian","Spanish"],
    "image": "https://www.guideindiatours.com/team/avneesh-dixit.jpg",
    "url": "https://www.guideindiatours.com/about/avneesh-dixit",
    "sameAs": ["[LinkedIn URL]"]
  }
}
```

---

## Phase 5 — Performance & Core Web Vitals

Could not run Lighthouse from this environment. **Observed positives:** SSG prerender, Vercel CDN with cache HIT, HTTP/2, all images carry `alt`. **Watch items:**
- Homepage HTML is **176 KB** (heavy for an SSG document) — check for inlined data/JSON-LD bloat and large inline styles.
- **No `srcset`** on homepage images — confirm `next/image` is doing responsive sizing and AVIF/WebP; preload the LCP hero image.
- Run Lighthouse on home / a plan page / contact and target Performance 95+, CLS <0.05, INP <200ms.

---

## Phase 6 — Off-Site Citation Footprint (recommendations)

Not verifiable from the codebase. Priorities for AI citation authority: **TripAdvisor verified operator** (highest travel-AI cite rate), **Wikidata entity**, **GetYourGuide/Viator/Klook supplier listings**, **YouTube channel** (transcripts get indexed), **Reddit/Quora helpful answers** under the founder/guide name, and inclusion in 1–2 "best Agra guides 2026" listicles. Each creates an authoritative inbound citation surface.

---

## Phase 7 — Analytics, Tracking & Measurement

| Tool | Status | Finding |
|---|---|---|
| GA4 | ❌ | No `gtag`/`G-` in served HTML. |
| Google Tag Manager | ❌ | No `googletagmanager`/`gtm.js`. |
| Microsoft Clarity | ❌ | No `clarity.ms`. |
| Vercel Analytics | ❌ | No `/_vercel/insights`. |
| Meta Pixel | ❌ | No `fbq(`. |
| GSC / Bing Webmaster | ⚠️ | Likely verified via DNS/file (no meta tag) — confirm both dashboards show data. |

**No analytics of any kind was detected in the served HTML.** Either it's gated behind a consent script that doesn't render server-side, or it's genuinely missing. At minimum add GA4 (or Vercel Analytics) + Microsoft Clarity (free heatmaps/session replay), and define conversion events: form submits, WhatsApp clicks, scroll depth. Use `@next/third-parties/google` for a clean GA4 install:

```tsx
// app/layout.tsx
import { GoogleAnalytics } from "@next/third-parties/google";
// ...inside <body>, after children:
<GoogleAnalytics gaId="G-XXXXXXXXXX" />
```

---

## Critical Path — next 7 days (ranked by leverage)

1. **Fix multiple `<h1>`** on every page (one `<h1>` each; logo → `<span>`). — *Phase 2.5*
2. **Add security headers** via `next.config.ts` (HSTS preload + nosniff + Referrer-Policy + X-Frame-Options + Permissions-Policy; CSP in Report-Only). — *Phase 1.5*
3. **Resolve host canonicalization**: make apex→www a 308, point canonical + sitemap + robots all at `www`. — *Phase 1.5 / 2.7*
4. **Fix robots ↔ sitemap `/digital-card` conflict** and add missing AI crawlers (`ClaudeBot`, `OAI-SearchBot`, `Perplexity-User`, `Applebot`, `CCBot`, etc.). — *Phase 1.1*
5. **Install analytics** (GA4 + Microsoft Clarity) and confirm GSC/Bing show data. — *Phase 7*
6. **Add per-tour `Review` + `FAQPage` schema** to plan pages. — *Phase 2.2*

## 30-day plan

7. `AboutPage` + team `Person` schema, and a `/about/avneesh-dixit` authority page linked from every tour. — *Phase 2.2 / 4.3*
8. `ContactPage` schema; expand homepage FAQ to 12 + per-tour FAQs (6–10 each). — *Phase 2.2 / 3.3*
9. Per-page `lastmod` in sitemap; add `en` + `en-IN` hreflang. — *Phase 1.2 / 2.4*
10. IndexNow key file + submit-on-publish; register key in Bing. — *Phase 1.4*
11. Multi-currency pricing (EUR/GBP/INR) + explicit included/excluded lists per tour. — *Phase 3.6*
12. New landing pages: `/taj-mahal-tour-guide`, `/delhi-to-agra-day-trip`, `/private-taj-mahal-sunrise-tour`, `/agra-photography-tour`, `/accessible-india-tours`. — *Phase 3.5*
13. One comparison page (`/compare/booking-direct-vs-getyourguide`). — *Phase 3.4*
14. Lighthouse pass: trim homepage HTML weight, confirm `next/image` responsive + LCP preload. — *Phase 5*

## 90-day plan (off-site citation footprint)

15. TripAdvisor verified operator + GetYourGuide/Viator/Klook supplier listings.
16. Wikidata entity; LinkedIn company page; YouTube channel (3+ videos).
17. Reddit/Quora helpful answers under founder/guide identity; secure inclusion in "best Agra guides 2026" listicles.

---

*Audit performed against the live site on 2026-05-20. Re-run quarterly. All code blocks are framework-matched (Next.js App Router) and ready to paste into the real source repository.*
