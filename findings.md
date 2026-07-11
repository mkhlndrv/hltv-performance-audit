# Findings — HLTV homepage

Grouped by area, corrective first within each. Each finding is a distinct,
independently observable problem — where several causes feed one symptom, they
sit under that finding. Numbers come from the PSI mobile Insights/Diagnostics, the
DevTools Network panel, the throttled-mobile PSI column (Slow 4G + 4× CPU), and a
by-hand DevTools inspection of the build outputs — see [baseline](baseline.md).

Context: at the 75th percentile HLTV already passes Core Web Vitals (see the
"Good" findings). The corrective findings are the low-end, slow-network,
cold-cache experience the lab profile exposes — the long tail, not the median
visit.

## Prioritization (WSJF)

Corrective findings are ranked with **WSJF** — Weighted Shortest Job First: Cost
of Delay ÷ Job Size — so the cheapest high-value fixes float up. It's a different
lens from a plain impact-vs-effort split: Cost of Delay is built from three
sub-scores, which is what lets unrelated fixes be compared on one number. It's
subjective by design; the value is a written-down, repeatable basis.

Cost of Delay = User value + Time criticality + Risk/opportunity (each 1–10):

- **User value (1–10)** — how much the fix improves the real experience, weighted
  to mobile (the primary profile).
- **Time criticality (1–10)** — how urgent it is. HLTV already passes field Core
  Web Vitals, so this sits low across the board — higher only where mobile ranking
  pressure applies or the problem worsens as the site grows. It's what keeps the
  ranking honest: these are improvements, not fires.
- **Risk / opportunity (1–10)** — does it cut risk (third-party/privacy, or
  protect the passing field score from regressing) or unblock later work (e.g.,
  splitting the CSS enables more)?

**Job Size (1–10)** — rough relative effort: 1 = an attribute; 10 = a code-split
or CSS rewrite.

WSJF = Cost of Delay ÷ Job Size. Higher = do sooner. Good findings aren't scored —
they're observations, not work.

| # | Finding | UV | TC | R/O | CoD | Job | WSJF | Tier |
|---|---------|:--:|:--:|:---:|:---:|:---:|:----:|:----:|
| 1 | Hero image loads late (LCP) | 8 | 6 | 4 | 18 | 1 | **18.0** | High |
| 2 | Top banner is a 624 KiB GIF | 5 | 3 | 2 | 10 | 2 | **5.0** | High |
| 3 | Mobile gets desktop-sized images | 6 | 4 | 3 | 13 | 3 | **4.3** | Med |
| 4 | Ranking photos oversized for their box | 5 | 3 | 3 | 11 | 3 | **3.7** | Med |
| 5 | Third-party tag/consent/ad stack | 6 | 4 | 8 | 18 | 5 | **3.6** | Med |
| 6 | Theme CSS ~2.3 MB, ~99% unused | 7 | 5 | 6 | 18 | 5 | **3.6** | Med |
| 7 | Font Awesome library ~99% unused | 3 | 2 | 2 | 7 | 2 | **3.5** | Med |
| 8 | Blank screen before render (FCP) | 7 | 5 | 6 | 18 | 6 | **3.0** | Med |
| 9 | Critical CSS not extracted; theme sheet blocks paint | 6 | 4 | 5 | 15 | 5 | **3.0** | Med |
| 10 | `hltv.js` ~3 MB, 42% unused, not split | 7 | 5 | 6 | 18 | 7 | **2.6** | Low |
| 11 | Far slower on mobile than desktop | 9 | 6 | 5 | 20 | 8 | **2.5** | Low |
| 12 | Render-blocking JS delays interactivity | 5 | 4 | 5 | 14 | 6 | **2.3** | Low |
| 13 | Freezes while loading (TBT) | 6 | 4 | 5 | 15 | 7 | **2.1** | Low |

The one-line hero `fetchpriority` fix tops the list at 18, and contained asset
swaps (the GIF, responsive images) beat the big rewrites. The mobile umbrella
(row 7) scores *low* despite mobile being the primary profile — not because it
doesn't matter but because its job size is the whole rendering program; read it as
"high value, big project." And because HLTV already passes for real users, nothing
here is urgent — WSJF is sorting nice-to-haves, which is the right posture for a
site that's healthy in the field.

## Rendering

- The page is blank for about three seconds before anything renders.
  - **Baseline**: FCP (3.2 s mobile lab).
  - **Cause**:
    - Render-blocking resources in the critical path: a 2.3 MB stylesheet (`EverythingDay.css`, ~311 KiB brotli), the 563 KiB `hltv.js`, and `font-awesome.min.css`.
    - Two third-party scripts also block render: cadmus `script.ac` (2,660 ms) and the Cookiebot consent script (1,800 ms). No origins are preconnected.
    - PSI estimates ~10,180 ms of render-blocking savings.
  - **Solution**:
    - Inline the critical CSS and load the rest non-blocking; split `EverythingDay.css` so a page only loads what it needs.
    - Defer/`async` the scripts, load the consent script without blocking render, and add `preconnect` to the key origins.
  - **Priority (WSJF)**: CoD 18 (UV 7 + TC 5 + R/O 6) ÷ Job 6 = **3.0** (Med).

- The main article image appears late.
  - **Baseline**: LCP (10.7 s mobile lab / 1.8 s field — field passes).
  - **Cause**:
    - The LCP hero image (`img-cdn.hltv.org`) is discoverable and not lazy-loaded, but has no `fetchpriority="high"`, so it competes for bandwidth instead of loading first.
    - It also sits behind the render-blocking chain above. The field LCP breakdown shows load-delay (745 ms) as the biggest component.
  - **Solution**:
    - Add `fetchpriority="high"` and a preload to the hero image, and preconnect to `img-cdn.hltv.org`.
  - **Priority (WSJF)**: CoD 18 (UV 8 + TC 6 + R/O 4) ÷ Job 1 = **18.0** (High — the top quick win: one attribute plus a preload at the failing lab LCP).

- The page freezes when you scroll or tap while it's still loading.
  - **Baseline**: TBT (970 ms mobile lab); main-thread work 2.8 s.
  - **Cause**:
    - 5.6 MB of first-party JavaScript to parse and run — PSI flags 458 KiB of it unused and 43 KiB of unnecessary legacy polyfills.
    - Two scripts force synchronous reflows: cadmus `script.ac` (99 ms) and liftdsp's `admtracker` (171 ms).
  - **Solution**:
    - Code-split and defer non-critical JS, ship modern ES (drop the legacy transpilation), and remove the unused code.
    - Stop reading layout (`offsetWidth` and similar) right after writing to the DOM.
  - **Priority (WSJF)**: CoD 15 (UV 6 + TC 4 + R/O 5) ÷ Job 7 = **2.1** (Low — worthwhile, but the code-split is the biggest job here).

- A stack of third-party tag, consent and ad scripts loads on every visit.
  - **Baseline**: TBT and Speed Index (plus bytes and privacy).
  - **Cause**:
    - Google Tag Manager 490 KiB (186 ms), Cookiebot 316 KiB (284 ms), Facebook 164 KiB (135 ms) and cadmus 163 KiB (265 ms), plus Outbrain, Yahoo, Doubleclick and Google Analytics.
  - **Solution**:
    - Load tag-manager and analytics after first paint, load the consent script without blocking render, and drop or defer the ad/tracking calls that aren't needed at load.
  - **Priority (WSJF)**: CoD 18 (UV 6 + TC 4 + R/O 8) ÷ Job 5 = **3.6** (Med — the R/O score is high: this is the main third-party/privacy risk cut).

## Networking

### Corrective

- Ranking images are downloaded far bigger than they're shown.
  - **Baseline**: image bytes (2.4 MB of the load); Speed Index / LCP.
  - **Cause**:
    - The image CDN returns WebP (good), but the ranking player photos come back at 400×417 for a 70×73 display — roughly 100 KiB wasted per photo across several.
  - **Solution**:
    - Request the photos at display size with responsive `srcset`/`sizes`.
  - **Priority (WSJF)**: CoD 11 (UV 5 + TC 3 + R/O 3) ÷ Job 3 = **3.7** (Med).

- The top banner is a 624 KiB animated GIF.
  - **Baseline**: image bytes; Speed Index. PSI estimates ~425 KiB saveable.
  - **Cause**:
    - A single animated GIF is used for the top "day-only" banner instead of a video.
  - **Solution**:
    - Replace the GIF with a video (or a static image with the animation on interaction).
  - **Priority (WSJF)**: CoD 10 (UV 5 + TC 3 + R/O 2) ÷ Job 2 = **5.0** (High — one heavy asset, contained swap).

### Good

- Text compression is effective.
  - **Baseline**: network compression.
  - JS and CSS ship brotli/gzip (one script even zstd): 8.0 MB uncompressed comes down as 1.7 MB (~78 %). Images aren't gzipped, which is correct — they're already compressed.

- First-party caching is effective.
  - **Baseline**: network caching.
  - Static assets are content-hashed with a long TTL, so a soft refresh drops transfer from 5.3 MB to 1.3 MB (~75 % less).

- Images already use a modern format.
  - **Baseline**: image format.
  - The CDN serves WebP, so the remaining image issue is dimensions, not format.

## Build output

Findings from the shipped build outputs (Day 7) — how the JS and CSS assets are
bundled. See the [baseline](baseline.md) bundle analysis. (Source maps aren't
exposed, so that's not a finding.)

- The theme stylesheet ships ~2.3 MB and is ~99% unused.
  - **Baseline**: one monolithic `EverythingNight.css` / `EverythingDay.css` — ~2.3 MB uncompressed (311 KB brotli), render-blocking on every page; DevTools Coverage measures **~99% of it unused** on the homepage (2.28 MB of 2.31 MB).
  - **Cause**:
    - The whole theme ships as one file with no per-route or per-component splitting, so a page loads styling for every template on the site.
  - **Solution**:
    - Extract the critical CSS the page needs and load the rest non-blocking; split or purge the sheet so a page only ships the rules it uses.
  - **Priority (WSJF)**: CoD 18 (UV 7 + TC 5 + R/O 6) ÷ Job 5 = **3.6** (Med).

- Font Awesome 4.7 loads the whole icon library for a few icons.
  - **Baseline**: `font-awesome.min.css` (plus its webfont); Coverage measures **98.9% unused** (30.5 KB of 31 KB CSS, and the font downloads in full).
  - **Cause**:
    - The entire Font Awesome 4.7 set is pulled in though the page uses only a handful of glyphs.
  - **Solution**:
    - Subset Font Awesome to the icons in use, or replace them with inline SVGs and drop the dependency.
  - **Priority (WSJF)**: CoD 7 (UV 3 + TC 2 + R/O 2) ÷ Job 2 = **3.5** (Med — a cheap, self-contained win).

- The main `hltv.js` bundle is ~3 MB and isn't code-split.
  - **Baseline**: a single `hltv.js` — ~3.0 MB uncompressed (577 KB brotli), render-blocking and dominant in the LCP path (~7.5 s in PSI); Coverage measures **42% unused** on the homepage (1.26 MB of 3.0 MB), and `hltv-base-js.js` is 51% unused.
  - **Cause**:
    - Everything is concatenated into one main bundle (content-hashed but not split by route or component), so the homepage parses and runs code for the whole site.
  - **Solution**:
    - Split by route and lazy-load heavy or rarely-used components on interaction/visibility, so each page ships only the JS it needs.
  - **Priority (WSJF)**: CoD 18 (UV 7 + TC 5 + R/O 6) ÷ Job 7 = **2.6** (Low — big win, but restructuring a 3 MB bundle is the largest job here).

## Frames and layers

Day 8 pass — critical CSS, the flame chart and compositing; see the baseline's
rendering/frames/layers section and `evidence/07-frames-layers/`. HLTV is healthy
here, so this set is one corrective per issue found plus a "good" — the Layers set is
genuinely a strength, not a corrective (forcing one would be dishonest).

### Corrective

- Critical CSS isn't extracted — the theme sheet blocks first paint.
  - **Baseline**: the `<head>` has no critical-CSS block — just a leftover `#asdasda {}` debug rule, a Cookiebot style and Font Awesome — then the ~2.3 MB theme sheet (`EverythingNight.css`) loads render-blocking via a `<link>` and a theme-switch script; Coverage shows ~99% of it unused.
  - **Cause**:
    - No above-the-fold critical CSS and no defer: the whole theme sheet gates the first paint on every load.
  - **Solution**:
    - Inline the critical CSS and load the theme sheet non-blocking (preload + swap); drop the leftover debug `<style>`.
  - **Priority (WSJF)**: CoD 15 (UV 6 + TC 4 + R/O 5) ÷ Job 5 = **3.0** (Med).

- Render-blocking JS delays interactivity.
  - **Baseline**: WebPageTest (clean, no extensions) shows **TBT 0.71 s** and "took a long time to become interactive"; the DevTools flame chart is inflated by browser extensions, but HLTV's own share is `hltv.js` evaluate-script plus GTM (241 ms) and cadmus (191 ms) on load.
  - **Cause**:
    - `hltv.js` is render-blocking and evaluates on the main thread during load, and the tag/ad scripts add to it, so the page is slow to become interactive even though first paint eventually lands.
  - **Solution**:
    - Defer and code-split `hltv.js`, and load GTM/cadmus after first paint (same lever as the bundle and third-party findings above).
  - **Priority (WSJF)**: CoD 14 (UV 5 + TC 4 + R/O 5) ÷ Job 6 = **2.3** (Low).

### Good

- Compositing is lean.
  - **Baseline**: the Layers panel shows **9 layers using 103 MB** of GPU memory, with **no slow-scroll regions** — versus AP's 20 layers / 814 MB. The rootScroller uses accelerated scrolling; the rest are ordinary promotions (nav, columns, sidebar strips).
  - Nothing is force-promoted with `will-change`/`translateZ`, so compositing isn't a jank source — a genuine strength, and the reason this question set has no corrective.

## Mobile

Mobile is Google's primary ranking signal, and Slow 4G + 4× CPU is the condition
the audit weights first. Both scores are recorded (PSI Performance 39 mobile / 69
desktop); the Day 6 gap metric — mobile TBT ÷ desktop TBT — is 970 ÷ 410 = **2.4×**.
HLTV's field data passes on mobile, so these are the throttled low-end tail — but
that's what the lab exposes and the profile mobile ranking is scored on. Each
finding is tagged **mobile-amplified** (worse under throttling) or
**mobile-exclusive** (only on mobile).

### Corrective

- The same page is far slower on mobile than on desktop. *(mobile-amplified)*
  - **Baseline**: throttled-mobile lab (Slow 4G + 4× CPU) vs desktop lab — Performance 39 vs 69, LCP 10.7 s vs 1.8 s (~6×), FCP 3.2 s vs 0.8 s, Speed Index 10.2 s vs 2.4 s, TBT 970 ms vs 410 ms.
  - **Cause**:
    - Mobile and desktop download almost the same bytes (5.2 MB vs 5.3 MB), so the gap isn't payload — it's the render-blocking chain (est. 10,180 ms; `hltv.js` 563 KiB / ~7,530 ms, `EverythingDay.css` 2.3 MB) and the JS main-thread work meeting a 4× slower CPU and a slow radio.
  - **Solution**:
    - The rendering fixes elsewhere in this report (inline critical CSS, split and defer the CSS/JS, `fetchpriority` on the hero) barely move the desktop score but are what rescue mobile — budget and test against Slow 4G + 4× CPU, not a desktop link.
  - **Priority (WSJF)**: CoD 20 (UV 9 + TC 6 + R/O 5) ÷ Job 8 = **2.5** (Low — an umbrella finding; its job size is the whole rendering program, so WSJF ranks the ROI low even though the value is the highest of the set).

- Mobile downloads desktop-sized images. *(mobile-amplified)*
  - **Baseline**: images are 2.2 MB of the 5.2 MB mobile load; Speed Index / LCP.
  - **Cause**:
    - There's no responsive `srcset`, so a phone fetches the same full-size assets desktop does — the ranking photos at 400×417 (shown ~70×73) and the 624 KiB animated GIF banner — and on Slow 4G those image bytes dominate the load. (The oversized-image root is the networking finding above; the mobile-specific point is that nothing is downscaled for the small screen or the slow link.)
  - **Solution**:
    - Serve responsive `srcset`/`sizes` at display size and replace the GIF with a video or static image — the saving is largest on mobile.
  - **Priority (WSJF)**: CoD 13 (UV 6 + TC 4 + R/O 3) ÷ Job 3 = **4.3** (Med).

### Good

- The layout is rock-stable on mobile.
  - **Baseline**: CLS 0 (field) / 0 (PSI lab) — well under the 0.1 threshold.
  - Despite the ad slots, ranking widgets and the banner, nothing shifts as the page loads on a phone. Layout stability is the one thing HLTV nails on mobile; the problem is load speed, not jumpiness.

## Overall

### Good

- Real users already have a good experience.
  - **Baseline**: Core Web Vitals (field / CrUX).
  - HLTV passes Core Web Vitals at the 75th percentile on both mobile and desktop (mobile LCP 1.8 s, INP 96 ms, CLS 0). The corrective findings above are the low-end and slow-network tail, not the median visit.
