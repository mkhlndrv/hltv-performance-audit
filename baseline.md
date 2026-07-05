# Baseline — HLTV homepage

Primary page: https://www.hltv.org/
Captured 3 July 2026 with PageSpeed Insights (Lighthouse 13.4.0). Mobile is an
emulated Moto G Power on Slow 4G with 4x CPU throttling; desktop is the emulated
desktop profile. Screenshots in `evidence/01-kickoff/`.

## Lighthouse category scores

| Category | Mobile | Desktop |
|----------|:------:|:-------:|
| Performance | 39 | 69 |
| Accessibility | 85 | 80 |
| Best Practices | 69 | 73 |
| SEO | 92 | 92 |

## Lab metrics (the single load PSI measured)

| Metric | Mobile | Desktop | Good |
|--------|:------:|:-------:|:----:|
| First Contentful Paint | 3.2 s | 0.8 s | < 1.8 s |
| Largest Contentful Paint | 10.7 s | 1.8 s | < 2.0 s |
| Total Blocking Time | 970 ms | 410 ms | < 200 ms |
| Speed Index | 10.2 s | 2.4 s | < 3.4 s |
| Cumulative Layout Shift | 0 | 0.002 | < 0.1 |

Worst on mobile, in order: LCP 10.7 s, Speed Index 10.2 s, TBT 970 ms,
FCP 3.2 s — all in the red band. CLS is fine on both.

## Field data (CrUX, real users, 75th percentile)

Core Web Vitals assessment: Passed on both mobile and desktop. (Good LCP is
< 2.0 s since the March 2026 tightening; mobile's 1.8 s clears it.)

| Metric | Mobile | Desktop |
|--------|:------:|:-------:|
| LCP | 1.8 s | 0.9 s |
| INP | 96 ms | 13 ms |
| CLS | 0 | 0.01 |

Real-user mobile LCP of 1,799 ms breaks down as: TTFB 456 ms, load delay
745 ms, load duration 136 ms, render delay 458 ms. The largest slice is load
delay, which means the LCP resource is discovered and fetched late in the load.

## Reading the baseline

Two things stand out.

Lab and field disagree sharply. At the 75th percentile real users pass Core Web
Vitals (LCP 1.8 s). The lab profile — a low-end phone on Slow 4G with a cold
cache — is where the page falls apart (LCP 10.7 s, 3.2 s to first paint). So the
problem isn't the median visitor; it's the long tail on slow devices and
connections, and how little headroom the page leaves them.

Every red metric is about rendering and main-thread work, not layout. CLS is 0,
while FCP, LCP, Speed Index and TBT all move together. That pattern points at
render-blocking resources, heavy JavaScript, and the ad/media load rather than
one bad asset.

## Top PageSpeed insights (mobile)

The mobile report's Insights and Diagnostics point at where the time and bytes
go. These are the numbers `findings.md` builds on:

- Render-blocking requests — est. 10,180 ms. Driven by `hltv.js` 563.5 KiB
  (~7,530 ms), `EverythingDay.css` 303.5 KiB, cadmus `script.ac` (2,660 ms), and
  the Cookiebot consent script (1,800 ms). No origins preconnected.
- LCP element — `<img class="hero-image">` from `img-cdn.hltv.org`. Not
  lazy-loaded and discoverable in the HTML, but `fetchpriority="high"` is not
  applied.
- Improve image delivery — est. 1,157 KiB. A 624.8 KiB animated GIF banner, and
  ranking photos served at 400×417 but displayed at 70×73.
- Third parties — Google Tag Manager 490 KiB (186 ms), Cookiebot 316 KiB
  (284 ms), Facebook 164 KiB (135 ms), cadmus 163 KiB (265 ms).
- Main-thread work — est. 2.8 s, mostly script evaluation (1,439 ms). Unused
  JavaScript est. 458 KiB, ~230 KiB of it in `hltv.js`.
- Legacy JavaScript — est. 43 KiB of unnecessary polyfills/transforms.
- Efficient cache lifetimes — est. 1,502 KiB; first-party static assets cache
  for only 8h.

Screenshots for these are in `evidence/02-baseline-findings/`: `psi-mobile-render-blocking.png`,
`psi-mobile-main-thread-js.png`, `psi-mobile-legacy-js.png`,
`psi-mobile-lcp-element.png`, `psi-mobile-image-delivery.png`,
`psi-mobile-oversized-images.png`, `psi-mobile-third-parties.png`.

## Note on method

The scores and metrics above are from PageSpeed Insights (score screenshots in
`evidence/01-kickoff/`); the causes come from the same report's Insights and
Diagnostics panels for the mobile run. I also tried a local
Lighthouse trace, but Cloudflare served the automated browser a bot challenge
instead of the real page (7 requests, 131 KB — the interstitial, not HLTV), so
that trace was discarded. The one exception is the CrUX field LCP breakdown,
which is real-user data pulled by URL and independent of that bad load.

PSI runs Lighthouse on Google's infrastructure, so this is a clean run without
local-machine noise. Mobile is the primary target (mobile-first indexing), and
since Lighthouse can't measure INP, TBT is the lab proxy for it.

## Network Activity

Captured 5 July 2026 from the DevTools Network panel (desktop, no throttling):
fresh load, then a soft refresh. Transfer size first, uncompressed resource size
in parentheses. Screenshots in `evidence/03-networking/`.

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
