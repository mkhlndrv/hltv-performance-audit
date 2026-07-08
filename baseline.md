# Baseline — HLTV homepage

Primary page: https://www.hltv.org/. Core Web Vitals and PageSpeed Insights
captured 3 July 2026 (Lighthouse 13.4.0); networking captured 5 July 2026 from
the DevTools Network panel. Screenshots in `evidence/`.

## Core Web Vitals

Field data (CrUX, real users, 75th percentile). Assessment: **Passed** on both —
the site is fine for the median user; the lab numbers below are the low-end tail.

### Desktop

- **Largest Contentful Paint (LCP)**: 0.9 s
- **Interaction to Next Paint (INP)**: 13 ms
- **Cumulative Layout Shift (CLS)**: 0.01
- **First Contentful Paint (FCP)**: 0.7 s
- **Time to First Byte (TTFB)**: 0.3 s

### Mobile

- **Largest Contentful Paint (LCP)**: 1.8 s
- **Interaction to Next Paint (INP)**: 96 ms
- **Cumulative Layout Shift (CLS)**: 0
- **First Contentful Paint (FCP)**: 1.6 s
- **Time to First Byte (TTFB)**: 0.5 s

## PageSpeed Insights at `/`

Lab data (Lighthouse). Mobile is throttled to the class setting — **Slow 4G with
a 4× CPU slowdown, applied (not simulated)** — on an emulated mid-range phone
(Moto G Power); desktop uses the emulated desktop profile with no throttling.
Mobile is the primary signal (mobile-first indexing), so the mobile column is the
one the findings track.

### Desktop

- **Performance**: 69
- **Accessibility**: 80
- **Best Practices**: 73
- **SEO**: 92
- **Agentic Browsing**: 1/2

Metrics: FCP 0.8 s · LCP 1.8 s · TBT 410 ms · CLS 0.002 · Speed Index 2.4 s

### Mobile

- **Performance**: 39
- **Accessibility**: 85
- **Best Practices**: 69
- **SEO**: 92
- **Agentic Browsing**: 1/2

Metrics: FCP 3.2 s · LCP 10.7 s · TBT 970 ms · CLS 0 · Speed Index 10.2 s

Worst on mobile: LCP 10.7 s, Speed Index 10.2 s, TBT 970 ms, FCP 3.2 s. The
lab/field gap is large — real users pass, but a low-end phone on a cold cache
does not.

## Network Activity

DevTools Network panel — fresh load, then a soft refresh. Transfer size first,
uncompressed resource size in parentheses. These are byte and request counts, so
they don't change with throttling; the mobile *timing* under the class throttle is
the PSI mobile column above.

- **protocol**: http/2 and http/3
- **caching**
  - first-party static assets: content-hashed, long TTL (served from cache on refresh)
  - the page and ad/tracker calls: re-fetched each load
- **compression**
  - text (HTML/JS/CSS): brotli/gzip (one script even uses zstd)
  - images: none — already served as WebP by the image CDN

### Desktop

- **requests**: 208
- **fresh**: 5.3 MB (11.8 MB)
  - **js/css**: 1.7 MB (8.0 MB)
  - **images**: 2.4 MB (2.4 MB)
- **refresh**: 1.3 MB (11.7 MB)

### Mobile

- **requests**: 174
- **fresh**: 5.2 MB (11.7 MB)
  - **js/css**: 1.8 MB (8.0 MB)
  - **images**: 2.2 MB (2.2 MB)
- **refresh**: 1.1 MB (11.6 MB)

Mobile and desktop are close here — the site sends roughly the same bytes to
both, so the mobile problem is the throttled CPU/network in the lab, not a
heavier payload.

## Diagnostics (mobile, PSI Insights)

Where the time and bytes go — the causes `findings.md` builds on. Screenshots in
`evidence/02-baseline-findings/`.

- Render-blocking requests — est. 10,180 ms (`hltv.js` 563.5 KiB / ~7,530 ms;
  `EverythingDay.css` 303.5 KiB transferred; cadmus `script.ac` 2,660 ms;
  Cookiebot 1,800 ms). No origins preconnected.
- LCP element — `<img class="hero-image">` from `img-cdn.hltv.org`; not
  lazy-loaded and discoverable, but no `fetchpriority="high"`. Field LCP
  breakdown: load-delay 745 ms dominant.
- Main-thread work — est. 2.8 s (script evaluation 1,439 ms). Unused JavaScript
  est. 458 KiB (~230 KiB in `hltv.js`). Legacy JavaScript est. 43 KiB.
- Third parties — Google Tag Manager 490 KiB (186 ms), Cookiebot 316 KiB
  (284 ms), Facebook 164 KiB (135 ms), cadmus 163 KiB (265 ms). Forced reflows
  from cadmus (99 ms) and liftdsp's `admtracker` (171 ms).
- Improve image delivery — est. 1,157 KiB. A 624.8 KiB animated GIF banner;
  ranking photos served at 400×417 for a 70×73 display.
- Efficient cache lifetimes — est. 1,502 KiB; first-party static assets cache for
  only 8h.

Screenshots for these are in `evidence/02-baseline-findings/`:
`psi-mobile-render-blocking.png`, `psi-mobile-main-thread-js.png`,
`psi-mobile-legacy-js.png`, `psi-mobile-lcp-element.png`,
`psi-mobile-image-delivery.png`, `psi-mobile-oversized-images.png`,
`psi-mobile-third-parties.png`.

## Bundle analysis

How the build outputs are shipped (Day 7 method). HLTV blocks automated tooling
(Cloudflare), so this is from DevTools run by hand on 8 July 2026 — the Network
panel and the Coverage tab (screenshots in `evidence/05-bundles/`); third-party
costs are from the PSI mobile report.

Coverage on a fresh load shows the page executes only **33% of the 23 MB** of JS
and CSS it pulls (**15.5 MB unused**); across just the first-party bundles it's
**36% of 5.7 MB used — 3.7 MB unused**.

### JavaScript

- **Bundling** — one large first-party bundle, `hltv.js` (**~3.0 MB uncompressed**,
  577 KB brotli), plus a small `hltv-base-js.js` (~96 KB) and a few tiny hashed
  chunks. Content-hashed for caching, but effectively a single main bundle — no
  route- or component-level splitting.
- **Unused JS** — Coverage: **42% of `hltv.js` unused** on the homepage (1.26 MB of
  3.0 MB) and **51% of `hltv-base-js.js`** unused. `hltv.js` is also render-blocking
  and dominates the LCP path (~7.5 s in PSI).
- **Source maps** — none surfaced in the capture; the minified bundles ship without
  public `.map` files, which is the correct production choice.

### CSS

- **Bundling** — one monolithic theme stylesheet, `EverythingNight.css` (or
  `EverythingDay.css` in light mode) — **~2.3 MB uncompressed** (311 KB brotli),
  loaded render-blocking — plus Font Awesome 4.7's `font-awesome.min.css`. No
  per-route/component splitting: the whole theme ships on every page.
- **Unused CSS** — Coverage: **~99% of the theme sheet unused** on the homepage
  (2.28 MB of 2.31 MB) and **98.9% of Font Awesome** unused (the entire icon library
  loads for a handful of glyphs).

### Images

- **Sizes/formats** — content photos go through the `img-cdn.hltv.org` proxy as
  **WebP**, resized per request (modern format, serve-time sizing). But UI assets
  like the news-list flags are shipped as individual **30×20 GIFs**
  (`/img/static/flags/…`) — a dated format, one request each, no sprite or SVG.
- **Responsive** — the content photos carry no `srcset`, so a phone gets the same
  file as desktop (the existing "mobile downloads desktop-sized images" finding),
  and the ranking photos arrive at 400×417 for a ~70×73 slot.
- **Full-resolution** — the `img-cdn` proxy serves fixed preset sizes rather than
  passing the raw source URL, so the full-res original isn't directly exposed
  (unlike AP's open proxy).

### Third-party resources

- **Inventory** — consent via **Cookiebot** (`uc.js` + `consent-sdk-2.3.js`,
  ~106 KB), **Google Analytics 4** (`gtag`, 181 KB), **Facebook Pixel**
  (`fbevents.js`, 101 KB), a **Twitter/X tag** (`uwt.js`), liftdsp's `admtracker`
  and a cadmus ad script, plus (PSI) Google Tag Manager, Outbrain, Yahoo and
  DoubleClick.
- **Loading** — mostly `async`; the Pixel, GA, Twitter tag and admtracker are all
  injected by a single tag-loader after the first scripts, and the consent SDK
  loads up front.
- **Impact** (PSI mobile) — Google Tag Manager 490 KiB (186 ms), the consent script
  316 KiB (284 ms), Facebook 164 KiB (135 ms) and cadmus 163 KiB (265 ms) are the
  heaviest; cadmus and `admtracker` also force synchronous reflows (99 ms and
  171 ms).

## Note on method

Scores are from PageSpeed Insights (Lighthouse on Google's infrastructure — a
clean run without local-machine noise), and PSI applies the same Slow 4G + 4× CPU
mobile throttle the class specifies, so the mobile column is the throttled-mobile
measurement. Mobile is weighted as primary (mobile-first indexing); since
Lighthouse can't measure INP, TBT is the lab proxy. The class's "applied, not
simulated" local Lighthouse run (Step 2) isn't possible on this site: Cloudflare
serves the automated Lighthouse run a bot challenge instead of the page — the same
reason an earlier local trace was discarded — while PSI gets through because it
runs from Google's own infrastructure. The field LCP breakdown is real-user CrUX
data pulled by URL. Networking numbers are from the DevTools Network panel.
