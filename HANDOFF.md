# Handoff — Guide India Tours AI-Readability Audit (paste this into the new chat)

> **Paste this whole file into the new Claude Code session that has the correct repo.** It carries everything understood in the prior session so nothing is lost. The companion file `AI-READABILITY-AUDIT.md` has the full findings tables and every ready-to-paste code block — share that too.

## What this is

I ran a full AI-readability audit of the **live** site `https://guideindiatours.com` (served on `www.guideindiatours.com`, Next.js App Router, SSG, hosted on Vercel). The previous session was pointed at an **empty repo** (`mohdbilal2000/guideindiatours` had no commits), so no fixes were applied — they were all captured as code blocks instead. **This session should be the correct repo.** Your job: verify these findings against the actual source, apply the fixes, commit, and push.

## Live-site facts (verified 2026-05-20, don't re-derive)

- Hosting: Vercel. Stack: Next.js App Router, SSG-prerendered (`x-nextjs-prerender: 1`), full HTML returned to bots — crawlability is good.
- Apex `guideindiatours.com` **307-redirects** (temporary) to `www.guideindiatours.com`.
- `robots.txt`, `sitemap.xml` (136 URLs), and `llms.txt` (text/plain, good content) all exist and return 200.
- Strong **global JSON-LD** block on every page: `TravelAgency`/`LocalBusiness`, `WebSite`+`SearchAction`, `AggregateRating` (4.9 / 366+), `ContactPoint`, address, geo, opening hours.
- `/reviews` has 18 `Review`+`Person`+`Rating` entries. Plan pages have `TouristTrip`+`Offer`+`BreadcrumbList`+`AggregateRating`. Blog has `BlogPosting`. `/faq` has `FAQPage` (5 Q&A). OG + Twitter cards present.
- Real data found: phone `+91 9410000991`, WhatsApp `+91 8979810991`, email `info@guideindiatours.com`. Founder: Bilal. Lead guide: Avneesh Dixit (government-approved). Ground ops: Danish.

## What's already good — do NOT redo

SSG/crawlability, `llms.txt`, sitemap coverage, global Organization/TravelAgency schema, reviews schema, blog schema, tour `TouristTrip` schema, OG/Twitter cards, image `alt` coverage.

## Confirmed gaps to fix (ranked) — full code blocks are in `AI-READABILITY-AUDIT.md`

**Critical (do first):**
1. **Multiple `<h1>` per page** — homepage has 3, every other page has 2. Make exactly one `<h1>` per page; the logo wordmark should be a `<span>`/link, not an `<h1>`.
2. **Security headers missing** — add via `next.config.ts`: `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `X-Frame-Options: SAMEORIGIN`, `Permissions-Policy`, CSP (start Report-Only); upgrade HSTS to `max-age=63072000; includeSubDomains; preload`.
3. **Host canonicalization mismatch** — pages serve on `www` but `<link rel=canonical>` + sitemap `<loc>`s use non-`www`, and apex→www is 307. Make apex→www a **308**, then point canonical + sitemap + robots `Sitemap:` all at one host (recommend `www`). Set `metadataBase` in `app/layout.tsx`.
4. **robots ↔ sitemap conflict** — `/digital-card` is `Disallow`'d in robots but listed in sitemap (returns 200). Pick one. Also add missing AI crawlers: `ClaudeBot` (current; `Claude-Web` is legacy), `OAI-SearchBot`, `Perplexity-User`, `Applebot`, `Applebot-Extended`, `CCBot`, `Diffbot`, `FacebookBot`, `Meta-ExternalAgent`. Remove deprecated `Host:` line.
5. **No analytics detected** in served HTML (no GA4 / GTM / Microsoft Clarity / Vercel Analytics / Meta Pixel). Install GA4 (`@next/third-parties/google`) + Microsoft Clarity; define events (form submit, WhatsApp click, scroll depth). Confirm GSC + Bing dashboards show data.
6. **Plan pages lack per-tour `Review` + `FAQPage` schema** — add both.

**30-day:**
7. `AboutPage` + team `Person` schema on `/about`; new `/about/avneesh-dixit` authority page (with `Person`/`ProfilePage` schema) linked from every tour.
8. `ContactPage` schema on `/contact`.
9. Expand homepage FAQ to 12 Q&A (+ schema); add 6–10 FAQs per tour.
10. Per-page `lastmod` in sitemap (currently all share one stale 2026-03-10 timestamp).
11. Add `en` + `en-IN` hreflang (currently only en-US/en-GB/en-AU/x-default).
12. IndexNow: key file + submit-on-publish + register key in Bing.
13. Multi-currency pricing (EUR/GBP/INR) + explicit included/excluded lists per tour.
14. New landing pages: `/taj-mahal-tour-guide`, `/delhi-to-agra-day-trip`, `/private-taj-mahal-sunrise-tour`, `/agra-photography-tour`, `/accessible-india-tours`. One comparison page (`/compare/booking-direct-vs-getyourguide`).
15. Blog author is `Organization` — switch to a named `Person` byline.
16. Homepage `<title>` is 78 chars — trim to 50–60, keyword-first.

**90-day (off-site):** TripAdvisor verified operator, GetYourGuide/Viator/Klook supplier listings, Wikidata entity, LinkedIn company page, YouTube channel, Reddit/Quora answers, "best Agra guides 2026" listicles.

## Instruction for the new session

1. Confirm you're in the real source repo (look for `app/`, `next.config.*`, `package.json`).
2. Work through the Critical list first, then 30-day. Apply each fix in code using the blocks in `AI-READABILITY-AUDIT.md`.
3. Fill in `[bracketed]` placeholders (street address, postal code, exact prices, cancellation windows, LinkedIn URL) with real data — don't invent.
4. After each change, re-verify (build, and where possible curl the deployed page) before claiming done.
5. Commit with clear messages and push.
