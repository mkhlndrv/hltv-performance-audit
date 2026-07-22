# Implementer notes — hltv.org

This is the technical layer of the audit, written for the developer who
picks up an approved finding and has to make it true. It answers one
question per finding: what do I change, and how will I know it worked. The
business case is [stakeholder-report.md](stakeholder-report.md); the
findings catalog with evidence and scores is [findings.md](findings.md).
All documents share finding IDs F-01–F-15, and the same ID carries the same
numbers everywhere.

Constraints up front. This is a black-box audit: names below are shipped
asset names (`EverythingNight.css`, `hltv.js`), selectors and hosts as
observed in production — mapping them to source happens on your side of the
repo wall. Audited pages: the homepage (`/`), plus one news article for the
rendering and caching checks. And one bench limitation is real: the site's
Cloudflare bot protection blocks automated local Lighthouse — plan your
verification loop around PSI and WebPageTest, which get through.

## The bench — reproduce what we saw

Reproduce a finding's number once before changing anything; after the
change, run the same bench, not a different one.

- **Field (CrUX)** — pagespeed.web.dev for `hltv.org`, "real users" panel,
  75th percentile, ~28-day window. Reference: mobile LCP 1.8 s / desktop
  0.9 s, INP 96 / 13 ms, CLS 0, TTFB 0.5 / 0.3 s. All passing — your job
  is to keep it that way while fixing the lab tail.
- **Mobile lab (PSI)** — pagespeed.web.dev lab section: emulated Moto G
  Power, Slow 4G, 4× CPU, run on Google's infrastructure (which is why it
  isn't bot-blocked). Reference: Performance 39, FCP 3.2 s, LCP 10.7 s,
  TBT 970 ms, SI 10.2 s. Desktop: 69, LCP 1.8 s, TBT 410 ms.
- **Local Lighthouse — not available.** Cloudflare serves the automated
  run a challenge page instead of the site. Don't burn time on it; PSI is
  the scored bench here.
- **Device lab (WebPageTest)** — iPhone 15, Chrome, 4G (9 Mbps, 170 ms
  RTT). Reference: Start Render 3.1 s, FCP 3.54 s, LCP 5.13 s, TBT
  0.71 s, Document Complete 9.7 s, 4 MB / 143 requests. This run finishes
  (no timeout), so Document Complete is usable on this site.
- **DevTools by hand** — the bot challenge hits automation, not humans, so
  hands-on inspection works. Use a **clean profile with extensions off**:
  our first Performance trace was dominated by browser extensions
  (Coupert 1,598 ms, Dark Reader 1,373 ms against HLTV's own ~419 ms) and
  had to be discounted; WebPageTest was the clean interactivity read.
  Panels used: Network (enable the Priority column; document Response
  Headers for the caching findings), Coverage (fresh load, no interaction
  — expect a few points of drift between runs), Performance, Layers.

**Scope flags**: **local** = a template/attribute/asset change you can ship
alone; **structural** = build pipeline, caching policy, or anything needing
a product/compliance decision — a different conversation with your lead.

## Work orders

### F-01 — Hero image loads late (LCP)

- **Mechanism**: the LCP element is `<img class="hero-image">` from
  `img-cdn.hltv.org`. It's in the initial HTML (discoverable, not
  lazy-loaded) but has no `fetchpriority`, and no origin is preconnected —
  so the fetch queues behind the render-blocking chain (F-09). The field
  LCP breakdown shows load delay (745 ms) as the dominant component: the
  cost is queueing, not decoding.
- **Change**: `fetchpriority="high"` on the hero `<img>` (only the hero);
  `<link rel="preconnect" href="https://img-cdn.hltv.org">` early in the
  head; optionally a `rel="preload" as="image"` for the hero with exactly
  the URL the template emits.
- **Verify**: Network Priority column shows High and the request starts in
  the first wave; PSI LCP load-delay component shrinks; lab LCP moves from
  10.7 s; field stays ≤ 1.8 s next CrUX cycle.
- **Scope**: local. Job 1 (hours to a day).

### F-02 — Top banner is a 624 KiB animated GIF

- **Mechanism**: the light-theme top banner ships as one animated GIF of
  624 KiB — an eighth of the mobile page weight; PSI estimates ~425 KiB
  saveable. GIF is the most expensive animation container there is.
- **Change**: replace with `<video autoplay muted loop playsinline>`
  (mp4/webm) or an animated AVIF; keep the rendered dimensions identical —
  CLS is 0 today and must stay 0. Poster image as fallback.
- **Verify**: the banner asset in Network drops to tens of KiB; Speed
  Index (10.2 s lab) improves; CLS still 0.
- **Scope**: local. Job 2 (about a day including encoding).

### F-03 — Phones download desktop-sized photos

- **Mechanism**: content photos carry no `srcset`, so a phone fetches the
  same file a desktop gets; images are 2.2 MB of the 5.2 MB mobile load.
  The image CDN already resizes per request — the capability exists, the
  markup never asks for it.
- **Change**: emit `srcset` with two-three `img-cdn` size presets plus a
  `sizes` attribute per slot, in the news-list/match/gallery templates.
- **Verify**: DevTools mobile emulation, Network: selected candidates near
  slot width; mobile image transfer drops from 2.2 MB. Visual spot-check
  per breakpoint.
- **Scope**: local. Job 3 (a few days across templates).

### F-04 — Ranking photos ~30× larger than their box

- **Mechanism**: ranking player photos are delivered at 400×417 for a
  ~70×73 slot — roughly 100 KiB of waste each, several per page. Same
  root as F-03: the CDN can resize, the template requests the big preset.
- **Change**: request the display-size preset (or fold into F-03's
  `srcset` floor).
- **Verify**: those requests drop to a few KiB each; PSI "improve image
  delivery" estimate (1,157 KiB) shrinks accordingly.
- **Scope**: local. Job 3 (with F-03 — same pass).

### F-05 — Third-party tag/consent/ad scripts load earlier than needed

- **Mechanism**: cadmus `script.ac` is render-blocking (2,660 ms in the
  PSI blocking list) and forces synchronous reflow (99 ms); the Cookiebot
  consent script loads up front (1,800 ms in the blocking list, ~106 KB
  with its SDK); Google Tag Manager (490 KiB / 186 ms) then injects GA4,
  Facebook `fbevents.js` (101 KB), a Twitter tag and liftdsp's
  `admtracker` (which forces another 171 ms reflow). The stack is small —
  the problem is *when* it runs, not *that* it runs.
- **Change**: load cadmus async (it's an ad script — confirm slots still
  fill), load the consent UI non-blocking, move the tag-loader behind
  first paint. The reflows are inside vendor code: you control their
  schedule, not their internals — deferring them moves the cost off the
  critical path.
- **Verify**: PSI render-blocking list no longer shows cadmus/Cookiebot;
  a clean-profile trace shows no forced-reflow warnings from them during
  load.
- **Scope**: loading policy is local; consent timing changes need a nod
  from whoever owns compliance. Job 5.

### F-06 — Theme stylesheet is 2.3 MB, ~99% unused

- **Mechanism**: one monolithic theme sheet — `EverythingNight.css` or
  `EverythingDay.css` (~2.3 MB raw, 311 KB brotli) — ships render-blocking
  on every page; Coverage on the homepage: ~99% unused (2.28 of 2.31 MB).
  Two parallel theme monoliths double the maintenance surface.
- **Change**: split along route/template lines so a page ships its slice
  plus a shared core; purge selectors no template uses (behind visual
  regression on at least homepage, news, match pages); consider CSS custom
  properties for theming so Day/Night stops being two full copies.
- **Verify**: Coverage unused% on `/` falls from ~99%; CSS transfer drops
  from 311 KB br; FCP (3.2 s lab) follows.
- **Scope**: structural — build pipeline. Job 5.

### F-07 — Font Awesome ships whole for a few icons

- **Mechanism**: `font-awesome.min.css` (4.7) plus its webfont load for a
  handful of glyphs; Coverage: 98.9% unused (30.5 of 31 KB CSS, font
  downloads in full). The sheet sits in the head, render-blocking.
- **Change**: inventory used glyphs (grep templates for `fa-` classes),
  then subset the font or replace with inline SVGs and drop the
  stylesheet.
- **Verify**: the FA requests disappear (or shrink to a subset); icons
  intact across templates.
- **Scope**: local. Job 2.

### F-08 — Same-for-everyone pages are never edge-cached

- **Mechanism**: document responses return `Cf-Cache-Status: DYNAMIC` on
  `/` *and* on a published news article — Cloudflare passes every HTML
  request to origin. HLTV's own origin proxy does cache
  (`X-Proxy-Cache: STALE`, `Server-Timing: cfOrigin;dur=0`), so origin
  answers fast, but every reader still pays the full round-trip instead
  of a nearby edge hit. Separately, hashed static assets carry only ~8 h
  TTL (PSI flags est. 1,502 KiB of short-lived cacheables).
- **Change**: a Cloudflare cache rule (or `s-maxage` +
  `stale-while-revalidate` from origin) for same-for-everyone routes —
  published news, finished matches — leaving the live homepage and forums
  DYNAMIC. Raise hashed-asset TTL to a year with `immutable` (names are
  content-hashed; safe). Get the per-route freshness decision in writing
  first: "what breaks if this page is five minutes stale?"
- **Verify**: second fetch of an article shows `Cf-Cache-Status: HIT`
  with `Age > 0`; asset responses show `max-age=31536000, immutable`;
  repeat-view transfer (1.1–1.3 MB today) drops.
- **Scope**: structural-lite — config plus a product decision. Job 4.

### F-09 — Blank screen before first render (FCP)

- **Mechanism**: first paint waits on the full blocking set: the theme
  sheet (F-06), `hltv.js` (563.5 KiB br — ~7,530 ms in the PSI blocking
  list), `font-awesome.min.css` (F-07), cadmus and Cookiebot (F-05). PSI
  estimates 10,180 ms of blocking savings; the device lab shows nothing on
  screen until ~3.1 s. No origins are preconnected.
- **Change**: this is the umbrella — the work is F-05 + F-06 + F-07 +
  F-10 + F-13 in that dependency order, plus `preconnect` for
  `img-cdn.hltv.org` and the tag origins.
- **Verify**: WPT Start Render (3.1 s) and PSI FCP (3.2 s) move together;
  the PSI render-blocking list empties top-down as each piece lands.
- **Scope**: shared workstream, tracked as one. Job 6.

### F-10 — No critical CSS; the theme sheet gates paint

- **Mechanism**: the head contains no above-the-fold CSS — a leftover
  `#asdasda {}` debug rule (delete it), a Cookiebot state style and the
  Font Awesome link — then the theme sheet loads render-blocking via a
  `themeCssLink` `<link>` managed by a theme-switch script. That script
  dependency is also why the page renders completely unstyled with
  JavaScript disabled.
- **Change**: inline the homepage's above-the-fold rules (generated from
  the rendered page, regenerated in the build); load the theme sheet
  non-blocking (preload + swap); make the theme `<link>` plain HTML with
  the script only switching `href` — styling stops depending on JS.
- **Verify**: Performance trace: FCP fires before the theme sheet finishes
  downloading; a JS-disabled reload shows a styled above-the-fold instead
  of bare HTML.
- **Scope**: structural with F-06 (build integration). Job 5.

### F-11 — `hltv.js` is ~3 MB and not split

- **Mechanism**: one main bundle, `hltv.js` (~3.0 MB raw, 577 KB br),
  render-blocking and dominant in the LCP request chain; Coverage: 42%
  unused on the homepage (1.26 MB), `hltv-base-js.js` 51% unused; PSI also
  flags ~43 KiB of legacy transpilation the served browsers don't need.
  Content-hashed for caching, but every page parses the whole site's code.
- **Change**: split by route; defer everything not needed before paint;
  ship a modern ES target. Do it after F-13's defer so the split lands on
  a non-blocking baseline.
- **Verify**: Coverage unused% drops; `hltv.js` leaves the PSI blocking
  list; per-route JS bytes diverge (news page ≪ live homepage); TBT
  (970 ms lab) falls.
- **Scope**: structural — build pipeline. Job 7.

### F-12 — Far slower on mobile than desktop (program acceptance)

- **Mechanism**: no separate cause. PSI 39 vs 69, LCP 10.7 vs 1.8 s, TBT
  970 vs 410 ms (2.4×) — while both profiles download almost the same
  bytes (5.2 vs 5.3 MB). The gap is the blocking chain (F-09) meeting a 4×
  slower CPU, which is why it closes from code changes, not
  infrastructure.
- **Change**: none of its own — this is the acceptance condition. After
  each fix above lands, re-run the *mobile* bench.
- **Verify**: PSI mobile against the reference table in
  [baseline.md](baseline.md); a WPT filmstrip as the visual regression
  check.
- **Scope**: program-level. Job 8 spans the constituent fixes.

### F-13 — Render-blocking JS delays interactivity

- **Mechanism**: `hltv.js` blocks parsing and then evaluates on the main
  thread; the clean device-lab read shows TBT 0.71 s and "took a long time
  to become interactive." In the hand-run trace HLTV's own share was
  ~419 ms plus GTM 241 ms and cadmus 191 ms (the rest was our browser
  extensions — hence the clean-profile rule in the bench).
- **Change**: `defer` `hltv.js` (audit for document.write/parse-time
  dependencies first), keep the tag stack post-paint (F-05); the deeper
  fix is F-11's split.
- **Verify**: WPT TBT from 0.71 s; manual check under throttle — the page
  scrolls and takes clicks while still loading.
- **Scope**: local for the defer; structural via F-11. Job 6.

### F-14 — Main-thread freezes while loading (TBT)

- **Mechanism**: lab TBT 970 ms; main-thread work ~2.8 s with script
  evaluation 1,439 ms — that's ~5.6 MB of first-party JS parsed per load.
  PSI flags ~458 KiB genuinely unused (≈230 KiB inside `hltv.js`) and
  43 KiB legacy; two vendor scripts force reflows during load (F-05).
- **Change**: same workstream as F-11/F-13, third lens: remove the unused
  code paths, modernize the target, and let the split shrink what parses
  at all. No first-party long-task rewrite is indicated beyond that — the
  parse volume *is* the problem.
- **Verify**: PSI TBT mobile (970 ms) and desktop (410 ms) both drop;
  long-task count in a clean trace during the first 5 s.
- **Scope**: structural with F-11. Job 7.

### F-15 — One rendering strategy for all page types

- **Mechanism**: the live homepage (ticker, updating scores), published
  news, finished matches and the forum all get identical treatment:
  per-request SSR document (F-08) plus the full 3 MB bundle plus the full
  theme sheet. Read-only pages carry live-page machinery.
- **Change**: after F-08 (route cache policy) and F-06/F-11 (split
  outputs), assign per-route payloads: content routes = edge-cached
  document + minimal JS; live routes = DYNAMIC + socket + full
  interactivity. This is configuration once the splits exist — not a
  rewrite, and explicitly not a framework migration.
- **Verify**: per-route diff: JS/CSS bytes shipped on a news article vs
  the live homepage; response headers per route (HIT vs DYNAMIC).
- **Scope**: structural program. Job 8.

## Domains reviewed with no work orders

Every domain the audit covered, where it landed, and — where there's no
corrective — why that's a considered conclusion, not a gap:

| Domain | Status |
|--------|--------|
| Rendering pipeline | Corrective: F-09, F-10, F-13, F-14 |
| Metrics & measurement | The bench above; conditions and the Cloudflare/Lighthouse limitation documented in [baseline.md](baseline.md) |
| Compression | **No finding — verified good.** brotli/gzip on text (~78% wire savings, 8.0 → 1.7 MB), one script on zstd; images correctly not text-compressed |
| HTTP protocol | **No finding — verified good.** h2 + h3 on the main origins |
| Caching | Mixed: hashed asset names are right, but 8 h TTLs and a never-HIT document → F-08 |
| Service workers / offline | **Considered; does not apply.** No service worker observed in any capture, and we don't recommend one: a live-scores product wants freshness, and the weight that hurts is first-party CSS/JS — a cache layer doesn't fix shipping too much (F-06/F-11 do). Offline reading would be a product feature, not a performance fix |
| Fonts | **Considered; no corrective beyond F-07.** No flash-of-invisible-text visible in the filmstrip; the FA webfont is the only font finding |
| Images | Corrective: F-01–F-04. Formats verified good (WebP via `img-cdn`); the news-list flag icons ship as individual 30×20 GIFs — cosmetic-scale, fold into F-03's template pass if touched |
| Bundles: JS / CSS / source maps | Corrective: F-06, F-07, F-11. Source maps: **verified correctly absent** in production |
| Mobile | F-12 (acceptance) plus the amplified tags on findings; no mobile-exclusive corrective found |
| Accessibility | **Open — honestly unfinished.** Scores were recorded (85 mobile / 80 desktop) but a per-check audit with failing-element lists was not run on this project. Treat as scheduled work, not as a clean bill |
| Animations | **Considered; no corrective.** The site's own animations aren't a jank source (Day 8 pass); the animated GIF is F-02, an asset problem |
| Compositing / layers | **No finding — verified good.** 9 layers / 103 MB, no slow-scroll regions, nothing force-promoted |
| Progressive enhancement | Mixed: content renders without JS (server-rendered — strength), but styling doesn't, because the theme sheet is attached by script — fixed inside F-10 |
| State management & data fetching | **Considered; no finding.** Live updates ride a websocket on top of server-rendered pages, connecting after load; it never appeared as a cost in any bench |
| Rendering strategy | **Verified good** as a base (per-request SSR, no hydration tax) — see [baseline.md](baseline.md). Residuals: F-08, F-15 |
| Consent UX | **Considered; no separate finding.** Cookiebot is a banner, not a full-screen wall; its cost is load timing, covered in F-05 |

## Effort basis

Job Size in the WSJF table is relative implementer effort including
testing: **1** an attribute-level change (F-01); **2** a contained asset or
subset swap (F-02, F-07); **3** a template image pass (F-03, F-04); **4**
config plus a written product decision (F-08); **5** a loading-policy or
build slice (F-05, F-06, F-10); **6** a render-path rework (F-09, F-13);
**7** bundle restructuring (F-11, F-14); **8** a per-route program (F-12,
F-15). If your estimate after reading a work order differs materially, say
so in writing — Job Size is an input to the ranking, and re-arguing it is
how a finding legitimately moves.
